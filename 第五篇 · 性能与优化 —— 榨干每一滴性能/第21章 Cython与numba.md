### 第21章 Cython与numba：Python的性能涡轮增压器

> 📖 本章你将学会：
> - 理解Cython的编译原理：从Python超集到C扩展模块
> - 用Cython的两种语法模式加速数值循环
> - 用numba的@jit装饰器实现零编译成本的即时加速
> - 理解nopython模式与object模式的本质差异
> - 对比Cython、numba、纯Python、Java的适用场景

---

#### 第1节 为什么Python循环慢：根因分析

在动手加速之前，先理解纯Python循环为什么慢。这不是缺陷，而是架构选择的必然代价。

CPython执行一次 `a + b` 的完整路径：检查 `a` 和 `b` 的类型 → 查找 `__add__` 方法 → 调用该方法 → 创建新的整数对象 → 返回。每一步都涉及动态类型检查和对象操作[^cpython-eval-loop]。而C/Java的 `a + b`（int类型）编译为一条ADD指令。

这意味着纯Python循环中，每次迭代的类型检查、对象创建、方法查找开销远超实际计算。对于100万次迭代，这些开销累积为10-30倍的减速。

Cython和numba从不同角度解决这个问题：
- **Cython**：在编译期将类型固定为C类型，跳过运行时类型检查，生成纯C代码
- **numba**：在运行时（JIT）分析类型，编译为机器码，跳过解释器开销

两者本质都是"把动态类型变成静态类型"，只是时机不同——Cython是提前编译(AOT)，numba是即时编译(JIT)。

---

#### 第2节 Cython：用C的速度写Python

Cython是Python的超集——在Python语法基础上加入C类型声明，编译后生成C扩展模块[^cython-superset]。Cython 3.0于2023年7月正式发布，经过近五年开发，是近年来最重要的版本更新[^cython3-release]。

Cython支持两种语法模式：

**模式一：.pyx 语法（传统Cython语法）**

```python
# fast_sum.pyx —— Cython专用语法
def fast_sum(int n):
    cdef long total = 0  # C类型声明
    cdef int i
    for i in range(n):
        total += i
    return total
```

`cdef` 关键字声明C类型变量，编译后这些变量直接映射为C的 `long` 和 `int`，无Python对象开销。`def` 定义的函数可被Python调用，`cdef` 定义的函数仅在C层面可用（更快但不能直接从Python调用）。

**模式二：纯Python模式（Cython 3.0扩展）**

Cython 3.0大幅扩展了纯Python模式，使代码完全兼容标准Python语法，可用普通Python工具链（mypy、ruff、black等）检查[^cython3-pure]:

```python
# fast_sum.py —— 纯Python语法，既是合法Python也能被Cython编译
import cython

@cython.cfunc  # 编译为C函数（对应 cdef）
def fast_sum(n: cython.int) -> cython.long:
    total: cython.long = 0
    i: cython.int
    for i in range(n):
        total += i
    return total
```

纯Python模式的优势：代码在未编译时也能作为普通Python运行（`cython` 模块提供fallback），可以用mypy做类型检查，可以用IDE高亮——这些在传统 `.pyx` 语法中都不行。

**编译流程**：

```bash
# 方式1：setup.py 编译
# setup.py
from setuptools import setup
from Cython.Build import cythonize

setup(ext_modules=cythonize("fast_sum.pyx"))
```

```bash
python setup.py build_ext --inplace
# 生成 fast_sum.c → 编译为 fast_sum.so (macOS) / fast_sum.pyd (Windows)
```

```bash
# 方式2：pyximport 快速原型（开发时用）
import pyximport
pyximport.install()
import fast_sum  # 自动编译
```

---

#### 第3节 Cython类型声明的加速层次

Cython的加速效果取决于类型声明的深度，分三个层次：

**层次1：纯Python函数（无类型声明）**

```python
def sum_pure(n):
    total = 0
    for i in range(n):
        total += i
    return total
```

