---
name: octopus
description: 大阪大学D3センターのスーパーコンピュータOCTOPUS（Intel Xeon 6980P×2/ノード、NEC NQSV、2段階認証SSH）を使うためのスキル。ジョブスクリプト（#PBS）の作成・修正、qsub/qstat/sstat/qlogin、ジョブクラス（OCT/OCT-H/OCT-S/DBG/INT/VA）の選択、module（BaseCPU/BaseGCC/BasePy等）、Intel oneAPI/Intel MPI/OpenMPでのコンパイル、ポイント消費の見積り、/octfs/home・/octfs/work、ファイル転送、Python/Miniforge/Apptainer、GROMACS/LAMMPS/Quantum ESPRESSO/Gaussian等のアプリ実行、さらにSSH接続を通じたジョブ投入・状態確認に使う。OCTOPUS、阪大スパコン、D3センター、octopus.hpc.osaka-u.ac.jp、NQSV、qsubで「#PBS --group」、elapstim_req、cpunum_job、NQSV_MPIOPTS、usage_viewが出てきたら、ユーザーが「OCTOPUS」と明示していなくても必ずこのスキルを使うこと。SQUIDなど他システムの話でも、OCTOPUSとの比較や移行が絡むなら使う。
---

# OCTOPUS（大阪大学 D3センター）利用スキル

OCTOPUSは汎用CPUノード140台からなる共有スパコン。ジョブ管理はNEC NQSV（PBS風の `#PBS` 指示文だが、Slurm や OpenPBS とは書き方が違う）。一般的なHPCの知識で書くと高確率で間違えるので、ここの事実を優先すること。

情報源：NEC「OCTOPUS 利用者向けマニュアル v8.0」（2026/04）と D3センターWeb（https://www.hpc.cmc.osaka-u.ac.jp/system/manual/octopus2-use/）。両者が食い違う場合はWebの方が新しい。運用値（ノード上限、燃料係数など）は変わるので、重要な判断ではWebの該当ページを案内するか、ログインできるなら実機で確かめる。

## まず押さえる事実

| 項目 | 値 |
|---|---|
| ログイン先 | `octopus.hpc.osaka-u.ac.jp`（HPCフロントエンド、2台: octopus1/octopus2）。占有型は `voctopusXX.hpc.osaka-u.ac.jp`（要申請） |
| 認証 | パスワード + 2段階認証（TOTP, `Verification code:`）。公開鍵だけではログインできない |
| 計算ノード | Xeon 6980P (128コア)×2 = **256コア/ノード**、メモリ768GB（ジョブで使えるのは **742GiB/ノード**）、InfiniBand NDR200 |
| OS | 計算ノード Rocky Linux 9.4、フロントエンド RHEL 9.4、ログインシェル bash |
| Home | `/octfs/home/<利用者番号>`、100GiB/人、拡張不可 |
| Work | `/octfs/work/<グループ名>/<利用者番号>`、5TiB/グループ（追加購入可）、ファイル数上限3億/グループ |
| バックアップ | **なし**。重要データは各自で退避 |
| グループ名 | `groups` か `id` で確認（`ocean G12345` の `G12345`。HPCI/JHPCNは `hp******`/`jh******`） |
| 利用状況 | `usage_view`（ポイント・ノード時間・ディスク。ノード時間は毎朝7時更新） |

## ジョブクラス（`#PBS -q`）

| クラス | 経過時間上限 | 規模 | 用途・注意 |
|---|---|---|---|
| `OCT` | 120時間 | 256コア×最大128ノード（※2026年5月から同時利用は**62ノードまで**。63ノード以上は要連絡） | 通常。**ノード占有なので1ノード分課金される** |
| `OCT-H` | 120時間 | OCTと同じ | 高優先度。消費係数が高い |
| `OCT-S` | 120時間 | 1ノード内、≤128コア・≤371GiB | ノード共有。小さいジョブは割安（下記） |
| `DBG` | **10分** | 2ノードまで | 動作確認用。回転が速い |
| `INT` | **10分** | 2ノードまで | `qlogin` 専用（インタラクティブ） |
| `VA` | 120時間 | 1ノード | NEC Vector Annealing用 |
| `DPS` | 120時間 | OCT同様 | DPS4Hによる来歴採取用 |
| `myOCT` | 無制限 | 占有ノード数 | 占有利用（別途申請） |

