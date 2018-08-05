---
layout:     post
title:      "Dummies guide to Python Virtualenv"
subtitle:   "Installing pip packages without breaking your system"
date:       2018-08-05
author:     "Michael Wadman"
catalog:    true
tags:
    - Python
---

# Overview

Recently I was trying to write a [script](https://gitlab.com/mwadman/artstation-downloader) that had dependencies on pip packages.
Now, I am fortunate enough to know that it's a bad idea to install pip packages globally [(especially](https://askubuntu.com/a/802594)[ using ](https://stackoverflow.com/a/15028735)[sudo)](https://stackoverflow.com/a/47268013), so I set about to find the best way to install those dependencies without breaking my system.

I'd heard about virtualenv (Virtual Environment) before, and after a quick google this seemed to be the solution I needed.  
Simply put, a virtualenv is a secluded environment (directory) where you install the dependencies your application requires, so that it can't interfere with the rest of your operating system (and its' possible seperate dependencies).

However, installing and using virtualenv seemed far too complicated, especially for a python newbie like myself.

This post is meant as a "Python Virtualenv for Dummies"; both so that I have a quick document to refer back to, and hopefully to help those as helpless as I am.

> Note that this post focuses on an installation of Ubuntu 18.04 with Python v2.7, but should hopefully be globally (both OS and Python version wise).

# Installation

First up is installing python and pip, both of which are done through apt:

```bash
$ sudo apt-get install python python-pip
```

Next is installing virtualenv using pip. This should be the only pip package you install globally (using sudo):

```bash
$ sudo pip install virtualenv
```

# Using Virtualenv

First we need to create an environment, this is done in the root directory of your project:

```bash
$ cd /path/to/project/dir
$ virtualenv venv
New python executable in venv/bin/python
Installing setuptools, pip, wheel...done.
```

The only argument supplied to the command is the subdirectory to create within the directory.  
The name "venv" is a common one, but you can name it whatever you want.

> Don't forget to add this directory to your .gitignore (or equivalent) if you're using SCM.

&nbsp; <!--- Used to add a double line break --->

Within this newly created directory are the python binaries that we can use to only affect our project.
For example, to install pip packages to our environment:

```bash
venv/bin/pip install $PackageName
```

Or to run a python program:

```bash
venv/bin/python $ProgramName.py
```

## Activating the environment

If you're like me and are too lazy to type in the path to the binaries folder every time you want to run a command, then there is the option to [activate](https://virtualenv.pypa.io/en/stable/userguide/#activate-script) the environment.  

```bash
$ source venv/bin/activate
```

This will change your shell prompt to show you that you're running python commands from within a venv as well.

Now we can run commands without referencing the binaries folder:

```bash
$ pip install $PackageName
```

&nbsp; <!--- Used to add a double line break --->

Behind the scenes, all this does is change the $PATH entry for the current shell session so that the first entry is the local virtualenv.

```bash
$ $PATH
bash: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:
$ source venv/bin/activate
$ $PATH
bash: /path/to/project/dir/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:
```

To reverse this, either leave the current session or run the deactivate script:

```bash
$ deactivate
```

# Conclusion

Hopefully this post was short and simple enough to understand, as I expect to refer back to this myself once I inevitably forget how to use virtualenv in a month.

## References:

[Virtualenv Official Documentation](https://virtualenv.pypa.io/en/stable/)
[A non-magical introduction to Pip and Virtualenv for Python beginners](https://www.dabapps.com/blog/introduction-to-pip-and-virtualenv-python/)

## Versions used:

Desktop Machine: *kubuntu-18.04.1*
Python: *Python 2.7.15rc1*
Virtualenv: 16.0.0
