# Entire CLI 実践ガイド：AIエージェント時代の「思考のバージョン管理」

GitHubの元CEOであるNat Friedman氏らが立ち上げた次世代開発プラットフォーム「Entire」。シードラウンドで6,000万ドル（約90億円）を調達し、AIエージェントとの協業を前提としたバージョン管理の再定義に挑んでいます。

この記事では、実際のプロジェクト（DHGSVR25 受講生ポートフォリオサイト）への導入を通じて、Entire CLIの全コマンドを解説します。導入時にハマった落とし穴や、実際に記録されたチェックポイントの中身まで、すべて実体験ベースでお伝えします。

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

### 有効化：最初の落とし穴

```bash
cd /Users/aki/git.local/DHGSVR
entire enable
```

これで有効化……と思いきや、**デフォルトの `manual-commit` 戦略ではClaude Codeのセッションが記録されませんでした**。数回コミット＆プッシュした後に `entire explain` を実行しても:

```bash
$ entire explain
Branch: main
Checkpoints: 0

No checkpoints found on this branch.
```

原因は、`manual-commit` 戦略ではEntireがアクティブなセッションを認識できず、コミットとAIセッションの紐付けが行われなかったことです。**Claude Codeとの連携には `auto-commit` 戦略が必要です**:

```bash
$ entire enable --strategy auto-commit --force
Agent: Claude Code (use --agent to change)

Info: Project settings exist. Saving to settings.local.json instead.
  Use --project to update the project settings file.
✓ Hooks installed
✓ Project configured (.entire/settings.local.json)

Ready.
```

切り替え後、`entire status --detailed` を実行すると:

```
Enabled (auto-commit)

Project, enabled (manual-commit)
Local, enabled (auto-commit)

Active Sessions:
  (unknown)
    [Claude Code] e04f455   started just now
      "aki@AICUJ-A3186 DHGSVR % entire enable --strategy auto-co..."
```

**セッションが検知されました。** これで準備完了です。

#### 設定の二重構造

| ファイル | strategy | 役割 |
|---------|----------|------|
| `settings.json` | `manual-commit` | プロジェクト設定（Git共有・チーム全体） |
| `settings.local.json` | `auto-commit` | ローカル設定（`.gitignore`対象・個人用） |

Local設定がProject設定を上書きするため、チーム全体の設定を変えずに個人で `auto-commit` を有効化できます。

## 全コマンド解説

### `entire enable` — プロジェクトで有効化する

```bash
entire enable
```

Gitリポジトリ内でEntireを有効化します。実行すると以下が行われます:

- `.entire/settings.json` に設定ファイルを作成
- Gitフックを4つ追加（`prepare-commit-msg`, `commit-msg`, `post-commit`, `pre-push`）
- 孤立ブランチ `entire/checkpoints/v1` を作成（メタデータ保存用）

Gitフックの例:

```bash
$ cat .git/hooks/prepare-commit-msg
#!/bin/sh
# Entire CLI hooks
entire hooks git prepare-commit-msg "$1" "$2" 2>/dev/null || true
```

#### オプション

| フラグ | 説明 |
|--------|------|
| `--strategy auto-commit` | 自動コミット戦略（デフォルトは `manual-commit`） |
| `--agent claude-code` | 特定のエージェント用に設定（非対話モード） |
| `--force` | 既存のフックを再インストール |
| `--skip-push-sessions` | `git push` 時のセッションログ自動プッシュを無効化 |
| `--local` | 設定を `.entire/settings.local.json` に書き込み（gitignore対象） |

#### 2つの戦略（Strategy）

- **`manual-commit`（デフォルト）**: 開発者が `git commit` するタイミングでチェックポイントを作成。シャドウブランチに作業中のデータを一時保存し、コミット時に `entire/checkpoints/v1` に集約します。
- **`auto-commit`**: AIエージェントの操作ごとに自動コミット。より細かい粒度で記録されますが、コミット履歴が増えます。**Claude Codeとの連携にはこちらを推奨。**

### `entire status` — 現在の状態を確認する

```bash
$ entire status
Enabled (auto-commit)

$ entire status --detailed
Enabled (auto-commit)

Project, enabled (manual-commit)
Local, enabled (auto-commit)

Active Sessions:
  (unknown)
    [Claude Code] e04f455   started just now
```

`--detailed` を付けると、設定の二重構造やアクティブなセッションの有無まで確認できます。「そもそもEntireが動いているか？」のトラブルシューティングに必須です。