编译为C扩展后约2倍加速——主要收益来自跳过解释器调度循环，但变量仍是Python对象[^cython-benchmark]。

**层次2：加C类型声明**

```python
def sum_typed(int n):
    cdef long total = 0
    cdef int i
    for i in range(n):
        total += i
    return total
```

变量变为C类型，跳过对象创建和类型检查，约20-30倍加速[^cython-benchmark]。

**层次3：typed memoryview（类型化内存视图）**

```python
# 处理NumPy数组的最快方式
def sum_array(double[:] arr):  # typed memoryview 声明
    cdef double total = 0
    cdef int i
    cdef int n = arr.shape[0]
    for i in range(n):
        total += arr[i]  # 直接访问C数组，无Python对象开销
    return total
```

`double[:]` 是typed memoryview，告诉Cython "这是一个连续的double数组"。Cython生成直接指针访问的C代码，跳过NumPy的Python层面API，可达150-200倍加速[^cython-memoryview]。

| 优化层次 | 加速倍数 | 代码改动 | 适用场景 |
|---------|---------|---------|---------|
| 纯Python编译 | ~2x | 无改动 | 快速获得小幅提升 |
| 加C类型声明 | ~20-30x | 加cdef声明 | 数值循环 |
| typed memoryview | ~150-200x | 加memoryview声明 | NumPy数组遍历 |

> ⚡ **Java老兵类比**：Cython的 `cdef` 就像Java的原始类型声明——把 `Integer`（对象）换成 `int`（原始类型）。区别是Java在编译时自动决定，Cython需要你手动声明。

---

#### 第4节 Cython 3.0的新武器：nogil与NumPy ufunc

**nogil上下文管理器**——在代码段中释放GIL，允许Cython代码与Python线程真正并行执行[^cython3-nogil]：

```python
# cython_module.pyx
from cython.parallel import prange

def parallel_sum(double[:] arr):
    cdef double total = 0
    cdef int i
    cdef int n = arr.shape[0]
    with nogil:  # 释放GIL
        for i in prange(n, num_threads=4):  # 并行循环
            total += arr[i]
    return total
```

`with nogil` 块内的代码不持有GIL，`prange` 自动将循环分配到多线程。这对CPU密集型数值计算可实现接近线性的多核加速——在纯Python中这是不可能的（GIL限制）。

**NumPy ufunc生成**——Cython 3.0支持直接在Cython中编写NumPy ufunc，可高效应用到整个数组[^cython3-ufunc]:

```python
# 定义一个Cython函数，自动生成NumPy ufunc
@cython.cfunc
def square(x: cython.double) -> cython.double:
    return x * x
# 注册为NumPy ufunc后，可对整个数组批量执行
```

---

#### 第5节 numba：一行装饰器的即时编译

numba通过LLVM在运行时将Python函数编译为机器码。与Cython相比，numba不需要写 `.pyx` 文件，不需要手动编译——一行装饰器即可[^numba-jit]。

```python
from numba import jit

@jit(nopython=True)  # 强制nopython模式编译
def fast_sum(n):
    total = 0
    for i in range(n):
        total += i
    return total
```

**numba 0.59.0的重要变更**：从numba 0.59.0（2024年1月发布）开始，`nopython` 参数默认值从 `False` 改为 `True`[^numba-059]。这意味着：

```python
# numba 0.59.0+ —— 以下两种写法等价
@jit          # nopython默认为True
def f(x): ...

@jit(nopython=True)  # 显式声明，等价
def f(x): ...
```

旧代码中的 `@jit(nopython=False)` 会触发弃用警告，object mode fall-back 已被移除[^numba-059]。

**nopython模式 vs object模式**——这是理解numba的关键：

| 模式 | 编译结果 | 性能 | 限制 |
|------|---------|------|------|
| nopython | 纯机器码 | 50-150x加速 | 只支持数值类型和NumPy数组 |
| object | 混合代码 | 小幅提升 | 支持Python对象但性能有限 |

