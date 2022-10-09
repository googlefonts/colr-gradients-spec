# Proposed revisions to ISO/IEC 14492 5th Edition Working Draft

_These are proposed changes to the WD reflected in ISO/IEC JTC 1/SC 29/WG 03 N0631. The main focus of these proposed revisions is on clause 5.7.11, COLR – Color Table, based on insights gains from vendors implementing support for enhancements to the COLR table introduced in Amendment 2 to ISO/IEC 14496-22:2019._

_Summary of proposed revisions:_
- _In 5.7.11.1.2.2, clarifications are added regarding interaction between color line definitions and font variations. Also, guidance is provided for implementers working with application platforms that have limiting constraints on color stop offsets._
- _In 5.7.11.1.2.3, guidance is provided for implementers working with application platforms that support linear gradients but that have limiting constraints on color stop offsets._
- _In 5.7.11.1.2.4, details are added regarding expected behaviour for radial gradients when variation deltas are applied to the gradient definition&mdash;specifically, when deltas applied to circle radii result in negative values. Also, guidance is provided for implementers working with application platforms that support radial gradients but that have limiting constraints on color stop offsets._
- _In 5.7.11.1.2.5, it has been found that the description of sweep gradients was confusing for implementers, resulting in non-interoperable implementation behaviours. This was due in part to the semantics for the sweep gradient data diverging from the semantics of analogous data in other existing libraries and specifications, such as CSS. Changes are proposed to the semantics for gradient data described in 5.7.11.1.2.5 to better align with other specifications, and which implementers of OFF have found clearer and more conducive to interoperability._
- _Other editorial revisions are proposed._

_Instructions for proposed changes are presented in italic style. Blocks of proposed text content are presented in normal style._

---

_In **5.7.11.1.2.4**, make the following changes:_

_Rev1) In the first note ("NOTE: The following figures..."), delete the word "circular"._

_Rev2) In the paragraph after the note following figure 5.29 ("A sweep gradient is defined..."), replace "starting and ending angles" with "start and end angles"._

_Rev3) Replace the next nine paragraphs &mdash; from the paragraph that begins "The color line is aligned..." to the note that begins "NOTE: Because the sweep gradient is drawn..." &mdash; and interleaved figures 5.30 &ndash; 5.32 with the following:_

For sampling and interpolating colors, the color line is aligned to an
infinitely spiraling circular arc around the center point, with arbitrary
radius. The position 0° on the arc means the direction of the positive x-axis,
360° means one full counter-clockwise rotation.  The ColorLine's stop offset 0
is aligned with the start angle, and stop offset 1 is aligned with the end
angle, independent of which angle is larger.

Outside the defined interval of the ColorLine, the color value of a position on
the ColorLine is filled in depending on its extend mode. See 5.7.11.1.2.1 Color
Lines for more details. In effect, this means that the spiraling circular arc
can be sampled for colors outside the defined ColorLine interval.

To draw the sweep gradient, for each position along the circular arc, starting
from from 0° up to but not including 360°, a ray from the center outward is
painted with the color of color line where the ray intersects with the circular
arc at that particular angle. Angle positions on the spiraling circular arc
below 0° and equal to or above 360° are not sampled for drawing the rays.

If the ColorLine's extend mode is reflect or repeat and start and end angle are
equal, nothing shall be drawn.

NOTE: When the sweep gradient's ColorLine uses the repeat or reflect extend mode
and the angular distance between start and end angle is small, this results in
very high spatial-frequency transitions that can lead to Moiré patterns or other
display artifacts. (See Figure 5.28 where this effect is shown for a similar
case involving radial gradients).

Figure 5.30 illustrates a sweep gradient with the drawing direction progressing
from 0° to 360°. The gradient is specified with a start angle of 110° and end
angle of 230°. The color line is specified with a yellow stop at offset 0
(aligned to the start angle) and a red stop at offset 1 (aligned to the end
angle). The pad extend mode is used, hence the color for angles below 110° is
yellow, and the color for angles above 150° is red.

![A sweep gradient, from yellow to red, with a start angle of 110° and an end
angle of 230°.](images/colr_conic_gradient_start_stop_angles.r1.png)

**Figure 5.30 A sweep gradient with start angle of 110° and an end angle of
230°, using a `ColorLine` with color stops 0 and 1 for yellow and red and pad
extend mode.**

NOTE: If the start angle is less than or equal to the end angle, the color line
progresses from smaller to larger offsets in the counter-clockwise direction
along the circular arc. If the start angle is larger than the end angle,
however, the color line is inverted and progresses in the clockwise direction,
as illustrated in figure 5.31.

![A sweep gradient, from yellow to red, with start angle from 210° to 110°,
showing the inversion of the color
line.](images/colr_conic_gradient_inverted.png)

**Figure 5.31 A sweep gradient, from yellow to red, with start angle of 210° and
end angle of 110°, showing the inversion of the color line when the start angle
is larger than the end angle.**

Not more than one full rotation is drawn and there is no overlap in drawing for
angles outside the 0° and 360° range. However, start and end angles can be
positioned at angles below 0° and above 360°. Through that, and through how wide
the color line interval is defined, color stops may lie outside the 0° to 360°
circle. This has an effect on the computation of the gradient colors inside the
interval of 0° to 360°, but colors are not sampled from outside this interval.
See figure 5.32 for an example. The color line goes from yellow to red with a
start angle of 330° and an end angle of 400°. The color for angles lower than
330° is yellow due to the pad extend mode, then the color line transitions from
yellow at 330° to red at 400°, but only the color values up until 360° are
sampled for drawing.

![A sweep gradient, from yellow to red, with a start angle of 330° and an end
angle of 400°.](images/colr_conic_gradient_stop_angle_outside.png)

**Figure 5.32 A sweep gradient with start angle of 330° and an end angle of 400°,
using a color line with color stops 0 and 1 for yellow and red and extend mode
pad.**

NOTE: Because the sweep gradient is drawn from 0° to 360° a sharp transition may
occur at 0°. This can be mitigated by adjusting the color stops at the 0° and
360° position on the arc to have the same color. The location of the transition
axis can also be shifted by nesting the sweep gradient inside a
a rotation transformation (see 5.7.11.1.5).

_Rev4) In the following note ("NOTE: When a sweep gradient..."), replace the last sentence with the following:_

Thus, a transform can result in the color line
progressing in the opposite direction compared to the non-transformed gradient.

_In **5.7.11.2.6.5**, make the following revisions:_

_Rev5) Replace the first sentence of the descriptions for startAngle in the tables for both format 8 and format 9 to the following:_

Start of the angular range of the gradient: add 1.0 and multiply by 180° to retrieve counter-clockwise degrees.

_Rev6) Replace the descriptions for endAngle in the tables for both format 8 and format 9 to the following:_

End of the angular range of the gradient: add 1.0 and multiply by 180° to retrieve counter-clockwise degrees.

_Rev7) Add the following note at the end of the clause:_

NOTE: To allow for a representation of +360°, a bias of 1.0 is used in the
representation of start and end angles of sweep gradients. For example, an
F2DOT14 value of -2.0 (0x8000) represents -180°; an F2DOT14 value of 0.0
(0x0000) represents +180°; an F2DOT14 value of 0.25 (0x1000) represents +225°;
an F2DOT14 value of 1.0 (0x4000) represents +360°. However, a bias is not used
for representation of angles in rotate or skew transforms (5.7.11.2.5.11,
5.7.11.2.5.12).