`qstat` の Queue 列には、投入したクラス（ルーティングキュー）ではなく、振り分け先の実行キュー名が出る。違う名前に見えても正常。実行キューはノード数ごとに `O1` / `O8` / `O64` / `O128` があり、高優先度版には `H` が付く（`O1H` など）。OCT-S は `O1S` / `O1SS` に入る（シェア1/2と1/4に対応すると推定）。混み具合は `qstat -Q` の QUE 列（待ち）と RUN 列（実行中）で分かる（2026年10月の実機では O1 に約940件が待っていた）。

**クラス選びの考え方**：1ノードを使い切らない計算（シリアル、数十コアのOpenMPなど）を `OCT` に投げるとノード丸ごと課金される。128コア以下・371GiB以下なら `OCT-S` で消費係数が1/2相当、64コア以下・185GiB以下なら1/4相当になる。ただし他ジョブと同居するので性能が揺れることがある。待ち時間が問題なら、まず `elapstim_req` を実際の実行時間＋余裕程度に縮める（スケジューラは要求時間で隙間に詰めるため）。それでも駄目なら `OCT-H`。

## ジョブスクリプトの書き方

最小形（`OCT`、1ノード）:

```bash
#!/bin/bash
#PBS -q OCT
#PBS --group=G12345
#PBS -l elapstim_req=01:00:00

module load BaseCPU
cd $PBS_O_WORKDIR
./a.out > result.txt
```

外すと失敗・事故につながるルール（理由つき）:

- **`--group=` は必須**。課金先グループ。ユーザーのグループ名が分からなければ `G12345` のような仮値にせず、`groups` で確認するよう伝えるか、プレースホルダであることを明示する。
- **`-l elapstim_req=hh:mm:ss`**。省略すると1時間。超えた瞬間に強制終了されるので、見積りより少し長めに。DBG/INTは10分まで。
- **`-l cpunum_job=N` は1ノードあたりのコア数**（総数ではない）。OCT/OCT-H/DBG はノードを丸ごと占有するので、全コアを使うなら書かない（マニュアルのOCT例でも省略されている）。書くのは、OCT-S で使う量を申告するときと、ノード内のコア数をわざと絞るときだけ。
- `-l memsz_job=` は OCT-S でのみ使う。超えるとジョブが止まるので、必要量ぎりぎりではなく1.2〜1.5倍程度の余裕を持たせる。料金はシェア区分（≤64コアかつ≤185GBなら1/4、≤128コアかつ≤371GBなら1/2）で決まるので、区分を跨がない範囲で増やす。ただし区分の上限いっぱいまで取るのは避ける（共有ノードの空きに入りにくくなりうるし、同居ジョブの分を無駄に押さえる）。
- `-l` は1行に1項目ずつ書く（`#PBS -l elapstim_req=...` と `#PBS -l cpunum_job=...` を分ける）。カンマでつなげることもできるが読みにくい。
- **MPIでは `-T intmpi`（Intel MPI）または `-T openmpi`（Open MPI）が必須**、複数ノードなら `-b <ノード数>`。
- **`mpirun ${NQSV_MPIOPTS} -np <総プロセス数> ./a.out`** と書く。`NQSV_MPIOPTS` に割り当てノードの `-machinefile` が入っている。このため Intel MPI の `-ppn`/`-perhost` は無視される。ノード当たりプロセス数を制御したいときは `mpirun -hostfile ${PBS_NODEFILE} -np 128 -ppn 64 ./a.out` の形にする。
- **環境変数は `#PBS -v VAR=value`** で渡すのが確実。スクリプト内で `export` した変数は、Intel MPI の `mpirun` で起動したランクには他ノードでも届いた（2026年10月に実機で確認）。ただしセンターの案内は「`export`/`setenv` はマスターノードにしか効かない」としており、Open MPI では `-x` を付けないと転送されない。迷ったら `-v` を使う。特に `OMP_NUM_THREADS` は必ず指定する（未指定だと過剰スレッドで遅くなったりエラーになる）。ハイブリッドでは `cpunum_job` と `OMP_NUM_THREADS × ノード当たりプロセス数` を揃える。
- **`cd $PBS_O_WORKDIR`** を忘れない（qsubを実行したディレクトリ）。
- 計算に使ったモジュールは**スクリプト内でも `module load`** する（ログインシェルの状態は引き継がれない）。
- **ファイル末尾に改行**を入れる。ないと最終行が実行されない（NQSVの既知の挙動）。
- 計算ノードは**外部ネットワークに出られない**。pip/mamba/apptainer pull などはフロントエンドで事前に済ませる。他クラスタからの移植ではvenv等を作り直す必要があるので、この理由も添えて手順を示す。
- 大きな出力はリダイレクトでファイルへ。標準出力は `<スクリプト名>.o<ID>`、標準エラーは `.e<ID>` に出る。
- コアダンプが要るなら `#PBS -l coresz_prc=50mb` のように指定（既定は0で出ない）。
- その他: `-N 名前`, `-o/-e ファイル`, `-j o`, `-m be -M addr`（開始/終了メール）, `-r n`（ノード障害時に自動再投入しない）。

