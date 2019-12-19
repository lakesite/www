 ---
title: "Using a Job Queue with Golang and Reel"
thumbnail: "/images/blog/7/thumbnail.png"
date: 2019-12-05
tags: ["golang", "channels", "reel", "utility"]
draft: false
id: 7
---

![head](/images/blog/7/head.png)

In the article, [Restoring App State for Demos and Development with Reel](/blog/restoring-app-state-for-demos-and-development-with-reel/), we introduced [reel](https://github.com/lakesite/reel), which among other features allows us to:

  - Restore an application's database state to a configurable default.
  - Provide a web interface for interactive resets for use with [vagrant](https://www.vagrantup.com/).

In the further work section, we noted the next steps included: 

1. Refactoring reel to use [ls-governor](https://github.com/lakesite/ls-governor)
2. Make sure API requests to rewind an app state are sent into a job queue, so 
we get a 200 OK response right away.  

Refactoring is a simple matter of following the pattern we established with 
[zefram](https://github.com/lakesite/zefram), starting with the interactive 
command in cmd/root.go:

{{< highlight go "linenos=inline" >}}
rootCmd = &cobra.Command{
    Use:   "reel -c [config.toml] -a [application name]",
    Short: "run reel with a config against an app.",
    Long:  `restore app state, for demos and development`,
    Run: func(cmd *cobra.Command, args []string) {
        gms := &governor.ManagerService{}
        gms.InitManager(config)
        gapi := gms.CreateAPI(application)
        reel.InitApp(application, gapi)
        reel.Rewind(application, "", gapi)
    },
}
{{< / highlight >}}

Running reel here doesn't require initializing a datastore, but we could.  We've
moved some business logic from the manager package into the reel package, which 
includes InitApp, PrintSources, GetSources, and Rewind.  The reel package also 
includes the specific logic for rewinding postgres and mysql databases.

We've also separated the api routes and handlers out into an api package per our
convention.  Our pkg/api/routes.go follows the convention:

{{< highlight go "linenos=inline" >}}
// routes handles setting up routes for our API
package api

import (
	"net/http"

	"github.com/lakesite/ls-governor"
)

// SetupRoutes defines and associates routes to handlers.
// Use a wrapper convention to pass a governor API to each handler.
func SetupRoutes(gapi *governor.API) {
	gapi.WebService.Router.HandleFunc(
		"/reel/api/v1/sources/", 
		func(w http.ResponseWriter, r *http.Request) {
			SourcesHandler(w, r, gapi)
		},
	)
	gapi.WebService.Router.HandleFunc(
		"/reel/api/v1/sources/{app}",
		func(w http.ResponseWriter, r *http.Request) {
			SourcesHandler(w, r, gapi)
		},
	)
...
{{< / highlight >}}

Then we define our RewindHandler and SourcesHandler in pkg/api/handlers.go.  

## Jobs ##

After refactoring, we need to handle submitting a job to a worker to process in
the background, using a [goroutine](https://tour.golang.org/concurrency/1) and a
[channel](https://tour.golang.org/concurrency/2) for communicating the work to
be processed.  

First we need to create a job package and create the most simple way of handling
jobs.  Our pkg/job/worker.go looks like this:

{{< highlight go "linenos=inline" >}}
package job

import (
	"fmt"

	"github.com/lakesite/ls-governor"

	"github.com/lakesite/reel/pkg/reel"
)

// Queue for reel jobs.
var ReelQueue = make(chan ReelJob)

// ReelJob contains the information required to submit a job
// to a worker for processing.
type ReelJob struct {
	App string
	Source string
	Gapi *governor.API
}

type ReelWorker struct {}

func (rw *ReelWorker) Start() {
	go func() {
		for {
			work := <- ReelQueue
			fmt.Printf("Received a work request for app: %s using source: %s\n", work.App, work.Source)
			reel.Rewind(work.App, work.Source, work.Gapi)
		}
	}()
}
{{< / highlight >}}

The ReelQueue channel will be used in our RewindHandler to submit a job, which
must be of a single type - in this case a ReelJob struct, which contains an app
name to rewind, a source to use, and a governor API.  Before we can go pushing
items into this ReelQueue, we'll have to start a worker in the manager command
we've refactored in cmd/root.go:

{{< highlight go "linenos=inline" >}}
managerCmd = &cobra.Command{
    Use:   "manager",
    Short: "Run the manager.",
    Long:  `Run the management interface.`,
    Run: func(cmd *cobra.Command, args []string) {
        // setup the job queue
        rw := &job.ReelWorker{}
        rw.Start()

        gms := &governor.ManagerService{}
        gms.InitManager(config)
        gms.InitDatastore("reel")
        gapi := gms.CreateAPI("reel")
        api.SetupRoutes(gapi)
        gms.Daemonize(gapi)
    },
}
{{< / highlight >}}

Our call to rw.Start() above will enter a goroutine, an anonymous function with
a for loop.  The routine will block and wait for work coming from the ReelQueue
channel, and when it receives it, make a call to the reel package's Rewind 
function.  This allows us submit the job in the RewindHandler and return status
ok, while the goroutine thread handles the work:

{{< highlight go "linenos=inline" >}}
		if reel.InitApp(app, gapi) {
			// submit a rewind job
			job.ReelQueue <- job.ReelJob{App: app, Source: source, Gapi: gapi}
			w.WriteHeader(http.StatusOK)
		} else {
			w.WriteHeader(http.StatusServiceUnavailable)
		}
{{< / highlight >}}

## Further Work ##

We could create a library for more robust handling of job queues, including the
ability to check on the status of a job, cancel a job, and get metrics about
the work being done.  Other programs certainly do this better, but we may choose
to create a library for governor to provide the convenience of handling jobs 
within our framework.

Should we do this, we'll want to look for both jobs and control signals.  We may
opt to have a separation between a dispatching routine and worker routine.  
