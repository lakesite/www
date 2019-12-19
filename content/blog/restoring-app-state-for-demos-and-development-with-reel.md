---
title: "Restoring App State for Demos and Development with Reel"
thumbnail: "/images/blog/6/thumbnail.png"
date: 2019-12-04
tags: ["golang", "demo", "development", "reel", "utility", "database"]
draft: false
id: 6
---

![head](/images/blog/6/head.png)

If you provide a demo site for an application, you might have a cron job setup 
to periodically restore the application's state to some default.  This allows
prospects to try out your service by making changes that are periodically reset 
to a default state.

You may also find yourself with a similar workflow to us on some projects, where
we have a [feature branch](https://www.atlassian.com/git/tutorials/comparing-workflows/feature-branch-workflow) that requires us to periodically load a version of the
database, so we can test migrations and our feature before the feature is merged
into production.

## 🖭 reel 🖭 ##

[Reel](https://github.com/lakesite/reel) aims to simplify these two cases by 
providing the following features:

  - Restore an application's database state to a configurable default.
  - Restore an application to a preset configuration.
  - Provide a web interface for interactive resets for use with vagrant.
  - Define configuration reset points.
  - Provide per-instance configuration proxy,

    e.g. https://app.biz/demos/<ID>/

    Which is unique to a prospect, has a unique database backed instance of
    an application

Reel is currently
being used to restore an application's database state for development within a
[vagrant](https://vagrantup.com/) box, by exposing a port from the guest OS to
the host where we can tell the system to list data sources and load one.

## api endpoints ##

The API endpoints reel provides include:

```
    /reel/                             - Management requests for reel itself.
    /reel/api/v1/sources/              - List of database source files for
                                         default app.
    /reel/api/v1/sources/{app}         - List of database source files for
                                         {app}.
    /reel/api/v1/rewind/               - Rewind the default app.
    /reel/api/v1/rewind/{app}          - Rewind {app}
    /reel/api/v1/rewind/{app}/{source} - Rewind {app} with {source}
    /reel/api/v1/proxy/{app}           - Proxy requests for {app}
```

The proxy feature is still being developed.  Postgresql and MySQL are both
supported.

## example config ##

An example config file for reel:

```
[reel]
token="{!#}%p13^JTNG$H@R@$HTBgBNbx/Y$&N21"
default_app="app"

[app]
dbserver = "127.0.0.1"
dbport = "3306"
database = "example"
dbuser = "example"
dbpassword = "example"
dbdriver = "mysql"
dbsource = "./dumpfiles/example.sql"
dbsources = "./dumpfiles/"
proxy_dest = "https://localhost:8101/"

[app2]
dbserver = "127.0.0.1"
dbport = "5432"
database = "example"
dbuser = "example"
dbpassword = "example"
dbdriver = "postgres"
dbsource = "./dumpfiles/pgdump.sql"
```

Above we have two applications, app and app2.  App uses mysql and app2 uses
postgresql.  The database configuration for either application is specified,
along with a source file, ./dumpfiles/pgdump.sql and ./dumpfiles/example.sql.
Reel itself will eventually use a token or API key, and has a setting for a
default app, if no app is provided.

## Further Work ##

Reel was developed prior to [ls-governor](https://github.com/lakesite/ls-governor),
and should be further developed to use the same pattern we've established in
[zefram](https://github.com/lakesite/zefram) for projects.  This refactoring
should be done first.

Reel needs test coverage and better documentation.  Overall, we can take zefram 
and reel as golang projects that standardize what we want to provide for 
documentation.  Once we've done that, we'll want to use it as a template and 
guide line for future work.

After refactoring reel, adding better documentation, and full test coverage, we 
need to use a worker pool and handle processing the Rewind() function in our 
RewindHandler with a goroutine and jobs channel.  We can create a status 
websocket for client updates (if needed) but initially, it's OK to simply 
schedule a rewind job.  Currently, a larger database dump will process for a 
while before a result is returned, and we simply want to return 200OK for the 
job being scheduled, and a token or job ID to check for on a websocket.
