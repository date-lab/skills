# OCTOPUS ジョブ実行リファレンス

目次
1. ジョブクラス表
2. qsub オプション
3. テンプレート（シリアル / OpenMP / Intel MPI / ハイブリッド / Open MPI / ppn制御 / 複数プログラム / Python / Apptainer / VTune / インタラクティブ）
4. ジョブ操作コマンドと状態コード
5. 待ち時間を短くする
6. ポイント計算
7. 他システムからの対応表

---

## 1. ジョブクラス表（D3センターWeb、2026年版）

| 利用 | クラス | 経過時間 | 最大コア | 最大メモリ | 同時利用ノード | 備考 |
|---|---|---|---|---|---|---|
| 共有 | OCT | 120h | 32,768（256/ノード） | 92.75TiB（742GiB/ノード） | 128 | ノード内は占有 |
| 共有 | OCT-H | 120h | 同上 | 同上 | 128 | 高優先度。消費係数が高い |
| 共有 | OCT-S | 120h | 128 | 371GiB | 1 | ノード共有。≤128コアかつ≤371GBで係数1/2、≤64コアかつ≤185GBで係数1/4 |
| 共有 | DBG | 10分 | 512（256/ノード） | 1,484GiB | 2 | デバッグ用 |
| 共有 | INT | 10分 | 512 | 1,484GiB | 2 | qlogin 用 |
| 共有 | VA | 120h | 256 | 742GiB | 1 | 擬似量子アニーリング |
| 共有 | DPS | 120h | 32,768 | 92.5TiB（740GiB/ノード） | 128 | DPS4H 来歴採取 |
| 占有 | myOCT | 無制限 | 256×占有ノード | 742GiB×占有ノード | 占有数 | 別途申請 |

- **2026年5月から同時利用可能ノード数は62ノードまで**。63ノード以上は問い合わせが必要。
- 経過時間の既定値は1時間。
- 高優先度は「混雑時に次に実行されやすくなる」だけで、実行予定が確定したジョブを押しのけはしない。
- 上限超過時の挙動: 実行キュー（DBG, VA, DPS）は qsub 時にエラー。転送キュー（OCT, OCT-H, OCT-S）は受け付けた後、`.e` ファイルに `Request could not be queued ... Exceeded resource limit.` が出て削除される。

## 2. qsub オプション

`#PBS` 行は、最初のシェルコマンドより前に書く。

| オプション | 意味 |
|---|---|
| `-q <クラス>` | **必須**。ジョブクラス |
| `--group=<グループ>` | **必須**。課金グループ（G12345 / hp****** / jh******） |
| `-l elapstim_req=hh:mm:ss` | 経過時間上限。既定は1時間 |
| `-l cpunum_job=<N>` | **1ノードあたり**のコア数。OCT/OCT-H/DBGで全コアを使うなら省略する（ノード占有） |
| `-l memsz_job=<size>` | 1ノードあたりのメモリ（OCT-S専用。例 `185GB`） |
| `-l coresz_prc=<size>` | プロセスあたりのコアファイル上限（既定0）。単位 b/kb/mb/gb/tb |
| `-b <N>` | ノード数（マルチノードMPIで必須） |
| `-T intmpi` / `-T openmpi` | MPI種別（MPIでは必須） |
| `-v VAR=val[,VAR2=val2]` | 全ノードに環境変数を渡す（最も確実。Intel MPI の mpirun は `export` した変数も他ノードのランクに転送するが、Open MPI は `-x VAR` が必要） |
| `-N <name>` | ジョブ名（既定はスクリプト名） |
| `-o <file>` / `-e <file>` | 標準出力/エラーの出力先（既定 `name.o<ID>` / `name.e<ID>`） |
| `-j o` / `-j e` | 標準出力とエラーをまとめる |
| `-m b|e|a|n` / `-M addr` | メール通知（開始/終了/異常終了/なし。`-m be` のように組み合わせ可） |
| `-r y|n` | ノード障害時の自動再投入の可否 |

`-l` は1行に1項目ずつ書くと読みやすい。カンマでつなげることもできる（その場合スペースは入れない）: `#PBS -l elapstim_req=2:00:00,cpunum_job=128`

ジョブ内で使える変数: `PBS_O_WORKDIR`（qsubしたディレクトリ）、`NQSV_MPIOPTS`（mpirun用のホスト指定）、`PBS_NODEFILE`（ホストファイル）。

## 3. テンプレート

`G12345` は自分のグループ名に置き換える。

### シリアル（小さいジョブは OCT-S で安く）

```bash
#!/bin/bash
#PBS -q OCT-S
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00
#PBS -l cpunum_job=1
#PBS -l memsz_job=16GB

module load BaseCPU
cd $PBS_O_WORKDIR
./a.out > out.txt
```

