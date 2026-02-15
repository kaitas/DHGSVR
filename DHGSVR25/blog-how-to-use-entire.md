# Entire CLI 実践ガイド：AIエージェント時代の「思考のバージョン管理」

GitHubの元CEOであるNat Friedman氏らが立ち上げた次世代開発プラットフォーム「Entire」。シードラウンドで6,000万ドル（約90億円）という巨額の資金を調達し、AIエージェントとの協業を前提としたバージョン管理の再定義に挑んでいます。

AIがコードを書くことが当たり前になった今、なぜ「新しいGit」が必要なのか。この記事では、実際のプロジェクト（DHGSVR25 受講生ポートフォリオサイト）への導入を通じて、Entire CLIの全コマンドを解説します。導入時にハマった落とし穴や、実際に記録されたチェックポイントの中身まで、すべて実体験ベースでお伝えします。

- 公式サイト: https://entire.io
- 解説記事: AIネイティブ時代のGitHub？元CEOが放つ次世代開発プラットフォーム「Entire」 https://note.com/o_ob/n/n3bc4cebbb72f

## Entireとは何か：30秒で理解する

Git は「何が変わったか（What）」を記録する。Entire は「なぜそう変えたか（Why）」を記録する。

Claude CodeやCursorなどのAIエージェントと対話しながらコードを書く時代、数百行の変更が一瞬で生まれます。しかしセッションを閉じれば、その思考プロセスや判断理由は消えてしまいます。Entireは、AIとの対話履歴（セッション）をGitリポジトリ内に自動保存し、「コードが生まれるまでの思考のバージョン」を管理します。

## なぜ今、新しい「Git」が必要なのか？

これまでのソフトウェア開発の中心は「人間がコードを書くこと」でした。しかしAIエージェントの台頭により、開発者の役割は「コードを書く人」から「エージェントの艦隊を指揮する人」へと急速に変化しています。既存のツールには、AI時代の開発における3つのボトルネックがあります。

- Gitの限界: Gitは「何が変わったか」を記録するが、「なぜその変更が行われたか」という推論プロセスは保存されない
- コンテキストの喪失: AIと対話して数百行のコードを生成しても、セッションを閉じればその思考プロセスや判断理由は消える
- レビューが追いつかない: 従来のIssue管理やPRは、AIが高速かつ並列でコードを生成するスピード感に対応しきれていない

## Entireを構成する「3つの柱」

Entireは、AIと人間が真にコラボレーションするためのプラットフォームとして、以下の3つのコンポーネントを掲げています。

1. Git互換のデータベース — 既存のGit技術を継承しつつ、「意図、制約、推論」を一元管理できるバージョン管理システム。メタデータは孤立ブランチ（orphan branch）に保存されるため、メインの履歴を汚さない。外部DB不要で、リポジトリをcloneするだけで過去のAIセッション履歴もすべて手元に揃う

2. ユニバーサル・セマンティック推論レイヤー — 「コンテキスト・グラフ」を通じて、複数のエージェント間の調整を可能にするセマンティックな理解層

3. AIネイティブなSDLC（ソフトウェア開発ライフサイクル） — エージェントと人間が共に学び、デプロイするための、AI時代に最適化されたフロー

2026年2月時点の対応状況は、Claude CodeとGoogle Geminiが対応済み。OpenAI Codex、GitHub Copilot CLI、Ampが近日対応予定、OpenCodeが計画中です。

## 実際に手を動かしてみた：インストールから最初のチェックポイントまで

### インストール

導入はターミナルから1行です。

  curl -fsSL https://entire.io/install.sh | bash

バージョン確認:

  entire version
  Entire CLI 0.4.4 (2f0ad9ab)
  Go version: go1.25.6
  OS/Arch: darwin/arm64

### 有効化：最初の落とし穴

プロジェクトのディレクトリで entire enable を実行します。

  entire enable

これで有効化完了……と思いきや、デフォルトの manual-commit 戦略ではClaude Codeのセッションが記録されませんでした。数回コミット＆プッシュした後に entire explain を実行しても:

  $ entire explain
  Branch: main
  Checkpoints: 0
  No checkpoints found on this branch.

チェックポイントがゼロ。何も記録されていません。

Gitフック（4つ）は正しく設置されており、孤立ブランチも作成済み。しかし中身は空でした。つまり、Entireは「箱」だけ設置された状態で、中身は空だったのです。

