---
layout: post
title: Compile-Time variables with Pyinstaller spec files
subtitle: Insert version number and build time directly into your code!
---

When I was developing [LIFX-Control-Panel](https://github.com/samclane/LIFX-Control-Panel), I was constantly running into
the problem of keeping my version numbers straight. Namely, I was keeping my version number in two separate places; one 
in the top of my `settings.py` file, and the other in the `default.ini` config file. The idea was, the binary would compare
its version string to `config.ini`'s on startup, and if the binary's version was greater than the existing config, it 
overwrite it with the newer `default.ini`. A brilliant, game-changing idea on it's own, but the problem was, I had to 
manually set the values every time I upped the version number or rebuilt the binaries. Naturally, I forgot to keep the
version numbers in lock-step, and much confusion ensued. How could I keep these constant values in some singular place,
where I only have to change them once and it will propagate through the rest of my code. 


The best way I've come up with involves a `_constants.py` file, and some changes to the beginning of my `build.spec` file.
The nice thing about spec files is that they are executed like Python scripts, meaning you can embed Python statements into the 
file and have them execute before the build starts. Without further ado, here's my spec file:

```python
# -*- mode: python -*-
import datetime

bd = datetime.datetime.now().isoformat()
auth = "Sawyer McLane"
vers = "1.3.4"
is_debug = False

# Write version info into _constants.py resource file
with open('_constants.py', 'w') as f:
    f.write("VERSION = \"{}\"\n".format(vers))
    f.write("BUILD_DATE = \"{}\"\n".format(bd))
    f.write("AUTHOR = \"{}\"\n".format(auth))
    f.write("DEBUGGING = {}".format(str(is_debug)))

# Write version info into default config file
with open('default.ini', 'r') as f:
    initdata = f.readlines()
initdata[-1] = "builddate = {}\n".format(bd)
initdata[-2] = "author = {}\n".format(auth)
initdata[-3] = "version = {}\n".format(vers)
with open('default.ini', 'w') as f:
    f.writelines(initdata)


# Rest of the boilerplate spec file build commands...
```

EDIT: [I've uploaded my full spec file as a gist. Here you can see how the whole file goes together.](https://gist.github.com/samclane/cbf44116a3c1e1a9c5e11c5e932360d3) 

So what's going on here? Well, I have 4 variables that I want to keep updated. The first 3 are self explanatory. `is_debug`
enables certain debug statements in the code, and turns on the console in the background. The first block simply overwrites the
`constants.py` file with the new values. The second block is a little more advanced. It reads in the existing `default.ini` file,
and changes the corresponding lines, then writes the value back. This keeps existing values unchanged. 

Afterwards, `_constants.py` looks like this:

```python
VERSION = "1.3.4"
BUILD_DATE = "2018-06-06T15:40:54.587551"
AUTHOR = "Sawyer McLane"
DEBUGGING = True
```

and `default.ini` looks like this:

```
[AverageColor]
defaultmonitor = get_primary_monitor()

[PresetColors]

[Keybinds]


# Used for diagnostic purposes. Please do not change.
[Info]
version = 1.3.4
author = Sawyer McLane
builddate = 2018-06-06T15:40:54.587551
```

And that's all there really is to it. Anywhere you want to use your constants, just `import _constants` and fire away!