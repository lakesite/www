---
title: "Lakespace"
date: 2018-12-02T20:01:48-05:00
draft: false
---

Lakespace is a workspace utility manager designed to bring up the
editor, browser and tabs, and associated information you need to get
right into editing your code and getting things done.

<!--more-->

Lakespace is available on [npm](https://www.npmjs.com/) as a [package](https://www.npmjs.com/package/lakespace).  Installation and use is pretty straight forward;

    ```
    # npm install -g lakespace
    ```

Lakespace uses [toml](https://github.com/toml-lang/toml) - Tom's obvious markup language, for configuration.  Once you have lakespace installed, create a folder and create a lakespace.toml file:

    $ cd /path/to/project
    $ cat <<EOF >>lakespace.toml

```
[example_window_name]
browser = "firefox"
tabs = [
  "https://ajduncan.org/",
  "https://lakesite.net/",
  "http://www.duncaningram.com/"

]

[ide]
editor = "atom"
project_directory = "."
EOF
```

Then you can type:

    $ lakespace

Which will open a new firefox browser window with three tabs, and open the [atom editor](https://ide.atom.io/) for the current directory.
