---
layout: post
title: Discord Connection Status Monitor using an Arduino and MAX7219
subtitle: Never worry about leaving your mic open again!
comments: true
---

TL;DR: [Source code here.](https://github.com/samclane/DiscordMatrix)

[In my last post](http://samclane.github.io/Python-Arduino-PyFirmata/), I talked about how one could rewrite an Arduino
sketch in Python using PyFirmata, and how we could leverage that to create an easy interface to a MAX7219 LED Matrix. 
Here, I'm going to show you one of the applications of this interface (and what initially inspired me to get the matrix
working in the first place). 

I love Discord. It's like a mix between Slack and Ventrillo, and has been fundamental in allowing me to keep in touch 
with friends on the other side of the country. Writing [extensions](https://github.com/samclane/Snake-Cogs) for an 
existing Discord Bot framework ([Red-Bot](https://github.com/Cog-Creators/Red-DiscordBot)) was my first real foray into 
writing open-source software, and a very positive one at that. 

Since I am on Discord very often, I will sometimes forget that I'm still voice-connected and leave the computer. This 
usually results in a torrent of messages from my friends complaining about my dogs barking into my microphone while I'm away
(they've gotta hold down the fort somehow). However, it's become enough of an inconvenience that I decided to come up with a 
simple solution. While I have 2 monitors, like any self-respecting developer<sup>1</sup>, the actual Discord window 
still somehow finds itself at buried below a couple of browser windows within no-time. So, at a glance, I cannot tell if 
I'm A) in the server and B) muted/deafened. This can be additionally challenging while playing a game that can't be 
paused or alt-tabbed to check on Discord. My first attempt at tackling the problem was to try and [write my own custom
Discord overlay](https://github.com/samclane/MinimalDiscordOverlay). Discord has an official overlay, but it only works 
in games, and only certain ones at that. Plus, it's really bloated, showing everyone in your voice server and taking up
significant screen real-estate. My idea was to have 2 small squares which would indicate if you were connected (left) 
and unmuted (right). The major benefits being that it takes up much less screen-space, and that it wasn't tied to games.
For example, in the screenshot below, the squares are on top of a PyCharm window. 

![](https://camo.githubusercontent.com/6eb4e1829961bdb617d1d5899ef9ceb6adbba4de/68747470733a2f2f692e696d6775722e636f6d2f6f4e53574a62332e706e67)

The major problem I encountered was that there's really no way to guarantee a GUI window (Tkinter in this case) stays on 
top of all other windows. A lot of games work really hard to be the top window, and even if we're constantly calling 
some Tk function to put our window on top, the game will win in the end. Also, several games suffered large framerate 
drops due to the overlay, with the working hypothesis being the aforementioned struggle for the top window. It was a 
good exercise, but ultimately not the solution I was hoping for. 

Ideally, I thought, I could get some kind of "3rd Monitor", one that could just tell me info about my Discord connection 
status. After all, I did solve the problem of getting real-time connection status from the Discord API (which we'll 
discuss later in the code). I had been playing around with the MAX LED matrix for a 
[voice spectrum experiment](https://github.com/samclane/SoundDisplay), and thought it would be a good candidate for this 
hypothetical "stripped-down" 3rd monitor. All I had to do was put the pieces together. 

### Materials

* PC with Python 3 and the following packages:
  * `discord`
  * `pyfirmata`
* Arduino Uno
* MAX7219 Dot Matrix Module ($1.31 USD on [AliExpress](https://www.aliexpress.com/item/MAX7219-Dot-led-matrix-module-MCU-control-LED-Display-module-for-Arduino/1875557666.html?spm=2114.search0104.3.135.78e9775aNry0Zj&ws_ab_test=searchweb0_0,searchweb201602_3_10152_10151_10065_10344_10130_10068_10324_10547_10342_10325_10546_10343_10340_10548_10341_10545_10696_10084_10083_10618_10307_10059_100031_524_10103_10624_10623_10622_10621_10620,searchweb201603_6,ppcSwitch_5&algo_expid=b4f623f2-b836-4fee-90bc-d80337113144-22&algo_pvid=b4f623f2-b836-4fee-90bc-d80337113144&transAbTest=ae803_2&priceBeautifyAB=0))

### The Code 

###### Final File Structure

- DiscordMatrix/
  - config.ini
  - default.ini
  - discord_matrix.py
  - led_matrix.py

[In my last article](http://samclane.github.io/Python-Arduino-PyFirmata/), I wrote an explanation of the LED-Matrix 
"driver" written in Python with PyFirmata. Take that code and put it in a file called `led_matrix.py` and save it for 
later.

Now, on to the meat of the program. How do we actually figure out our Discord connection status? Luckily, there's a 
Discord module for python, aptly named `discord`. Go ahead and `pip install discord`, as well as `pyfirmata` if you
haven't already. 

```python
import atexit
import configparser
import os
import sched
import sys
import threading
import time
from shutil import copyfile

import discord
from pyfirmata import Arduino

from led_matrix import LedMatrix

REFRESH_RATE = 0.5  # seconds

ZEROS = [[0 for _ in range(8)] for _ in range(8)]

DISCONNECTED = [[1, 0, 0, 0, 0, 0, 0, 1],
                [0, 1, 0, 0, 0, 0, 1, 0],
                [0, 0, 1, 0, 0, 1, 0, 0],
                [0, 0, 0, 1, 1, 0, 0, 0],
                [0, 0, 0, 1, 1, 0, 0, 0],
                [0, 0, 1, 0, 0, 1, 0, 0],
                [0, 1, 0, 0, 0, 0, 1, 0],
                [1, 0, 0, 0, 0, 0, 0, 1]]

CONNECTED = [[0, 0, 0, 0, 0, 0, 0, 0],
             [0, 0, 0, 0, 0, 0, 0, 0],
             [0, 0, 0, 0, 0, 0, 0, 1],
             [0, 0, 0, 0, 0, 0, 1, 0],
             [0, 0, 0, 0, 0, 1, 0, 0],
             [1, 0, 0, 0, 1, 0, 0, 0],
             [0, 1, 0, 1, 0, 0, 0, 0],
             [0, 0, 1, 0, 0, 0, 0, 0]]

MUTED = DEAFENED = [[0, 0, 0, 0, 0, 0, 0, 0],
                    [0, 0, 0, 0, 0, 0, 0, 0],
                    [0, 0, 1, 0, 0, 1, 0, 0],
                    [0, 0, 0, 1, 1, 0, 0, 0],
                    [0, 0, 0, 1, 1, 0, 0, 0],
                    [0, 0, 1, 0, 0, 1, 0, 0],
                    [0, 0, 0, 0, 0, 0, 0, 0],
                    [0, 0, 0, 0, 0, 0, 0, 0]]
```

We start by doing all our imports, then by defining a couple constants. `REFRESH_RATE` is how often the program will 
check Discord for voice-status updates. `ZEROS` is an 8x8 matrix of 0s, which is useful for clearing the display. 
`DISCONNECTED`, `CONNECTED` and `MUTED`/`DEAFENED` are all "icon" matrices that we will pass to the `LedMatrix` object so it 
can be drawn to the display. Refer to the [legend](https://github.com/samclane/DiscordMatrix#current-legend) if you want to see what they will look like.

We're going to quickly extend the base `LedMatrix` class so it has internal memory of it's currently drawn matrix. This 
will aid us in reducing how often we have to communicate with the Arduino, which is a relatively lengthy operation. 

```python
class ExtendedMatrix(LedMatrix):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._matrix = ZEROS

    def __eq__(self, other):
        if isinstance(other, ExtendedMatrix):
            return self._matrix == other._matrix
        else:
            return self._matrix == other

    def clear(self):
        super().clear()
        self._matrix = ZEROS

    def draw_matrix(self, point_matrix):
        super().draw_matrix(point_matrix)
        self._matrix = point_matrix
```

We override 2 magic functions, and 2 `LedMatrix` functions. `__init__`, `__clear__`, and `draw_matrix` all get 
overridden simply to update the private `_matrix` variable when the actual hardware state is changed. `__eq__` defines 
comparison for 2 matrices, by directly comparing their LEDs. It also allows comparison with plain 2D arrays. 
This will come in use when we decide whether to update the display or not. 

With that out of the way, we can finally get to interfacing with Discord. We'll do this through a class called 
`DiscordListener`, which will periodically check on our connection. 

```python
class DiscordListener:
    def __init__(self):
        # Initialize client
        self.client = discord.Client()
```

We start our initialization by creating a `discord.Client` object, which will represent our connection to Discord. 
However, it's not logged in yet, and won't be until the end of initialization. I chose to acquire that information 
through a config file, which we will now read from.

```python
        # Try and find username and password from config file
        self.config = configparser.ConfigParser()
        if not os.path.isfile("config.ini"):
            copyfile("default.ini", "config.ini")
        self.config.read("config.ini")
        self.username = self.config['LoginInfo']['Username']
        self.password = self.config['LoginInfo']['Password']
```

It might be a little safer to `pickle` the password instead of adding it to the config file, but I like to live 
dangerously. Just know that your password should go in `config.ini` and not `default.ini`. And don't ever share a 
`config.ini` with anyone if it has your password in it. And **_especially_** don't commit your `config.ini` to a public 
repository<sup>2</sup>.

The other benefit of using a custom `config.ini` is that you can safe custom Arduino pin-outs for the MAX, as we'll see 
below. This code should look a lot like the `__main__` function from the original LedMatrix article. Also, remember we're
using our `ExtendedMatrix` class, instead of the "vanilla" `LedMatrix`.

```python
        # Initialze Arduino and LED-matrix
        board = Arduino('COM3')
        # grab pinout from config file
        dataIn = int(self.config['Pins']['dataIn'])
        load = int(self.config['Pins']['load'])
        clock = int(self.config['Pins']['clock'])
        self.matrix = ExtendedMatrix(board, dataIn, load, clock)
        self.matrix.setup()
```

We're going to use `sched` in combination with `threading` to repeatedly check for voice status updates in the 
background. We'll also save a reference to the thread if we need it later. `sched` will be calling a function called
`self.update_status()` which will be what actually checks our current connection status, and adjusts the matrix 
accordingly. 

```python
        # Will hold a reference to each running thread
        self.threads = {}

        # Schedule update callback
        self.sched = sched.scheduler(time.time, time.sleep)
        self.sched.enter(REFRESH_RATE, 1, self.update_status)
        t_sched = threading.Thread(target=self.sched.run, daemon=True)
        t_sched.start()
        self.threads['t_sched'] = t_sched
```

And finally, we'll run a function called `attempt_login`. This will connect our `Client` to the Discord servers using the 
provided username and password. This also happens in a background thread (`t_client`). 

```python
        self.attempt_login()
        
    def attempt_login(self):
        print("Attempting to log in as {}...".format(self.username))
        # Kill the existing client thread if it already exists
        if self.threads.get("t_client"):
            self.client.logout()
        t_client = threading.Thread(target=self.client.run, args=(self.username, self.password), daemon=True)
        t_client.start()
        self.threads['t_client'] = t_client
        print("Done")
```

And finally, here is `update_status`, the core of the `DiscordListener`. The reason we had to provide our own login info
is so the listener can know which servers to check. As far as I've been able to figure, this is the only way to check 
one's voice connection status. If we're able to find one instance of "non-disconnected behavior" (i.e. `CONNECTED` or 
`MUTED`), then we immediately `break` and update the matrix to the new icon matrix.  

```python
    def update_status(self):
        state = DISCONNECTED
        if self.client.is_logged_in:
            for server in self.client.servers:
                mem = server.get_member(self.client.user.id)
                vs = mem.voice
                if vs.voice_channel is not None:
                    if vs.mute or vs.self_mute:
                        state = MUTED
                        break
                    else:
                        state = CONNECTED
                        break
        if self.matrix != state:
            self.matrix.draw_matrix(state)
        self.sched.enter(REFRESH_RATE, 1, self.update_status)
```

Note that we check `if self.matrix != state` before writing the new icon. This prevents us from doing unnecessary serial
transactions that can tie up the CPU.  

Also, here's the `exit` function, which simply displays a goodbye message, clears the display (really helpful for knowing
when the program is running or not) and closes all the daemon threads with `sys.exit()`. 

```python
    def exit(self):
        print("Exiting...")
        self.matrix.clear()
        sys.exit()
```

And finally, our main function:

```python
def main():
    dl = DiscordListener()
    atexit.register(dl.exit)


if __name__ == "__main__":
    main()
```

If all goes well, you should be seeing the appropriate connection icon on your LED Matrix when you run the code. The 
[full source code along with the default icon legend can be found here](https://github.com/samclane/DiscordMatrix). This
version also includes a system tray icon, which is pretty neat but makes it restricted to Windows. I believe with the
code found only in this tutorial, it should be platform independent.

This display has been hugely helpful for me. I'm always able to instantly tell if I'm in a server and people are able to 
hear me, even if I'm not sitting at my desk (that display is pretty bright!). And there are still plenty of improvements 
to be made, like a separate icon for being "deafened" than "muted".  Let me know if you find any bugs or are unable to 
get the code to work. 

<br />
---

<sup>1</sup>Read: Nerd

<sup>2</sup>This may or may not be autobiographical 