### `entire explain` — チェックポイントを読み解く

```bash
entire explain
```

これがEntireの真骨頂です。AIエージェントとの対話セッションで生まれたチェックポイントを「人間が読める形」で表示します。

#### DHGSVRプロジェクトでの実行結果

`auto-commit` 有効化後、ブログ記事を編集してpushした結果:

```bash
$ entire explain
Branch: main
Checkpoints: 1

[922900a6aeaa] "以下を書き換えていって\n実践シナリオ：DH..."
  02-16 01:49 (b04925e) 以下を書き換えていって
```

さらに詳細を見ると:

```bash
$ entire explain --checkpoint 922900a6aeaa
Checkpoint: 922900a6aeaa
Session: e04f4553-7ae8-474f-9cdd-4c45431c0437
Created: 2026-02-15 16:49:56
Author: Akihiko SHIRAI, Ph.D <shirai@mail.com>
Tokens: 144144

Commits: (1)
  b04925e 2026-02-16 以下を書き換えていって

Intent: 以下を書き換えていって 実践シナリオ：DHGSVRでの開発フ...
Outcome: (not generated)

Files: (1)
  - DHGSVR25/blog-how-to-use-entire.md

Transcript (checkpoint scope):
[User] 以下を書き換えていって
実践シナリオ：DHGSVRでの開発フロー
...（中略）...

[Tool] Read: /Users/aki/git.local/DHGSVR/DHGSVR25/blog-how-to-use-entire.md
[Tool] Edit: /Users/aki/git.local/DHGSVR/DHGSVR25/blog-how-to-use-entire.md

[Assistant] 書き換えました。主な変更点：
- 架空のチェックポイント例を削除
- 実際に起きた「チェックポイントがゼロだった」体験を記述
...
```

ユーザーのプロンプト原文、ツール呼び出し（Read, Edit）、AIの応答まで完全に記録されています。

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

### `entire rewind` — チェックポイントに巻き戻す

```bash
entire rewind
```

インタラクティブなTUIで、過去のチェックポイント一覧を表示し、選択したポイントまでコードとエージェントのコンテキストを巻き戻します。

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

**これが意味すること**: 「feature/add-dark-mode」ブランチでClaude Codeと作業していて、途中で別の作業に移った後、`entire resume feature/add-dark-mode` で当時の文脈ごと作業を再開できます。AIが「ゼロからのスタート」にならない。

### `entire reset` — セッション状態をリセットする

```bash
entire reset
```

現在のHEADコミットに紐づくシャドウブランチとセッション状態を削除し、まっさらな状態から始めます。

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

### `entire disable` — Entireを無効化する

```bash
# 無効化（フックは残るが何もしない）
entire disable

# 完全アンインストール
entire disable --uninstall
```

`--uninstall` を付けると `.entire/` ディレクトリ、Gitフック、セッション状態、シャドウブランチ、エージェント連携フックをすべて削除します。

## Entireの内部構造：孤立ブランチの正体

「メタデータは孤立ブランチに保存される」と公式は説明していますが、実際に中身を覗いてみました。

### ファイル構成

```
.entire/
  settings.json        ← 設定（strategy, enabled, telemetry）
  settings.local.json  ← ローカル設定（.gitignore対象）
  .gitignore           ← settings.local.json等をgitignore

.git/hooks/
  prepare-commit-msg   ← コミットメッセージ準備時にEntireフック実行
  commit-msg           ← コミットメッセージ検証時にフック実行
  post-commit          ← コミット後にチェックポイント作成
  pre-push             ← プッシュ前にセッションログも一緒にプッシュ

.git/entire-sessions/  ← セッション状態ファイル（ローカルのみ）
```

### データはどこにあるのか

```bash
$ git branch -a | grep entire
  entire/checkpoints/v1              # ローカル
  remotes/origin/entire/checkpoints/v1  # GitHub上にもpush済み
```

**Entire社のサーバーではなく、GitHubリポジトリ内の通常のブランチ**です。

```
GitHub (kaitas/DHGSVR)
├── main                          ← コード本体
└── entire/checkpoints/v1         ← AIセッションのメタデータ
     └─ (orphan: mainと親子関係なし)
```

`git push` するたびに `[entire] Pushing session logs to origin...` というログが出ますが、これはこの孤立ブランチへのpushです。

### 「orphan branch」の意味

