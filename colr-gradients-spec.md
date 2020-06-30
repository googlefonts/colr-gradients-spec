# Gradients for COLR/CPAL Fonts

December 2019

### Authors:
* Behdad Esfahbod ([@behdad](https://github.com/behdad))
* Dominik Röttsches ([@drott](https://github.com/drott))

## Table of Contents

- [Introduction](#introduction)
- [Backwards Compatibility](#backwards-compatibility)
- [Color Palette Variation](#color-palette-variation)
- [High-level Design](#high-level-design)
- [Graphical Primitives](#graphical-primitives)
  * [Color Line](#color-line)
    + [Extend Mode](#extend-mode)
      - [Extend Pad](#extend-pad)
      - [Extend Repeat](#extend-repeat)
      - [Extend Reflect](#extend-reflect)
  * [Linear Gradients](#linear-gradients)
  * [Radial Gradients](#radial-gradients)
- [Structure of gradient COLR v1 extensions](#structure-of-gradient-colr-v1-extensions)
- [Implementation](#implementation)
  * [Font Tooling](#font-tooling)
  * [Rendering](#rendering)
    + [FreeType](#freetype)
    + [Chromium](#chromium)
  * [HarfBuzz](#harfbuzz)
- [Acknowledgements](#acknowledgements)


# Introduction

We’re proposing a format extension to the
[COLR](https://docs.microsoft.com/en-us/typography/opentype/spec/COLR) table to
allow gradient fills in addition to the existing solid color fills defined for
COLR. Current version number of COLR table is 0.  We propose this as COLR table
format version 1.  Older proposal and discussion [are
here](https://docs.google.com/document/d/1--J9CubVEIC1Pe9r7Nri6MNtZM0I92MjrYOCdJZteEI/edit).
This version addresses all issues raised with, and supersedes, the older
proposal.

It is our understanding that this brings the capabilities of COLR/CPAL to match
those of SVG Native for vector graphics.  SVG Native allows embedding PNG and
JPEG images while this proposal does not.  We like to explore in the future, how
COLR/CPAL can be mixed with sbix to address that limitation as well.

# Backwards Compatibility

The proposed design allows full backwards compatibility. This means, that a font
designed for COLR format v1 specification, can contain sufficient information to
be readable by a layout and rasterization engine that understands the v0 format.
This is possible because the format version of the COLR table is a short format,
as such considered a "minor", not a “major” version number.

If table format v1 is chosen, additional data will be read which specifies the
additional information for gradients.  All backward-compatibility concerns from
the earlier proposal are addressed in this version.

# Color Palette Variation

Allowing palette colors in
[CPAL](https://docs.microsoft.com/en-us/typography/opentype/spec/CPAL) to change
by variations is desired.  However, this needs more thought.  Colors expressed
in sRGB r/g/b channels *cannot* be easily interpolated.  Another solution is
needed, perhaps involving transforming to a linear color space and back.  To be
pursued separately.

# High-level Design

COLR table is extended to expose a new vector of layers per glyph.  If a glyph
is not found in the new vector, client will try finding it in the COLR v0 glyph
vector and fall back to no-color if the glyph is not found there either.

A glyph using the new extension is mapped to an ordered list of layers.  Each
layer includes a glyph id of shape as well as a *paint*.  There are three types
of paint defined currently, with capacity for future extensions: solid, linear
gradient, radial gradient.

We have added a transparency scalar to each invocation of a palette color.  This
allows for the expression of translucent versions of palette entries, as well as
foreground, which we find useful.  Without this, various translucent shades of
the same color would need to be encoded separately in the color palette, which
is undesirable since color palette entries are designed to be exposed to
end-users.

All values expressed are *variable* by way of OpenType 1.8 Font Variations
mechanisms.

# Graphical Primitives

The two main graphical primitives that are added are linear gradients and radial
gradients.

In most graphics systems, linear gradients are declared using two points, and
radial gradients are declared using two circles.  Such graphics systems also
support a transformation matrix, via which one can get shear in linear
gradients, or arbitrary ellipses with radial gradients.  Since our proposed
format does *not* have such universal transform underneath, the definition of
linear and radial gradients are extended from their typical form to accommodate
for these transformations in the gradient declaration itself.

## Color Line

A color line is a function that maps real numbers to a color value to define a
1-dimensional gradient, to be used and referenced from [Linear
Gradients](#heading=h.696rgjwvuoq9) and [Radial
Gradients](#heading=h.2iel67nie7a). Colors of the gradient are defined by *color
stops*.

Color stops are defined at color stop positions. Color stop position 0 maps to
the start point of a linear gradient or the center of the first circle of a
radial gradient. Color stop position 1 maps to the end point of a linear
gradient or the center of the second circle of a radial gradient.  In the
interval [0, 1] the color line must contain at least one color stop, but may
contain multiple color stops that define the gradient.

Outside the defined interval, the gradient pattern in between the outer defined
positions is repeated according to the color line [extend
mode](#heading=h.4dwrambuyuzf).

If there are multiple color stops defined for the same coordinate, the first one
is used for computing the color value for values below the coordinate, the last
one is used for computing the color value for values above. All other color
stops for this coordinate are ignored.

Limiting the specified interval to a sub-range of [0, 1] allows for looping
through colors repeatedly along the mapped distance, without having to encode
them multiple times.  In that sense, our color line is similar to CSS
[repeating-linear-gradient()](https://developer.mozilla.org/en-US/docs/Web/CSS/repeating-linear-gradient)
and
[repeating-radial-gradient()](https://developer.mozilla.org/en-US/docs/Web/CSS/repeating-radial-gradient)
functions.

In order to achieve:

* one gradient along the gradient positions (linear, or radial) and padded
  colors outside this range, color stops at 0 and 1 must be defined, and color
  line extend mode *pad* must be used. This achieves similarly behavior as
  defined in CSS
  [linear-gradient()](https://developer.mozilla.org/en-US/docs/Web/CSS/linear-gradient)
  and
  [radial-gradient()](https://developer.mozilla.org/en-US/docs/Web/CSS/radial-gradient)
  functions.

* a repeated gradient along the gradient positions (linear or radial): divide 1
  by the number of desired repetitions and use the result as your maximum color
  stop, then use color line extend mode *repeat* to have it continue outside the
  defined interval.

* a mirrored / color-circle gradient: divide 1 by two times the number of
  desired full color stripes, and define the color stops between the 0 and the
  result of this division, then use color line extend mode *reflect* to have it
  continue mirrored.

![Repeating linear gradient](images/repeating_linear.png) ![Repeating radial gradient](images/repeating_radial.png)

***Figure 1:** Repeating linear and radial gradients
([source](https://cssnewbie.com/apply-cool-linear-and-radial-gradients-using-css/))*

### Extend Mode

We propose three extend modes to control the behavior of the gradient outside
its specified endpoints:

#### Extend Pad

For numbers outside the defined interval the color line continues to map to the
outer color values, i.e. for values less than the leftmost defined color stop,
it maps to the leftmost color stop value; for values greater than the rightmost
defined color stop value, it maps to the rightmost defined color value.

#### Extend Repeat

For numbers outside outside the interval, the color line continues to map as if
the defined interval was repeated.

#### Extend Reflect

For numbers outside the defined interval, the color continues to map as if the
interval would continue mirrored from the previous interval. This allows
defining stripes in rotating colors.

## Linear Gradients

We propose definitions of linear gradients with two color line points P0 and P1
between which a gradient is interpolated. A point P2 is defined to rotate the
gradient angle / orientation separately from the color line endpoints.

If the dot-product (P₁ - P₀) . (P₂ - P₀) is zero (or near-zero for an
implementation-defined definition) then gradient is ill-formed and nothing must
be rendered.

![Defining points for linear gradients](images/linear_defining_points.png)

*__Figure 2:__ Linear gradient defining points*

## Radial Gradients

A radial gradient in this proposal is a gradient between two—optionally
transformed—circles, namely with center c0 and radius r0, and center c1 and
radius r1 and a specified color line.  The circle c0, r0 will be drawn with the
color at color line position 0. The circle c1, r1 will be drawn with the color
at color line colorLine position 1.

The optionally defined affine transformation matrix is used to transform the
circles into ellipses around their center.

The drawing algorithm radial gradients follows the [HTML WHATWG Canvas spec for
createRadialGradient()](https://html.spec.whatwg.org/multipage/canvas.html#dom-context-2d-createradialgradient).
Quoting and adapting from there.  With circle center points c0 and c1 defined as
c0 = (x0, y0) and c1 = (x1, y1):

Radial gradients must be rendered by following these steps:

1. If c0 = c1 and r0 = r1 then the radial gradient must paint nothing. Return.

2. Let x(ω) = (x1-x0)ω + x0
Let y(ω) = (y1-y0)ω + y0
Let r(ω) = (r1-r0)ω + r0  
Let the color at ω be the color at that position on the gradient color line (with the colors coming from the interpolation and extrapolation described above).

3. For all values of ω where r(ω) > 0, starting with the value of ω nearest to
   positive infinity and ending with the value of ω nearest to negative
   infinity, draw the circumference of the ellipse resulting from translating
   circle with radius r(ω) by affine transform at position (x(ω), y(ω)), with
   the color at ω, but only painting on the parts of the bitmap that have not
   yet been painted on by earlier circles in this step for this rendering of the
   gradient.

![Example radial gradient rendering](images/example_radial.png)

***Figure 3:** Example of a radial gradient rendering.*

**TODO:** Add illustration of center points, radii, etc. similar to the radial
one.

**Note:** The coordinates for the circle/ellipse centers are *not*
transformed by the affine matrix if present.  Implementations might need to
apply the inverse of the affine matrix to center coordinates before passing all
to underlying graphics systems like Skia or Cairo that apply their (often called
*user-*) transformation matrix to everything.

**Note:** Implementations must be careful to properly render radial gradient
even if the provided affine matrix is
*[degenerate](https://en.wikipedia.org/wiki/Invertible_matrix)* or
*near-degenerate*. Such radial gradients do have a well-defined shape, which is
a strip or a cone filled with a linear gradient.  Implementations using the
inverse-transform approach noted above can fall back to a linear-gradient
combined with a clipping path to achieve proper rendering of problematic affine
transforms.

# Structure of gradient COLR v1 extensions

```C++
// Base types

template <typename T, typename Length=uint16>
struct ArrayOf
{
  Length count;
  T      array[/*count*/];
};

template <typename T>
struct Variable
{
  T      value;
  VarIdx varIdx;
};

typedef uint32 VarIdx;

template <typename T>
struct Variable
{
  T      value;
  VarIdx varIdx;
};

typedef Variable<Fixed> Scalar;

typedef Variable<FWORD> Position;

typedef Variable<FUWORD> Distance;

Typedef Variable<F2DOT14> NormalizedScalar; // [-1.0, 1.0]

struct Affine2x2
{
  Scalar    xx;
  Scalar    xy;
  Scalar    yx;
  Scalar    yy;
};

Struct Point
{
  Position  x;
  Position  y;
};

// Building blocks

struct Color
{
  uint16           paletteIndex;
  NormalizedScalar transparency; // Values outside [0.,1.] reserved.
};

struct ColorStop
{
  NormalizedScalar stopOffset;
  Color            color;
};

enum Extend : uint16
{
  EXTEND_PAD     = 0,
  EXTEND_REPEAT  = 1,
  EXTEND_REFLECT = 2,
};

struct ColorLine
{
  Extend             extend;
  ArrayOf<ColorStop> stops;
};

// Paint types

union Paint
{
  uint16              format;
  PaintSolid          solid;
  PaintLinearGradient linearGradient;
  PaintRadialGradient radialGradient;
};

struct PaintSolid
{
  uint16 format; // = 1
  Color  color;
};

struct PaintLinearGradient
{
  uint16              format; // = 2
  Offset32<ColorLine> colorLine;
  Point               p0;
  Point               p1;
  Point               p2; // Normal; Equal to p1 in simple cases.
};

struct PaintRadialGradient
{
  uint16              format; // = 3
  Offset32<ColorLine> colorLine;
  Point               c0;
  Point               c1;
  Distance            r0;
  Distance            r1;
  Offset32<Affine2x2> affine; // May be NULL.
};

// Header

struct LayerV1Record
{
  uint32          gid;
  Offset32<Paint> paint;
};

typedef ArrayOf<LayerV1Record, uint32> LayerV1Array;

struct BaseGlyphV1Record
{
  uint32                 gid;
  Offset32<LayerV1Array> layers;
};

typedef ArrayOf<BaseGlyphV1Record, uint32> BaseGlyphV1Array;

struct COLRv1
{
  // Version-0 fields
  uint16                                            version;
  uint16                                            numBaseGlyphsV0;
  Offset32<SortedUnsizedArrayOf<BaseGlyphRecordV0>> baseGlyphsV0;
  Offset32<UnsizedArrayOf<LayerRecordV0>>           layersV0;
  uint16                                            numLayersV0;
  // Version-1 additions
  Offset32<BaseGlyphV1Array>                        baseGlyphsV1;
  Offset32<ItemVariationStore>                      varStore;
};

```

# Implementation

## Font Tooling

Cosimo ([@anthrotype](https://github.com/anthrotype)) and Rod ([@rsheeter](https://github.com/rsheeter))
have implemented [nanoemoji](https://github.com/googlefonts/nanoemoji) to compile a set of SVGs into color
font formats, including COLR v1.

[color-fonts](https://github.com/googlefonts/color-fonts) has a collection of sample color fonts.

## Rendering

### FreeType

FreeType API extensions needed for a) rasterisation b ) as well as API exposing
all the details of the gradients so clients can *render* them as they wish, by
encoding as gradients in PDF output for example.

### Chromium

Prototype an implementation inside Skia's FreeType based COLR/CPAL
implementation extending solid color fills with gradient fills, after extracting
gradient fill implementation from FreeType. See
[`SkFonstHost_FreeType_common.cpp`](https://cs.chromium.org/chromium/src/third_party/skia/src/ports/SkFontHost_FreeType_common.cpp?q=fonthost+common&sq=package:chromium&dr=C&l=433)

## HarfBuzz

HarfBuzz implementation will follow later.  No major client relies on HarfBuzz
for color fonts currently, but we certainly want to implement later as there are
clients who like to remove FreeType dependency completely.

# Acknowledgements

Thanks to Benjamin Wagner ([@bungeman](https://github.com/bungeman)), Dave
Crossland ([@davelab6](https://github.com/davelab6)), and Roderick Sheeter
([@rsheeter](https://github.com/rsheeter)) for review and detailed feedback on
earlier proposal.
