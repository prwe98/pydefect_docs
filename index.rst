Pydefect 使用向导
=================

这个教程展示了如何使用\ ``pydefect``\ 这个code

**Note1：Pydefect现在仅支持Vienna ab-initio
模拟包（VASP），因此我们需要使用VASP中的输入输出文件（例如POSCAR、POTCAR、OUTCAR）和其使用的计算技术（例如周期性边界条件）**

**Note2：Pydefect中使用的单位是eV表示能量，Angstrom表示长度，遵循VASP的惯例**

**Note3：仅支持非磁性主体材料**

非金属固体本征点缺陷计算的工作流程如下图所示。其中有一些任务可以用来并行计算，而另一些还需要等待其他任务，这个过程非常复杂且耗时，研究人员很容易犯一些错误。Pydefect的主要目的是为了给研究点缺陷计算的研究人员提供一种自动化的过程，以节省时间并减少错误。

.. figure:: https://i.loli.net/2021/06/15/UTISVN1BODGbtLK.png
   :alt: 

在这里我们假设有下面的目录树，<project_name>通常是目标材料的晶体结构。例如，金红石-TiO2

.. code:: 

   <project_name>
    │
    ├ pydefect.yaml
    ├ vise.yaml
    │
    ├ unitcell/ ── structure_opt/
    │            ├ band/
    │            └ dos/
    │
    ├ competing_phases/ ── <competing_phase 1>
    │                    ├── <competing_phase 2>
    │                    ....
    │
    └ defects/ ── perfect/
                 ├─ Va_X_0/
                 ├─ Va_X_1/
                 ├─ Va_X_2/
                ...

我们建议用户遵循相同的目录结构。整个计算的过程细节展示可以通过使用PBEsol泛函计算MgSe的示例来展示。

.. _1-原胞松弛:

1. 原胞松弛
-----------

点缺陷的计算通常在给定的函数和PAW电位下理论松弛结构中进行，因为它没有引起对超胞尺寸没有依赖性的人工应变和应力。因此，通常从优化晶格常数和晶胞中原子位置的分数坐标开始。

我们首先准备原始的块体原胞的\ ``POSCAR``\ ，并且创建一个\ ``unitcell/``\ 目录和\ ``unitcell/structure_opt/``\ 子目录（\ ``mkdir -p unitcell/structure_opt/``\ ），之后移动到子目录。（\ **注：本教程中以\ ``/``\ 结尾的名称表示目录**\ ）当\ ``Pydefect``\ 需要构建vasp输入文件，即\ ``INCAR``\ ，\ ``POTCAR``\ ，\ ``KPOINTS``\ 文件时，我们使用vise（=
``vasp intergrated supporting environment``\ ）代码，生成各种任务和交换相关（XC）函数的输入文件。\ ``Vise``\ 依赖于Pymatgen，因此，如

所示，我们需要在主目录的\ ``.pmgrc.yaml``\ 文件中设置\ ``PMG_DEFAULT_FUNCTIONAL``\ 和\ ``PMG_VASP_PSP_DIR``\ ，例如：

.. code:: 

   PMG_DEFAULT_FUNCTIONAL: PBE_54
   PMG_MAPI_KEY: xxxxxxxxxxxxxxxx
   PMG_VASP_PSP_DIR: /home/kumagai/potcars/

在\ ``Pydefect``\ 中，查询\ ``POSCAR``\ 文件和材料的总能量需要实用\ ``PMG_MAPI_KEY``\ 。

使用PBEsol函数优化原胞的输入文件由以下命令生成：

.. code:: 

   vise vasp_set -x pbesol

其中\ ``vasp_set``\ 或其缩写\ ``vs``\ ，是\ ``vise``\ 主函数的子命令选项。同样，所有的子命令在\ ``pydefect``\ 和vise中都有自己的缩写。在这里，PBE函数是\ ``vise``\ 中的默认值，因此我们使用\ ``-x``
参数将XC函数切换到PBEsol。每个子命令也有许多参数，可以通过 ``-h``
参数来调取帮助获取细节。

.. code:: 

   pydefect vs -h

**Note：**\ 结构优化通常必须以比1.3倍还大的cutoff energy
进行迭代，直到力和应力在第一个离子步骤收敛。有关详细信息请参阅

。\ ``Pydefect``\ 不支持vasp计算这种迭代，但可以编写简单的runsell脚本来实现。

在\ ``Pydefect``\ 中，用户可以通过\ ``vise``\ 中的vise,yaml文件控制各种默认参数。有关详细信息，请参阅

可控变量参见\ ``pydefect``\ 中的\ ``default.py``\ 。

.. _2-能带dos和介电张量的计算:

2. 能带，DOS，和介电张量的计算
------------------------------

计算能带结构（BS）、态密度（DOS）、
和介电常数。在缺陷的计算中，BS用于确定价带顶（VBM）和导带底（CBM），而静态介电常数，或ion-clamped和离子介电张量的总和，用于校正缺陷形成能。

首先，我们在unitcell/中创建band/，dos/和dielectric/文件夹，并且从unitcell/structure_opt/复制POSCAR文件，在每个目录下输入以下命令

