# web-punchcard

> See this tool in context on the [blog post](https://kishlay-notabot.github.io/reviving-ibm-punch-card.html).

## Card Punch Writer - Implementation Process

## 1. The Goal

I had a scanned image of an IBM punch card (`punch.png`, 2212×974 px) and a
transparent hole cutout (`hole.png`, 40×68 px). I wanted to build a tool where
you type text and it punches the correct holes on the card image in real time.

To do that, I first needed to know the exact pixel position of every hole
center on this particular scan - all 960 of them (80 columns × 12 rows).

---

## 2. The Grid Discovery Tool

The first tool built wasn't the writer - it was a grid explorer.
It loaded the scanned card and attempted to automatically find all the hole
positions.

### Step 1: Automatic Blob Detection

The card image has existing printed holes (from the original physical card).
I used flood-fill to detect every bright spot:

```
For every pixel on the image:
    If it's bright (R>220, G>220, B>220) and unvisited:
        Flood-fill outward to collect all connected bright pixels
        If the blob is large enough (>20 pixels):
            Save its center of mass (centroid x, y)
```

This found most holes. The raw blobs were then sorted by Y to group them
into rows:

```js
blobs.sort((a, b) => a.y - b.y);

const rows = [];
const tolerance = 12;

for (const b of blobs) {
    for (const r of rows) {
        if (Math.abs(r.y - b.y) < tolerance) {
            r.points.push(b);
            r.y = average of all points in this row;
            found = true;
            break;
        }
    }
    if (!found) rows.push({ y: b.y, points: [b] });
}
```

### Step 2: Anchoring with Reference Rows

Not every row in the image has all 80 holes punched. But I identified two
rows that were fully punched (or nearly so) - these became my reference rows.
I used these as anchors to compute the column spacing and row spacing of the
grid.

```
Row A:  ●──●──●──●──●──●──●──●──●──● ...
        │  │  │  │  │  │  │  │  │  │
        colSpacing = average(x[i+1] - x[i])
        │  │  │  │  │  │  │  │  │  │
Row B:  ●──●──●──●──●──●──●──●──●──● ...

rowSpacing = average(rowB[i].y - rowA[i].y)
```

This gave me a working grid. The tool drew green crosshairs at every
computed intersection and let me click any point to see its column, row,
and pixel coordinates.

### Step 3: Visual Verification

With the grid drawn over the image, I could see if the intersections
aligned with the actual holes. Clicking around confirmed the spacing was
correct. The tool also generated the full JSON export:

```json
[
  [
    { "column": 0, "row": 0, "x": 72.00, "y": 79.00 },
    { "column": 1, "row": 0, "x": 98.09, "y": 79.00 },
    ...
  ],
  ...
]
```

---

## 3. From Exploration to Fixed Coordinates

The blob detection worked for exploration but was too fragile for the final
tool - it depended on the specific scan and could vary. Instead, I identified
the four corner holes visually from the image and recorded their approximate
pixel ranges:

```
Top-left  hole (col 0, row 0):  (72,   79)
Top-right hole (col 79, row 0): (2133, 79)

    (72,79) ●─────────────────────────────● (2133,79)
            │                             │
            │        Card image           │
            │    2212 × 974 px            │
            │                             │
   (72,897) ●─────────────────────────────● (2133,897)

Bottom-left  hole (col 0, row 11): (72, 897)
Bottom-right hole (col 79, row 11):(2133, 897)
```

These define the bounding box of all 960 holes. The spacing became a simple
computation:

```
colSpacing = (2133 - 72) / 79  ≈ 26.09 px
rowSpacing = (897 - 79) / 11   ≈ 74.36 px

x(col) = 72 + col × colSpacing
y(row) = 79 + row × rowSpacing
```

The exploration tool was updated to use these hard-coded corners instead
of blob detection - making it deterministic and removing the image-analysis
dependency. It kept the click-to-inspect and JSON export features.

---

## 4. The Card Punch Writer (`cardpunch-live.html`)

With the grid coordinates solved, I took inspo from this [blogpost](https://digitaltrails.github.io/punchedcardreader/js-cardpunch.html) and mixed them both.

### The IBM 029 Character Map

Each character is encoded as a 12-bit integer, one bit per row. Row 0 (the
12-edge) is the most significant bit (bit 11). Row 11 (the 9-edge) is the
least significant bit (bit 0).

```
Bits:   11  10   9   8   7   6   5   4   3   2   1   0
Rows:    0   1   2   3   4   5   6   7   8   9  10  11

"A" = 1001 0000 0000   → holes at rows 0 and 3
"5" = 0000 0001 0000   → hole at row 6
```

The map covers 57 characters, stored in `charMap`:

```js
charMap["A"] = 0b100100000000
charMap["5"] = 0b000000010000
```

### Writing Pipeline

```
Type "HELLO"
  │
  ▼
For each column c with text:
  │
  ├─► Uppercase the character
  ├─► Look up its 12-bit pattern in charMap
  └─► For each row r (0..11):
        if bit (11-r) is set:
            draw hole.png centered at (col_x, row_y)
```

The hole.png is drawn centered at each active intersection:

```js
ctx.drawImage(hole, x - 20, y - 34);
```

An `input` event listener calls `render()` on every keystroke, which
redraws the background and overlays all holes - so holes update
instantly as you type or delete.

---

## 5. Key Constants

| Constant     | Value | Description |
|--------------|-------|-------------|
| `COLS`       | 80    | Columns per card |
| `ROWS`       | 12    | Rows per card |
| `originX`    | 72    | X of top-left hole center |
| `originY`    | 79    | Y of top-left hole center |
| `colSpacing` | ~26.09 | Horizontal spacing between columns |
| `rowSpacing` | ~74.36 | Vertical spacing between rows |
| `HOLE_W`     | 40    | `hole.png` width |
| `HOLE_H`     | 68    | `hole.png` height |

## 6. Grid Formula

```
colX(col) = 72 + col × ((2133 - 72) / 79)
rowY(row) = 79 + row × ((897 - 79) / 11)
```

---

## 7. Printed Text Above the Card

The original punch card had text printed above each column.
I took out a text strip from the same original punch card and used it as a 'sprite sheet' for the text. (`textlist2.png`, 1701×50 px).

### Sprite Sheet Layout

The glyphs are arranged horizontally in a single row:

```
startX = 2   startY = 0
width  = 25  height = 50
gap    = 1

sx = startX + index × (width + gap)
```

### Character Order

The sprite sheet maps to the following character sequence, with skipped
positions at sprite indices 50 and 56. Also I couldn't find 2 (backtick and caret) characters, They were not there in the original image. (1-indexed: positions 51 and 57):

```
0123456789
ABCDEFGHIJKLMNOPQRSTUVWXYZ
#,$.-@%*<-/+_)  [SKIP51]  |&>:;  [SKIP57]  '?"=!(,
```

### Integration

The text rendering runs after the punching loop in `render()`:

```
Type "HELLO"
  │
  ▼
Earlier: punch holes (unchanged)
  │
  ▼
For each column c with text:
  ├─► Uppercase the character
  ├─► Look up its sprite index in the lookup table
  │     (skipped positions 50 and 56 are handled automatically)
  └─► Crop glyph from textlist2.png at:
        sx = 2 + spriteIndex × 26
        sy = 0
        sw = 25   sh = 50
      Draw at:
        dx = columnCenter + textOffsetX
        dy = textOffsetY
```

A `charToSprite` Map is built automatically from the character sequence
using the skip-aware parser. The mapping accounts for the two unused
sprite indices.

### Constants

| Constant      | Value | Description |
|---------------|-------|-------------|
| `textOffsetX` | 5     | Horizontal offset for fine-tuning glyph centering |
| `textOffsetY` | 0     | Vertical offset from the top of the card |
| `SPRITE_W`    | 25    | Width of each glyph in the sprite sheet |
| `SPRITE_H`    | 50    | Height of each glyph in the sprite sheet |
| `SPRITE_X`    | 2     | X offset of the first glyph in the sheet |
| `SPRITE_GAP`  | 1     | Gap between adjacent glyph sprites |

### Unassigned Characters

Characters in `charMap` that have no corresponding sprite glyph:
- `` ` `` (backtick)
- `^` (caret)

These punch correctly but appear blank in the printed text. Their
punch patterns remain unaffected. (I could design them manually but I chose to leave it as is.)

### Unused Sprites

Sprite indices 50, 56, and 64 have glyphs in `textlist2.png` that are
not mapped to any typed character. Index 64 is an extra slot beyond
the defined character sequence (the sheet has 65 slots / indices 0–64,
of which 62 are mapped, 2 skipped, and 1 extra).
