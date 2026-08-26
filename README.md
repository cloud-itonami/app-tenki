# app-tenki

`cloud-itonami/app-tenki` は `tenki.etzhayyim.com` 向け天気アプリの repo である。

**現在この repo に入っているのは ClojureScript（shadow-cljs + reagent + re-frame +
jp-go-dds）の appview scaffold 一式だけで、天気の実装は無い。** 下の「宣言と実体の
ずれ」を読んでから触ること。

## いま実際に入っているもの

`appview/tenki-weather-component/cljs/` の 1 パッケージ:

| ファイル | 中身 |
|---|---|
| `src/tenki/app.cljs` | reagent + re-frame の scaffold（`:initialize-db` / `:heading` `:message` sub + jp-go-dds hiccup 1 見出し + 1 段落、`main` が `reagent.dom/render` でマウント） |
| `test/tenki/app_test.cljs` | `cljs.test` の実質テスト（event/sub の初期化・上書き・再読込を検査） |
| `deps.edn` / `shadow-cljs.edn` / `package.json` | ビルド・テスト設定（`:cljs` alias に reagent/re-frame/clojurescript/shadow-cljs、`:test` alias にテストパス） |
| `public/index.html` | `jp-go-dds.page/->page` で生成した SSR shell（DADS の vendored CSS を inline、`js/app.js` を relative path で読む） |

2026-08-26 に旧 `appview/tenki-weather-component/svelte/`（Svelte 5 + Vite）から
移行した。旧スタックの見出しテキスト
「Vite entry scaffold after SvelteKit cleanup.」は、ビルドチェーンの記述として
もう正しくないので「ClojureScript entry scaffold after Svelte migration.」に
更新した——機能はどちらも変わらず、見出し 1 行 + 状態を示す段落 1 行だけ。

手順は **[docs/operator-quickstart.md](docs/operator-quickstart.md)**。

## 宣言と実体のずれ（未解消）

この repo は `etzhayyim/root` の `60-apps/etzhayyim-project-tenki` から抽出された
（`migration.edn`）。**メタデータは抽出前の姿を宣言したままで、tree と一致しない。**
cljs 移行はこのずれを解消していない——scaffold の実装言語が変わっただけ。

| 宣言している場所 | 宣言の内容 | 実体 |
|---|---|---|
| `PROJECT.jsonld` | `"stack": "go"`、routes `/` と `/api/mcp` | Go のファイルは 1 本も無い |
| `appview/tenki-weather-component/kotodama.jsonld` | `component.path: "component.wasm"` | `component.wasm` は存在しない |
| `appview/README.md` | 「UI (`/`) / MCP endpoint (`/api/mcp`) / Open-Meteo 連携」 | どれも実装が無い |
| 旧 `README.md`（本ファイルの前身の前身） | `wasm/tenki-weather-component`、`etzhayyim build`、`kubectl apply -f k8s/http-routes.yaml`、MCP ツール `weather.search_city` / `weather.current` / `weather.forecast_daily` | `wasm/` も `k8s/` も無く、`etzhayyim` コマンドも MCP ツールも無い |
| `README.edn` は `app-tenki`、`PROJECT.jsonld` は `etzhayyim-project-tenki` | — | repo 名としてはどちらも生きている（identity の一本化は未了） |

**旧 README の手順をそのまま踏むことはできない。** 実装を書き足すときは、
このずれをどちら向きに解消するか（実装を足すのか、宣言を落とすのか）を先に決めること。

## 既知の欠け

- **天気機能が無い。** 地名検索・現在天気・予報・MCP は 1 つも実装されていない。
- **`@etzhayyim/design-system` が解決できない。** 抽出時に取り残された
  workspace パッケージで、npm にも公開されていない（`npm view` は 404）。
  UI はこのワークスペースの基本 design system（`jp-go-dds`、デジタル庁デザイン
  システム）に置き換えたので、この未解決依存は Svelte 側の設定ごと撤去済み。
  独自デザインシステムを繋ぐ場合は、その供給元を用意してから検討すること。
