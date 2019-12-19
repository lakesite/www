---
title: "Building a Form Validation and Processing API in Go"
thumbnail: "/images/blog/3/thumbnail.png"
date: 2019-12-01
tags: ["golang", "library", "web", "api", "microservices"]
draft: false
id: 3
---

![head](/images/blog/3/head.png)

At Lakesite.Net, we love using [JAMStack](https://jamstack.org) (Javascript APIs
and Markup) because it simplifies the developer experience, allows for a
performant and scalable site, at an overall lower cost.  Business web sites
typically require marketing "who we are," "what we do," "why should you pay us
to do it?" style information.  This doesn't require any programming, but does
require markup, style and content.  Removing complexity is desirable.  However,
you'll likely want to process contact forms.

There are APIs and services which will process contact forms, such as Mailchimp.
So, other than developing a micro-service API for fun, why should we do this?

1. You want to run your own API for a JAMStack without relying on others.
2. You want to retain a copy of contact information without relying on others.
3. You presume to be more responsible with your customer's data than a third party.
4. You want to use an open source solution.
5. You're interested in providing your own API service.

Let's create our own API.  It should provide an endpoint to submit contact form
data.  The endpoint will validate form fields, save the data to a database, and
e-mail the contact information. We want to accept a confirmation or thanks page
to redirect to after valid form submission, and receive messages if form fields
are invalid, so we can let users know if, for example, their email is invalid.

## I. Dependencies ##

The following dependencies will be used in our project to simplify the process.
We want to minimize the amount of code we actually write and keep our API clean,
concise, well documented and tested.