.. code:: 

   vise vs -x pbesol -t <band, dos or dielectric_dfpt>

``Vise`` 还提供 BS 和 DOS的绘图功能. 在 `document of
vise <https://kumagai-group.github.io/vise/>`__ 中查看细节。

.. _3-收集与点缺陷相关的原胞信息:

3. 收集与点缺陷相关的原胞信息
-----------------------------

接下来，我们使用unitcell（=
u）子命令来收集大量信息，即带边缘和ion-clamped和离子介电张量

.. code:: 

   pydefect u --vasprun_band band/vasprun.xml --outcar_band band/OUTCAR --outcar_dielectric_clamped dielectric/OUTCAR --outcar_dielectric_ionic dielectric/OUTCAR

在这里，可以使用不同的OUTCAR文件设置ion-clamped和离子介电常数。然后，生成uitcell.json用于分析缺陷计算。一般json文件可读性价差，所以我们实现了print（=p）子命令，从json文件生成刻度的命令行输出，如下。

.. code:: 

   pydefect p -f unitcell.json

原胞信息展示如下

.. code:: 

   Unitcell(vbm=0.5461, cbm=3.0807, ele_dielectric_const=[[4.645306, 0.0, 0.0], [0.0, 4.645306, -0.0], [0.0, -0.0, 4.645306]], ion_dielectric_const=[[2.584237, -0.0, -0.0], [-0.0, 2.584192, -0.0], [-0.0, -0.0, 2.584151]])

用户有时候想要手动设置原胞参数，在这种情况下，使用python
script或是ipython设置参数，转储yaml文件，如下：

.. code:: python

   In [1]: from pydefect.analyzer.unitcell import Unitcell

   In [2]: u = Unitcell(vbm=3.0675,cbm=7.7262, ele_dielectric_const=[[3.157296,0,0],[0,3.157296,0],[0,0,3.157296]], ion_dielectric_const=[[6.811496,0,0]
      ...: , [0, 6.811496,0], [0,0,6.811496]])

   In [3]: u.to_json_file()

.. _4-计算竞争相:

4. 计算竞争相
-------------

当引入缺陷时，原子与热力学框架内的假设的原子库交换。
在大多数情况下，为了计算与缺陷形成能近似的缺陷形成自由能，我们需要确定与产生缺陷相关的原子化学势。
通常，我们考虑竞争相与主体材料共存条件下的化学势，由化学势图确定。

为此，我们在\ ``competition_phases/`` 中创建目录。 我们可以从Materials
Project (MP) 中检索稳定或略微不稳定的竞争相的 POSCAR。 为此，需要 MP 的
API 密钥。 在这里，我们获得了与 MgSe 竞争的材料，其凸包上方的能量小于
0.5 meV/atom，使用

.. code:: 

   pydefect mp -e Mg Se --e_above_hull 0.0005

此命令创建以下目录：

.. code:: 

   Mg149Se_mp-1185632/ MgSe_mp-13031/ Mg_mp-1094122/ Se_mp-570481/

每个目录下都有POSCAR和prior_info.yaml。 prior_info.yaml 包含了 Materials
Project 数据库中的一些信息，这对于确定第一性原理计算条件很有用。

比如， ``Mg_mp-1094122/prior_info.yaml`` ：

.. code:: 

   band_gap: 0.0
   data_source: mp-1094122
   total_magnetization: 0.00010333333333333333

这意味着 Mg 是一种非磁性金属系统。 ``Vise`` 解析\ ``prior_info.yaml``
并通过INCAR 中的\ ``ISPIN`` 标签确定\ ``KPOINTS`` 中的k
点密度和自旋极化。

请注意，O2、H2、N2、NH3 和 NO2 分子不是从 MP 中提取的，而是由
``pydefect`` 产生的，因为这些分子在 MP
中已计算为固体，这可能不足以用于缺陷计算的竞争相。

之后为竞争的固体和分子生成 ``INCAR``\ 、\ ``POTCAR``\ 、\ ``KPOINTS``
文件。 请注意，我们使用常规的截止能量 ENCUT 来比较总能量（total
energy），该能量增加到\ ``POTCARs`` 组分之间最大 的\ ``ENMAX`` 的 1.3
倍。 MgSe，Mg 和 O 的 ``ENMAX`` 为 200.0 和 211.555 eV，因此我们需要设置
``ENCUT`` = 275.022，使用vise：

.. code:: 

   for i in *_*/;do cd $i; vise vs -uis ENCUT 275.022 -x pbesol ; cd ../;done

本例中的 MgSe 已经计算完毕，因此我们不必重复相同的计算； 在删除
``MgSe_mp-13031/`` 后通过 ``ln -s ../unitcell/structure_opt MgSe``
创建符号链接。 但是，如果我们用不同的 ``ENMAX``
计算它来使得其与更大的掺杂原子 ``ENMAX`` 保持一致，这里就需要重新计算。

**Note：**\ 如果竞争相是气体，我们需要将 ``ISIF`` 更改为
2，以免晶格常数松弛（参见

