# octopus skill

大阪大学 D3センターのスーパーコンピュータ OCTOPUS を使うための Claude Code スキル。
ジョブスクリプトの作成・修正、ジョブクラスの選び方とポイントの見積り、module やコンパイラの使い方、アプリの実行、SSH 経由でのジョブ投入や状態確認を扱う。

## インストール

リポジトリ直下の [README](../../README.md) を参照。

## 構成

- `SKILL.md`: 基本事項、ジョブスクリプトのルール、SSH 経由で操作するときの手順
- `references/jobs.md`: ジョブクラス、qsub オプション、並列の形態ごとのテンプレート、ポイント計算
- `references/environment.md`: ログイン、ファイル転送、ストレージ、module、Python、Apptainer
- `references/apps.md`: インストール済みアプリとジョブスクリプトの例
- `evals/evals.json`: 動作確認に使ったテストの質問

情報源は NEC「OCTOPUS 利用者向けマニュアル v8.0」（2026/04）と D3センターの Web マニュアル。モジュールのバージョン、キューの構成、MPI の挙動は 2026年10月に実機で確認した。
