# PowerVR GPU Optimizations for GLFW

This document describes the PowerVR-specific optimizations implemented in this GLFW fork for improved performance on PowerVR GPUs, commonly found in Android devices and embedded systems.

## Overview

PowerVR GPUs use Tile-Based Deferred Rendering (TBDR) architecture, which differs significantly from traditional Immediate Mode Rendering (IMR) GPUs. This GLFW fork includes several optimizations specifically tailored for TBDR architectures.

## Key Optimizations

### 1. Packed Depth-Stencil Format (GL_DEPTH24_STENCIL8_OES)

**Location:** `src/window.c:305-338`

By default, GLFW forces a 24-bit depth buffer and 8-bit stencil buffer configuration. This encourages the EGL driver to select the `GL_DEPTH24_STENCIL8_OES` packed format, which is highly efficient on PowerVR GPUs.

**Benefits:**
- Reduces memory bandwidth
- Improves tile memory utilization
- Prevents visual corruption issues with mismatched formats
- Better cache coherency

**Control:** Set `GLFW_POWERVR_FORCE_DEPTH_STENCIL=0` to disable this optimization.

### 2. Framebuffer Attachment Discard

**Location:** `src/egl_context.c:494-557`

Before swapping buffers, GLFW automatically discards depth and stencil attachments using `glDiscardFramebufferEXT`. This is critical for TBDR performance.

**How it works:**
- On TBDR GPUs, rendering happens in fast on-chip tile memory
- Discarding attachments tells the driver these tiles don't need to be written back to main memory
- This eliminates expensive memory writes and significantly improves performance

**Supported Extensions:**
- `GL_EXT_discard_framebuffer` - Explicit framebuffer discard
- `GL_IMG_multisampled_render_to_texture` - PowerVR MSAA optimization

**Control:**
- `GLFW_POWERVR_DISCARD_COLOR=1` - Also discard color attachments (use with caution)
- `GLFW_POWERVR_DEBUG=1` - Enable debug logging for discard operations

### 3. EGL Config Selection

**Location:** `src/egl_context.c:107-133, 272-286`

GLFW includes a PowerVR-aware config scoring system that prioritizes TBDR-friendly pixel formats.

**Prioritized Configurations:**
- Packed depth-stencil (24+8 bits): +100 score
- RGBA8888 color format: +50 score
- RGB565 for non-alpha content: +40 score
- No accumulation buffers: +20 score
- MSAA samples: +10 score (TBDR handles MSAA efficiently)

### 4. EGL Surface Attributes

**Location:** `src/egl_context.c:1005-1040`

PowerVR-specific surface attributes are applied during context creation:

**EGL_SWAP_BEHAVIOR:**
- Set to `EGL_BUFFER_DESTROYED` by default
- Allows driver to discard tile memory without writeback
- More efficient than `EGL_BUFFER_PRESERVED` on TBDR

**EGL_IMG_context_priority:**
- Optional high/medium priority context for better scheduling
- Available when `EGL_IMG_context_priority` extension is supported

**Control:**
- `GLFW_POWERVR_PRESERVE_BUFFER=1` - Use buffer preservation (not recommended)
- `GLFW_POWERVR_CONTEXT_PRIORITY=high` - Request high priority context
- `GLFW_POWERVR_CONTEXT_PRIORITY=medium` - Request medium priority context

### 5. Automatic GPU Detection

**Location:** `src/egl_context.c:82-105`

GLFW can automatically enable PowerVR optimizations based on:
- Platform detection (enabled by default on `_GLFW_ZOMDROID`)
- Environment variable override

**Control:**
- `GLFW_POWERVR_OPTIMIZE=1` - Force enable PowerVR optimizations
- `GLFW_POWERVR_OPTIMIZE=0` - Force disable PowerVR optimizations

## Environment Variables Summary

| Variable | Default | Description |
|----------|---------|-------------|
| `GLFW_POWERVR_OPTIMIZE` | `1` on Zomdroid | Enable/disable all PowerVR optimizations |
| `GLFW_POWERVR_FORCE_DEPTH_STENCIL` | `1` | Force 24-bit depth + 8-bit stencil |
| `GLFW_POWERVR_DISCARD_COLOR` | `0` | Discard color attachments (use carefully) |
| `GLFW_POWERVR_PRESERVE_BUFFER` | `0` | Use EGL_BUFFER_PRESERVED instead of DESTROYED |
| `GLFW_POWERVR_CONTEXT_PRIORITY` | (none) | Set to `high` or `medium` for priority context |
| `GLFW_POWERVR_DEBUG` | `0` | Enable debug logging for optimizations |

## Performance Tips

### Do's ✓
- Use the default depth/stencil configuration (24+8 bits)
- Let GLFW discard depth/stencil attachments automatically
- Use `EGL_BUFFER_DESTROYED` swap behavior
- Enable MSAA if needed (TBDR handles it efficiently)
- Use double buffering (default)

### Don'ts ✗
- Don't disable framebuffer discard unless necessary
- Avoid reading from depth/stencil buffers after rendering
- Don't use `EGL_BUFFER_PRESERVED` without a specific reason
- Avoid mismatched depth/stencil formats (e.g., separate 16-bit depth)
- Don't use accumulation buffers (legacy feature)

## Technical Background: TBDR Architecture

Traditional IMR GPUs render primitives in submission order and immediately write to framebuffer memory. PowerVR's TBDR works differently:

1. **Geometry Phase:** GPU processes all geometry and builds a per-tile display list
2. **Rasterization Phase:** Each tile is rendered independently in fast on-chip memory
3. **Writeback Phase:** Completed tiles are written to main memory

**Key Advantage:** Only visible pixels are shaded (deferred rendering), and all shading happens in fast tile memory.

**Key Challenge:** Writing tile memory back to main memory is expensive. The optimizations in this fork minimize these writebacks.

## Debugging

Enable debug logging to verify optimizations are active:

```bash
export GLFW_POWERVR_DEBUG=1
export GLFW_POWERVR_OPTIMIZE=1
```

You should see log messages like:
```
GLFW PowerVR: Discarding 2 framebuffer attachment(s) per frame
```

## Compatibility

These optimizations are designed to be safe and compatible with all OpenGL ES applications. They can be disabled individually or globally if needed.

- **Zomdroid Platform:** Optimizations enabled by default
- **Other Platforms:** Controlled via `GLFW_POWERVR_OPTIMIZE` environment variable
- **Non-PowerVR GPUs:** Optimizations are no-op or minimal impact when disabled

## References

- [PowerVR Performance Recommendations](https://docs.imgtec.com/graphics-driver-optimisation-guides/)
- [EGL_EXT_discard_framebuffer Specification](https://www.khronos.org/registry/EGL/extensions/EXT/EGL_EXT_discard_framebuffer.txt)
- [Tile-Based Rendering (TBR) Overview](https://developer.arm.com/documentation/102662/0100/Tile-based-rendering)

## License

Same as GLFW - zlib/libpng license. See LICENSE.md for details.

## Contributing

When adding new PowerVR optimizations:
1. Add an environment variable to control the optimization
2. Document it in this file
3. Test on both PowerVR and non-PowerVR hardware
4. Ensure optimizations are safe when disabled