），并将 ``KPOINTS`` 更改为 Gamma 点采样。
这里是通过\ ``prior_info.yaml``\ 使用\ ``vise``\ 自动调整的。

完成\ ``vasp``\ 计算后，我们可以使用\ ``make_cpd(= mcpd)``\ 子命令生成化学势图的json文件：

.. code:: 

   pydefect mcpd -d *_*/

将\ ``vasprun.xml``\ 和\ ``OUTCAR``\ 文件重命名，例如：\ ``vasprun-finish.xml``\ 和\ ``OUTCAR-finish``\ ，此时需要在\ ``pydefect.yaml``\ 文件中写入以下内容：

.. code:: yaml

   # VASP file names
   outcar: OUTCAR-finish
   vasprun: vasprun-finish.xml

要绘制化学势图，请使用 ``plot_cpd`` (= ``pcpd``) 子命令：

.. figure:: https://i.loli.net/2021/06/15/It8ZAjBPudvsETO.png
   :alt: 

.. figure:: https://i.loli.net/2021/06/15/6KeOhAUNd5n4cHz.png
   :alt: 

此时，顶点处的相对化学势显示如下：

.. code:: 

   +----+---------+--------+---------+
   |    |   mu_Ba |   mu_O |   mu_Sn |
   |----+---------+--------+---------|
   | A  |  -5.927 |  0     |  -4.966 |
   | B  |  -5.581 |  0     |  -5.312 |
   | C  |  -3.124 | -2.59  |   0     |
   | D  |  -5.352 | -0.114 |  -5.198 |
   | E  |  -2.753 | -2.713 |   0     |
   | F  |  -3.558 | -2.37  |  -0.226 |
   | G  |  -3.503 | -2.4   |  -0.189 |
   +----+---------+--------+---------+

如果需要修改化学势图的能量，可以直接修改\ ``vertices_MgO.yaml``\ 文件。

竞争相的计算通常很费力，有时我们想尽快粗略地检查缺陷形成能。
``Pydefect`` 支持从 Materials Project 数据库创建化学势图。
然而，要做到这一点，需要准备调整元素能量标准所需的原子能量。

使用\ ``vise``\ ，可以轻松准备原子计算目录。 在这里，我们展示了 BaSnO3
的示例：

.. code:: 

   vise map -e Ba Sn O

然后创建vasp输入文件：

.. code:: 

   for i in */;do cd $i; vise vs ; cd ../;done

运行 vasp。 使用 python 脚本将原子能收集到 yaml 文件中。

.. code:: python

   # -*- coding: utf-8 -*-
   #  Copyright (c) 2020. Distributed under the terms of the MIT License.

   from pymatgen import Element
   from pymatgen.io.vasp import Outcar

   for e in Element:
       try:
           o = Outcar(str(e) + "/OUTCAR-finish")
           name = str(e) + ":"
           print(f"{name:<3} {o.final_energy:11.8f}")
       except:
           pass

假设输出保存到 ``atom_energies.yaml``\ 。 然后使用以下命令生成
``cpd.yaml`` 文件。

.. code:: 

   pydefect mcpd -e Ba Sn O -t BaSnO3 -a atom_energies.yaml

.. _5-构建超胞和缺陷初始设置文件:

5. 构建超胞和缺陷初始设置文件
-----------------------------

我们已经完成了晶胞和竞争相的计算，最终准备进行点缺陷计算。
让我们创建\ ``defect/``\ 目录并从例如复制unitcell ``POSCAR``\ 文件
``unitcell/dos/``\ 到\ ``defect/``

然后，我们使用 ``supercell`` (= ``s``) 和\ ``defect_set`` (= ``ds``)
子命令创建超胞和缺陷类型等相关文件。 ``Pydefect``
推荐由中等数量的原子组成的近乎各向同性（有时类似于立方体）的超胞。
使用以下命令，可以创建 ``SPOSCAR`` 文件

.. code:: 

   pydefect s

如果输入结构与标准化原胞不同，会引发 ``NotPrimitiveError``\ 错误。

``pydefect``\ 是通过扩展\ **惯用原胞**\ （\ *conventional*
unitcell）来构建超胞。

可以改变超胞的晶格角，而不是惯用原胞的晶格角。
例如，我们可以制作一个超胞，其中 a、b 和 c 轴在立方晶系中相互正交。
然而，这对于点缺陷计算并不是一个好的情况，因为这种晶格打破了原始的对称性，降低了点缺陷计算的准确性，并且难以分析缺陷位点的对称性。
pydefect 中的一个例外是四方晶系，可将超胞旋转45度来保持原始对称性。

在\ ``pydefect``\ 中，用户可以指定晶胞矩阵：

.. code:: 

   pydefect s --matrix 2 1 1

该矩阵适用于惯用原胞。如果想要知道惯用原胞，键入：

.. code:: 

   pydefect s --matrix 1

来检视更多细节。

``supercell_info.json`` 文件包含有关超胞的完整信息，可以使用 ``-p``
选项查看这些信息。

