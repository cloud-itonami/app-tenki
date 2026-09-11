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

## step 3.5 — wiring check（依存ゼロ、repo ルート）

```bash
cd <repo root>
nbb test/appview_wiring_test.cljk
```

実測: `CHECKED	8` → `appview-wiring: OK`（exit 0）。`npm install` も
shadow-cljs のビルドも要らない——committed なファイルだけを読む。

step 3 の `cljs.test` suite が見られないものを見る。あの suite は
`tenki.app` **1 名前空間の中**を検査するが、ビルド設定・配信される文書・
そこへマウントするコードを**繋いでいる文字列**は 2 つのファイルに重複して
書かれていて、誰も突き合わせていない。片側を変えると:

- `shadow-cljs compile app` は通る（違反した検査が無い）
- `node out/tests.js` も `0 failures, 0 errors` を出す
- そして**画面には何も出ない**

検査している 8 つ（どれも「壊すと赤くなる」ことを実測済み）:

| 不変条件 | 壊れると |
|---|---|
| `:wiring/exactly-one-document` | shell が 2 枚に割れ、片方だけが更新に追随しなくなる |
| `:wiring/asset-path-must-be-relative` | path prefix 配下で bundle が 404（ルート直下では動くので気付けない） |
| `:wiring/bundle-src-mismatch` | module 名を変えると `js/<新名>.js` が出るのに文書は `js/app.js` を要求し続ける |
| `:wiring/output-dir-not-under-document` | 出力先を移すと文書からの相対解決が外れる |
| `:wiring/init-fn-namespace-has-no-source-file` | `:init-fn` が実在しない ns を指す |
| `:wiring/mount-id-absent-from-document` | `getElementById` の id と文書の id がずれ、`render` が null に描く |
| `:wiring/npm-test-does-not-run-shadow-test-output` | `:output-to` を変えると `npm test` は**前のビルドの成果物**を走らせて緑を出す |
| `:wiring/test-ns-not-matched-by-ns-regexp` | `:ns-regexp` に合わない test ファイルは**一度も実行されない**のに suite は緑 |

exit は 3 値。**0 = 全部検査して通った / 1 = 検査して違反があった /
2 = REFUSED（測れなかった）**。2 が要るのは、入力が読めなかった実行と
読んで問題が無かった実行が同じ値を返すと、沈黙が緑として積み上がるため
（実測: shadow-cljs.edn が 2 form になっている / `:ns-regexp` が無い /
`test/` に 1 ファイルも無い、の 3 つはどれも exit 2 を返す）。

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
