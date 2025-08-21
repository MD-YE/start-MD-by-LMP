#VMD2lmp
===========================

LAMMPS 是最广泛使用的分子动力学模拟软件包之一，因其灵活性、易用性和开源利于二次开发而备受青睐。本手册旨在搭建一个LAMMPS经典分子动力学模拟的工作流。作为分子动力学模拟手册，总共包含三个部分：制作data文件；根据需求确定in.lammps文件；结构后处理。


****

## 目录
* [Make_data](#Make_data)
    * [如何生成data.lmp文件](#如何生成data.lmp文件)
    * [Get_TOP&PAR](#Get_TOP&PAR)
        *  [Autoff抓取力场信息](#Autoff抓取力场信息)      
    * [Get_PDB&PSF](#Get_PDB&PSF)
        *  [生成初始pdb和psf文件](#生成初始pdb和psf文件)
           *  [VMD_TOPO](#VMD_TOPO)
        *  [替换pdb中原子位置](#替换pdb中原子位置)
           *  [Packmol](#Packmol)
           *  [NAMD](#NAMD)
    * [Switch2lmpdata](#Switch2lmpdata)
* [Make_lammps](#Make_lammps)
    * lammps的in文件

* [Make_trj](#Make_trj)
***

# Make_data

## 如何生成data.lmp文件
方案一：借助Material Stuidio的msi2lmp功能实现转换，具体方法可参考网址https://molakirlee.github.io/2020/12/06/lammps_msi2lmp/。
该方法适用力场有限（具体可见msi2lmp文件中frc文件夹中力场信息，以pcff,cvff，oplsaa,compass为主）。

方案二：从底层的力场信息出发，制作LAMMPS_data文件
LAMMPS 使用的数据文件主要包含两部分核心内容：原子坐标，成键及对应势能信息，可以将data文件拆分成 [pdb、psf、top 和 par]四个部分。
| Data Components  | Matched Info                              |
|------------------|------------------------------------------|
| sys.pdb          | atomic coordinate                         |
| sys.psf          | charge & mass & top                       |
| TOP              | topology                                  |
| PAR              | bond & angle & dihedral & non-bond parameters |

参考网址 http://bbs.keinsci.com/thread-4753-1-1.html （VMD_TOPO）
Gromacs软件可以通过sobtop实现data文件的制作。

## Get_TOP&PAR
TOP和PAR的信息是根据所使用的力场得到的。力场信息主要包含了成键（键，键角，二面角）方式（top）和参数（par）。首先建议在做了力场参数和DFT精度校对的文章中提取其使用的参数信息，按TOP和PAR文件格式填写。此处需要注意`文章参数中单位是否与格式中单位一致`。
[TOP format]

    MASS      4     HT    1.0080 H
    MASS     75     OT   15.9994 O

    RESI TIP3         0.000 ! tip3p water model, generate using noangle nodihedral
    GROUP   
    ATOM OH2  OT     -0.834
    ATOM H1   HT      0.417
    ATOM H2   HT      0.417
    BOND OH2 H1 OH2 H2    ! the last bond is needed for shake
    ANGLE H1 OH2 H2            
    PATCHING FIRS NONE LAST NONE 

[PAR format]

    BONDS
    OT    HT         450.000      0.9572 ! ALLOW   WAT  
    
    ANGLES
    HI   OI   HI    79.0263   113.4000

    _DIHEDRALS_
    _IMPROPERS_
    
    NONBONDED nbxmod  5 atom cdiel shift vatom vdistance vswitch -cutnb 14.0 ctofnb 12.0 ctonnb 10.0 eps 1.0 e14fac 0.833
    OT     0.000000  -0.152100     1.768200 ! ALLOW   WAT
    HT     0.000000  -0.046000     0.224500 ! ALLOW WAT


若无法找到对应信息，可以根据所选体系确定合适力场，常见力场有CHARMM，OPLS，Dreiding，UFF等。通过下载该力场的文件以抓取对应信息。该过程可通过网站CHARMM-GUI或者Autoff实现。介于Charmm-GUI过程较繁琐，下面贴出Autoff的网站和对应使用方法。

### Autoff抓取力场信息

    网站链接 https://autoff.readthedocs.io/en/latest/Introduction.html

需要确认的是体系力场和体系最小单元的pdb文件（网站处理大量原子困难，现在目的是确认TOP和PAR信息，pdb和psf信息可通过packmol和vmd的topo方法得到）
上传pdb文件至AutoFF中，如存在水分子确认所需要的水分子模型。

最小单元原子数较少时可选择更精确的电荷计算方法，如Qeq。此时可以选择计算软件，选择CHARMM方式，下载得到prm和rtf（对应top）文件。

>注考虑到为软件判定的准确性，对于聚合物有机体系而言，直接生成的lammps文件不一定准确以及不利于计算后处理，本方案通过将data分类成pdb,psf,top,prm文件以便于灵活地调整。

将得到的prm和rtf文件进行原子名称替换以及判断prm参数和rtf成键和电荷判定的准确性。


## Get_PDB&PSF
### 得到原始pdb和psf文件
通过VMD的TOPO功能，利用TOP功能把最小单元的pdb文件合并成整个体系的pdb文件。
#### VMD_TOPO
将已有最小单元的pdb文件和生成的TOP和PAR文件信息对应后。可进行pdb文件的复制以扩展体系，调整segment的数量以实现不同原子数大小体系的构建，通过在VMD-Extensions-TK console中执行以下命令生成sys.pdb和sys.psf文件。

      topology top_naf.rtf
      segment L1 {pdb L1.pdb; auto angles dihedrals}  #自动生产键角二面角信息，与TOP和PAR信息需要对应
      coordpdb L1.pdb L1 #添加segname信息L1
      segment OH3 {pdb OH3.pdb}
      coordpdb OH3.pdb OH3
      segment WAT {pdb solvate.pdb}
      coordpdb solvate.pdb WAT
      guesscoord
      writepdb sys.pdb
      writepsf sys.psf


### 替换pdb中原子位置
以上生成的pdb文件中可能存在原子重叠等问题导致无法直接跑MD发生报错，为了修改pdb中的原子位置，可以通过Packmol获得合理的体系。
#### Packmol
##### 参考解析网址
官网网址 https://m3g.github.io/packmol/
案例网址 http://sobereva.com/473

##### inp文件格式

     tolerance 2.0 # 最近距离
     filetype pdb
     output sys.pdb
     add_box_sides 1.2 # 添加边界

     structure solvate.pdb
       number 8000 # 添加单体数量
       inside box -60 -60 -60 60 60 60 #盒子大小，应小于际要求大小保证所有原子均位于盒子内部
     end structure
     
     structure chloroform.pdb
       number 200
       inside box 0. 0. 0. 40. 40. 20.
       inside sphere 20. 20. 20. 15. #(2,2,2) nm 1.5 nm半径的圆
     end structure
   
     structure IrO2.pdb
       number 1 # 添加单体数量
       inside box -60 -60 -60 60 60 60
       center
       fixed 0. 0. 0. 0. 0. 0.
     end structure

将得到的pdb文件用VMD以对应生成psf文件打开后，再另存以保证对应原子信息，或者通过python代码替换原sys.pdb的原子坐标。

#####  NAMD
虽然NAMD和lammps都是分子动力学模拟运行软件，LAMMPS的运行结果和可实现方案比NAMD更为广泛。NAMD较不容易发生报错且可以主动修改盒子大小，因此可以进行结构的初步系综处理。 NAMD的运行方式是通过pdb,psf和par文件进行运行。具体NAMD软件运行in文件可参考[in.NAMD]文件。另外对于特定力场（pcff之类的),Material Studio软件通过forcite分子动力学跑到体系平衡，根据内置力场文件进行msi2lmp也可以生成MD的data文件。

   NAMD参考网址 https://www.cnblogs.com/jszd/p/11178789.html

## Switch2lmpdata
将得到的pdb,psf,par,top文件通过charmm2lammps.pl脚本转换成lammps的data文件。如果文件命名为sys.pdb,sys.psf,top_naf.rtf,par_naf.prm执行命令为./charmm2lammps.pl naf sys。





# Make_lammps
在in_MD文件夹中，持续更新用经典分子动力学力场，反应力场以及机器学习力场的使用实例。进行MD计算软件包含LAMMPS(LMP),OPENMM,ase,NAMD。

电场原理

MtD
VASP
LAMMPS
CP2K

PIMD
i-pi


# Make_trj
分子动力学（MD）模拟生成的数据量巨大，通常需要后处理以提取物理化学性质或进行统计分析。数据后处理方法主要可分为三类：

模拟软件内置分析工具：多数 MD 软件（如 GROMACS、LAMMPS、AMBER 等）提供丰富的内置计算模块，通过在in文件中补充compute具体信息，可直接计算能量、温度、压力、径向分布函数（RDF）、均方位移（MSD）等基本物理量。这类方法的优点是计算高效、与模拟高度集成，适合快速得到系统整体性质。
专用分子动力学后处理软件：如 VMD, OVITO, PyMOL 等，可实现轨迹可视化和高级分析功能，例如构象聚类、氢键统计、分子间作用力分析等。这类工具通常具备图形界面，便于用户直观观察分子结构演化和动力学行为，适合对复杂体系进行定性或半定量分析。
编程库与自定义计算：近年来 Python 生态下的库（如 MDAnalysis、MDTraj、ParmEd 等）提供灵活的轨迹处理和数据分析接口，可自定义计算各种结构和动力学性质，如时间相关函数、自由能剖面、扩散系数及离子迁移率等。相比前两类方法，这类方式适合高通量、可重复的定量分析，并能方便地与机器学习或数据可视化工具结合，实现更复杂的数据处理流程。

## 模拟软件内置分析工具
下方是LAMMPS软件的MSD的计算方法实例
fix 1 all nvt temp 300 300 1
compute msd all msd com yes
fix 2 all vector 10 c_msd[4]
fix 3 all ave/time 10 1000 10000 c_msd[*] file MSD.out mode vector
thermo_style 		custom step temp c_msd[4]

## 专用分子动力学后处理软件
### VMD
VMD（Visual Molecular Dynamics）是常用的分子可视化与轨迹分析软件，支持多种文件格式（如 DCD、XTC、XYZ 等）并可进行复杂的动力学分析。其后处理功能主要包括轨迹可视化、构象分析、氢键统计、径向分布函数（RDF）、均方位移（MSD）等。
基本设置: 载入文件后在graphics任务栏中调整视图内容（VDW+DynamicBonds;对特定原子修改color颜色)，在File任务栏中save visualization保存修改格式；Mouse任务栏通过Label读取原子，键长，键角信息；Extensions任务栏中进行氢键，MSD后处理以及生成石墨烯，金属等体系，可通过添加VMD插件进行工作内容的补充（插件比如有density profile）
在Extensions任务栏中的TCL Console 提供强大的脚本接口，用于运行tcl脚本以实现自定义分析和批量处理。具体tcl脚本命令放在相应文件夹中。
氢键
VND脚本计算 https://www.cnblogs.com/sysu/p/17006091.html

VMD自带脚本 https://github.com/myonkunas/tcl_scripts


## 编程库

### MDAnalysis
https://userguide.mdanalysis.org/stable/index.html
MDAnalysis 是一个 Python 库，用于读取和分析分子动力学（MD）轨迹及拓扑文件，支持多种格式如 PDB、XYZ、DCD、LAMMPSDUMP 等。它可以按原子类型、分子或残基选择原子，计算距离、角度、二面角、配位数、RDF 等几何特征，并能分析均方位移（MSD）、速度自相关、氢键数量及寿命等动力学性质。MDAnalysis 还支持质心、旋转和平移分析、多帧统计、周期性边界处理，并能将结果输出为多种格式以便可视化和后续处理，非常适合大规模轨迹分析和定量研究。


### liquidlib
[https://github.com/dadaoqiuzhi/RMD_Digging.git](https://data.mendeley.com/datasets/tyggwp7656/1)
可得结果 中子散射长度、径向分布函数、结构因子、键定向序参数、平均平方位移、非高斯参数、四点关联函数、速度自相关函数、自 van Hove 关联函数、集体 van Hove 关联函数、自中间散射函数、集体中间散射函数。
RDAnalyzer https://github.com/RDAnalyzer/release/releases


### zeo++ 
http://zeoplusplus.org/
可得结果 孔径大小，表面积，孔径分布,可计算水团簇数量及大小 
软件安装 https://github.com/lsmo-epfl/zeopp-lsmo
软件使用指南 https://mp.weixin.qq.com/s/8BrJa62YBIkJim36l5owgA
去除水分子，探测水分子通道大小
![image](https://github.com/user-attachments/assets/bc3b496c-560e-40b3-8f74-f5eb53142ec3)
文献 J.Phys. Chem. B 2018, 122, 22, 6107–6119

### reax_tools
https://github.com/tgraphite/reax_tools
画出反应网络

### MDPlot
https://github.com/MDplot/MDplot
Gromacs蛋白方向，包含聚类，氢键分析，蛋白RMSE分析等功能。

## python案例
### RDF计算
RDF_ensemble.py：集成分析多条轨迹的RDF结果，统计其原子配位数以及g(r)绘图。
配位数直接计算方法参考
def count_cn(atoms1, atoms2, cutoff_hi, cutoff_lo=None, cell=None):
    """count the coordination number(CN) for atoms1 (center atoms), where atoms2 are coordinate atom. This function will calculate CN within range cutoff_lo < d < cutoff_lo, where d is the distance between atoms1 and atoms2. Minimum image convention is applied if cell is not None """
    Args:
        atoms1 (numpy.ndarray): Array with shape (N, 3), where N is the number of center atoms. 'atoms1' are the position of center atoms. 
        atoms2 (numpy.ndarray): Array with shape (M, 3), where M is the number of coordination atoms. 'atoms2' are the positions of coordination atoms.
        cutoff_hi (float): Max cutoff for calculating coordination number. 
        cutoff_lo (float or None, optional): Min cutoff for calculating coordination number. This function will calculate CN within range cutoff_lo < d < cutoff_lo, where d is the distance between atoms1 and atoms2. Defaults to None.
        cell (numpy.ndarray, optional): Array with shape (6,), Array([a, b, c, alpha, beta, gamma]). Simulation cell parameters. If it's not None, the CN calculation will use minimum image convention. Defaults to None.
    Returns:
        results: Array with shape (N,), CN of each atoms atoms1
    """
    pairs, _ = capped_distance(reference=atoms1,
                               configuration=atoms2,
                               max_cutoff=cutoff_hi,
                               min_cutoff=cutoff_lo,
                               box=cell)
    _minlength = atoms1.shape[0]
    results = np.bincount(pairs[:, 0], minlength=_minlength)
    return results
    
### 均方位移MSD
具体思路参考[https://zhuanlan.zhihu.com/p/31392262515#:~:text=%E2%80%9C%20%E7%BB%9F%E8%AE%A1%E5%8A%9B%E5%AD%A6%E4%B8%AD%EF%BC%8C%20%E5%9D%87%E6%96%B9%E4%BD%8D%E7%A7%BB%20%28mean%20square%20displacement%2C%20MSD%29%20%E6%98%AF%E7%B2%92%E5%AD%90%E4%BD%8D%E7%BD%AE%E7%9B%B8%E5%AF%B9%E4%BA%8E%E5%8F%82%E8%80%83%E4%BD%8D%E7%BD%AE%E9%9A%8F%E6%97%B6%E9%97%B4%E5%8F%98%E5%8C%96%E7%9A%84%E5%81%8F%E5%B7%AE%E7%9A%84%E5%BA%A6%E9%87%8F%E3%80%82,%E5%B9%B6%E8%A1%8C%E7%89%88%E6%9C%AC%E3%80%81%20%E5%BF%AB%E9%80%9F%E5%82%85%E9%87%8C%E5%8F%B6%E5%8F%98%E6%8D%A2%20%E7%89%88%E6%9C%AC%E3%80%82%20MSD%E6%98%AF%20%E5%88%86%E5%AD%90%E5%8A%A8%E5%8A%9B%E5%AD%A6%E6%A8%A1%E6%8B%9F%20%E4%B8%AD%E7%9A%84%E5%85%B3%E9%94%AE%E5%B7%A5%E5%85%B7%EF%BC%8C%E7%94%A8%E4%BA%8E%E7%A0%94%E7%A9%B6%E7%B2%92%E5%AD%90%E6%89%A9%E6%95%A3%E3%80%81%20%E7%9B%B8%E5%8F%98%20%E3%80%81%E7%B3%BB%E7%BB%9F%E5%B9%B3%E8%A1%A1%E5%8F%8A%E6%9D%90%E6%96%99%E6%80%A7%E8%83%BD%E7%AD%89%E3%80%82](https://zhuanlan.zhihu.com/p/31392262515#:~:text=%E2%80%9C%20%E7%BB%9F%E8%AE%A1%E5%8A%9B%E5%AD%A6%E4%B8%AD%EF%BC%8C%20%E5%9D%87%E6%96%B9%E4%BD%8D%E7%A7%BB%20%28mean%20square%20displacement%2C%20MSD%29%20%E6%98%AF%E7%B2%92%E5%AD%90%E4%BD%8D%E7%BD%AE%E7%9B%B8%E5%AF%B9%E4%BA%8E%E5%8F%82%E8%80%83%E4%BD%8D%E7%BD%AE%E9%9A%8F%E6%97%B6%E9%97%B4%E5%8F%98%E5%8C%96%E7%9A%84%E5%81%8F%E5%B7%AE%E7%9A%84%E5%BA%A6%E9%87%8F%E3%80%82,%E5%B9%B6%E8%A1%8C%E7%89%88%E6%9C%AC%E3%80%81%20%E5%BF%AB%E9%80%9F%E5%82%85%E9%87%8C%E5%8F%B6%E5%8F%98%E6%8D%A2%20%E7%89%88%E6%9C%AC%E3%80%82%20MSD%E6%98%AF%20%E5%88%86%E5%AD%90%E5%8A%A8%E5%8A%9B%E5%AD%A6%E6%A8%A1%E6%8B%9F%20%E4%B8%AD%E7%9A%84%E5%85%B3%E9%94%AE%E5%B7%A5%E5%85%B7%EF%BC%8C%E7%94%A8%E4%BA%8E%E7%A0%94%E7%A9%B6%E7%B2%92%E5%AD%90%E6%89%A9%E6%95%A3%E3%80%81%20%E7%9B%B8%E5%8F%98%20%E3%80%81%E7%B3%BB%E7%BB%9F%E5%B9%B3%E8%A1%A1%E5%8F%8A%E6%9D%90%E6%96%99%E6%80%A7%E8%83%BD%E7%AD%89%E3%80%82)
MSD_C.py：显式计算粒子位移，MSD_Cz.py对应z轴方向的MSD。
MSD_E.py：MDAnalysis内置的EinsteinMSD，利用粒子位移平方平均随时间增长的线性规律计算，MSD_Cz.py对应z轴方向的MSD。

### 氢键分析
hba = HBA(
    universe=u,
    donors_sel="type 2",        # 供体 O
    hydrogens_sel="type 1",     # 氢
    acceptors_sel="type 2",     # 受体 O
    d_h_cutoff=1.2,             # D-H 最大距离
    d_a_cutoff=3.2,             # D-A 最大距离
    d_h_a_angle_cutoff=120      # D-H-A 角度阈值
)

### 反应网络

### 电场位移距离drift length计算


物种覆盖度
数据后处理脚本   https://github.com/dadaoqiuzhi/RMD_Digging.git

linux命令替换原子名词
   sed -e 's/^1 /HT  /g' -e 's/^2 /CA /g' -e 's/^3 /OT  /g' -e 's/^4 /CF  /g' -e 's/^5 /FC  /g' -e 's/^6 /OC  /g'  -e 's/^7 /OS  /g' -e 's/^8 /SO  /g' -e 's/^9 /OI  /g' -e 's/^10 /HI  /g'  dump.xyz > nafion.xyz



