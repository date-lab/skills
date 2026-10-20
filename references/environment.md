# OCTOPUS 利用環境リファレンス

目次
1. ログインと2段階認証
2. ファイル転送
3. ストレージと利用状況
4. フロントエンドの決まり
5. module とベース環境
6. コンパイラとライブラリ
7. Python / Miniforge
8. Apptainer（コンテナ）
9. Amazon DCV（リモートデスクトップ）
10. VS Code Remote-SSH

---

## 1. ログインと2段階認証

- ホスト: `octopus.hpc.osaka-u.ac.jp`（HPCフロントエンド）、占有型は `voctopusXX.hpc.osaka-u.ac.jp`。
- OCTOPUS には D3C-Sinet（全国共同利用スパコンのネットワーク）経由でアクセスする。
- 認証: 利用者管理システムのパスワード、続いて `Verification code:` にTOTPを入力する。パスワードを忘れたら「WEB利用申請システム」で初期化する。
- 動作確認済みのTOTPアプリ: Google Authenticator / Microsoft Authenticator（iOS・Android）、WinAuth（Windows）、Step Two（macOS）。
- HPCI 利用者は OAuth でもログインできる（HPCI ログインマニュアル OAuth 対応版を参照）。

**初回ログイン**: パスワードでログインすると QR コードと `Your new secret key is:` が表示される。アプリに登録したあと、`Enter code from app (-1 to skip):` に `-1` を入力し、`ok? Hit enter:` で Enter を押すと一度ログアウトする。次回からは TOTP が必要になる。QR コードが崩れるときは端末のウィンドウを広げるか、シークレットキーを手入力する。緊急用スクラッチコードは無効化されている。

**端末を紛失・機種変更した場合**: 管理者の操作が必要になる。system@cmc.osaka-u.ac.jp に利用者番号を添えて連絡する。

`~/.ssh/config` の例（ControlMaster で接続を使い回すと、TOTP の入力が1回で済む）:

```
Host octopus
    HostName octopus.hpc.osaka-u.ac.jp
    User u6a123
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 8h
    ServerAliveInterval 60
```

## 2. ファイル転送

- `scp file u6a123@octopus.hpc.osaka-u.ac.jp:` / `sftp u6a123@octopus.hpc.osaka-u.ac.jp`。いずれもパスワードと TOTP が必要。上記の config があれば `scp file octopus:/octfs/work/G12345/u6a123/` や `rsync -av dir/ octopus:path/` で同じ接続を使える。
- WinSCP: ホスト `octopus.hpc.osaka-u.ac.jp`、ポート22、ユーザー名は利用者番号。テキストファイル（スクリプトなど）は転送モードを「テキスト」にする。改行コードが CRLF のままだと、ジョブスクリプトが正しく動かない。
- Web ブラウザ（Nextcloud）による転送、SQUID との NFS 経由のデータ交換もある。詳細は D3 センター Web の `transfer_web` と `transfer_squid` のページを参照。

## 3. ストレージと利用状況

| 領域 | パス | 容量 | 備考 |
|---|---|---|---|
| Home | `/octfs/home/<利用者番号>` | 100GiB/人 | 拡張不可。ブラウザアクセス領域を含む |
| Work | `/octfs/work/<グループ>/<利用者番号>` | 5TiB/グループ（追加購入可） | 高速・大容量。ファイル数はソフト上限3億、ハード上限5億。グループ配下は共有に使える |

ファイルシステムは Lustre（DDN EXAScaler、実効 3.58PB）。**バックアップはない**。

`usage_view` の出力:
- `[Group summary]`: OCTOPUS points（使用量/上限/残り/使用率）と HDD（Work 使用量）
- `[Detail]`: ユーザーごとのポイント、Home 使用量/上限、Work 使用量
- `[Node-hours]`: 使用したノード時間と、残りポイントを換算した利用可能ノード時間

ノード時間は毎日7:00に前日0:00時点の値へ更新される。ディスクは15分ごとの更新で、100MB 未満は切り捨てて表示される。

Web ポータル: https://octopusportal.hpc.osaka-u.ac.jp/ （ログインには2段階認証が必要）。ジョブ件数やノード時間の統計、ノード稼働状況を確認できる。

