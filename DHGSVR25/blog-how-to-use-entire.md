# Entire CLI 実践ガイド：AIエージェント時代の「思考のバージョン管理」

GitHubの元CEOであるNat Friedman氏らが立ち上げた次世代開発プラットフォーム「Entire」。シードラウンドで6,000万ドル（約90億円）を調達し、AIエージェントとの協業を前提としたバージョン管理の再定義に挑んでいます。

この記事では、実際のプロジェクト（DHGSVR25 受講生ポートフォリオサイト）への導入を通じて、Entire CLIの全コマンドを解説します。

- 公式サイト: [entire.io](https://entire.io)
- 解説記事: [AIネイティブ時代のGitHub？元CEOが放つ次世代開発プラットフォーム「Entire」](https://note.com/o_ob/n/n3bc4cebbb72f)

## Entireとは何か：30秒で理解する

**Git は「何が変わったか（What）」を記録する。Entire は「なぜそう変えたか（Why）」を記録する。**

Claude CodeやCursorなどのAIエージェントと対話しながらコードを書く時代、数百行の変更が一瞬で生まれます。しかしセッションを閉じれば、その思考プロセスや判断理由は消えてしまいます。Entireは、AIとの対話履歴（セッション）をGitリポジトリ内に自動保存し、「コードが生まれるまでの思考のバージョン」を管理します。

### 3つの特徴

1. **Git互換** — 既存のGitワークフローを壊さない。メタデータは孤立ブランチ（orphan branch）に保存
2. **外部DB不要** — リポジトリをcloneするだけで過去のAIセッション履歴もすべて手元に揃う
3. **エージェント記憶の永続化** — 過去の意思決定プロセスをコンテキストとして次のセッションに引き継げる

## インストールと有効化

```bash
# インストール（1行）
curl -fsSL https://entire.io/install.sh | bash

# バージョン確認
entire version
# Entire CLI 0.4.4 (2f0ad9ab)
# Go version: go1.25.6
# OS/Arch: darwin/arm64
```

## 全コマンド解説

### `entire enable` — プロジェクトで有効化する

```bash
entire enable
```

Gitリポジトリ内でEntireを有効化します。実行すると以下が行われます:

- `.entire/settings.json` に設定ファイルを作成
- Gitフックを4つ追加（`prepare-commit-msg`, `commit-msg`, `post-commit`, `pre-push`）
- 孤立ブランチ `entire/checkpoints/v1` を作成（メタデータ保存用）

#### DHGSVRプロジェクトでの実例

```bash
cd /Users/aki/git.local/DHGSVR
entire enable
```

実行後の設定ファイル（`.entire/settings.json`）:

```json
{
  "strategy": "manual-commit",
  "enabled": true,
  "telemetry": true
}
```

Gitフックが自動追加されます:

```bash
$ cat .git/hooks/prepare-commit-msg
#!/bin/sh
# Entire CLI hooks
entire hooks git prepare-commit-msg "$1" "$2" 2>/dev/null || true
```

これにより、コミットするたびにEntireが自動的にチェックポイントを記録します。

#### オプション

| フラグ | 説明 |
|--------|------|
| `--strategy auto-commit` | 自動コミット戦略（デフォルトは `manual-commit`） |
| `--agent claude-code` | 特定のエージェント用にフックを設定（非対話モード） |
| `--force` | 既存のフックを再インストール |
| `--skip-push-sessions` | `git push` 時のセッションログ自動プッシュを無効化 |
| `--local` | 設定を `.entire/settings.local.json` に書き込み（gitignore対象） |

#### 2つの戦略（Strategy）

- **`manual-commit`（デフォルト）**: 開発者が普通に `git commit` するタイミングでチェックポイントを作成。シャドウブランチ（`entire/<commit-hash>`）に作業中のデータを一時保存し、コミット時に `entire/checkpoints/v1` ブランチに集約（condense）します。
- **`auto-commit`**: AIエージェントの操作ごとに自動コミット。より細かい粒度で記録されますが、コミット履歴が増えます。

### `entire status` — 現在の状態を確認する

```bash
entire status
```

Entireの有効/無効状態と、使用中の戦略を表示します。

```
Enabled (manual-commit)
```

最もシンプルなコマンドですが、「そもそもEntireが有効になっているか？」を確認する第一歩です。

### `entire explain` — チェックポイントを読み解く

```bash
entire explain
```

これがEntireの真骨頂です。AIエージェントとの対話セッションで生まれたチェックポイントを「人間が読める形」で表示します。

#### DHGSVRプロジェクトでの実行例

```bash
$ entire explain
Branch: main
Checkpoints: 0

No checkpoints found on this branch.
Checkpoints will appear here after you save changes during a Claude session.
```

現時点ではEntireを有効化した直後のため、チェックポイントはまだありません。次にClaude Codeでコードを変更してコミットすると、ここにチェックポイントが表示されるようになります。

#### 表示レベル

| フラグ | 表示内容 |
|--------|----------|
| （なし） | チェックポイント一覧 |
| `--short` | サマリのみ（ID, セッション, タイムスタンプ, トークン数, 意図） |
| `--full` | 完全なトランスクリプト（すべてのプロンプトと応答） |
| `--raw-transcript` | 生のJSONL形式のトランスクリプト |

#### 特定のコミットやチェックポイントを調べる

```bash
# 特定のコミットに紐づくチェックポイントを表示
entire explain --commit abc1234

# 特定のチェックポイントを詳細表示
entire explain --checkpoint <id>

# セッションIDでフィルタ
entire explain --session <session-id>
```

#### AI要約の生成

```bash
# チェックポイントのAI要約を生成
entire explain --checkpoint <id> --generate

# 既存の要約を再生成
entire explain --checkpoint <id> --generate --force
```

**これが意味すること**: 例えばDHGSVRプロジェクトで「フリップブック演出を追加した」セッションがあったとします。数ヶ月後にそのコードを見直す際、`entire explain --commit <hash>` を実行すれば、「なぜ `setTimeout` を使ったのか」「なぜ `requestAnimationFrame` ではなかったのか」といった設計判断の理由が、AIとの対話ログから逆引きできるのです。

### `entire rewind` — チェックポイントに巻き戻す

```bash
entire rewind
```

インタラクティブなTUIで、過去のチェックポイント一覧を表示し、選択したポイントまでコードとエージェントのコンテキストを巻き戻します。

#### オプション

| フラグ | 説明 |
|--------|------|
| `--list` | チェックポイント一覧をJSON出力（スクリプト連携用） |
| `--to <commit>` | 指定コミットまで非対話的に巻き戻す |
| `--logs-only` | ログのみ復元、コードは変更しない |
| `--reset` | ブランチをコミットにリセット（破壊的操作） |

**これが意味すること**: AIに「この関数をリファクタリングして」と頼んだ結果が期待と違った場合、`entire rewind` で「リファクタリングを頼む前」の状態にコードもAIのコンテキストも戻せます。`git reset` と違い、AIの「記憶」も一緒に巻き戻る点が革命的です。

### `entire resume` — ブランチのセッションを再開する

```bash
entire resume <branch-name>
```

指定したブランチに切り替え、そのブランチで最後に行われたAIセッションを復元します。

内部的には:
1. ブランチをチェックアウト
2. ブランチ固有のコミットからセッションIDを特定
3. セッションログをローカルに復元
4. セッション再開コマンドを表示

**これが意味すること**: 例えば「feature/add-dark-mode」ブランチでClaude Codeと作業していて、途中で別の作業に移った後、`entire resume feature/add-dark-mode` で当時の文脈ごと作業を再開できます。AIが「ゼロからのスタート」にならない。

### `entire reset` — セッション状態をリセットする

```bash
entire reset
```

現在のHEADコミットに紐づくシャドウブランチとセッション状態を削除し、まっさらな状態から始めます。

内部的には:
1. 現在のHEADに一致するセッション状態ファイル（`.git/entire-sessions/<id>.json`）を検索
2. 該当するセッションファイルを削除
3. シャドウブランチ（`entire/<commit-hash>-<worktree-hash>`）を削除

```bash
# 特定のセッションだけリセット
entire reset --session <session-id>

# 確認なしで実行
entire reset --force
```

**注意**: `manual-commit` 戦略でのみ動作します。

### `entire doctor` — スタックしたセッションを修復する

```bash
entire doctor
```

1時間以上動きのないセッションや、未集約のチェックポイントデータを検出し、修復を提案します。

各セッションに対して3つの選択肢:
- **Condense**: セッションデータを永続ストレージ（`entire/checkpoints/v1`）に保存
- **Discard**: セッション状態とシャドウブランチデータを破棄
- **Skip**: そのまま放置

```bash
# すべて自動修復
entire doctor --force
```

**これが意味すること**: Claude Codeのセッションが途中でクラッシュした場合や、ターミナルを閉じてしまった場合に、中途半端な状態のデータを安全に処理できます。

### `entire clean` — 孤立したデータを削除する

```bash
# プレビュー（何が削除されるか確認）
entire clean

# 実際に削除
entire clean --force
```

以下の孤立データを検出・削除します:

| データ種別 | 場所 | 孤立条件 |
|-----------|------|----------|
| シャドウブランチ | `entire/<commit-hash>` | セッション集約後に残った一時ブランチ |
| セッション状態 | `.git/entire-sessions/` | 参照するチェックポイントやブランチがない |
| チェックポイントメタデータ | `entire/checkpoints/v1` | rebase/squash後にコミットが存在しない（auto-commitのみ） |

**ポイント**: `manual-commit` のチェックポイントは「集約済み（condensed）」なので孤立扱いにならず、永続保存されます。

### `entire disable` — Entireを無効化する

```bash
# 無効化（フックは残るが何もしない）
entire disable

# 完全アンインストール
entire disable --uninstall
```

`--uninstall` を付けると以下をすべて削除:
- `.entire/` ディレクトリ
- Gitフック（4つ）
- セッション状態ファイル
- シャドウブランチ
- エージェント連携フック（Claude Code, Gemini CLI）

## Entireの内部構造：DHGSVRプロジェクトで観察する

Entireは「メインブランチを汚さない」設計が徹底されています。DHGSVRプロジェクトで有効化後の構造を見てみましょう。

### ファイル構成

```
.entire/
  settings.json        ← 設定（strategy, enabled, telemetry）
  .gitignore           ← settings.local.json 等をgitignore

.git/hooks/
  prepare-commit-msg   ← コミットメッセージ準備時にEntireフック実行
  commit-msg           ← コミットメッセージ検証時にフック実行
  post-commit          ← コミット後にチェックポイント作成
  pre-push             ← プッシュ前にセッションログも一緒にプッシュ

.git/entire-sessions/  ← セッション状態ファイル（ローカルのみ）
```

### ブランチ構成

```bash
$ git branch -a | grep entire
  entire/checkpoints/v1    ← メタデータ永続保存用の孤立ブランチ
```

`entire/checkpoints/v1` は **orphan branch**（孤立ブランチ）です。mainブランチとは一切の親子関係がなく、チェックポイントのメタデータだけを保存する独立した名前空間として機能します。これにより:

- `git log` でメインの履歴が汚れない
- `git clone` でメタデータも一緒にクローンされる
- ベンダーロックインなし（データはすべてGitリポジトリ内）

## 実践シナリオ：DHGSVRでの開発フロー

このプロジェクトでは、Claude Codeと対話しながら以下の作業を行いました:

1. **v1**: 受講生ポートフォリオ一覧（ダークテーマ）
2. **v2**: Playwrightで23サイトのスクリーンショット自動撮影 + カードUI
3. **v3**: フリップブック演出 + ライトテーマ化 + OGP設定

Entireが有効な状態でこれらの作業を行うと、各コミットに以下が自動的に紐づきます:

```
コミット: "DHGSVR25: フリップブック演出、ライトテーマ化"
  └─ チェックポイント
       ├─ セッションID: 2026-02-14-abc123
       ├─ トークン数: 45,000
       ├─ 意図: "ランディング時にスクリーンショットをパタパタアニメーション"
       ├─ プロンプト履歴:
       │    ├─ User: "フリップブック演出を追加して"
       │    ├─ AI: [計画モードで設計 → 実装]
       │    ├─ User: "ライトテーマにして"
       │    └─ AI: [CSS変数オーバーライドで対応]
       └─ 変更ファイル: index.html, making-of.html, ...
```

3ヶ月後に「なぜ `setTimeout` で可変フレームレートにしたのか？」と疑問に思ったら:

```bash
entire explain --commit 6f6cf3e
```

で当時のAIとの対話を丸ごと読み返せます。

## まとめ：コマンド早見表

| コマンド | 一言で | 破壊的？ |
|---------|--------|---------|
| `entire enable` | プロジェクトで有効化 | No |
| `entire status` | 現在の状態確認 | No |
| `entire explain` | チェックポイントを読む | No |
| `entire rewind` | コード+コンテキストを巻き戻す | Yes |
| `entire resume` | ブランチのセッションを再開 | No |
| `entire reset` | セッション状態をリセット | Yes |
| `entire doctor` | スタックしたセッションを修復 | 選択可 |
| `entire clean` | 孤立データを掃除 | `--force`時 |
| `entire disable` | Entireを無効化/アンインストール | `--uninstall`時 |

## Entireが変える未来

Gitが「コードのスナップショット」だとすれば、Entireは「開発の思考ログ」です。

AIエージェントが書くコードの量が人間を上回りつつある今、重要なのは「何行書いたか」ではなく「なぜそう書いたか」です。Entireは、その「なぜ」を失わないためのインフラを提供します。

特にチーム開発において:
- 「このコード、誰がどういう意図で書いたの？」→ `entire explain`
- 「AIの提案がおかしかった、やり直したい」→ `entire rewind`
- 「昨日の続きから再開したい」→ `entire resume`

AIネイティブ時代の開発ワークフローに、Entireは静かに、しかし確実に根を張りつつあります。

---

*このブログ記事は [DHGSVR25 ポートフォリオサイト](https://akihiko.shirai.as/DHGSVR/DHGSVR25/) の制作過程で、Claude Code + Entire CLI を使いながら書かれました。*
