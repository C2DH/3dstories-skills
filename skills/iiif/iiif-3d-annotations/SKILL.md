---
name: iiif-3d-annotations
description: Add point-based annotations to IIIF 3D scenes with titles, HTML descriptions, and 3D positions. Use when the user wants to annotate specific points in a 3D model, add metadata to scenes, or create interactive hotspots.
---

# Skill: Adding Point-Based Annotations to IIIF 3D Scenes

This skill guides you through implementing and working with the annotation system for IIIF 3D scenes. Annotations are point markers in 3D space with associated metadata (title, description, time range).

## 1. Core Concept

Annotations are an **extension** to IIIF Presentation 4.0, not part of the core spec. They use a custom `iiif3d:annotations` array on Scene objects. Each annotation has:

- **id**: Unique identifier (e.g., `"tuning-dial"`)
- **label**: Display title (IIIF language map: `{ "en": "Title" }`)
- **description**: HTML-enabled text (e.g., `{ "en": "The <b>tuning dial</b>..." }`)
- **position**: 3D coordinates `{ x, y, z }`
- **timeRange** (optional): When visible `{ start: 0, end: 10 }` in seconds
- **focusDistance** (optional): how far back the camera sits when the annotation
  is clicked, in scene units. Omit to let the viewer choose (halfway in)
- **camera** (optional): an exact eye position `{ x, y, z }` for the click-to-focus.
  Wins over `focusDistance` and aims at `position` — use it for a point only
  visible from one side

## 2. Architecture

The system uses a **plugin pattern**:

```
Manifest (JSON)
  └─ Scene
      └─ "iiif3d:annotations": [...]
                ↓
        parseManifest()
             ↓
         parseScene()
             ↓
         parseAnnotations()  ← Separate plugin module
             ↓
         Scene3D { annotations: SceneAnnotation[] }
             ↓
         SceneNavigator
             └─ Renders purple circles (clickable)
             └─ Shows AnnotationPopover on click
```

**Benefits**:
- Annotation parsing lives in separate module (`iiif3d-annotations.ts`)
- Renderer-agnostic (works with any viewer)
- Easy to extend for future features
- Reusable in different contexts

## 3. Adding Annotations to a Manifest

### 3.1 Basic Structure

Edit your IIIF manifest to add an `iiif3d:annotations` array to the Scene:

```json
{
  "id": "https://example.org/iiif/my-scene.json",
  "type": "Manifest",
  "items": [
    {
      "id": "https://example.org/scene/1",
      "type": "Scene",
      "duration": 10,
      "iiif3d:annotations": [
        {
          "id": "point-1",
          "label": { "en": "Point Name" },
          "description": { "en": "Description text" },
          "position": { "x": 1, "y": 2, "z": 3 },
          "timeRange": { "start": 0, "end": 10 }
        }
      ],
      "items": [...]  // Objects, cameras
    }
  ]
}
```

### 3.2 Multiple Annotations

Add as many as needed:

```json
"iiif3d:annotations": [
  {
    "id": "dial",
    "label": { "en": "Tuning Dial" },
    "description": { "en": "Adjust <b>frequency</b> selection." },
    "position": { "x": 0.8, "y": 0.5, "z": -0.2 },
    "timeRange": { "start": 0, "end": 10 }
  },
  {
    "id": "knob",
    "label": { "en": "Volume Control" },
    "description": { "en": "Controls <b>audio level</b>." },
    "position": { "x": -0.6, "y": 0.3, "z": -0.15 },
    "timeRange": { "start": 0, "end": 10 }
  }
]
```

## 4. Language Map Format

Labels and descriptions follow IIIF language map conventions. All these are valid:

```json
// Preferred (language key with string)
"label": { "en": "Title" }
"label": { "fr": "Titre" }

// Also valid (language key with array)
"label": { "en": ["Title"] }
"label": { "en": ["Title", "Alternative"] }

// Also valid (generic fallback)
"label": { "none": "Title" }
"label": { "en": "English", "fr": "Français" }
```

Parser extracts: tries `en` first, then `none`, then any language, returns first string found.

## 5. HTML in Descriptions

Descriptions support HTML for formatting:

```json
"description": {
  "en": "The <b>tuning dial</b> allows <i>precise</i> <u>frequency</u> selection."
}
```

Supported tags: `<b>`, `<i>`, `<u>`, `<strong>`, `<em>`, `<a>`, etc.

⚠️ **Note**: Suitable only for trusted manifest data (your own scenes). For untrusted input, implement sanitization.

## 6. Positioning Annotations

Coordinates are in **scene space** (same coordinate system as objects and cameras):

- **x, y, z**: World coordinates in meters (for realistic scales)
- **Y-up**: Standard right-handed coordinate system
- **Units match scene objects**: If your radio is at (0, 0, 0), position annotations relative to that

