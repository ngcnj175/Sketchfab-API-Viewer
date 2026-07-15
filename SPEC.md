# Direct VR Viewer 仕様書

公開URL: https://ngcnj175.github.io/Sketchfab-API-Viewer/
リポジトリ: https://github.com/ngcnj175/Sketchfab-API-Viewer
最終コミット: `7bfa7e2 Update index.html`

---

## 1. 概要

ブラウザ単体で動作する 3D モデルビューア。Sketchfab / Dropbox / 任意の直リンク URL、あるいはローカルファイルから 3D モデルを読み込み、WebXR による VR 表示までを行う。サーバー側処理は Sketchfab API を仲介する Cloudflare Worker のみ。

- 構成: 単一 HTML ファイル ([index.html](index.html)) + 外部 CDN ライブラリ
- 実行環境: モダンブラウザ (WebXR 対応が VR モードに必要 / Meta Quest ブラウザ想定)
- ホスティング: GitHub Pages

---

## 2. 依存ライブラリ

すべて CDN から取得(バンドル無し)。

| ライブラリ | バージョン | 用途 |
|---|---|---|
| three.js | r128 | 3D レンダリング / WebXR |
| GLTFLoader / OBJLoader / STLLoader / FBXLoader | three r0.128 examples | 各形式ローダー |
| JSZip | 3.10.1 | ZIP 解凍 |
| fflate | 0.7.4 | (読み込みのみ、実処理はJSZip) |

外部 API プロキシ: `https://sketchfab-proxy.ngcnj175.workers.dev` (Cloudflare Workers)

---

## 3. 対応入力

### 3.1 URL / ID 入力欄
- **Sketchfab URL** 例: `https://sketchfab.com/3d-models/<slug>-<32桁hex>`
- **Sketchfab モデル ID** 32 桁の 16 進文字列
- **Dropbox 共有 URL** `www.dropbox.com` → `dl.dropboxusercontent.com` に自動変換、`?dl=0` を除去
- **任意の直リンク URL** (`http(s)://…` で始まる)

判定は [index.html:597-606](index.html) `detectUrlType()`、ID 抽出は [index.html:574-595](index.html) `extractModelId()`。

### 3.2 ローカルファイル入力
`.glb .gltf .obj .stl .fbx .zip` を `<input type="file">` から選択。

### 3.3 ZIP 内容
ZIP は自動解凍され、拡張子優先順 `glb → gltf → obj → stl → fbx` で最初に見つかった 1 ファイルを読み込む。`.gltf` の場合、参照する `buffers` と `images` の URI を ZIP 内実体に置換して Blob URL に差し替え ([index.html:825-865](index.html))。

---

## 4. Sketchfab 連携フロー

`downloadAndLoadSketchfabModel()` ([index.html:730-785](index.html))

1. `GET {WORKER}/?modelId=<id>&action=info` — モデル名等のメタ取得
2. `GET {WORKER}/?modelId=<id>&action=download` — ダウンロード URL 取得
   - レスポンスの `gltf.url` を優先、なければ `glb.url`
   - 403 → 「このモデルはダウンロードできません」
3. 取得した URL から ZIP を直接 fetch
4. `loadZipBlob()` に渡して解凍・表示

Cloudflare Worker が Sketchfab API トークンを保持し、CORS とトークン秘匿を担う想定(本リポジトリ外)。

---

## 5. 3D シーン仕様

`init()` ([index.html:316-346](index.html))

| 項目 | 値 |
|---|---|
| 背景色 | `#1a1a1a` (ダーク) ↔ `#e0e0e0` (ライト) 切り替え可 |
| カメラ | Perspective, FOV 75°, near 0.1 / far 1000, 初期位置 `(0, 1.6, 0)` |
| ライト | AmbientLight 0.6 + DirectionalLight 0.8 (位置 5,10,7.5) |
| グリッド | `GridHelper(20, 40)` グレー、表示 ON/OFF 可 |
| レンダラ | WebGLRenderer, `antialias: true`, `xr.enabled: true` |
| モデル初期位置 | `(0, 1.5, -2)` |
| モデル自動スケール | 最大辺が 2 になるように等比 |

---

## 6. VR モード

