> [!WARNING]
> This content was drafted with an AI agent and must be verified by a human developer. After human verification, this alert may be removed.

# design: Web ビューア（graphcal.org/view）の実装方針 — GitHub Pages 版

2026-10-13 作成。`web/view`（描画コア）を graphcal.org に置き、利用者のリポジトリの `.gcl` を閲覧者自身の権限で読んでブラウザ内で評価する「中央ビューア」の作り方をまとめる。VS Code 版（別セッションで進行中）とは描画コアと `Host` インターフェースを共有し、違いは Host 実装 1 つに閉じ込める。

## 1. 決まっていること（2026-10-12 の議論の再掲）

- 利用者の組織は GitHub **Free**。Free は private リポジトリに Pages を作れず、Pages のアクセス制御は Enterprise Cloud 限定。よって「リポジトリごとの Pages」は採らない。
- graphcal.org に**値を一切持たない殻**を 1 つ置く（HTML + JS + WASM）。開いた人が自分の GitHub トークンで `.gcl` を読み、ブラウザ内の WASM で評価する。読めない人には API が 404 を返すので何も出ない。権限判定は GitHub に任せる。
- 利用者側のリポジトリにはワークフローもデプロイも不要。README にリンクを 1 行置くだけ。
- ログインは v1 は PAT 貼り付け、v2 で graphcal-lang 側が運用する小さなトークン交換の中継（Cloudflare Worker 程度）を足す。
- 作業順は ViewModel JSON → 描画コア → VS Code → Web ビューア + ログイン。前の 3 つは 2026-10-13 時点で動いているので、この文書はその 4 つ目。

## 2. URL の規約

```
https://graphcal.org/view/#<owner>/<repo>@<ref>/<path/to/file.gcl>
```

- `ref` はブランチ名、タグ、SHA のいずれか。`@<ref>` を省いたらリポジトリの既定ブランチ。
- `path` はリポジトリルートからのパス。ファイル 1 つ = シート 1 枚（per-file view の決定どおり）。
- フラグメント（`#` 以降）に置くので、GitHub API 以外のどこにも送信されない。Pages 側は `view/index.html` 1 枚で済み、Pages の 404 問題も起きない。
- `#` が無いときは今までどおりフィクスチャの開発シェル（`FixtureHost`）。VS Code 内（`acquireVsCodeApi` がある）なら `VsCodeHost`。この 3 択は `main.ts` の先頭で決める。
- 後回し: PR の head（`@pr/123` → `GET /pulls/123` で head.sha を解決）、what-if 上書きの URL 載せ（§9）。

## 3. 構成: 3 つ目の Host

`web/view/src/host.ts` の `Host` インターフェースはそのままで、`GitHubHost` を足す。シート側のコードは触らない。

| 役割 | 新規ファイル | 中身 |
|---|---|---|
| GitHub REST の薄い層 | `github.ts` | `fetch` + zod。認証ヘッダ、`Accept`、ETag キャッシュ、404 / 401 / 403(rate limit) の型付きエラー |
| URL ↔ 座標 | `route.ts` | `#owner/repo@ref/path` の parse / format。`hashchange` で再読込 |
| リポジトリ → Project | `repoProject.ts` | ファイルからプロジェクトルートを決め、必要なファイル一式を `Project`（`project.ts`）に詰める |
| Host 実装 | `githubHost.ts` | `FixtureHost` と同じ「メモリ上のソースを WASM で再投影」。違いは初期ソースの出どころだけ |
| ログイン UI | `login.ts` | トークン入力ダイアログ、状態表示、忘れるボタン |

既存への小さな手直し（別セッションと衝突しないよう、着手は VS Code 側が落ち着いてから）:

- `project.ts` の `projectFor` はフィクスチャ専用（`import.meta.glob`）なので `fixtureHost.ts` へ移し、`Project` クラスだけを共有にする。
- `engine.ts` の WASM の置き場所を `import.meta.env.VITE_ENGINE_BASE`（既定は今の `BASE_URL + pkg/`）で差し替えられるようにする（§6 の共有用）。
- `index.html` のフィクスチャ選択（`#fixture`）は GitHub モードでは `owner/repo @ ref · path` の表示とログイン状態に置き換わる。

### 3.1 プロジェクトルートとファイル集合

`graphcal` の loader と同じ規則をブラウザで再現する（`crates/graphcal-project/src/loader.rs` の `graphcal.toml` 探索）。

