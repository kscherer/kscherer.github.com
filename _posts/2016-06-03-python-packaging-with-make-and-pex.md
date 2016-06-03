---
layout: post
title: "Python Packaging with make and pex"
category: Python
tags: [linux, python, pex, make]
---

As it often happens in the life of a professional programmer, a small
python script had grown into a large script and needed to be split
apart and properly packaged. Most of my experience with python had
been with small scripts. I had tried before to understand the python
packaging ecosystem but always got confused by the combinations of
tools and formats.

1. Python development tools like virtualenv and pip
1. Code distributed in eggs and/or wheels
1. Packages installed using easy_install and/or pip
1. Python packaging tools like setuptools and distutils

There seemed to be at least two different tools that did almost the
same thing, but neither had good documentation. I did find some decent
blog posts like [Open Sourcing a Python Project the Right Way][1] but
there were still workflow steps that I needed to figure out. In the
past I was able to avoid figuring it out, but this time was different
because my "small" script had grown to over 1000 lines of python and
there was no way to avoid it.

I had an informal set of requirements:

1. No root access should be required. Python supports local
   installation and virtualenv
1. Bootstrap a development environment quickly
1. The development setup should be self contained and not affect any
   other part of the machine

My research took me all over the web, but one of the most important
pieces of inspiration was this small post on
[Virtualenv and Makefiles][2]. I was also inspired by [Pex][3] which
provided a way to bundle all the python pieces together into a single
self extracting package.

It took a few days but I was able to combine make, mkvirtualenv, pip
and pex to implement a nice workflow. The [Makefile][4] will:

1. Install pip into `$HOME/.local/bin/pip`
1. Use local pip to install `virtualenv` and `virtualenvwrapper` into
   `$HOME/.local/bin`
1. Create a per project virtualenv for the project and install all the
   development dependencies like `pylint`, `flake8` and `pex`
1. Check if required development packages are installed. Some python
   packages have C extensions and require a compiler and header
   files.
1. Runs `python setup.py develop` which installs the package
   dependencies like yaml and redis. This step also adds the package
   to the virtualenv and can be used if development is spread across
   multiple git repositories.
1. Uses `python setup.py bdist_pex` to build the pex file

Other nice touches:

1. The source py files are dependencies on the pex package so editing
   a file causes the pex file to be rebuilt. Regex support in Make
   simplifies this step
1. Has `make help` which reads comments embedded in the Makefile to
   generate nice help output
1. Has `make clean` for easy cleanup
1. Each make step loads the proper virtualenv, so the developer does
   not even have to activate the virtualenv manually.

Some annoyances:

1. Pex does not pick up local python file changes unless I delete the
   egg file in the pex build dir.
1. To keep timestamps in order, sometimes it is necessary to touch
   certain files.
1. I had to create a .check file to prevent the system package
   checking from running every build
1. Dependent on Pypi being available, though pip does cache downloads
   locally

The last step was to integrate the pex file into a docker image. If
the package does not contain dependencies on system libraries, the
Alpine Linux Python docker images can be used as a base. Unfortunately
the python mesos.native packages I am using have dependencies on
libraries like libsaml and I could not use Alpine Linux. But I was
able to use the base Ubuntu image and only needed to install a few
libraries which made the image much smaller than before.

I noticed that pex file is unpacked into `PEX_ROOT` which is `$HOME`
by default. The last tweak I made was to ensure that `PEX_ROOT` was a
docker volume to avoid the overhead of writing to the union
filesystem. This isn't strictly necessary, but I try to work as if the
docker image is effectively read-only.

I have already reused this Makefile structure for other python
projects. I was pleasantly surprised when a colleague of mine was able
to clone the project and rebuild the docker image without any
intervention.

I am now able to focus on refactoring and developing the project. The
packaging part is solved in a clean way that can easily be shared with
others.

[1]: http://jeffknupp.com/blog/2013/08/16/open-sourcing-a-python-project-the-right-way/
[2]: http://blog.bottlepy.org/2012/07/16/virtualenv-and-makefiles.html
[3]: https://pex.readthedocs.io/en/stable/
[4]: https://github.com/kscherer/wraxl-scheduler/blob/master/Makefile
