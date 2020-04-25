# Meep Python 基础教程

我们将通过几个使用Python接口的例子来演示计算场、透射/反射光谱和谐振模式的过程。这些例子主要是1d或2d模拟，只是因为它们比3d更快，而且它们说明了大部分的基本功能。更多涉及3D模拟的高级功能，请参见[Simpetus](http://www.simpetus.com/projects.html)项目页面。

## 目录

## Meep 库

Meep 使用 Python 脚本进行仿真，仿真涉及到指定器件的几何形状、材料、光源、监控器区域，以及其他一切必要的计算设置。Python 脚本提供的灵活性，可以为几乎任何应用定制仿真，特别是那些涉及参数扫描和优化的仿真。Python库，如 [NumPy](http://www.numpy.org/) 、[SciPy](https://www.scipy.org/) 和 [Matplotlib](https://matplotlib.org/) 等，可以用来增强仿真功能性，并将进行演示。大部分低级 C++ 接口的功能已经在 Python 中被抽象出来，这意味着你不需要成为一个有经验的程序员就可以设置仿真。在必要的时候，提供合理的默认值。

通常在Unix命令行中执行Meep模拟的过程如下

``` shell
unix% python foo.py >& foo.out
```

读取 Python 脚本 foo.py 并执行它，将输出保存到 foo.out 文件中。如果在交互式模式下设置仿真，你可以输入命令并立即看到结果，你需要通过shell终端使用IPython，或者通过浏览器使用Jupyter Notebook。如果你使用这些方法中的一种，你可以将教程中的命令粘贴进去，并在后面的教程中看到它们的作用。

## 波导区域

对于我们的第一个例子，让我们研究一下波导中的局部 [连续波（CW）](https://en.wikipedia.org/wiki/Continuous_wave) 源激发的场型-先是直的，然后弯曲。该波导将具有频率无关的相对介电常数ε=12，宽度为1μm。这个例子中的单位长度是1μm。另见[Units](https://meep.readthedocs.io/en/latest/Introduction/#units-in-meep)。

### 长直波导

仿真脚本在 [examples/straight-waveguide.py](https://github.com/NanoComp/meep/blob/master/python/examples/straight-waveguide.py)。 Jupyter Notenook 在 [examples/straight-waveguide.ipynb](https://nbviewer.jupyter.org/github/NanoComp/meep/blob/master/python/examples/straight-waveguide.ipynb)。

首先载入 Meep Libray。

``` Python
import meep as mp
```

我们可以开始指定每个仿真对象的计算区域（Cell）。我们要在一端放置场源，观察场在 x 方向上的波导向传播，所以让我们在 x 方向上使用一个长度为16μm的区域，给它一些传播距离。在y方向，我们只需要足够的空间，让边界不影响波导模式；让我们给它一个8μm的大小。

``` Python
cell = mp.Vector3(16,8,0)
```

**Vector3** 对象存储了三个坐标方向上每个区域的大小。这是一个在 x 和 y 中的2d区域，其中z方向的大小为0。

接下来我们添加波导。最常见的作法是，仿真结构由存储在几何对象中的一组GeometricObjects 指定。

``` Python
geometry = [mp.Block(mp.Vector3(mp.inf,1,mp.inf),
                     center=mp.Vector3(),
                     material=mp.Medium(epsilon=12))]
```

波导由一个大小为 ∞ × 1 × ∞ 的 **Block**（parallelepiped）指定，ε=12，以(0,0)为中心，即仿真区域的中心。默认情况下，任何没有物体的地方都是空气（ε=1），当然这可以通过设置default_material变量来改变。由此产生的结构如下图所示。

![](./material/Python-Tutorial-wvg-straight-eps-000000.00.png)

我们有了结构，需要使用 sources 对象指定当前的场源。最简单的就是增加一个单点源Jz。

``` Python
sources = [mp.Source(mp.ContinuousSource(frequency=0.15),
                     component=mp.Ez,
                     center=mp.Vector3(-7,0))]
```

我们设置场源的频率为0.15，并指定了一个 **ContinuousSource**，它只是一个固定频率的正弦波 **exp(-*iωt*)**，默认在 ***t=0*** 时打开。 回想一下，在 [Meep Units](https://meep.readthedocs.io/en/latest/Introduction/#units-in-meep) 中，角频率是以 2πc 为单位，相当于真空波长的倒数。因此，0.15 对应的真空波长约为 1/0.15=6.67 μm，或者说在 ε=12 的材料中，波长约为2μm——因此，我们的波导是半个波长宽的，使其成为单模。事实上，这种波导中的单模行为的截止点是可以分析求解的，对应的频率为1/2√11 或大约 0.15076。还需要注意的是，要指定一个 Jz，我们要指定一个分量 Ez（例如，如果我们想要一个磁流，我们会指定 Hx、Hy 或 Hz ）。电流位于（-7,0），这是在仿真区域的左边缘右侧1μm处 - 我们总是希望在场源和仿真区域边界之间留下一点空间，以保持边界条件不受干扰。

至于边界条件，我们要在单元格周围添加吸收边界。Meep 中的吸收边界是由 [完美匹配层](https://meep.readthedocs.io/en/latest/Perfectly_Matched_Layer/)（PML）来处理的--这根本不是一个真正的边界条件，而是在仿真区域的边缘添加一个虚构的吸收材料。为了在仿真区域的所有边上添加一个厚度为 1μm 的吸收层，我们可以这样做。

``` Python
pml_layers = [mp.PML(1.0)]
```

**pml_layers** 是一组PML对象——如果你想让PML图层只在单元格的某些边上，你可以有一个以上的PML对象，**e.g. mp.PML(thickness=1.0,direction=mp.X,side=mp.high)** 只在 +*x* 边上指定一个PML图层。重要的一点是：**PML图层在仿真区域内**，与已有对象重叠。所以，在这种情况下，我们的PML重叠了我们的波导，这是我们想要的，这样它就能正确地吸收波导模式。PML的有限厚度对于减少数值反射很重要。更多信息请参见 [完美匹配层](https://meep.readthedocs.io/en/latest/Perfectly_Matched_Layer/) 。

Meep 将在空间和时间上对这种结构进行离散化，这是由一个单一变量——分辨率（resolution）来指定的，它给出了每个距离单位的像素数。我们将这个分辨率设置为10个像素/μm，对应于约67个像素/波长，或在高折射率材料中约20个像素/波长。一般来说，在最高介电材料中，至少8个像素/波长是个不错的选择。这样，我们就可以得到一个 160×80 的仿真区域。

> 译者注：Meep 中的像素（Pixel）等价于 FDTD Solutions 中的 Mesh

``` Python
resolution = 10
```

最后要指定的对象是 **Simulation**，它是基于之前定义的所有对象。

``` Python
sim = mp.Simulation(cell_size=cell,
                    boundary_layers=pml_layers,
                    geometry=geometry,
                    sources=sources,
                    resolution=resolution)
```

我们准备运行仿真。仿真时间设为200（in Meep units）。

``` Python
sim.run(until=200)
```

应该不到一秒钟就能完成。我们可以用 NumPy 和 Matplotlib 库对场进行分析和可视化。

``` Python
import numpy as np
import matplotlib.pyplot as plt
```

首先我们将创建一个介电函数 ε 的图像，这涉及到使用 **get_array** 获取数据的切片，该例程输出到一个NumPy数组，然后显示结果。

``` Python
eps_data = sim.get_array(center=mp.Vector3(), size=cell, component=mp.Dielectric)
plt.figure()
plt.imshow(eps_data.transpose(), interpolation='spline36', cmap='binary')
plt.axis('off')
plt.show()
```

接下来，我们通过叠加介电函数来创建标量电场 Ez 的图像。我们使用 "RdBu" colormap，从暗红色（负）到白色（零）到深蓝色（正）。

``` Python
ez_data = sim.get_array(center=mp.Vector3(), size=cell, component=mp.Ez)
plt.figure()
plt.imshow(eps_data.transpose(), interpolation='spline36', cmap='binary')
plt.imshow(ez_data.transpose(), interpolation='spline36', cmap='RdBu', alpha=0.9)
plt.axis('off')
plt.show()
```

![](./material/Python-Tutorial-wvg-straight-ez-000200.00.png)

我们看到，场源激发了波导模式，但也激发了从波导上传播的辐射场。在边界处，由于PML的作用，场迅速归零。
