# Parallel Meep

Meep 通过 MPI 支持[分布式存储（Distributed memory）](https://en.wikipedia.org/wiki/Distributed_memory)并行化。 这使 Meep 可以从单个多核计算机扩展到多节点集群和超级计算机，并处理可能无法容纳在一台计算机内存中的超大问题。 如果需要，Meep 仿真可以使用数百个处理器。 当然，您的问题必须足够大才能[从许多处理器中受益](https://meep.readthedocs.io/en/latest/FAQ/#should-i-expect-linear-speedup-from-the-parallel-meep)。 （注意：在笔记环境中无法运行并行仿真。）

> 译者注：笔记环境指类似 Jupyter Notebook 的 IPython 环境

## 内容列表

## 安装并行 Meep

要从源码构建 Meep 的并行版本，你必须在系统上安装了 MPI 版本。有关概述，请参阅 [Build From Source/MPI](https://meep.readthedocs.io/en/latest/Build_From_Source/#mpi)。

如果你打算使用多个内核/处理器运行，我们强烈建议安装支持并行I/O的HDF5包。HDF5需要使用标志 `--enable-parallel` 进行配置。你可能还需要将 `CC` 环境变量设置为 `mpicc`。

> 译者注：
>
> CC环境： C Compiler 环境
>
> mpicc： 用来编译和链接用C语言编写的MPI程序

如果你没有安装支持并行 I/O 的 HDF5 ，你仍然可以从 MPI 中进行 I/O--Meep有办法使用多个进程的串行 I/O 来写入 HDF5 文件，一次一个一个地写入。然而，这并不能很好地扩展到多处理器。一些 MPI 实现程序在试图同时从多个进程写入的压力下会“冻结”。

那么你只需要用 `--with-mpi` 这个标志来配置 Meep。如果你运行生成的 Python 或 Scheme 脚本，它将在一个进程上运行；如果要在多核或多处理器上运行，你应该使用 `mpirun`，会在下一节讲述。因为你可以用这种方法在一个进程中运行并行的 Meep (例如，`mpirun -np 1 python foo.py` 或只是 `python foo.py`, `mpirun -np 1 meep foo.ctl` 或只是 `meep foo.ctl`)，所以不需要单独编译和安装 Meep 的串行版本。

## 使用并行版 Meep

并行版的 Meep 的设计是完全透明的（译者注：用户无感知的）：你使用与串行版相同的 Python 或 Scheme 脚本；输出是一样的，只是速度更快。在 Python中，每个非主进程(等级为0)的输出都会被发送到 `devnull`，而在Scheme中，print函数只打印主进程的输出。

为了运行 MPI 程序，通常需要使用像 mpirun 这样的命令，并带一个参数来指示你要使用多少进程。请参考你的 MPI 文档。例如，在许多流行的 MPI 实现程序中，要运行4个进程，你可以使用类似的命令。

### Python

``` Shell
mpirun -np 4 python foo.py > foo.out
```

### Scheme 
``` Shell
mpirun -np 4 meep foo.ctl > foo.out
```

有一个重要的要求：每个 MPI 进程必须能够读取 `foo.py`或`foo.ctl` 输入文件，或者是你的脚本文件的名字。在大多数系统中，这没有问题，但如果由于某些原因，你的 MPI 进程并不都能访问本地文件系统，那么你可能需要复制你的输入文件什么的。这个要求也适用于用于输入（例如 `epsilon_input_file`）或输出（例如`output_epsilon()`、`output_efield()`等）的HDF5文件。任何影响[网络文件系统](https://en.wikipedia.org/wiki/Network_File_System)的网络中断或个别机器上的磁盘故障都可能导致 Meep 冻结/挂起。

对于负载均衡的潜在改进，你可以尝试在仿真结构体中设置 `split_chunks_evenly=False`。

一般来说，你不能在多个处理器上交互运行 Meep 。

**警告**：当运行并行的 PyMeep 作业时，任何一个 MPI 进程的失败都可能导致f仿真死锁而不是中止。这是由于`mpi4py`的一个[行为](https://mpi4py.readthedocs.io/en/stable/mpi4py.run.html)造成的。为了避免手动杀死所有剩余的进程，一个简单的解决办法是在`mpirun`命令行中加载`mpi4py`模块（适用于3.0以上版本）。

``` Shell
mpirun -np 4 python -m mpi4py foo.py
```

## 不同形式的并行化

并行 Meep 的工作原理是将您的仿真区域在 MPI 进程中划分。这是并行化单个仿真的唯一方法，可以仿真非常大的问题。

然而，还有一种并行化的替代策略。如果你有许多较小的仿真，比如说你想运行许多不同的参数值，那么你可以将这些仿真作为单独的作业来运行。这种并行化被称为[embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel)，因为不需要通信。Meep 没有为这种运行模式提供明确的支持，但当然这很容易做到：只要启动多少个Meep作业就可以了，也许通过命令行使用 shell 脚本改变参数。

Meep 还支持[线程级并行（即多线程）](https://en.wikipedia.org/wiki/Task_parallelism)，在一台共享内存的多核机器上进行多频率的[近-远场](https://meep.readthedocs.io/en/latest/Python_User_Interface/#near-to-far-field-spectra)计算。

## 技术细节

当你在 MPI 下运行 Meep 时，以下是对幕后情况的简要描述。在大多数情况下，你不需要知道这些东西。只需使用与单处理器仿真相同的 Python/Scheme 脚本文件就可以了。

首先，每个 MPI 进程并行执行 Python/Scheme 文件。然而，这些进程之间的通信是为了只同步执行一个仿真。特别是，仿真区域被分成 "chunks"，每个进程一个，大致平均分配工作和内存。关于更多的细节，请参阅[《计算机物理通讯》第181卷第687-702页，2010年，第2.2节("网格块和拥有点")](http://ab-initio.mit.edu/~oskooi/papers/Oskooi10.pdf)。

通过 Python 的 `meep.Simulation.run(until=...)`或 Scheme 的`run-until`等命令,进行时间分阶时，各单元块的时间分阶是并行的，相互之间传递着边界上的像素值。一般情况下，任何对整个仿真区域或其中很大一部分执行某些集合操作的 Meep 函数都是并行化的，包括：时间分阶、HDF5 I/O、积累通量谱，以及通过`integrate_field_function(Python)`或`integrate-field-function(Scheme)`进行场积分，尽管*结果*是传递给所有进程的。

只涉及孤立的点的计算，如 `get_field_point` (Python) 或 `get-field-point` (Schemeas)，或 `harminv` (Python) 或 `harminv` (Scheas) 分析，都是由所有进程重复执行的。在 `get_field_point` 或 `get-field-point` 的情况下，Meep 会计算出哪个进程在给定的场区处 "知道 "该场，然后将该进程的场值发送给所有其他进程。这样做是无害的，因为这样的计算很少会出现性能瓶颈。

虽然所有进程都并行执行 Python/Scheme 文件，但除了一个进程(进程#0)之外，所有进程的打印语句都被忽略。这样一来，你只得到一个输出的副本。

有时，你只希望一个操作只在一个进程上发生。一个常见的用例是用 plt.show() 来显示 matplotlib 的绘图，或者用 plt.savefig() 来保存一个文件。如果你需要在 Python/cheme 文件中区分不同的 MPI 进程，你可以使用以下函数。

`meep.am_master()`, `(meep-am-am-master)` - 如果当前进程是主进程 (等级为 0)，则返回 true。

这对于调用外部的 I/O 或可视化例程（如Matplotlib绘图函数等）很有用，因为你只想在主进程上执行。请注意，Scheme `(print)` 或 Python 的 `print` 函数已经设置好了，所以默认情况下它们的输出在非主进程上被压制。

**警告**：大多数在仿真中操作的 Meep 函数 (例如 fields 或 structure ) 都是 "集体 "操作，必须以相同的顺序从所有进程中调用--如果只从一个进程中通过 `am_master` (或 `my_rank`) 校验调用它们，那么它们将[死锁](https://en.wikipedia.org/wiki/Deadlock)。`am_master` 校验里面的代码一般只能调用非Meep 库的函数。

`meep.count_processors()`, `(meep-count-processors)` - 返回 Meep 并行使用的进程数。

`meep.my_rank()`, `(meep-my-rank)` - 返回运行当前文件的进程索引，从0到(meep-count-processors)-1。

`meep.all_wait()`, `(meep-all-wait)` - 阻断直到所有进程执行此语句(MPI_Barrier)。

对于带 I/O 的大型多核作业，可能需要将 `(meep-all-wait)` 作为 Scheme 文件中的最后一行，以确保所有的处理器在执行的同一时刻终止。否则，一个处理器完成后可能会突然终止其他处理器。