1. 指定ファイルのディレクトリから上に向かって `graphcal.toml` を探す。見つかればそこがルート。
2. 見つからなければファイルのディレクトリをルートにする（単独ファイル、仮想パッケージ）。
3. ルート以下の `graphcal.toml` と `*.gcl` を全部取る。マニフェストの `source_dir`（既定 `src`）の下だけで足りるが、仮想パッケージも扱うのでルート以下すべてにしておく。
4. WASM の `prepareProject({ entry, files })` にルート相対パスで渡す。`entry` は指定ファイル。

Pin / Open（別ファイルをペインで開く）は同じ `Project` のファイル集合から `evaluateView` し直すだけなので、`FixtureHost.request` と同じ形で済む。

## 4. GitHub からの読み出し

REST を `fetch` で直接叩く。`api.github.com` は `Access-Control-Allow-Origin: *` を返す（2026-10-13 に確認）ので Pages の origin から読める。octokit は入れない: 使う呼び出しが 3 つしかなく、zod で型を付ける流儀が `engine.ts` と揃うため。

| 目的 | 呼び出し | 備考 |
|---|---|---|
| 既定ブランチ、見えるかの確認 | `GET /repos/{owner}/{repo}` | 404 = 見えない（private か存在しない）。ここでログインを促す |
| ファイル一覧 | `GET /repos/{owner}/{repo}/git/trees/{ref}?recursive=1` | `.gcl` と `graphcal.toml` だけ残す。`truncated: true` ならディレクトリ単位の `GET /contents/{dir}` に落とす |
| 本文 | `GET /repos/{owner}/{repo}/contents/{path}?ref={ref}`、`Accept: application/vnd.github.raw+json` | 生テキストで返る。1 ファイル 1 リクエスト |

- 認証は `Authorization: Bearer <token>`。無しでも公開リポジトリは読める（60 回/時）。トークンがあれば 5000 回/時。
- ETag をメモリに持ち `If-None-Match` を付ける。304 はレート制限に数えられないので、ref 切替や再読込が安い。
- 制限値（`crates/graphcal-wasm/src/project.rs`）: 1 ファイル 256 KiB、合計 1 MiB、パス 1 KiB、パス合計 16 KiB。超えたら読み込みを止めてメッセージを出す。
- `[dependencies]` やプラグインのあるマニフェストは `graphcal-wasm` が拒否する（Playground と同じ）。これもメッセージにする。

## 5. ログイン

### v1: PAT 貼り付け（この順で作る）

- Fine-grained PAT。権限は **Contents: Read-only**（Metadata は自動で付く）。対象リポジトリは利用者が選ぶ。
- 組織が fine-grained PAT を制限している場合は classic PAT（`repo` スコープ）でも動く。どちらも `Bearer` で同じ。
- 保存先は `localStorage` の 1 キー。「忘れる」ボタンで消す。送る先は `api.github.com` だけ。
- `index.html` に CSP の `<meta>` を置く: `default-src 'self'; connect-src 'self' https://api.github.com; img-src 'self' data:`。外部スクリプトは読まない（今も読んでいない）。
- トークンを URL に載せない。コンソールにも出さない。

### v2: GitHub ログイン（maintainer との相談事項）

ブラウザだけでは OAuth の code → token 交換ができない（client secret が要り、`github.com/login/oauth/access_token` は CORS を返さない）。graphcal-lang 側で 50 行程度の中継を 1 つ運用する。

- 候補は OAuth App か GitHub App。OAuth App はインストール不要だが組織の「第三者アプリ制限」に引っかかりうる。GitHub App は組織に一度インストールしてもらう必要があるが、権限がリポジトリ単位で、`setup-graphcal` の流れにも乗る。**GitHub App を勧める**（トークンは 8 時間で切れるので中継が refresh も受ける）。
- 中継は `code` を受けて secret で交換し、トークンを返すだけ。CORS は `https://graphcal.org` に限定。
- これは graphcal-lang の新しい運用物になるので、着手前に maintainer に相談する。v1 の PAT で機能は全部動くので、v2 を待たずに公開できる。

## 6. ビルドと配信

今の Pages は `docs.yaml` が `just docs-build` で `site/` を組み立て、main への push で deploy する（`build_type: workflow`、CNAME graphcal.org、HTTPS 強制、2026-10-13 に確認）。ここに `view/` を足す。

