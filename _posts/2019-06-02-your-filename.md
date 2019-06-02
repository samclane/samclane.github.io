---
layout: post
comments: true
published: false
title: ''
subtitle: ''
---
1. Why?

I'm currently working on a project that needs Python 3.7 to function. To the chagrin of most developers, the Raspberry Pi only natively supports Python 2.7 and 3.5, so we're going to have to figure out how to get that on there first.

## Requirements
* Raspberry Pi 3B or 3B+
* Raspbian Stretch installed
* Expanded Filesystem (via raspi-config)

2. Getting Python 3.7 (and pyenv)

We're going to use pyenv as a tool to manage our Python installations, as Python 2.7 and 3.5 are dependencies of Raspbian Stretch.

Start by installing the following packages:

```shell
sudo apt install -y make build-essential libssl-dev zlib1g-dev libbz2-dev \
libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev \
xz-utils tk-dev libffi-dev liblzma-dev python3-openssl git unzip default-jre
```

Then download pyenv:

```shell
curl -L https:&#x2F;&#x2F;github.com&#x2F;pyenv&#x2F;pyenv-installer&#x2F;raw&#x2F;master&#x2F;bin&#x2F;pyenv-installer | bash
You may see a warning about pyenv not being in the load path. Follow the given instructions to fix it, then close and reopen your shell.
```

Since there's no distribution of Python 3.6+ for Raspberry Pi, we're going to need to compile Python 3.7 from source.

CONFIGURE_OPTS=&quot;--enable-shared --enable-optimizations&quot; pyenv install 3.7.2 -v
--enable-shared is critical, as it gives us access to to libpython3.so files later on.

This command took about 45 minutes on my Raspberry Pi 3B+. If you're pressed for time, remove the CONFIGURE_OPTS=--enable-optimizations from the command. However, this change will make Python ~10% slower.

Finally run the following to set Python 3.7 as your default interpreter:

pyenv global 3.7.2
Ensure this works by running python --version; you should get "Python 3.7.2" as output.

Installing OpenCV4
This is where the fun begins. Start by updating your system packages:

sudo apt-get update &amp;&amp; sudo apt-get upgrade
Then install dev tools and CMake:

sudo apt-get install build-essential cmake unzip pkg-config
The following image and video libraries make the backbone of OpenCV's processing:

sudo apt-get install libjpeg-dev libpng-dev libtiff-dev
sudo apt-get install libavcodec-dev libavformat-dev libswscale-dev libv4l-dev
sudo apt-get install libxvidcore-dev libx264-dev
Optionally you can install the GTK Graphical-User-Interface (GUI) backend:

sudo apt-get install libgtk-3-dev
sudo apt-get install libcanberra-gtk*
Grab some numerical optimizations libraries:

sudo apt-get install libatlas-base-dev gfortran
And finally install the Python headers.

sudo apt-get install python3-dev## A New Post

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
