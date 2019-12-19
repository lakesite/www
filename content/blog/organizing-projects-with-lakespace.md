---
title: "Organizing Projects with Lakespace"
thumbnail: "/images/blog/5/thumbnail.png"
date: 2019-12-03
tags: ["npm", "javascript", "development", "utility"]
draft: false
id: 5
---

![head](/images/blog/5/head.png)

When researching, designing and developing software, our goal is to organize and
reduce complexity.  When you're working on more than one project, it's nice to 
be able to save the state of your workflow when you need to switch context to 
another project.

[Lakespace](https://github.com/lakesite/lakespace) was designed for this purpose,
but can be utilized for broader workspace goals (running certain commands in 
terminal windows, resizing your browser and terminal windows, moving them to
the appropriate desktop - for example.)  It's a workspace utility manager 
designed to bring up the editor, browser, and associated information you need to
get right back to editing your code and getting things done.

Using a simple config.toml file, you can define a project workspace comprised of
an IDE, browser windows and tabs, and command line terminal windows.  

Lakespace is available on npm as a [package](https://www.npmjs.com/package/lakespace).
Installation and use is straight forward:

```
# npm install -g lakespace
```

Lakespace uses [toml](https://github.com/toml-lang/toml) - Tom's obvious markup
language, for configuration.  Once you have lakespace installed, go into your 
project and create a lakespace.toml file:

```
[example_window_name]
browser = "firefox"
tabs = [
  "https://ajduncan.org/",
  "https://lakesite.net/",
  "https://duncaningram.com/"
]

[ide]
editor = "codium"
project_directory = "."
```
Then you can type:

                $ lakespace

Which will open a new firefox browser window with three tabs, and open the 
[codium](https://github.com/VSCodium/vscodium) editor for the current directory.

Lakespace is actively developed for use with Ubuntu Linux, with gnome terminal.
A new release will be out that supports OSX and iTerm2.  Windows 10 support may
be limited, depending on command line options or scripting for the new 
[Windows Terminal](https://github.com/microsoft/terminal).

A more complex example of a lakespace configuration:

```
[lakespace]
desktop = "3"

[applicant]
browser = 'firefox'
tabs = [
	"https://github.com/lakesite/lakespace",
	"https://github.com/mozilla/multi-account-containers/issues/319",
	"https://github.com/toml-lang/toml",
	"https://github.com/ogt/valid-url",
	"https://www.npmjs.com/package/lakespace"
]

[ide]
editor = 'atom'
path = ''

[terminal_1]
working_directory = "/home/andy/projects"
command_1 = "ls -la"
command_2 = "cd ~/projects; ls -lah"
offset = "500,500"

[terminal_2]
desktop = "4"
working_directory = "/var/www/development/"
command = "ls"
offset = "2048,250"
```

When we run Lakespace the desktop will switch to the associated "3" desktop.  
Firefox will then be launched with five tabs.  The Atom editor will run. Two
terminal windows will start.  The first will have two tabs with a working 
directory of '/home/andy/projects', the first tab will issue the command 'ls -la',
the second tab will issue the command 'cd ~/projects; ls -lah".  The terminal
window itself will have an offset of 500, 500 in the desktop.

The next terminal will load under the "4" desktop, with one tab, using the 
command ls, with an offset of 2048,250.

This layout represents three physical screens across 9 desktops.

With this utility, you can have several projects allocated to different virtual
desktops, each with a state you can restore on boot, allowing you to open the
associated URLs for your project, run commands (cd project && vagrant up), and
generally control the workspace.  