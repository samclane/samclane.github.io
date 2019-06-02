---
layout: post
comments: true
published: false
title: How to install OpenCV4 on a Raspberry Pi with Python 3.7
subtitle: If you're version-locked and have nowhere to turn to
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

```shell
CONFIGURE_OPTS="--enable-shared --enable-optimizations" pyenv install 3.7.2 -v
```

`--enable-shared` is critical, as it gives us access to to libpython3.so files later on.

This command took about 45 minutes on my Raspberry Pi 3B+. If you're pressed for time, remove the `CONFIGURE_OPTS=--enable-optimizations` from the command. However, this change will make Python ~10% slower.

Finally run the following to set Python 3.7 as your default interpreter:

```shell
pyenv global 3.7.2
Ensure this works by running python --version; you should get "Python 3.7.2" as output.
```

3. Installing OpenCV4

This is where the fun begins. Start by updating your system packages:

```shell
sudo apt-get update && sudo apt-get upgrade
```

Then install dev tools and CMake:

```shell
sudo apt-get install build-essential cmake unzip pkg-config
```

The following image and video libraries make the backbone of OpenCV's processing:

```shell
sudo apt-get install libjpeg-dev libpng-dev libtiff-dev
sudo apt-get install libavcodec-dev libavformat-dev libswscale-dev libv4l-dev
sudo apt-get install libxvidcore-dev libx264-dev
```

Optionally you can install the GTK Graphical-User-Interface (GUI) backend:

```shell
sudo apt-get install libgtk-3-dev
sudo apt-get install libcanberra-gtk*
```

Grab some numerical optimizations libraries:

```shell
sudo apt-get install libatlas-base-dev gfortran
```
And finally install the Python headers.

```shell
sudo apt-get install python3-dev
```

With all the pregame out of the way, we're finally ready to install OpenCV itself. We'll be downloading both `opencv` and `opencv_contrib` into our home directory.

```shell
cd ~
wget -O opencv.zip https://github.com/opencv/opencv/archive/4.0.0.zip
wget -O opencv_contrib.zip https://github.com/opencv/opencv_contrib/archive/4.0.0.zip
```

Unzip the files, and remove the version numbers for easier scripting.

```shell
unzip opencv.zip
unzip opencv_contrib.zip
mv opencv-4.0.0 opencv
mv opencv_contrib-4.0.0 opencv_contrib
```

Before we go installing this giant package into our one-and-only Python 3.7 installation, we should make a virtual environment for it first. This will keep our main installation clean and allow us to enable/disable OpenCV at will. We'll be using a virtualenvironment mangager called `virtualenvwrapper`. 

```shell
sudo pip install virtualenv virtualenvwrapper
```

We then need to edit our `~/.profile` to add the venv to our PATH. Using a text-editor, change the lines at the bottom of `~./profile`:

```shell
export WORKON_HOME=$HOME/.virtualenvs
export VIRTUALENVWRAPPER_PYTHON=$HOME/.pyenv/shims/python3
source /usr/local/bin/virtualenvwrapper.sh
```

It's **vital** that `VIRTUALENVWRAPPER_PYTHON` is set to the intepreter in `~/.pyenv`, as this is our Python 3.7 install. 

In order to put these changes into effect, source the profile:

```shell
source ~/.profile
```