**Finding good positions**:
1. Open dev server: `npm run dev`
2. Look at the 2D navigator (top-down view)
3. Note positions of interesting parts of your model
4. Click on objects to see their positions
5. Use nearby coordinates for annotations

Example positions on a radio:
```json
// Front left corner
{ "x": -0.9, "y": 0.6, "z": 0 }

// Center dial
{ "x": 0.8, "y": 0.5, "z": -0.2 }

// Back right
{ "x": 0.5, "y": 0.3, "z": 0.2 }
```

### 6.1 Camera focus on click

Clicking an annotation's hotspot in `SplatViewport` flies the camera to centre
that point (a hand-flown move, so **R** / the reset button returns it to the
Scene's track). By default the eye stops halfway in. Pin it with
`focusDistance` (scene units):

```json
{
  "id": "tuning-dial",
  "label": { "en": "Tuning Dial" },
  "description": { "en": "..." },
  "position": { "x": 0.8, "y": 0.5, "z": -0.2 },
  "focusDistance": 1.5
}
```

Pick a value a little larger than the feature you want to frame; too small and
the camera ends up inside the splat.

For a point that can only be seen from one side (the back of an object, say),
name the eye position instead. The camera flies there and aims at `position`:

```json
{
  "id": "camera-back",
  "label": { "en": "Back Panel" },
  "description": { "en": "..." },
  "position": { "x": 0.1, "y": 0.4, "z": -0.3 },
  "camera": { "x": 0.1, "y": 0.7, "z": -2.5 }
}
```

`camera` wins over `focusDistance` when both are present.

When **neither** is set, the viewer falls back to half the distance from the
**Scene's own camera** to the annotation. That distance is fixed for the
annotation, so repeated clicks always fly to the same spot (no creeping
closer). Either way it stays a hand-flown move, so the reset button / **R**
puts the camera back on the Scene's track.

## 7. Time Ranges

`timeRange` is optional. If omitted, annotation is visible for entire scene:

```json
// Visible entire duration (same as 0 → duration)
"position": { "x": 1, "y": 2, "z": 3 }

// Visible first 5 seconds
"timeRange": { "start": 0, "end": 5 }

// Visible from 3s to 8s
"timeRange": { "start": 3, "end": 8 }
```

⚠️ **Future feature**: Time-based visibility not yet implemented in UI. Currently all annotations visible always. Use `timeRange` for future compatibility.

## 8. Validation Checklist

When adding annotations to a manifest:

- [ ] Valid JSON (use `jq . manifest.json`)
- [ ] Each annotation has unique `id`
- [ ] Labels are in IIIF language map format (`{ "en": "..." }`)
- [ ] Descriptions use HTML language maps (not strings)
- [ ] Positions are objects with `x`, `y`, `z` fields
- [ ] Numbers are valid (not NaN)
- [ ] timeRange (if present) has `start` < `end`
- [ ] All required fields present (id, label, description, position)
- [ ] No duplicate IDs across annotations

## 9. Common Patterns

### 9.1 Single Annotation

```json
"iiif3d:annotations": [
  {
    "id": "main-feature",
    "label": { "en": "Main Feature" },
    "description": { "en": "This is the <b>main feature</b> of the model." },
    "position": { "x": 0, "y": 0, "z": 0 }
  }
]
```

### 9.2 Multilingual Annotations

```json
"iiif3d:annotations": [
  {
    "id": "dial",
    "label": { "en": "Tuning Dial", "fr": "Cadran d'accord" },
    "description": {
      "en": "Adjusts frequency selection.",
      "fr": "Ajuste la sélection de fréquence."
    },
    "position": { "x": 0.8, "y": 0.5, "z": -0.2 }
  }
]
```

Parser uses: English (`en`) if available, then `none`, then any language.

### 9.3 Rich Descriptions

```json
"description": {
  "en": "Click this <b>knob</b> to adjust <i>audio output</i>. " +
        "Supports values from 0 to 100. " +
        "See <a href='#docs'>documentation</a> for details."
}
```

### 9.4 Annotations with Specific Time Windows

```json
"iiif3d:annotations": [
  {
    "id": "startup",
    "label": { "en": "Startup Phase" },
    "description": { "en": "Watch the <b>indicator light</b> during startup." },
    "position": { "x": -0.3, "y": 0.8, "z": 0 },
    "timeRange": { "start": 0, "end": 3 }
  },
  {
    "id": "operation",
    "label": { "en": "Normal Operation" },
    "description": { "en": "The device is now <b>fully operational</b>." },
    "position": { "x": 0, "y": 0.5, "z": 0 },
    "timeRange": { "start": 3, "end": 10 }
  }
]
```

## 10. Interacting with Annotations in Code

### 10.1 Parsing Annotations

```typescript
import { parseManifest } from './components/iiif3d'

const manifest = await fetch('/iiif/my-scene.json').then(r => r.json())
const scenes = parseManifest(manifest)
const annotations = scenes[0]?.annotations ?? []

// Each annotation:
annotations.forEach(anno => {
  console.log(`${anno.label}: ${anno.description}`)
  console.log(`Position: (${anno.position.x}, ${anno.position.y}, ${anno.position.z})`)
  console.log(`Visible: ${anno.timeStart}s → ${anno.timeEnd}s`)
})
```

### 10.2 Using AnnotationPopover Component

```typescript
import AnnotationPopover from './components/AnnotationPopover'
import type { SceneAnnotation } from './components/iiif3d-annotations'

function MyComponent() {
  const [selected, setSelected] = useState<string | null>(null)
  const annotation: SceneAnnotation = /* ... */

  return (
    <>
      <button onClick={() => setSelected(annotation.id)}>
        Show annotation
      </button>
      
      {selected === annotation.id && (
        <AnnotationPopover
          annotation={annotation}
          left="50%"
          top="50%"
          onClose={() => setSelected(null)}
        />
      )}
    </>
  )
}
```

### 10.3 Filtering by Time

```typescript
const currentTime = 5 // seconds
const visibleAnnotations = annotations.filter(
  a => currentTime >= a.timeStart && currentTime <= a.timeEnd
)
```

## 11. Troubleshooting

### Annotations not appearing in UI
1. Check manifest loads: Browser DevTools → Network tab
2. Verify `iiif3d:annotations` is present in Scene
3. Check JSON is valid: `jq . public/iiif/your-scene.json`
4. Verify `parseAnnotations()` called in parser
5. Check `scene.annotations` in browser console

### Popover doesn't show
1. Verify `AnnotationPopover` imported in `SceneNavigator`
2. Click on purple circle in 2D navigator
3. Check browser console for React errors
4. Verify `selectedAnnotation` state updates

### HTML not rendering
1. Check description field contains HTML tags (not escaped)
2. Verify `dangerouslySetInnerHTML` in component
3. Use DevTools to inspect popover DOM

### Wrong color/position in 2D view
1. Verify position coordinates (x, y, z)
2. Check they're within scene bounds
3. Verify Tailwind `text-purple-400` class applied
4. Check SVG circle radius values

## 12. Best Practices

✅ **Do**:
- Keep descriptions concise (1-2 sentences)
- Use HTML tags sparingly for emphasis
- Position annotations at interesting features
- Test in dev server: `npm run dev`
- Use multilingual labels when possible
- Validate JSON before committing

❌ **Don't**:
- Use description for long text (use external links instead)
- Nest HTML tags deeply
- Position annotations far outside scene bounds
- Add thousands of annotations (performance degrades >100)
- Use unsafe HTML in descriptions (unless you trust the source)
- Skip validation

## 13. Example: Adding Annotations to Radio Scene

**File**: `public/iiif/radio-neyva-m.json`

```json
{
  "id": "https://telefunken-340w.local/scene/radio-neyva-m",
  "type": "Scene",
  "duration": 10,
  "iiif3d:annotations": [
    {
      "id": "tuning-dial",
      "label": { "en": "Tuning Dial" },
      "description": {
        "en": "The <b>tuning dial</b> allows precise frequency selection. Vintage AM band radio."
      },
      "position": { "x": 0.8, "y": 0.5, "z": -0.2 },
      "timeRange": { "start": 0, "end": 10 }
    },
    {
      "id": "volume-knob",
      "label": { "en": "Volume Control" },
      "description": {
        "en": "This <b>rotary knob</b> controls audio output. Classic analog adjustment."
      },
      "position": { "x": -0.6, "y": 0.3, "z": -0.15 },
      "timeRange": { "start": 0, "end": 10 }
    }
  ],
  "items": [
    // ... objects and cameras ...
  ]
}
```

**Result**: Click purple circles in 2D navigator → see annotation popover

## 14. Future Enhancements

Not yet implemented, but the plugin system supports:
- Time-based visibility (hide/show with timeline)
- Animation (fade/pulse effects)
- Interactive navigation (click to move camera)
- Metadata extension (tags, URLs, categories)
- Editor UI (CRUD operations)
- Validation schema (JSON Schema)
- Sanitization (for untrusted data)

## 15. File References

- **Parsing**: `src/components/iiif3d-annotations.ts`
- **Component**: `src/components/AnnotationPopover.tsx`
- **Integration**: `src/components/SceneNavigator.tsx`
- **Type defs**: See `SceneAnnotation` in `iiif3d-annotations.ts`
- **Example manifest**: `public/iiif/radio-neyva-m.json`

## 16. Documentation

For more information, see:
- `ANNOTATIONS_README.md` - Overview and quick start
- `ANNOTATIONS.md` - Full API reference
- `ARCHITECTURE.md` - System design and diagrams
- `TESTING.md` - Testing procedures
- `IMPLEMENTATION.md` - Implementation details