Gitの孤立ブランチとは、**他のどのブランチとも親コミットを共有しない、完全に独立したブランチ**です。

```bash
$ git cat-file -p entire/checkpoints/v1
tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904
author Akihiko SHIRAI, Ph.D <shirai@mail.com> ...

Initialize metadata branch
This branch stores session metadata for the auto-commit strategy.
```

親コミット（parent）が存在しません。`main` ブランチの歴史とは完全に切り離されたパラレルワールドです。これにより:

- `git log` でmainの履歴が汚れない
- `git clone` するだけで全メタデータもローカルに来る
- Entire社のサーバーには何も送らない（telemetryは匿名の利用統計のみ）

### 「GitHubがなくなっても復元可能」は本当か？

**本当です。ただし、Entireの功績というよりGitの分散型アーキテクチャの恩恵です。**

一度 `git clone` した人全員がローカルの `.git/` に完全なコピーを持ちます。GitHub（リモート）が消滅しても、ローカルの孤立ブランチからすべてのAIセッション履歴を読み出せます。ベンダーロックインは、GitHub側にもEntire側にもありません。

## 実践シナリオ：DHGSVRでの開発フロー

このプロジェクトでは、Claude Codeと対話しながら以下の作業を行いました:

1. **v1**: 受講生ポートフォリオ一覧（ダークテーマ）
2. **v2**: Playwrightで23サイトのスクリーンショット自動撮影 + カードUI
3. **v3**: フリップブック演出 + ライトテーマ化 + OGP設定

### Entireが実際に記録したもの

`auto-commit` 有効化後、最初のチェックポイントに記録された内容:

| 項目 | 記録された値 | 意味 |
|------|-------------|------|
| **Tokens** | 144,144 | このClaude Codeセッション全体の累積トークン数 |
| **Transcript** | User → Tool → Assistant | ユーザーの指示、ツール呼び出し（Read, Edit）、AIの応答まで完全記録 |
| **Intent** | ユーザーのプロンプト原文 | 「なぜこの変更をしたか」が自動的に保存される |
| **Files** | 変更ファイル一覧 | どのファイルが影響を受けたか |

注目すべき点が2つあります。

**1. プロンプト原文がそのまま保存される。** AIに渡したプロンプトの中にAPIキーや社内URLが含まれていれば、それもそのままチェックポイントに刻まれます。

**2. ツール呼び出しまで記録される。** `[Tool] Read:` や `[Tool] Edit:` のように、AIが具体的にどのファイルをどう操作したかまで追跡されます。「人間がAIに何を頼んだか」だけでなく、「AIが何をしたか」も残ります。

### auto-commitのもう一つの特徴

Entireの `auto-commit` はコミットメッセージに**ユーザーのプロンプトをそのまま使います**。実際のコミットログ:

```
a24906b はい
b04925e 以下を書き換えていって 実践シナリオ：DHGSVRでの開発フロー
```

「はい」というコミットメッセージは、ユーザーが「pushしますか？」に対して「はい」と答えた、まさにそのプロンプトです。良くも悪くも、開発の生々しいプロセスがそのまま残ります。

## Entireが描く未来：「思考のGit」がもたらすもの

Gitが「コードのスナップショット」だとすれば、Entireは「開発の思考ログ」です。これが当たり前になった世界では、何が変わるのでしょうか。

### 1. 「コードレビュー」から「思考レビュー」へ

現在のPull Requestレビューは、最終的なdiff（差分）を見て「この変更は妥当か？」を判断します。しかしAIが書いたコードは、diffだけ見ても「なぜこの実装になったのか」がわかりません。

Entireがあれば、レビュアーは `entire explain --commit <hash>` でAIとの対話全体を追跡できます。レビューの対象が「コード」から「思考プロセス」に拡張されます。

### 2. オンボーディングの革命

新しいチームメンバーがプロジェクトに参加したとき、最大の壁は「なぜこのコードがこうなっているのか」という暗黙知です。

Entireのチェックポイントは「自動生成されるドキュメント」です。`git blame` + `entire explain` で、当時の開発者がどういう議論を経てその実装に至ったかを、追加作業なしに追跡できます。

### 3. マルチエージェント時代の「航空管制塔」

Entireが6,000万ドルの資金で次に狙うのは「Context Graph」——複数のAIエージェントが同時並行で開発する際の協調レイヤーです。