並列形態ごとの完全なテンプレート（OpenMP、Intel MPI、MPI+OpenMP、Open MPI、1ジョブ複数プログラム、Python、Apptainer、VTune）は `references/jobs.md` にある。新しくスクリプトを作るときはそこから始めると取りこぼしがない。

## 開発環境

- **モジュールは Base 環境から読む**: `BaseCPU`（Intel oneAPI + Intel MPI + MKL/HDF5/NetCDF、推奨）、`BaseGCC`（GCC + Open MPI + GSL）、`BasePy`（Python 3.13.5 + PyTorch/TensorFlow/Keras。scikit-learn は入っていない）、`BaseR`、`BaseJDK`、`BaseJulia`、`BaseApp`（アプリ用）。アプリやライブラリのモジュールはBaseを読んだ後でないと `module avail` に出ない。Base 環境には `/2025` と `/2026` があり、既定は `/2026`（2026年10月に実機で確認。BaseCPU/2026 は oneAPI 2025.3）。マニュアルは `BaseCPU/2025` と書いているが、スクリプトではバージョンを付けずに `module load BaseCPU` と書き、既定を使う。古い環境で作ったバイナリを動かすときだけ、コンパイル時と同じ `/2025` を明示する。
- コンパイラ: Intel は `icx`/`icpx`/`ifx`、MPIは `mpiicx`/`mpiicpx`/`mpiifx`、OpenMPは `-qopenmp`、MKLは `-qmkl=sequential|parallel|cluster`。GCCは `gcc`/`g++`/`gfortran` と `-fopenmp`、MPIは `mpicc`/`mpic++`/`mpif90`。
- **フロントエンドでの重い計算は禁止**（見つかると止められる）。コンパイル・編集・軽い前処理だけにし、テスト実行は `DBG` か `qlogin -q INT` で行う。10日以上アイドルのログインは切断される。
- Python: `module load BasePy` の後、venv を **Work領域** に作って pip install（Homeは100GiBしかない）。Miniforge を使う場合は `MAMBA_EXTRACT_THREADS=8` を設定しないと多コアのフロントエンドでハングすることがある。
- Apptainer: `apptainer exec --bind $(pwd) image.sif ./a.out`。Work領域で build/sandbox を作るときは先に `newgrp <グループ名>`。キャッシュは `APPTAINER_CACHEDIR` でWorkへ。

詳細（ライブラリのリンク例、Python/Miniforge/Apptainer手順、Amazon DCV、VS Code Remote-SSH）は `references/environment.md`。

## アプリケーション

