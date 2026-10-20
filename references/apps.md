# OCTOPUS アプリケーションリファレンス

どのアプリも、まず `module load BaseApp` を実行してから各アプリのモジュールを読む。モジュール名はバージョンアップで変わることがあるので、実機では `module avail` で確認する（マニュアル v8.0 の `VisItv/3.4.2` は誤記で、実機は `VisIt/3.4.2`）。サンプル入力の多くは `/system/apps/rhel9/cpu/<アプリ>/InteloneAPI2025.2.0/<版>/` 以下にある。

## 一覧

| アプリ | 版 | モジュール | 実行場所 |
|---|---|---|---|
| AVS/Express Developer | 8.5 | `AVSExpress/8.5` | フロントエンド（GUI、同時5ライセンス） |
| IDL | 9.0 | `IDL/9.0` | フロントエンド（GUI、同時4ライセンス） |
| Gaussian | g16 C.02 | `Gaussian/g16c02` | バッチ（年度ごとの申請が必要） |
| ABINIT-MP | 2.8 / 1.22 | `abinit-mp/2.8` | バッチ |
| ADIOS | 1.13.1 | `adios/1.13.1` | バッチ |
| CTFFIND | 4.1.14 | `CTFFIND/4.1.14` | バッチ |
| FLASHcode | 4.8 | `FLASHcode/4.8` | バッチ |
| FreeFem++ | 4.15 | `FreeFemXX/4.15` | バッチ |
| GAMESS | v2024.2.1 | `GAMESS/v2024.2.1` | バッチ |
| GENESIS | 2.1.5 | `GENESIS/2.1.5.double` / `.mixed` / `.single` | バッチ |
| GROMACS | 2025.2 | `gromacs/2025.2` | バッチ（申請不要） |
| LAMMPS | 22Jul2025 | `lammps/22Jul2025` | バッチ |
| Quantum ESPRESSO | 7.4.1 | `QuantumESPRESSO/7.4.1` / `QuantumESPRESSO/7.4.1.hdf5` | バッチ |
| OpenFOAM | v2412 | `OpenFOAM/v2412` | バッチ |
| SMASH | 3.0.2 | `smash/3.0.2` | バッチ |
| OpenMX | 3.9.9 | `OpenMX/3.9.9` | バッチ |
| Octave | 10.2.0 | `octave/10.2.0` | |
| NEC Vector Annealing | | （BasePy に numpy / pyqubo） | VA キュー |
| MotionCor3 | 1.0.1 | `MotionCor3/1.0.1` | フロントエンドのみ |
| ParaView | 6.0.0 | `ParaView/6.0.0` | フロントエンド（GUI） |
| VisIt | 3.4.2 | `VisIt/3.4.2` | フロントエンド（GUI） |
| Relion | 5.0.0 | `relion/5.0.0` | フロントエンド（GUI） |
| ResMap | 1.1.4 | `ResMap/1.1.4` | フロントエンド（GUI） |
| Gnuplot 5.4 / ImageMagick 6.9 / NcView 2.1.8 | | モジュール不要 | フロントエンド |

GUI アプリは、X 転送（`ssh -X`）か Amazon DCV を使って表示する。

## GROMACS（2ノード、ノードあたり1 MPI × 256 スレッド）

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00
#PBS -b 2
#PBS -T intmpi
#PBS -v OMP_NUM_THREADS=256

module load BaseApp
module load gromacs/2025.2
cd $PBS_O_WORKDIR
mpirun ${NQSV_MPIOPTS} -np 2 gmx_mpi_d mdrun -s topol.tpr
```

`OMP_NUM_THREADS` を指定しないと過剰なスレッドが立ち、エラーになることがある。実行ファイル名は `gmx_mpi_d`（倍精度の MPI 版）。性能を出すには、ノードあたりの MPI 数を増やして（例: 16 MPI × 16 スレッド）比較するとよい。その場合も `-np 総MPI数` と `OMP_NUM_THREADS` の積が、ノード数 × 256 を超えないようにする。

## LAMMPS（2ノード、512 MPI）

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00
#PBS -b 2
#PBS -T intmpi
#PBS -v OMP_NUM_THREADS=1

module load BaseApp
module load lammps/22Jul2025
cd $PBS_O_WORKDIR
mpirun ${NQSV_MPIOPTS} -np 512 lmp -in in.lj
```

入力は `-in` で渡す（マニュアル例の `lmp < in.lj` は、MPI実行では標準入力が全ランクに届かないことがある）。

導入済みのパッケージ: ASPHERE, ATC, AWPMD, BOCS, BODY, CG-DNA, CG-SPICA, CLASS2, COLLOID, COLVARS, COMPRESS, CORESHELL, DIFFRACTION, DIPOLE, DPD-MESO, DRUDE, EFF, ELECTRODE, EXTRA-COMPUTE/DUMP/FIX/MOLECULE/PAIR, FEP, GRANULAR, INTEL, INTERLAYER, KSPACE, LATBOLTZ, MANIFOLD, MANYBODY, MC, MDI, MGPT, MISC, MOFFF, MOLECULE, NETCDF, OPENMP, OPT, ORIENT, PERI, PHONON, POEMS, PYTHON, QEQ, QTB, REAXFF, REPLICA, RIGID, SHOCK, SMTBQ, SPH, SPIN, SRD, TALLY, UEF, VORONOI