エージェントAがフロントエンドを、エージェントBがAPIを同時に開発している場合、互いの「意図」を共有できなければコンフリクトが頻発します。Entireのセマンティック推論レイヤーは、各エージェントの推論プロセスを相互参照可能にし、「航空管制塔」として機能することを目指しています。

### 4. AI生成コードの「来歴証明」

AIが生成したコードの著作権や帰属は、法的にまだグレーゾーンです。しかしEntireのチェックポイントには「誰が」「どういう指示で」「どのAIモデルに」書かせたかが完全に記録されます。

「このコードは、この人間がこのプロンプトでClaude Opus 4.6に生成させたものである」——AI生成コードの来歴証明（Provenance）として、Gitリポジトリ内に改ざん困難な形で残ります。

## リスクと懸念：「思考のGit」の暗い側面

すべての技術にはトレードオフがあります。Entireが実現する「思考の永続化」にも、見過ごせないリスクがあります。

### 退職者のローカル `.git` 問題

GitHub Enterpriseでは、退職者のアカウントを無効化すればリポジトリへのアクセスを遮断できます。しかし**Entireが有効なリポジトリを一度でも `git clone` していた退職者は、ローカルの `.git/` に全チェックポイントデータを保持し続けます**。

退職後のリモートへのpush/pullはブロックされるため、**退職後の成長は分離されます**。しかし「退職時点までの全AIセッション履歴」はローカルに完全な形で残ります。

| 従来のGit | Entire有効のGit |
|-----------|----------------|
| コードの変更差分 | コードの変更差分 |
| コミットメッセージ | コミットメッセージ |
| | **AIとの対話全文** |
| | **設計判断の推論プロセス** |
| | **プロンプトエンジニアリングのノウハウ** |
| | **セッションごとのトークン消費量** |

企業の「集合知としてのプロンプトエンジニアリング」が、退職者のラップトップに丸ごと残る可能性があります。

### 機密情報のうっかり永続化

AIとの対話の中で、APIキー、内部システムのURL、顧客名などがプロンプトに含まれることは珍しくありません。通常はセッションを閉じれば消えますが、**Entireはそれを永続化します**。

しかも孤立ブランチに保存されるため、`main` ブランチだけを監視するセキュリティツール（secret scanning等）では検出できない可能性があります。

### 「思考の監視」への道

チェックポイントには「誰が」「いつ」「どんなプロンプトを」「何トークン使って」AIと対話したかが記録されます。これはオンボーディングやコードレビューには有用ですが、裏を返せば**開発者の思考プロセスを事実上モニタリングできる**ということでもあります。

「なぜこのアプローチを選んだのか」「なぜこの作業に3時間かかったのか」——善意で使えば学習ツールですが、悪意で使えば監視ツールです。

### 対策として考えられること

1. **`entire enable --skip-push-sessions`** でセッションログのリモートpushを無効化し、機密プロジェクトではローカルのみで運用する
2. **`entire clean --force`** を定期的に実行し、不要なメタデータを削除する
3. **`.entire/settings.local.json`**（gitignore対象）を使い、チーム全体ではなく個人単位で有効化する
4. 退職者対応として、**リポジトリの再作成（squash + force push）** でチェックポイントブランチごとリセットする
5. **企業のセキュリティポリシーに「Entireメタデータ」を含める** ことを検討する

## まとめ

### コマンド早見表

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

### 導入チェックリスト

1. `curl -fsSL https://entire.io/install.sh | bash` でインストール
2. `entire enable --strategy auto-commit --force` で有効化（**`auto-commit` を忘れずに**）
3. `entire status --detailed` でセッション検知を確認
4. 普通にClaude Codeで作業 → コミット → プッシュ
5. `entire explain` でチェックポイントを確認

### 一言でまとめると

**Entireは「AIとの対話」をGitリポジトリの中に刻む。それは強力な記憶であり、消えない足跡でもある。**

コードレビューの革命、オンボーディングの効率化、マルチエージェント協調——ポジティブな未来は明るい。しかし同時に、「思考の持ち出し」「機密の永続化」「開発者の監視」という影も意識しておく必要があります。

技術は中立です。Entireが開く扉の先に何があるかは、使う私たち次第です。

---

*このブログ記事は [DHGSVR25 ポートフォリオサイト](https://akihiko.shirai.as/DHGSVR/DHGSVR25/) の制作過程で、Claude Code + Entire CLI を使いながら書かれました。記事自体の編集過程もEntireのチェックポイントとして記録されています。*