GROMACS、LAMMPS、Quantum ESPRESSO、OpenFOAM、GAMESS、GENESIS、SMASH、ABINIT-MP、Gaussian（年度申請＋`newgrp gaussian` が必要）、ParaView/VisIt/AVS/IDL（フロントエンドでGUI）、Octave、OpenMX などが入っている。いずれも `module load BaseApp` の後に各モジュールを読む。モジュール名・バージョン・ジョブスクリプト例は `references/apps.md`。

## ポイント（課金）

```
消費ポイント = ノード数 × 時間(h) × 消費係数 × 季節係数 × 燃料係数
```

OCTOPUS汎用CPUの消費係数: 高優先度(OCT-H) 0.6219 / 通常(OCT) 0.5287 / シェア1/2 0.3732 / シェア1/4 0.2488。季節係数は2025年度は通年1。燃料係数は2026年度 9月まで0.85、**10月から1.15**（年度中に見直しあり）。例: OCTで10ノード×3時間、燃料係数1.15 → 10×3×0.5287×1×1.15 ≈ 18.24ポイント。1,000ポイント（共有10万円コース）でOCTなら約1,644ノード時間。

ポイントはグループ単位で、使い切ると新規投入ができなくなる（実行中のジョブは最後まで走る）。見積りを聞かれたら、ノード占有かシェアかで係数が変わる点も添える。

## ジョブ操作コマンド

| 目的 | コマンド |
|---|---|
| 投入 | `qsub job.sh` → `Request 12345.oct submitted to queue: OCT.` |
| 状態 | `qstat`（自分）、`qstat -f <ID>`（詳細）、`qstatgroup`、`qstatall`、`qstat -Q`（キュー） |
| 開始予定時刻 | `sstat`（ASG状態になると PlannedStartTime が出る。これより遅れることはない）、`sstatgroup`、`sstatall` |
| 実行中の出力を見る | `qcat -o <ID>` / `qcat -e <ID>`（`-f` で追従、`-n 行数`、`-b` 先頭から） |
| 削除・保留 | `qdel <ID>`、`qhold <ID>`、`qrls <ID>` |
| 過去ジョブ | `acstat`（24時間）、`acstat -A`（年度）、`acstatgroup` |
| インタラクティブ | `qlogin -q INT -l elapstim_req=00:10:00 --group=G12345`（プロンプトが返るまで待つ。最大10分） |

状態: `QUE`待ち → `ASG`割当済 → `RUN`実行中 → `EXT`結果転送。`HLD`保留、`S`列が`Y`ならストール。
投入直後にエラーにならず `.e` ファイルに `Exceeded resource limit` が出た場合、OCT/OCT-H/OCT-S（転送キュー）で上限超過している。ノード数・コア数・メモリ・時間を見直す。

## SSH経由でエージェントが操作する場合

2段階認証があるのでエージェントが単独でログインすることはできない。ユーザーが一度対話的にログインしたSSH接続を、ControlMaster で共有して使う。

1. まず `~/.ssh/config` に OCTOPUS 用の Host があるか確認し、あればその名前を使う。管理者用など複数のアカウントがある場合は、ユーザーが指定したものを使う（管理者アカウントは `usage_view` などの表示が一般ユーザーと違う）。そのうえで `ssh -O check <host> 2>&1` を実行し、`Master running` と出れば接続を使える。
2. 未接続なら、ユーザーに次を案内する（config に Host がなければ、先に次の内容の追記を提案する）:

   ```
   Host octopus
       HostName octopus.hpc.osaka-u.ac.jp
       User <利用者番号>
       ControlMaster auto
       ControlPath ~/.ssh/cm-%r@%h:%p
       ControlPersist 8h
       ServerAliveInterval 60
   ```

   そのうえで、ユーザー自身に**別のターミナルで** `ssh -fN octopus` を実行してもらい、パスワードとワンタイムコードを入力してもらう（パスワード入力は対話端末が必要）。パスワードやTOTPをエージェントが受け取ったり保存したりしてはいけない。接続が切れたら（`ControlPersist` 切れ、10日アイドル切断など）同じ手順をもう一度頼む。
3. 以後は `ssh octopus '<command>'` で実行する。リモートは非ログインシェルなので、moduleを使うときは `ssh octopus 'bash -lc "module load BaseCPU && ifx --version"'` のようにログインシェル経由にする。ファイルは `scp`/`rsync -e ssh` で同じ接続を使える。