nopython模式下，numba将整个函数编译为不依赖Python解释器的机器码。如果函数中使用了不支持的Python特性（如自定义类、字符串操作），编译会失败。这是feature不是bug——它保证了真正的C级速度。

**更简洁的写法**——`@njit` 是 `@jit(nopython=True)` 的简写：

```python
from numba import njit

@njit  # 等价于 @jit(nopython=True)
def fast_sum(n):
    total = 0
    for i in range(n):
        total += i
    return total
```

---

#### 第6节 numba进阶：并行与GPU

numba的强大远不止单函数加速。

**自动并行化**——`@njit(parallel=True)` 自动识别可并行化的循环：

```python
from numba import njit, prange
import numpy as np

@njit(parallel=True)
def parallel_process(arr):
    result = np.empty_like(arr)
    for i in prange(len(arr)):  # prange 自动并行化
        result[i] = arr[i] ** 2 + np.sin(arr[i])
    return result
```

`prange` 是numba的并行range，自动将循环迭代分配到多线程。与Cython的 `prange` 类似，但numba版本不需要显式释放GIL——nopython模式本身就不持有GIL。

**GPU加速**——`@cuda.jit` 将函数编译为CUDA内核，在NVIDIA GPU上执行：

```python
from numba import cuda
import numpy as np

@cuda.jit
def gpu_add(a, b, result):
    idx = cuda.threadIdx.x + cuda.blockDim.x * cuda.blockIdx.x
    if idx < len(a):
        result[idx] = a[idx] + b[idx]

# 在GPU上执行
a = cuda.to_device(np.random.random(1000000))
b = cuda.to_device(np.random.random(1000000))
result = cuda.device_array_like(a)
gpu_add[blocks, threads](a, b, result)
```

> ⚠️ numba的GPU功能需要NVIDIA GPU和CUDA工具包。macOS上不可用。

---

#### 第7节 性能对比：四种方案实测

以下是一个典型数值循环（计算1到N的平方和）的性能对比，数据为典型场景示意值：

| 方案 | 100万次循环耗时 | 加速倍数 | 代码改动 |
|------|---------------|---------|---------|
| 纯Python | ~45ms | 1x (基准) | 无 |
| Cython (cdef声明) | ~1.5ms | ~30x | 加类型声明 |
| Cython (typed memoryview) | ~0.3ms | ~150x | 加memoryview |
| numba @njit | ~2ms | ~22x | 一行装饰器 |
| Java (int循环) | ~3ms | ~15x | — |

> ⚠️ **关于性能数据**：以上数据为典型场景示意值，实际效果取决于硬件配置、数据规模、代码模式和Python版本。请以你自己的实测结果为准。

几个值得注意的点：

Cython和numba都能让Python纯循环**超过Java**——因为它们编译为C/机器码，绕过了CPython解释器的动态类型开销。Java的15倍"仅"是相对纯Python的，而Cython/numba直接达到C级速度。

numba首次调用有编译开销（约0.5-2秒），后续调用直接执行机器码。对于只调用一次的函数，编译开销可能抵消加速收益。Cython提前编译无此问题。

Cython适合开发可复用的性能库——NumPy、Pandas、scikit-learn的底层都大量使用Cython[^cython-libs]。numba适合数据科学快速原型——一行装饰器，零编译配置。

---

#### 第8节 决策指南：Cython vs numba

| 维度 | Cython | numba |
|------|--------|-------|
| 编译时机 | 提前编译(AOT) | 即时编译(JIT) |
| 类型声明 | 手动(cdef/annotation) | 自动类型推断 |
| 首次调用开销 | 无（已编译） | 有（JIT编译~1s） |
| 代码改动 | 需要类型声明或.pyx文件 | 一行装饰器 |
| 调试体验 | .pyx语法不易调试 | 可设断点但有限 |
| NumPy集成 | typed memoryview极快 | 原生支持，同样快 |
| C库调用 | 原生支持(cimport) | 通过ctypes/cffi |
| 多线程 | nogil + prange | parallel=True + prange |
| GPU | 不支持 | @cuda.jit支持 |
| 适用场景 | 可复用性能库 | 快速原型/数据科学 |