## Quantum ESPRESSO（1ノード、256 MPI）

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00
#PBS -T intmpi
#PBS -v OMP_NUM_THREADS=1

module load BaseApp
module load QuantumESPRESSO/7.4.1
cd $PBS_O_WORKDIR
mpirun ${NQSV_MPIOPTS} -np 256 pw.x -in scf.in > scf.out
```

HDF5 形式で出力する場合は `QuantumESPRESSO/7.4.1.hdf5` を使う。こちらは .dat 形式を扱えない。

## Gaussian 16

年度ごとの利用申請が必要（D3センター Web の「Gaussian16 申請について」を参照）。登録されると `gaussian` グループに入る。

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00

module load BaseApp
module load Gaussian/g16c02
newgrp gaussian
export GAUSS_SCRDIR=/octfs/work/G12345/u6a123/
cd $PBS_O_WORKDIR
$g16root/g16/g16 < input.gjf >& result
```

入力ファイルの末尾には必ず空行を入れる（ないとエラーになる）。並列数は入力ファイルの `%NProcShared=` で指定する。1ノード未満のジョブなら OCT-S にして、`cpunum_job` / `memsz_job` を `%NProcShared` / `%Mem` に合わせると安くなる。

## OpenFOAM（シリアルのチュートリアル）

```bash
#!/bin/bash
#PBS -q OCT-S
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00
#PBS -l cpunum_job=1

cd $PBS_O_WORKDIR
module load BaseApp
module load OpenFOAM/v2412
cp -r /system/apps/rhel9/cpu/OpenFOAM/InteloneAPI2025.2.0/v2412/OpenFOAM-v2412/tutorials/incompressible/icoFoam/cavity/cavity .
cd cavity
blockMesh
icoFoam
```

並列で実行する場合は、`decomposePar` のあと `#PBS -T intmpi` と `-b` を付け、`mpirun ${NQSV_MPIOPTS} -np N icoFoam -parallel` とする。

## GAMESS

```bash
#PBS -q OCT
#PBS -b 1
#PBS -T intmpi
cd ${PBS_O_WORKDIR}
module load BaseApp
module load GAMESS/v2024.2.1
export SCR=${PBS_O_WORKDIR}
export USERSCR=${PBS_O_WORKDIR}
rungms exam01.inp 00 24
```

## GENESIS（倍精度の例）

```bash
#PBS -q OCT
#PBS -l cpunum_job=8
#PBS -b 1
#PBS -T intmpi
module load BaseApp
module load GENESIS/2.1.5.double      # .mixed / .single もある
mpirun ${NQSV_MPIOPTS} -np 8 spdyn input.inp
```

## SMASH

```bash
#PBS -q OCT
#PBS -l cpunum_job=4
#PBS -b 1
#PBS -T intmpi
module load BaseApp
module load smash/3.0.2
mpirun ${NQSV_MPIOPTS} -np 4 smash < b3lyp-energy.inp
```

## ABINIT-MP

実行ファイルは4種類ある。F関数あり／なしと、SMP 対応／非対応の組み合わせで選ぶ: `abinitmp_smp-fint` / `abinitmp-fint` / `abinitmp_smp` / `abinitmp`。

```bash
#PBS -q OCT-S
#PBS -l cpunum_job=20
#PBS -b 1
#PBS -T intmpi
module load BaseCPU
module load BaseApp
module load abinit-mp/2.8
mpiexec -n 20 abinitmp < gly5-mpi.ajf > gly5.log
```

## FLASHcode / FreeFem++ / ADIOS / CTFFIND

- FLASHcode: `module load FLASHcode/4.8` のあと `mpirun -np 4 flash4`（`flash.par` が必要）
- FreeFem++: `module load FreeFemXX/4.15`。サンプルは `share/FreeFEM/4.15/examples`
- ADIOS: `module load adios/1.13.1` のあと `mpiifx x.F90 -I$CPATH -ladiosf`
- CTFFIND: `module load CTFFIND/4.1.14` のあと `ctffind < ctffind_input.txt`

## NEC Vector Annealing

```bash
#!/bin/bash
#PBS -q VA
#PBS --group=G12345
#PBS -l elapstim_req=00:05:00
#PBS -b 1
#PBS -v OMP_NUM_THREADS=8
cd ${PBS_O_WORKDIR}
module load BasePy
python va_program.py
```

## GUI 系（フロントエンドで実行）

```
module load BaseApp
module load ParaView/6.0.0 && paraview
module load VisIt/3.4.2    && visit
module load relion/5.0.0   && relion
module load ResMap/1.1.4   && ResMap        # フォント関連の警告が出るが動作に問題はない
module load AVSExpress/8.5 && express
module load IDL/9.0        && idlde
module load MotionCor3/1.0.1 && MotionCor3 -InMrc in.mrcs -OutMrc out.mrc
```

ライセンスが足りないときは、AVS では `license limit exceeded`、IDL では `Unable to license IDL.` と表示される。
