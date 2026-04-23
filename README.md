
# Low-Overhead ASCII Cube

A high-performance 3D software rendering engine written in C. This project implements a custom 3D graphics pipeline optimized for resource-constrained environments, operating entirely within the integer pipeline to eliminate reliance on Floating Point Units (FPUs).

---

## Overview
I designed this as a constrained target project for the Arduino Uno R3, with a heavily optimized version intended to fit within ~1KB RAM (framebuffer + working state) using LUTs stored in flash and minimal stack usage (~300 bytes budget). The main reason FPU's were eliminated was to compensate for the board having a measely 16MHz CPU and a WHOLE 2KB OF RAM. I am pleased to say I have accomplished my goal of making it as light as possible, even though it looks a little choppy here and there because of the aggressive optimizations leaving the myth of "accuracy" to be lower on my priorities list.

Traditional terminal-based 3D visualizers often rely on point-cloud rendering—drawing discrete vertices—and standard floating-point mathematics. This engine implements a complete span-based software rasterizer. It processes continuous polygonal faces, applying perspective projection, backface culling, and per-pixel depth shading. 

The architecture is designed for direct portability to embedded systems and microcontrollers by utilizing fixed-point arithmetic, pre-computed lookup tables, and highly optimized I/O buffering routines.

![demo](cube.gif)


What currently bars me from porting the current code: 
- uses write()
- uses full terminal framebuffer
- assumes POSIX terminal

These will be adressed in a future commit.


I was inspired to do this after watching Code Fiction's video on him making a spinning ASCII cube in the terminal, I wanted a version that could run on the only microcontroller I had laying around: The Arduino Uno R3. Here is a link to his video:
<https://www.youtube.com/watch?v=p09i_hoFdd0>
Here is his github repo: <https://github.com/tarantino07/cube.c>

here is another example of a cube spinning, this time the buffer is wayyy above what the Arduino's puny 2 kilobytes of sram could ever dream of handling.
![demo](largeCube.gif)


---

## Mathematical Foundations

### Fixed-Point Arithmetic
Floating-point operations introduce significant latency on CPUs lacking dedicated hardware support. This engine circumvents this by utilizing fixed-point mathematics, primarily employing 16.16 and 8.8 fractional representations. Coordinate transformations and interpolations are executed using integer multiplication followed by bitwise shifts. This maintains sub-pixel precision while operating at integer execution speeds.

### Pre-computed Lookup Tables (LUTs)
To further reduce instruction cycles, complex mathematical functions are replaced with memory lookups:

1. **Trigonometry**: Sine and cosine values are stored in a 1.8.7 fixed-point array. Full 360-degree resolution is achieved via conditional mirroring of the pre-computed quadrant data.
2. **Division Avoidance**: Division is among the most expensive CPU instructions. During rasterization, calculating the Z-depth slope requires division by the horizontal width of a span. This is replaced by a reciprocal lookup table (`inv_width`), which stores the evaluation of `2^16 / x`. 

```c
// Division operation: dz = (z_diff) / width
// Optimized equivalent using LUT and bitwise shift:
dz = ((zB - zA) * (int32_t)inv_width[width]) >> 16;
```

---

## Code Walkthrough

The rendering pipeline is implemented sequentially per frame, converting mathematical 3D models into a 2D character array.

### 1. Vertex Transformation and Projection
The model data consists of 8 unit-cube vertices. For each frame, these vertices undergo rotation, translation, and projection.

```c
int16_t x1 = ((int32_t)X * cb - (int32_t)Z * sb) >> 8;
int16_t z1 = ((int32_t)X * sb + (int32_t)Z * cb) >> 8;
```
Vertices are rotated across the X and Y axes using the trigonometric LUTs. The values are then translated along the Z-axis to position the camera. Finally, perspective projection is applied using the formula `screen_x = (x * K) / z`, mapping the 3D coordinates into 2D screen space.

### 2. Backface Culling
Processing invisible faces wastes CPU cycles. Before rasterizing a quad, the engine computes the 2D cross-product of its screen-space vertices.

```c
if ((px[v1]-px[v0]) * (py[v2]-py[v0]) - (py[v1]-py[v0]) * (px[v2]-px[v0]) < 0) continue; 
```
If the cross-product yields a negative scalar, the surface normal points away from the camera, and the face is immediately discarded from the rendering loop.

### 3. Triangle Rasterization (Digital Differential Analyzer)
Quads are divided into two distinct triangles. To ensure consistent performance, the engine utilizes a top-bottom split rasterization technique. Vertices are sorted by their Y-coordinates, and the triangle is horizontally bifurcated at the middle vertex. This geometry guarantees that horizontal boundaries change at a constant rate, eliminating the need for complex branching logic within the inner loop.

### 4. Horizontal Span Filling
For each scanline of the rasterized triangle, the `draw_span` function is invoked to fill the row:

```c
int32_t current_z = zA;
for (int x = x_start; x <= x_end; x++) {
    int32_t z = current_z >> 16;
    current_z += dz; 
    // Luminance mapping logic...
}
```
This function interpolates the Z-depth linearly across the X-axis. `current_z` tracks the depth, incremented by the pre-calculated `dz` derivative for every horizontal step.

---

## Shading and Tone Mapping

To simulate lighting and depth, the engine maps the calculated Z-depth of each pixel to a specific character from a 12-character luminance ramp: `".,-~:;=!*#$@"`.

```c
int L = 10 - (z >> 10);
if (L < 0) L = 0; else if (L > 11) L = 11;
buf[offset + x] = ".,-~:;=!*#$@"[L];
```

The bit-shift operation `(z >> 10)` dynamically scales the depth integer into an index range. This specific wide-shift constant stabilizes the mapping, successfully hiding the mathematical seams where subdivided triangles meet and preventing character flickering during rotation.

---

## I/O and Memory Management

Standard output routines (such as `printf` or `putchar`) invoke excessive system calls and buffer flushing. This engine allocates a single contiguous block of memory to represent the framebuffer.

```c
#define PITCH (W + 1) 
#define N (H * PITCH)
static char buf[N];
```

The `PITCH` includes the visible width `W` plus a single byte for the newline character `\n`. During the buffer clearing phase, these newline characters are statically written into the appropriate memory addresses. Once the frame is drawn, it is dispatched to the terminal emulator using a single POSIX system call:

```c
write(1, buf, N);      
```

---

## Installation and Execution

### Compilation
The codebase is written in standard C and requires no external dependencies. Use optimization flags for the best performance.

```bash
gcc -O3 cube.c -o cube
```

### Execution
Run the compiled binary within a POSIX-compliant terminal emulator.

```bash
./cube
```

---

## License

This source code is provided as an open-source reference for embedded software rendering techniques and fixed-point mathematical operations.
