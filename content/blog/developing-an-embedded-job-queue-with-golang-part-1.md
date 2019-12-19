---
title: "Developing an Embedded Job Queue With Golang - Part 1"
thumbnail: "/images/blog/8/thumbnail.png" 
date: 2019-12-09
tags: ["golang", "library", "concurrency", "task-queue"]
draft: false
id: 8 
---

![head](/images/blog/8/head.png)

# Task Queues #

In the article, [Using a Job Queue with Golang and Reel](/blog/using-a-job-queue-with-golang-and-reel/), we added a channel for handling rewind
requests through a simple job package.  We'd like to create a general library 
for other programs to use which supports the following features:

    1. Enqueue jobs to be executed.
    2. Specify an arbitrary number of workers.
    3. Handle processing functions by workers.
    4. Communicate status of jobs through the dispatcher.
    5. Communicate orders to workers, such as halting the work.
    6. Track the result of the job.

Eventually, we'd like to get more information from workers about jobs, such as
the run time of a process and if the job succeeded.

Go is an excellent choice to solve this problem, in particular due to the
specific concurrency features it provides which solve many problems for a 
developer while encouraging a way of problem solving that's beneficial for
parallel programming.  Before we dive into our implementation, some background
is necessary to appreciate the simplicity of our design.

## I. The Basics ##

Here's a simplified summary of what happens when you run a program.  When a 
program is executed by an operating system, the OS allocates memory space
and assigns that program a process ID.  This running process has at least one 
thread of execution known as a primary thread or main thread.  There are 
different strategies for utilizing access to system resources which processes
require to run.

### Threads ###

A thread is a sequence of instructions which are managed by a scheduler. Threads 
are a component of a process which is managed by the OS.  The scheduler keeps 
all the resources of a computer utilized by the many running processes which 
require system resources.  Managing and scheduling thread access to system resources 
allows us to have a multitasking environment.  A multitasking environment just 
means, many tasks appear to execute at the same time to the end users of a 
system (but, not really).  

All the threads of a process share that processes memory space.  Because the
threads share the same memory context and share resources such as file handles,
threads are often described as lightweight processes.  In systems with multiple
CPU cores, we can run threads in parallel by allowing different cores to run
them.  In this case, the threads of instructions may execute at the same time, 
in parallel.

### Concurrency ###

When we have a set of instructions to execute by a processor, we could execute
the instructions sequentially.  This means there would be no interruption of a
task, which would execute until that task has finished.  This would make running
an application on a system extremely inconvenient, as we have many processes
competing for system resources.

Concurrency is simply executing two or more tasks in overlapping time periods.
This does not mean that the tasks necessarily execute at the exact same time.
When the tasks do execute at the same time (via multiple CPU cores, for example)
we call this parallelism.

