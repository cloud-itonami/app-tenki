# operator quickstart — app-tenki

この repo を初めて触る人が、**何が動いて何が動かないかを自分の手で確かめる**ための
手順。下のコマンドは 2026-08-26 の Svelte → ClojureScript 移行時に実測して全部
通っている。

- 実測環境: macOS (darwin 25.3.0) / node **v26.7.0** / npm **11.19.0** /
  Clojure CLI **1.12.5.1654**
- 作業ディレクトリは全ステップ共通:
  `appview/tenki-weather-component/cljs/`

```bash
cd appview/tenki-weather-component/cljs
```

## step 1 — install

```bash
npm install
```

実測: `added 129 packages, and audited 130 packages in 11s`。
`package-lock.json` はこの install で生成済み・コミット済み（旧 Svelte 版に
あった「install が毎回解決し直す」という既知の欠けはこの移行で解消した）。

## step 2 — build（app）

```bash
npx shadow-cljs compile app
```

実測: `[:app] Build completed. (111 files, 110 compiled, 0 warnings, 28.47s)`。
`public/js/app.js` が出力される（`public/index.html` が相対パス `js/app.js` で
読む——これらのページはパスプレフィックス配下で配信されるため、`asset-path`
は `shadow-cljs.edn` で明示的に相対にしてある）。

このマシンは高負荷で、重いビルドは
`node <superproject root>/scripts/resource-guard.mjs run build -- <command>`
経由にすること（ロック保持中は exit 2 を返す——失敗ではなく「今は空いていない」
という意味なので、60 秒待って再試行する）。

## step 3 — test

```bash
npx shadow-cljs compile test
node out/tests.js
```

実測: `[:test] Build completed. (112 files, 111 compiled, 0 warnings, 10.87s)` →
`Ran 4 tests containing 6 assertions. 0 failures, 0 errors.`

> `test/tenki/app_test.cljs` は re-frame の `:initialize-db` イベントと
> `:heading` / `:message` サブスクリプションを実際に検査する——初期値・上書き・
> db 状態からの読み出しをそれぞれ assert している。旧 Svelte 版の
> `test/tenki.test.ts`（`expect(true).toBe(true)` の恒真 placeholder）とは違い、
> 実装を壊すと赤くなる。

`re-frame: Subscribe was called outside of a reactive context.` という警告が
テスト実行時に出るが、これは `cljs.test` の中で `rf/subscribe` を reagent の
render 外から直接 deref しているために出る re-frame 自身の情報警告であり、
テスト結果（0 failures, 0 errors）には影響しない。

## step 4（任意）— dev サーバで実際に見る

```bash
npx shadow-cljs watch app
```

ブラウザで `public/index.html` を開くと、jp-go-dds（DADS）の見出し 1 行 +
状態を示す段落 1 行が出る（天気は出ない——実装が無いため）。確認したら
`Ctrl-C` で止める。

## 書けない手順（実演済み）

「まだ書けない」と言うために、実際に叩いて失敗を確認したもの:

| コマンド | 結果 |
|---|---|
| 旧 README の `cd 60-apps/etzhayyim-project-tenki/wasm/tenki-weather-component` | そのパスは存在しない（`wasm/` ディレクトリ自体が無い） |
| 旧 README の `etzhayyim build` | `etzhayyim` コマンドは無い |
| 旧 README の `kubectl apply -f k8s/http-routes.yaml` | `k8s/` は存在しない |
| `npm view @etzhayyim/design-system version` | `E404 Not Found`（npm に公開されていない） |

MCP エンドポイント（`POST /api/mcp`）と Open-Meteo 連携は `PROJECT.jsonld` /
`appview/README.md` が宣言しているが**実装が無い**ので、叩く手順を書けない。

## この repo に加えた変更（2026-08-26、Svelte → ClojureScript 移行）

`appview/tenki-weather-component/svelte/`（Svelte 5 + Vite + vitest）を削除し、
`appview/tenki-weather-component/cljs/`（shadow-cljs + reagent 1.2.0 +
re-frame 1.4.3 + jp-go-dds）に置き換えた。ワークスペース標準の UI スタックへの
揃えが目的で、機能は増減していない（見出し 1 行 + 段落 1 行のまま）。

| 変更 | 理由 |
|---|---|
| `.svelte` / `.ts` / `vite.config.ts` / `vitest.config.ts` / `svelte.config.js` 一式を削除 | ワークスペース標準（shadow-cljs + reagent + re-frame + jp-go-dds、CLAUDE.md 記載）に揃える |
| `src/tenki/app.cljs` を新規作成 | re-frame の `reg-event-db` / `reg-sub` + jp-go-dds hiccup の reagent view。`main` が `reagent.dom/render` でマウント |
| `test/tenki/app_test.cljs` を新規作成 | 恒真 placeholder だった旧テストを、event/sub を実際に検査する `cljs.test` に置き換え |
| `public/index.html` を `jp-go-dds.page/->page` で生成 | DADS の vendored CSS を inline した SSR shell。`js/app.js` は相対パス（asset-path が path prefix 配下でも壊れないように） |
