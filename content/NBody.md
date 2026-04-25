+++
title = "Parallelizing N-Body Simulation"
date = 2024-01-01
description = "A comparative study of parallel programming paradigms for N-body gravitational simulation, including OpenMP, CUDA, MPI, and AVX."
+++

# Parallelizing N-Body Simulation

**Authors:** Stefano De Rosa, Andrea Zito, Davide Camino  
**Affiliation:** Department of Computer Science, University of Turin

---

## Introduction

This work presents various approaches to simulating the N-body problem using different parallel programming paradigms:

- **Shared Memory** — OpenMP
- **GPU Acceleration** — CUDA
- **Distributed Memory** — MPI
- **Vectorization** — AVX instructions

We analyzed speedup, weak scalability, and strong scalability across these technologies, and studied the impact of `float` vs `double` precision on numerical stability.

---

## Background

The N-body problem involves simulating a system of N discrete bodies interacting through mutual gravitational forces. The force between bodies *i* and *j* is:

$$F_{ij} = G \frac{m_i m_j}{r_{ij}^2} \hat{r}_{ij}$$

The total force on body *i* is the sum over all other bodies:

$$F_i = m_i a_i = \sum_{\substack{j=1 \\ j \neq i}}^{N} F_{ij}$$

To integrate over time, we use the **Euler method**:

$$v_i(t + \Delta t) = v_i(t) + a_i(t)\,\Delta t$$
$$r_i(t + \Delta t) = r_i(t) + v_i(t)\,\Delta t$$

Total energy (kinetic + potential) and angular momentum are conserved quantities used to verify numerical stability:

$$E_{tot} = \sum_{i=1}^{N} \frac{1}{2} m_i v_i^2 - \sum_{i=1}^{N} \sum_{j=i+1}^{N} G \frac{m_i m_j}{r_{ij}}$$

---

## Algorithms

### Algorithm 1 — All-Pairs (Sequential)

The baseline algorithm computes pairwise accelerations in a double nested loop — **O(N²)** time complexity. Each body's velocity is updated after computing all force contributions, then positions are updated.

```
procedure AllPairs(P, Δt, t_end):
    for t = 0 to t_end by Δt:
        for each particle p in P:
            for each particle q in P \ {p}:
                add force contribution F(q → p)
            update velocity of body p
        for each particle p in P:
            update position of body p
        t = t + Δt
```

### Algorithm 2 — All-Pairs with OpenMP (Shared Memory)

The outer loop over particles is parallelized with `#pragma omp parallel for schedule(runtime)`. Work is partitioned using a block strategy, since each body requires the same amount of computation. The schedule can be tuned at runtime for the specific workload and hardware.

```
procedure AllPairsOpenMP(P, Δt, t_end):
    for t = 0 to t_end by Δt:
        #pragma omp parallel for schedule(runtime)
        for each particle p in P:
            for each particle q in P \ {p}:
                add force contribution F(q → p)
            update velocity of body p
        for each particle p in P:
            update position of body p
        t = t + Δt
```

### Algorithm 3 — All-Pairs with CUDA (GPU)

Two CUDA kernels handle the computation: `ComputeInteractions` assigns one thread per particle to compute all incoming forces and update velocity, while `UpdatePosition` advances particle positions. Particle data is transferred to the GPU once before the time loop and copied back after.

```
procedure ComputeInteractions(P, Δt):
    p = blockDim.x * blockIdx.x + threadIdx.x
    #pragma unroll
    for each particle q in P \ {p}:
        add force contribution F(q → p)
    update velocity of body p

procedure UpdatePosition():
    p = blockDim.x * blockIdx.x + threadIdx.x
    update position of body p

procedure AllPairsCUDA(P, Δt, t_end):
    CopyToGpu(positions)
    CopyToGpu(velocities)
    for t = 0 to t_end by Δt:
        ComputeInteractions(P, Δt)
        UpdatePosition()
        t = t + Δt
    CopyFromGpu(positions)
    CopyFromGpu(velocities)
```

### Algorithm 4 — MPI Version 3 (Broadcast + AllGather)

Rank 0 broadcasts all particle data at initialization. Each process computes forces and updates velocities for its assigned block of particles independently. After position updates, `MPI_Allgather` synchronizes all processes for the next time step. Communication occurs only once per time step.

```
procedure AllPairsMPI_v3(P, Δt, t_end):
    MPI_Broadcast(P, root=0)
    P' = rank-th block of bodies of P
    for t = 0 to t_end by Δt:
        for each particle p in P':
            for each particle q in P \ {p}:
                add force contribution F(q → p)
            update velocity of body p
        for each particle p in P':
            update position of body p
        MPI_Allgather(P, P')
        t = t + Δt
```

### Algorithm 5 — MPI Version 2 (Scatter + Async Send/Recv)

Particles are scattered across processes. Each process asynchronously sends and receives particle blocks (`MPI_Isend` / `MPI_Irecv`), partially computing forces as chunks arrive via `MPI_Waitany`. Final data is gathered in rank 0 after the simulation ends.

