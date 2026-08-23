# Thermal Lesion Tracer

A single-file browser tool for outlining and measuring regions in false-colour thermal images. You load a screenshot from a thermal camera, set a threshold on the temperature scale, and the app finds the regions above that threshold, traces their outlines, measures them, and exports the results as CSV, a labelled PNG, or ImageJ `.roi` files.

Built for an axillary thermography workflow (hidradenitis lesions imaged with a lab thermal scale), but the modality, palette, and value range are all configurable, so it also handles bioluminescence, fluorescence, perfusion, pH, and oxygenation images.

**[Live demo](https://acader05.github.io/thermal-lesion-tracer/)** (see Deployment below)

---

## Running it

Download `index.html` and open it in any modern browser. That's the whole install. There is no build step, no server, no npm, and no external libraries. Everything runs client-side, so images never leave the machine they're opened on, which matters when the inputs are patient photos.

## Workflow

1. **Load an image.** Any PNG or JPEG screenshot with a false-colour scale.
2. **Pick a modality and palette.** Temperature (°C) with the "Swift" palette is the default. The palette is the app's link between colour and value, so it has to match the scale baked into the image.
3. **Set the range and threshold.** Type the minimum and maximum of the image's colourbar (for example -5 to +5 °C), then drag the marker on the scale strip. Pixels at or above the threshold light up as you drag.
4. **Detect regions.** Smoothing and minimum size filter out speckle. Candidates appear in a list and can be accepted one at a time or all at once.
5. **Or draw by hand.** Draw mode places outline points on click; clicking the first point again or pressing Enter closes the shape.
6. **Calibrate (optional).** Click both edges of a 1 cm sticker in the image to set a pixels-per-mm scale. Without it, areas are reported in pixels only.
7. **Export.** CSV of per-lesion measurements plus a summary row, a PNG with the outlines burned in, or ImageJ ROIs (`.roi` for one, `RoiSet.zip` for several).

---

## How it works

The core is a six-stage pipeline, marked with numbered comment banners in the source.

### 1. Colour scale to lookup table

The app needs an ordered list of colours running cold to hot. It builds one from the selected palette: `paletteToLut()` takes the palette's hex stops (11 for Swift, one per degree) and linearly interpolates them into a 256-entry LUT.

There is also a `detectColorbar()` / `sampleColorbar()` pair that reads the scale straight out of the image instead. It scans rows in the lower 45% of the picture looking for a strip whose hue genuinely sweeps across the spectrum, using hue spread and the difference between the left and right ends to reject vivid-but-flat bands of actual image content. The strip is then averaged vertically and resampled to 256 colours. It works, but the palette-driven path turned out to be more predictable in practice, so that's what the UI currently uses. The detector is still in the file.

### 2. Temperature map

`buildTempMap()` draws the image to an offscreen canvas, pulls the raw RGBA bytes with `getImageData()`, and for every pixel finds the nearest LUT entry by squared distance in RGB space. The index of that entry maps linearly onto the user's min/max range, giving a `Float32Array` of one value per pixel.

A parallel `Uint8Array` mask marks which pixels are trustworthy. Anything more than 60 units away from every colour on the scale is off-colormap (text overlays, skin outside the thermal region, UI chrome) and gets excluded, along with the colourbar strip itself.

Nearest-neighbour search over 256 entries per pixel is brute force. On a full-resolution photo it's a few million distance calculations, which is fine at load time but is why the temperature map is built once and cached rather than recomputed on every threshold change.

### 3. Live highlight

`buildHighlight()` writes a translucent white wash into a separate offscreen canvas wherever a pixel is valid and at or above the threshold. Because it only reads the cached temperature map, it's fast enough to rerun on every frame of a threshold drag. The render loop then stacks three layers onto the visible canvas: the image, the highlight, and the vector overlay of outlines and labels.

### 4. Region detection and contour tracing

`detectRegions()` runs four steps:

- Build a binary mask of pixels above threshold.
- Blur it. `gaussianBlurMask()` is a separable running-sum box blur, applied horizontally then vertically, which approximates a gaussian at a fraction of the cost. Re-thresholding the blurred mask at 0.5 removes isolated speckle and closes small gaps.
- Label connected components with an iterative 4-connected flood fill using an explicit stack. Recursion would blow the call stack on a region of any real size.
- Trace each label's boundary with `traceContour()`, a Moore neighbourhood walk that circles the region clockwise, and drop anything below the minimum pixel count.

Contours over 600 points are decimated, then `smoothSpline()` resamples them as a periodic Catmull-Rom spline. The smoothing slider controls how many samples are taken per segment, so the same control governs both blur strength and outline smoothness.

### 5. Measurement

Area comes from the shoelace formula, perimeter from summed segment lengths. Temperature statistics need per-pixel values inside the polygon, so `measureROI()` scans the bounding box and tests each pixel with a ray-casting point-in-polygon check, accumulating min, max, and mean. If a calibration exists, pixel areas are divided by pixels-per-mm squared to give mm² and cm².

### 6. Export

- **CSV** is assembled as text and handed to a data URI. It's written with a UTF-8 byte order mark so Excel renders °C, cm², and StO₂ instead of mojibake.
- **PNG** redraws the image and all outlines to an offscreen canvas at full resolution, so the export isn't limited by the on-screen preview size.
- **ImageJ ROI** was the interesting one. `makeImageJRoi()` writes ImageJ's undocumented binary `.roi` format by hand into an `ArrayBuffer`: the `Iout` magic bytes, a big-endian 64-byte header holding the polygon type and bounding box, then x and y coordinates as `Int16` offsets from the box origin, then a second header carrying the name. When there's more than one ROI, `makeZip()` builds a valid ZIP archive byte by byte (local file headers, a CRC-32 implemented from the standard table, central directory, end-of-central-directory record) using the STORE method, no compression. That avoids pulling in a zip library for what amounts to a few hundred bytes per file.

### State and UI

All application state lives in one plain object, `S`, holding the image, the LUT, the typed arrays, the accepted ROIs, and the current mode. Every handler mutates `S` and calls a render function; there is no framework and no virtual DOM. Each ROI carries its own stroke colour, stroke width, and fill opacity, so styles can be set per lesion or pushed to all of them at once.

The palette editor includes a small HSV colour picker drawn with canvas gradients, letting any stop in the palette be edited and reapplied, which rebuilds the LUT and the temperature map.

---

## Deployment

The repo is served with GitHub Pages. In the repository, go to Settings, then Pages, and set the source to the `main` branch and the root folder. GitHub then hosts the app at `https://acader05.github.io/thermal-lesion-tracer/`. The file is named `index.html`, so that URL opens it directly. Because the app is one static HTML file with no backend, that's all the deployment it needs.

## Limitations

- The value map is only as good as the palette match. If the image was rendered with a scale the app doesn't have, values will be systematically wrong even though the outlines look reasonable.
- Colours that appear twice on a scale are ambiguous by construction. Nearest-neighbour lookup picks one and cannot know which is right.
- Anti-aliased pixels along colour boundaries land between LUT entries and are either mapped to a neighbour or dropped by the validity mask.
- Measurements are pixel-based. Anything the camera's perspective distorts, the areas inherit.

## Tech

Vanilla JavaScript, HTML, and CSS. Canvas 2D, `ImageData`, typed arrays, `ArrayBuffer` and `DataView` for binary output, Blob and data URIs for downloads. No dependencies.

## License

MIT
