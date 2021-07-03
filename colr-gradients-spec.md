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
  - [Font Tooling](#font-tooling)
  - [Chromium, Skia, FreeType support](#chromium-skia-freetype-support)
  - [C++ Structures](#c-structures)
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

## Font Tooling

Cosimo ([@anthrotype](https://github.com/anthrotype)) and Rod ([@rsheeter](https://github.com/rsheeter))
have implemented [nanoemoji](https://github.com/googlefonts/nanoemoji) to compile a set of SVGs into color
font formats, including COLR v1.

[color-fonts](https://github.com/googlefonts/color-fonts) has a collection of sample color fonts.

## Chromium, Skia, Freetype support

Chrome Canary as of version 90.0.4421.5 and above supports COLRv1 fonts when switching on the COLR v1 flag.

To enable the feature and experiment with it, follow these steps:
1. Download and open [Chrome Canary](https://www.google.com/chrome/canary/),
   make sure it's updated to a version equal or newer than 90.0.4421.5.
2. Go to [chrome://flags/#colr-v1-fonts](chrome://flags/#colr-v1-fonts) and
   enable the feature.

![Screenshot of Chrome flag settings page.](images/colrv1_enable.png)

__Skia__ support is in tip-of-tree Skia, [implementation details](https://source.chromium.org/chromium/chromium/src/+/master:third_party/skia/src/ports/SkFontHost_FreeType_common.cpp;l=375)

__FreeType__ support is in tip-of-tree FreeType, see [freetype.h](https://gitlab.freedesktop.org/freetype/freetype/-/blob/master/include/freetype/freetype.h#L4995) for API details.


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

// Index of the first variation record in the variation index map or
// variation store if no variation index map is present.
// A record with N variable fields finds them at VarIdBase+0, ...,+N-1
// Use 0xFFFFFFFF to indicate no variation.
typedef uint32 VarIdxBase;

template <typename T>
struct Variable
{
  T      value;
  VarIdx varIdx; // Use 0xFFFFFFFF to indicate no variation.
};

// Color structures

// The ColorIndex alpha is multiplied into the alpha of the CPAL entry
// (converted to float -- divide by 255) looked up using paletteIndex to
// produce a final alpha.
struct ColorIndex
{
  uint16     paletteIndex;
  F2DOT14 alpha; // Default 1.0. Values outside [0.,1.] reserved.
};

struct VarColorIndex
{
  uint16     paletteIndex;
  F2DOT14    alpha; // Default 1.0. Values outside [0.,1.] reserved. VarIdx varBase + 0.
  VarIdxBase varBase;
};

struct ColorStop
{
  F2DOT14 stopOffset;
  ColorIndex color;
};

struct VarColorStop
{
  F2DOT14 stopOffset; // VarIdx varBase + 0
  ColorIndex color; // VarIdx varBase + 1
  VarIdxBase varBase;
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

struct VarColorLine
{
  Extend             extend;
  ArrayOf<VarColorStop> stops;
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
  Fixed xx;
  Fixed yx;
  Fixed xy;
  Fixed yy;
  Fixed dx;
  Fixed dy;
};

struct VarAffine2x3
{
  Fixed xx;  // VarIdx varBase + 0
  Fixed yx;  // VarIdx varBase + 1
  Fixed xy;  // VarIdx varBase + 2
  Fixed yy;  // VarIdx varBase + 3
  Fixed dx;  // VarIdx varBase + 4
  Fixed dy;  // VarIdx varBase + 5
  VarIdBase varBase;
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
  uint32              firstLayerIndex;  // index into COLRv1::layerList
}

struct PaintSolid
{
  uint8      format; // = 2
  ColorIndex color;
};

struct PaintVarSolid
{
  uint8      format; // = 3
  VarColorIndex color;
};

struct PaintLinearGradient
{
  uint8                  format; // = 4
  Offset24<ColorLine>    colorLine;
  FWORD                  x0;
  FWORD                  y0;
  FWORD                  x1;
  FWORD                  y1;
  FWORD                  x2; // Normal; Equal to (x1,y1) in simple cases.
  FWORD                  y2;
};

struct PaintVarLinearGradient
{
  uint8                  format; // = 5
  Offset24<VarColorLine> colorLine;
  FWORD                  x0; // VarIdx varBase + 0
  FWORD                  y0; // VarIdx varBase + 1
  FWORD                  x1; // VarIdx varBase + 2
  FWORD                  y1; // VarIdx varBase + 3
  FWORD                  x2; // VarIdx varBase + 4. Normal; Equal to (x1,y1) in simple cases.
  FWORD                  y2; // VarIdx varBase + 5
  VarIdBase              varBase;
};

struct PaintRadialGradient
{
  uint8                  format; // = 6
  Offset24<ColorLine>    colorLine;
  FWORD                  x0;
  FWORD                  y0;
  UFWORD                 radius0;
  FWORD                  x1;
  FWORD                  y1;
  UFWORD                 radius1;
};

struct PaintVarRadialGradient
{
  uint8                  format; // = 7
  Offset24<VarColorLine> colorLine;
  FWORD                  x0; // VarIdx varBase + 0
  FWORD                  y0; // VarIdx varBase + 1
  UFWORD                 radius0; // VarIdx varBase + 2
  FWORD                  x1; // VarIdx varBase + 3
  FWORD                  y1; // VarIdx varBase + 4
  UFWORD                 radius1; // VarIdx varBase + 5
  VarIdBase              varBase;
};

struct PaintSweepGradient
{
  uint8                  format; // = 8
  Offset24<ColorLine>    colorLine;
  FWORD                  centerX;
  FWORD                  centerY;
  F2DOT14                startAngle; // 180° in counter-clockwise degrees per 1.0 of value
  F2DOT14                endAngle;   // 180° in counter-clockwise degrees per 1.0 of value
};

struct PaintVarSweepGradient
{
  uint8                  format; // = 9
  Offset24<VarColorLine> colorLine;
  FWORD                  centerX; // VarIdx varBase + 0
  FWORD                  centerY; // VarIdx varBase + 1
  F2DOT14                startAngle; // VarIdx varBase + 2. 180° in counter-clockwise degrees per 1.0 of value
  F2DOT14                endAngle; // VarIdx varBase + 3. 180° in counter-clockwise degrees per 1.0 of value
  VarIdBase              varBase;
};

// Paint a non-COLR glyph, filled as indicated by paint.
struct PaintGlyph
{
  uint8                  format; // = 10
  Offset24<Paint>        paint;
  uint16                 gid;    // not a COLR-only gid
                                 // shall be less than maxp.numGlyphs
}

struct PaintColrGlyph
{
  uint8                  format; // = 11
  uint16                 gid;    // shall be a COLR gid
}

struct PaintTransform
{
  uint8                  format; // = 12
  Offset24<Paint>        src;
  Offset24<Affine2x3>    transform;
};

struct PaintVarTransform
{
  uint8                  format; // = 13
  Offset24<Paint>        src;
  Offset24<VarAffine2x3> transform;
};

struct PaintTranslate
{
  uint8                  format; // = 14
  Offset24<Paint>        src;
  FWORD                  dx;
  FWORD                  dy;
};

struct PaintVarTranslate
{
  uint8                  format; // = 15
  Offset24<Paint>        src;
  FWORD                  dx; // VarIdx varBase + 0
  FWORD                  dy; // VarIdx varBase + 1
  VarIdBase              varBase;
};

struct PaintScale
{
  uint8                  format; // = 16
  Offset24<Paint>        src;
  F2DOT14                scaleX;
  F2DOT14                scaleY;
};

struct PaintVarScale
{
  uint8                  format; // = 17
  Offset24<Paint>        src;
  F2DOT14                scaleX; // VarIdx varBase + 0
  F2DOT14                scaleY; // VarIdx varBase + 1
  VarIdBase              varBase;
};

struct PaintScaleAroundCenter
{
  uint8                  format; // = 18
  Offset24<Paint>        src;
  F2DOT14                scaleX;
  F2DOT14                scaleY;
  FWORD                  centerX;
  FWORD                  centerY;
};

struct PaintVarScaleAroundCenter
{
  uint8                  format; // = 19
  Offset24<Paint>        src;
  F2DOT14                scaleX; // VarIdx varBase + 0
  F2DOT14                scaleY; // VarIdx varBase + 1
  FWORD                  centerX; // VarIdx varBase + 2
  FWORD                  centerY; // VarIdx varBase + 3
  VarIdBase              varBase;
};

struct PaintScaleUniform
{
  uint8                  format; // = 20
  Offset24<Paint>        src;
  F2DOT14                scale;
};

struct PaintVarScaleUniform
{
  uint8                  format; // = 21
  Offset24<Paint>        src;
  F2DOT14                  scale; // VarIdx varBase + 0
  VarIdBase              varBase;
};

struct PaintScaleUniformAroundCenter
{
  uint8                  format; // = 22
  Offset24<Paint>        src;
  F2DOT14                scale;
  FWORD                  centerX;
  FWORD                  centerY;
};

struct PaintVarScaleUniformAroundCenter
{
  uint8                  format; // = 23
  Offset24<Paint>        src;
  F2DOT14                scale; // VarIdx varBase + 0
  FWORD                  centerX; // VarIdx varBase + 1
  FWORD                  centerY; // VarIdx varBase + 2
  VarIdBase              varBase;
};

struct PaintRotate
{
  uint8                  format; // = 24
  Offset24<Paint>        src;
  F2DOT14                angle; // 180° in counter-clockwise degrees per 1.0 of value
};

struct PaintVarRotate
{
  uint8                  format; // = 25
  Offset24<Paint>        src;
  F2DOT14                angle; // VarIdx varBase + 0. 180° in counter-clockwise degrees per 1.0 of value
  VarIdBase              varBase;
};

struct PaintRotateAroundCenter
{
  uint8                  format; // = 26
  Offset24<Paint>        src;
  F2DOT14                angle; // 180° in counter-clockwise degrees per 1.0 of value
  FWORD                  centerX;
  FWORD                  centerY;
};

struct PaintVarRotateAroundCenter
{
  uint8                  format; // = 27
  Offset24<Paint>        src;
  F2DOT14                angle; // VarIdx varBase + 0. 180° in counter-clockwise degrees per 1.0 of value
  FWORD                  centerX; // VarIdx varBase + 1
  FWORD                  centerY; // VarIdx varBase + 2
  VarIdBase              varBase;
};

struct PaintSkew
{
  uint8                  format; // = 28
  Offset24<Paint>        src;
  F2DOT14                xSkewAngle; // 180° in counter-clockwise degrees per 1.0 of value
  F2DOT14                ySkewAngle; // 180° in counter-clockwise degrees per 1.0 of value
};

struct PaintVarSkew
{
  uint8                  format; // = 29
  Offset24<Paint>        src;
  F2DOT14                xSkewAngle; // VarIdx varBase + 0. 180° in counter-clockwise degrees per 1.0 of value
  F2DOT14                ySkewAngle; // VarIdx varBase + 1. 180° in counter-clockwise degrees per 1.0 of value
  VarIdBase              varBase;
};

struct PaintSkewAroundCenter
{
  uint8                  format; // = 30
  Offset24<Paint>        src;
  F2DOT14                xSkewAngle; // 180° in counter-clockwise degrees per 1.0 of value
  F2DOT14                ySkewAngle; // 180° in counter-clockwise degrees per 1.0 of value
  FWORD                  centerX;
  FWORD                  centerY;
};

struct PaintVarSkewAroundCenter
{
  uint8                  format; // = 31
  Offset24<Paint>        src;
  F2DOT14                xSkewAngle; // VarIdx varBase + 0. 180° in counter-clockwise degrees per 1.0 of value
  F2DOT14                ySkewAngle; // VarIdx varBase + 1. 180° in counter-clockwise degrees per 1.0 of value
  FWORD                  centerX; // VarIdx varBase + 2
  FWORD                  centerY; // VarIdx varBase + 3
  VarIdBase              varBase;
};

struct PaintComposite
{
  uint8                  format; // = 32
  Offset24<Paint>        src;
  CompositeMode          mode;   // If mode is unrecognized use COMPOSITE_CLEAR
  Offset24<Paint>        backdrop;
};

struct BaseGlyphPaintRecord
{
  uint16                gid;
  Offset32<Paint>       paint;  // Typically PaintColrLayers
};

// Entries shall be sorted in ascending order of the `glyphID` field of the `BaseGlyphPaintRecord`s.
typedef ArrayOf<BaseGlyphPaintRecord, uint32> BaseGlyphList;

// Only layers accessed via PaintColrLayers (format 1) need be encoded here.
typedef ArrayOf<Offset32<Paint>, uint32> LayerList;

struct COLRv1
{
  // Version-0 fields
  uint16                                            version;
  uint16                                            numBaseGlyphRecords;
  Offset32<SortedUnsizedArrayOf<BaseGlyphRecord>>   baseGlyphRecords;
  Offset32<UnsizedArrayOf<LayerRecord>>             layerRecords;
  uint16                                            numLayerRecords;
  // Version-1 additions
  Offset32<BaseGlyphList>                           baseGlyphList;
  Offset32<LayerList>                               layerList;
  Offset32<VarIdxMap>                               varIdxMap;  // May be NULL
  Offset32<ItemVariationStore>                      varStore;
};

```

## Rendering

### Pseudocode

```
Allocate a bitmap for the glyph according to extents of base glyph contours for gid
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
    7) PaintTransform
          saveLayer()
          apply transform
          call a) for paint
          restore
    8) PaintTranslate
          saveLayer()
          apply transform
          call a) for paint
          restore
    9) PaintScale
          saveLayer()
          apply transform
          call a) for paint
          restore
    10) PaintRotate
          saveLayer()
          apply transform
          call a) for paint
          restore
    11) PaintSkew
          saveLayer()
          apply transform
          call a) for paint
          restore
    12) PaintComposite
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
