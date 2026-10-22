# skills

研究室で使う Codex / Claude Code 共通のスキル集。どちらもプラグインマーケットプレイスからインストールできる。

## スキル一覧

| スキル | 内容 |
| --- | --- |
| [octopus](skills/octopus/) | 大阪大学 D3センターのスーパーコンピュータ OCTOPUS の利用（ジョブスクリプト作成、ジョブクラス選択とポイント見積り、module、アプリ実行、SSH 経由のジョブ投入） |

## Claude Code へのインストール

Claude Code の中で、まずこのリポジトリをマーケットプレイスとして登録する。

```
/plugin marketplace add date-lab/skills
```

次に使いたいスキルをプラグインとしてインストールする。

```
/plugin install octopus@date-lab
```

`/plugin` を引数なしで実行すると、一覧から選んでインストールすることもできる。インストール後は Claude Code を再起動する。

### 更新

```
/plugin marketplace update date-lab
```

## Codex へのインストール

まずこのリポジトリをマーケットプレイスとして登録する。

```sh
codex plugin marketplace add date-lab/skills
```

次に使いたいスキルをプラグインとしてインストールする。

```sh
codex plugin add octopus@date-lab
```

### 更新

```sh
codex plugin marketplace upgrade date-lab
```

## スキルの追加方法

1. `skills/<スキル名>/SKILL.md` を作る（必要なら `references/` などを同じディレクトリに置く）
2. Claude Code のプラグインとして配布する場合は、`.claude-plugin/marketplace.json` の `plugins` に次のエントリを追加する

   ```json
   {
     "name": "<スキル名>",
     "description": "<一行の説明>",
     "source": "./",
     "strict": false,
     "skills": ["./skills/<スキル名>"]
   }
   ```

3. Codex 向けに `.codex-plugin/plugin.json` の `skills` に `./skills/<スキル名>` を追加する。同じプラグインに含めるスキルをここで指定する。別プラグインとして配布する場合は、独立したプラグインフォルダと `.agents/plugins/marketplace.json` のエントリを作る
4. 上の「スキル一覧」の表に追記する
5. Claude Code 向けのマーケットプレイスを変更した場合は `claude plugin validate .` で記述をチェックする


## 構成

```
.agents/plugins/marketplace.json Codex 用マーケットプレイスの定義
.codex-plugin/plugin.json        Codex 用プラグインの定義
.claude-plugin/marketplace.json   Claude Code 用マーケットプレイスの定義
skills/<スキル名>/SKILL.md        各スキル本体
```
