---
layout: post
title: "Kubernetes CUDA IPC: The Permissions I Wish I Knew"
description: "Field-tested configurations and security tradeoffs for CUDA IPC on Kubernetes"
date: 2025-10-15
author: Harshal Patil
---

*Author: [Harshal Patil](https://github.com/harche){:target="_blank" rel="noopener"}*

I went down a rabbit hole trying to get CUDA Inter-Process Communication (IPC) working reliably in Kubernetes, without throwing security out the window. I tested four real setups, flipped every permission I could, broke things on purpose, and kept notes on what actually mattered.

This post is the write-up I wish I had at the start. It keeps all the gritty details, but tells the story like a human who has had `cudaIpcOpenMemHandle()` stare back with "invalid argument" at 2 a.m.

## TL;DR

- hostIPC is not required if you pass IPC handles via a shared file or volume.
- DRA (Dynamic Resource Allocation, Kubernetes 1.31+) removes the need for privileged containers.
- For single-pod setups, shareProcessNamespace is a safer substitute for hostPID.
- The most secure working setup: single pod + DRA + shareProcessNamespace, with both hostPID and privileged turned off.

If you just want working YAML, jump to Configuration #4.

## How the repo is laid out

This work lives in the repo: [cuda-ipc-debugging](https://github.com/harche/cuda-ipc-debugging). There are four self-contained examples with code and manifests:

```
cuda-ipc-debugging/
|-- shared-volume-example/          # #1 Multi-Pod, traditional device plugin
|-- dra-shared-gpu-example/         # #2 Multi-Pod, DRA
|-- single-pod-example/             # #3 Single Pod, traditional device plugin
`-- single-pod-dra-example/         # #4 Single Pod, DRA (best security)
```

Each directory has a README and runnable YAMLs. All findings below come from actually running these on Kubernetes and toggling permissions until things either worked or failed with specific errors.

## A quick mental model of the knobs

- hostIPC: Lets you join the host IPC namespace. For CUDA IPC we do not need this if handles are exchanged via files.
- hostPID: Lets a container see processes in the host PID namespace. CUDA IPC cares about process visibility; without it, the consumer cannot open the producer's handle in multi-pod setups.
- privileged: Drops most isolation. With the traditional NVIDIA device plugin, it is often used to let multiple containers touch the same physical GPU devices. It is a big security hammer.
- shareProcessNamespace: Shares a PID namespace across containers in the same pod. Crucially, this gives CUDA IPC the process visibility it needs without exposing the whole host.
- DRA: The newer Kubernetes (1.31+) way to allocate GPUs via ResourceClaims. It solves many of the device-sharing problems that used to force us into privileged mode.

With that out of the way, here is what I actually saw, configuration by configuration.

---

## Configuration #1: Multi-Pod, traditional device plugin

Two pods (producer and consumer), a hostPath volume to drop the handle file, and the classic NVIDIA device plugin managing GPU allocation.

### What worked in practice

- `hostPID` must be true; without it, the consumer cannot see the producer's process context and you will hit "invalid device context".
- `hostIPC` is not needed.
- `privileged` unfortunately is required. Without it, the two pods end up with access to different physical GPUs, and `cudaIpcOpenMemHandle()` returns "invalid argument". Privileged lets both pods see the device nodes they need to coordinate on the same physical GPU.

### Why privileged really matters here

The device plugin isolates GPUs per pod. I verified with `strace`, Bus-Id checks, and by toggling capabilities (SYS_ADMIN, MKNOD, DAC_OVERRIDE, SYS_RAWIO); none of those capabilities alone fixed it. Privileged did, because it bypasses the device isolation enough for both pods to land on the same card.

### Producer pod (example)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-ipc-producer-simple
spec:
  hostIPC: false
  hostPID: true
  containers:
  - name: producer
    image: nvidia/cuda:12.4.1-devel-ubuntu22.04
    resources:
      limits:
        nvidia.com/gpu: 1
    securityContext:
      privileged: true
      capabilities:
        add: ["IPC_LOCK"]
    volumeMounts:
    - name: shared-volume
      mountPath: /shared
  volumes:
  - name: shared-volume
    hostPath:
      path: /tmp/cuda-ipc-shared
      type: DirectoryOrCreate
```

### When I would reach for this

- Older clusters without DRA.
- Quick experiments or local testing.

### Security notes

This is the least secure option (privileged plus hostPID). Use it only when you must, and keep it isolated.

---

## Configuration #2: Multi-Pod with DRA

Still two pods and a shared volume for the handle file, but GPUs are allocated via a shared ResourceClaim using Kubernetes DRA.

### What changed with DRA

- `hostPID` still needs to be true for the producer and consumer to see each other's process context across pods. Disabling it produced "invalid device context".
- `hostIPC` continues to be unnecessary.
- `privileged` can be false. This is the big win. DRA gets both pods coordinated access to the same underlying GPU resources without smashing the isolation wall.

### GPU selection that stays sane

I used `CUDA_VISIBLE_DEVICES` so each pod sees only the GPU it is meant to touch:

```yaml
env:
- name: CUDA_VISIBLE_DEVICES
  value: "0"
```

```yaml
env:
- name: CUDA_VISIBLE_DEVICES
  value: "1"
```

Both containers can still use straightforward code like `cudaSetDevice(0)` internally.

### ResourceClaim (example)

```yaml
apiVersion: resource.k8s.io/v1beta1
kind: ResourceClaim
metadata:
  namespace: cuda-ipc-dra
  name: shared-dual-gpus
spec:
  devices:
    requests:
    - name: gpu-1
      deviceClassName: gpu.nvidia.com
    - name: gpu-2
      deviceClassName: gpu.nvidia.com
```

### Producer pod (sketch)

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: cuda-ipc-dra
  name: cuda-ipc-producer-dra
spec:
  hostIPC: false
  hostPID: true
  containers:
  - name: producer
    image: nvidia/cuda:12.4.1-devel-ubuntu22.04
    env:
    - name: CUDA_VISIBLE_DEVICES
      value: "0"
    resources:
      claims:
      - name: shared-gpus
    securityContext:
      privileged: false
      capabilities:
        add: ["IPC_LOCK"]
    volumeMounts:
    - name: shared-volume
      mountPath: /shared
  resourceClaims:
  - name: shared-gpus
    resourceClaimName: shared-dual-gpus
  volumes:
  - name: shared-volume
    hostPath:
      path: /tmp/cuda-ipc-shared-dra
      type: DirectoryOrCreate
```

### When I would use this

- Production pipelines that need separate pods.
- Places where privileged is a non-starter but cross-pod GPU IPC is required.

### Security notes

Much better than Configuration #1. You still need `hostPID`, but you can keep `privileged` off.

---

## Configuration #3: Single pod, traditional device plugin

One pod with two containers. The handle file lives in an `emptyDir`. This is where `shareProcessNamespace` shines.

### What mattered

- You need process visibility between containers. That can be `hostPID: true`, or (better) `shareProcessNamespace: true`.
- You still need `privileged: true` with the traditional device plugin; otherwise you will hit "invalid argument" opening the IPC handle.
- `hostIPC` is not required.

### Two minimal shapes that worked

Option A: hostPID

```yaml
spec:
  hostIPC: false
  hostPID: true
  containers:
  - name: producer
    securityContext:
      privileged: true
```

Option B: shareProcessNamespace (preferred)

```yaml
spec:
  hostIPC: false
  shareProcessNamespace: true
  containers:
  - name: producer
    securityContext:
      privileged: true
```

### Why shareProcessNamespace is a big deal

It gives each container in the pod the process visibility CUDA IPC needs, without exposing host processes. In single-pod scenarios it is strictly safer than `hostPID` and worked reliably in my tests.

### Sketch YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-ipc-concurrent-pod
spec:
  hostIPC: false
  shareProcessNamespace: true
  containers:
  - name: producer
    image: nvidia/cuda:12.4.1-devel-ubuntu22.04
    resources:
      limits:
        nvidia.com/gpu: 1
    securityContext:
      privileged: true
      capabilities:
        add: ["IPC_LOCK"]
    volumeMounts:
    - name: shared-volume
      mountPath: /shared
  - name: consumer
    image: nvidia/cuda:12.4.1-devel-ubuntu22.04
    resources:
      limits:
        nvidia.com/gpu: 1
    securityContext:
      privileged: true
      capabilities:
        add: ["IPC_LOCK"]
    volumeMounts:
    - name: shared-volume
      mountPath: /shared
  volumes:
  - name: shared-volume
    emptyDir: {}
```

### When I would use this

- Tightly coupled stages that should colocate.
- Simpler scheduling and resource guarantees in one pod.

### Security notes

Better with `shareProcessNamespace` than with `hostPID`, but still uses `privileged` due to the device plugin. If you can, jump to DRA.

---

## Configuration #4: Single pod with DRA (best security)

This is the setup I recommend if you can use Kubernetes 1.31+ and the NVIDIA GPU DRA driver. One pod, two containers, `emptyDir` for the handle file, and DRA to coordinate GPU access.

### What makes this great

- `hostPID: false` (default) - not needed.
- `hostIPC: false` (default) - not needed.
- `shareProcessNamespace: true` - gives the CUDA IPC process visibility, pod-scoped.
- `privileged: false` (default) - DRA means you do not need it.

### Minimal shape

```yaml
spec:
  hostIPC: false
  hostPID: false
  shareProcessNamespace: true
  containers:
  - name: producer
    securityContext:
      privileged: false
      capabilities:
        add: ["IPC_LOCK"]
```

### GPU selection without surprises

Both containers can see two devices but pick different ones. I used environment variables for clarity:

```yaml
env:
- name: CUDA_VISIBLE_DEVICES
  value: "0,1"
```

Then in code:

- Producer: `cudaSetDevice(0)` allocates and exports.
- Consumer: `cudaSetDevice(1)` imports and operates.

### ResourceClaim and pod (example)

```yaml
apiVersion: resource.k8s.io/v1beta1
kind: ResourceClaim
metadata:
  namespace: cuda-ipc-single-dra
  name: single-pod-dual-gpus
spec:
  devices:
    requests:
    - name: gpu-1
      deviceClassName: gpu.nvidia.com
    - name: gpu-2
      deviceClassName: gpu.nvidia.com
---
apiVersion: v1
kind: Pod
metadata:
  namespace: cuda-ipc-single-dra
  name: cuda-ipc-single-pod-dra
spec:
  shareProcessNamespace: true
  containers:
  - name: producer
    image: nvidia/cuda:12.4.1-devel-ubuntu22.04
    env:
    - name: CUDA_VISIBLE_DEVICES
      value: "0,1"
    resources:
      claims:
      - name: shared-gpus
    securityContext:
      capabilities:
        add: ["IPC_LOCK"]
    volumeMounts:
    - name: shared-volume
      mountPath: /shared
  - name: consumer
    image: nvidia/cuda:12.4.1-devel-ubuntu22.04
    env:
    - name: CUDA_VISIBLE_DEVICES
      value: "0,1"
    resources:
      claims:
      - name: shared-gpus
    securityContext:
      capabilities:
        add: ["IPC_LOCK"]
    volumeMounts:
    - name: shared-volume
      mountPath: /shared
  resourceClaims:
  - name: shared-gpus
    resourceClaimName: single-pod-dual-gpus
  volumes:
  - name: shared-volume
    emptyDir: {}
```

### When I would use this

- Production workloads in multi-tenant or regulated environments.
- Anywhere `privileged` is off-limits (which should be most places).

### Security notes

This hits the sweet spot: no host namespaces, no privileged containers, only pod-scoped process sharing.

---

## Cheat sheet

- Multi-Pod, traditional: `hostPID=true`, `privileged=true`, `hostIPC=false`.
- Multi-Pod, DRA: `hostPID=true`, `privileged=false`, `hostIPC=false`.
- Single pod, traditional: `shareProcessNamespace=true` (or `hostPID=true`), `privileged=true`, `hostIPC=false`.
- Single pod, DRA: `shareProcessNamespace=true`, `privileged=false`, `hostPID=false`, `hostIPC=false`.

Or, in table form:

| Configuration | hostIPC | hostPID | shareProcessNamespace | privileged |
|---------------|---------|---------|-----------------------|------------|
| #1 Multi-Pod, traditional | No | Yes | N/A | Yes |
| #2 Multi-Pod, DRA | No | Yes | N/A | No |
| #3 Single pod, traditional | No | Yes or No | Yes | Yes |
| #4 Single pod, DRA | No | No | Yes | No |

Notes:

- When I disabled `hostPID` in the multi-pod cases, I consistently saw "invalid device context".
- When I disabled `privileged` under the traditional device plugin, I saw "invalid argument" from `cudaIpcOpenMemHandle()`.
- `hostIPC` never mattered when the handle moved through a file or volume.

## Recommendations

- If you can, adopt DRA (Kubernetes 1.31+) and aim for Configuration #4. It is the cleanest and most secure.
- In single-pod scenarios, prefer `shareProcessNamespace` over `hostPID`.
- Avoid `privileged` unless you are stuck with the traditional device plugin and truly have no alternative.
- Treat `hostPID` as a last resort, and only where cross-pod visibility is unavoidable.

## Resources

- NVIDIA GPU DRA Driver: <https://github.com/NVIDIA/k8s-dra-driver>
- Kubernetes DRA docs: <https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/>
- Pod Security Standards: <https://kubernetes.io/docs/concepts/security/pod-security-standards/>

---

This post is based on hands-on testing of all four setups. The successes and failures (and their exact errors) are from real runs, not theory. If you spot something different in your cluster, I would love to hear about it.
