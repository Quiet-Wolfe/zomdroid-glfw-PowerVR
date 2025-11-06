This repository is a fork of [GLFW](https://github.com/LWJGL-CI/glfw) library (from LWJGL), modified for use in [Zomdroid](https://github.com/liamelui/zomdroid) project.

## PowerVR GPU Optimizations

This fork includes comprehensive optimizations for PowerVR GPUs commonly found in Android devices. These optimizations are specifically designed for Tile-Based Deferred Rendering (TBDR) architectures.

**Key Features:**
- Automatic packed depth-stencil format selection (GL_DEPTH24_STENCIL8_OES)
- Framebuffer attachment discard for efficient tile memory management
- PowerVR-aware EGL config selection
- Optimized surface attributes for TBDR
- Configurable via environment variables

For detailed information, see [POWERVR.md](POWERVR.md).

## Quick Start

The PowerVR optimizations are enabled by default on the Zomdroid platform. To control them:

```bash
# Force enable PowerVR optimizations
export GLFW_POWERVR_OPTIMIZE=1

# Enable debug logging
export GLFW_POWERVR_DEBUG=1

# Disable depth/stencil forcing (if needed)
export GLFW_POWERVR_FORCE_DEPTH_STENCIL=0
```