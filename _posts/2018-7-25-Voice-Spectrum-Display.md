---
layout: post
title: Create a low-resolution, real-time sound spectrum display
subtitle: Wow your friends by looking like a less-than-professional sound engineer
comments: true
published: false
---

I've made a [few](http://samclane.github.io/Python-Arduino-PyFirmata/) [posts](http://samclane.github.io/Arduino-Discord-Matrix/)
about Arduino and LED Matrices. In continuing that series, I wanted to make an implementation that used all 64 LEDs in
the matrix meaningfully, instead of as just a canvas to draw pictures to. This project originally came out of the Discord
Status display project, as I wanted to have the matrix display the Fourier Spectrum of my speech when the mic was enabled.
However, this proved to be too much at one time, and so I split the project into two parts. As I began to get the second
part working, I found that it might be too processor intensive to be used practically, so I kept to using static images
as my display for the project. However, I still did manage to get the spectral display to work, and so I've decided to
share it.

### Materials

* PC with Python 3 and the following packages:
  * `pyfirmata`
  * `matplotlib`
  * `numpy`
  * `pyaudio`
  * `scipy`
* Arduino Uno
* MAX7219 Dot Matrix Module ($1.31 USD on [AliExpress](https://www.aliexpress.com/item/MAX7219-Dot-led-matrix-module-MCU-control-LED-Display-module-for-Arduino/1875557666.html?spm=2114.search0104.3.135.78e9775aNry0Zj&ws_ab_test=searchweb0_0,searchweb201602_3_10152_10151_10065_10344_10130_10068_10324_10547_10342_10325_10546_10343_10340_10548_10341_10545_10696_10084_10083_10618_10307_10059_100031_524_10103_10624_10623_10622_10621_10620,searchweb201603_6,ppcSwitch_5&algo_expid=b4f623f2-b836-4fee-90bc-d80337113144-22&algo_pvid=b4f623f2-b836-4fee-90bc-d80337113144&transAbTest=ae803_2&priceBeautifyAB=0))

### The Code

The complete code can be found [here](https://github.com/samclane/SoundDisplay).

From the completed code repository, download the file `led_matrix.py`. This is the code that will allow us to talk to the
matrix. I already discussed how I wrote the driver, which can be found [here](http://samclane.github.io/Python-Arduino-PyFirmata/).

Now, create another file called `spectrum.py`. Here is where we will do all the dirty work. In order for this to all make
some kind of sense, we're not just going to grab data and FFT it willy-nilly. Rather, we're going to use `matplotlib` to
display several figures of data at various stages of processing, all in real time. First, we'll show the raw microphone
input (time-domain). Next, we'll show the data in the frequency domain



