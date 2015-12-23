#BioHeatMap tooltips.

# Introduction #

The BioHeatMap displays 'heatmap' diagrams from dense tables of numeric data. The aim of this work is to extend the API and code-base to enable pop-up tooltips to present cell-specific additional information to the user.

The initial aims are as follows:

  * Implementation of a tooltip showing metadata for each cell of a BioHeatMap Google visualization when a user hovers the mouse over the cell.
  * Tooltip text data can be split into one or more lines using the newline character "\n".
  * The tooltip functionality can be turned on or off with a "displayCellToolTips" parameter, which defaults to true, enabling tooltip display.

# Initial Design #

## Alterations to the public API ##

Currently, the data to be displayed is published to BioHeatMap through the `setCell()` call. This will be extended to accept an optional fourth argument containing tooltip text.

```
data.setCell(1, 1, 3.5); // legacy call
data.setCell(1, 2, 3.2, "Note about this cell"); // simple tooltip
data.setCell(2, 1, 0.3, "This is a\nmulti-line comment"); // multi-line message
data.setCell(2, 2, 2.1, ""); // an empty tooltip
```

## Open questions ##

  1. Should empty tooltip text be normalized to null, so have no display, or should they display a small tooltip with no content?
  1. Is plain text the only content? Perhaps we should optionally allow nested HTML.

## Configuration ##

It would be nice to be able to configure aspects of the tooltip display. In any event, the display of tooltips will require some choices to be made about aspects such as the color of the background. These may include things like:

  * tooltip delays
    * delay before the tooltip displays
    * delay before the tooltip auto-hides (can be disabled)
  * tooltip box
    * the background color
    * border color, thickness, style, ...
  * tooltip text
    * font, size, color, ...
    * face, style, weight, ...
    * justification/alignment (right/left/centered), direction, ...

## Implementation choice ##

The tooltips themselves could be implemented either through the `canvas` API or by js/dhtml actions on a `div` element. The `div` element is probably easiest to implement, while `canvas` may be more customizable.

# The Implementation #

## Actual API ##

Now that I have started implementing this, some things have changed. The google data table API already has a 4-arg form of [`setCell()`](http://code.google.com/apis/visualization/documentation/reference.html#DataTable_setCell), where the fourth argument is a pre-formatted representation of the value. However, it also accepts a 5th argument that is an object (key-value pairs) of supplementary data. This seems like a good place to put our tooltip text.

```
var data = new google.visualization.DataTable();
data.addColumn('string', 'Gene Name');
data.addColumn('number', 'liver_1');
data.addColumn('number', 'liver_2');
data.addRows(2);
data.setCell(0, 0, 'gene_1');
data.setCell(0, 1, 1.0); // no tooltip
data.setCell(0, 2, 2.0, null, { tooltip: 'some interesting facts' }); // a simple tooltip
data.setCell(1, 0, null, 'gene_2');
data.setCell(1, 1, 3.0, null, { tooltip: 'some more\ninteresting facts' }); // a tooltip with newlines
data.setCell(1, 2, 2.0, null, { tooltip: '' } ); // an empty tooltip
```

This is not as pretty as I'd originally hoped, but it appears to allow tooltip text to be passed through without affecting existing Google visualization code.

## Implementation Details ##

The tooltip is represented by a div, stored in the `tooltipElement` property of the heat map. This div is managed by `initialize()`, `draw()` and `_setTooltipElement`. In `draw()`, `_getMouseXY` is used to register both the `onclick` and `onmousemove` handlers.

The `_getMouseXY` method has been significantly re-worked. It now takes a parameter called `handler` that is a function to be called with the position relative to the canvas and the mouse position. Closures for `_onClickEvent()` and `_onMoveEvent()` are passed to `getMouseXY` in `draw()` to attach them to the relevant canvas events.

In addition to the change in the `getMouseXY` signature, the implementation has been gutted. The old code was buggy. It didn't seem to correctly calculate the position of the mouse event relative to the heatmap element. I've liberally adapted code from quirksmode to fix this. I've not tested it in all browsers.

To support the mouse movement calls, I have added `_onMoveEvent`. This calculates the cell currently under the mouse and then modifies the `tooltipElement` div to display a tooltip.

## Styles ##

The tooltip div defaults to the CSS class `bioHeatMapTooltip`. A default class is supplied for this, but you can supply your own.

## Configuration Options ##

| Option | Default | Description |
|:-------|:--------|:------------|
| `tooltipClass` | `.bioHeatMapTooltip` | name of the style class to be applied to the tooltip div |
| `tooltipDelay` | `500`   | delay (in ms) before a tooltip pops up |
| `displayCellTooltips` | true    | heatmap-wide toggle for tooltip display - set to false to dissable tooltips, true (or any other value) to enable them |