.. code:: json

   Space group: F-43m
   Transformation matrix: [-2, 2, 2]  [2, -2, 2]  [2, 2, -2]
   Cell multiplicity: 32

      Irreducible element: Mg1
           Wyckoff letter: a
            Site symmetry: -43m
            Cutoff radius: 3.373
             Coordination: {'Se': [2.59, 2.59, 2.59, 2.59]}
         Equivalent atoms: 0..31
   Fractional coordinates: 0.0000000  0.0000000  0.0000000
        Electronegativity: 1.31
          Oxidation state: 2

      Irreducible element: Se1
           Wyckoff letter: c
            Site symmetry: -43m
            Cutoff radius: 3.373
             Coordination: {'Mg': [2.59, 2.59, 2.59, 2.59]}
         Equivalent atoms: 32..63
   Fractional coordinates: 0.1250000  0.1250000  0.1250000
        Electronegativity: 2.55
          Oxidation state: -2

使用\ ``defect_set``\ （=
``ds``\ ）子命令，构建\ ``defect_in.yaml``\ 文件。MgSe的\ ``defect_in.yaml``\ 如下

.. code:: yaml

   Mg_Se1: [0, 1, 2, 3, 4]
   Se_Mg1: [-4, -3, -2, -1, 0]
   Va_Mg1: [-2, -1, 0]
   Va_Se1: [0, 1, 2]

其中显示了缺陷类型及其电荷。 如有必要，我们可以使用编辑器进行修改。
如果我们想掺杂，可以输入如下：

.. code:: 

   pydefect ds -d Ca

有一些与\ ``supercell_info.json`` 和\ ``defect_in.yaml``
相关的注意事项：

1. 反位点缺陷和取代缺陷由取代和去除原子之间的电负性差异确定。
   默认最大差异写在 ``defaults.py`` 中，但可以通过 ``pydefect.yaml``
   更改它，如上所述。

2. 氧化态决定缺陷电荷态。 例如，Sn2+ 的空位（间隙）可以采用 0、-(+)1 或
   -(+)2 电荷态，而 Sn4+ 的空位（间隙）则介于 0 和 -(+)4 电荷态之间。
   对于反位点和替代缺陷，\ ``pydefect`` 考虑空位和间隙的所有可能的组合。
   因此，例如，Sn2+ -on-S2- 具有 0、+1、+2、+3 和 +4 电荷态。 使用
   ``pymatgen`` 中 Composition 类的 ``oxi_state_guesses``
   方法确定氧化态。 用户也可以手动设置氧化态如下：

.. code:: 

   pydefect ds --oxi_states Mg 4

然而，在某些情况下，电荷状态的范围可能不够。 例如，已知 ZnO 中的 Zn
空位显示 +1 电荷态，因为它们可以在相邻的 O 位点捕获多个极化子。 参见

用户必须自己添加这些异常值。

1. 默认情况下，与缺陷相邻的原子的位置被扰动，使得对称性降低到 P1。
   然而，这在某些情况下是不需要的，因为它增加了不可约
   k-points的数量然后，需要通过 ``pydefect.yaml`` 将
   ``displace_distance`` 设置为 0。

2. 如果你想计算特定的缺陷，例如，只有氧空位，你可以用 ``-k`` 选项和
   python 正则表达式来限制计算的缺陷，例如，当输入如下时，

.. code:: python

   pydefect ds -k "Va_O[0-9]?_[0-9]+"

创建这些目录。

.. code:: 

   perfect/ Va_O1_0/ Va_O1_1/ Va_O1_2/

.. _6-决定间隙位点:

6. 决定间隙位点
---------------

除了空位和反位点，人们可能还想考虑间隙。
大多数人通过观察主体晶体结构来确定它们，有一些程序也可以推荐间隙位点。
然而，推测最可能的间隙位点通常不是一件容易的事，因为它们取决于被取代的元素。

最大的空位应该是带有封闭壳层的带正电阳离子（例如
Mg2+、Al3+）的间隙位点，因为它们往往不会与其他原子形成牢固的键合。
另一方面，质子 (H+) 更喜欢位于 O2- 或 N3- 附近以形成强的 O-H 或 N-H 键。
相反，氢化物离子 (H-) 应该更喜欢位于不同的位置。
因此，我们需要仔细确定间隙位置。

``pydefect`` 拥有一个实用程序，它使用 pymatgen 中实现的
``ChargeDensityAnalyzer`` 类，根据晶胞中的所有电子电荷密度推荐间隙位点。
为此，我们需要基于标准化的原胞生成 ``AECCAR0`` 和 ``AECCAR2``\ 。

也可以在 DOS 计算中添加此任务。 ``vise``\ 的命令是：

.. code:: 

   vise vs -uis LAECHG True -t dos

这不应该在 BS 计算中完成，因为原胞可能与特定空间群中的标准化原胞不同。

运行vasp计算后，运行\ ``pydefect``\ 中的\ ``recommote_interstitials.py``

.. code:: 

   python pydefect/cli/vasp/util_commands/recommend_interstitials.py AECCAR0 AECCAR2

，其显示电荷密度的局部极小点如下。

