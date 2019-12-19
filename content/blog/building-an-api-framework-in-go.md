---
title: "Building an API Framework in Go"
thumbnail: "/images/blog/4/thumbnail.png"
date: 2019-12-02
tags: ["golang", "library", "web", "api", "microservices"]
draft: false
id: 4
---

![head](/images/blog/4/head.png)

In the article [Building a Form Validation and Processing API in Go](/blog/building-a-form-validation-and-processing-api-in-go/), we defined how a
microservice API can be created to handle validating and processing form data.
In the further work section we noted:

"We can generalize our manager service and have it handle the boiler plate work..." 
The goal being that we want to simplify the code in zefram around some 
conventions. As we continue to express our opinion in code, we're making 
decisions that lead toward a framework and set of conventions we'll use in other
projects.  

We've removed the manager package and abstracted that into a new package, 
[ls-governor](https://github.com/lakesite/ls-governor/), which provides a 
comprehensive API:

{{< highlight go "linenos=inline" >}}
// API contains a fibre web service and our management service.
type API struct {
	WebService *fibre.WebService
	ManagerService *ManagerService
}
{{< / highlight >}}

We've also abstracted out our manager service, which references a new package
for handling datastore configurations and connections, [ls-superbase](https://github.com/lakesite/ls-superbase/), which favors gorm for our object
relational mapper.  

{{< highlight go "linenos=inline" >}}
// ManagerService contains the configuration settings required to manage the api.
type ManagerService struct {
	Config   *toml.Tree
	DBConfig map[string]*superbase.DBConfig
}
{{< / highlight >}}

Now we've far less complexity to manage in Zefram.  Our root command for setting
up and running Zefram is now:

cmd/root.go:
{{< highlight go "linenos=inline" >}}
rootCmd = &cobra.Command{
    Use:   "zefram -c [config.toml]",
    Short: "run zefram",
    Long:  `run zefram with config.toml as a daemon`,
    Run: func(cmd *cobra.Command, args []string) {
        gms := &governor.ManagerService{}
        if config == "" {
            config = "config.toml"
        }

        // setup manager and create api
        gms.InitManager(config)
        gms.InitDatastore("zefram")
        gapi := gms.CreateAPI("zefram")

        // bridge logic
        model.Migrate(gapi, "zefram")
        api.SetupRoutes(gapi)

        // now daemonize the api
        gms.Daemonize(gapi)
    },
}
{{< / highlight >}}

We've created a new package, [ls-governor](https://github.com/lakesite/ls-governor/), 
which handles our manager service, and integrating it with our api and models.  
This means zefram's new api package consists of two files.  A routes definition 
and related handlers.

To allow governor to bridge our route logic with handlers that have access to
the full governor API, we'll wrap them in a function [ls-fibre](https://github.com/lakesite/ls-fibre/)'s 
Mux Router expects.

pkg/api/routes.go:
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
		"/zefram/api/v1/contact/", 
		func(w http.ResponseWriter, r *http.Request) {
			ContactHandler(w, r, gapi)
		},
	).Methods("POST")
}
{{< / highlight >}}

Now we can write handlers with full access to governor's API, which includes a
datastore and app configuration settings.

pkg/api/handlers.go:
{{< highlight go "linenos=inline" >}}
// handlers contains the handlers to manage API endpoints
package api

import (
	"fmt"
	"net/http"

	valid "github.com/asaskevich/govalidator"
	"github.com/gorilla/schema"
	"github.com/lakesite/ls-mail"
	"github.com/lakesite/ls-governor"

	"github.com/lakesite/zefram/pkg/models"
)

// ContactHandler handles POST data for a Contact.
// The handler is wrapped to provide convenience access to a governor.API
func ContactHandler(w http.ResponseWriter, r *http.Request, gapi *governor.API) {
	// parse the form
	err := r.ParseForm()
	if err != nil {
		gapi.WebService.JsonStatusResponse(w, "Error parsing form data.", http.StatusBadRequest)
		return
	}

	// create a new contact
	c := new(model.Contact)

	// using a new decoder, decode the form and bind it to the contact
	decoder := schema.NewDecoder()
	decoder.Decode(c, r.Form)

	// validate the structure:
	_, err = valid.ValidateStruct(c)
	if err != nil {
		gapi.WebService.JsonStatusResponse(w, fmt.Sprintf("Error: %s", err.Error()), http.StatusBadRequest)
		return
	}

	// insert the contact structure
	if dbc := gapi.ManagerService.DBConfig["zefram"].Connection.Create(c); dbc.Error != nil {
		gapi.WebService.JsonStatusResponse(w, fmt.Sprintf("Error: %s", dbc.Error.Error()), http.StatusInternalServerError)
		return
	}

	// send an email
	mailfrom, _ := gapi.ManagerService.GetAppProperty("zefram", "mailfrom")
	mailto, _ := gapi.ManagerService.GetAppProperty("zefram", "mailto")
	subject := "Contact from: " + c.Email
	body := "First Name: " + c.FirstName + "\nLast Name: " + c.LastName + "\nMessage: \n\n" + c.Message
	err = mail.LocalhostSendMail(mailfrom, mailto, subject, body)

	if err != nil {
		gapi.WebService.JsonStatusResponse(w, fmt.Sprintf("Error: %s", err.Error()), http.StatusInternalServerError)
		return
	}

	// Return StatusOK with Contact made:
	gapi.WebService.JsonStatusResponse(w, "Contact made", http.StatusOK)
}
{{< / highlight >}}

Further, we can push the business logic in the handler out into our model when
our project grows in complexity.  Our model convention now has two files, 
contact.go and migrations.go.  We haven't changed anything in our contact.go 
file, which has our contact model.  

Our migrations.go file uses the governor API and it's datastore connection to
handle a gorm migration:

pkg/models/migrations.go:
{{< highlight go "linenos=inline" >}}
package model

import (
	"errors"
	"fmt"

	"github.com/lakesite/ls-governor"
)

// Migrate takes a governor API and app name and migrates models, returns error
func Migrate(gapi *governor.API, app string) error {
	if gapi == nil {
		return errors.New("Migrate: Governor API is not initialized.")
	}

	if app == "" {
		return errors.New("Migrate: App name cannot be empty.")
	}

	dbc := gapi.ManagerService.DBConfig[app]
	
	if dbc == nil {
		return fmt.Errorf("Migrate: Database configuration for '%s' does not exist.", app)
	}

	if dbc.Connection == nil {
		return fmt.Errorf("Migrate: Database connection for '%s' does not exist.", app)
	}

	dbc.Connection.AutoMigrate(&Contact{})
	return nil
}
{{< / highlight >}}

We call Migrate in cmd/root.go, and for now, use gorm's auto migration feature.

Using governor, our new project layout for zefram looks like this:

```
    cmd/root.go
    docs
    pkg/api
        handlers.go
        routes.go
    pkg/models
        contact.go
        migrations.go
    config.toml
    LICENSE.md
    main.go
    Makefile
    README.md
```

Logic simplified.