# DQGUI Layout Design

## Overview

DQGUI is intended to be an OpenGL-based GUI toolkit with dynamic, DPI-independent layout.

The basic visual type is `OWidget`.

A window should preferably **contain a root widget/container** rather than inherit from `OWidget`:

```dq
object OWindow:
    scale_factor : float32
    root_widget  : OWidget
endobject
```

The window owns OS- and OpenGL-specific resources, while widgets form the GUI hierarchy.

---

## Logical Coordinates and Scaling

Widgets use logical coordinates:

```dq
x, y, w, h : float32
```

These values are independent of framebuffer pixels.

Each `OWindow` has its own scale factor:

```text
physical coordinate = logical coordinate * scale_factor
```

The layout engine works entirely in logical GUI units.

OpenGL rendering then maps logical coordinates to framebuffer coordinates.

This allows each window to have an independent DPI / scale factor.

---

## Computed Geometry

The widget fields

```dq
x, y, w, h : float32
```

represent the **computed geometry**.

They should not necessarily represent the widget's requested size.

A widget may also have preferred or constrained dimensions, for example:

```text
preferred width / height
minimum width / height
maximum width / height
```

The layout engine recalculates `x`, `y`, `w`, and `h` whenever the parent geometry or relevant layout properties change.

A dirty-layout mechanism can avoid unnecessary recalculation:

```text
property change
    ->
layout marked dirty
    ->
layout recalculated once before rendering
```

---

## Widgets, Containers, and Layouts

It is useful to distinguish ordinary widgets from containers.

Conceptually:

```text
OWidget
    ^
    |
OContainer
```

An ordinary widget contains geometry, rendering, input handling, sizing constraints, and a parent reference.

A container additionally owns child widgets.

Example:

```text
OWidget
    geometry
    sizing constraints
    rendering
    input
    parent

OContainer : OWidget
    children[]
    layout
```

The layout algorithm itself should remain separate from the container.

This is similar in spirit to the wxWidgets distinction between windows/controls and sizers.

A container **has a layout**, rather than necessarily being a particular kind of layout.

---

## Row-Based Layout as the Default

Instead of forcing the complete form into a rigid grid, the default layout can consist of independent horizontal rows.

Conceptually:

```text
OLayout
    Row 0
    Row 1
    Row 2
    ...
```

Each row is an independent horizontal sizer.

Example:

```text
Row 0: [Label] [Edit]
Row 1: [Label] [Edit]
Row 2: [Checkbox]
Row 3:                     [Cancel] [OK]
```

This fits desktop forms naturally because different rows often contain different numbers of widgets.

A bottom button row therefore does not require dummy grid cells.

---

## Basic Construction API

A simple initial API could be:

```dq
object OLayout:
    cells  : [*][*]OWidget
    currow : int = 0

    func NewRow():
        currow += 1
        cells.Append([])
    endfunc

    func AddWidget(aw : OWidget):
        cells[currow].Append(aw)
        return aw
    endfunc
endobject
```

The first row should exist when the layout is constructed.

Example use:

```dq
var layout : OLayout

layout.AddWidget(new OLabel(100, "something"))
ed1 = layout.AddWidget(new OEdit(200, ""))

layout.NewRow()

layout.AddWidget(new OLabel(100, "second"))
ed2 = layout.AddWidget(new OEdit(300, ""))
```

This produces two independent rows:

```text
[ Label ] [ Edit ]
[ Label ] [ Edit ]
```

The code visually resembles the resulting form.

---

## Optional Grid Alignment

Independent rows are flexible, but form fields often need aligned columns.

Instead of making the entire layout a grid, selected regions can opt into grid-like alignment.

Proposed API:

```dq
AlignCells(arow : int, arowcount : int,
           acol : int, acolcount : int)
```

The arguments use **start + count**, which avoids ambiguity about inclusive or exclusive end indices.

Example:

```dq
AlignCells(0, 3, 0, 2)
```

means:

```text
rows:    0, 1, 2
columns: 0, 1
```

The selected cells share column geometry.

For example:

```text
Row 0: [ Name       ] [ Edit ]
Row 1: [ Address    ] [ Edit ]
Row 2: [ Country    ] [ Edit ]
```

could become:

```text
+-------------+--------------------------+
| Name        | Edit                     |
| Address     | Edit                     |
| Country     | Edit                     |
+-------------+--------------------------+
```

The width of the first aligned column could be the maximum preferred width of all labels in the aligned region.

Rows outside this region remain independent.

Example:

```text
Row 3:                    [ Cancel ] [ OK ]
```

does not need to participate in the grid.

---

## Flexible Rows and Columns

The layout also needs a way to specify which rows or columns consume remaining space.

Proposed API:

```dq
FlexRows(arow : int, arowcount : int)
FlexCols(acol : int, acolcount : int)
```

Example:

```dq
FlexRows(1, 1)
```

could describe:

```text
row 0: toolbar       intrinsic/fixed height
row 1: main content  takes remaining height
row 2: buttons       intrinsic/fixed height
```

Result:

```text
+----------------------------+
| Toolbar                    |
+----------------------------+
|                            |
|       flexible area        |
|                            |
+----------------------------+
|               Cancel   OK  |
+----------------------------+
```

Likewise:

```dq
FlexCols(1, 1)
```

is useful for a typical form row:

```text
[ Label ][ Edit........................ ]
```

where the label column keeps its intrinsic width and the edit column consumes the remaining width.

If multiple rows or columns are marked flexible, the initial implementation can divide the remaining space equally.

Weighted flex distribution can be added later if needed.

---

## Flexibility vs Widget Stretching

A flexible row or column and widget stretching are separate concepts.

For example:

```text
flexible row
```

means:

> the row may receive extra available height.

Whereas:

```text
widget stretch
```

means:

> the widget expands to fill the geometry assigned to its cell.

These should remain separate layout properties.

---

## Why Not Use a Mandatory Grid Everywhere?

A pure grid model can express most forms, because:

```text
row layout    = grid with one row
column layout = grid with one column
```

However, a mandatory grid becomes awkward when rows contain different numbers of controls.

For example:

```text
[ Label ] [ Edit ]
[ Label ] [ Edit ]
[ Checkbox ]
                    [ Cancel ] [ OK ]
```

The row-based model keeps such forms simple.

Grid behavior is then enabled only where useful using `AlignCells()`.

This provides:

```text
independent row sizing
        +
optional shared column alignment
        +
flexible rows/columns
```

without the complexity of a full CSS-style layout engine.

---

## Relation to Existing GUI Systems

The design has similarities to wxWidgets:

```text
wxWindow / wxControl
        +
wxSizer
```

DQGUI can use the same broad separation:

```text
OWidget / OContainer
        +
OLayout
```

However, the proposed DQGUI layout model can be simpler than wxWidgets' deeply nested sizer trees.

Instead of creating a horizontal sizer for every form row and nesting them in a vertical sizer, an `OLayout` directly contains rows.

Selected rows and columns can then be aligned through `AlignCells()`.

---

## Possible Future Extensions

The initial API can remain intentionally small:

```dq
AddWidget(...)
NewRow()

AlignCells(row, rowcount, col, colcount)

FlexRows(row, rowcount)
FlexCols(col, colcount)
```

Possible later additions include:

```text
cell margins
row/column spacing
horizontal alignment
vertical alignment
widget stretch
row span
column span
minimum / maximum size
weighted flex values
nested layouts
spacers
```

If cell-specific metadata becomes necessary, raw widget references could later be wrapped in an `OLayoutCell` type:

```dq
object OLayoutCell:
    widget   : OWidget
    rowspan  : int = 1
    colspan  : int = 1
    align_x
    align_y
    margin
endobject
```

This does not need to be part of the initial implementation.

---

## Current Design Direction

The current preferred model is:

```text
OWindow
    scale_factor
    root_widget

OWidget
    computed x, y, w, h
    preferred/min/max sizing
    rendering
    input handling

OContainer : OWidget
    children
    layout

OLayout
    independent horizontal rows
    AddWidget()
    NewRow()
    AlignCells()
    FlexRows()
    FlexCols()
```

The main design principle is:

> Keep ordinary layout simple and row-oriented, and introduce grid-like behavior only where explicit alignment is required.

This should provide a compact API for manually written DQ GUI code while still supporting complex, dynamically resizable desktop forms.