- `web/view/vite.config.ts` は既に `base: "/view/"`。`vp build` の `dist/` を `site/view/` にコピーする。
- WASM: v1 は `pnpm run engine` と同じ出力を `web/view/public/pkg` に置く（Pages の成果物が 5.8 MB 増える）。v2 で `VITE_ENGINE_BASE=/playground/pkg/` にして Playground の 1 本を共有する。
- justfile に追加: `view-assets`（wasm-pack）、`view-check`（`vp check`）、`view-test`（`vp test`）、`view-build`（`vp build`）。`docs-build` の依存に `view-build` を足す。`lint` / `test` にも `view-check` / `view-test` を足す。
- `internals/site-assemble.mjs`: 入力チェックに `web/view/dist/index.html`、コピー先 `site/view/`。`site-assemble.test.mjs` と `site-verify.mjs` に `view/index.html`、`view/pkg/graphcal_wasm.js`、`view/pkg/graphcal_wasm_bg.wasm` を足す。
- `.github/workflows/docs.yaml` と `browser-e2e.yaml`: `vp -C web/view install --frozen-lockfile` を 1 行足す（`setup-vp` の `cache` は working-directory 単位なので、`web/view` 用にもう 1 つ `setup-vp` を置くか cache を諦める）。
- ドキュメントのナビ（`zensical.toml` の `nav`）に「View」を足すのは、人に見せられる状態になってから。
- feature-ui のまま deploy はされない（main からだけ）。それまでは `just site-serve`（localhost:4173）で `site/view/` を確かめる。

## 7. 画面の追加分

- 上のバー: 左に `owner/repo @ ref`、その右にパス。右端にログイン状態（`未ログイン` / `ログイン済み · 忘れる`）。
- 読み込み中・失敗の見せ方は式バーの note スロット（`Host.onNote`）に流す。文言は 5 種類: 見つからない（404）、トークンが無効（401）、レート制限（403 + `X-RateLimit-Remaining: 0`、解除時刻を出す）、コンパイルエラー、ブラウザ非対応（依存関係 / サイズ超過）。
- 公開リポジトリならログイン無しで開く。404 のときだけログインを促す。
- what-if 編集はこれまでどおりメモリ上。リセットで GitHub から読み直したソースに戻る（ETag があるので安い）。

## 8. 実装の順番（1 段ごとに動かして確かめる）

1. `github.ts` と `route.ts`。vitest で `fetch` を差し替えて、ヘッダ、ETag、エラー分類を確かめる。
2. `repoProject.ts`。ルート探索とファイル集合を、フィクスチャのディレクトリ構造（`power_project`、`power_budget_project`）を GitHub の tree 形式に見立てたテストで確かめる。
3. `githubHost.ts` と `main.ts` の Host 選択。公開リポジトリ（graphcal-lang/graphcal の `tests/fixtures/valid/rocket.gcl` など）をトークン無しで開けるところまで。
4. `login.ts`（PAT）。private リポジトリが開けるところまで。
5. justfile / site-assemble / verify / docs.yaml。`just docs-build` と `just site-serve` で `/view/#...` が動くところまで。
6. v2: ref 切替、PR head、what-if の URL 載せ、GitHub ログイン中継。

## 9. 未決

- what-if 上書きを URL に載せる形式（Playground の share-codec を流用するか、`#...;p_a=12 W` のような可読形式にするか）。
- トークンの保存先を `localStorage` にするか `sessionStorage`（タブを閉じたら消える）にするか。
- WASM を Playground と共有するタイミング（§6 の v2）。
- 中継の運用主体と GitHub App の所有者（graphcal-lang 組織）。
- `web/view` のブラウザテスト（Playwright）を Playground と同じ `browser-e2e.yaml` に載せるか。

## 10. 2026-10-13 に済ませた準備

- `zensical` 0.0.68（CI と同じ版）を `uv tool install` で入れた。`just docs-render` がローカルで通る（英日とも 1.3 秒）。
- `wasm-pack` 0.15.0、`just` 1.58.0 は CI の版と一致。`wasm-bindgen-cli` 0.2.129、`wasm-opt` も入っている。
- `vp -C web/view install --frozen-lockfile` が通る（vp が pnpm 12.9.1 を自前で解決するので、壊れている corepack の pnpm は不要）。
- `web/view/public/pkg` に WASM バンドル（5.8 MB）がある。
- `api.github.com` の CORS ヘッダとレート制限ヘッダ、graphcal-lang/graphcal の Pages 設定、組織プラン（free）を確認した。
- 新しい npm 依存は入れていない。実装に要るのは `fetch`、zod（既存）、WASM（既存）だけ。

## 11. 参考

- 画面モデル: `playground-view-design.md`（同じディレクトリ）
- Figma: https://www.figma.com/design/d1C3mKXXPCFowcxliHcT8U/graphcal （G4、案 3b）
- Playground の公開手順: `graphcal/internals/playground.md`、`graphcal/.github/workflows/docs.yaml`
- Host インターフェース: `graphcal/web/view/src/host.ts`、実装例 `fixtureHost.ts` / `vscodeHost.ts`
