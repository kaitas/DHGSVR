# DHGSVR25 ポートフォリオサイトの作り方

デジタルハリウッド大学大学院「先端テクノロジー特論：人工現実」の受講生ポートフォリオサイト（DHGSVR25）を制作した過程をまとめます。

公開URL: https://akihiko.shirai.as/DHGSVR/DHGSVR25/

## サイトの進化

### v1: 初版（ダークテーマ）

最初のバージョンは、共有CSSを使ったダークテーマのシンプルな一覧ページでした。

- 共有CSS（`assets/styles.css`）のダークカラースキームをそのまま利用
- 受講生のGitHub Pagesリンクをカード形式で一覧化
- レスポンシブグリッドレイアウト（`auto-fit, minmax`）

### v2: スクリーンショット追加 + セクション分離

各受講生サイトのスクリーンショットを撮影してカードに追加しました。

- **Playwright** で全23サイトのスクリーンショットを自動撮影
- `sips -Z 640` でサムネイルサイズにリサイズ（合計約3MB）
- 1行3列の固定グリッドに変更（レスポンシブ: 900px以下で2列、560px以下で1列）
- 「New Year 2026 年賀状プロジェクト」と「受講生ポートフォリオ」にセクション分離

#### スクリーンショット撮影の工夫

```javascript
// networkidle でタイムアウトするサイトは load + 待機で対応
await page.goto(url, { waitUntil: 'load', timeout: 20000 });
await page.waitForTimeout(3000);
await page.screenshot({ path: filePath, type: 'png' });
```

一部のサイト（marinakawaguchi.com など）は `networkidle` だとタイムアウトしたため、`load` イベント + 3秒待機でフォールバックしました。

### v3: フリップブック演出 + ライトテーマ化

ページの印象を大きく変える2つの変更を行いました。

#### フリップブック（パタパタ）演出

ページ読み込み時に、23枚のスクリーンショットが高速でめくれる「パタパタ」演出を実装しました。

**技術的なポイント:**

- 23枚のPNG画像を `new Image()` で並列プリロード
- `setTimeout` で可変フレームレートを実現:
  - 開始時: 60ms/frame（約16fps、テンポよく）
  - 終了時: 300ms/frame（減速して「落ち着く」演出）
  - 二次曲線 `progress²` で自然な減速カーブ
- 2周（46フレーム）サイクル後にCSSトランジション（1秒）でフェードアウト
- 3秒のフォールバックタイマーで低速回線にも対応

```javascript
// 可変フレームレートの核心部分
var progress = currentFrame / totalFrames;
var interval = 60 + (progress * progress) * 300;
setTimeout(tick, interval);
```

**アクセシビリティ対応:**

```css
@media (prefers-reduced-motion: reduce) {
  #flipbook-overlay { display: none; }
}
```

`prefers-reduced-motion` を検知してアニメーション非表示。JS無効環境にも `<noscript>` で対応。

#### ライトテーマ化

共有CSSはルートのポートフォリオ（ダークテーマ）と共用のため、DHGSVR25ページのみにライトテーマを適用しました。

**アプローチ: CSS Custom Properties のインラインオーバーライド**

```html
<link rel="stylesheet" href="../assets/styles.css" />
<style>
  :root {
    --bg: #fafbfc;      /* was #0f1117 */
    --panel: #ffffff;   /* was #0f131d */
    --text: #1a1d26;    /* was #e9ecf5 */
    --accent: #0ea573;  /* was #7ce7c1 (白背景用に暗く) */
    /* ... */
  }
</style>
```

`<link>` の後に `<style>` を配置することで、同じセレクタ（`:root`）の後勝ちルールにより変数が上書きされます。

**課題と解決:**

| 問題 | 解決 |
|------|------|
| `.site-header` の背景が `rgba(15,17,23,0.85)` とハードコード | 個別にオーバーライド |
| アクセントカラー `#7ce7c1` が白背景で薄すぎる | `#0ea573` に暗めに調整 |
| ルートの index.html への影響 | インライン `<style>` なので影響なし |

## 使用技術

- HTML5 / CSS3（Custom Properties）
- Vanilla JavaScript（外部ライブラリなし）
- Playwright（スクリーンショット自動撮影）
- macOS `sips`（画像リサイズ）
- GitHub Pages（デプロイ）

## ファイル構成

```
DHGSVR25/
  index.html                  ← メインページ（ライトテーマ + フリップブック）
  index-v1-dark.html          ← v1アーカイブ（ダークテーマ）
  making-of.html              ← 制作の舞台裏ページ
  blog-making-of-dhgsvr25.md  ← このファイル（ブログ用）
  screenshots/                ← 受講生作品スクリーンショット（23枚）
  screenshots/archive/        ← バージョン比較用スクリーンショット
    v1-dark-full.png
    v2-light-full.png
```

## 振り返り

- CSS Custom Properties のオーバーライドは、共有CSSを壊さずにページ単位でテーマを切り替える実用的な手法
- フリップブック演出は `requestAnimationFrame` ではなく `setTimeout` を使うことで「パタパタ」感のある離散的なフレーム切り替えを実現
- Playwright による自動スクリーンショットは、学生サイトのビジュアルプレビューを効率的に生成できた
- `prefers-reduced-motion` や `<noscript>` によるアクセシビリティ配慮は最初から組み込むべき
