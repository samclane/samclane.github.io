---
layout: post
comments: true
published: true
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

3. Getting OpenCV

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

With `virtualenvwrapper` in your environment, it's finally time to make the environment.

```shell
mkvirtualenv cv -p python3
```

This creates a Python 3.7 environment called `cv`. Verify `cv` works with:

```shell
workon cv
```

If this worked correctly, the shell prompt should be prefixed with a `(cv)`.

There's only 1 Python requirement for OpenCV, `numpy`:

```shell
pip install numpy
```

We now have everything we need to build OpenCV from source.

4. Building OpenCV

Now for the tense part. We're going to build `opencv` from source. 

Change directory to `opencv` and make a `build` directory:

```shell
cd ~/opencv`
mkdir build
cd build
```

Because we're using `virtualenvwrapper`, none of the correct Python files or libraries are in the PATH. Therefore, we're going to have to hold `cmake`'s hand for each and every Python resource. 

### If you gain nothing from this tutorial, at least see this

```shell
cmake -D CMAKE_BUILD_TYPE=RELEASE     -D CMAKE_INSTALL_PREFIX=/usr/local     -D OPENCV_EXTRA_MODULES_PATH=~/opencv_contrib/modules     -D ENABLE_NEON=ON     -D ENABLE_VFPV3=ON     -D BUILD_TESTS=OFF     -D OPENCV_ENABLE_NONFREE=ON     -D INSTALL_PYTHON_EXAMPLES=OFF     -D BUILD_EXAMPLES=OFF -DPYTHON3_EXECUTABLE=/home/pi/.virtualenvs/cv/bin/python -DPYTHON3_INCLUDE_DIR=/home/pi/.virtualenvs/cv/include/python3.7m -DPYTHON3_LIBRARY=/home/pi/.pyenv/versions/3.7.2/lib/libpython3.so -D BUILD_opencv_python3=yes ..
```

This will take a few minutes to complete. Once finished, inspect the output. Ensure that `cv` Python3 interpreter and `numpy` package were both discovered, with the correct paths. The next step is very time-consuming, and it would be a waste if you were to compile for the wrong version.

Before we build, we must increase the SWAP size of the Raspberry Pi. This can be accomplished via editing some text files:

```shell
sudo nano /etc/dphys-swapfile
```

```shell
# set size to absolute value, leaving empty (default) then uses computed value
#   you most likely don't want this, unless you have an special disk situation
# CONF_SWAPSIZE=100
CONF_SWAPSIZE=2048
```

<sub>Swapsize was set from 100MB to 2048MB</sub>

This is a requirement; if this is not completed your Pi will most likely freeze during compilation. Note that having an increased SWAP size burns out your SD card faster (as there are more I/O operations). Remember to change the `CONF_SWAPSIZE` back down to `100` (MB).

To put the swap changes into effect, restart the `dphys-swap` service:

```shell
sudo /etc/init.d/dphys-swapfile stop
sudo /etc/init.d/dphys-swapfile start
```

Finally, after all the teasing, we're here: building OpenCV4. The commmand is simple, 

```shell
make -j4
```

Where `-j4` says to use all 4 cores of the Pi. However, this can lead to race conditions, and I've personally had bad luck with it, so you can just enter `make` if you like. 

Either way, this going to take a couple of hours. Check in every so often to make sure the device isn't hanging. Once the process hits 100% and sends you back to the shell, you've officially copiled OpenCV4 for a Raspberry Pi's ARM processor. Pat yourself on the back!

5. Installing OpenCV

To install the binaries, simply enter the following:

```shell
sudo make install
sudo ldconfig
```

(Note: Remember to reset your SWAPSIZE to 100MB and restart the SWAP service)

6. Link OpenCV to Python

OpenCV has been installed, but our Python installation doesn't know about it. We need to link the library into its namespace in `site-packages`. 

```shell
cd ~/.virtualenvs/cv/lib/python3.7/site-packages/
ln -s /usr/local/python/cv2/python-3.7/cv2.cpython-37m-arm-linux-gnueabihf.so cv2.so
cd ~
```

7. Test your installation

To ensure you've done everything right, give a trial by fire: activate the `cv` environment and attempt to `import cv2`

```shell
workon cv
python
```
```python
>>> import cv2
>>> cv2.__version__
'4.0.0'
>>> exit()
```

You're now running the latest version of Python with the most powerful Computer Vision library publicly available. Go blur some images!

---

This tutorial borrows heavily from [Adrian Rosebrock's Amazing OpenCV + Raspberry Pi Tutorial](https://www.pyimagesearch.com/2018/09/26/install-opencv-4-on-your-raspberry-pi/).

I also used a [pyenv tutorial for a Discord Bot](https://docs.discord.red/en/v3-develop/install_linux_mac.html?highlight=stretch#install-python-pyenv) as a way to install 3.7.