### 解決策：auto-commit 戦略に切り替える

Claude Codeとの連携には auto-commit 戦略が必要です。

  $ entire enable --strategy auto-commit --force
  Agent: Claude Code (use --agent to change)
  Info: Project settings exist. Saving to settings.local.json instead.
  ✓ Hooks installed
  ✓ Project configured (.entire/settings.local.json)
  Ready.

切り替え後に entire status --detailed を実行すると:

  Enabled (auto-commit)
  Project, enabled (manual-commit)
  Local, enabled (auto-commit)
  Active Sessions:
    (unknown)
      [Claude Code] e04f455   started just now

セッションが検知されました。ここで初めてEntireがClaude Codeのセッションを認識し、記録を開始します。

なお、設定は二重構造になっています:

- settings.json（プロジェクト設定）: manual-commit。Gitで共有され、チーム全体に適用
- settings.local.json（ローカル設定）: auto-commit。.gitignore対象で個人用

Local設定がProject設定を上書きするため、チーム全体の設定を変えずに個人でauto-commitを有効化できます。

### 最初のチェックポイントが記録された

auto-commit に切り替えた直後、ブログ記事の編集作業を行い git push しました。すると:

  [entire] Pushing session logs to origin...

このメッセージが出るようになり、entire explain で確認すると:

  $ entire explain
  Branch: main
  Checkpoints: 3

  [979621aec034] "全体を振り返ってリライトしよう。"
    02-16 01:57 (4e719ce) 全体を振り返ってリライトしよう。
  [405a04adaad0] "はい"
    02-16 01:53 (a24906b) はい
  [922900a6aeaa] "以下を書き換えていって\n実践シナリオ：DH..."
    02-16 01:49 (b04925e) 以下を書き換えていって

3件のチェックポイントが記録されています。さらに詳細を見ると:

  $ entire explain --checkpoint 922900a6aeaa
  Checkpoint: 922900a6aeaa
  Session: e04f4553-7ae8-474f-9cdd-4c45431c0437
  Created: 2026-02-15 16:49:56
  Author: Akihiko SHIRAI, Ph.D
  Tokens: 144144
  Files: (1)
    - DHGSVR25/blog-how-to-use-entire.md
  Transcript (checkpoint scope):
  [User] 以下を書き換えていって 実践シナリオ：DHGSVRでの開発フロー ...
  [Tool] Read: /Users/aki/git.local/DHGSVR/DHGSVR25/blog-how-to-use-entire.md
  [Tool] Edit: /Users/aki/git.local/DHGSVR/DHGSVR25/blog-how-to-use-entire.md
  [Assistant] 書き換えました。主な変更点：...

ユーザーのプロンプト原文、ツール呼び出し（Read, Edit）、AIの応答まで完全に記録されています。セッション全体の累積トークン数（144,144）、変更ファイル一覧、そして「なぜこの変更をしたか」がIntentとして自動保存されています。

注目すべき点が2つあります。

1つ目は、プロンプト原文がそのまま保存されること。AIに渡したプロンプトの中にAPIキーや社内URLが含まれていれば、それもそのままチェックポイントに刻まれます。

2つ目は、ツール呼び出しまで記録されること。AIが具体的にどのファイルをどう操作したかまで追跡されます。「人間がAIに何を頼んだか」だけでなく、「AIが何をしたか」も残ります。

もう一つ面白いのは、auto-commitのコミットメッセージにユーザーのプロンプトがそのまま使われることです。実際のコミットログ:

  a24906b はい
  b04925e 以下を書き換えていって 実践シナリオ：DHGSVRでの開発フロー

「はい」というコミットメッセージは、ユーザーが「pushしますか？」に対して「はい」と答えた、まさにそのプロンプトです。良くも悪くも、開発の生々しいプロセスがそのまま残ります。

## 全コマンド解説

### entire enable — プロジェクトで有効化する

Gitリポジトリ内でEntireを有効化します。実行すると以下が行われます:

- .entire/settings.json に設定ファイルを作成
- Gitフックを4つ追加（prepare-commit-msg, commit-msg, post-commit, pre-push）
- 孤立ブランチ entire/checkpoints/v1 を作成（メタデータ保存用）

主なオプション:

- --strategy auto-commit : 自動コミット戦略。Claude Codeとの連携にはこちらを推奨
- --agent claude-code : 特定のエージェント用に設定（非対話モード）
- --force : 既存のフックを再インストール
- --skip-push-sessions : git push 時のセッションログ自動プッシュを無効化
- --local : 設定を settings.local.json に書き込み（gitignore対象）

2つの戦略（Strategy）があります:

- manual-commit（デフォルト）: 開発者が git commit するタイミングでチェックポイントを作成。シャドウブランチに作業中のデータを一時保存し、コミット時に entire/checkpoints/v1 に集約
- auto-commit: AIエージェントの操作ごとに自動コミット。より細かい粒度で記録されるが、コミット履歴が増える。Claude Codeとの連携にはこちらが必要

### entire status — 現在の状態を確認する

Entireの有効/無効状態と使用中の戦略を表示します。--detailed を付けると、設定の二重構造やアクティブなセッションの有無まで確認できます。「そもそもEntireが動いているか？」のトラブルシューティングに必須です。

### entire explain — チェックポイントを読み解く

これがEntireの真骨頂です。AIエージェントとの対話セッションで生まれたチェックポイントを「人間が読める形」で表示します。

表示レベルは4段階:

- オプションなし: チェックポイント一覧
- --short : サマリのみ（ID, セッション, タイムスタンプ, トークン数, 意図）
- --full : 完全なトランスクリプト（すべてのプロンプトと応答）
- --raw-transcript : 生のJSONL形式のトランスクリプト

特定のコミットやチェックポイントを調べるには:

- entire explain --commit abc1234 : 特定のコミットに紐づくチェックポイント
- entire explain --checkpoint <id> : 特定のチェックポイントの詳細
- entire explain --session <session-id> : セッションIDでフィルタ

AIによる要約の生成も可能です:

- entire explain --checkpoint <id> --generate : AI要約を生成
- entire explain --checkpoint <id> --generate --force : 既存の要約を再生成

例えばDHGSVRプロジェクトで「フリップブック演出を追加した」セッションがあったとします。数ヶ月後にそのコードを見直す際、entire explain --commit <hash> を実行すれば、「なぜ setTimeout を使ったのか」「なぜ requestAnimationFrame ではなかったのか」といった設計判断の理由が、Claude Codeとの対話ログから逆引きできるのです。

### entire rewind — チェックポイントに巻き戻す

インタラクティブなTUIで、過去のチェックポイント一覧を表示し、選択したポイントまでコードとエージェントのコンテキストを巻き戻します。

主なオプション:

- --list : チェックポイント一覧をJSON出力（スクリプト連携用）
- --to <commit> : 指定コミットまで非対話的に巻き戻す
- --logs-only : ログのみ復元、コードは変更しない
- --reset : ブランチをコミットにリセット（破壊的操作）

AIに「この関数をリファクタリングして」と頼んだ結果が期待と違った場合、entire rewind で「リファクタリングを頼む前」の状態にコードもAIのコンテキストも戻せます。git reset と違い、AIの「記憶」も一緒に巻き戻る点が革命的です。

### entire resume — ブランチのセッションを再開する

指定したブランチに切り替え、そのブランチで最後に行われたAIセッションを復元します。「feature/add-dark-mode」ブランチでClaude Codeと作業していて、途中で別の作業に移った後、entire resume feature/add-dark-mode で当時の文脈ごと作業を再開できます。AIが「ゼロからのスタート」にならない。

### entire reset — セッション状態をリセットする

現在のHEADコミットに紐づくシャドウブランチとセッション状態を削除し、まっさらな状態から始めます。--session <session-id> で特定のセッションだけリセットすることも可能。manual-commit 戦略でのみ動作します。

### entire doctor — スタックしたセッションを修復する

1時間以上動きのないセッションや、未集約のチェックポイントデータを検出し、修復を提案します。各セッションに対して Condense（永続保存）、Discard（破棄）、Skip（放置）の3つの選択肢が提示されます。Claude Codeのセッションが途中でクラッシュした場合や、ターミナルを閉じてしまった場合に使います。

### entire clean — 孤立したデータを削除する

セッション集約後に残った一時ブランチ、参照先のないセッション状態、rebase/squash後に孤立したチェックポイントメタデータなどを検出・削除します。まず entire clean でプレビューし、entire clean --force で実際に削除します。

