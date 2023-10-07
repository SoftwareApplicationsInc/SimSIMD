# SimSIMD 📏

## Efficient Alternative to [`scipy.spatial.distance`][scipy] and [`numpy.inner`][numpy]

SimSIMD leverages SIMD intrinsics, capabilities that only select compilers effectively utilize. This framework supports conventional AVX2 instructions on x86, NEON on Arm, as well as __rare__ AVX-512 FP16 instructions on x86 and Scalable Vector Extensions on Arm. Designed specifically for Machine Learning contexts, it's optimized for handling high-dimensional vector embeddings.

- ✅ __3-200x faster__ than NumPy and SciPy distance functions.
- ✅ Euclidean (L2), Inner Product, and Cosine (Angular) spatial distances.
- ✅ Hamming (~ Manhattan) and Jaccard (~ Tanimoto) binary distances.
- ✅ Single-precision `f32`, half-precision `f16`, and `i8` vectors.
- ✅ Compatible with NumPy, PyTorch, TensorFlow, and other tensors.
- ✅ Has __no dependencies__, not even LibC.

[scipy]: https://docs.scipy.org/doc/scipy/reference/spatial.distance.html#module-scipy.spatial.distance
[numpy]: https://numpy.org/doc/stable/reference/generated/numpy.inner.html

## Benchmarks

### Apple M2 Pro

Given 10,000 embeddings from OpenAI Ada API with 1536 dimensions, running on the Apple M2 Pro Arm CPU with NEON support, here's how SimSIMD performs against conventional methods:

| Conventional                         | SimSIMD       | `f32` improvement | `f16` improvement | `i8` improvement |
| :----------------------------------- | :------------ | ----------------: | ----------------: | ---------------: |
| `scipy.spatial.distance.cosine`      | `cosine`      |          __39 x__ |          __84 x__ |        __196 x__ |
| `scipy.spatial.distance.sqeuclidean` | `sqeuclidean` |           __8 x__ |          __25 x__ |         __22 x__ |
| `numpy.inner`                        | `inner`       |           __3 x__ |          __10 x__ |         __18 x__ |

### Intel Sapphire Rapids

On the Intel Sapphire Rapids platform, SimSIMD was benchmarked against autovectorized-code using GCC 12. GCC handles single-precision `float` and `int8_t` well. However, it fails on `_Float16` arrays, which has been part of the C language since 2011.

|               | GCC 12 `f32` | GCC 12 `f16` | SimSIMD `f16` | `f16` improvement |
| :------------ | -----------: | -----------: | ------------: | ----------------: |
| `cosine`      |     3.28 M/s |   336.29 k/s |      6.88 M/s |          __20 x__ |
| `sqeuclidean` |     4.62 M/s |   147.25 k/s |      5.32 M/s |          __36 x__ |
| `inner`       |     3.81 M/s |   192.02 k/s |      5.99 M/s |          __31 x__ |

__Technical Insights__:

- Uses Arm SVE and x86 AVX-512's masked loads to eliminate tail `for`-loops.
- Substitutes LibC's `sqrt` calls with bithacks using Jan Kadlec's constant.
- Avoids slow PyBind11 and SWIG, directly using the CPython C API.
- Avoids slow `PyArg_ParseTuple` and manually unpacks argument tuples.

## Using in Python

### Installation

```sh
pip install simsimd
```

### Distance Between 2 Vectors

```py
import simsimd
import numpy as np

vec1 = np.random.randn(1536).astype(np.float32)
vec2 = np.random.randn(1536).astype(np.float32)
dist = simsimd.cosine(vec1, vec2)
```

Supported functions include `cosine`, `inner`, `sqeuclidean`, `hamming`, and `jaccard`.

### Distance Between 2 Batches

```py
batch1 = np.random.randn(100, 1536).astype(np.float32)
batch2 = np.random.randn(100, 1536).astype(np.float32)
dist = simsimd.cosine(batch1, batch2)
```

If either batch has more than one vector, the other batch must have one or same number of vectors.
If it contains just one, the value is broadcasted.

### All Pairwise Distances

For calculating distances between all possible pairs of rows across two matrices (akin to [`scipy.spatial.distance.cdist`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.spatial.distance.cdist.html)):

```py
matrix1 = np.random.randn(1000, 1536).astype(np.float32)
matrix2 = np.random.randn(10, 1536).astype(np.float32)
distances = simsimd.cdist(matrix1, matrix2, metric="cosine")
```

### Multithreading

By default, computations use a single CPU core. To optimize and utilize all CPU cores on Linux systems, add the `threads=0` argument. Alternatively, specify a custom number of threads:

```py
distances = simsimd.cdist(matrix1, matrix2, metric="cosine", threads=0)
```

### Hardware Backend Capabilities

To view a list of hardware backends that SimSIMD supports:

```py
print(simsimd.get_capabilities())
```

### Using Python API with USearch

Want to use it in Python with [USearch](https://github.com/unum-cloud/usearch)?
You can wrap the raw C function pointers SimSIMD backends into a `CompiledMetric`, and pass it to USearch, similar to how it handles Numba's JIT-compiled code.

```py
from usearch.index import Index, CompiledMetric, MetricKind, MetricSignature
from simsimd import pointer_to_sqeuclidean, pointer_to_cosine, pointer_to_inner

metric = CompiledMetric(
    pointer=pointer_to_cosine("f16"),
    kind=MetricKind.Cos,
    signature=MetricSignature.ArrayArraySize,
)

index = Index(256, metric=metric)
```
## Using SimSIMD in C

If you're aiming to utilize the `_Float16` functionality with SimSIMD, ensure your development environment is compatible with C 11. For other functionalities of SimSIMD, C 99 compatibility will suffice.

For integration within a CMake-based project, add the following segment to your `CMakeLists.txt`:

```cmake
FetchContent_Declare(
    simsimd
    GIT_REPOSITORY https://github.com/ashvardanian/simsimd.git
    GIT_SHALLOW TRUE
)
FetchContent_MakeAvailable(simsimd)
include_directories(${simsimd_SOURCE_DIR}/include)
```

Stay updated with the latest advancements by always using the most recent compiler available for your platform. This ensures that you benefit from the newest intrinsics.

Should you wish to integrate SimSIMD within USearch, simply compile USearch with the flag `USEARCH_USE_SIMSIMD=1`. Notably, this is the default setting on the majority of platforms.

## Upcoming Features

Here's a glance at the exciting developments on our horizon:

- [x] Exposing Hamming and Tanimoto bitwise distances to the Python interface.
- [ ] Intel AMX backend. Note: Currently, the intrinsics are functional only with Intel's latest compiler.

__To Rerun Experiments__ utilize the following command:

```sh
cmake -DCMAKE_BUILD_TYPE=Release -DSIMSIMD_BUILD_BENCHMARKS=1 -B ./build_release && make -C ./build_release && ./build_release/simsimd_bench
```

__To Test with PyTest__:

```sh
pip install -e . && pytest python/test.py -s -x
```