.. code:: 

             a         b         c  Charge Density
   0  0.750000  0.750000  0.750000        0.527096
   1  0.500000  0.500000  0.500000        0.669109
   2  0.611111  0.611111  0.166667        1.020380
   3  0.166667  0.611111  0.611111        1.020382
   4  0.611111  0.166667  0.611111        1.020382
   Host symmetry R3m
   ++ Inequivalent indices and site symmetries ++
     0   0.7500   0.7500   0.7500 3m
     1   0.5000   0.5000   0.5000 3m
     2   0.6111   0.6111   0.1667 .m

再次注意，局部最小值可能不是某些特定间隙的最佳初始点，用户必须注意到此过程的限制。

要在例如 0.75 0.75 0.75
处添加间隙位点，其中分数坐标基于标准化原胞，我们使用间隙 (= ``i``)
子命令，如

.. code:: 

   pydefect ai -s supercell_info.json -p ../unitcell/structure_opt/POSCAR -c 0.75 0.75 0.75

然后更新 ``supercell_info.json``\ ，其中包括间隙位点的信息。

.. code:: json

   ...
   -- interstitials
   #1
   Fractional coordinates: 0.3750000  0.3750000  0.3750000
           Wyckoff letter: c
            Site symmetry: -43m
             Coordination: {'Mg': [2.59, 2.59, 2.59, 2.59], 'Se': [3.0, 3.0, 3.0, 3.0, 3.0, 3.0]}

如果我们想添加另一个位点，例如 0.5 0.5 0.5 ，
在\ ``supercell_info.json``\ 再次输入 。

要弹出间隙位点，使用：

.. code:: 

   pydefect pi -i 1 -s supercell_info.json

从 ``supercell_info.json`` 中删除了位于 (0.75, 0.75, 0.75)
的第一个间隙位点。

.. _7-缺陷计算目录的创建:

7. 缺陷计算目录的创建
---------------------

我们接下来使用\ ``defect_entries``\ （=
``de``\ ）子命令为点缺陷计算创建目录，

.. code:: 

   pydefect de

使用该命令创建缺陷计算目录，包括\ ``perfect/``\ 。

如果再次键入相同的命令，则会出现以下信息，

.. code:: 

   2020/11/24 20:40:27    INFO pydefect.cli.vasp.main_function
    --> perfect dir exists, so skipped...
   2020/11/24 20:40:27    INFO pydefect.cli.vasp.main_function
    --> Va_Se1_1 dir exists, so skipped...
   2020/11/24 20:40:27    INFO pydefect.cli.vasp.main_function
    --> Va_Se1_2 dir exists, so skipped...
   2020/11/24 20:40:27    INFO pydefect.cli.vasp.main_function
    --> Va_Se1_0 dir exists, so skipped...
   ...

没有新创建的目录。 这是一种防失误处理，以免误删除计算出的目录。
如果确实要重新创建目录，则需要先删除或移动原目录。

在每个目录中，可以找到\ ``defect_entry.json``
文件，该文件包含有关运行第一性原理计算之前获得的点缺陷的信息。
要查看\ ``defect_entry.json``\ ，请再次使用\ ``-p`` 选项。

当你想添加一些特定的缺陷时，你可以修改\ ``defect_in.yaml``\ 并再次输入\ ``de``\ 选项。

.. _8-生成defectentryjson文件:

8. 生成defect_entry.json文件
----------------------------

有时，人们可能想要处理复杂的缺陷。 例如，O2 分子在 MgO2
中充当阴离子，其中 O2 分子空位能够存在。 还有其他例子，比如甲基铵卤化铅
(MAPI)，其中甲基铵离子充当单个正阳离子 (CH3NH3+) 和 DX
中心，其中阴离子空位和阳离子间隙共存。

在这些情况下，需要准备输入文件并自己运行 vasp 计算。
但是，\ ``pydefect`` 需要\ ``defect_entry.json``
文件用于后处理，用户无法轻松生成该文件。

为此，\ ``pydefect`` 提供了 ``create_defect_entry.py``\ ，它解析
``POSCAR`` 文件和缺陷名称：

.. code:: 

   python $PATH_TO_FILE/create_defect_entry.py complex_2 complex_2/POSCAR perfect/POSCAR

它创建了\ ``defect_entry.json`` 文件。 然后将目录名称解析为

.. code:: 

   A_B -> name='A', charge=B

可以使用这个脚本来分析正在进行的缺陷计算。

.. _9-解析超胞计算结果:

9. 解析超胞计算结果
-------------------

然后，让我们运行 vasp 计算。

要创建 vasp 输入文件，请键入

.. code:: 

   for i in */;do cd $i; vise vs -t defect ; cd ../;done

不要忘记添加 ``-t defect``\ ，为缺陷创建输入文件。

运行 vasp 时，如果 k point仅在大型超胞的 Gamma 点采样，我们建议用户使用
Gamma-only vasp。

在（部分）完成 Vasp
计算后，我们可以生成包含与缺陷属性相关的第一性原理计算结果的
``calc_results.json``\ 。

