# skills

研究室で使う Claude Code のスキル集。Claude Code のプラグインマーケットプレイスとしてインストールできる。

## スキル一覧

| プラグイン | 内容 |
| --- | --- |
| [octopus](skills/octopus/) | 大阪大学 D3センターのスーパーコンピュータ OCTOPUS の利用（ジョブスクリプト作成、ジョブクラス選択とポイント見積り、module、アプリ実行、SSH 経由のジョブ投入） |

## インストール

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

### スキルだけを手で入れる場合

プラグインを使わず、スキルのディレクトリを `~/.claude/skills/` にリンクしてもよい。

```sh
git clone https://github.com/date-lab/skills.git
ln -s "$PWD/skills/skills/octopus" ~/.claude/skills/octopus
```

## スキルの追加方法

1. `skills/<スキル名>/SKILL.md` を作る（必要なら `references/` などを同じディレクトリに置く）
2. `.claude-plugin/marketplace.json` の `plugins` に次のエントリを追加する

   ```json
   {
     "name": "<スキル名>",
     "description": "<一行の説明>",
     "source": "./",
     "strict": false,
     "skills": ["./skills/<スキル名>"]
   }
   ```

3. 上の「スキル一覧」の表に追記する
4. `claude plugin validate .` でマーケットプレイスの記述をチェックする

## 構成

```
.claude-plugin/marketplace.json   マーケットプレイスの定義
skills/<スキル名>/SKILL.md        各スキル本体
```
