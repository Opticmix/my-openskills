---
name: blender-glb-alignment
description: Aligns 3D GLB models to a reference model (glasses1.glb) using Blender MCP. Automatically handles scale and position, with manual rotation confirmation via screenshot. For WebAR glasses try-on applications.
---

# Blender GLB Alignment Skill

## When to Use This Skill
- User asks to "fix", "align", or "adjust" a GLB 3D model
- Importing new 3D glasses models for WebAR
- Models appear at wrong scale, position, or orientation in the demo

## Target Specifications (from README.md)
> **IMPORTANT**: These are the official specs from `WebAR.rocks.face/demos/VTOGlasses/README.md`

| Specification | Value |
|--------------|-------|
| **Interior Width** | Exactly **2.0 world units** |
| **Vertical Axis** | Z axis (blue) in Blender |
| **Center** | X axis (red, horizontal) |
| **Pupils Position** | On X axis (Z = 0) |
| **Eyeball Tangent** | Most prominent part at Y = 0, Z = 0 |
| **Branches** | Parallel, touching ears at Z = 0 |

## Reference Model
- Path: `WebAR.rocks.face/demos/VTOGlasses/assets/models3D/glasses1.glb`
- This model already follows the specs above - use it as visual reference

## Alignment Process (4 Steps)

### Step 0: Clear Scene (ALWAYS DO FIRST)
Before any alignment, clear all existing objects from Blender:
```python
import bpy
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete()
# Also purge orphan data
for block in bpy.data.meshes: bpy.data.meshes.remove(block)
for block in bpy.data.materials: bpy.data.materials.remove(block)
```

### Step 1: Load & Align (Automatic)
Use Blender MCP to:
1. Clear the Blender scene
2. Import reference model (`glasses1.glb`)
3. Import target model
4. Calculate bounding boxes for both
5. Create container Empty at target center
6. Parent target meshes to container
7. Apply scale: `ref_width / target_width`
8. Move container to reference center

```python
import bpy
import mathutils
import os

# Clear scene
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete()

# Paths
base = r"<PROJECT_PATH>/WebAR.rocks.face/demos/VTOGlasses/assets/models3D"
ref_path = os.path.join(base, "glasses1.glb")
target_path = os.path.join(base, "<TARGET_FILENAME>")
export_path = os.path.join(base, "<TARGET_BASENAME>_fixed.glb")

# Import reference
bpy.ops.import_scene.gltf(filepath=ref_path)
ref_obs = bpy.context.selected_objects[:]
bpy.ops.object.select_all(action='DESELECT')

# Import target
bpy.ops.import_scene.gltf(filepath=target_path)
tgt_obs = bpy.context.selected_objects[:]

# Helper function
def get_bbox(objects):
    min_v = mathutils.Vector((float('inf'),)*3)
    max_v = mathutils.Vector((float('-inf'),)*3)
    for o in objects:
        if o.type == 'MESH':
            for v in o.bound_box:
                w = o.matrix_world @ mathutils.Vector(v)
                for i in range(3):
                    min_v[i] = min(min_v[i], w[i])
                    max_v[i] = max(max_v[i], w[i])
    return (max_v - min_v), (min_v + max_v) / 2

ref_size, ref_center = get_bbox(ref_obs)
tgt_size, tgt_center = get_bbox(tgt_obs)

# Create container and parent
bpy.ops.object.empty_add(type='PLAIN_AXES', location=tgt_center)
container = bpy.context.active_object
for o in tgt_obs:
    if o.parent is None:
        o.parent = container
        o.matrix_parent_inverse = container.matrix_world.inverted()

# Apply scale
scale = ref_size[0] / tgt_size[0]
container.scale = (scale, scale, scale)
bpy.context.view_layer.update()

# Align center
new_size, new_center = get_bbox(tgt_obs)
container.location += ref_center - new_center
```

### Step 2: Visual Confirmation (Manual)
After alignment, ALWAYS:
1. Take a Blender viewport screenshot using `mcp_blender_get_viewport_screenshot`
2. Show it to the user
3. Ask: "방향이 맞나요? (Is the orientation correct?)"

### Step 3: Rotation Adjustment (If Needed)
If user says the model is rotated wrong:
- Apply Z-axis rotation: `container.rotation_euler[2] = math.radians(90)` (or -90, 180)
- Recalculate center alignment after rotation
- Take another screenshot for confirmation

### Step 4: Export
Once confirmed:
```python
bpy.ops.object.select_all(action='DESELECT')
container.select_set(True)
for o in tgt_obs: o.select_set(True)

# IMPORTANT: Use these export settings per README.md
bpy.ops.export_scene.gltf(
    filepath=export_path,
    use_selection=True,
    export_format='GLB',
    export_extras=True,          # Export custom properties (material settings)
    export_yup=True              # Convert Z-up to Y-up (required!)
)
```

## Export Settings (from README.md)
> **CRITICAL**: These export options must be enabled:
> - `export_extras=True` - Exports custom material properties (roughness, metalness)
> - `export_yup=True` - Converts Blender's Z-up to WebGL's Y-up coordinate system

## Output
- File saved as `<original_name>_fixed.glb` in the same directory
- Interior width: **2.0 world units**
- Center position: X-centered, pupils at Z=0, eyeball tangent at Y=0

## Common Rotation Fixes
| Problem | Solution |
|---------|----------|
| Model facing sideways (90° off) | `--rotate_z 90` or `--rotate_z -90` |
| Model facing backwards (180° off) | `--rotate_z 180` |
| Model lying flat | `--rotate_x 90` |

## Example Usage
User: "glasses_transparent.glb를 glasses1에 맞게 정렬해줘"

Claude:
1. Runs alignment code via Blender MCP
2. Takes screenshot
3. Asks user for confirmation
4. Applies rotation if needed
5. Exports `glasses_transparent_fixed.glb`
