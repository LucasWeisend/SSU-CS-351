# WebGL Shader Playground

An incremental WebGL 2.0 project built entirely in the browser — no bundlers, no frameworks, no GPU buffers. Each stage introduces a new GLSL or rendering concept, evolving from a single hard-coded triangle to a fully animated, color-shifting star rendered in real time.

*This project was overseen and graded by professor [Dave Shreiner](https://www.linkedin.com/in/daveshreiner/)*

---

## Overview

All geometry is generated **procedurally inside the vertex shader** using `gl_VertexID` — there are no vertex buffers, no attribute arrays, and no data uploaded to the GPU at draw time. Position, shape, and animation are computed purely from math in GLSL. The JavaScript host is intentionally minimal: it compiles the shaders, passes a handful of uniforms, and calls `requestAnimationFrame`.

| Stage | File | What It Demonstrates |
|---|---|---|
| [Triangle](start_2.html) | `start_2.html` | Vertex shader math, `gl_VertexID`, trigonometric polygon generation |
| [Polygon](start_3.html) | `start_3.html` | Uniform variables (`N`), dynamic vertex count, `TRIANGLE_FAN` |
| [Star](start_4.html) | `start_4.html` | Alternating inner/outer radii via modulo arithmetic |
| [Spinning Star](start_5.html) | `start_5.html` | Time uniform (`t`), continuous animation loop, angle offset |
| [Colorful Star](start_6.html) | `start_6.html` | Vertex-to-fragment interpolation via `out`/`in`, `mix()`, time-driven color blending |

---

## How It Works

### Bufferless Geometry

Every vertex position is derived entirely from its index:

```glsl
float angle = vid * 2.0 * Pi / N;
vec2 v = radius * vec2(cos(angle), sin(angle));
gl_Position = vec4(v, 0.0, 1.0);
```

The GPU generates `N` vertices, and the vertex shader maps each index to a point on a circle — no CPU-side geometry required.

### Star Shape via Modulo

Alternating between two radii on even/odd vertex IDs creates the star's inner and outer points:

```glsl
float radius = (gl_VertexID % 2 == 0) ? 1.0 : 0.4;
```

Paired with `TRIANGLE_FAN` (which anchors all triangles to vertex 0 at the origin), this fans out into a star with N/2 points.

### Animation

A time value `t` is incremented each frame in JavaScript and passed as a uniform. The vertex shader adds it as an angle offset — spinning the star — and the fragment shader uses it to smoothly blend colors:

```glsl
// Vertex shader: rotation
float angle = t + (vid * 2.0 * Pi / N);

// Fragment shader: color blend
float blend = sin(t) / 2.0;
fColor = mix(red, blue, blend + radius);
```

The `radius` varying (passed from vertex to fragment shader via `out`/`in`) also feeds into the color blend, so inner and outer star points shade differently.

---

## Shader Pipeline

```
JavaScript (Host)
│
├─ Compiles vertex + fragment shaders via initShaders.js
├─ Passes uniforms: N (vertex count), t (time)
└─ Calls gl.drawArrays() each frame
        │
        ▼
Vertex Shader (runs once per vertex)
  • Reads gl_VertexID, N, t
  • Computes position via trigonometry
  • Outputs gl_Position + radius varying
        │
        ▼
Fragment Shader (runs once per pixel)
   • Reads radius varying + t
   • Outputs final RGBA color via mix()
```

---

## Running Locally

No build step required. Just serve the files over HTTP (browsers block WebGL on `file://`) and open any stage:

```bash
# Python 3
python3 -m http.server 8000

# Then open: http://localhost:8000/start_6.html
```

`initShaders.js` handles shader compilation, GLSL ES version detection (WebGL 1 vs 2), and error reporting automatically.

---

## Key Concepts Demonstrated

- **Procedural GPU geometry** — positions computed in GLSL with no vertex buffers
- **GLSL ES 3.0 / WebGL 2.0** — `out`/`in` varyings, `gl_VertexID`, explicit fragment outputs
- **Uniform-driven animation** — time passed from JS to shaders for stateless, frame-based rendering
- **Shader interpolation** — per-vertex data smoothly blended across fragments by the rasterizer
- **`TRIANGLE_FAN` topology** — efficient fan geometry anchored to a center vertex
