# Gradients for COLR/CPAL Fonts

December 2019

### Authors:
* Behdad Esfahbod ([@behdad](https://github.com/behdad))
* Dominik Röttsches ([@drott](https://github.com/drott))
* Roderick Sheeter ([@rsheeter](https://github.com/rsheeter))

## Table of Contents

- [Introduction](#introduction)
- [Extending ISO/IEC 14496-22 Open Font Format](#extending-isoiec-14496-22-open-font-format)
- [Implementation](#implementation)
  - [C++ Structures](#c-structures)
  - [Font Tooling](#font-tooling)
  - [Chromium, Skia, FreeType support](#chromium-skia-freetype-support)
  - [Rendering](#rendering)
    - [Pseudocode](#pseudocode)
  - [HarfBuzz](#harfbuzz)
- [References](#references)
- [Acknowledgements](#acknowledgements)

# Introduction

We propose an extension of the COLR table to support greater graphic
capabilities. The current version number of COLR table is 0. We propose this as
COLR table format version 1.

The initial design proposal aimed to add support for gradient fills, in addition
to the existing solid color fills, and to integrate variation into color formats
in variable fonts. As the design work progressed, it seemed appropriate to
introduce other capabilities:

* Different composition modes
* Affine transformations
* Flexible means of re-use of components to provide size reduction

While extending the design for these additional capabilities, it also seemed
appropriate to change the basic structure for color glyph descriptions from a
vector of layered elements to be a directed, acyclic graph.

It is our understanding that this brings the capabilities of COLR/CPAL to nearly
match those of [SVG Native](https://svgwg.org/specs/svg-native/) for vector
graphics.  SVG Native allows embedding PNG and JPEG images while this proposal
does not.  We'd like to explore in the future, how COLR/CPAL can be mixed with
sbix to address that limitation as well.

The aim is for version 1 of the COLR table to be incorporated into a future
version of the OpenType spec and into an update of the ISO/IEC 14496-22 Open
Font Format (OFF) standard.

Earlier revisions of this document provided many details regarding the proposed
COLR extensions. These have since been moved and elaborated with full details in
a separate document for submission to OFF—see the next section.

# Extending ISO/IEC 14496-22 Open Font Format

The COLR version 1 enhancements have been proposed for incorporation into the
ISO/IEC 14496-22 Open Font Format standard. In ISO process, this would be done
as an amendment (Amendment 2) of the current edition of that standard. Content
for a working draft of that amendment has been prepared in a separate doc:
[Proposed changes to ISO/IEC 14496-22 (Amendment 2)](OFF_AMD2_WD.md).

# Implementation

**This section is NOT meant for ISO submissions**

## C++ Structures

The following provides a C++ implementation of the structures defined above.

```C++
// Base template types

template <typename T, typename Length=uint16>
struct ArrayOf
{
  Length count;
  T      array[/*count*/];
};

typedef uint32 VarIdx;

template <typename T>
struct Variable
{
  T      value;
  VarIdx varIdx; // Use 0xFFFFFFFF to indicate no variation.
};

// Variation structures

typedef Variable<FWORD> VarFWORD;

typedef Variable<UFWORD> VarUFWORD;

typedef Variable<Fixed> VarFixed;

typedef Variable<F2DOT14> VarF2DOT14;

// Color structures

// The ColorIndex alpha is multiplied into the alpha of the CPAL entry
// (converted to float -- divide by 255) looked up using paletteIndex to
// produce a final alpha.
struct ColorIndex
{
  uint16     paletteIndex;
  VarF2DOT14 alpha; // Default 1.0. Values outside [0.,1.] reserved.
};

struct ColorStop
{
  VarF2DOT14 stopOffset;
  ColorIndex color;
};

enum Extend : uint8
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


// Composition modes

// Compositing modes are taken from https://www.w3.org/TR/compositing-1/
// NOTE: a brief audit of major implementations suggests most support most
// or all of the specified modes.
enum CompositeMode : uint8
{
  // Porter-Duff modes
  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators
  COMPOSITE_CLEAR          =  0,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_clear
  COMPOSITE_SRC            =  1,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_src
  COMPOSITE_DEST           =  2,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_dst
  COMPOSITE_SRC_OVER       =  3,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_srcover
  COMPOSITE_DEST_OVER      =  4,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_dstover
  COMPOSITE_SRC_IN         =  5,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_srcin
  COMPOSITE_DEST_IN        =  6,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_dstin
  COMPOSITE_SRC_OUT        =  7,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_srcout
  COMPOSITE_DEST_OUT       =  8,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_dstout
  COMPOSITE_SRC_ATOP       =  9,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_srcatop
  COMPOSITE_DEST_ATOP      = 10,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_dstatop
  COMPOSITE_XOR            = 11,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_xor
  COMPOSITE_PLUS           = 12,  // https://www.w3.org/TR/compositing-1/#porterduffcompositingoperators_plus

  // Blend modes
  // https://www.w3.org/TR/compositing-1/#blending
  COMPOSITE_SCREEN         = 13,  // https://www.w3.org/TR/compositing-1/#blendingscreen
  COMPOSITE_OVERLAY        = 14,  // https://www.w3.org/TR/compositing-1/#blendingoverlay
  COMPOSITE_DARKEN         = 15,  // https://www.w3.org/TR/compositing-1/#blendingdarken
  COMPOSITE_LIGHTEN        = 16,  // https://www.w3.org/TR/compositing-1/#blendinglighten
  COMPOSITE_COLOR_DODGE    = 17,  // https://www.w3.org/TR/compositing-1/#blendingcolordodge
  COMPOSITE_COLOR_BURN     = 18,  // https://www.w3.org/TR/compositing-1/#blendingcolorburn
  COMPOSITE_HARD_LIGHT     = 19,  // https://www.w3.org/TR/compositing-1/#blendinghardlight
  COMPOSITE_SOFT_LIGHT     = 20,  // https://www.w3.org/TR/compositing-1/#blendingsoftlight
  COMPOSITE_DIFFERENCE     = 21,  // https://www.w3.org/TR/compositing-1/#blendingdifference
  COMPOSITE_EXCLUSION      = 22,  // https://www.w3.org/TR/compositing-1/#blendingexclusion
  COMPOSITE_MULTIPLY       = 23,  // https://www.w3.org/TR/compositing-1/#blendingmultiply

  // Modes that, uniquely, do not operate on components
  // https://www.w3.org/TR/compositing-1/#blendingnonseparable
  COMPOSITE_HSL_HUE        = 24,  // https://www.w3.org/TR/compositing-1/#blendinghue
  COMPOSITE_HSL_SATURATION = 25,  // https://www.w3.org/TR/compositing-1/#blendingsaturation
  COMPOSITE_HSL_COLOR      = 26,  // https://www.w3.org/TR/compositing-1/#blendingcolor
  COMPOSITE_HSL_LUMINOSITY = 27,  // https://www.w3.org/TR/compositing-1/#blendingluminosity
};

// Affine 2D transformations

// This is a standard 2x3 matrix for 2D affine transformation.
struct Affine2x3
{
  VarFixed xx;
  VarFixed yx;
  VarFixed xy;
  VarFixed yy;
  VarFixed dx;
  VarFixed dy;
};

// Paint tables

// Each layer is composited on top of previous with mode COMPOSITE_SRC_OVER.
// NOTE: uint8 size saves bytes in most cases and does not
// preclude use of large layer counts via PaintComposite or a tree
// of PaintColrLayers.
struct PaintColrLayers
{
  uint8               format; // = 1
  uint8               numLayers;
  uint32              firstLayerIndex;  // index into COLRv1::layersV1
}

struct PaintSolid
{
  uint8      format; // = 2
  ColorIndex color;
};

struct PaintLinearGradient
{
  uint8               format; // = 3
  Offset24<ColorLine> colorLine;
  VarFWORD            x0;
  VarFWORD            y0;
  VarFWORD            x1;
  VarFWORD            y1;
  VarFWORD            x2; // Normal; Equal to (x1,y1) in simple cases.
  VarFWORD            y2;
};

struct PaintRadialGradient
{
  uint8               format; // = 4
  Offset24<ColorLine> colorLine;
  VarFWORD            x0;
  VarFWORD            y0;
  VarUFWORD           radius0;
  VarFWORD            x1;
  VarFWORD            y1;
  VarUFWORD           radius1;
};

struct PaintSweepGradient
{
  uint8               format; // = 5
  Offset24<ColorLine> colorLine;
  VarFWORD            centerX;
  VarFWORD            centerY;
  VarFixed            startAngle;
  VarFixed            endAngle;
};

// Paint a non-COLR glyph, filled as indicated by paint.
struct PaintGlyph
{
  uint8               format; // = 6
  Offset24<Paint>     paint;
  uint16              gid;    // not a COLR-only gid
                              // shall be less than maxp.numGlyphs
}

struct PaintColrGlyph
{
  uint8               format; // = 7
  uint16              gid;    // shall be a COLR gid
}

struct PaintTransformed
{
  uint8               format; // = 8
  Offset24<Paint>     src;
  Affine2x3           transform;
};

struct PaintTranslate
{
  uint8               format; // = 9
  Offset24<Paint>     src;
  VarFixed            dx;
  VarFixed            dy;
};

struct PaintRotate
{
  uint8               format; // = 10
  Offset24<Paint>     src;
  VarFixed            angle;
  VarFixed            centerX;
  VarFixed            centerY;
};

struct PaintSkew
{
  uint8               format; // = 11
  Offset24<Paint>     src;
  VarFixed            xSkewAngle;
  VarFixed            ySkewAngle;
  VarFixed            centerX;
  VarFixed            centerY;
};

struct PaintComposite
{
  uint8               format; // = 12
  Offset24<Paint>     src;
  CompositeMode       mode;   // If mode is unrecognized use COMPOSITE_CLEAR
  Offset24<Paint>     backdrop;
};

struct BaseGlyphV1Record
{
  uint16                gid;
  Offset32<Paint>       paint;  // Typically PaintColrLayers
};

// Entries shall be sorted in ascending order of the `glyphID` field of the `BaseGlyphV1Record`s.
typedef ArrayOf<BaseGlyphV1Record, uint32> BaseGlyphV1List;

// Only layers accessed via PaintColrLayers (format 1) need be encoded here.
typedef ArrayOf<Offset32<Paint>, uint32> LayerV1List;

struct COLRv1
{
  // Version-0 fields
  uint16                                            version;
  uint16                                            numBaseGlyphsV0;
  Offset32<SortedUnsizedArrayOf<BaseGlyphRecordV0>> baseGlyphsV0;
  Offset32<UnsizedArrayOf<LayerRecordV0>>           layersV0;
  uint16                                            numLayersV0;
  // Version-1 additions
  Offset32<BaseGlyphV1List>                         baseGlyphsV1;
  Offset32<LayerV1List>                             layersV1;
  Offset32<ItemVariationStore>                      varStore;
};

```

## Font Tooling

Cosimo ([@anthrotype](https://github.com/anthrotype)) and Rod ([@rsheeter](https://github.com/rsheeter))
have implemented [nanoemoji](https://github.com/googlefonts/nanoemoji) to compile a set of SVGs into color
font formats, including COLR v1.

[color-fonts](https://github.com/googlefonts/color-fonts) has a collection of sample color fonts.

## Chromium, Skia, Freetype support

Early work on Chrome support is underway:

* Intent to prototype [announcement](https://groups.google.com/a/chromium.org/g/blink-dev/c/XIXudwZdr44/m/emDJRTFoAwAJ)
* WIP Skia [implementation](https://skia-review.googlesource.com/c/skia/+/300558)
* FreeType can read COLR v1 information as of [268bdd7764e6ab4029f](https://gitlab.freedesktop.org/freetype/freetype/-/commit/268bdd7764e6ab4029fe7f8bc35626294d791c6d)

## Rendering

### Pseudocode

```
Allocate a bitmap for the glyph according to glyf table entry extents for gid
0) Start at base glyph paint.
 a) Paint a paint, switch:
    1) PaintColrLayers
         Paint each referenced layer by performing a)
    2) PaintSolid
          SkCanvas::drawColor with color configured
    3) PaintLinearGradient
          SkCanvas::drawPaint with liner gradient configured
          (expected to be bounded by parent composite mode or clipped by current clip, check bounds?)
    4) PaintRadialGradient
          SkCanvas::drawPaint with radial gradient configured
          (expected to be bounded by parent composite mode or clipped by current clip, check bounds?)
    5) PaintGlyph
         gid must not COLRv1
         saveLayer
         setClipPath to gid path
           recurse to a)
         restore
    6) PaintColrGlyph
         gid must be from
         if gid on recursion blacklist, do nothing
         recurse to 0) with different gid
    7) PaintTransformed
          saveLayer()
          apply transform
          call a) for paint
          restore
    8) PaintTranslate
          saveLayer()
          apply transform
          call a) for paint
          restore
    9) PaintRotate
          saveLayer()
          apply transform
          call a) for paint
          restore
    10) PaintSkew
          saveLayer()
          apply transform
          call a) for paint
          restore
    11) PaintComposite
          paint Paint for backdrop, call a)
          saveLayer() with setting composite mode, on SkPaint
          paint Paint for src, call a)
          restore with save composite mode
```

## HarfBuzz

HarfBuzz implementation will follow later.  No major client relies on HarfBuzz
for color fonts currently, but we certainly want to implement later as there are
clients who like to remove FreeType dependency completely.

# References

* <a name="2d-graphics-libraries">2D Graphics Libraries</a>
   * [Cairo](https://cairographics.org/)
   * [Skia](https://skia.org/)
   * [CoreGraphics](https://developer.apple.com/documentation/coregraphics)
   * [Direct2D](https://docs.microsoft.com/en-us/windows/win32/direct2d/direct2d-portal)

# Acknowledgements

Thanks to Benjamin Wagner ([@bungeman](https://github.com/bungeman)), Dave
Crossland ([@davelab6](https://github.com/davelab6)), and Roderick Sheeter
([@rsheeter](https://github.com/rsheeter)) for review and detailed feedback on
earlier proposal.
- [Gradients for COLR/CPAL Fonts](#gradients-for-colrcpal-fonts)
    - [Authors:](#authors)
  - [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
- [Extending ISO/IEC 14496-22 Open Font Format](#extending-isoiec-14496-22-open-font-format)
- [Implementation](#implementation)
  - [C++ Structures](#c-structures)
  - [Font Tooling](#font-tooling)
  - [Rendering](#rendering)
    - [Pseudocode](#pseudocode)
    - [FreeType](#freetype)
    - [Chromium](#chromium)
  - [HarfBuzz](#harfbuzz)
- [References](#references)
- [Acknowledgements](#acknowledgements)
