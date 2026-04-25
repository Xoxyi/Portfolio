+++
title = "Ray Tracing in One Weekend – My Implementation"
date = 2026-04-25
description = "A from-scratch ray tracer built by following Ray Tracing in One Weekend, covering spheres, diffuse/metallic/dielectric materials, reflections, refractions, and anti-aliasing with plans to parallelise it using CUDA, OpenMP, MPI, and SIMD."
+++

## Overview

I followed the [*Ray Tracing in One Weekend*](https://raytracing.github.io/books/RayTracingInOneWeekend.html) tutorial and implemented a basic ray tracer from scratch.  
This project helped me understand fundamentals like rays, intersections, materials, and lighting in a simple case of offline rendering.

## What I Built

- A simple ray tracer that renders spheres
- Diffuse, metallic, and dielectric materials
- Reflections and refractions
- Anti-aliasing
- Basic camera modeling

## Final Render

![Final Image](/images/raytracing/final.jpg) 

## What's Next

This was just a first step. Now I want to focus on making it faster.

My plan is to implement the ray tracer a few more times using different approaches:

- CUDA (run it on the GPU)
- OpenMP (use multiple CPU threads)
- MPI (run it across multiple processes/machines)
- Vector instructions (SIMD on the CPU)

## Goal

I want to compare them and see:

- how much faster they are
- how well they scale with bigger images or scenes
- how hard they are to implement

The idea is to get a better feeling for how different parallel techniques affect performance.

## Code

You can find the full implementation here:  
[GitHub Repository](https://github.com/Xoxyi/Raytracing-in-one-weekend---My-implementation)