### entire disable — Entireを無効化する

entire disable で無効化（フックは残るが何もしない）。entire disable --uninstall で完全アンインストール（.entire/ ディレクトリ、Gitフック、セッション状態、シャドウブランチ、エージェント連携フックをすべて削除）。

## Entireの内部構造：孤立ブランチの正体

「メタデータは孤立ブランチに保存される」と公式は説明していますが、実際に中身を覗いてみました。

entire enable を実行すると、リポジトリ内に以下の構造が作られます:

- .entire/settings.json : 設定ファイル（strategy, enabled, telemetry）
- .entire/settings.local.json : ローカル設定（.gitignore対象）
- .git/hooks/ : 4つのGitフック（prepare-commit-msg, commit-msg, post-commit, pre-push）
- .git/entire-sessions/ : セッション状態ファイル（ローカルのみ）

そしてブランチを確認すると:

  $ git branch -a | grep entire
    entire/checkpoints/v1
    remotes/origin/entire/checkpoints/v1

Entire社のサーバーではなく、GitHubリポジトリ内の通常のブランチとしてデータが保存されています。git push するたびに出る [entire] Pushing session logs to origin... というメッセージは、この孤立ブランチへのpushです。

### 「orphan branch」とは何か

Gitの孤立ブランチとは、他のどのブランチとも親コミットを共有しない、完全に独立したブランチです。実際に中身を見てみると:

  $ git cat-file -p entire/checkpoints/v1
  tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904
  author Akihiko SHIRAI, Ph.D <shirai@mail.com> ...

  Initialize metadata branch
  This branch stores session metadata for the auto-commit strategy.

親コミット（parent）が存在しません。main ブランチの歴史とは完全に切り離されたパラレルワールドです。これにより:

- git log でmainの履歴が汚れない
- git clone するだけで全メタデータもローカルに来る
- Entire社のサーバーには何も送らない（telemetryは匿名の利用統計のみ）

### 「GitHubがなくなっても復元可能」は本当か？

本当です。ただし、Entireの功績というよりGitの分散型アーキテクチャの恩恵です。一度 git clone した人全員がローカルの .git/ に完全なコピーを持ちます。GitHubが消滅しても、ローカルの孤立ブランチからすべてのAIセッション履歴を読み出せます。ベンダーロックインは、GitHub側にもEntire側にもありません。

## Entireが描く未来：「思考のGit」がもたらすもの

Gitが「コードのスナップショット」だとすれば、Entireは「開発の思考ログ」です。これが当たり前になった世界では、何が変わるのでしょうか。

### 1. 「コードレビュー」から「思考レビュー」へ

現在のPull Requestレビューは、最終的なdiff（差分）を見て「この変更は妥当か？」を判断します。しかしAIが書いたコードは、diffだけ見ても「なぜこの実装になったのか」がわかりません。

Entireがあれば、レビュアーは entire explain --commit <hash> でAIとの対話全体を追跡できます。レビューの対象が「コード」から「思考プロセス」に拡張されます。

### 2. オンボーディングの革命

新しいチームメンバーがプロジェクトに参加したとき、最大の壁は「なぜこのコードがこうなっているのか」という暗黙知です。ドキュメントは書かれないか、すぐに陳腐化します。

Entireのチェックポイントは「自動生成されるドキュメント」です。git blame + entire explain で、当時の開発者がどういう議論を経てその実装に至ったかを、追加作業なしに追跡できます。

### 3. マルチエージェント時代の「航空管制塔」

Entireが6,000万ドルの資金で次に狙うのは「Context Graph」——複数のAIエージェントが同時並行で開発する際の協調レイヤーです。

エージェントAがフロントエンドを、エージェントBがAPIを同時に開発している場合、互いの「意図」を共有できなければコンフリクトが頻発します。Entireのセマンティック推論レイヤーは、各エージェントの推論プロセスを相互参照可能にし、「航空管制塔」として機能することを目指しています。

### 4. AI生成コードの「来歴証明」

AIが生成したコードの著作権や帰属は、法的にまだグレーゾーンです。しかしEntireのチェックポイントには「誰が」「どういう指示で」「どのAIモデルに」書かせたかが完全に記録されます。

