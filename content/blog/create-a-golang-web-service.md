---
title: "Create a Golang Web Service"
thumbnail: "/images/blog/2/thumbnail.png" 
date: 2019-11-27
tags: ["golang", "library", "web", "tools"]
draft: false
id: 2
---

![head](/images/blog/2/head.png)

\
If you use Golang to create performant web APIs, you will find yourself adding
the same handlers for your new projects.  Don't forget to handle requests for
favicon.ico.  You'll probably want to handle generic requests for templated page
content, so you don't wind up with a lot of routes.  Most developers create
health checks for their service, for monitoring and metrics.  You may even needed
advanced handlers in services like proxying requests and accepting API keys.
If you're creating an API, you probably want to handle errors with some generic
json error codes.

\
Enter [ls-fibre](https://github.com/lakesite/ls-fibre/), a generic default web
service with basic handlers to get you up and going.

\
Per the [README.md](https://github.com/lakesite/README.md), let's take the
example main.go program code and modify that for an example application we'll
call carbon.  The code is as follows:

```
package main

import (
	"github.com/lakesite/ls-config/pkg/config"
	"github.com/lakesite/ls-fibre/pkg/service"
)

func main() {
	address := config.Getenv("MAIN_HOST", "127.0.0.1") + ":" + config.Getenv("MAIN_PORT", "8080")
	ws := service.NewWebService("main", address)
	ws.RunWebServer()
}
```

\
Next build the application, run it, and visit http://127.0.0.1:8080/

![build](/images/blog/2/1.png)

\
Next either visit the URL in your browser or use curl as such:

        $ curl http://127.0.0.1:8080/
        404 page not found

\
The result isn't what we want yet, but we're using a library that provides flexibility and a lot of basics out of the box.  The default handler for GET requests to /favicon.ico is handled:

        $ curl http://127.0.0.1:8080/favicon.ico
        data:image/x-icon;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQEAYAAABPYyMiAAAABmJLR0T///////8JWPfcAAAACXBIWXMAAABIAAAASABGyWs+AAAAF0lEQVRIx2NgGAWjYBSMglEwCkbBSAcACBAAAeaR9cIAAAAASUVORK5CYII=

\
We can also perform a health check and get a json response for monitoring purposes, and extend this as needed:

        $ curl http://127.0.0.1:8080/healthcheck
        {"alive": true}

\
Next, let's change the code in carbon.go a bit.  Change lines 9 and 10 from:

```
	address := config.Getenv("MAIN_HOST", "127.0.0.1") + ":" + config.Getenv("MAIN_PORT", "8080")
	ws := service.NewWebService("main", address)
```

To:

```
	address := config.Getenv("CARBON_HOST", "127.0.0.1") + ":" + config.Getenv("CARBON_PORT", "8080")
	ws := service.NewWebService("carbon", address)
```

\
We've changed the name of the web service to carbon, instead of main.  Your code
should look like the following:

![Atom Editor Code](/images/blog/2/2.png)

\
Now when you build carbon.go and run the application, it should say *carbon* serving on: 127.0.0.1:8080:

![Build 2](/images/blog/2/3.png)

\
Since we're using [ls-config](https://github.com/lakesite/ls-config) we can use environment variables to control what IP or port the application binds to and listens on:

![Environment variables](/images/blog/2/4.png)

\
Now we want to test the template response functionality in [ls-fibre](https://github.com/lakesite/ls-fibre/), but we need create a directory matching the web service name.  Create the following:

        $ mkdir -p web/carbon/page
        $ mkdir -p web/carbon/templates

\
Make sure web/carbon/page/index.html contains:

```
{{define "content"}}
<p>Hello from the carbon service.</p>
{{end}}
```

\
Also make sure web/carbon/templates/base.html contains:

```
{{define "base"}}
<!DOCTYPE html>
<html lang="en">
<head>
  <link href="data:image/x-icon;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQEAYAAABPYyMiAAAABmJLR0T///////8JWPfcAAAACXBIWXMAAABIAAAASABGyWs+AAAAF0lEQVRIx2NgGAWjYBSMglEwCkbBSAcACBAAAeaR9cIAAAAASUVORK5CYII=" rel="icon" type="image/x-icon">
</head>
<body>
  {{template "content" .}}
</body>
</html>
{{end}}
```

\
Now make sure carbon is running on the default port, 8080.  When you visit http://127.0.0.1:8080/ you should see "Hello from the carbon service."

![hello](/images/blog/2/5.png)
