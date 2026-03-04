# Element Reference

## Fixed Defaults (every element — never change)
```
fillStyle:"solid"  roughness:0  opacity:100  angle:0  version:1
versionNonce:<any int>  seed:<any int>  isDeleted:false  groupIds:[]  link:null  locked:false
```
**Exception**: section containers use `opacity:30`.

## Coordinate System
`(0,0)` = top-left. X→right, Y→down. Plan non-overlapping regions with 100px buffers:
Section A: x `0`–`800` | Section B: x `900`–`1700` | Section C: x `1800`–`2600`

## Typography Scale
| Context | fontSize |
|---------|----------|
| Diagram title | `32` |
| Section header | `24`–`28` |
| Component label | `16`–`20` |
| Annotation | `13`–`16` |
| Evidence | `12`–`13` |

## Sizing Guidelines
| Element | Width | Height |
|---------|-------|--------|
| Section container | `600`–`1000` | `400`–`800` |
| Process box | `160`–`240` | `60`–`100` |
| Evidence artifact | `200`–`400` | `100`–`250` |
| Decision diamond | `160`–`200` | `90`–`120` |
| Entry/exit ellipse | `120`–`160` | `60`–`80` |
| Marker dot | `12` | `12` |

---

## Templates

### Rectangle (process/component)
```json
{ "type":"rectangle","id":"name","x":0,"y":0,"width":180,"height":90,
  "strokeColor":"<stroke>","backgroundColor":"<fill>",
  "strokeWidth":2,"strokeStyle":"solid","roundness":{"type":3},
  "boundElements":[{"id":"name-label","type":"text"}] }
```

### Text inside shape
```json
{ "type":"text","id":"name-label","x":30,"y":32,"width":120,"height":25,
  "text":"Label","originalText":"Label","fontSize":16,"fontFamily":3,
  "textAlign":"center","verticalAlign":"middle",
  "strokeColor":"<text color>","backgroundColor":"transparent",
  "strokeWidth":1,"strokeStyle":"solid",
  "containerId":"name","lineHeight":1.25,"boundElements":null }
```

### Text free-floating
Like above but: `"textAlign":"left"`, `"verticalAlign":"top"`, `"containerId":null`

### Ellipse (entry/exit/external)
Like rectangle but `"type":"ellipse"` — omit `roundness`.

### Diamond (decision)
Like rectangle but `"type":"diamond"` — omit `roundness`.

### Arrow
```json
{ "type":"arrow","id":"src-to-tgt","x":0,"y":0,"width":100,"height":0,
  "strokeColor":"<source stroke>","backgroundColor":"transparent",
  "strokeWidth":2,"strokeStyle":"solid","points":[[0,0],[100,0]],
  "startBinding":{"elementId":"src","focus":0,"gap":2},
  "endBinding":{"elementId":"tgt","focus":0,"gap":2},
  "startArrowhead":null,"endArrowhead":"arrow","boundElements":null }
```
For curves: add midpoint e.g. `[[0,0],[50,-40],[100,0]]`.

### Line (timeline/tree — structural)
Like arrow but `"type":"line"` — omit `startBinding`, `endBinding`, `*Arrowhead` fields.

### Section container
Rectangle variant: `"strokeStyle":"dashed"`, `"opacity":30`, `"backgroundColor":"#f0f7ff"`, `"strokeColor":"#1e3a5f"`, `boundElements:null`

### Evidence artifact
Rectangle variant: `"strokeColor":"#334155"`, `"backgroundColor":"#1e293b"`, `"strokeWidth":1`
Text inside: `fontSize:13`, `strokeColor:"#22c55e"` (JSON) or syntax-colored (code).

### Marker dot (timeline node)
Ellipse `12×12`: `strokeColor` = `backgroundColor` = `#3b82f6`, `strokeWidth:1`, `boundElements:null`

---

## BoundElements Rule
Every shape with text must declare it: `"boundElements":[{"id":"label-id","type":"text"}]`
Add arrows too when connected: `{"id":"arrow-id","type":"arrow"}`
Text elements always use `"boundElements":null`.