**选择原则**：
- 如果在开发一个将被反复使用的库（如内部工具包）→ Cython（一次编译反复用，无JIT开销）
- 如果在Jupyter Notebook中做数据科学实验 → numba（一行装饰器，即时生效）
- 如果需要调用C/C++库 → Cython（cimport原生支持）
- 如果需要GPU加速 → numba（@cuda.jit）
- 如果需要释放GIL做多线程 → 两者都行（Cython的nogil或numba的parallel=True）

---

#### 本章小结

Cython和numba是Python的两大加速武器，本质都是"将动态类型变为静态类型，编译为C/机器码"。

Cython通过手动类型声明+提前编译，适合开发可复用的性能库。加速层次分三级：纯Python编译(~2x)→加类型声明(~30x)→typed memoryview(~150x)。Cython 3.0的纯Python模式让代码兼兼容标准Python工具链。

numba通过自动类型推断+即时编译，适合数据科学快速原型。`@njit`（nopython模式，0.59.0+默认）一行装饰器即可获得50-150倍加速。还支持自动并行化(`parallel=True`)和GPU加速(`@cuda.jit`)。

> 💡 **Java老兵记住**：Cython相当于在Java中用JNI写C扩展——功能强大但需要额外编译步骤。numba相当于Java的GraalVM JIT——运行时自动优化，无需改代码结构。两者都让Python在数值计算领域达到甚至超过Java的速度。

---

[^cython-superset]: Cython docs, "Language Basics", https://docs.cython.org/en/stable/src/userguide/language_basics.html — "Cython is a superset of the Python programming language."
[^cython3-release]: Cython 3.0 release notes, 2023-07, https://cython.readthedocs.io/en/latest/src/changes.html — "Cython 3.0 is better than any previous Cython version in every way."
[^cython3-pure]: Cython docs, "Pure Python Mode", https://docs.cython.org/en/stable/src/tutorial/pure.html — "Pure Python mode allows developers to use their existing Python linting and code analysis tools on Cython."
[^cython-benchmark]: Cython tutorial examples and community benchmarks. Typical results: pure Python compilation ~2x, cdef typed variables ~20-30x, typed memoryview ~150-200x. Actual results vary by workload.
[^cython-memoryview]: Cython docs, "Typed Memoryviews", https://docs.cython.org/en/stable/src/userguide/memoryviews.html — Typed memoryviews provide direct C-level access to buffer Protocol objects like NumPy arrays.
[^cython3-nogil]: Cython docs, "Parallel Programming", https://docs.cython.org/en/stable/src/userguide/parallelism.html — "with nogil" releases the GIL; "prange" enables parallel loops.
[^cython3-ufunc]: Cython 3.0 release notes, "Automatic NumPy ufunc generation", https://cython.readthedocs.io/en/latest/src/changes.html
[^numba-jit]: numba docs, "JIT compilation", https://numba.readthedocs.io/en/stable/user/jit.html — "@jit decorator compiles Python functions to machine code using LLVM."
[^numba-059]: numba 0.59.0 release notes, 2024-01-31, https://numba.readthedocs.io/en/stable/release/0.59.0-notes.html — "The default for the nopython key-word argument has been changed to True... Object mode fall-back support has been removed."
[^cython-libs]: Cython is used extensively in NumPy, Pandas, scikit-learn, and other scientific Python libraries for performance-critical code. See https://cython.org/#projects-using-cython
[^cpython-eval-loop]: CPython evaluates bytecode in the `_PyEval_EvalFrameDefault` loop, performing dynamic type checks on every operation. See CPython source: Python/ceval.c
