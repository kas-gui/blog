Line wrapping: the hardest problem in text layout?
===============================

September 2020

*Obviously* the title can't be true. Can it?

Unicode provides a ~~short~~ somewhat long [technical report on line breaking](https://www.unicode.org/reports/tr14/).
To quote from its overview:

> Line breaking, also known as word wrapping, is the process of breaking a section of text into lines such that it will fit in the available width of a page, window or other display area. The Unicode Line Breaking Algorithm performs part of this process. Given an input text, it produces a set of positions called "break opportunities" that are appropriate points to begin a new line. The selection of actual line break positions from the set of break opportunities is not covered by the Unicode Line Breaking Algorithm, but is in the domain of higher level software with knowledge of the available width and the display size of the text.

To summarise, this algorithm tells us about mandatory breaks (e.g. after `\n`)
and optional breaks (roughly speaking, the start of each word).

Fortunately for us, the Unicode Line Breaking Algorithm has already been
implemented in Rust, by the [`xi-unicode`](https://docs.rs/xi-unicode/0.2.1/xi_unicode/)
library (from the same people as [Xi Editor](https://github.com/xi-editor/xi-editor)
and [Druid](https://github.com/linebender/druid)).


## One foot in front of the other

![glyph metrics](https://www.freetype.org/freetype2/docs/glyphs/metrics.png)

Lets start with the basics: horizontal, left-to-right text with simple layout.
In this case, we use a *caret* starting at the line's origin. The first glyph is
placed on this caret, then the caret is advanced by the glyph's advance width.
The next glyph is placed similarly, except that if this pair of glyphs appears
in the font's *kerning table*, then an offset is applied (this allows e.g. the
'T' in 'To' to hang over the 'o').

Line wrapping such text is simple: whenever the caret position extends beyond
the available width, we back-step to the last optional line-break position, and
line-break there. Well, not quite: we allow whitespace to extend beyond the
line width (so if a bunch of extra spaces are inserted at the point a line is
wrapped, they don't actually add space anywhere).

The above is what the `glyph_brush_layout` crate (of
[glyph-brush](https://github.com/alexheretic/glyph-brush)) does. It works fine
for most left-to-right languages, provided no complicated layout is needed.

Actually, there is another point missing from this story: hyphenation.
We leave this as a foot-note for now.


## Shaping

Shaping was [discussed previously](why-kas-text.md#shaping) and is an essential
part of complex text support: the above advance-and-apply-kerning rules are
insufficient for ligatures and entirely insufficient for complex texts like Arabic.
A *shaper* is a separate tool which, given a sequence of
(Unicode) text and a font, returns a list of positioned glyphs from this font.
One such shaper is [HarfBuzz](https://harfbuzz.github.io/).

Modifying our text layout system to support a shaper is not so hard (given a
suitable design). Integrating line wrapping with an external shaper is only a
little harder: the shaper returns a *single* line of text, within which we must
track the positions of optional line-breaks.

KAS-text implements support for both simple layout (directly) and complex layout
(via [`harfbuzz-rs](https://docs.rs/harfbuzz-rs)) within its
[shaper module](https://github.com/kas-gui/kas-text/blob/master/src/shaper.rs).


## Right-to-left and bi-directional text

This is where things start to get *fun*. When going left-to-right (hereafter
LTR), we only need to implement the caret position. When going RTL, if only we
could simply use the same logic but with flipped direction: alas, all font
glyphs are positioned *from the left*, so we have to typeset them *backwards*.

Worse, RTL texts may embed short sequences (such as numbers) or even quote
another language in the LTR direction — in some cases even embedding RTL text
within LTR within RTL. [Unicode TR9](https://www.unicode.org/reports/tr9/)
specifies the *Basic Display Algorithm*, which essentially has three parts:

1.  Split the input text into paragraphs (trivial).
2.  Resolve embedding levels, where levels run from 0 to 125 and odd levels
    indicate RTL direction. This is complex but well specified and implemented
    by libraries such as [unicode-bidi](http://docs.rs/unicode-bidi) (albeit
    with bugs).
3.  Re-order the text. This is complex and inseperable from line-wrapping.

To go into further detail, according to Unicode TR9, re-ordering text involves:

1.  Split the input text into *level runs*: maximal sub-sets of characters with
    consistent embedding level.
2.  For each level run, apply shaping to yield a glyph sequence.
3.  Using the result of shaping, calculate line-wrapping positions.
4.  For each line, apply a sequence of rules (L1-L4) to re-order
    characters on that line.

Unfortunately this leaves us a problem: we cannot resolve where lines start and
end without first applying shaping, and we but are given a set of rules to
re-order characters (i.e. Unicode code points) not glyphs. There are two ways
of, er, "solving", this problem:

1.  Apply shaping, calculate line-break positions, re-order (at `char` level),
    shape again. Not only does this require doing shaping twice, but further
    there is no guarantee that the result of doing so will still fit within our
    length bounds. Also, shapers like HarfBuzz expect text in *logical* order.
2.  Transform the re-ordering logic to work with glyph sequences instead of
    characters. HarfBuzz has implicit support for RTL text, so we never re-order
    *characters*, but only *runs* and only then at embedding level 2 and higher.

Option (2) is now the obvious choice, but there are still several details to
work out: line-wrapping both LTR and RTL text, correctly applying alignment,
embedding LTR within RTL and vice-versa, and a few corner cases. Each line has
a dominant (initial) direction (which may not be the same as the paragraph
direction). On that line one may append a whole run (in either direction) or
part of a wrapped line — but since the logical end of a line should not be in
its middle we only allow line wrapping when the run direction matches the line
direction. [This may need adjustment.]


## Summary

As we have seen, line-wrapping is only a small problem within the scope of text
layout, but surprisingly complex in practice due to its inherent
inseperability from other aspects of text layout.
