#VMD2lmp
===========================

该文件为分子动力学模拟手册，总共包含三个部分：制作data文件；根据需求确定in.lammps文件；结构后处理。


****

## 目录
* [Make_data](#Make_data)
    * [如何生成data.lmp文件](#如何生成data.lmp文件)
    * [Get_TOP&PAR](#Get_TOP&PAR)
        *  [Autoff抓取力场信息](#Autoff抓取力场信息)      
    * [Get_PDB&PSF](#Get_PDB&PSF)
        *  [生成初始pdb和psf文件](#生成初始pdb和psf文件)
           *  [VMD_TOPO](#VMD_TOPO)
        *  修改pdb的原子坐标
           *  [Packmol](#Packmol)
           *  [NAMD](#NAMD)
    * Switch2lmpdata
        *  lmpdata info   
        *  xyz2lmp
        *  Charmm2lmp
    * 删除线

***

# Make_data
## 如何生成data.lmp文件
LAMMPS is one of the most widely used molecular dynamics simulation packages due to its flexibility, ease of use, and open-source nature. The data file used in LAMMPS contains two main components: atomic coordinates and potential (force field) information, further concluded by [pdb, psf, top and par]. The first par of manual provides a straightforward introduction to the simplest method for creating a data file from scratch.

| Data Components  | Matched Info                              |
|------------------|------------------------------------------|
| sys.pdb          | atomic coordinate                         |
| sys.psf          | charge & mass & top                       |
| TOP              | topology                                  |
| PAR              | bond & angle & dihedral & non-bond parameters |

用一张图说明流程为

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
将已有最小单元的pdb文件和生成的TOP和PAR文件信息对应后。在VMD-Extensions-TK console中执行以下命令。

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

得到初始pdb和psf文件信息

### 修改pdb的原子坐标
通过Material Studio/gauss等软件得到体系的最小单元后，需要批量复制原子已得到所需大小的体系。批量复制可以通过Material Studio内置内容得到，后续可以通过xyz2lmp以得到data文件。本手册主要介绍通过Packmol获得合理的体系。同时也简要说明了用NAMD完成这个过程的方式。
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
虽然NAMD和lammps都是分子动力学模拟运行软件，LAMMPS的运行结果和可实现方案比NAMD更为广泛。NAMD主要与MS起类似作用，进行结构的初步系综处理。 NAMD的运行方式是通过pdb,psf和par文件进行运行。

NAMD运行in文件例子
    
    structure           sys.psf
    coordinates         sys.pdb
    paraTypeCharmm	    on
    parameters          par_naf.prm
    if {0} { #固定原子位置，对应pdf文件倒数第二列1.0为固定
        fixedAtoms on
        fixedAtomsFile sys.pdb
        fixedAtomsCol O
   }

    set La              80
    set temperature     300
    temperature         $temperature

    if {1} {
      cellBasisVector1    $La     0.     0.
      cellBasisVector2     0.    $La     0.
      cellBasisVector3     0.     0.    $La
      cellOrigin           0.     0.     0.
      PME                 yes
      PMEGridSizeX        100
      PMEGridSizeY        100
      PMEGridSizeZ        100
    }
    wrapWater           on
    wrapAll             on

    exclude             scaled1-4
    cutoff              10.
    switching           on
    switchdist          8.
    pairlistdist        12.

    timestep            0.5

    if {1} { #NVT操作
      langevin            on   
      langevinDamping     5    
      langevinTemp        $temperature
      langevinHydrogen    on   
    }

    if {1} { #NPT操作
        useGroupPressure      yes
        useFlexibleCell       no

        langevinPiston        on
        langevinPistonTarget  100 ;#  in bar -> 1 atm
        langevinPistonPeriod  100.0
        langevinPistonDecay   50.0
        langevinPistonTemp    $temperature
    }
    if {0} { #添加电场
        eFieldOn          yes
        eField            0.0 0.0 1
    }

    outputName        md_o
    dcdfreq           1000
    outputEnergies    1000
    outputTiming      1000

    if {1} {
      minimize        100
      reinitvels      $temperature
    }

    run               1000000

最后，将得到的pdb,psf,par,top文件通过charmm2lammps.pl脚本转换成lammps的data文件。如果文件命名为sys.pdb,sys.psf,top_naf.rtf,par_naf.prm执行命令为./charmm2lammps.pl naf sys。

# Make_lammps
