---
layout: post
comments: true
published: true
title: Cutting a Sprite Sheet with ImageMagick
subtitle: Save time trimming pixel-by-pixel
---
## What is a sprite sheet?

A spritesheet is a single image that contains data on all the graphical elements in a videogame. It can be thought of like a painter's pallet, with each color being replaced by a specific "tile" or image, usually of a uniform size. Games in the past would use spritesheets to hold their graphical information, as passing a single, large texture to a GPU has less overhead than many tiny textures. Here's an example of a spritesheet:

![Roguelike Rpg Asset Pack](https://www.kenney.nl/content/3-assets/96-roguelike-rpg-pack/preview-medals.png)

## Why would you want to cut a sprite sheet?

I'm working on a game, [Discordia](https://github.com/samclane/Discordia), that's procedurally generated. It's currently a lot easier to render with static images than by cutting an image programatically; at least for the time being. Using individual images also makes the structure of game entities a little more apparent, with statically-named images and folders. 

At some point, surely, using sprite sheets might be appropriate, but right now the gameplay is quite slow and even a modest GPU could keep up, so I'll be sticking to my ways. However, not all of Kenney's Sprite Packs have separated sprites; only those with a relatively small number of discrete elements (like the Urban RPG Pack) would have them presliced. The rest come in their raw, untamed Sprite Sheet format. I want to see what each sprite looks like on it's own, and I'm not about to start cutting 1700 squares in MS Paint. 

This seemed to be a trivial problem at a glance, howevever it appeared that many online tools made some critical assumptions about the sprite sheet. Arbitrary information, such as the margin between sprites (if there was any) was set automatically, and led to almost every webiste giving unusable results. It was time to bring out the heavy weaponry: ImageMagick. 

## Method

[ImageMagick](https://imagemagick.org/index.php) is a commandline image creation/editing/conversion tool that's become the standard "Swiss Army Knife" of image manipulation. In an amazing stroke of luck, the tool's `convert [FILE] -crop` function has a flag (`@`) that, when set, allows us to save to __separate__ file names. All the user has to do is provide a C-style formatting string. If we wanted to cut the sprite sheet above, all we have to do is enter the following in a terminal:

`magick convert roguelikeSheet_transparent.png -crop 57x31-1-1@ +repage +adjoin spaced-1_%d.png`

This command states that there are 57 columns and 31 rows (`57x31`), each separated by a 1px offset that we __do not__ want to keep (`-1-1@`). 

The problem of sheet-style assumptions still remains: there doesn't seem to be a way to cut sprites with a non-uniform border. The edge-cases (literally; the sprites on the edge of the sprite sheet) have at least one edge that's 0px. 

Now, I cheated here, because I didn't want to overcompicate things, so I threw the spritesheet into paint.net and increased the canvas size by 2px on both dimensions, essentially extending the 1px margin into the border. I'm sure the `magick convert -extent` would work for padding, but there doesn't seem to be a way to add a relative border; just set a new absolute one. 

![canvas-resize.png]({{site.baseurl}}/images/canvas-resize.png)

Now, with uniform margins, we can move back to the original console command:

`magick convert roguelikeSheet_transparent.png -crop 57x31-1-1@ +repage +adjoin spaced-1_%d.png`

This particular example will output `57x31=1767` individual sprite files. Unfortunately, none of these files are labeled (other than the formatting-string naming convention provided, `spaced-1%d.png`. This is one drawback of cutting via this method: each sprite will have to be inspected manually to determine what it actually is. 

![Split Tiles]({{site.baseurl}}/images/split-tiles.png)

## Conclusion

If you only need 1 or 2 sprites from a sheet, just cut them out yourself. It will be much faster than getting everything setup properly. If you need __ALL/MOST__ the sprites from a sheet, just pass the sheet into your program and split it programatically at runtime. However, if you need the best of both worlds (__lots__ of static images), then you should consider ImageMagick as a viable spritesheet splitting alternative. 