1. The Gorilla web toolkit's [schema](http://www.gorillatoolkit.org/pkg/schema)
package will handle filling our contact structure with form values.
2. Alex Saskevich's [govalidator](https://github.com/asaskevich/govalidator)
will handle validating the contact structure.  
3. [ls-fibre](https://github.com/lakesite/ls-fibre)
will handle the web API service.
4. [ls-config](https://github.com/lakesite/ls-config) will handle configuration
settings.
5. [Cobra commander](github.com/spf13/cobra) will be used to handle command line
arguments (path to configuration file, versioning, etc).
6. Thomas Pelletier's [go-toml](https://github.com/pelletier/go-toml) will be
used to process our configuration file.

## II. Structure ##

Let's call our project *zefram*, for Zefram Cochrane, from Star Trek's First
Contact.  Our eventual file structure will look like the following:

```
zefram/
    bin/          - Executable and bundled files from 'make'.
    cmd/root.go   - Cobra commander command line handler.
    docs/         - Documentation.
    pkg/
      api/        - API routes and handlers.
      mail/       - Mail convenience package.
      manager/    - Main manager package.
      models/     - Database related structures and models.
    config.toml   - Application configuration.
    main.go       - Application entry point.
    Makefile      - Makefile to build app
    README.md     - Basic README with references to documentation and usage.
```

The full [source](https://github.com/lakesite/zefram/) is available and MIT
licensed.  We'll provide an overview of how things fit together in a general way,
and then review how we perform validation, save the data, and e-mail the contact.

## III. Boilerplate ##

Let's dive right in with a summary of how we get to the actual work.

zefram/main.go references cmd.Execute:

{{< highlight go "linenos=inline" >}}
func main() {
	cmd.Execute()
}
{{< / highlight >}}

The root command for the project is defined as such:

zefram/cmd/root.go:

{{< highlight go "linenos=inline" >}}
rootCmd = &cobra.Command{
  Use:   "zefram -c [config.toml]",
  Short: "run zefram",
  Long:  `run zefram with config.toml as a daemon`,
  Run: func(cmd *cobra.Command, args []string) {
    ms := &manager.ManagerService{}
    if config == "" {
      config = "config.toml"
    }
    ms.Init(config)
    ms.Daemonize()
  },
}
{{< / highlight >}}

We create a new manager service, which provides an interface between the command
line and the web service API.  We require a configuration file, by default
config.toml, then initialize the application with any configuration options
we've provided.  We'll cover some of the initialization logic later.  Finally we
tell the manager service to daemonize the app so it runs continuously and
processes requests.

The manager's Daemonize function determines the host and IP for the service to
listen on.  We create a new API instance with an ls-fibre WebService, an app
configuration, and a database configuration (and connection).  Next we setup
routes for our application.  Finally, we run the web service.

zefram/pkg/manager/manager.go:

{{< highlight go "linenos=inline" >}}
// Daemonize sets up the web service and defines routes for the API.
func (ms *ManagerService) Daemonize() {
        address := config.Getenv("ZEFRAM_HOST", "127.0.0.1") + ":" + config.Getenv("ZEFRAM_PORT", "7990")
				ws := service.NewWebService("zefram", address)
				api := api.NewAPI(
					ws,           // web service
					ms.Config,    // app configuration
					ms.DBConfig,  // database connection and configuration
				)
        api.SetupRoutes()
        api.WebService.RunWebServer()
}
{{< / highlight >}}


The api package has a function to setup routes.  The route we care about will
be posting data to /contact, which we define:

zefram/pkg/api/routes.go:

{{< highlight go "linenos=inline" >}}
// SetupRoutes defines and associates routes to handlers.
func (api *API) SetupRoutes() {
        api.WebService.Router.HandleFunc("/zefram/api/v1/contact/", api.ContactHandler).Methods("POST")
}
{{< / highlight >}}

Our ContactHandler should accept POST requests which contain:

  1. First name
  2. Last name
  3. Email
  4. Message
  5. Add to mailing list?

Email should be required and must be a valid email address.  The other fields
are optional and require no further validation.  We should return a json
response for invalid email or missing email field information, otherwise, we
should return a status 'contact made'.  In either case we should return 200 OK,
unless we have an internal error.

Our contact model handles the data:

zefram/pkg/models/contact.go

{{< highlight go "linenos=inline" >}}
// Contact holds the minimal data for a web contact.
type Contact struct {
  gorm.Model
  FirstName string
  LastName string
  Email string `gorm:"type:varchar(100);"valid:"email"`
  Message string
  AddMailing bool `gorm:"default:false;"`
}
{{< / highlight >}}

We also initialize our DB (presently using sqlite3) and run the necessary
migration to create a contact table.  The Migrate command is called before we
Daemonize, when we call Init with our app configuration.  Init parses our config
file and initializes our app, zefram, which populates a DBConfig structure and
calls Migrate:

zefram/pkg/models/dbconfig.go:

{{< highlight go "linenos=inline" >}}
func (db *DBConfig) Init() {
	if db.Driver == "sqlite3" {
		db.Connection, _ = gorm.Open("sqlite3", db.Path)
	} else {
		// handle connections with other drivers
	}
}

func (db *DBConfig) Migrate() {
	if db.Connection != nil {
		db.Connection.AutoMigrate(&Contact{})
	}
}
{{< / highlight >}}

For now we'll focus on a simple sqlite3 database which only requires a driver
type of sqlite3 and a path to the database.  Our config.toml is simple:

zefram/config.toml:

{{< highlight go "linenos=inline" >}}
[zefram]
apikey   = "secretkey"
dbdriver = "sqlite3"
dbpath   = "zefram.db"
mailto   = "andy"
mailfrom = "zefram@lakesite.net"
{{< / highlight >}}

## IV. Handling Contacts ##

Our API's ContactHandler needs to do a few things.

First, we parse the form data.  If we can't, we return an error with a Bad
Request status code.

zefram/pkg/api/api.go: ContactHandler breakdown

```
// parse the form
err := r.ParseForm()
if err != nil {
  api.WebService.JsonStatusResponse(w, "Error parsing form data.", http.StatusBadRequest)
}
```

Now we need to map the form data into our contact structure, using Gorilla's
scheme library and decoder:

```
// create a new contact
c := new(model.Contact)

// using a new decoder, decode the form and bind it to the contact
decoder := schema.NewDecoder()
decoder.Decode(c, r.Form)
```

Next we need to validate the structure, using govalidator.  The only field we
really care about being proper is the contact email address.

```
// validate the structure:
_, err = valid.ValidateStruct(c)
if err != nil {
  api.WebService.JsonStatusResponse(w, fmt.Sprintf("Error: %s", err.Error()), http.StatusBadRequest)
  return
}
```

With our form data bound to a structure and validated, we can proceed to save
the structure to our database using gorm.  At this point, if we do encounter any
errors, we should return an Internal Server Error response.

```
// insert the contact structure
if dbc := api.DBConfig["zefram"].Connection.Create(c); dbc.Error != nil {
  api.WebService.JsonStatusResponse(w, fmt.Sprintf("Error: %s", dbc.Error.Error()), http.StatusInternalServerError)
  return
}
```

Next we'll send an e-mail per our configuration.

```
// send an email
mailfrom, _ := api.GetAppProperty("zefram", "mailfrom")
mailto, _ := api.GetAppProperty("zefram", "mailto")
subject := "Contact from: " + c.Email
body := "First Name: " + c.FirstName + "\nLast Name: " + c.LastName + "\nMessage: \n\n" + c.Message
err = mail.LocalhostSendMail(mailfrom, mailto, subject, body)

if err != nil {
  api.WebService.JsonStatusResponse(w, fmt.Sprintf("Error: %s", err.Error()), http.StatusInternalServerError)
  return
}
```

Finally, we return http.StatusOK and a Contact made message:

```
// Return StatusOK with Contact made:
api.WebService.JsonStatusResponse(w, "Contact made", http.StatusOK)
```

zefram/pkg/api/api.go ContactHandler in full:

{{< highlight go "linenos=inline" >}}
// ContactHandler handles POST data for a Contact.
func (api *API) ContactHandler(w http.ResponseWriter, r *http.Request) {
	// parse the form
	err := r.ParseForm()
	if err != nil {
		api.WebService.JsonStatusResponse(w, "Error parsing form data.", http.StatusBadRequest)
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
		api.WebService.JsonStatusResponse(w, fmt.Sprintf("Error: %s", err.Error()), http.StatusBadRequest)
		return
	}

	// insert the contact structure
	if dbc := api.DBConfig["zefram"].Connection.Create(c); dbc.Error != nil {
		api.WebService.JsonStatusResponse(w, fmt.Sprintf("Error: %s", dbc.Error.Error()), http.StatusInternalServerError)
		return
	}

	// send an email
	mailfrom, _ := api.GetAppProperty("zefram", "mailfrom")
	mailto, _ := api.GetAppProperty("zefram", "mailto")
	subject := "Contact from: " + c.Email
	body := "First Name: " + c.FirstName + "\nLast Name: " + c.LastName + "\nMessage: \n\n" + c.Message
	err = mail.LocalhostSendMail(mailfrom, mailto, subject, body)

	if err != nil {
		api.WebService.JsonStatusResponse(w, fmt.Sprintf("Error: %s", err.Error()), http.StatusInternalServerError)
		return
	}

	// Return StatusOK with Contact made:
	api.WebService.JsonStatusResponse(w, "Contact made", http.StatusOK)
}
{{< / highlight >}}

## V. Testing Form Submission ##

Using curl, we can submit some data to our /contact/ endpoint to test:

        $ curl -d "email=test@notvalid&message=This%20is%20a%20test" -X POST http://localhost:7990/api/zefram/v1/contact/

We get an expected bad response, because test@notvalid is not a valid message:

        $ curl -d "email=test@notvalid&message=This%20is%20a%20test" -X POST http://localhost:7990/zefram/api/v1/contact/
        "Error: Email: test@notvalid does not validate as email"
        $

Now if we use a valid e-mail:

        $ curl -d "email=test@lakesite.net&message=This%20is%20a%20test" -X POST http://localhost:7990/zefram/api/v1/contact/
        "Contact made"
        $

Note: our config.toml has a mailto value of `andy`, which is not valid e-mail.  For
testing purposes, the validation for mailto was temporarily removed, so we can
deliver mail to a local user.

## VI. Further work ##

Zefram requires further work to support mysql and postgresql, and further API
endpoints to support export and import (in JSON format) of data.  Zefram should
handle different contact points, for example an inquiry about a specific product
or service.  

Zefram currently assumes you're sending mail from localhost, which is fine if
you have a system configured for this.  You may instead have an ephemeral cloud
instance which probably won't be accepted by Google or other mail services that
require, at a bare minimum, forward and reverse DNS to match.  So we'll want to
use another function to send authenticated mail.  This should be broken out into
another library.

Other code used in zefram should be further abstracted.  We really only care
about the implementation details in api.go, where we handle contact and
inquiries.

We can generalize our manager service and have it handle the boiler plate work
of:

  1. Accept a project configuration.
  2. Setup app configurations per project.
  2. Setup a web service configuration, using config and environment variables.
  3. Setup a database service if desired and run migrations (if required).
  4. Setup routes for the web service configuration using an API configuration.
  5. Run the web service if desired.

We can extend ls-config to handle toml configuration management, and possibly
hold database configuration options, but we need the connection and ORM
accessible.  We'll want to generalize pkg/api/utils.go and pkg/manager/utils.go
and include this in ls-config.

We should generalize the mail package as a convenience tool for us to use in
other projects.  We could also make use of ls-config here.

Finally, in both the code we abstract into libraries and in our project, we need
full test coverage.  It's OK to prototype an initial solution, but before we use
this in production, we must make sure the specification is covered by tests.
Afterward, any new feature we deliver should have a test developed first and
then the minimal solution will be implemented to ensure the test passes.
