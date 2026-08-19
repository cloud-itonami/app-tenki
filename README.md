# app-tenki

`cloud-itonami/app-tenki` は `tenki.etzhayyim.com` 向け天気アプリの repo である。

**現在この repo に入っているのは Svelte 5 + Vite の appview scaffold 一式だけで、
天気の実装は無い。** 下の「宣言と実体のずれ」を読んでから触ること。

## いま実際に入っているもの

`appview/tenki-weather-component/svelte/` の 1 パッケージ（npm 名 `tenki-ui`）:

| ファイル | 中身 |
|---|---|
| `src/App.svelte` | 見出しを出すだけの scaffold（本文に `Vite entry scaffold after SvelteKit cleanup.` と書かれている） |
| `src/main.ts` | `mount(App, …)` |
| `test/tenki.test.ts` | `expect(true).toBe(true)` の placeholder。**アプリについて何も検査していない** |
| `vite.config.ts` / `vitest.config.ts` / `svelte.config.js` / `tsconfig.json` | ビルド・テスト・型検査の設定 |

手順は **[docs/operator-quickstart.md](docs/operator-quickstart.md)**。4 コマンド
（install / test / build / check）が実測で通る。

## 宣言と実体のずれ（未解消）

この repo は `etzhayyim/root` の `60-apps/etzhayyim-project-tenki` から抽出された
（`migration.edn`）。**メタデータは抽出前の姿を宣言したままで、tree と一致しない。**

| 宣言している場所 | 宣言の内容 | 実体 |
|---|---|---|
| `PROJECT.jsonld` | `"stack": "go"`、routes `/` と `/api/mcp` | Go のファイルは 1 本も無い |
| `appview/tenki-weather-component/kotodama.jsonld` | `component.path: "component.wasm"` | `component.wasm` は存在しない |
| `appview/README.md` | 「UI (`/`) / MCP endpoint (`/api/mcp`) / Open-Meteo 連携」 | どれも実装が無い |
| 旧 `README.md`（本ファイルの前身） | `wasm/tenki-weather-component`、`etzhayyim build`、`kubectl apply -f k8s/http-routes.yaml`、MCP ツール `weather.search_city` / `weather.current` / `weather.forecast_daily` | `wasm/` も `k8s/` も無く、`etzhayyim` コマンドも MCP ツールも無い |
| `README.edn` は `app-tenki`、`PROJECT.jsonld` は `etzhayyim-project-tenki` | — | repo 名としてはどちらも生きている（identity の一本化は未了） |

**旧 README の手順をそのまま踏むことはできない。** 実装を書き足すときは、
このずれをどちら向きに解消するか（実装を足すのか、宣言を落とすのか）を先に決めること。

## 既知の欠け

- **天気機能が無い。** 地名検索・現在天気・予報・MCP は 1 つも実装されていない。
- **テストが placeholder。** `test/tenki.test.ts` は恒真を主張するだけで、
  壊しても赤くならない。実装を書くならここも同時に実質化する。
- **`package-lock.json` が無い。** install は毎回解決し直すので再現しない。
- **`@etzhayyim/design-system` が解決できない。** 抽出時に取り残された
  workspace パッケージで、npm にも公開されていない（`npm view` は 404）。
  これに依存していた Tailwind/PostCSS の設定は、どのソースからも使われて
  いなかったので撤去した（経緯は quickstart の「この repo に加えた変更」）。
  デザインシステムを繋ぐときは、その供給元を用意してから設定を戻す。
