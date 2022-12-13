Rust: state of GUI, December 2022
=====================

There was a recent
[call for blogs about Rust GUI](https://poignardazur.github.io/2022/12/11/rust-gui-blog-posts-2023/).
So, [Are we GUI yet?](https://www.areweguiyet.com/)

Contents:

-   [Categorised listing of toolkits](#categorised-listing-of-toolkits)
-   [State of KAS](#state-of-kas)
-   [State of GUI](#state-of-gui)

## Categorised listing of toolkits

Lets start by categorising entries from Are we GUI yet, ignoring those which
appear abandoned or not very functional.

### Bindings

Wrappers around platform-specific toolkits:

-   *Mac OS / iOS* - [`cacao`](https://github.com/ryanmcgrath/cacao) - Rust bindings for AppKit and UIKit
-   *Mac OS / iOS* - [`core-foundation`](https://github.com/servo/core-foundation-rs) - Bindings to Core Foundation for macOS
-   *Win32* - [`winsafe`](https://github.com/rodrigocfd/winsafe) - Windows API and GUI in safe, idiomatic Rust

Wrappers around multi-platform non-Rust toolkits:

-   *[FLTK](https://www.fltk.org/)* - [`fltk`](https://github.com/fltk-rs/fltk-rs) - Rust bindings for the FLTK Graphical User Interface library
-   *[Flutter](https://flutter.dev/)* - [`flutter_rust_bridge`](https://github.com/fzyzcjy/flutter_rust_bridge) - High-level memory-safe binding generator for Flutter/Dart <-> Rust
-   *[GTK](https://www.gtk.org/)* - [`gtk`](https://gtk-rs.org/) - Rust bindings for the GTK+ 3 library
-   *[ImGui](https://github.com/ocornut/imgui)* - [`imgui`](https://github.com/imgui-rs/imgui-rs) - Rust bindings for Dear ImGui
-   *[LVGL](https://lvgl.io/)* - [`lvgl`](https://github.com/rafaelcaricio/lvgl-rs) - Open-source Embedded GUI Library in Rust
-   *[Qt](https://www.qt.io/)* - [`cxx-qt`](https://kdab.github.io/cxx-qt/book/) - Safe interop between Rust and Qt
-   *[Qt](https://www.qt.io/)* - [`qmetaobject`](https://github.com/woboq/qmetaobject-rs) - Expose rust object to Qt and QML
-   *[Qt](https://www.qt.io/)* - [`rust_qt_binding_generator`](https://invent.kde.org/sdk/rust-qt-binding-generator) - Generate code to build Qt applications with Rust
-   *[Sciter](https://sciter.com/)* - [`sciter-rs`](https://github.com/sciter-sdk/rust-sciter) - Rust bindings for Sciter

### Web/DOM based

Frameworks built over web technologies:

-   [Dioxus](https://dioxuslabs.com/) - React-like Rust toolkit for deploying DOM-based apps
-   [Tauri](https://tauri.app/) - Deploy an app written in HTML or one of
    several web frontends to desktop or mobile platforms

Note that Dioxus uses Tauri for desktop deployments.

### Rust toolkits

Finally, Rust-first toolkits (excluding the web-based ones above):

-   [Druid](https://github.com/linebender/druid) - Data-oriented Rust UI design toolkit
-   [egui](https://github.com/emilk/egui) - an easy-to-use GUI in pure Rust
-   [fui](https://github.com/marek-g/rust-fui) - MVVM UI Framework
-   [Iced](https://github.com/iced-rs/iced) - A cross-platform GUI library inspired by Elm
-   [KAS] - A pure-Rust GUI toolkit with stateful widgets
-   [OrbTk](https://github.com/redox-os/orbtk) - The Orbital Widget Toolkit
-   [Relm4](https://github.com/Relm4/relm4) - An idiomatic GUI library inspired by Elm
-   [Slint](https://slint-ui.com/) - a toolkit to efficiently develop fluid graphical user interfaces for any display

### Details

Lets take a deeper look at a few of the above.

**Tauri** is a framework for bringing web apps (supporting several major web
frontend frameworks) to the desktop and (eventually) mobile. **Dioxus** is a
React-like frontend written in Rust with built-in support for Tauri.
Tauri has its own rendering library ([WRY]) over platform-specific WebView
libraries along with a windowing library ([TAO]), a fork of [winit].

**Druid**, a data-first Rust toolkit, is interesting in a few ways.
First is the UI design around a shared data model,
giving widgets focussed views through a system of *lenses*.
Second is that it uses its own platform-integration library,
`druid-shell`, along with its own graphics library, [Piet].
Finally, all widgets (from the user or library) implement only a simple
interface, but must be stored within a special `WidgetPod`.
[Watch](https://www.youtube.com/watch?v=zVUTZlNCb8U) a recent talk by Raph
Levien on the state and design of Druid, including the current **Xilem**
re-design.

**egui**, an immediate-mode pure-Rust toolkit, succeeds both in being extremely
easy to use and in having an impressive feature set while being fast.
It has [some limitations](https://github.com/emilk/egui#disadvantages-of-immediate-mode).
Widgets may be implemented with a single function.
Windowing may use [winit], [glutin] or integration; rendering may use [glow],
[Glium] or [wgpu].

**Iced** is an Elm-like toolkit.
It supports integration, [winit] or [glutin] windowing with [wgpu] or [glow]
for rendering or web deployment.

**Relm4** is an Elm-like Rust toolkit over [GTK 4](https://www.gtk.org/).

**Slint** is a UI built around its own markup language (`.slint`) in the style
of QML, with support for apps written in Rust, C++ and JavaScript.
The markup language makes description of simple UIs easy, but as a result
user-defined widgets have completely different syntax and limited
capabilities compared to library-provided widgets.
Slint uses either Qt or [winit] + [femtovg] + various platform-specific
libraries.

**KAS**, by myself, is an efficient retained-state toolkit.
It supports windowing via [winit] and rendering via [wgpu].


## State of KAS

Being a toolkit author myself, I ought to say a few words about the status of [KAS].
In case you hadn't guessed already, I like lists:

-   In September 2021, KAS v0.10 reorganised crates, added support for dynamic
    linking, improved the default theme, and standardised keyboard navigation.
-   Exactly a year later, v0.11 heavily revised the `Widget` traits and
    supporting macros, supported declarative complex static layout within
    widgets, plus many more small changes all aimed at making KAS easier to use.
-   I have just released v0.12 which uses Rust's recent support for Generic
    Associated Types to update temporary APIs.

To state the obvious, KAS isn't very popular. Despite this, it compares well
with other Rust toolkits on features, missing a few (such as screen-reader
support and *dynamic* declarative layout), but also supporting some less common
ones, such as complex text formatting (currently limited to web frameworks,
bindings, Druid and KAS) and fast momentum touch scrolling over large data models.

My main concerns regarding KAS are:

-   Without a userbase and with a (now) healthy competition, motivation to
    further develop the library is limited.
-   Immediate-mode GUIs, the Elm model and Druid all make dynamic layout easy.
    In KAS, all widgets must be declared with static layout (though hiding,
    paging etc. is possible and the declaration is often implicit).
-   Several aspects of KAS from glyph fallback and font discovery to the
    drawing interface and themes are hacks. Producing good implementations from
    these is possible, but alternatives should also be considered (such as
    switching to COSMIC Text or Piet). Either way, significant breaking changes
    should be expected before KAS v1.0.
-   Some redesign to `EventState` is needed to properly support overlay layers.

From another point of view, KAS has had several successes:

-   A powerful [size model](https://docs.rs/kas/latest/kas/layout/struct.SizeRules.html),
    if a little more complex to use than most toolkits
-   Robust and powerful event-handling model
-   Very fast and scalable
-   Not too hard to use for non-dynamic layout (if you tried KAS v0.10 or
    earlier, usability is much improved, and certainly much better than
    traditional toolkits like GTK or Win32)
-   The [easy-cast](https://crates.io/crates/easy-cast) library
-   I learned a plenty about proc-macros, resulting in a rather interesting
    method of implementing the [`Widget`](https://docs.rs/kas/latest/kas/trait.Widget.html)
    trait and the [impl-tools](https://crates.io/crates/impl-tools) library.

Future work, if I feel so inspired, may involve the following:

-   Dynamic declarative layout. Druid's Xilem project has given me ideas for
    a solution which should work over KAS's current model, though likely less
    efficient. This is not the right place to elaborate on this.
-   Async support? Currently, KAS only supports async code via threads and a
    proxy which must be created from the toolkit before the UI starts. Real
    async support allows non-blocking slow event handlers in a single thread.
    KAS *could* support this, e.g. via a button-press handler generating a
    future which yields a message on completion, then resuming the current
    event-handling stack with that message. However, this may not be the best
    approach (especially not if it requires every event-handling widget to
    directly support resuming a future). A variant of this approach may be more
    viable together with the dynamic declarative layout idea mentioned above.


## State of GUI

The purpose of this post is not just to talk about *KAS*, but about Rust GUIs
in general. To quote [a reddit comment](https://www.reddit.com/r/rust/comments/zioh6o/a_call_for_blogs_about_rust_gui_in_2023/izsg62b/):

> GUI will never be "solved" because it has become a problem too complex to be solved with one solution to rule them all. GUI is not one problem anymore, it is a family of problems.

As listed above, we now have many (nascent) solutions.
These share a few common requirements:

-   A need to create (at least one) window
-   A need to render, often with GPU acceleration
-   A need to type-set complex text
-   A need to support accessibility (a11y) and internationalisation (i18n)

So lets talk a little about libraries supporting those things.

### Windowing

[winit] is, perhaps, the de-facto Rust windowing/platform integration library.
It handles window creation and input from at least keyboard, mouse and
touchscreen devices. Making things more difficult is that there is room for
*considerable* scope for feature creep in this project:
Input Method Editors, theme (dark mode) detection, gamepad/joystick support,
clipboard integration, overlay/pop-up layers, virtual-keyboard invocation,
and probably *much* more.

Designing a cross-platform windowing and system-integration library is a hard
problem, and though Winit already has a lot of success, it also has a *lot* of
open issues and in-progress features.
Winit's nearest equivalent is probably SDL2, which focussess mainly on game
requirements (window creation and game controller support).
Winit attempts to be significantly more (particularly regarding
[text input](https://github.com/rust-windowing/winit/issues/1806)).

So, is Winit the right answer, or is attempting to solve all windowing and input
requirements on all platforms for all types of applications a doomed prospect?
My understanding is that Winit's biggest issue is a lack of maintainers with
adequate time and knowledge of the various target platforms.

Thanks to [`raw-window-handle`], there is a degree of flexibility between the
renderer and window manager used, supporting some alternatives to Winit:

-   Tauri maintains [TAO], a fork of winit with deeper GTK integration, most
    notably to support [WRY] via [WebKitGTK](https://webkitgtk.org/).
-   Druid has its own [`druid-shell`](https://docs.rs/druid-shell/latest/druid_shell/)
    library designed for use with [Piet], but thanks to support for
    [`raw-window-handle`] it should also be usable with other renderers.
-   We don't have to use Rust! Some of toolkits above, even those considered
    Rust toolkits like Slint and Relm4, make use of GTK or Qt for platform
    integration and rendering.

### Rendering (backends)

Quite a few solutions are available for *drawing to windows*. Pick one, or
like a few of the above toolkits, support several:

-   [softbuffer](https://crates.io/crates/softbuffer) facilitates pure-CPU
    rendering over a [`raw-window-handle`]
-   [glutin] supports the creation of an OpenGL context over a
    [`raw-window-handle`], thus supporting [glow] (GL on Whatever; the library
    may also be used with SDL2) or [Glium] (an "elegant and safe OpenGL wrapper")
-   Several (low- and high-level) bindings are available to Vulkan, Metal and
    Direct3D; to my knowledge none of these are used directly by any significant
    Rust GUI toolkits
-   [wgpu] is "a cross-platform, safe, pure-rust graphics api. It runs natively
    on Vulkan, Metal, D3D12, D3D11, and OpenGLES; and on top of WebGPU on wasm."
    As a high-level, portarble and modern accelerated graphics API, it is
    (optionally) used by multiple Rust GUI toolkits (eGUI, Iced, KAS) despite
    adding a considerable layer of complexity between the toolkit renderer and
    the GPU.

### Rendering (high-level)

Several Rust toolkits add their own high-level rendering libraries:

-   [WRY] is a rendering layer over platform-dependent WebView backends
-   [Piet] is Druid's rendering layer, leveraging system libraries especially
    for complex font support
-   [Piet-GPU](https://github.com/linebender/piet-gpu) is a research project
    to construct a high-level GPU-accelerated rendering layer
-   [Kurbo](https://crates.io/crates/kurbo) is used by Piet for drawing curves
    and paths
-   [epaint](https://crates.io/crates/epaint/) is "a bare-bones 2D graphics
    library for turning simple 2D shapes and text into textured triangles"

Unaffiliated high-level rendering libraries used by GUI toolkits include:

-   [Lyon](https://github.com/nical/lyon), a path tessellation library (supports
    GPU-accelerated rendering of SVG content)
-   [resvg](https://github.com/RazrFalcon/resvg) is a high-quality CPU-rendering
    library for static SVGs using `tiny-skia`
-   [tiny-skia] is "a tiny [Skia](https://skia.org/) subset ported to Rust"
-   [femtovg] is an "antialiased 2D vector drawing library written in Rust"

### Text and type-setting

To say that type-setting text is a complex problem is a gross understatement.
Sub-problems for fonts include discovering system fonts and configuration,
matching a "font family", use of "fallback" fonts for missing glyphs,
glyph rendering, hinting, sub-pixel rendering and font synthesis.
Sub-problems for text include breaking into level runs (i.e. same left/right
direction), breaking these into same-font runs, shaping (or at least kerning),
potentially multiple times where multiple fonts are required, line-breaking,
hyphenating, indenting, dealing with ligatures, combining diacritics,
navigation, and then there's still everyone's favourite: emojis.
Then there is drawing highlighting and underlines, and if you're still looking
for things to do, correctly justifying Arabic texts (by extending words) and
gapping underlines around descenders.

Right, what do we have? We really have to thank Yevhenii Reizner (@RazrFalcon)
for his efforts including [ttf-parser](https://github.com/RazrFalcon/ttf-parser),
[fontdb](https://github.com/RazrFalcon/fontdb/) and
[rustybuzz](https://github.com/RazrFalcon/rustybuzz)
(not to mention [tiny-skia]!). We have several libraries for rendering and
caching glyphs including [`ab_glyph`](https://crates.io/crates/ab_glyph) and
[Fontdue](https://crates.io/crates/fontdue) which perform basic rendering;
more recently [Swash](https://crates.io/crates/swash) has significantly raised
the bar on the quality of typesetting and rendering.

The above are all low-level components. If you want a high-level interface able
to type-set and render text, what libraries are out there?

-   [`glyph_brush`](https://crates.io/crates/glyph_brush/) is around four years
    old, covering rendering and basic line wrapping. Though this is enough for
    quite a few use-cases, it cannot handle complex text nor does it properly
    handle navigation (which is presumably why Iced still does not support
    multi-line text editing).
-   [Piet] (or more accurately `piet-common`), around three years old, supports
    at rich text, leveraging system libraries (Cairo on Linux).
-   [KAS Text](https://github.com/kas-gui/kas-text/), around two years old and
    written by myself, supports (at least partially) rich text and bi-directional text,
    while missing a few features such as emojis and using a few crude hacks
    (indentation and font discovery).
-   [femtovg], at least since around a year, supports moderately complex text.
-   [COSMIC Text](https://github.com/pop-os/cosmic-text) is a (very) recent
    project by System76 (creators of Pop! OS) for complex text layout in Rust,
    leveraging fontdb, rustybuzz, Swash and other Rust font libraries.
    It seems we can finally have high-quality pure-Rust type-setting.
    Thanks System76!


### Accessibility and internationalisation

Admittedly this is not a topic I have researched. Accessibility for GUIs means
at least supporting keyboard control (now common) and screen readers (less
widely supported). Internationalisation, though a little more complicated than
just string substitution, is less reliant on toolkit support.
We now have a few libraries on these topics:

-   [AccessKit](https://crates.io/crates/accesskit/0.8.1), "UI accessibility
    infrastructure across platforms and programming languages"
-   [TTS](https://github.com/ndarilek/tts-rs) "provides a high-level
    Text-To-Speech (TTS) interface"
-   [rust-i18n](https://crates.io/crates/rust-i18n) is "a crate for loading
    localized text from a set of YAML mapping files", inspired by ruby-i18n
    and Rails i18n
-   [Fluent](https://crates.io/crates/fluent) is "a Rust implemetnation of
    [Project Fluent](https://projectfluent.org/)"
-   [`gettext-rs`](https://crates.io/crates/gettext-rs), "Safe bindings for
    gettext"

[KAS]: https://github.com/kas-gui/kas
[winit]: https://github.com/rust-windowing/winit/
[Piet]: https://github.com/linebender/piet
[glow]: https://github.com/grovesNL/glow
[wgpu]: https://github.com/gfx-rs/wgpu
[Glium]: https://github.com/glium/glium
[glutin]: https://github.com/rust-windowing/glutin
[TAO]: https://github.com/tauri-apps/tao
[WRY]: https://github.com/tauri-apps/wry
[femtovg]: https://github.com/femtovg/femtovg
[`raw-window-handle`]: https://crates.io/crates/raw-window-handle
[tiny-skia]: https://crates.io/crates/tiny-skia/