```
procedure AllPairsMPI_v2(P, Δt, t_end):
    nproc = MPI_Comm_Size()
    localN = |P| / nproc
    MPI_Scatter(P, P', localN, root=0)
    for t = 0 to t_end by Δt:
        for i = 0 to nproc:
            MPI_Isend(P', localN, dest=i)
            MPI_Irecv(P,  localN, src=i, requests)
        for i = 0 to nproc:
            MPI_Waitany(requests, k)
            P_k = k-th block of P
            for each particle p in P':
                for each particle q in P_k:
                    add force contribution F(q → p)
                update velocity of body p
        for each particle p in P':
            update position of body p
        t = t + Δt
    MPI_Gather(P, P', root=0)
```

---

## Experiments

Experiments ran on the **hpc4ai** heterogeneous cluster, which includes:

| Partition | Nodes | CPU | RAM | GPU | Network |
|---|---|---|---|---|---|
| Broadwell | 68 | 2× Xeon-E5-2697 (18c) | 128 GB | — | Omni-Path 100 Gb/s |
| Cascadelake | 4 | 2× Xeon-Gold-6230 (20c) | 1.5 TB | T4 + V100S | InfiniBand 56 Gb/s |
| Epito | 4 | Ampere Altra Q80-30 (80c) | 500 GB | 2× A100 | InfiniBand 100 Gb/s |
| GraceHopper | 4 | Nvidia Grace Hopper | 500 GB | — | InfiniBand 100 Gb/s |

### CUDA: GPU Cores Filling Phase

![CUDA runtime when filling single GPU cores](/images/NBody/cuda_runtime_analysis_fill.png)

*CUDA runtime as the number of bodies grows toward full GPU utilization (GraceHopper node, single GPU). Our implementation outperforms NVIDIA's reference up to roughly 1000 bodies, where kernel launch overhead dominates over computation. Beyond that threshold, NVIDIA's tiled memory strategy takes over.*

### CUDA: Saturated GPU Regime

![CUDA runtime with saturated GPU cores](/images/NBody/cuda_runtime_analysis_regime.png)

*Once GPU cores are fully saturated, NVIDIA's tiled approach consistently achieves lower elapsed times. The performance gap widens with N, reflecting the efficiency advantage of shared-memory data reuse in the tiled kernel over our straightforward all-pairs implementation.*

### Float vs. Double Precision

![Energy conservation: float vs. double](/images/NBody/cuda_float_double.png)

*Total energy E_tot over 100 iterations for 2¹⁶ bodies (GraceHopper, single GPU). The `float` curve drifts significantly away from the theoretical value of −0.25, while `double` remains tightly conserved after an initial transient. This confirms that double precision is essential for numerically stable N-body simulations.*

### Distributed Memory: MPI Variants Comparison

![Runtime comparison of MPI approaches on Broadwell partition](/images/NBody/distrib_mem_comparison.png)

*Runtime with 2¹⁶ bodies across up to 64 Broadwell nodes. MPIv2 and MPIv3 behave nearly identically because both are bottlenecked by `MPI_Broadcast` (~7176 s), which dwarfs all other collective costs. The hybrid variant (MPIv3+OMP+AVX) breaks free of this bottleneck by combining distributed and shared memory parallelism, achieving consistently lower runtimes.*

### OpenMP vs. MPI — Strong Scalability

![Strong scalability: OpenMP vs. MPI speedup](/images/NBody/shared_vs_distributed_speedup.png)

*Speedup with 2¹⁶ bodies, sweeping from 2 to 64 processing units. OpenMP (Epito node, shared memory) tracks closer to the ideal linear curve than MPI (Broadwell nodes, one core each). Both fall well short of ideal, but the gap between them is driven entirely by communication cost: distributed memory synchronization is orders of magnitude more expensive than a shared-memory thread barrier.*

---

## Conclusion

| Approach | Strengths | Weaknesses |
|---|---|---|
| OpenMP | Low overhead, good speedup | Limited by shared memory capacity |
| CUDA | Massive parallelism, fast for large N | Memory management complexity |
| MPI | Scales across machines | High communication overhead |
| AVX | Vectorized throughput gains | Hardware-dependent |

The recommended strategy is a **hybrid approach**: combine MPI for distributed scaling across nodes with OpenMP (or CUDA) for intra-node parallelism. This yields the best of both worlds — broad scalability without sacrificing communication efficiency.

---

## References

1. D. Heggie and R. Mathieu, "Standardised units and time scales," in *The Use of Supercomputers in Stellar Dynamics*, Springer, 1986, pp. 233–235.
2. P. Hut, J. Makino, and S. McMillan, "Building a better leapfrog," *Astrophysical Journal Letters*, vol. 443, pp. L93–L96, 1995.
3. L. Nyland, M. Harris, and J. Prins, "Fast N-body simulation with CUDA," GPU Gems 3, Chapter 31, NVIDIA, 2007.
