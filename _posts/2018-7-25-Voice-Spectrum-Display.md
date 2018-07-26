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

From the completed code repository, download the file `led_matrix.py` and place it in a new folder. This is the code that will allow us to talk to the
matrix. I already discussed how I wrote the driver, which can be found [here](http://samclane.github.io/Python-Arduino-PyFirmata/).

Now, create another file called `spectrum.py`. Here is where we will do all the dirty work. In order for this to all make
some kind of sense, we're not just going to grab data and FFT it willy-nilly. Rather, we're going to use `matplotlib` to
display several figures of data at various stages of processing, all in real time. First, we'll show the raw microphone
input (time-domain). Next, we'll show the spectrum of the data in the frequency domain (after we apply a windowing function
to boost the human vocal frequencies). Finally, we'll graph the spectrum decimated and discretized to fit on the 8x8
display.

```python
import threading
from math import ceil
from time import sleep

import matplotlib.pyplot as plt
import numpy as np
import pyaudio
from pyfirmata import Arduino
from scipy.signal import butter, sosfiltfilt, decimate

import led_matrix



global keep_going # will use to start and stop the "animation"

class SpectrumPlotter:
    def __init__(self):
        self.init_plot()
        self.init_mic()
        self.init_matrix()

        self.annotation_list = []
```

We initialize the plot, microphone, then matrix (in that order). We also keep an annotation list, which we'll use to draw
numbers on the values of the final plot.

```python
    def init_plot(self):
        print('Initializing plot...')
        _, self.ax = plt.subplots(3)

        # Prepare the Plotting Environment with random starting values
        x1 = np.arange(10000)
        y1 = np.random.randn(10000)
        x2 = np.arange(8)
        y2 = np.random.randn(8)

        # Plot 0 is for raw audio data
        self.li, = self.ax[0].plot(x1, y1)
        self.ax[0].set_xlim(0, 1000)
        self.ax[0].set_ylim(-5000, 5000)
        self.ax[0].set_title("Raw Audio Signal")
        # Plot 1 is for the FFT of the audio
        self.li2, = self.ax[1].plot(x1, y1)
        self.ax[1].set_xlim(0, 2000)
        self.ax[1].set_ylim(0, 1000000)
        self.ax[1].set_title("Fast Fourier Transform")
        # Plot 2 is for the binned FFT
        self.li3 = self.ax[2].plot(x2, y2, 'ro')[0]  # for some reason, returned as a list of 1
        self.ax[2].set_xlim(0, 7)
        self.ax[2].set_ylim(0, 7)
        self.ax[2].set_title("8-Binned FFT")
        # Show the plot, but without blocking updates
        plt.pause(0.01)
        plt.tight_layout()
        print('Done')

    def init_mic(self):
        print('Initializing mic...')
        FORMAT = pyaudio.paInt16  # We use 16bit format per sample
        CHANNELS = 1
        self.RATE = 44100 // 2
        self.CHUNK = 1024  # 1024bytes of data red from a buffer

        self.audio = pyaudio.PyAudio()

        # start Recording
        self.stream = self.audio.open(format=FORMAT,
                                      channels=CHANNELS,
                                      rate=self.RATE,
                                      input=True)

        global keep_going
        keep_going = True
        print('Done')

    def init_matrix(self):
        print('Initializing Arduino...')
        self.board = Arduino('COM3')
        print('Done')
        self.matrix = led_matrix.LedMatrix(self.board)
        self.matrix.setup()
```

Now it's time to actually start seeing some data. We'll write a `process_data` function, which does most of the work.
It performs the bandpass windowing function on the data, isolating the voice frequency bands used in telephony (300Hz-3400Hz).
It then takes the real-valued Fast Fourier Transform of the filtered data, converting it to a frequency-domain spectrum.
Finally, it adds the data to the 3 plots, providing a discretized spectrum in the form of the 3rd plot.

However, we'll need to define the  bandpass-filter and discretize functions first.

```python
def butter_bandpass(lowcut, highcut, fs, order=5):
    nyq = 0.5 * fs
    low = lowcut / nyq
    high = highcut / nyq
    sos = butter(order, [low, high], analog=False, btype='bandpass', output='sos')
    return sos


def butter_bandpass_filter(data, lowcut, highcut, fs, order=5):
    sos = butter_bandpass(lowcut, highcut, fs, order=order)
    y = sosfiltfilt(sos, data)
    return y
```

We use `sosfilt`<sup><a href="#1">1</a></sup> as it can handle higher orders without giving strange negative results
(like the `ba` filter can).

---
<sup id="1">1</sup>`sos` stands for Second-order sections, which you can read about more [here](https://en.wikipedia.org/wiki/Digital_biquad_filter)