`vrBtn` クリックで開始 ([index.html:1051-1078](index.html))。

- `navigator.xr.isSessionSupported('immersive-vr')` チェック
- `requestSession('immersive-vr', { optionalFeatures: ['local-floor', 'bounded-floor', 'hand-tracking'] })`
- セッション中は UI (`#ui`) を非表示

### 6.1 コントローラー操作

`updateControllerInput()` ([index.html:449-546](index.html))

**右コントローラー (index 1)**
| 入力 | 動作 |
|---|---|
| スティック Y (axes[3]) | モデル X 軸回転 (速度 0.02, デッドゾーン 0.1) |
| スティック X (axes[2]) | モデル Y 軸回転 |
| ボタン 4 (A) | テクスチャ表示 ON/OFF (単色 Standard マテリアルに切替) |
| ボタン 5 (B) | VR セッション終了 |

**左コントローラー (index 0)**
| 入力 | 動作 |
|---|---|
| スティック X (axes[0]) | モデル X 移動 (速度 0.05) |
| スティック Y (axes[1]) | モデル Z 移動 |
| ボタン 4 (X) | グリッド ON/OFF |
| ボタン 5 (Y) | 背景ダーク/ライト反転 |

**両手グリップ (squeeze)**
- 片手グリップ: モデルをコントローラー移動に追従(移動量 ×2)
- 両手同時グリップ: 両手間の距離比でモデルを拡縮

コントローラーには先端に赤(左)/青(右)の LineBasicMaterial 製ポインタ表示。

---

## 7. UI 構成

`<div id="ui">` (画面上部オーバーレイ)

- タイトル「Direct VR Viewer」+ サブタイトル
- ライブラリ読み込み状況表示 (`#loading`)
- URL 入力欄 + 📋 貼り付けボタン + 📥 読み込みボタン
- ファイル選択入力
- クリアボタン / 「VRモードで表示」ボタン
- 進捗バー (0-100%)
- ステータステキスト
- 操作説明パネル

初期化中は 📋 と 📥 が disabled、`initialize()` 完了後に有効化。

---

## 8. 状態管理 (グローバル変数)

[index.html:208-229](index.html)

- `scene, camera, renderer, model, gridHelper`
- `controllers[]` (2 個)
- `isDarkMode / isTextureVisible / isGridVisible`
- `modelScale / modelPosition / modelRotationX / modelRotationY`
- `isDragging / dragController / previousControllerPosition`
- `vrSession`
- `leftGrip / rightGrip / initialGripDistance / initialScale`
- `downloadedBlobUrls[]` — 生成した Blob URL を保持し、クリア/アンロード時に `URL.revokeObjectURL` で解放

クリア処理 (`clearAll()` [index.html:637-680](index.html)) は geometry/material の `dispose()` を含む完全破棄。

---

## 9. 進捗表示ステージ

`updateProgress()` の呼出タイミング

| % | 内容 |
|---|---|
| 10 | Sketchfab: 情報取得開始 |
| 20 | ダウンロード URL 取得 / 直リンク fetch 開始 |
| 30 | Sketchfab: ZIP ダウンロード開始 |
| 50 | ダウンロード完了 / ファイル読み込み開始 |
| 60 | ZIP 解凍中 |
| 70 | モデル抽出中 |
| 80 | モデル parse 開始 |
| 80→100 | ローダー progress コールバック |
| 0 (1.5s 後) | バー非表示 |

---

## 10. エラー・制限事項

- **VR 未対応環境**: 「VRがサポートされていません」を表示、ビューア機能のみ利用可
- **Sketchfab ダウンロード不可モデル**: 403 で明示エラー
- **CORS**: 任意 URL の直リンクは相手側が CORS 許可していない場合失敗する
- **AR モード非対応**: `immersive-vr` のみ要求
- **アニメーション再生非対応**: GLTF の animations は読み込むが再生ロジック無し
- **モデル差し替え時**: 前モデルの Blob URL は保持されたまま(`clearAll` で明示解放)

---

## 11. デプロイ / 同期

- GitHub リポジトリの `main` ブランチ直下 `index.html` を GitHub Pages が配信
- ローカル更新: `git add index.html && git commit -m "..." && git push` で即反映
- ビルド工程なし