## 4. フロントエンドの決まり

- HPC フロントエンドは2台。Xeon 6980P×2、768GB、NVIDIA L4×1（可視化用）、RHEL 9.4。
- **高負荷なプログラムは実行禁止**。長時間動いているプロセスは止められることがある。実行は qsub か qlogin で行う。占有型フロントエンドは各グループ専用の VM なので、高負荷な処理を実行してよい。
- ログインシェルは bash で、変更できない。初期設定は `~/.bash_profile` に書く。csh スクリプトで module を使う場合は、先に `source /etc/profile.d/modules.csh` を実行する。
- 10日以上アイドル状態のログインセッションは通知なしで切断される。
- Core ファイルは既定で出力されない（soft limit が0）。必要なら `ulimit -c unlimited` を設定する。

## 5. module とベース環境

| コマンド | 説明 |
|---|---|
| `module avail` | 利用できるモジュールの一覧 |
| `module list` | ロード済みのモジュール |
| `module load X` / `unload X` | ロード / アンロード |
| `module switch A B` | A を B に入れ替える |
| `module purge` | すべてアンロードする |
| `module show X` | モジュールの設定内容を表示する |

各ベース環境には `/2025` と `/2026(default)` がある（2026年10月に実機で確認）。バージョンを省略すると `/2026` が読まれる。

| ベース環境 | 内容 |
|---|---|
| `BaseCPU` | Intel oneAPI（/2026 は 2025.3、/2025 は 2025.2）。コンパイラ、Intel MPI、MKL、TBB、VTune、Advisor など。推奨環境 |
| `BaseGCC` | GNU コンパイラ、Open MPI、GSL |
| `BasePy` | Python 3.13.5、PyTorch、TensorFlow、Keras（v8.0 マニュアルの時点では 2.8.0 / 2.20.0rc0 / 3.11.2）。scikit-learn は入っていない |
| `BaseR` | R 4.5.1、pbdR |
| `BaseJDK` | OpenJDK 21.0.8 |
| `BaseJulia` | Julia 1.11.6 |
| `BaseApp` | ISV/OSS アプリケーション用 |
| `BaseExtra` | 追加環境 |

ライブラリやアプリのモジュールは、ベース環境をロードしたあとに表示される（例: `BaseCPU` を読むと `hdf5/1.14.6` や `netcdf-c/4.9.3` が現れる）。

## 6. コンパイラとライブラリ

| | C | C++ | Fortran | OpenMP |
|---|---|---|---|---|
| Intel (BaseCPU) | `icx` | `icpx` | `ifx` | `-qopenmp` |
| Intel MPI | `mpiicx` | `mpiicpx` | `mpiifx` | |
| GNU (BaseGCC) | `gcc` | `g++` | `gfortran` | `-fopenmp` |
| Open MPI | `mpicc` | `mpic++` | `mpif90` | |

計算ノードは Granite Rapids（Xeon 6980P）。最適化には `-O3 -xHost` などが使える。コンパイルはフロントエンドで行い、フロントエンドと計算ノードは同じ CPU である。

**MKL**（BLAS / LAPACK / ScaLAPACK を含む）:
```
icx -qmkl=sequential sample.c          # シリアル
icx -qmkl=parallel sample.c            # スレッド並列
mpiicx -qmkl=cluster sample.c          # MPI (ScaLAPACK)
# -static-intel を付けると静的リンク
```

**HDF5**: `module load hdf5`（シリアル / C++）または `module load hdf5.mpi`（並列版）を読み、`-lhdf5` を付ける。

**NetCDF**:
```
module load netcdf-c       && icx  x.c   -lnetcdf
module load netcdf-cxx     && icpx x.cpp -lnetcdf_c++4 -lnetcdf
module load netcdf-fortran && ifx  x.f90 -lnetcdff -lnetcdf
```

**PnetCDF**: `pnetcdf-c` / `pnetcdf-cxx` / `pnetcdf-fortran` のいずれかを読み、`mpiicx` などで `-lpnetcdf` を付ける。