操作の線引き:
- `qstat`、`sstat`、`usage_view`、`groups`、`ls`、`cat`、`module avail` などの読み取りは自由に実行してよい。
- **`qsub` はポイントを消費し、`qdel`・`rm` は取り消せない**。実行前にスクリプト（キュー、ノード数、時間）と概算ポイントを示して確認を取る。ユーザーが明示的に任せた場合は例外。
- フロントエンドで重い処理を走らせない。ビルドが重い場合も `make -j8` 程度に抑える。
- 投入後は `sstat` で開始予定時刻を、終了後は `.o`/`.e` ファイルを確認して結果を報告する。DBG なら20秒ほどで始まることが多い。状態は QUE → PRR → RUN → POR と進み、終わると `qstat` から消える。`acstat` への反映には少し時間がかかる。
- `.e` ファイルに出る `Loading BaseCPU/2026 ... Loading requirement: ...` は module の通常のメッセージで、エラーではない。

2026年10月に一般ユーザーで実機を確認した結果:
- `cpunum_job` を指定しない DBG ジョブで `nproc` は 256 だった（ノード全体が割り当てられる）。
- `NQSV_MPIOPTS` は `-T intmpi`/`openmpi` を付けたジョブでのみ設定され、付けないと空になる。MPI の起動に失敗したら、まず `-T` が付いているかを疑う。
- `PBS_NODEFILE` は常に設定される。
- `-b 2 -T intmpi` の DBG ジョブで、`NQSV_MPIOPTS` は `-machinefile .../mpinodes` だった。`mpirun ${NQSV_MPIOPTS} -np 512` は256ランクずつ2ノードに分かれた。
- `mpirun ${NQSV_MPIOPTS} -np 8 -ppn 4` では `-ppn` が無視され、8ランクすべてが1ノードに載った（machinefile が埋まる順に詰めるため）。`mpirun -hostfile ${PBS_NODEFILE} -np 8 -ppn 4` なら4ランクずつに分かれた。ノードあたりのランク数を減らしたいときは、必ず後者を使う。
- マルチノードのジョブでは、`.o`/`.e` が `%NQSV(INFO): ------- Output from job:0000 -------` のようにノードごとの区切りを含む。スクリプト本体の出力は job:0000（マスター）の部分に出る。
- 計算ノードから外部（pypi.org など）へは接続できない。

## 回答の作法

- ジョブスクリプトを出すときは、各 `#PBS` 行が何を要求しているか（ノード数×コア、時間、課金先）が分かる短いコメントを付け、推定ポイント消費を添えると親切。
- ユーザーのグループ名・利用者番号が不明ならプレースホルダ（`G12345`/`u6a123` など）と明記し、`groups` で確認する方法を一言添える。
- SlurmやPBS Proの書き方（`#SBATCH`, `-l select=`, `srun`, `mpiexec -n` 直書きなど）を持ち込まない。他システムからの移行相談では対応表を示す（`references/jobs.md` 末尾）。
- 手順がマニュアルにない質問（特殊なライブラリ、GPU利用など）は推測で断言しない。OCTOPUSにはGPU計算ノードがない（フロントエンドに可視化用L4が1枚あるだけ）ことなど、確実な事実を伝えたうえで、D3センターの問い合わせ窓口（https://www.hpc.cmc.osaka-u.ac.jp/support/contact/auto_form/）を案内する。2段階認証の端末を失くした場合は system@cmc.osaka-u.ac.jp に利用者番号を添えて連絡。

## 参照ファイル

- `references/jobs.md`: ジョブクラス詳細、qsubオプション全表、並列形態別テンプレート、状態コード、待ち時間短縮、Slurm対応表
- `references/environment.md`: ログイン/2FA初期設定、ファイル転送、ストレージ、module、コンパイラとライブラリ、Python/Miniforge、Apptainer、DCV、VS Code
- `references/apps.md`: インストール済みアプリ一覧（モジュール名）とアプリ別ジョブスクリプト