「このコードは、この人間がこのプロンプトでClaude Opus 4.6に生成させたものである」——AI生成コードの来歴証明（Provenance）として、Gitリポジトリ内に改ざん困難な形で残ります。

## リスクと懸念：「思考のGit」の暗い側面

すべての技術にはトレードオフがあります。Entireが実現する「思考の永続化」にも、見過ごせないリスクがあります。

### 退職者のローカル .git 問題

GitHub Enterpriseでは、退職者のアカウントを無効化すればリポジトリへのアクセスを遮断できます。しかしEntireが有効なリポジトリを一度でも git clone していた退職者は、ローカルの .git/ に全チェックポイントデータを保持し続けます。

退職後のリモートへのpush/pullはブロックされるため、退職後の成長（新しいコミットやセッション）は分離されます。しかし「退職時点までの全AIセッション履歴」はローカルに完全な形で残ります。

従来のGitでは「コードの変更差分」と「コミットメッセージ」だけでしたが、Entire有効のGitではそれに加えて「AIとの対話全文」「設計判断の推論プロセス」「プロンプトエンジニアリングのノウハウ」「セッションごとのトークン消費量」まで含まれます。企業の「集合知としてのプロンプトエンジニアリング」が、退職者のラップトップに丸ごと残る可能性があるのです。

### 機密情報のうっかり永続化

AIとの対話の中で、APIキー、内部システムのURL、顧客名などがプロンプトに含まれることは珍しくありません。通常はセッションを閉じれば消えますが、Entireはそれを永続化します。しかも孤立ブランチに保存されるため、main ブランチだけを監視するセキュリティツール（secret scanning等）では検出できない可能性があります。

### 「思考の監視」への道

チェックポイントには「誰が」「いつ」「どんなプロンプトを」「何トークン使って」AIと対話したかが記録されます。これはオンボーディングやコードレビューには有用ですが、裏を返せば開発者の思考プロセスを事実上モニタリングできるということでもあります。

「なぜこのアプローチを選んだのか」「なぜこの作業に3時間かかったのか」——善意で使えば学習ツールですが、悪意で使えば監視ツールです。

### 対策として考えられること

- entire enable --skip-push-sessions でセッションログのリモートpushを無効化し、機密プロジェクトではローカルのみで運用する
- entire clean --force を定期的に実行し、不要なメタデータを削除する
- settings.local.json（gitignore対象）を使い、チーム全体ではなく個人単位で有効化する
- 退職者対応として、リポジトリの再作成（squash + force push）でチェックポイントブランチごとリセットする
- 企業のセキュリティポリシーに「Entireメタデータ」を含めることを検討する

## まとめ

### 導入手順（3行で動く）

  curl -fsSL https://entire.io/install.sh | bash
  entire enable --strategy auto-commit --force
  entire status --detailed

あとは普通にClaude Codeで作業するだけ。entire explain でチェックポイントを確認できます。

### コマンド早見表

- entire enable : プロジェクトで有効化（非破壊）
- entire status : 現在の状態確認（非破壊）
- entire explain : チェックポイントを読む（非破壊）
- entire rewind : コード+コンテキストを巻き戻す（破壊的）
- entire resume : ブランチのセッションを再開（非破壊）
- entire reset : セッション状態をリセット（破壊的）
- entire doctor : スタックしたセッションを修復（選択可）
- entire clean : 孤立データを掃除（--force時に破壊的）
- entire disable : Entireを無効化/アンインストール（--uninstall時に破壊的）

### 一言でまとめると

Entireは「AIとの対話」をGitリポジトリの中に刻む。それは強力な記憶であり、消えない足跡でもある。

コードレビューの革命、オンボーディングの効率化、マルチエージェント協調——ポジティブな未来は明るい。しかし同時に、「思考の持ち出し」「機密の永続化」「開発者の監視」という影も意識しておく必要があります。

技術は中立です。Entireが開く扉の先に何があるかは、使う私たち次第です。

AIエージェントを本格的に業務に組み込んでいるチームにとって、これは「失われた文脈」を取り戻すための必須ツールになるかもしれません。

---

このブログ記事は DHGSVR25 ポートフォリオサイト https://akihiko.shirai.as/DHGSVR/DHGSVR25/ の制作過程で、Claude Code + Entire CLI を使いながら書かれました。記事自体の編集過程もEntireのチェックポイントとして記録されています。