## 7. Python / Miniforge

**BasePy + venv（推奨）**:
```bash
module load BasePy
python3 -m venv /octfs/work/G12345/u6a123/myenv
source /octfs/work/G12345/u6a123/myenv/bin/activate
pip install numpy
```
- 計算ノードからは外部に接続できない。インストールはフロントエンドで済ませておく。
- システム側で BasePy にパッケージを追加することはない。必要なパッケージは各自でインストールする。
- venv を Work に置くと、Home の100GiB を圧迫しない。BasePy に入っている PyTorch などを venv から使いたい場合は、`--system-site-packages` を付けて venv を作る。

**Miniforge**:
```bash
echo 'export MAMBA_EXTRACT_THREADS=8' >> ~/.bashrc && source ~/.bashrc   # 多コアでのハング対策
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
bash Miniforge3-Linux-x86_64.sh          # 例: /octfs/home/u6a123/miniforge3 にインストール
source ~/.bashrc
mamba config set auto_activate_base False
# 環境とパッケージキャッシュを Work に置く（Home の容量対策）
mamba config prepend envs_dirs /octfs/work/G12345/u6a123/mamba_envs
mamba config prepend pkgs_dirs /octfs/work/G12345/u6a123/mamba_pkgs
mamba create -n test-env python=3.12 && mamba activate test-env && mamba install numpy
```
ジョブスクリプトの中では、`mamba activate test-env` の前に `source ~/.bashrc` が必要になる場合がある。

## 8. Apptainer（コンテナ）

- イメージの取得:
  - OCTOPUS のローカルレジストリ: `apptainer build test.sif oras://cntm:5000/master_image/test:1`
  - Docker Hub: `apptainer build rocky9.sif docker://rockylinux:9`
  - 手元で作った `.sif` を scp で転送してもよい（root 権限がある手元の環境でビルドする方が制約が少ない）。
- Work 領域で build や sandbox の作成を行うときは、事前に `newgrp <グループ>` を実行する。ディレクトリのグループ所有権とプライマリグループが一致している必要があるため。
- キャッシュは既定で Home に置かれる。`export APPTAINER_CACHEDIR=/octfs/work/G12345/u6a123/.apptainer` で Work に移せる。削除は `apptainer cache clean`。
- sandbox を使ったカスタマイズ:
  ```
  apptainer build -f --sandbox --fix-perms test test.sif   # 作成（-f は fakeroot）
  apptainer run -f -w test                                  # 中に入って dnf / pip でインストール → exit
  apptainer build -f test.sif test                          # sif に固める
  ```
  `rm -fr` で消せない sandbox は `apptainer exec -f --bind <親ディレクトリ> <任意のsif> rm -r <sandbox>` で削除する。
- def ファイルを使う場合: `Bootstrap: oras` / `From: cntm:5000/master_image/test:1`、`%files`、`%post`、`%runscript` を書き、`apptainer build -f test.sif test.def` でビルドする。
- 実行: `apptainer exec image.sif cmd`。Home と `/tmp` は自動でマウントされる。Work は `--bind $(pwd)` などで明示する。環境変数は `--env VAR=val` や `--env-file` で渡す。

## 9. Amazon DCV（リモートデスクトップ）

フロントエンドの GPU を使い、GUI での可視化（ParaView、VisIt など）を行える。

```
dcv create-session --type=virtual mysession
dcv list-sessions                            # どちらのフロントエンド（octopus1 / octopus2）にいるか確認
```
手元の DCV クライアントで `octopus1.hpc.osaka-u.ac.jp:5901/#mysession` に接続し、ユーザー名とパスワードでログインする。使い終わったら、セッションを作ったのと同じホストで `dcv close-session mysession` を実行する。

## 10. VS Code Remote-SSH

`~/.ssh/config` に `Host OCTOPUS` などを書き、コマンドパレットから「Remote-SSH: Connect to Host」を選んで接続する。`Enter password` にはパスワード、`Enter verification code` には TOTP を入力する（複数回求められることがある）。接続後の VS Code ターミナルでは qsub などのコマンドを実行できるが、重い処理はジョブとして投入する。
