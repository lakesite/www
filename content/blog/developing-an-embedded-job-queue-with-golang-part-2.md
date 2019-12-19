---
title: "Developing an Embedded Job Queue With Golang - Part 2"
thumbnail: "/images/blog/9/thumbnail.png" 
date: 2019-12-11
tags: ["golang", "library", "concurrency", "task-queue"]
draft: false
id: 9
---

![head](/images/blog/9/head.png)

# Building a Factory #

In the article, [Developing an Embedded Job Queue With Golang - Part 1](/blog/developing-an-embedded-job-queue-with-golang-part-1/), we briefly reviewed the
excellent concurrency primitives Go offers, then prototyped an example job queue.

We're going to create a package for [reel](https://github.com/lakesite/reel) and
other applications to use, such that they can queue work to be done by workers
and eventually specify when the work should be done, check on the status of a 
job, and send commands to workers through the dispatcher. To restate our problem, 
we want to hand off long running tasks to workers so we can immediately return 
and respond to (for example) a web request. We're modeling a task queue or 
[job queue](https://en.wikipedia.org/wiki/Job_queue) using goroutines and 
channels.  

We could do this directly in reel.  The use case for reel is that we might want 
to query an API endpoint to see the status of our task, if it's completed, how 
long the task has taken, and we may also want to cancel the job.  Further we can
generalize this and create our own message queue system, which uses our 
conventions such that clients can subscribe and listen for work that's 
published, perform the work and respond with the result.  Before we consider 
doing that, we'll create an embedded task queue package in Go.

## I. ls-factory ##

Enter [ls-factory](https://github.com/lakesite/ls-factory/) - a simple task 
queue.  Like all projects, we'll start small and incrementally add in features.
To start with we'll focus on creating jobs, workers and a dispatcher.  Once we
have these working, we'll extend the package to generalize a message queue with
an event bus between job publishers and job subscribers (client workers) in a
[PubSub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) 
pattern.  For now, we want an embedded task queue package we can re-use that
gives us some extra features outside of simply using some channels and 
goroutines.

As we model a task queue, the idea of a factory comes to mind, as we're 
dispatching jobs to workers and getting work done.  As we proceed we'll add in
most of the following features:

    1. Enqueue jobs to be executed.
    2. Specify an arbitrary number of workers.
    3. Handle processing functions by workers.
    4. Communicate status of jobs through the dispatcher.
    5. Communicate orders to workers, such as halting the work.
    6. Track the result of the job.
    7. Specify a time for a job to be executed.
    8. Set execution priority for a job.

## II. job.go ##

The simple part of factory is the job definition.  We could, and will, extend 
this further over time.  For now we will define a structure for jobs: a unique 
ID, start and end run time, completed, (running is inferred through start time).

The code for job.go follows:

{{< highlight go "linenos=inline" >}}
package factory

import (
	"time"
)

// JobExecutable defines our spec for jobs, which are functions that return
// an error.
type JobExecutable func() error

// Job has an ID, start time, end time and reference to a JobExecutable.
type Job struct {
	ID         int
	StartTime  time.Time
	EndTime    time.Time
	Executable JobExecutable
}

func CreateNewJob(id int, je JobExecutable) *Job {
	return &Job{
		ID:         id,
		Executable: je,
	}
}
{{< / highlight >}}

## III. worker.go ##

The worker will concern itself with waiting for jobs, processing them and 
responding to signals.  It should perform:

    1. Process jobs from the job queue channel.
    2. Respond to orders from the dispatcher to halt or get job status.
    3. Set properties on a job (start time, end time, result).

We'll add additional features to the worker during our next iteration, for now
we'll keep things simple.

{{< highlight go "linenos=inline" >}}
package factory

import (
	log "github.com/sirupsen/logrus"
	"time"
)

// Worker has an ID, internal channel for jobs, internal channel for reporting
// status to the dispatcher, and a channel for quitting.
type Worker struct {
	ID             int
	jobs           chan *Job
	dispatchStatus chan *DispatchStatus
	workerCmd      chan *WorkerCommand
	log            *log.Logger
}

type WorkerCommand struct {
	ID      int
	Command string
}

// CreateNewWorker accepts an ID, channel for worker registration, channel for
// jobs, and a channel for dispatch reports.
func CreateNewWorker(id int, wCmd chan *WorkerCommand, jobQueue chan *Job, dStatus chan *DispatchStatus, l *log.Logger) *Worker {
	w := &Worker{
		ID:             id,
		jobs:           jobQueue,
		dispatchStatus: dStatus,
		workerCmd:      wCmd,
		log:            l,
	}

	return w
}

// Start enables the worker for processing jobs.
func (w *Worker) Start() {
	go func() {
		for {
			select {
			case job := <-w.jobs:
				w.log.WithFields(log.Fields{
					"ID":     w.ID,
					"Job ID": job.ID,
				}).Info("Worker executing job.")
				job.StartTime = time.Now()
				w.dispatchStatus <- &DispatchStatus{
					Type:   DTYPE_JOB,
					ID:     job.ID,
					Status: DSTATUS_START,
				}

				job.Executable()
				job.EndTime = time.Now()
				w.dispatchStatus <- &DispatchStatus{
					Type:   DTYPE_JOB,
					ID:     job.ID,
					Status: DSTATUS_END,
				}

			case wc := <-w.workerCmd:
				if wc.ID == w.ID || w.ID == 0 {
					if wc.Command == WCMD_QUIT {
						w.log.WithFields(log.Fields{
							"ID": w.ID,
						}).Info("Worker received quit command.")
						w.dispatchStatus <- &DispatchStatus{
							Type:   DTYPE_WORKER,
							ID:     w.ID,
							Status: DSTATUS_QUIT,
						}
						return
					}
				}
			}
		}
	}()
}
{{< / highlight >}}

The bulk of our code is handled in the for - select section of the Start 
function, which listens for commands or jobs and handles them appropriately.

## IV. dispatcher.go ##

The dispatcher will handle most of the work in our library, it performs
the following:

    1. Accept jobs in the form of an anonymous wrapped function, from main.go
    2. Maintains a count of jobs.
    3. Accept a request for a number of workers to perform the work.
    4. Creates workers.
    5. Dispatch work to the workers.

The code:

{{< highlight go "linenos=inline" >}}
package factory

import (
	"errors"

	log "github.com/sirupsen/logrus"
)

/*
 * DispatchStatus is a struct for passing job and worker status reports.
 * Type can be: worker, job, ID represents the ID of the worker or job.
 * Status: see WSTATUS constants.
 */
type DispatchStatus struct {
	Type   string
	ID     int
	Status string
}

// Dispatch keeps track of an internal job request queue, a work queue of jobs
// that will be processed, a worker queue of workers, and a channel for status
// reports for jobs and workers.
type Dispatcher struct {
	jobCounter     int                  // internal counter for number of jobs
	jobQueue       chan *Job            // channel of submitted jobs
	dispatchStatus chan *DispatchStatus // channel for job/worker status reports
	workQueue      chan *Job            // channel of work dispatched
	workerQueue    chan *Worker         // channel of workers
	workerCommand  chan *WorkerCommand  // channel for worker commands
	log            *log.Logger          // log API (logrus)
	running        bool                 // Is the dispatcher running
}

// CreateNewDispatcher creates a new dispatcher by making the necessary channels
// for goroutines to communicate and initializes an internal job counter.
func CreateNewDispatcher(l *log.Logger) *Dispatcher {
	return &Dispatcher{
		jobCounter:     0,
		jobQueue:       make(chan *Job),
		dispatchStatus: make(chan *DispatchStatus),
		workQueue:      make(chan *Job),
		workerCommand:  make(chan *WorkerCommand),
		log:            l,
		running:        false,
	}
}

// QueueJob accepts a process (func() error) of JobExecutable, creates a job
// which will be tracked, and adds the job into the internal work queue for
// execution.
// If there's a constraint on the number of jobs, return an error.
func (d *Dispatcher) QueueJob(je JobExecutable) error {
	// Create a new job:
	j := CreateNewJob(d.jobCounter, je)

	// Add the job to the internal queue:
	go func() { d.jobQueue <- j }()

	// Increment the internal job counter:
	d.jobCounter++
	d.log.WithFields(log.Fields{
		"jobCounter": d.jobCounter,
	}).Info("Job Queued.")

	return nil
}

// Finished returns true if we have no more jobs to process, otherwise false.
func (d *Dispatcher) Finished() bool {
	if d.jobCounter < 1 {
		return true
	} else {
		return false
	}
}

// Running returns true if the dispatcher has been issued Start() and is running
func (d *Dispatcher) Running() bool {
	return d.running
}

// Start has the dispatcher create workers to handle jobs, then creates a
// goroutine to handle passing jobs in the queue off to workers and processing
// dispatch status reports.
func (d *Dispatcher) Start(numWorkers int) error {
	if numWorkers < 1 {
		return errors.New("Start requires >= 1 workers.")
	}

	// Create numWorkers:
	for i := 0; i < numWorkers; i++ {
		worker := CreateNewWorker(i, d.workerCommand, d.workQueue, d.dispatchStatus, d.log)
		worker.Start()
	}

	d.running = true

	// wait for work to be added then pass it off.
	go func() {
		for {
			select {
			case job := <-d.jobQueue:
				d.log.WithFields(log.Fields{
					"ID": job.ID,
				}).Info("Adding a new job to the queue for dispatching.")
				// Add the job to the work queue, and don't block the dispatcher.
				go func() { d.workQueue <- job }()

			case ds := <-d.dispatchStatus:
				d.log.WithFields(log.Fields{
					"Type":   ds.Type,
					"ID":     ds.ID,
					"Status": ds.Status,
				}).Info("Received a dispatch status report.")

				if ds.Type == DTYPE_WORKER {
					if ds.Status == DSTATUS_QUIT {
						d.log.WithFields(log.Fields{
							"ID": ds.ID,
						}).Info("Worker quit.")
					}
				}

				if ds.Type == DTYPE_JOB {
					if ds.Status == DSTATUS_START {
						d.log.WithFields(log.Fields{
							"ID": ds.ID,
						}).Info("Job started.")
					}

					if ds.Status == DSTATUS_END {
						d.log.WithFields(log.Fields{
							"ID": ds.ID,
						}).Info("Job finished.")
						d.jobCounter--
					}
				}
			}
		}
	}()
	return nil
}
{{< / highlight >}}

Like the worker, our dispatcher is primarily handled in the Start function.

## V. constants.go ##

The dispatcher and worker need a definition of messaging through constants, so
we'll define them in constants.go.  These define dispatch types, dispatch 
statuses, and worker commands.

{{< highlight go "linenos=inline" >}}
package factory

const (
	DTYPE_JOB    = "job"
	DTYPE_WORKER = "worker"

	DSTATUS_START = "start"
	DSTATUS_END   = "end"
	DSTATUS_QUIT  = "quit"

	WCMD_QUIT = "quit"
)
{{< / highlight >}}

## VI. main.go ##

This is our example application which uses the embedded job queue.  It will 
import the package, create some dummy work that emulates a long running job, and
document our package so others can see how to use it or create their own.

The code:

{{< highlight go "linenos=inline" >}}
package main

import (
	"time"

	"github.com/lakesite/ls-factory"
	"github.com/sirupsen/logrus"
)

// GenericJobStruct is a generic structure we'll use for submitting a job,
// this can be any structure.
type GenericJobStruct struct {
	Name string
}

// An example function, ProcessWork.
func (gjs *GenericJobStruct) ProcessWork() error {
	// Generic process, let's print out something, then wait for a time:
	log.WithFields(logrus.Fields{"Name": gjs.Name}).Info("Processing some work.")
	time.Sleep(5 * time.Second)
	log.Info("Work finished.")
	return nil
}

var log = logrus.New()

func main() {
	// Create two separate jobs to execute:
	gjs1 := &GenericJobStruct{Name: "Testing1"}
	gjs2 := &GenericJobStruct{Name: "Testing2"}

	// We'll use the convention of simply wrapping the work in a function literal.
	process1 := func() error {
		log.WithFields(logrus.Fields{"Name": gjs1.Name}).Info("Wrapping work in an anonymous function.")
		return gjs1.ProcessWork()
	}

	process2 := func() error {
		log.WithFields(logrus.Fields{"Name": gjs2.Name}).Info("Wrapping work in an anonymous function.")
		return gjs2.ProcessWork()
	}

	// Create a dispatcher:
	dispatcher := factory.CreateNewDispatcher(log)

	// Enqueue n*2 jobs:
	n := 10
	for i:= 0; i<n; i++ {
		dispatcher.QueueJob(process1)
		dispatcher.QueueJob(process2)
	}

	// Tell the dispatcher to start processing jobs with n workers:
	if err := dispatcher.Start(n); err != nil {
		log.Fatal("Dispatch startup error: %s", err.Error())
	}

	for dispatcher.Running() {
		if dispatcher.Finished() {
			log.Info("All jobs finished.")
			break
		}
	}
}
{{< / highlight >}}

Here's the output of a run from the above:

```
$ ./main
INFO[0000] Job Queued.                                   jobCounter=1
INFO[0000] Job Queued.                                   jobCounter=2
INFO[0000] Job Queued.                                   jobCounter=3
INFO[0000] Job Queued.                                   jobCounter=4
INFO[0000] Job Queued.                                   jobCounter=5
INFO[0000] Job Queued.                                   jobCounter=6
INFO[0000] Job Queued.                                   jobCounter=7
INFO[0000] Job Queued.                                   jobCounter=8
INFO[0000] Job Queued.                                   jobCounter=9
INFO[0000] Job Queued.                                   jobCounter=10
INFO[0000] Job Queued.                                   jobCounter=11
INFO[0000] Job Queued.                                   jobCounter=12
INFO[0000] Job Queued.                                   jobCounter=13
INFO[0000] Job Queued.                                   jobCounter=14
INFO[0000] Job Queued.                                   jobCounter=15
INFO[0000] Job Queued.                                   jobCounter=16
INFO[0000] Job Queued.                                   jobCounter=17
INFO[0000] Job Queued.                                   jobCounter=18
INFO[0000] Job Queued.                                   jobCounter=19
INFO[0000] Job Queued.                                   jobCounter=20
INFO[0000] Adding a new job to the queue for dispatching.  ID=0
INFO[0000] Adding a new job to the queue for dispatching.  ID=1
INFO[0000] Adding a new job to the queue for dispatching.  ID=2
INFO[0000] Adding a new job to the queue for dispatching.  ID=3
INFO[0000] Adding a new job to the queue for dispatching.  ID=4
INFO[0000] Adding a new job to the queue for dispatching.  ID=5
INFO[0000] Adding a new job to the queue for dispatching.  ID=6
INFO[0000] Worker executing job.                         ID=2 Job ID=2
INFO[0000] Worker executing job.                         ID=3 Job ID=1
INFO[0000] Adding a new job to the queue for dispatching.  ID=7
INFO[0000] Adding a new job to the queue for dispatching.  ID=8
INFO[0000] Adding a new job to the queue for dispatching.  ID=9
INFO[0000] Received a dispatch status report.            ID=2 Status=start Type=job
INFO[0000] Job started.                                  ID=2
INFO[0000] Worker executing job.                         ID=9 Job ID=8
INFO[0000] Worker executing job.                         ID=6 Job ID=4
INFO[0000] Worker executing job.                         ID=1 Job ID=7
INFO[0000] Adding a new job to the queue for dispatching.  ID=10
INFO[0000] Received a dispatch status report.            ID=1 Status=start Type=job
INFO[0000] Job started.                                  ID=1
INFO[0000] Worker executing job.                         ID=4 Job ID=5
INFO[0000] Worker executing job.                         ID=8 Job ID=6
INFO[0000] Worker executing job.                         ID=0 Job ID=0
INFO[0000] Worker executing job.                         ID=7 Job ID=3
INFO[0000] Worker executing job.                         ID=5 Job ID=9
INFO[0000] Wrapping work in an anonymous function.       Name=Testing1
INFO[0000] Processing some work.                         Name=Testing1
INFO[0000] Received a dispatch status report.            ID=8 Status=start Type=job
INFO[0000] Job started.                                  ID=8
INFO[0000] Wrapping work in an anonymous function.       Name=Testing1
INFO[0000] Processing some work.                         Name=Testing1
INFO[0000] Received a dispatch status report.            ID=4 Status=start Type=job
INFO[0000] Job started.                                  ID=4
INFO[0000] Wrapping work in an anonymous function.       Name=Testing2
INFO[0000] Processing some work.                         Name=Testing2
INFO[0000] Wrapping work in an anonymous function.       Name=Testing1
INFO[0000] Processing some work.                         Name=Testing1
INFO[0000] Received a dispatch status report.            ID=7 Status=start Type=job
INFO[0000] Job started.                                  ID=7
INFO[0000] Wrapping work in an anonymous function.       Name=Testing2
INFO[0000] Wrapping work in an anonymous function.       Name=Testing2
INFO[0000] Processing some work.                         Name=Testing2
INFO[0000] Processing some work.                         Name=Testing2
INFO[0000] Received a dispatch status report.            ID=5 Status=start Type=job
INFO[0000] Job started.                                  ID=5
INFO[0000] Received a dispatch status report.            ID=6 Status=start Type=job
INFO[0000] Job started.                                  ID=6
INFO[0000] Adding a new job to the queue for dispatching.  ID=12
INFO[0000] Wrapping work in an anonymous function.       Name=Testing1
INFO[0000] Processing some work.                         Name=Testing1
INFO[0000] Received a dispatch status report.            ID=0 Status=start Type=job
INFO[0000] Job started.                                  ID=0
INFO[0000] Adding a new job to the queue for dispatching.  ID=11
INFO[0000] Adding a new job to the queue for dispatching.  ID=13
INFO[0000] Wrapping work in an anonymous function.       Name=Testing1
INFO[0000] Processing some work.                         Name=Testing1
INFO[0000] Adding a new job to the queue for dispatching.  ID=14
INFO[0000] Adding a new job to the queue for dispatching.  ID=15
INFO[0000] Received a dispatch status report.            ID=3 Status=start Type=job
INFO[0000] Job started.                                  ID=3
INFO[0000] Received a dispatch status report.            ID=9 Status=start Type=job
INFO[0000] Wrapping work in an anonymous function.       Name=Testing2
INFO[0000] Processing some work.                         Name=Testing2
INFO[0000] Job started.                                  ID=9
INFO[0000] Adding a new job to the queue for dispatching.  ID=16
INFO[0000] Adding a new job to the queue for dispatching.  ID=17
INFO[0000] Adding a new job to the queue for dispatching.  ID=18
INFO[0000] Wrapping work in an anonymous function.       Name=Testing2
INFO[0000] Processing some work.                         Name=Testing2
INFO[0000] Adding a new job to the queue for dispatching.  ID=19
INFO[0005] Work finished.                               
INFO[0005] Worker executing job.                         ID=2 Job ID=10
INFO[0005] Received a dispatch status report.            ID=2 Status=end Type=job
INFO[0005] Job finished.                                 ID=2
INFO[0005] Received a dispatch status report.            ID=10 Status=start Type=job
INFO[0005] Job started.                                  ID=10
INFO[0005] Wrapping work in an anonymous function.       Name=Testing1
INFO[0005] Processing some work.                         Name=Testing1
INFO[0005] Work finished.                               
INFO[0005] Worker executing job.                         ID=9 Job ID=12
INFO[0005] Received a dispatch status report.            ID=8 Status=end Type=job
INFO[0005] Job finished.                                 ID=8
INFO[0005] Received a dispatch status report.            ID=12 Status=start Type=job
INFO[0005] Job started.                                  ID=12
INFO[0005] Wrapping work in an anonymous function.       Name=Testing1
INFO[0005] Processing some work.                         Name=Testing1
INFO[0005] Work finished.                               
INFO[0005] Worker executing job.                         ID=3 Job ID=11
INFO[0005] Received a dispatch status report.            ID=1 Status=end Type=job
INFO[0005] Job finished.                                 ID=1
INFO[0005] Received a dispatch status report.            ID=11 Status=start Type=job
INFO[0005] Job started.                                  ID=11
INFO[0005] Wrapping work in an anonymous function.       Name=Testing2
INFO[0005] Processing some work.                         Name=Testing2
INFO[0005] Work finished.                               
INFO[0005] Worker executing job.                         ID=6 Job ID=13
INFO[0005] Received a dispatch status report.            ID=4 Status=end Type=job
INFO[0005] Job finished.                                 ID=4
INFO[0005] Received a dispatch status report.            ID=13 Status=start Type=job
INFO[0005] Wrapping work in an anonymous function.       Name=Testing2
INFO[0005] Processing some work.                         Name=Testing2
INFO[0005] Job started.                                  ID=13
INFO[0005] Work finished.                               
INFO[0005] Worker executing job.                         ID=4 Job ID=14
INFO[0005] Received a dispatch status report.            ID=5 Status=end Type=job
INFO[0005] Job finished.                                 ID=5
INFO[0005] Received a dispatch status report.            ID=14 Status=start Type=job
INFO[0005] Job started.                                  ID=14
INFO[0005] Wrapping work in an anonymous function.       Name=Testing1
INFO[0005] Processing some work.                         Name=Testing1
INFO[0005] Work finished.                               
INFO[0005] Worker executing job.                         ID=1 Job ID=15
INFO[0005] Received a dispatch status report.            ID=7 Status=end Type=job
INFO[0005] Job finished.                                 ID=7
INFO[0005] Received a dispatch status report.            ID=15 Status=start Type=job
INFO[0005] Job started.                                  ID=15
INFO[0005] Wrapping work in an anonymous function.       Name=Testing2
INFO[0005] Processing some work.                         Name=Testing2
INFO[0005] Work finished.                               
INFO[0005] Worker executing job.                         ID=8 Job ID=16
INFO[0005] Received a dispatch status report.            ID=6 Status=end Type=job
INFO[0005] Job finished.                                 ID=6
INFO[0005] Received a dispatch status report.            ID=16 Status=start Type=job
INFO[0005] Job started.                                  ID=16
INFO[0005] Wrapping work in an anonymous function.       Name=Testing1
INFO[0005] Processing some work.                         Name=Testing1
INFO[0005] Work finished.                               
INFO[0005] Worker executing job.                         ID=0 Job ID=18
INFO[0005] Received a dispatch status report.            ID=0 Status=end Type=job
INFO[0005] Job finished.                                 ID=0
INFO[0005] Received a dispatch status report.            ID=18 Status=start Type=job
INFO[0005] Job started.                                  ID=18
INFO[0005] Wrapping work in an anonymous function.       Name=Testing1
INFO[0005] Processing some work.                         Name=Testing1
INFO[0005] Work finished.                               
INFO[0005] Worker executing job.                         ID=7 Job ID=17
INFO[0005] Received a dispatch status report.            ID=3 Status=end Type=job
INFO[0005] Job finished.                                 ID=3
INFO[0005] Received a dispatch status report.            ID=17 Status=start Type=job
INFO[0005] Job started.                                  ID=17
INFO[0005] Wrapping work in an anonymous function.       Name=Testing2
INFO[0005] Processing some work.                         Name=Testing2
INFO[0005] Work finished.                               
INFO[0005] Received a dispatch status report.            ID=9 Status=end Type=job
INFO[0005] Job finished.                                 ID=9
INFO[0005] Worker executing job.                         ID=5 Job ID=19
INFO[0005] Wrapping work in an anonymous function.       Name=Testing2
INFO[0005] Processing some work.                         Name=Testing2
INFO[0005] Received a dispatch status report.            ID=19 Status=start Type=job
INFO[0005] Job started.                                  ID=19
INFO[0010] Work finished.                               
INFO[0010] Received a dispatch status report.            ID=10 Status=end Type=job
INFO[0010] Job finished.                                 ID=10
INFO[0010] Work finished.                               
INFO[0010] Received a dispatch status report.            ID=12 Status=end Type=job
INFO[0010] Job finished.                                 ID=12
INFO[0010] Work finished.                               
INFO[0010] Received a dispatch status report.            ID=11 Status=end Type=job
INFO[0010] Job finished.                                 ID=11
INFO[0010] Work finished.                               
INFO[0010] Received a dispatch status report.            ID=13 Status=end Type=job
INFO[0010] Job finished.                                 ID=13
INFO[0010] Work finished.                               
INFO[0010] Received a dispatch status report.            ID=14 Status=end Type=job
INFO[0010] Job finished.                                 ID=14
INFO[0010] Work finished.                               
INFO[0010] Received a dispatch status report.            ID=15 Status=end Type=job
INFO[0010] Job finished.                                 ID=15
INFO[0010] Work finished.                               
INFO[0010] Received a dispatch status report.            ID=16 Status=end Type=job
INFO[0010] Job finished.                                 ID=16
INFO[0010] Work finished.                               
INFO[0010] Received a dispatch status report.            ID=18 Status=end Type=job
INFO[0010] Job finished.                                 ID=18
INFO[0010] Work finished.                               
INFO[0010] Received a dispatch status report.            ID=17 Status=end Type=job
INFO[0010] Job finished.                                 ID=17
INFO[0010] Work finished.                               
INFO[0010] Received a dispatch status report.            ID=19 Status=end Type=job
INFO[0010] Job finished.                                 ID=19
INFO[0010] All jobs finished.    
```

## VI. Conclusion ##

Our first iteration works and gives us a task queue with an arbitrary number of
workers which subscribe to events through a channel to perform work.  Next we'll
need to add more features to ls-factory, which will also allow us to create a
stand-alone messaging system (if we desire) instead of an embedded task queue.

* We need a message bus for dispatching commands to workers.
* We should actually tell our workers to quit if needed, close channels, etc.
* We should get status reports on jobs and store results.
* We should put constraints on the system (how much we store, etc)
* We should be able to specify a time to start a job.
* We should be able to specify priority for a job.