通过使用 ``calc_results`` (= cr) 子命令，我们可以在所有计算的目录中生成
``calc_results.json``\ 。

.. code:: 

   pydefect cr -d *_*/ perfect

当想要为某些特定目录（例如 Va_O1_0）生成 ``calc_results.json``
时，请键入

.. code:: 

   pydefect cr -d Va_O1_0

.. _10-有限尺寸超胞缺陷形成能量的修正:

10. 有限尺寸超胞缺陷形成能量的修正
----------------------------------

当在周期性边界条件下采用超胞方法时，由于缺陷、其图像和背景电荷之间的相互作用，带电缺陷的总能量无法正确估计。
因此，我们需要将带电缺陷超胞的总能量修正为稀释极限内的能量。

使用\ ``extended_fnv_correction`` (= ``efnv``) 子命令进行校正，

.. code:: 

   pydefect efnv -d *_*/ -pcr perfect/calc_results.json -u ../unitcell/unitcell.json

对于修正，我们需要无缺陷超胞中的静态介电常数和原子位点电位。
因此，必须分配到无缺陷超胞的\ ``unitcell.json``
和\ ``calc_results.json`` 的路径。 注意，此命令需要运行一段时间。

此时 ``pydefect`` 中的能量校正现在使用所谓的扩展
Freysoldt-Neugebauer-Van de Walle (eFNV) 方法进行。
如果使用更正，请引用以下论文。

-  `C. Freysoldt, J. Neugebauer, and C. Van de Walle, Fully Ab Initio
   Finite-Size Corrections for Charged-Defect Supercell Calculations,
   Phys. Rev. Lett., 102 016402
   (2009). <https://journals.aps.org/prl/abstract/10.1103/PhysRevLett.102.016402>`__

-  `Y. Kumagai\* and F. Oba, Electrostatics-based finite-size
   corrections for first-principles point defect calculations, Phys.
   Rev. B, 89 195205
   (2014). <https://journals.aps.org/prb/abstract/10.1103/PhysRevB.89.195205>`__

获取更正.pdf
文件，其中包含有关缺陷诱导和点电荷电位的信息，以及它们在原子位点的差异，如下所示。

.. figure:: https://i.loli.net/2021/06/15/fRkEiHtAgq4BopJ.png
   :alt: 

水平线的高度表示点电荷电位与缺陷引起的电位之间的平均电位差，即有缺陷超胞的电位减去无缺陷超胞的电位。
线的范围表示平均区域。 有关详细信息，请参阅\ `Y. Kumagai\* and F. Oba,
Electrostatics-based finite-size corrections for first-principles point
defect calculations, Phys. Rev. B, 89 195205
(2014). <https://journals.aps.org/prb/abstract/10.1103/PhysRevB.89.195205>`__\ 。

在进行更正时，强烈建议您检查所有更正.pdf
文件中的计算缺陷，以尽可能减少错误。

.. _11-检查超胞计算中的缺陷特征值和能带边缘状态:

11. 检查超胞计算中的缺陷特征值和能带边缘状态
--------------------------------------------

通常，点缺陷分为三种类型。

(1)
带隙内具有深局域态的缺陷。这种类型的缺陷通常被认为不利于器件性能，因为载流子被定域态俘获。此外，它们可以作为色心，如
NaCl 中的空位所示。因此，了解局部状态的位置及其起源很重要。

(2) 具有氢载流子状态或扰动主状态 (PHS)
的缺陷，其中载流子位于带边缘，被带电缺陷中心松散地捕获。例如，Si 中的
B-on-Si（p 型）和 P-on-Si（n
型）置换掺杂剂。这些缺陷对器件性能也几乎没有危害，但会引入载流子电子/空穴或杀死源自小俘获能量的反载流子。
PHS
的波函数广泛应用于数百万个原子。因此，为了计算它们的热力学转变能级，我们需要超巨超胞计算，到目前为止，这几乎是第一性原理计算所禁止的。因此，我们通常避免计算这些量，并表示缺陷具有
PHS，并且它们的跃迁能量仅定性地位于带边缘附近。

(3)
带隙内或带边缘附近没有任何缺陷状态的缺陷，只要它们的浓度不是太高，不会对电子特性产生很大影响。

请参阅我们已发表论文中的一些示例。

-  `Y. Kumagai*, M. Choi, Y. Nose, and F. Oba, First-principles study of
   point defects in chalcopyrite ZnSnP2, Phys. Rev. B, 90 125202
   (2014). <https://link.aps.org/pdf/10.1103/PhysRevB.90.125202>`__

-  `Y. Kumagai*, L. A. Burton, A. Walsh, and F. Oba, Electronic
   structure and defect physics of tin sulfides: SnS, Sn2S3, and SnS2,
   Phys. Rev. Applied, 6 014009
   (2016). <https://link.aps.org/doi/10.1103/PhysRevApplied.6.014009>`__