[Rob Pike](http://research.google.com/pubs/r.html) gave an excellent talk at 
[Heroku's Waza 2012](https://blog.heroku.com/concurrency_is_not_parallelism) to 
explain concurrency is better than parallelism. Rob Pike summarizes the 
difference between concurrency and parallelism with the following statements.

Concurrency is the "composition of independently executing processes."  Here we
are *dealing* with a lot of things at once.  Parallelism is the "simultaneous 
execution of things."  Here we are *doing* a lot of things at once.

[His talk](https://www.youtube.com/watch?v=cN_DpYBzKso) is worth watching.

### Go Design Philosophy ###

Go has excellent [documentation](https://golang.org/doc/effective_go.html#concurrency)
covering concurrency.  The go way of thinking when it comes to concurrent
programming is:  

*Do not communicate by sharing memory; instead, share memory by communicating.*  

This is because Go shares values passed around on channels,
where *only one goroutine* has access to the value at any given time.  If two
goroutines could write to the same location in memory at the same time, without
synchronizing their operations, we'd have a data race problem.  This is how Go 
resolves data races, which can't occur by design.

[The Go Blog](https://blog.golang.org/) article [Share Memory by Communicating](https://blog.golang.org/share-memory-by-communicating) by Andrew Gerrand asserts that 
Go's concurrency primitives (goroutines and channels) have roots in the concepts
outlined in C. A. R. Hoare's [Communicating Sequential Processes](https://www.cs.cmu.edu/~crary/819-f09/Hoare78.pdf).  This paper is also worth reading.

### Goroutines ###

A goroutine is a function executing concurrently with other goroutines in the
same address space.  Their design simplifies the complexity of thread creation
and management.  To pass messages between goroutines, we use channels. A 
summary of implementation details is provided by [Krishna Sundarram](https://blog.nindalf.com/posts/how-goroutines-work/).

When we preface 'go' in front of a function in Golang, we don't have to wait
for the function to return.  They share the same address space within the 
program, they get multiplexed dynamically onto operating system threads as 
required, so you don't have to worry about scheduling and blocking.  They "feel"
like threads, but Golang handles the scheduling, blocking, and reading details 
for you.

Goroutines are cheap.  You can have many more goroutines than you would threads 
in an application, and this is by design. This is because unlike threads, 
goroutines have extremely low upfront memory costs.  [Dave Cheney](https://dave.cheney.net/2014/06/07/five-things-that-make-go-fast) explains how the Go compiler inserts a check 
as part of every function call to ensure there's sufficient memory for the 
function to run, otherwise more stack space is allocated.  This allows us to 
create a goroutine with an initial stack size of 8kb, compared to 2 megabytes 
per thread on Linux.   With this design, you don't have to deal with memory 
issues for thousands of incoming requests (for example, to a web server) using a
thread pool.

Goroutines are also fast.  Goroutines are cooperatively scheduled and do not 
rely on the kernel to handle time sharing.  Switching between goroutines happens
at defined points when an explicit call is made to the Go runtime scheduler: 

    1. On channel sending and receiving.
    2. On the go statement.
    3. During a blocking syscall.
    4. When performing garbage collection.

When we say goroutines are fast, we mean they're well defined on what will block
them.  Goroutines do not cause the thread they're multiplexed on to block if 
they are blocked on: 
  
    1. Sleeping
    2. Network input
    3. Channel operations
    4. Blocking on primitives in the sync package.

### Channels ###

So we share memory by communicating through channels.  We can create an 
unbuffered channel to store string values, and then process them.  An example of
this would be:

{{< highlight go "linenos=inline" >}}
package main

import (
	"fmt"
)

func main() {
	stringchan := make(chan string)

	go func() {
		for {
			select {
			case message := <-stringchan:
				fmt.Printf("Got a message from stringchan: %s\n", message)
			}
		}
	}()

	fmt.Printf("Adding some messages...\n")

	stringchan <- "Hello there."
	stringchan <- "Another message."
	stringchan <- "Yet another message."
}
{{< / highlight >}}

If we only ran the above once, we might get all three messages.  If we run it
several times, we'll find that sometimes we don't get all the messages before
the program ends.  This is because the main goroutine exits and terminates the
anonymous function we have handling messages.  We can alter our program to wait
for messages in a variety of ways (such as using sync.WaitGroup).  

Let's alter our anonymous function to also handle messages to quit, and report
back that we're done on a separate channel:

{{< highlight go "linenos=inline" >}}
package main

import (
	"fmt"
)

func main() {
	stringchan := make(chan string)
	quitchan := make(chan bool)
	done := make(chan bool)

	go func() {
		for {
			select {
			case message := <-stringchan:
				fmt.Printf("Got a message from stringchan: %s\n", message)
			case <-quitchan:
				// We don't care what quitchan's value is here, we just return.
				done <- true
				return
			}
		}
	}()

	fmt.Printf("Adding some messages...\n")

	stringchan <- "Hello there."
	stringchan <- "Another message."
	stringchan <- "Yet another message."
	quitchan <- true

	<-done
	fmt.Printf("Exiting...\n")
}
{{< / highlight >}}

So our anonymous goroutine can wait for either a message or for quitchan.  If 
we receive anything on quitchan, we send true to the done channel and return. In
the main goroutine, we're blocked before exiting by waiting on a read from the 
done channel.  This allows us to send our messages, tell the anonymous goroutine
to quit, then wait for a response before exiting.

We can also have buffered channels, where sends to a buffered channel block only
when the buffer is full and receives from the channel block when the buffer is
empty.

### Select ###

Go allows you to use select to handle non-blocking reads against channels.  In
our above example we're able to handle reading from different channels inside
our for loop without blocking.

## II. Design Philosophy ##

When developing a program with this in mind, we'd like to solve the problem by
creating a concurrent design which *may* be parallelized; we want to think about
breaking the problem down into independent components which are separate but can
be composed together.  This allows us to take a concurrent design and refactor 
the independent pieces into different configurations to solve our problem.

Our components:

### main.go ###

In here, we'll setup a dispatcher and queue jobs to be done, then wait until 
the work is finished (since this is an embedded job queue, our primary goroutine waits.)

This is our example application which uses the embedded job queue.  It will 
import the package, create some dummy work that emulates a long running job, and
document our package so others can see how to use it or create their own.

### dispatcher.go ###

The dispatcher will handle most of the work in our library, it should perform
the following:

    a. Accept jobs in the form of an anonymous wrapped function, from main.go
    b. Maintain a list of jobs.
    c. Accept a request for a number of workers to perform the work.
    d. Create workers.
    e. Maintain a list of workers.
    f. Provide work to the workers.

### worker.go ###

The worker will concern itself with waiting for jobs, processing them and 
responding to signals.  It should perform the following:

    a. Register itself with the dispatcher's channel of workers.
    b. Process jobs from the job queue channel.
    c. Respond to orders from the dispatcher to halt or get job status.
    d. Set properties on a job (start time, end time, result).

### job.go ###

Here we define a structure for jobs: a unique ID, start and end run time, 
completed, (running is inferred through start time), and result.

## III. Example ##

Before we start creating a package to manage jobs, let's prototype one.  The 
following example allows us to submit two separate jobs and wait for them to
finish.  We're making liberal use of channels to communicate status to the 
dispatcher, which listens for submitted jobs and status reports of jobs and
workers.

{{< highlight go "linenos=inline" >}}
package main

import (
	"fmt"
	"time"
)

type Job struct {
	ID int
	F func() error
}

type Worker struct {
	ID int
	jobs chan *Job
	dispatchStatus chan *DispatchStatus
	Quit chan bool
}

func CreateNewWorker(id int, workerQueue chan *Worker, jobQueue chan *Job, dStatus chan *DispatchStatus) *Worker {
	w := &Worker{
		ID: id, 
		jobs: jobQueue,
		dispatchStatus: dStatus,
	}

	go func() { workerQueue <- w }()
	return w
}

func (w *Worker) Start() {
	go func() {
		for {
			select {
			case job := <- w.jobs:
				fmt.Printf("Worker[%d] executing job[%d].\n", w.ID, job.ID)
				job.F()
				w.dispatchStatus <- &DispatchStatus{Type: "worker", ID: w.ID, Status: "quit"}
				w.Quit <- true
			case <- w.Quit:
				return
			}
		}
	}()
}

type DispatchStatus struct {
	Type string
	ID int
	Status string
}

type Dispatcher struct {
	jobCounter int                      // internal counter for number of jobs
	jobQueue chan *Job                  // channel of jobs submitted by main()
	dispatchStatus chan *DispatchStatus // channel for job/worker status reports
	workQueue chan *Job                 // channel of work dispatched
	workerQueue chan *Worker            // channel of workers
}

func CreateNewDispatcher() *Dispatcher {
	d := &Dispatcher{
		jobCounter: 0,
		jobQueue: make(chan *Job),
		dispatchStatus: make(chan *DispatchStatus),
		workQueue: make(chan *Job),
		workerQueue: make(chan *Worker),
	}
	return d
}

type JobExecutable func() error

func (d *Dispatcher) Start(numWorkers int) {
	// Create numWorkers:
	for i := 0; i<numWorkers; i++ {
		worker := CreateNewWorker(i, d.workerQueue, d.workQueue, d.dispatchStatus)
		worker.Start()
	}

	// wait for work to be added then pass it off.
	go func() { 
		for {
			select {
			case job := <- d.jobQueue:
				fmt.Printf("Got a job in the queue to dispatch: %d\n", job.ID)
				// Sending it off;
				d.workQueue <- job
			case ds := <- d.dispatchStatus:
				fmt.Printf("Got a dispatch status:\n\tType[%s] - ID[%d] - Status[%s]\n", ds.Type, ds.ID, ds.Status)
				if ds.Type == "worker" {
					if ds.Status == "quit" {
						d.jobCounter--
					}
				}
			}
		}
	}()
}

func (d *Dispatcher) AddJob(je JobExecutable) {
	j := &Job{ID: d.jobCounter, F: je}
	go func() { d.jobQueue <- j }()
	d.jobCounter++
	fmt.Printf("jobCounter is now: %d\n", d.jobCounter)
}

func (d *Dispatcher) Finished() bool {
	if d.jobCounter < 1 {
		return true
	} else {
		return false
	}
}

func main() {
	job1 := func() error {
		fmt.Printf("job1: performing work.\n")
		time.Sleep(2 * time.Second)
		fmt.Printf("Work done.\n")
		return nil
	}

	job2 := func() error {
		fmt.Printf("job2: performing work.\n")
		time.Sleep(4 * time.Second)
		fmt.Printf("Work done.\n")
		return nil
	}

	d := CreateNewDispatcher()
	d.AddJob(job1)
	d.AddJob(job2)
	d.Start(2)

	for {
		if d.Finished() {
			fmt.Printf("All jobs finished.\n")
			break
		}
	}
}
{{< / highlight >}}

The above when run produced the following output:

```
$ ./ex2
jobCounter is now: 1
jobCounter is now: 2
Got a job in the queue to dispatch: 0
Got a job in the queue to dispatch: 1
Worker[0] executing job[1].
job2: performing work.
Worker[1] executing job[0].
job1: performing work.
Work done.
Got a dispatch status:
	Type[worker] - ID[1] - Status[quit]
Work done.
Got a dispatch status:
	Type[worker] - ID[0] - Status[quit]
All jobs finished.
```

## IV. Conclusion ##

So we add jobs, create two workers, let them pick off work and execute the job,
then report that they're done.  The next step after this prototype would be
to handle an arbitrary number of jobs with an arbitrary number of workers. We'll
have the dispatcher handle sending a quit command to a worker to halt them, 
individually or all at once, and track the status and run time of jobs.
