Why I created kas-text
==============

September 2020

## Existing libraries

This is going to be a very brief (and fairly naive) dive into text rendering and
the state of available Rust libraries, written in easy English with minimal
jargon.

### The basics

Lets see. You're writing a small graphical game/tool/general UI and wish to
print a simple string of text, say, "Hello World!" There are many APIs that can
do this, but still several steps involved:

1.  Selecting a font: this might be embedded or the app may try to discover a
    "standard" system font. The `font-kit` crate can discover system fonts.
2.  Reading the font: there are a selection of libraries available, from the
    low level `ttf-parser` to the mid-level `ab_glyph` to more complex toolsets
    including `font-kit`, `rusttype` and `fontdue`.
3.  Translating the input string to a sequence of positioned and scaled glyphs
    (*layout*): the above-mentioned `font-kit`, `rusttype` and `fontdue` can
    all accomplish this, along with the more specialised `glyph_brush_layout`.
4.  Rasterising glyphs: this is the job of `glyph_brush` and, again, `font-kit`,
    `rusttype` and `fontdue`.
5.  Integrating rasterisation with the graphics pipeline: e.g. `gfx_glyph` and
    `wgpu_glyph` do this job. Several high-level libraries (e.g. `ggez` and
    `amethyst_ui`) provide direct text support.

Of course, mostly you only need consider steps 1 and 5 and you're done.

### Wrapping and alignment

Above, we considered only a short line of text: "Hello World!" Frequently you
will have longer pieces of text that require *wrapping* (usually on word
boundaries), and you may wish the result to be *aligned* to the left, centre or
right, or even *justified*. Fortunately the above libraries already have you
covered (aside from justified text, which is less often supported).

Text wrapping is a deceptively simple problem, but more on that in my
[next post](line-wrapping.md).

### Shaping

The above libraries all use a fairly simple layout system: match each `char` to
a font glyph, place this at the previous glyph's "advance position", then if the
font provides kerning data for this pair of glyphs use it to adjust the gap
between the two. For English (and probably most languages) this is sufficient,
at least most of the time.

*Shaping* allows more complex glyph selection and positioning, and is an
important part of *real* text libraries. See e.g.
[HarfBuzz on shaping](https://harfbuzz.github.io/what-is-harfbuzz.html#what-is-text-shaping)
and
[Wikipedia on Complex Text Layout](https://en.wikipedia.org/wiki/Complex_text_layout)
(although the latter also concerns bidirectional text; more on that later).

Essentially, shaping takes a text string along with a font and spits out a
sequence of positioned glyphs. This means that replacing the basic layout system
used above with a *text shaper* should, in theory, be straightforward.

For Rust, we have the HarfBuzz binding `harfbuzz-rs`, as well as two immature
pure-Rust libraries, `rustybuzz` and `allsorts`. Of the general-purpose text
libraries, only `fontdue` includes shaping in its roadmap.

### Right-to-left and bidirectional text

Several scripts, such as Hebrew and Arabic, are written right-to-left.
Supporting such scripts generally also requires supporting embedded left-to-right
fragments, and thus requires supporting *bidirectional* texts.

Unicode, as part of [TR9](https://www.unicode.org/reports/tr9/), specifies how
to determine the *bidi embedding level* and thus the text direction for each
character in the source text, as well as how to re-order the input from logical
order into display order. Unfortunately, integrating support for bidi
texts into layout is not so simple (this is most of the topic of the
[next post](line-wrapping.md)).

The `unicode-bidi` crate implements much of Unicode TR9, but on its own this
is insufficient. *None* of the above libraries support bidirectional text.

### Formatting

Formatting may apply one of several effects to text. These include:

-   adjusting the font size
-   changing the font's (foreground) colour
-   using a bold variant (or more generally, adjusting the weight)
-   using an italic variant (either via use of another font variant or via
    slanting glyphs during rendering)
-   drawing underline or strikethrough

Some of the above font libraries have limited support for formatting specified
via a list of text sections each with a font size and font identifier, and in
some cases also colour information. The user is expected to translate formatted
text from its source format as well as to select appropriate fonts.

### Font properties and variable fonts

When mentioning font selection above, we glossed over a couple of issues. How
does one select an appropriate italic or bold variant? Can these be synthesised
if not provided?

The `font-kit` crate can help with the above: it allows font selection by family,
weight and style. Some fonts are *variable*, meaning that glyphs can be
synthesised to the required weight (giving much more control than simply "bold"
or "normal"). The style allows selection of normal, *italic* (curved) or
*oblique* (slanted) variants.

I have not investigated this topic, but it appears that only `font-kit`'s
rasteriser API supports selection of weight or style; all other Rust libraries
select font only by path/bytes and variant index (for multiple fonts embedded
in a font pack).

### Font fallbacks

What if your chosen font doesn't include all required glyphs, e.g. if you choose
a font covering most European alphabets, but your user starts writing Korean?
I believe that no Rust libraries have even started trying to address this
problem.

The first sub-problem is an extension of font selection: choosing appropriate
fallback font(s). CSS allows the user direct control here: see
[article](https://css-tricks.com/css-basics-fallback-font-stacks-robust-web-typography/).
On Linux this is usually handled by Fontconfig.

The other sub-problem(s) concern integration: breaking text layout into multiple
runs should your shaper not support multiple fonts (as HarfBuzz doesn't), then
stitching the result back together.


## KAS-text

The above libraries cover simple layout and rasterisation fairly well, but
also leave a lot out, especially regarding complex layout and use of formatted
input texts.

KAS-text attempts to fill this gap: translate from a raw input text to a
sequence of positioned glyphs, which can be easily adapted for use by
renderers such as `wgpu_glyph`. Additionally, it exposes a stateful
`prepared::Text` object, allowing fast re-draws and re-wrapping of a given
input text (though this stateful API may not be appropriate for all users).

Since this article is written after-the-fact,
[`kas-text`](https://github.com/kas-gui/kas-text/) v0.1 already exists. You can
read its [API docs here](https://docs.rs/kas-text/). v0.1 already supports
shaping (via HarfBuzz) and bidirectional text, which, to my knowledge, makes it
the first Rust library to support these features (on the full layout cycle from
raw text input to positioned glyphs), admittedly with imperfections.

Near-future plans include support for formatted text, including translation from
Markdown or (simple) HTML input and mostly-automatic font selection.
Other topics, such as vertical text, font synthesis, font fallbacks and
emoticons, have not been planned but could eventually be added.

### KAS GUI

The v0.5 release of KAS has integration with KAS-text, including a reasonably
functional multi-line text-editor. [Check it out](https://github.com/kas-gui/kas/tree/master/kas-wgpu/examples#layout) or run yourself:
```sh
git clone https://github.com/kas-gui/kas.git
cd kas/kas-wgpu
cargo run --example layout --features shaping
```