-  `Y. Kumagai*, K. Harada, H. Akamatsu, K. Matsuzaki, and F. Oba,
   Carrier-Induced Band-Gap Variation and Point Defects in Zn3N2 from
   First Principles, Phys. Rev. Applied, 8 014015
   (2017). <https://journals.aps.org/prapplied/abstract/10.1103/PhysRevApplied.8.014015>`__)

-  `Y. Kumagai*, N. Tsunoda, and F. Oba, Point defects and p-type doping
   in ScN from first principles, Phys. Rev. Applied, 9 034019
   (2018). <https://journals.aps.org/prapplied/abstract/10.1103/PhysRevApplied.9.034019>`__

-  `N. Tsunoda, Y. Kumagai*, A. Takahashi, and F. Oba, Electrically
   benign defect behavior in ZnSnN2 revealed from first principles,
   Phys. Rev. Applied, 10 011001
   (2018). <https://journals.aps.org/prapplied/abstract/10.1103/PhysRevApplied.10.011001>`__

要区分这三种缺陷类型，需要查看缺陷能级并判断缺陷是否会产生 PHS 或
缺陷局部状态。

``Pydefect`` 通过以下步骤显示特征值和能带边缘状态。

首先，可以使用以下命令生成 ``band_edge_eigenvalues.json`` 和
``eigenvalues.pdf`` 文件。

``eigenvalues.pdf`` 文件：

.. figure:: https://i.loli.net/2021/06/15/umJhSebvPkDYUCZ.png
   :alt: 

这张图可以看到，单粒子能级及其在自旋向上和向下通道中的占位。 x
轴是计算出的 k points的分数坐标，而 y 轴是绝对能量标度。
图中实心圆点是每个 k point的单个粒子的能级。

两条水平虚线表示无缺陷的超胞（\ **perfect
supercell**\ ）中的价带顶和导带底。图中离散的数字表示从 1
开始的能带指数，红色、绿色和蓝色圆点分别表示被占据、部分被占据（从 0.1
到 0.9）和未被占据的本征态。

然后使用以下命令生成 ``edge_characters.json`` 文件：

.. code:: 

   pydefect make_edge_characters -d *_*/ -pcr perfect/calc_results.json

并使用此命令分析文件并显示能带边缘状态：

.. code:: 

   pydefect edge_states -d *_*/ -p perfect/edge_characters.json

.. code:: json

   -- Mg_i1_0
   spin up   Donor PHS
   spin down Donor PHS
   -- Mg_i1_1
   spin up   Donor PHS
   spin down No in-gap state
   -- Mg_i1_2
   spin up   No in-gap state
   spin down No in-gap state
   -- Va_Mg1_-1
   spin up   No in-gap state
   spin down In-gap state
   -- Va_Mg1_-2
   spin up   In-gap state
   spin down In-gap state
   -- Va_Mg1_0
   spin up   No in-gap state
   spin down In-gap state

有四个状态\ ``donor_phs``\ 、\ ``acceptor_phs``\ 、\ ``localized_state``\ 、no_in_gap，前两个被认为是浅能级状态，在图中应被略去。

在\ ``pydefect``\ 中，这些状态由最高占位和最低未占位特征值以及最高占用（最低未占用）状态和\ **VBM**\ （\ **CBM**\ ）的波函数的相似性确定。

在此我们强调，自动确定的带边状态可能是不正确的，因为通常很难自动确定它们。
因此，请仔细检查带边状态，如果带边状态不明显，请绘制它们的能带分解电荷密度。

能带边缘状态可以通过每个缺陷目录中的 ``band_edge_states.yaml``
文件进行修改，在绘制缺陷形成能量时将对其进行解析。

.. _12-绘制缺陷形成能:

12. 绘制缺陷形成能
------------------

在这里，我们展示了如何绘制缺陷形成能（defect formation energy）。

缺陷形成能量图需要多种信息，即能带边缘、竞争相的化学势以及无缺陷和有缺陷超胞的总能量。

使用 ``plot_energy`` (= ``pe``) 子命令将缺陷形成能绘制为费米能级函数

.. code:: 

   pydefect e --unitcell ../unitcell/unitcell.json --perfect perfect/calc_results.json -d Va*_* -c ../competing_phases/cpd.yaml -l A

.. figure:: https://i.loli.net/2021/06/15/cGvsgpauUDHR5W4.png
   :alt: 

当改变化学势的条件，即化学势图中顶点的位置时，使用 ``-l`` 选项。

第一性原理计算点缺陷相关提示
============================

.. _1-如何处理点缺陷的对称性:

1. 如何处理点缺陷的对称性
-------------------------

正如 `Tutorial of
pydefect <https://kumagai-group.github.io/pydefect/tutorial.html>`__\ 中提到的，缺陷附近的相邻原子最初会受到轻微扰动以破坏对称性。
然而，在结构优化过程中，一些缺陷往往会回到对称原子配置或恢复部分对称操作。

即使在这些情况下，最终结构的对称性也并不明显。 ``Pydefect``
提供了一个允许对缺陷结构进行对称化的脚本：

.. code:: 

   python $PYDEFECT_PATH/pydefect/cli/vasp/util_commands/make_refined_poscar.py

