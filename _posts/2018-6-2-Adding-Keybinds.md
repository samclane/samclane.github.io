---
layout: post
title: Adding Keybinds to your Python App
---

Today, many of the desktop applications we write are designed to live in the background or desktop tray. They may be 
useful to have open, providing utility for the user in the background. Keybinds can provide a convenient way for the user
to interact with these app without opening them or interacting with them directly. 

Python, being a higher level language, doesn't have a great way to intercept keypresses, especially if the application
in question isn't in focus. The only real solution is [pyHook](https://pypi.org/project/pyHook/), or so I thought, until 
I found out that it has a [horrible bug](https://stackoverflow.com/questions/26156633/pythoncom-crashes-on-keydown-when-used-hooked-to-certain-applications).
Luckily, there's a fork that fixes the problem for Python 3, aptly named [pyHook3](https://pypi.org/project/PyHook3/).

pyHook3 is a little involved to install. Using the standard `pip install PyHook3` yields some cryptic errors, and you
have to end up moving some compiler binaries into your Python folder. I'm sure I was probably missing some dependencies 
in the first place, but that's my problem.

On to the meat and potatoes. Once pyHook is installed, it's time to integrate into your app. Integration can be broken 
down into 2 parts.

1. The user should be able to input custom keybinds into some sort of settings menu. This is almost non-negotiable, as 
since keybinds should be context-free, your Python application might be fighting for keyboard real-estate with other apps.
Take the easy way out and let the user assign them. That way when things don't work accordingly, it's their fault. 

2. The application should be listening for keypresses continuously in the background, and should execute the corresponding
code immediately. This should go without saying. This is the reason you're breaking your code in the first place.

The following class should provide a framework that works for both of these cases.

```python
import logging
from PyHook3 import HookManager
from PyHook3.HookManager import HookConstants


class Keystroke_Watcher:
    def __init__(self, master, sticky=False):
        self.logger = logging.getLogger(master.logger.name + '.Keystroke_Watcher')
        self.hm = HookManager()
        self.hm.KeyDown = self.on_key_down
        self.hm.KeyUp = self.on_key_up
        self.function_map = {}
        self.keys_held = set()
        self.sticky = sticky
        self.hm.HookKeyboard()
```

The core of this class is the `HookManager()`. This is the PyHook3 class that will be doing listening and event callbacks.
We register 2 callbacks, `KeyDown` and `KeyUp`, which will be called every time their respective event occurs. 

`function_map` will contain a list of callback function, and the keys are the specific keypresses. This is how our "bindings"
actually bind to the functionality. We will show how functions are added and called later on. 

`keys_held` holds a list of the key objects being held. If `sticky` is True, `KeyUp` will not clear the set, meaning the 
class holds a history of the unique buttons pressed. This is how the user will input their keybindings into a settings menu.

Finally, we start by "hooking" the keyboard with `self.hm.HookKeyboard()`. It is important we remember to unhook the keyboard 
later on, or else the listener will keep listening and launching callbacks.

```python
    def register_function(self, key_combo, function):
        self.function_map[key_combo.lower()] = function
        self.logger.info(
            "Registered function <{}> to keycombo <{}>.".format(function.__name__, key_combo.lower()))

    def unregister_function(self, key_combo):
        self.logger.info(
            "Unregistered function <{}> at keycombo <{}>".format(self.function_map[key_combo.lower()].__name__,
                                                                 key_combo.lower()))
        del self.function_map[key_combo.lower()]
        
    def get_key_combo_code(self):
        return '+'.join([HookConstants.IDToName(key) for key in self.keys_held])
        
    def on_key_down(self, event):
        try:
            self.keys_held.add(event.KeyID)
        finally:
            return True

    def on_key_up(self, event):
        keycombo = self.get_key_combo_code().lower()
        try:
            if keycombo in self.function_map.keys():
                self.logger.info(
                    "Shortcut <{}> pressed. Calling function <{}>.".format(keycombo,
                                                                           self.function_map[keycombo].__name__))
                self.function_map[keycombo]()
        finally:
            if not self.sticky:
                self.keys_held.remove(event.KeyID)
            return True
```


Here is the main functionality of the class. We can see the first two functions are simple setter/getter functions for 
adding callbacks to the keybinds. `key_combo`s are simple strings constructed by the `get_key_combo_code` function, which
simply returns a `+` separated list of key names. For example, holding the left control button and  "a" would 
yield `lcontrol+a`.

`on_key_down`, we just add the key's ID number to the set of keys. `on_key_up` is where the magic happens. If the keycombo 
is found to be registered to a callback, the corresponding function is executed by calling `self.function_map[keycombo]()`.
Then, if the listener is not sticky, the keys are removed from the set to reflect the new state of the keyboard.

```python
    def shutdown(self):
        self.hm.UnhookKeyboard()

    def restart(self):
        self.keys_held = set()
        self.hm.HookKeyboard()
```

Being able to stop and restart the listener is key for preventing cross-interferance. You will most likely want 2 `Keyboard_Listeners`
in your project. 1 sticky to allow initial binding, and 1 non-sticky to listen for the actual presses. 

This is just the bare-skeleton of what a keylistener should look like in python. This class can be easily extended to 
accept arguments for the keybound functions. However, the bare utility it provides will work out of the box for almost 
any project.