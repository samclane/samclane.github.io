---
layout: post
comments: true
published: true
title: Rust eGUI Color Sliders
description: Create a custom color picker in Rust using eGUI
---

When creating [Mantle](https://github.com/samclane/mantle), I again found myself needing to create custom color sliders for Hue, Saturation, Brightness, and Kelvin. I was able to
create sliders in [LIFX-Control-Panel](https://github.com/samclane/LIFX-Control-Panel) with Tkinter, but they were far from ideal. I wanted to create a consistent look and feel. Luckily, egui
makes it easy to create custom widgets.

![Mantle](..\images\color-slider-egui\mantle.png)

## eGUI

Egui is a simple, fast, and highly portable immediate mode GUI library written in Rust. It allows developers to create graphical user interfaces in a straightforward and expressive manner.
Immediate mode GUIs allow for a more reactive and dynamic user interface, as the GUI is redrawn every frame, perfect for monitoring smart bulbs.

Since I'm packaging Mantle as a native application, I used [eframe](https://github.com/emilk/egui/tree/master/crates/eframe)

## Setup

First, we should create a new Rust project.

```bash
cargo new color_slider_demo
cd color_slider_demo
```

To get started with eframe, add the following to your `Cargo.toml`:

```toml
eframe = { version = "0.28.1" }
```

## Creating the Color Slider

We'll start by creating a function called color_slider that will render our custom slider and handle user interactions.

Create a new file `src/ui.rs` and add the following code:

```rust
use egui::*;
const N: u32 = 6 * 6;

pub fn color_slider(
    ui: &mut Ui,
    value: &mut u16,
    range: std::ops::RangeInclusive<u16>,
    label: &str,
    color_at: impl Fn(u16) -> Color32,
) -> Response {
    let desired_size = vec2(ui.spacing().slider_width, ui.spacing().interact_size.y);
    let (rect, response) = ui.allocate_at_least(desired_size, Sense::click_and_drag());

    // Handle user input
    if let Some(pointer_pos) = response.interact_pointer_pos() {
        *value = remap_clamp(
            pointer_pos.x,
            rect.left()..=rect.right(),
            *range.start() as f32..=*range.end() as f32,
        )
        .round() as u16;
    }

    // Provide widget information for accessibility and debugging
    response.widget_info(|| WidgetInfo::selected(
        WidgetType::Slider,
        ui.is_enabled(),
        response.drag_started(),
        label,
    ));

    // Render the slider if it's visible
    if ui.is_rect_visible(rect) {
        let visuals = ui.style().interact(&response);

        // Draw the gradient mesh
        {
            let mut mesh = Mesh::default();
            for i in 0..=N {
                let t = i as f32 / N as f32;
                let color = color_at((t * u16::MAX as f32) as u16);
                let x = lerp(rect.left()..=rect.right(), t);
                let y_offset = ui.spacing().slider_rail_height / 2.0;
                mesh.colored_vertex(pos2(x, rect.center().y + y_offset), color);
                mesh.colored_vertex(pos2(x, rect.center().y - y_offset), color);
                if i < N {
                    let idx = 2 * i;
                    mesh.add_triangle(idx, idx + 1, idx + 2);
                    mesh.add_triangle(idx + 1, idx + 2, idx + 3);
                }
            }
            ui.painter().add(Shape::mesh(mesh));
        }

        // Draw the slider outline
        ui.painter().rect_stroke(rect, 0.0, visuals.bg_stroke);

        // Render the slider handle
        {
            let x = lerp(
                rect.left()..=rect.right(),
                remap_clamp(
                    *value as f32,
                    *range.start() as f32..=*range.end() as f32,
                    0.0..=1.0,
                ),
            );
            let radius = ui.spacing().slider_rail_height / 1.3;
            let picked_color = color_at(*value);
            ui.painter().circle(
                pos2(x, rect.center().y),
                radius,
                picked_color,
                Stroke::new(visuals.fg_stroke.width, picked_color),
            );
        }

        // Optional: Add a text field for precise input
        let mut text = value.to_string();
        let text_response = ui.add(TextEdit::singleline(&mut text).desired_width(50.0));
        if text_response.changed() {
            if let Ok(v) = text.parse::<u16>() {
                *value = v;
            }
        }
    }

    response
}
```

Our parameters are:

- `ui`: The egui context
- `value`: The current value of the slider
- `range`: The range of the slider of values the slider can take
- `label`: A label for the slider
- `color_at`: A function that maps a value to a `Color32`

We define a desired size for the slider based on Egui's spacing settings. We then check if the user is interacting with the slider, and update it accordingly. To render our widget,
we create a gradient mesh of colored rectangles across the width of the slider. We outline the slider and render the slider handle at the current value. Finally, we add a text
field for precise input.

Admittedly, the most difficult portion is the drawing of the gradient mesh.

```rust
{
    let mut mesh = Mesh::default();
    for i in 0..=N {
        let t = i as f32 / N as f32;
        let color = color_at((t * u16::MAX as f32) as u16);
        let x = lerp(rect.left()..=rect.right(), t);
        let y_offset = ui.spacing().slider_rail_height / 2.0;
        mesh.colored_vertex(pos2(x, rect.center().y + y_offset), color);
        mesh.colored_vertex(pos2(x, rect.center().y - y_offset), color);
        if i < N {
            let idx = 2 * i;
            mesh.add_triangle(idx, idx + 1, idx + 2);
            mesh.add_triangle(idx + 1, idx + 2, idx + 3);
        }
    }
    ui.painter().add(Shape::mesh(mesh));
}
```

In essence, all we're doing is:

- Iterating over N steps to create a smooth gradient.
- For each step, calculate a color using the color_at function.
- Create vertices at the top and bottom of the slider for each color.
- Add triangles between vertices to form the gradient mesh.

The user interactions are captured in the `value` when the slider is clicked or dragged.

```rust
if let Some(pointer_pos) = response.interact_pointer_pos() {
    *value = remap_clamp(
        pointer_pos.x,
        rect.left()..=rect.right(),
        *range.start() as f32..=*range.end() as f32,
    )
    .round() as u16;
}
```

Our main purpose here is just to remap the pointer position to the slider's range and vice versa.

Mantle uses a custom 32-bit HSBK color format to prevent overflows, even though LIFX uses a 16-bit format:

```rust
pub struct HSBK32 {
    pub hue: u32,
    pub saturation: u32,
    pub brightness: u32,
    pub kelvin: u32,
}

impl From<Color32> for HSBK32 {
    fn from(color: Color32) -> HSBK32 {
        let rgb = RGB8 {
            red: color.r(),
            green: color.g(),
            blue: color.b(),
            temperature: None,
        };

        let hsbk: HSBK = rgb.into();
        hsbk.into()
    }
}

impl From<HSBK32> for Color32 {
    fn from(hsbk: HSBK32) -> Color32 {
        let rgb: RGB8 = hsbk.into();
        rgb.into()
    }
}
```

## Integrating the Color Slider

Now that we have our function, let's integrate it into our main application's UI.

In `src/main.rs`, add the following code:

```rust
use egui::*;
use crate::color_slider;
use eframe;
mod ui;

struct MyApp {
    hue: u16,
    saturation: u16,
    brightness: u16,
    kelvin: u16,
}

impl Default for MyApp {
    fn default() -> Self {
        Self {
            hue: 0,
            saturation: 0,
            brightness: 0,
            kelvin: 2500,
        }
    }
}

impl eframe::App for MyApp {
    fn name(&self) -> &str {
        "Custom Color Sliders"
    }

    fn update(&mut self, ctx: &egui::Context, frame: &eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.vertical(|ui| {
                ui.horizontal(|ui| {
                    ui.label("Hue");
                    color_slider(ui, &mut self.hue, 0..=u16::MAX, "Hue", |v| {
                        HSBK32 {
                            hue: v as u32,
                            saturation: u32::MAX,
                            brightness: u32::MAX,
                            kelvin: 0,
                        }
                        .into()
                    });
                });
                // Repeat for saturation, brightness, and kelvin
            });
        });
    }
}
```

In this example:

- We define a MyApp struct to hold the color values.
- We implement the eframe::App trait to create the application.
- In the update function, we create sliders for hue, saturation, brightness, and kelvin.

## Conclusion

By creating a custom color_slider function, we've built a flexible and visually appealing way to adjust color parameters in a Rust application using Egui. This approach allows for extensive customization and can be adapted for various types of controls beyond color sliders.

Feel free to extend this example by adding more features, such as alpha adjustment, presets, or anything else you can think of!