D3センターのチュートリアルでは `cpunum_job=128` としているが、1コアしか使わないなら必要量を書く方がシェア係数1/4が効いて安い。メモリが必要量に足りないとジョブが止まるので、初回は大きめに取る。区分（≤64コアかつ≤185GBで1/4）を跨がなければ料金は同じなので、必要量の1.2〜1.5倍程度を目安にする（区分上限いっぱいまでは取らない）。

### OpenMP（1ノード256スレッド）

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=00:30:00
#PBS -v OMP_NUM_THREADS=256

module load BaseCPU
cd $PBS_O_WORKDIR
./a.out
```

コンパイル: `ifx -qopenmp -o a.out sample.f90`（GCCなら `-fopenmp`）。

### Intel MPI（4ノード × 256プロセス = 1,024プロセス）

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=00:30:00
#PBS -b 4
#PBS -T intmpi

module load BaseCPU
cd $PBS_O_WORKDIR
mpirun ${NQSV_MPIOPTS} -np 1024 ./a.out
```

コンパイル: `mpiifx` / `mpiicx` / `mpiicpx`。

### MPI + OpenMP ハイブリッド（4ノード × 1プロセス × 256スレッド）

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00
#PBS -b 4
#PBS -T intmpi
#PBS -v OMP_NUM_THREADS=256

module load BaseCPU
cd $PBS_O_WORKDIR
mpirun ${NQSV_MPIOPTS} -np 4 ./a.out
```

コンパイル: `mpiifx -qopenmp ...`。ノード当たりプロセス数 × `OMP_NUM_THREADS` を256に合わせる（`cpunum_job` を書く場合はその値とも一致させる。ずれると性能が落ちる）。

### Open MPI（GCC）

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00
#PBS -T openmpi
#PBS -b 4

module load BaseGCC
cd $PBS_O_WORKDIR
mpirun ${NQSV_MPIOPTS} -np 1024 -npernode 256 ./a.out
```

コンパイル: `mpicc` / `mpic++` / `mpif90`。

### ノード当たりのプロセス数を制御する（Intel MPI）

`NQSV_MPIOPTS` には `-machinefile` が入っているため、`-ppn`/`-rr`/`-perhost` は無視される。実機では `-np 8 -ppn 4` の8ランクがすべて1ノードに載った。ppn を効かせたいときは `PBS_NODEFILE` を使う（実機で4+4に分かれることを確認済み）。

```bash
#PBS -b 2
#PBS -T intmpi
...
mpirun -hostfile ${PBS_NODEFILE} -np 128 -ppn 64 ./a.out
```

単にコア数を絞るだけなら `-l cpunum_job=` でもよい。

### 1ジョブで複数プログラム（ノードを使い切る）

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00

module load BaseCPU
cd $PBS_O_WORKDIR
export OMP_NUM_THREADS=64
./a.out > a.log &
./b.out > b.log &
./c.out > c.log &
./d.out > d.log &
wait
```

1ノード内なら `export` で足りる。`wait` を忘れると、スクリプトが終わった時点でバックグラウンドのプロセスが殺される。

MPIプログラムを2本同時に走らせる（MPMD）:

```bash
#PBS -b 3
#PBS -T intmpi
mpirun ${NQSV_MPIOPTS} -np 256 ./a-mpi.out : -np 512 ./b-mpi.out
```

### Python（venv）

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00
#PBS -v OMP_NUM_THREADS=256

module load BasePy
source /octfs/work/G12345/u6a123/myenv/bin/activate
cd $PBS_O_WORKDIR
python train.py
```

計算ノードは外部に接続できないので、`pip install` はフロントエンドで済ませておく。Miniforge を使う場合は `source ~/.bashrc` の後に `mamba activate <env>`。

### Apptainer

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=00:30:00
#PBS -v OMP_NUM_THREADS=256

