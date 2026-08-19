# `src/actions/draw-tool.js`

## 1. What it does

A **flat action-creator registry**. 130 lines, 116 `export const` creators, and nothing else — every line is the same shape:

```js
import { createAction } from 'redux-actions';
export const zoomIn = createAction('ZOOM_IN');
```

`createAction(type)` from `redux-actions` returns `payload => ({ type, payload })`. There is:

| | |
| --- | --- |
| **No logic** | no payload creators, no validation, no defaults — whatever you pass becomes `action.payload` verbatim |
| **No async** | no thunks, no sagas, no API calls. Async lives in `src/containers/App/index.jsx` and `src/middleware/product.js` |
| **No shared type constants** | each string is written once, here. Reducers and middleware match it by literal string, so a typo is silent |
| **One import** | `redux-actions` |

**Where the actions go.** Dispatch order is set in [src/store/index.js](../src/store/index.js#L11-L15):

```
dispatch(action) → logger → middleware/draw-tool.js → middleware/product.js → reducers/draw-tool.js
```

The middleware mutates the fabric canvas **before** the store is updated, and a bare `return;` in either middleware stops the action from ever reaching the reducer.

**Coverage of the 116 creators:**

| Handled by | Count | Meaning |
| --- | --- | --- |
| Reducer only | 60 | pure store state — UI flags, catalogue data, selections |
| Reducer + `middleware/draw-tool.js` | 27 | store update **and** a canvas side effect |
| `middleware/draw-tool.js` only | 21 | pure canvas command, leaves no trace in the store |
| `middleware/product.js` only | 2 | `updateFontsAl`, `updateFontsByName` — web-font loading |
| **Nothing at all** | **6** | `setStepAcrylic`, `selectLaserErase`, `selectShape`, `selectText`, `selectTextOff`, `upload` |

**Consumers:** [containers/App](../src/containers/App/index.jsx), [containers/Options](../src/containers/Options/index.jsx) (the biggest by far), [Toolbar](../src/containers/Toolbar/index.jsx), [Header](../src/containers/Header/index.jsx) / [HeaderMobile](../src/containers/HeaderMobile/index.jsx), [MobileNavigation](../src/containers/MobileNavigation/index.jsx), and the `laser-*`, `text-options`, `text-2-img`, `3d-*` components. Both middleware files import this module too and re-dispatch from it.

---

## 2. Descriptions

**Legend for the "Handled" column**

| Mark | Where |
| --- | --- |
| **R** | [src/reducers/draw-tool.js](../src/reducers/draw-tool.js) |
| **M** | [src/middleware/draw-tool.js](../src/middleware/draw-tool.js) |
| **P** | [src/middleware/product.js](../src/middleware/product.js) |
| **—** | nowhere — dispatching it is a no-op |

All examples assume the standard wiring (see §3.1):

```js
import * as actions from '../../actions/draw-tool';
const { dispatch } = this.props;
```

### 2.1 Tool & canvas control

| Creator | Action type | Payload | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `setActiveTool` | `SET_ACTIVE_TOOL` | tool name `string` | The single biggest action in the app. Stops the current action, fires `update:activeTool`, then switches panning / drawing / grid / embroidery mode (~360 lines of middleware). The reducer stores `activeTool` and clears `file_upload`. Known names: `pointer`, `panning`, `layers`, `layer`, `sides`, `brush`, `brushlaser`, `laserLineVectorBrush`, `laserCurveVectorBrush`, `changeToCropPathBrush`, `grid`, `laser`, `carpet`, `uploadFile`, `shapes`, `sticker`, `removeColor`, `changeBorder`, `saveBorder`, `embroider`, `embroider_extend`, `_embroider` | R M | `dispatch(actions.setActiveTool('pointer'))` |
| `resetDefaultParas` | `RESET_DEFAULT_PARAS` | `boolean` | Sets `DrawTool.is_embroidery` and mirrors it into `state.is_embroidery`. Only ever dispatched from `middleware/product.js` | R M | `dispatch(actions.resetDefaultParas(false))` |
| `zoomIn` | `ZOOM_IN` | – | Zooms the canvas in and recalculates every object's coords. No store change | M | `dispatch(actions.zoomIn())` |
| `zoomOut` | `ZOOM_OUT` | – | Zooms out, same recalculation | M | `dispatch(actions.zoomOut())` |
| `zoomZone` | `ZOOM_ZONE` | `boolean` | Sets the side's `zoom_zone` flag and mirrors it to `state.zoomZone` | R M | `dispatch(actions.zoomZone(!DrawTool.sides.selected.zoom_zone))` |
| `empty` | `EMPTY` | – | Confirm dialog → clears the current side and refunds the upload capacity it used | M | `dispatch(actions.empty())` |
| `emptyNow` | `EMPTY_NOW` | – | Same clear, but **no** dialog and no collaboration broadcast | M | `dispatch(actions.emptyNow())` |
| `emptyLaser` | `EMPTY_LASER` | – | Confirm dialog → drops laser items on **every** side, not just the selected one | M | `dispatch(actions.emptyLaser())` |
| `remove` | `REMOVE` | – | Deletes the active object. Branches for die-cut (dialog) and paired acrylic parts | M | `dispatch(actions.remove())` |
| `updateGridSize` | `UPDATE_GRID_SIZE` | `{ grid_rows, grid_cols }` | Redraws the grid overlay at a new density. ⚠ the middleware case has no `break` | R M | `dispatch(updateGridSize({ grid_rows: 20, grid_cols: 20 }))` |
| `updateTransform` | `UPDATE_TRANSFORM` | CSS transform `string` | Stores the wrapper's CSS `transform` so option panels can follow the canvas | R | `dispatch(actions.updateTransform(block.style.transform))` |

### 2.2 Brush

| Creator | Action type | Payload | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `updateBrushes` | `UPDATE_BRUSHES` | `[{ DrawerBrush: { value } }]` | Loads the brush catalogue from the API into `availableBrushes` and makes the **first** entry active | R | `dispatch(actions.updateBrushes(data))` |
| `selectBrush` | `SELECT_BRUSH` | brush method `string` | The payload is a *method name* on `side.items` — the middleware calls `items[payload](brushOptions)` | R M | `dispatch(actions.selectBrush(brush))` |
| `selectBrushColor` | `SELECT_BRUSH_COLOR` | rgba `string` | Re-invokes the current brush with a new colour and stores it in `brushOptions.color` | R M | `dispatch(actions.selectBrushColor('rgba(0, 162, 255, 1)'))` |
| `selectBrushSize` | `SELECT_BRUSH_SIZE` | `number` (px) | Re-invokes the current brush with a new `width` | R M | `dispatch(actions.selectBrushSize(size))` |
| `selectBrushOpacity` | `SELECT_BRUSH_OPACITY` | `number` | Writes `brushOptions.opacity`. **Never dispatched** anywhere in the app | R | `dispatch(actions.selectBrushOpacity(0.5))` |

### 2.3 Text

| Creator | Action type | Payload | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `addText` | `ADD_TEXT` | `string` | Adds a text item to the canvas using the store's `textOptions`. ⚠ the middleware forces font size to **30**, ignoring `textOptions.size` | R M | `dispatch(actions.addText(val))` |
| `changeText` | `CHANGE_TEXT` | `string` | Replaces the selected text item's content on the canvas | R M | `dispatch(actions.changeText(val))` |
| `changeTextVal` | `CHANGE_TEXT_VAL` | `string` | Store-only twin of `changeText` — updates `state.text` without touching the canvas (used while typing) | R | `dispatch(actions.changeTextVal(val))` |
| `selectTextFont` | `SELECT_TEXT_FONT` | font name `string` | Sets `fontFamily` on the selection and clears fabric's char-width cache | R M | `dispatch(actions.selectTextFont(font))` |
| `selectTextAlign` | `SELECT_TEXT_ALIGN` | `'left' \| 'center' \| 'right'` | Sets text alignment | R M | `dispatch(actions.selectTextAlign('center'))` |
| `selectTextBold` | `SELECT_TEXT_BOLD` | `boolean` | Toggles `fontWeight` between `bold` and `normal` | R M | `dispatch(actions.selectTextBold(false))` |
| `selectTextItalic` | `SELECT_TEXT_ITALIC` | `boolean` | Toggles `fontStyle` between `italic` and `normal` | R M | `dispatch(actions.selectTextItalic(false))` |
| `selectTextVertical` | `SELECT_TEXT_VERTICAL` | `boolean` | Rewrites the string one character per line, or flattens it back | R M | `dispatch(actions.selectTextVertical(true))` |
| `selectTextCategoriesFonts` | `SELECT_TEXT_CATEGORIES_FONTS` | category title `string` | Switches the font-picker tab (`日本語`, `英語`, `かな`, `デザイン書体`, `筆文字`, `欧文`, `ゴシック`, `明朝・基本`, …). Store-only | R | `dispatch(actions.selectTextCategoriesFonts('日本語'))` |
| `selectTextColor` | `SELECT_TEXT_COLOR` | rgba `string` | Fills the selected text item. **Never dispatched** — text colour goes through `selectShapeColor` instead | R M | `dispatch(actions.selectTextColor('rgba(0,0,0,1)'))` |
| `selectTextSize` | `SELECT_TEXT_SIZE` | `number` | Sets `fontSize` on the selection. **Never dispatched** | R M | `dispatch(actions.selectTextSize(24))` |
| `updateTextEffect` | `UPDATE_TEXT_EFFECT` | effect `string` | Records the active warp effect: `straight` (default), `curved`, `arc`, `smallToLarge`, `bulge`, `largeToSmallBottom`. Store-only — the component calls `item.setEffect()` itself | R | `dispatch(actions.updateTextEffect(value))` |
| `updateTextSpacing` | `UPDATE_TEXT_SPACING` | `number` (default 20) | Letter spacing. The middleware branches on the current effect: warped text gets `setSpacing(payload / 10)`, straight text gets `setCharSpacing(payload)` | R M | `dispatch(actions.updateTextSpacing(value))` |
| `setTextOption` | `SET_TEXT_OPTION` | *ignored* | Hard-resets `textOptions.font` to the literal `'normal_font'`. The payload is discarded | R | `dispatch(actions.setTextOption())` |
| `setIsText2IMG` | `SET_TEXT_2_IMG` | `boolean` | Opens/closes the text-to-image dialog | R | `dispatch(actions.setIsText2IMG(true))` |

### 2.4 Layers, selection & transforms

| Creator | Action type | Payload | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `updateLayers` | `UPDATE_LAYERS` | `{ layers, side }` | Rebuilds the layer list for one side. The middleware only fires `layers:update`; the payload is consumed by the reducer, which also sets `change_design: true` and clears `layersSelected` | R M | `dispatch(actions.updateLayers({ layers: DrawTool.sides.selected.layers.update().reverse(), side: DrawTool.sides.selected.id }))` |
| `sortLayers` | `SORT_LAYERS` | `{ items: [{ index }] }` | Reorders layers from a **top-first** uuid list. Canvas events are muted for 100 ms during the reorder | M | `dispatch(actions.sortLayers({ items }))` |
| `focusLayer` | `FOCUS_LAYER` | layer id | Appends the id to `layersSelected` (multi-select) | R | `dispatch(actions.focusLayer(id))` |
| `blurLayer` | `BLUR_LAYER` | layer id | Removes the id from `layersSelected` | R | `dispatch(actions.blurLayer(id))` |
| `alignLayer` | `ALIGN_LAYER` | method name `string` | Payload is a *method name*: `layers[payload](...layersSelected)`. One of `toLeft`, `toRight`, `toTop`, `toBottom`, `toHCenter`, `toVCenter` | M | `dispatch(actions.alignLayer('toHCenter'))` |
| `alignItem` | `ALIGN_ITEM` | method name `string` | Same six names, applied to the single selected item: `items.selected[payload]()` | M | `dispatch(actions.alignItem('toLeft'))` |
| `selectItem` | `SELECT_ITEM` | fabric object | Mirrors the fabric selection into the store: `selected`, `is_image`, `item_brush`, `image_uuid`, `image_url_re`, plus `textOptions` / `shapeColor` and the matching font-category tab. Store-only | R | `dispatch(actions.selectItem(item))` |
| `unselectItem` | `UNSELECT_ITEM` | – | Fires `item:unselect` and clears `selected`, `textEl`, `text` | R M | `dispatch(actions.unselectItem())` |
| `updateDegree` | `UPDATE_DEGREE` | `number` (degrees) | `setAngle()` on the selection, then re-runs the acrylic / carpet placement checks. Blocked for acrylic-laser items whose `mode_design.mode === "1"` | R M | `dispatch(actions.updateDegree(nextProps.selected.angle \|\| 0))` |
| `updateScale` | `UPDATE_SCALE` | `number` (scale factor) | Scales the selection and stores the factor for the size readout | R M | `dispatch(actions.updateScale(item.scaleX))` |
| `updateChangeDesign` | `UPDATE_CHANGE_DESIGN` | `boolean` | The "unsaved changes" dirty flag. Set to `true` automatically by `UPDATE_LAYERS`; components clear it with `false` after a save | R | `dispatch(actions.updateChangeDesign(false))` |

### 2.5 Insert — shapes, stickers, laser, pedestal

| Creator | Action type | Payload | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `insertShape` | `INSERT_SHAPE` | url `string` | Adds an SVG shape filled with the store's current `shapeColor` | M | `dispatch(actions.insertShape(url))` |
| `insertSticker` | `INSERT_STICKER` | url `string` | Adds an SVG sticker **keeping its own fills** (unlike `insertShape`) | M | `dispatch(actions.insertSticker(sticker))` |
| `insertLaser` | `INSERT_LASER` | shape `object` | Adds a laser SVG plus its cut points, then re-runs `checkLaserColor()` | M | `dispatch(actions.insertLaser(cat))` |
| `insertPedestalLaser` | `INSERT_PEDESTAL_LASER` | pedestal `object` | Adds a pedestal image. Only acts in `ACRYLIC_LASER` mode | M | `dispatch(actions.insertPedestalLaser(selectData))` |
| `setPedestalLaser` | `SET_PEDESTALL_LASER` | `{ active, data }` | Opens/closes the pedestal picker. ⚠ the type string has a typo — `PEDESTALL`, two L's | R | `dispatch(actions.setPedestalLaser({ active: true, data: dataPad }))` |
| `insertImage` | `INSERT_IMAGE` | url `string` | Adds an image by URL; reducer sets `loading: false`. **Never dispatched** — image insert goes through `DrawTool` directly | R M | `dispatch(actions.insertImage(url))` |
| `rimImageToSvg` | `RIM_IMAGE_TO_SVG` | – | Runs `items.rimImageProcessing()` — traces the selected bitmap into an SVG outline | M | `dispatch(actions.rimImageToSvg())` |

### 2.6 Colour

| Creator | Action type | Payload | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `selectShapeColor` | `SELECT_SHAPE_COLOR` | rgba `string` | Fills the current selection (shapes **and** text) and stores `shapeColor` | R M | `dispatch(actions.selectShapeColor(color))` |
| `toggleColorPicker` | `TOGGLE_COLOR_PICKER` | `boolean` | ⚠ **not** a toggle despite the name — pass the value you want | R M | `dispatch(actions.toggleColorPicker(!colorPicker))` |
| `updateColorPicker` | `UPDATE_COLOR_PICKER` | rgba `string` | Stores the eyedropper's picked colour in `colorPickerColor`. Store-only | R | `dispatch(actions.updateColorPicker(color))` |
| `removeColor` | `REMOVE_COLOR` | – | Removes `state.colorPickerColor` from the selection at tolerance 25. Reads the colour from the store, so `updateColorPicker` must land first | M | `dispatch(actions.removeColor())` |
| `fillColors` | `FILL_COLORS` | `{ old_color, new_color }` | Swaps one colour for another inside a stamp. In tape/ribbon modes it recolours the whole item and broadcasts to collaborators | M | `dispatch(actions.fillColors({ old_color, new_color }))` |
| `updateColorsFill` | `UPDATE_COLORS_FILL` | `string[]` | Caches the palette extracted from the selected item, for the swap UI | R | `dispatch(actions.updateColorsFill(colors))` |
| `updateColorImagesFill` | `UPDATE_COLORS_IMAGE_FILL` | `string[]` | Same for images. **Never dispatched** | R | `dispatch(actions.updateColorImagesFill(colors))` |
| `tapOrRibbonProcessingColorData` | `TapOrRibbonProcessing` | – | Recolours the selection to the nearest colour the current tape/ribbon mode allows. ⚠ the only action type that isn't SCREAMING_SNAKE_CASE | M | `dispatch(actions.tapOrRibbonProcessingColorData())` |

### 2.7 History

| Creator | Action type | Payload | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `undo` | `UNDO` | history `object` | The middleware ignores the payload and steps `DrawTool`'s own history back, then re-applies the ribbon colour 300 ms later. The reducer just stores the payload as `state.history` | R M | `dispatch(actions.undo(DrawTool.history.history[DrawTool.sides.selected.id]))` |
| `redo` | `REDO` | history `object` | Same, forwards | R M | `dispatch(actions.redo(DrawTool.history.history[DrawTool.sides.selected.id]))` |
| `updateHistory` | `UPDATE_HISTORY` | history `object` | Replaces `state.history` and clears `file_upload`, without moving the canvas. Used to resync the undo/redo buttons | R | `dispatch(actions.updateHistory(DrawTool.history.history[DrawTool.sides.selected.id]))` |
| `saveBorder` | `SAVE_BORDER` | *ignored* | Wipes the history to `{ currentIndex: 0, collection: [{}] }` — a new border becomes the new baseline, so you can't undo past it | R | `dispatch(actions.saveBorder())` |

### 2.8 Laser border & cutting

| Creator | Action type | Payload | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `updateBorder` | `UPDATE_BORDER` | – | Recomputes the laser-sticker cut border from every laser item's outline | M | `dispatch(actions.updateBorder())` |
| `updateCurentLaserBorder` | `UPDATE_CURENT_LASER_BORDER` | – | After a 200 ms delay, rebuilds a rectangular laser border — or falls back to the layers tool. ⚠ `CURENT` is misspelled in the type string | M | `dispatch(actions.updateCurentLaserBorder())` |
| `updateCuttingImageBorder` | `UPDATE_CUTTING_IMAGE` | – | Crops every image on the side to the cut line, then deletes the cut item. ⚠ creator name and type name don't match | M | `dispatch(actions.updateCuttingImageBorder())` |
| `setBorderSameCM` | `SET_BORDER_SAME_CM` | – | Resets a custom border back to its rectangular bounding box | M | `dispatch(actions.setBorderSameCM())` |
| `changeBorder` | `CHANGE_BORDER` | `number` (px) | Shifts every item left by N pixels; reducer writes `state.margin` (a key not in `initialState`). **Never dispatched** | R M | `dispatch(actions.changeBorder(10))` |

### 2.9 Fonts

All `updateFonts*` creators take the same payload — the API's font array, `[{ DrawerFont: { name, title, urls } }]` — and each writes its own `availableFonts*` list. ⚠ **every one of them also overwrites `textOptions.font` with `payload[0]`**, so loading a category resets the user's current font.

| Creator | Action type | Writes | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `updateFonts` | `UPDATE_FONTS` | `availableFonts`, `availableFontsTitle`, `allFonts`, `allFontsList` | The master font list — the only one that also fills `allFonts` / `allFontsList` and the name→title map | R | `dispatch(actions.updateFonts(data))` |
| `updateFontsJP` | `UPDATE_FONTS_JP` | `availableFontsJP` | Japanese print fonts | R | `dispatch(actions.updateFontsJP(jpPrintFonts))` |
| `updateFontsEN` | `UPDATE_FONTS_EN` | `availableFontsEN` | English print fonts | R | `dispatch(actions.updateFontsEN(enPrintFonts))` |
| `updateFontsWD` | `UPDATE_FONTS_WD` | `availableFontsWD` | かな (kana) fonts | R | `dispatch(actions.updateFontsWD(PrintFontsWD))` |
| `updateFontsDGT` | `UPDATE_FONTS_DGT` | `availableFontsDGT` | デザイン書体 (design typefaces) | R | `dispatch(actions.updateFontsDGT(PrintFontsDGT))` |
| `updateFontsBC` | `UPDATE_FONTS_BC` | `availableFontsBC` | 筆文字 (brush script) | R | `dispatch(actions.updateFontsBC(PrintFontsBC))` |
| `updateFontsEUL` | `UPDATE_FONTS_EUL` | `availableFontsEUL` | 欧文 (Latin) | R | `dispatch(actions.updateFontsEUL(PrintFontsEUL))` |
| `updateFontsGothic` | `UPDATE_FONTS_GOTHIC` | `availableFontsGothic` | ゴシック (gothic) | R | `dispatch(actions.updateFontsGothic(PrintFontsGT))` |
| `updateFontsToMor` | `UPDATE_FONTS_TO_MO` | `availableFontsToMo` | 明朝・基本 (mincho / basic). ⚠ creator, type and state key all use a different abbreviation | R | `dispatch(actions.updateFontsToMor(PrintFontsTM))` |
| `updateFontsJPEM` | `UPDATE_FONTS_JP_EMBROIDERY` | `availableFontsJPEM` | Japanese **embroidery** fonts | R | `dispatch(actions.updateFontsJPEM(jpEmbroideryFonts))` |
| `updateFontsENEM` | `UPDATE_FONTS_EN_EMBROIDERY` | `availableFontsENEM` | English **embroidery** fonts | R | `dispatch(actions.updateFontsENEM(enEmbroideryFonts))` |
| `updateCategoriesFonts` | `UPDATE_CATEGORIES_FONTS` | `availableFontsCategories` | The category tab titles, from `[{ DrawerFontCategory: { title } }]` | R | `dispatch(actions.updateCategoriesFonts(data))` |
| `updateFontsByName` | `UPDATE_FONTS_BY_NAME` | – | **Loads** one web font: `DrawTool.fontLoader(name, urls)`, or an injected `@font-face` `<style>` on IE11. Store-only in the sense that no reducer handles it | P | `dispatch(actions.updateFontsByName(font))` |
| `updateFontsAl` | `UPDATE_FONTS_AL` | – | Same, for an **array** of fonts. ⚠ the handler exists in `product.js` but the creator is **never dispatched** — orphaned | P | `dispatch(actions.updateFontsAl(fonts))` |

### 2.10 Catalogue data — shapes, stickers, stamps, 3D

Every creator here takes the raw API array and the reducer unwraps it with `.map(x => x.SomeKey)`.

| Creator | Action type | Payload shape | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `updateShapesCategories` | `UPDATE_SHAPES_CATEGORIES` | `[{ DrawerShapeCategory }]` | Shape category list | R | `dispatch(actions.updateShapesCategories(data))` |
| `updateShapes` | `UPDATE_SHAPES` | `[{ DrawerShape: { content_url } }]` | Shape URLs for the current category | R | `dispatch(actions.updateShapes(data))` |
| `updateShapesLaser` | `UPDATE_SHAPES_LASER` | `[{ DrawerLaserCategory }]` | Laser shape categories | R | `dispatch(actions.updateShapesLaser(data))` |
| `updateModelShapesLaser` | `UPDATE_MODEL_SHAPES_LASER` | `[{ PedestalModelLaser }]` | Pedestal models for acrylic laser | R | `dispatch(actions.updateModelShapesLaser(data))` |
| `updateStickersCategories` | `UPDATE_STICKERS_CATEGORIES` | `[{ DrawerStickerCategory }]` | Sticker categories | R | `dispatch(actions.updateStickersCategories(data))` |
| `updateStickers` | `UPDATE_STICKERS` | `[{ DrawerSticker: { content_url } }]` | Sticker URLs | R | `dispatch(actions.updateStickers(data))` |
| `updateStampsImageCategories` | `UPDATE_STAMPS_IMAGE_CATEGORIES` | `[{ DrawerStickerCategory }]` | Stamp-image categories | R | `dispatch(actions.updateStampsImageCategories(data))` |
| `updateStampsLoveCategories` | `UPDATE_STAMPS_LOVE_CATEGORIES` | `[{ StickerCategories }]` | Stamp-"love" categories. ⚠ note the different wrapper key | R | `dispatch(actions.updateStampsLoveCategories(data))` |
| `updateStampsImage` | `UPDATE_STAMPS_IMAGE` | `[{ DrawerSticker }]` | Stamp images for the current category | R | `dispatch(actions.updateStampsImage(data))` |
| `updateStampsLove` | `UPDATE_STAMPS_LOVE` | `[{ DrawerSticker }]` | "Love" stamps | R | `dispatch(actions.updateStampsLove(data))` |
| `updateStampsImageFavorite` | `UPDATE_STAMPS_IMAGE_FAVORITE` | `[{ DrawerSticker }]` | Favourite stamp images — the reducer **truncates to the first 30** | R | `dispatch(actions.updateStampsImageFavorite(data))` |
| `updateStampsLovesFavorite` | `UPDATE_STAMPS_LOVES_FAVORITE` | `[{ DrawerSticker }]` | Favourite love stamps — **truncated to 50** | R | `dispatch(actions.updateStampsLovesFavorite(data))` |
| `updateShapes3D` | `UPDATE_SHAPES_3D` | `[{ Drawer3DCategory }]` | 3D model categories | R | `dispatch(actions.updateShapes3D(data))` |
| `update3DViewData` | `UPDATE_3D_DATA_VIEW` | `{ selected, url, info, position, type_mesh }` | The currently-previewed 3D mesh. Dispatched from `middleware/product.js`. ⚠ creator and type name are reversed | R | `dispatch(actions.update3DViewData({ selected: true, url, info, position, type_mesh }))` |

### 2.11 Loading, progress & uploads

| Creator | Action type | Payload | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `setLoading` | `LOADING` | `boolean` | The main full-screen spinner (`state.loading`, starts `true`) | R | `dispatch(actions.setLoading(false))` |
| `setLoadingProcess` | `LOADING_BAR` | `boolean` | Shows/hides the progress bar | R | `dispatch(actions.setLoadingProcess(true))` |
| `updateProcessBar` | `UPDATE_PROCESS` | `number` | Progress value. ⚠ inconsistent scale — nearly all call sites pass a **0–1 fraction** (`0.4`, `0.99`), but at least one passes `100` | R | `dispatch(actions.updateProcessBar(0.4))` |
| `setLoadingCategories` | `LOADING_CATEGORIES` | `boolean` | Category-panel spinner (starts `true`) | R | `dispatch(actions.setLoadingCategories(false))` |
| `setLoadingColors` | `LOADING_COLORS` | `boolean` | Colour-list spinner | R | `dispatch(actions.setLoadingColors(true))` |
| `setLoadingKeys` | `LOADING_KEY` | `boolean` | Laser key-holder list spinner | R | `dispatch(actions.setLoadingKeys(true))` |
| `setLoadingAllProducts` | `LOADING_ALL_PRODUCT` | `boolean` | Product-list spinner. ⚠ state key is misspelled: `loadingAllProructs` | R | `dispatch(actions.setLoadingAllProducts(true))` |
| `setLoadingAPI` | `LOADING_API` | `boolean` | Tracks whether the initial API burst has finished (`loadingAPIComplate`, starts `true`) | R | `dispatch(actions.setLoadingAPI(false))` |
| `setFilter` | `FILTER` | `boolean` | Opens/closes the filter panel | R | `dispatch(actions.setFilter(true))` |
| `setFileUpload` | `SET_FILEUPLOAD` | `array` | The pending upload queue. Also cleared as a side effect by `SET_ACTIVE_TOOL` and `UPDATE_HISTORY` | R | `dispatch(actions.setFileUpload(this.props.file_upload))` |
| `setImageUploader` | `IMAGE_UPLOADER` | `number` | A counter bumped to force the uploader component to remount | R | `dispatch(actions.setImageUploader(this.props.image_uploader + 1))` |
| `updateImageupload` | `UPDATE_IMAGE_UPLOAD` | uuid `string` | **Appends** (concat) the uuid of an uploaded image — never replaces | R | `dispatch(actions.updateImageupload(itemData.uuid))` |
| `updateCapacityUploading` | `UPDATE_CAPACITY_UPLOADING` | `number` (**delta**, bytes) | Adds to the running upload total and mirrors it onto `DrawTool.capacity_uploadingAllImage`. Pass a **negative** value to refund a deleted image. Not an absolute value | R | `dispatch(actions.updateCapacityUploading(0 - itemDelete.sizeImage))` |

### 2.12 3D preview & misc flags

| Creator | Action type | Payload | Description | Handled | Example |
| --- | --- | --- | --- | --- | --- |
| `change3DPreviewStatus` | `CHANG_3D_PREVIEW_STATUS` | `boolean` | Opens/closes the 3D preview. ⚠ `CHANG`, not `CHANGE` | R | `dispatch(actions.change3DPreviewStatus(!this.props.is_3d_preview))` |
| `change3DPreviewLaserStatus` | `CHANG_3D_PREVIEW_LASER_STATUS` | `boolean` | Same, for the laser 3D preview | R | `dispatch(actions.change3DPreviewLaserStatus(!this.props.is_3d_preview_laser))` |
| `setFlagOption` | `SET_FLAG_OPTION` | `string` | A free-form mode marker for the options panel, e.g. `"design"` | R | `dispatch(actions.setFlagOption("design"))` |

### 2.13 Exported but handled nowhere

These six creators have **no reducer case and no middleware case**. Dispatching them reaches the logger, passes through both middleware untouched, and changes nothing.

| Creator | Action type | Notes |
| --- | --- | --- |
| `setStepAcrylic` | `SET_STEP_ACRYLIC` | Never dispatched either — the acrylic step wizard is driven by `setActiveTool` |
| `selectLaserErase` | `SELECT_LASER_ERASE` | Never dispatched |
| `selectShape` | `SELECT_SHAPE` | Never dispatched — `insertShape` is what's actually used |
| `selectText` | `SELECT_TEXT` | Never dispatched — superseded by `selectItem` |
| `selectTextOff` | `SELECT_TEXT_OFF` | Never dispatched — superseded by `unselectItem` |
| `upload` | `UPLOAD` | Never dispatched — uploads go through `setFileUpload` / `updateCapacityUploading` |

Plus these, which **are** handled but never dispatched: `selectBrushOpacity`, `selectTextColor`, `selectTextSize`, `insertImage`, `changeBorder`, `updateColorImagesFill`, `updateFontsAl`.

---

## 3. Examples

### 3.1 The wiring pattern

Nothing in this project uses `mapDispatchToProps` for draw-tool actions except [GridChange.jsx](../src/containers/Options/GridChange.jsx). Everywhere else, `connect()` is called with only `mapStateToProps`, so React-Redux injects a raw `dispatch` prop:

```jsx
import * as actions from '../../actions/draw-tool';
import { connect } from 'react-redux';

class Toolbar extends React.Component {
  render() {
    const { dispatch, colorPicker } = this.props;
    return <button onClick={() => dispatch(actions.toggleColorPicker(!colorPicker))} />;
  }
}

export default connect(mapStateToProps)(Toolbar);   // no mapDispatchToProps
```

The one exception, for reference:

```jsx
// src/containers/Options/GridChange.jsx
import { updateGridSize } from '../../actions/draw-tool';

function mapDispatchToProps(dispatch) {
  return { updateGridSize: (data) => dispatch(updateGridSize(data)) };
}
// then: props.updateGridSize({ grid_rows: 20, grid_cols: 20 })
```

Both middleware files use a third form — re-dispatching through the store:

```js
// src/middleware/product.js
import * as actions from '../actions/draw-tool';
store.dispatch(actions.setActiveTool('pointer'));
```

### 3.2 Real call sites

| Scenario | Code | Source |
| --- | --- | --- |
| Undo, then resync the layer panel | `dispatch(actions.undo(DrawTool.history.history[DrawTool.sides.selected.id]));`<br>`dispatch(actions.updateLayers({ layers: DrawTool.sides.selected.layers.update().reverse(), side: DrawTool.sides.selected.id }));` | [Toolbar/index.jsx:171-177](../src/containers/Toolbar/index.jsx#L171-L177) |
| Enter/exit embroidery mode | `dispatch(actions.setActiveTool(is_extend ? 'embroider_extend' : 'embroider'));` | [Toolbar/index.jsx:217-225](../src/containers/Toolbar/index.jsx#L217-L225) |
| Drag-reorder the layer list | `<Layers callbackNewOrder={(items) => dispatch(actions.sortLayers({ items }))} />` | [Options/index.jsx:5405](../src/containers/Options/index.jsx#L5405) |
| Swap one colour in a stamp | `const color = { old_color, new_color };`<br>`dispatch(actions.fillColors(color));` | [Options/index.jsx:5749-5752](../src/containers/Options/index.jsx#L5749-L5752) |
| Refund capacity when deleting an image | `dispatch(actions.updateCapacityUploading(0 - itemDelete.sizeImage));` | [Options/index.jsx](../src/containers/Options/index.jsx) |
| Toggle 3D preview | `dispatch(actions.change3DPreviewStatus(!this.props.is_3d_preview));` | [App/index.jsx](../src/containers/App/index.jsx) |
| Follow the fabric rotation handle | `dispatch(actions.updateDegree(this.intervalCheck.rotating.value));` | [App/index.jsx](../src/containers/App/index.jsx) |
| Toggle the zoom zone | `dispatch(actions.zoomZone(!DrawTool.sides.selected.zoom_zone));` | [Toolbar/index.jsx](../src/containers/Toolbar/index.jsx) |

### 3.3 Adding a new action

1. Add one line here — `export const myThing = createAction('MY_THING');`
2. Decide **where** it belongs:
   - pure store state → a case in [src/reducers/draw-tool.js](../src/reducers/draw-tool.js) **and** a default in `initialState`;
   - canvas side effect → a case in [src/middleware/draw-tool.js](../src/middleware/draw-tool.js), ending in `break;` (a bare `return;` also blocks the reducer **and** `product.js`);
   - both → both.
3. Dispatch it from a component with `dispatch(actions.myThing(payload))`.

### 3.4 Gotchas worth knowing before you dispatch

| Gotcha | Detail |
| --- | --- |
| **Order matters** | The canvas is mutated by the middleware *before* the reducer runs. If a middleware case `return`s early, the store silently keeps its old value |
| **Blanket side effect** | For every action *except* `UPDATE_LAYERS`, `UNSELECT_ITEM`, `EMPTY`, `UPDATE_PRICE`, `UPDATE_COLORS_FILL`, `UPDATE_COLORS_IMAGE_FILL`, `LOADING_CATEGORIES`, `middleware/draw-tool.js` calls `finalizeBrush()` and schedules a `layers:update` 1 s later — even for types it has no case for |
| **Nothing runs before a side exists** | `middleware/draw-tool.js` bails if `DrawTool.sides.selected` is falsy, so actions dispatched during initial load are pass-through |
| **`textOptions` is mutated in place** | The `UPDATE_FONTS*` and `SET_TEXT_OPTION` reducers use `Object.assign(state.textOptions, …)` on the *existing* object, not a copy — referential-equality checks won't see the change |
| **`SELECT_ITEM` mutates `state` directly** | It writes `state.is_image`, `state.image_uuid`, `state.item_brush`, `state.image_url_re` and `state.categoriesFontsOptions.title` before returning a new object |
| **Names lie in places** | `toggleColorPicker` isn't a toggle; `updateCuttingImageBorder` → `UPDATE_CUTTING_IMAGE`; `update3DViewData` → `UPDATE_3D_DATA_VIEW`; `setPedestalLaser` → `SET_PEDESTALL_LASER`; `TapOrRibbonProcessing` isn't SCREAMING_SNAKE_CASE; `CHANG_3D_*` and `CURENT` are misspelled |
| **No dead-string safety net** | Since types are bare literals with no shared constants, a renamed type in the reducer and not here (or vice versa) fails silently — that's how the seven orphans above came about |
