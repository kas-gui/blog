# Calculator

![Calculator](https://raw.githubusercontent.com/kas-gui/kas/master/screenshots/calculator.png)

This tutorial rebuilds the [example calculator](https://github.com/kas-gui/kas/tree/master/kas-wgpu/examples#calculator).
You may refer to [its source code](https://github.com/kas-gui/kas/blob/master/kas-wgpu/examples/calculator.rs).

## Calculator API

I assume you are familiar with Rust? This is the API we will be using for the calculator itself:
```rust
#[derive(Debug)]
enum Key {
    Clear,
    Divide,
    Multiply,
    Subtract,
    Add,
    Equals,
    Char(char),
}

#[derive(Debug)]
struct Calculator {
    ...
}

impl Calculator {
    fn display(&self) -> String {
        ...
    }

    // return true if display changes
    fn handle(&mut self, key: Key) -> bool {
        ...
    }
}
```
I'll let you fill in the details yourself (or copy from the example linked above).

As you see, the API is nice and simple: one method to generate our display string
and another to handle key presses, with an enum to encode supported operations.

We do derive `Debug` on both types. This is a design decision in KAS: supporting
`Debug` in KAS widgets requires that this is supported by user types used by the
widgets, and unfortunately this cannot be optional. You can always
[implement Debug yourself](https://doc.rust-lang.org/stable/std/fmt/trait.Debug.html)
should the `derive(Debug)` macro fail (or spill out details you wanted to keep secret)!

## A simple window

Lets start by building a simple UI with just an edit-box. (This is basically just a template.)
```rust
use kas::widget::{EditBox, Window};

fn main() -> Result<(), kas_wgpu::Error> {
    env_logger::init();

    let content = EditBox::new("0");
    let window = Window::new("Calculator", content);

    let theme = kas_theme::ShadedTheme::new();
    kas_wgpu::Toolkit::new(theme)?.with(window)?.run()
}
```
Simple? To explain:

-   our `main` function may fail with the `kas_wgpu::Error` type; `Toolkit::new`
    and `Toolkit::with` can fail (`?` "try" operator)
-   we construct an `EditBox` and a `Window` around that
-   we use the `ShadedTheme` (with default colours)
-   we initialise the toolkit with our theme, add our window, and run it

Note that `Toolkit::run` does not return. It is in fact a wrapper around
[`winit::event_loop::EventLoop::run`](https://docs.rs/winit/0.24.0/winit/event_loop/struct.EventLoop.html#method.run),
which does not return. By default, the program will exit after all windows
have closed.

## A button!

Of course the above doesn't *do* anything, so lets add a button... as it happens,
a button *must* send a message, and messages *must* be handled, so lets implement
a counter. Handlers typically reside in a layout (parent) widget, so we add that too.
```rust
use kas::class::HasString;
use kas::event::{Manager, Response, VoidMsg};
use kas::macros::make_widget;
use kas::widget::{EditBox, TextButton, Window};

fn main() -> Result<(), kas_wgpu::Error> {
    env_logger::init();

    let content = make_widget!{
        #[layout(column)]
        #[handler(msg = VoidMsg)]
        struct {
            #[widget] display: impl HasString = EditBox::new("0").editable(false),
            #[widget(handler = count)] _ = TextButton::new("count", ()),
            counter: u32 = 0,
        }
        impl {
            fn count(&mut self, mgr: &mut Manager, _: ()) -> Response<VoidMsg> {
                self.counter += 1;
                *mgr |= self.display.set_string(self.counter.to_string());
                Response::None
            }
        }
    };
    let window = Window::new("Calculator", content);

    let theme = kas_theme::ShadedTheme::new();
    kas_wgpu::Toolkit::new(theme)?.with(window)?.run()
}
```

Whoah... a lot just happened there right? Lets start with the little things:

-   we set `.editable(false)` on our `EditBox` since it is for display only
-   we construct a button: `TextButton::new("count", ())`

Wait, what's that `()` doing there? That is our message. When the button is
clicked, it sends a message to its parent widget, which calls `count`:

```rust
fn count(&mut self, mgr: &mut Manager, _: ()) -> Response<VoidMsg> {
    self.counter += 1;
    *mgr |= self.display.set_string(self.counter.to_string());
    Response::None
}
```
Perfectly ordinary method here. Its parameters are `&mut self` (the widget
doing the handling), `mgr: &mut Manager` (the "event manager"), and `_: ()`
(the message our button passed, which we ignore here).




```
#[derive(Clone, Debug, VoidMsg)]
enum Key {
```