cd $PBS_O_WORKDIR
apptainer exec --bind `pwd` image.sif ./a.out
```

Homeと `/tmp` は自動でマウントされる。Work領域は `--bind` で明示する。

### VTune / APS プロファイル

```bash
#PBS -T intmpi
module load BaseCPU
cd $PBS_O_WORKDIR
aps --result-dir=./result --collection-mode=all mpirun ${NQSV_MPIOPTS} -ppn 2 -np 2 ./a.out
```

`aps_report_YYYYMMDD_HHMISS.html` が出力される。`--collection-mode` には hwc/omp/mpi/all を指定する。

### インタラクティブ

```bash
qlogin -q INT -l elapstim_req=00:10:00 --group=G12345
# Request 12345.oct submitted to queue: INT. → 空いていれば20〜30秒で [user@oct-xxx ~]$
logout   # 終了
```

最大10分まで。`-o/-e`、ホールド、リラン、ステージングの指定は使えない。待っている間はプロンプトが返らないので、別の端末から `qstat` で状態を見る。

## 4. ジョブ操作コマンドと状態コード

| コマンド | 用途 |
|---|---|
| `qsub script.sh` | 投入 |
| `qstat` / `qstat -l --adjust-column` / `qstat -f <ID>` | 自分のジョブ / 省略なし / 詳細 |
| `qstatgroup` / `qstatall` | グループ / 全体（他人のジョブはマスクされる。15分ごとの更新） |
| `sstat` / `sstatgroup` / `sstatall` | スケジュール（開始予定時刻） |
| `qcat [-o|-e] [-f] [-n N] [-b] <ID>` | 実行中ジョブのスクリプト/出力を見る |
| `qdel [-g 秒] <ID>` | 削除。RUN中ならSIGTERMを送り、既定5秒後にSIGKILL |
| `qhold <ID>` / `qrls <ID>` | 保留 / 解除 |
| `acstat [-A]` / `acstatgroup [-A]` | 過去ジョブ（24h / 年度） |
| `qstat -Q` | キュー状態 |

STT列: QUE待ち、ASG割当済（開始予定時刻あり）、RUN実行中、PRR/POR前処理/後処理、EXT結果転送、STGステージング、HLD保留、HOL保留要求中、WAT時刻待ち、SUS中断、TRS/ARI転送、MIG移動、RST/CHKチェックポイント。S列が `Y` ならストール。

`qstat -Q` では、ROUTING QUEUE（OCT / OCT-H / OCT-S など、ユーザーが投入するクラス）と EXECUTION QUEUE（O1 / O8 / O64 / O128 と各 H 版、O1S / O1SS、DBG、VA、DPS）の両方が表示される。O1 の QUE 数を見ると、1ノードジョブの混み具合がおおよそ分かる。

ジョブ終了後は `qstat` から消える。結果は `<名前>.o<ID>` / `.e<ID>` と、リダイレクトしたファイルで確認する。

## 5. 待ち時間を短くする

1. `elapstim_req` を実際の実行時間＋少しの余裕にする。スケジューラは要求時間でノードを予約するので、長く書くほど空きに入りにくい。
2. 長い計算はリスタート可能にして、数時間ずつのジョブに分ける。
3. 動作確認は `DBG`（10分以内、回転が速い）か `qlogin -q INT` で行う。
4. ノード稼働状況（Webポータル、5分ごとの更新）で空き時間帯を見て、それに収まる時間を指定する。
5. 最後の手段として `OCT-H`（割高）を使う。

## 6. ポイント計算

```
消費ポイント = ノード数 × 時間[h] × 消費係数 × 季節係数 × 燃料係数
```

| キュー | 消費係数 |
|---|---|
| OCT-H（高優先度） | 0.6219 |
| OCT（通常） | 0.5287 |
| OCT-S シェア1/2 | 0.3732 |
| OCT-S シェア1/4 | 0.2488 |

- 季節係数: 2025年度は通年1（年度末に翌年度分が公表される）。
- 燃料係数: 2026年度は9月まで0.85、10月から1.15（随時見直し）。
- ノード時間 = ノード数 × 計算時間。システム障害で中断したジョブは課金されない。
- 残量は `usage_view` で確認する。Available Node-hours は「残りポイントをすべてそのノード群で使った場合」の換算値。

例: OCTで4ノード×10時間、燃料係数1.15 → 4×10×0.5287×1.15 = 24.32ポイント。

## 7. 他システムからの対応表

| やりたいこと | Slurm | OCTOPUS (NQSV) |
|---|---|---|
| キュー | `#SBATCH -p` | `#PBS -q OCT` |
| 課金先 | `#SBATCH -A` | `#PBS --group=G12345` |
| 時間 | `-t 1:00:00` | `-l elapstim_req=1:00:00` |
| ノード数 | `-N 4` | `-b 4` |
| ノード当たりコア | `--cpus-per-task` 等 | OCTでは省略（ノード占有）。OCT-Sは `-l cpunum_job=N` |
| MPI起動 | `srun ./a.out` | `#PBS -T intmpi` + `mpirun ${NQSV_MPIOPTS} -np N ./a.out` |
| 環境変数 | `--export` | `-v VAR=val` |
| 作業ディレクトリ | 投入ディレクトリ | `cd $PBS_O_WORKDIR` が必要 |
| 投入/状態/削除 | sbatch/squeue/scancel | qsub/qstat・sstat/qdel |
| インタラクティブ | `srun --pty bash` | `qlogin -q INT ...` |

SQUID（同じD3センターの別システム）もNQSVだが、キュー名（SQUID, SQUID-H など）、ホスト、ノード構成が違う。SQUID向けのスクリプトを流用するときは `-q` とコア数、モジュールを必ず書き換える。
