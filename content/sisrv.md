+++
title = "SISRV – OpenGL Renderer"
date = 2025-01-01
description = "A physically-based OpenGL renderer built from scratch, featuring deferred shading, PBR, IBL, shadow mapping, SSAO, and more."
+++

# SISRV – OpenGL Renderer

**Authors:** Andrea Zito, Davide Camino  
**Affiliation:** Department of Computer Science, University of Turin

---

## Introduction

SISRV is a real-time renderer built from scratch in C++ on top of **OpenGL 4.5**. The goal was to implement a modern GPU rendering pipeline end-to-end — from loading a mesh with Assimp all the way to a final tone-mapped, bloom-composited image on screen.

The implementation was developed by following the [Learn OpenGL](https://learnopengl.com/) tutorial, which provided the foundation for many of the techniques used. Some of the reference images are taken from there, while all the final renders shown here were produced by us.

![image](../images/SISRV/rendering.png)

The implemented features include model loading, Phong/Blinn-Phong illumination, Physically Based Rendering (PBR), sky-box with environment/irradiance/specular maps, normal mapping, parallax mapping, shadow mapping, deferred shading, instancing, SSAO, and a full post-processing stack (HDR, gamma correction, bloom).

---

## Complete Rendering Pipeline

The diagram below summarises every stage of the pipeline assembled for this project.

![image](../images/SISRV/pipeline_completa.png)

The pipeline is organised into four main phases: **pre-computation**, **geometry**, **lighting**, and **post-processing**. The sections below describe each stage in order of execution.

---

## Phase 0 — Environment Pre-computation

Before the main render loop starts, an HDR equirectangular `.hdr` image is converted into a cubemap and baked into three look-up textures that the lighting pass will sample at runtime:

- **Irradiance map** — a low-resolution cubemap that stores the pre-convolved diffuse integral over the hemisphere, providing environment-based ambient light for IBL.
- **Pre-filter map** — a mip-mapped specular environment map whose levels encode progressively rougher reflections (first term of the split-sum approximation).
- **BRDF LUT** — a 2D look-up table storing the scale and bias to the Fresnel term as a function of roughness and $N \cdot V$ (second term of the split-sum approximation).

![image](../images/SISRV/ibl_brdf_lut.png)

These textures are computed once at startup via dedicated shaders (`equirectangular_to_cubemap.fs`, `irradiance_convolution.fs`, `prefilter.fs`, `brdf.fs`) and remain constant for the entire session.

---

## Phase 1 — Shadow Depth Pass

For each point light in the scene a **cube shadow map** is rendered. The geometry is drawn six times (once per cubemap face) from the light's point of view, writing only depth values. During the lighting pass, each fragment queries the corresponding depth cube to determine whether it is in shadow.

A small **shadow bias** is added to the stored depth to avoid self-shadowing ("shadow acne").

![image](../images/SISRV/shadow_mapping_acne.png)

---

## Phase 2 — Geometry Pass (Deferred Shading)

The scene is rendered once into a **G-Buffer** using deferred shading. Instead of computing lighting immediately, each fragment writes its shading data into a set of off-screen textures (Multiple Render Targets):

| Attachment | Content |
|---|---|
| `gPosition` | World-space position |
| `gNormal` | World-space normal (after normal mapping) |
| `gAlbedo` | Diffuse albedo + material type flag |
| `gSpecular` | Specular / roughness / metalness |  

This decouples geometry complexity from lighting complexity: adding more lights costs only one fullscreen quad pass each, not another full scene traversal.

![image](../images/SISRV/deferred_shading.png)

The geometry shader supports two material workflows selected per-object:

- **Phong / Blinn-Phong** (`obj_geometry.fs`) — classic ambient + diffuse + specular, with normal mapping and parallax mapping support.
- **PBR** (`pbr_geometry.fs`) — albedo, metalness, and roughness textures fed into the Cook-Torrance BRDF.

### Normal Mapping

Surface normals stored in a tangent-space texture are transformed into world space via the **TBN matrix** (Tangent-Bitangent-Normal), giving per-pixel lighting detail without any extra geometry.

$$\text{world} \\_ \text{normal} = TBN \cdot \text{texture} \\_ \text{normal}$$

![image](../images/SISRV/normal_mapping.png)

### Parallax Mapping

A height map offsets the UV coordinates along the view direction using ray-marching through depth layers, creating the illusion of geometric depth on flat surfaces.

![image](../images/SISRV/parallax_mapping_parallax_occlusion_mapping.png)

---

## Phase 3 — SSAO

Before lighting, **Screen-Space Ambient Occlusion** runs on the G-Buffer. 64 random samples are distributed in a hemisphere around each fragment; samples that fall behind nearby geometry contribute to a per-pixel occlusion factor. The raw SSAO buffer is then blurred to remove noise, and the result modulates the ambient term in the lighting pass.

![image](../images/SISRV/ssao_overview.png)

---

## Phase 4 — Lighting Pass

A fullscreen quad reads from the G-Buffer and accumulates light contributions from all point lights.

### Blinn-Phong lighting

The diffuse and specular terms are computed as:

$$D = k_d(\hat{N} \cdot \hat{L}), \qquad S = k_s(\hat{H} \cdot \hat{N})^n$$

The halfway vector $\hat{H}$ replaces the reflection vector $\hat{R}$ from classic Phong, producing a more physically plausible highlight shape.

![image](../images/SISRV/advanced_lighting_comparrison.png)

### PBR — Cook-Torrance BRDF

The full reflectance equation is evaluated per light:

$$L_o(p,\omega_o) = \int_\Omega \left(k_d \frac{c}{\pi} + \frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)}\right) L_i(p,\omega_i)\, n \cdot \omega_i \, d\omega_i$$