如果结构不是 P1 对称，此命令将创建对称 ``POSCAR`` 文件。 然后，之前的
``OUTCAR`` 和 CONTCAR 分别重命名为 ``OUTCAR.sym_1`` 和
``CONTCAR.sym_1``\ 。

也可以在 runshell 脚本中包含此命令，例如，

.. code:: shell

   $VASP_cmd

   hostname > host
   name=`basename "$PWD"`
   if [ $name != "perfect" ]; then
       python $PYDEFECT_PATH/pydefect/cli/vasp/util_commands/make_refined_poscar.py
       if [ -e CONTCAR.sym_1 ]; then
           $VASP_cmd
       fi
   fi

.. _2-混合函数计算的技巧:

2. 混合函数计算的技巧
---------------------

混合泛函(Hybrid functionals)，尤其是 HSE06
泛函，以及具有不同交换混合参数或筛选距离的泛函，也经常用于点缺陷计算。

通常，混合泛函计算比基于局部或半局部密度近似的泛函计算成本要高几十倍。
因此，我们需要花点心思来降低他们的计算成本。

为此，我们定期准备使用 GGA 函数获得的 WAVECAR 文件。
（虽然我们也可以事先使用 GGA
放宽原子位置，但它可能不适合点缺陷计算，因为 GGA
计算的缺陷的位点对称性可能与混合泛函不同。）

例如，可以使用 GGA 使用以下命令创建 INCAR 文件以生成 WAVECAR 文件。

.. code:: 

   grep -v LHFCALC INCAR | grep -v ALGO | sed s/"NSW     =  50"/"NSW     =   1"/ > INCAR-pre

计算垂直跃迁能级向导
====================

我们在此以 NaCl 为例说明如何计算垂直跃迁能级 (**VTL**)。 对于 VTL
的计算，我们需要应用特殊的校正方案，这里我们称之为 GKFO 校正。 请阅读
`T. Gake, Y. Kumagai*, C. Freysoldt, and F. Oba, Phys. Rev. B, 101,
020102(R)
(2020) <https://kumagai-group.github.io/pydefect/link.aps.org/doi/10.1103/PhysRevB.101.020102>`__\ 获取详情。

假设已经按照教程中的介绍完成了基于 PBEsol 泛函的 NaCl
中的重要缺陷的计算，并且进一步希望通过中性电荷态的 Cl 空位计算光吸收能。

.. code:: 

   NaCl
    │
    ├ unitcell/ ── unitcell.json
    │
    └ defects/ ── perfect/
                └ Va_Cl_0/ ── absorption/

首先，在 ``Va_Cl_0/`` 创建 ``absorption/``\ 目录并从 ``Va_Cl_0/`` 复制
vasp 输入文件。 然后，编辑 ``INCAR`` 将 ``NSW`` 更改为 1，并添加
``NELECT`` 标记，将其从中性标记减少 1。 或者可以输入：

.. code:: 

   vise vs -uis NSW 1 --options charge -1 -d ../ -t defect

运行vasp后，我们然后在\ ``absolute/``\ 目录中使用以下命令创建\ ``calc_results.json``\ 。

.. code:: 

   pydefect cr -d .

.. code:: 

   pydefect gkfo -u ../../../unitcell/unitcell.json -iefnv ../correction.json -icr ../calc_results.json -fcr calc_results.json -cd 1

通过该命令可以获取\ ``gkfo_correction.pdf``\ 和\ ``gkfo_correction.json``\ 文件，校正能量如下：

.. code:: 

   +--------------------+------------+
   | charge             |  0         |
   | additional charge  |  1         |
   | pc 1st term        |  0         |
   | pc 2nd term        |  0.731247  |
   | alignment 1st term | -0.0338952 |
   | alignment 2nd term | -0.113709  |
   | alignment 3rd term | -0         |
   | correction energy  |  0.583643  |
   +--------------------+------------+

``gkfo_correction.pdf`` 显示了由添加/移除电子及其对齐项引起的电位分布。

.. figure:: https://i.loli.net/2021/06/15/B5VDHaMRsPqp2Jj.png
   :alt: 

对于吸收能量，需要知道导带最小位置，现在是 4.7777 eV。
初态和终态的总能量为-219.02114546 eV和-222.32750506 eV。 因此，吸收能为

.. code:: 

   -222.32750506+219.02114546+4.7777+0.583643 = 2.0549834 eV

检查初始状态和最终状态的特征值也是必要的。 使用 eig 子解析器（eig
sub-parser）：

.. code:: 

   pydefect -d . -pcr ../../perfect/calc_results.json

我们可以获得\ ``eigenvalues.pdf``\ ，如下：

.. figure:: https://i.loli.net/2021/06/15/P1kRgnTIzibasql.png
   :alt: 

初始的 eigenvalues.pdf 如下：

.. figure:: https://i.loli.net/2021/06/15/pR8cEAmv3SJTI14.png
   :alt: 

Note
====

本中文教程基于kumagai's
Group发布的pydefect英文向导进行的翻译校正，源文档如下：

https://kumagai-group.github.io/pydefect/change_log.html

Based on Version 0.2.6
