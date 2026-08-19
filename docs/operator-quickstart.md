# operator quickstart — app-tenki

この repo を初めて触る人が、**何が動いて何が動かないかを自分の手で確かめる**ための
手順。下の 4 コマンドは 2026-08-19 に実測して全部通っている。

- 実測環境: macOS (darwin 25.3.0) / node **v26.3.0** / npm **11.16.0**
- 作業ディレクトリは全ステップ共通:
  `appview/tenki-weather-component/svelte/`

```bash
cd appview/tenki-weather-component/svelte
```

## step 1 — install

```bash
npm install
```

実測: `added 73 packages, and audited 74 packages in 8s` / `found 0 vulnerabilities`。

`npm warn allow-scripts` が `esbuild` と `fsevents` について出るが、**これは通してよい**
—— どちらも postinstall を実行しないまま install が完了し、以降の 3 ステップは
そのまま通る（実測）。`npm approve-scripts` を走らせる必要は無い。

> `package-lock.json` が無いので、この 73 という数は将来ずれる。
> 数が違っても step 2〜4 が通れば問題ない。

## step 2 — test

```bash
npm test
```

実測: `Test Files 1 passed (1)` / `Tests 1 passed (1)`、約 0.4s。

> ⚠ **これが緑でもアプリは検査されていない。** `test/tenki.test.ts` の中身は
> `expect(true).toBe(true)` で、実装を全部消しても緑のままになる。
> この repo に機能を足すときは、この 1 本を実質のあるテストに置き換えること。

## step 3 — build

```bash
npm run build
```

実測: `✓ 109 modules transformed` → `✓ built in 1.08s`、`dist/` に 3 ファイル:

```
dist/index.html                  0.41 kB │ gzip:  0.27 kB
dist/assets/index-C-zwCK5o.css   0.24 kB │ gzip:  0.21 kB
dist/assets/index-Ch2YJEZC.js   28.29 kB │ gzip: 10.88 kB
```

## step 4 — 型・テンプレート検査

```bash
npm run check
```

実測: `COMPLETED 112 FILES 0 ERRORS 0 WARNINGS 0 FILES_WITH_PROBLEMS`。

## step 5（任意）— dev サーバで実際に見る

```bash
npm run dev -- --port 5199
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5199/      # => 200
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5199/src/main.ts  # => 200
```

実測: どちらも `200`。ブラウザで開くと見出し 1 行の scaffold が出る（天気は出ない
—— 実装が無いため）。確認したら `Ctrl-C` で止める。

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

## この repo に加えた変更（2026-08-19）

上の step 3 と step 4 は、**この日まで両方とも赤だった**。抽出時に取り残された
設定が原因で、機能ではないので撤去した。

| 変更 | 理由 |
|---|---|
| `tailwind.config.js` を削除 | 1 行目で `@etzhayyim/design-system/plugin` を import していたが、この package は `package.json` に無く npm にも無い（404）。`content` glob も抽出前の monorepo パス `../../../../../packages/ts/design-system/dist/**` を指していて解決しない |
| `postcss.config.js` を削除 | `tailwindcss` を読み込むためだけの設定。上を消すと役目が無い |
| `package.json` から `tailwindcss` / `autoprefixer` / `postcss` を削除 | 設定を消したので未使用。install が 145 → 73 packages に減った |
| `svelte.config.js` を追加 | `svelte-check` が vite 設定から svelte plugin を見つけられず `No Svelte configuration found in vite config` で赤かった。`vitePreprocess()` を宣言して解消 |

**Tailwind を消してよいと判断した根拠**（推測ではなく計数した）: 追跡対象の
ソース 7 ファイル（`.svelte` / `.ts` / `.html` / `.css`、`node_modules` 除く）を
`@tailwind` / `@apply` / `class=` / `design-system` で検索して **0 件**。
`src/App.svelte` は素の `<style>` で書かれており、Tailwind のユーティリティを
1 つも使っていなかった。

変更前後の実測（同じコマンドを抽出直後の tree でも走らせて比べた）:

| step | 変更前 | 変更後 |
|---|---|---|
| `npm install` | ok（145 packages） | ok（73 packages） |
| `npm test` | 1 passed | 1 passed（同じ） |
| `npm run build` | **FAIL** `Cannot find module '@etzhayyim/design-system/plugin'` | **PASS** |
| `npm run check` | **FAIL** 1 ERROR | **PASS** 0 ERRORS |

デザインシステムを本当に繋ぐときは、`@etzhayyim/design-system` を供給する
workspace を用意した上で、この 4 行を戻すこと。