The specular term is composed of three functions:

- **D** — GGX Normal Distribution Function, modelling the statistical orientation of microfacet normals.
- **F** — Schlick Fresnel approximation, controlling the ratio of reflected vs. refracted light as a function of the view angle.
- **G** — Smith Geometry function, accounting for microfacet self-shadowing and masking.

$$D = \frac{\alpha^2}{\pi\left((n \cdot h)^2(\alpha^2 - 1) + 1\right)^2}, \qquad F \approx F_0 + (1 - F_0)(1 - h \cdot v)^5$$

![image](../images/SISRV/untitled.png)

### Image-Based Lighting (IBL)

The rendering equation is split into a diffuse and a specular part, each pre-computed in Phase 0:

$$L_o = k_d \frac{c}{\pi} \int_\Omega L_i\, n \cdot \omega_i\, d\omega_i \;+\; \int_\Omega \frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)} L_i\, n \cdot \omega_i\, d\omega_i$$

- **Diffuse IBL**: the irradiance map is sampled along the surface normal.
- **Specular IBL**: the pre-filter map is sampled at the mip level corresponding to the material roughness; the BRDF LUT supplies the view-dependent scale and bias. Together they implement the split-sum approximation.

---

## Phase 5 — Forward Pass

Objects that cannot go through deferred shading, such as the skybox and light source meshes, are rendered on top of the existing depth buffer in a standard forward pass. The **skybox** is drawn last, writing only to pixels not already covered by geometry, reusing the cubemap computed in Phase 0.

![image](../images/SISRV/cubemaps_reflection1.png)

---

## Phase 6 — Bloom

Fragments whose luminance exceeds a threshold are extracted into a separate buffer, **Gaussian blurred** in two passes (horizontal then vertical), and additively composited back onto the main image.

![image](../images/SISRV/bloom_steps.png)

---

## Phase 7 — Post-processing

A final fullscreen shader applies:

- **Tone mapping** — Reinhard or exposure-based mapping compresses HDR values into the displayable $[0, 1]$ range:

$$\text{Col}\_{rgb} = \frac{\text{Col}\_{rgb}}{\text{Col}\_{rgb} + 1} \qquad \text{or} \qquad \text{Col}\_{rgb} = 1 - e^{-\text{exposure} \cdot \text{Col}\_{rgb}}$$

- **Gamma correction** — the result is raised to the power of $1/2.2$ to account for the non-linear response of displays:

$$\text{Col}\_{rgb} = \text{Col}\_{rgb}^{1/\gamma}$$

![image](../images/SISRV/gamma_correction_example.png)

---

## Code Structure

The renderer is organised into lightweight C++ classes, each owning a specific piece of GPU state:

| Class | Responsibility |
|---|---|
| `Shader` | Compiles and links GLSL programs, uploads uniforms |
| `Model` / `Mesh` | Loads assets via Assimp; manages VBO/EBO/VAO |
| `GBuffer` | Owns the G-Buffer FBO and MRT textures |
| `LightingPass` | Fullscreen quad + lighting shaders |
| `SkyBox` | HDR → cubemap conversion, irradiance/prefilter bake |
| `ShadowMap` | Per-light depth cubemap |
| `Scene` | Holds objects, lights, and per-frame update logic |
| `Camera` | View/projection matrices, WASD + mouse control |

The main loop in `main.cpp` wires all classes together in the order described above.

---

## Result

![image](../images/SISRV/rendering1.png)

The final scene combines all techniques: PBR materials with IBL reflections, normal and parallax mapping on the brick walls, point-light shadow maps, SSAO in the corners, and bloom on bright emissive surfaces — running in real time at interactive frame rates.

You can find the full implementation here:  
[GitHub Repository](https://github.com/Xoxyi/SISRV)

