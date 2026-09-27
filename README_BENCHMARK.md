# ORB-SLAM3 — TUM RGB-D Benchmark (Ubuntu 24.04)

Fork of [UZ-SLAMLab/ORB_SLAM3](https://github.com/UZ-SLAMLab/ORB_SLAM3) with build fixes for modern toolchains, plus a full benchmarking run on the TUM RGB-D fr1_xyz sequence.

---

## Build Fixes Applied

The upstream source targets C++11 and was written against older Pangolin/Sophus APIs. Two changes were required to build on Ubuntu 24.04 (GCC 13, Pangolin 0.8+):

### 1. C++ standard: C++11 → C++14

**File:** `CMakeLists.txt`

Pangolin 0.8 uses `std::decay_t` and `std::enable_if_t`, which are C++14 features. Building with `-std=c++11` produces ~200 template errors inside Pangolin headers.

```cmake
# Before
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
...
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
add_definitions(-DCOMPILEDWITHC11)

# After
CHECK_CXX_COMPILER_FLAG("-std=c++14" COMPILER_SUPPORTS_CXX14)
...
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
add_definitions(-DCOMPILEDWITHC11 -DCOMPILEDWITHC14)  # both defined — see note below
```

> **Why both defines?** All example files (`rgbd_tum.cc`, `mono_euroc.cc`, etc.) guard
> `std::chrono::steady_clock` behind `#ifdef COMPILEDWITHC11`. Without that define the
> preprocessor falls through to the `#else` branch, which uses
> `std::chrono::monotonic_clock` — removed from the standard since C++11. Defining
> `COMPILEDWITHC11` activates the correct `steady_clock` path regardless of which
> standard version is actually in use.

### 2. Sophus CMakeLists

**File:** `Thirdparty/Sophus/CMakeLists.txt`

Matching C++ standard update so the Sophus sub-build uses the same flags as the main project.

---

## Running on TUM RGB-D

### Dataset
[TUM RGB-D fr1_xyz](https://cvg.cit.tum.de/data/datasets/rgbd-dataset/download) — a short desk sequence with primarily translational motion, used as the standard ORB-SLAM accuracy benchmark.

```
~/datasets/TUM/rgbd_dataset_freiburg1_xyz/
├── rgb/          # 792 color frames (640×480, 30 fps)
├── depth/        # aligned depth maps (factor 5000 → metres)
├── rgb.txt
├── depth.txt
└── groundtruth.txt
```

### Command

```bash
cd ~/ORB_SLAM3
LD_PRELOAD=/lib/x86_64-linux-gnu/libpthread.so.0 \
./Examples/RGB-D/rgbd_tum \
    Vocabulary/ORBvoc.txt \
    Examples/RGB-D/TUM1.yaml \
    ~/datasets/TUM/rgbd_dataset_freiburg1_xyz \
    Examples/RGB-D/associations/fr1_xyz.txt
```

> **`LD_PRELOAD` note:** On Ubuntu 24.04 with snap installed, Pangolin's Qt backend
> resolves `libpthread` from `/snap/core20/current/lib/…`, which is missing
> `__libc_pthread_init` (a glibc-private symbol). Preloading the system pthread
> prevents the symbol-lookup error. The process exits with a Qt timer segfault
> *after* trajectories are written — the output is unaffected.

---

## Results

### Tracking

| Metric | Value |
|---|---|
| Frames processed | 792 / 792 |
| Tracking failures | 0 |
| Keyframes | 60 |
| Initial map points | 837 |
| Median tracking time | 39.6 ms / frame (~25 fps) |
| Mean tracking time | 39.3 ms / frame |

### Absolute Trajectory Error (ATE)

Evaluated with [evo](https://github.com/MichaelGrupp/evo) against `groundtruth.txt`.

```bash
evo_ape tum groundtruth.txt CameraTrajectory.txt --align --correct_scale
```

| Metric | Sim(3) alignment | SE(3) alignment |
|---|---|---|
| **RMSE** | **0.0101 m** | **0.0102 m** |
| Mean | 0.0086 m | 0.0087 m |
| Median | 0.0075 m | 0.0077 m |
| Max | 0.0394 m | 0.0393 m |
| Scale correction | 1.007× | — |
| Poses compared | 790 / 792 | 790 / 792 |

**RMSE of 1.01 cm** — consistent with the ORB-SLAM3 paper (Table 1, fr1/xyz: ~0.010 m).

### Output Files

| File | Lines | Description |
|---|---|---|
| `CameraTrajectory.txt` | 792 | Full camera path (`timestamp tx ty tz qx qy qz qw`) |
| `KeyFrameTrajectory.txt` | 60 | Keyframe-only trajectory (same format) |

---

## Environment

| Component | Version |
|---|---|
| OS | Ubuntu 24.04 LTS |
| GCC | 13 |
| C++ standard | C++14 |
| OpenCV | system |
| Pangolin | 0.8 |
| Eigen | 3.x |
| Python (evo) | 3.12 |
| evo | latest |
