# 実装計画

`LEARNING_PLAN.md`（最初の計画メモ）を grill-with-docs で詰めた結果をまとめた計画。用語は `GLOSSARY.md`、後から理由を知りたくなる判断は `docs/adr/` に置く。

## 決まったこと

### アプリ

- 利用者は自分 1 人で、認証はなし。
- メモの項目はタイトル、本文（Markdown）、作成日時、更新日時の 4 つ。ブログ記事も本文を貼り付けてメモとして登録する。URL からの取り込みはしない。
- チャットは 1 問 1 答で、前の質問の内容は引き継がない。

### RAG

- メモの作成・更新時に、同じリクエストの中でチャンクへの分割と埋め込みの作成を行う。更新時はそのメモのチャンクをすべて作り直す。
- チャンクは Alembic で管理する自前のテーブルに置き、メモと同じトランザクションで保存する。削除は外部キー（`ON DELETE CASCADE`）でチャンクに伝わる（ADR 0001）。
- 根拠は、類似度がしきい値を超えたチャンクを含むメモとする。根拠はシステムが決め、LLM には回答の本文だけを書かせる。
- 根拠が 0 件なら LLM を呼ばず、「メモには見つかりませんでした」と返す。プロンプトでも「コンテキストにないことは答えない」と指示する。
- しきい値と取得件数（top-k）は、ステップ 6 で自分のメモを使って実測して決める。
- すべてのメモを作り直す再インデックス用のコマンドを用意する。

### 技術スタック

| 分類 | 採用するもの |
| --- | --- |
| ツールのバージョン管理 | mise（Python、Node.js、pnpm、uv を `mise.toml` に固定する） |
| よく使うコマンド | mise のタスク（`mise run check` など）。中で `uv run` や `pnpm` を呼ぶ |
| Python | 3.14 |
| パッケージ管理 | uv（パッケージ管理だけに使う。Python 本体は mise が入れる） |
| Web フレームワーク | `fastapi[standard]`、Pydantic |
| 設定 | `pydantic-settings`（`.env` を読む） |
| ORM とドライバ | SQLAlchemy 2.1（同期、2.0 スタイル）、`psycopg[binary]`（psycopg3） |
| マイグレーション | Alembic |
| ベクトル | PostgreSQL（`pgvector/pgvector` イメージ）、`pgvector`（Python） |
| LLM | LangChain v1、`langchain-openai`、`langchain-text-splitters` |
| モデル | チャットは `gpt-6-luna`、埋め込みは `text-embedding-3-small`（1536 次元） |
| テスト | pytest、`TestClient`、Fake のモデルと埋め込み |
| Lint と型チェック | Ruff、mypy |
| フロント | Nuxt 4、Nuxt UI v4、TypeScript、pnpm |
| 実行環境 | Docker Compose（ローカルのみ） |

使わないもの：langchain-postgres（ADR 0001）、SQLModel、LCEL と `create_agent`（RAG は普通の関数として組む）、Pinia（`useState` で足りる範囲にとどめる）、CORS の設定（Nitro のプロキシで転送する）。

計画にないツールやライブラリを作業中に足したくなったら、入れる前に確認する。

### 設計

- バックエンドはルーターとリポジトリの 2 層で始める。ステップ 6 で埋め込みが入るときに `MemoService` を切り出す。
- `MemoRepository` は `Protocol` で定義し、メモリ実装と DB 実装を `Depends` で差し替える。
- RAG は「検索 → 根拠が 0 件なら終了 → プロンプト → チャットモデル」の順に、普通の関数として書く。部品は `ChatPromptTemplate` と `init_chat_model` を使う。
- フロントからは `/api/**` を Nitro の `routeRules` で FastAPI に転送する。SSR とブラウザのどちらからも同じ相対パスで呼べる。
- フロントの型は手で書く。

### テスト

- テスト用の DB は、Compose の PostgreSQL の中に作る別データベース（`memo_test`）を使う。テストごとにトランザクションをロールバックして後片付けする。
- テストでは本物の LLM を呼ばない。チャットは `GenericFakeChatModel`、埋め込みは `DeterministicFakeEmbedding` を使う。本物を使った確認は手動で行う。
- フロントのテストは書かない（発展ステップで扱う）。

### `.env` と API キー

- `.env` は pydantic-settings と Docker Compose だけが読む。mise の `[env]` では読み込まない。
- `.env` は Git に含めない。`.env.example` だけコミットする。

## 進め方

- 各ステップを次の 7 項目で書く：ゴール、新しい用語、自分で書くところ、AI に任せるところ、テスト、完了条件、理解度チェックの問い。
- 新しい用語は、ステップを始める前に Claude が 3 行程度で解説する。下に挙げた用語はその候補で、React / TypeScript / Next.js に対応する概念があれば比べて説明する。
- 理解度チェックは、問いに自分の言葉で答え、Claude が添削する形で行う。
- 各ステップの最後に `docs/learning/step-N.md` へまとめを残す。中身は、理解度チェック（問い・自分の答え・添削）、新しく学んだこと、つまずいたところと解決方法。下書きは Claude が会話をもとに作り、自分で読んで直す。理解度チェックの答えは自分の文をそのまま載せる。
- 「自分で書くところ」のコードは、頼まれない限り Claude は書かない。代わりにヒント、方針、参考ドキュメントを示す。
- 1 ステップずつ確認しながら進める。

## ステップ

### 1. 最小の FastAPI サーバー

- ゴール：mise でツールを入れ、uv で作ったプロジェクトの FastAPI をホスト上で起動する。
- 新しい用語：mise（nvm や Volta をまとめたもの）、uv（pnpm に相当）、`pyproject.toml`（`package.json` に相当）、仮想環境（`node_modules` をプロジェクトごとに分ける仕組み）、ASGI と uvicorn、`TestClient`。
- 自分で書くところ：`GET /health` のエンドポイント、`TestClient` を使ったテスト 1 本。
- AI に任せるところ：`mise.toml`（ツールのバージョンとタスク）、`pyproject.toml`、Ruff と mypy の設定。
- テスト：`/health` が 200 と JSON を返すこと。
- 完了条件：`mise run check`（Ruff と mypy）と `mise run test` が通る。`/docs` で API ドキュメントが見られる。
- 理解度チェックの問い：mise と uv はそれぞれ何を管理しているか。`/docs` はどこから作られているか。

### 2. メモの CRUD を 1 つずつ追加

- ゴール：作成、一覧、更新、削除を 1 機能ずつ足す。データはメモリ上に保存する。
- 新しい用語：Pydantic モデル（zod のスキーマに近い）、パスパラメータとリクエストボディ、ステータスコードと `HTTPException`。
- 自分で書くところ：作成と一覧のエンドポイントとスキーマ、それぞれのテスト。
- AI に任せるところ：更新と削除（2 つ目以降の似たコード）。ただし任せた分にも自分でテストを書く。
- テスト：各エンドポイントの正常系と、存在しない ID で 404 になること。
- 完了条件：4 つの操作が `/docs` から試せて、テストが通る。
- 理解度チェックの問い：リクエスト用とレスポンス用でスキーマを分ける理由は何か。

### 3. リポジトリパターンと依存性注入

- ゴール：保存処理を `MemoRepository` に切り出し、`Depends` で注入する。
- 新しい用語：`Protocol`（TS の `interface` に近い構造的な型）、依存性注入と `Depends`（React の Context で値を渡すのに近い）、`app.dependency_overrides`（テストでのモック差し替え）。
- 自分で書くところ：`MemoRepository` の `Protocol`、メモリ実装、`Depends` による注入、`dependency_overrides` を使うテスト。
- AI に任せるところ：なし。
- テスト：テスト用のリポジトリに差し替えて、ルーターが正しく呼び出すこと。
- 完了条件：ルーターがリポジトリの具体的な実装を import していない。
- 理解度チェックの問い：ルーターの中で直接 `MemoryMemoRepository()` を作ると、何が困るか。

### 4. PostgreSQL を導入する（ここで Docker Compose を入れる）

- ゴール：Compose で PostgreSQL を立て、リポジトリの DB 実装に差し替える。FastAPI はホストで動かしたままでよい。
- 新しい用語：Docker Compose、SQLAlchemy の `Session` と `Mapped`、マイグレーション（Prisma の migrate に近い）、トランザクション、pydantic-settings。
- 自分で書くところ：ORM モデル、リポジトリの DB 実装、セッションを渡す `Depends`、リポジトリのテスト。
- AI に任せるところ：`docker-compose.yml`（DB のみ）、`.env.example`、Alembic の初期設定、設定クラスの雛形、テスト用 DB のフィクスチャの雛形。
- テスト：`memo_test` に対してリポジトリの各メソッドを確かめ、テストごとにロールバックする。
- 完了条件：`mise run db:up` で DB が起動し、`alembic upgrade head` でテーブルができる。サーバーを再起動してもメモが残る。
- 理解度チェックの問い：Pydantic のスキーマと SQLAlchemy のモデルを分けるのはなぜか。テストのロールバックは何を守っているか。

### 5. LangChain で LLM を呼ぶ

- ゴール：`POST /chat` で質問を受け取り、LLM の回答を返す（この時点では RAG なし）。
- 新しい用語：チャットモデルとメッセージ、`ChatPromptTemplate`、`init_chat_model`、Fake のチャットモデル。
- 自分で書くところ：プロンプトとモデルを組み合わせる関数、`/chat` のエンドポイント、チャットモデルを渡す `Depends`、Fake に差し替えるテスト。
- AI に任せるところ：`.env` への API キーの追加と設定の読み込み。
- テスト：`GenericFakeChatModel` に差し替え、決まった回答が返ること。
- 完了条件：テストが API を呼ばずに通る。手動の確認で本物の OpenAI から回答が返る。
- 理解度チェックの問い：LLM を `Depends` で注入すると、テストと差し替えの面で何が楽になるか。

### 6. RAG（メモをもとに答える）

- ゴール：メモの保存時にチャンクと埋め込みを作り、質問に近いチャンクから根拠を決めて回答する。
- 新しい用語：埋め込み、ベクトルと距離（`<=>`）、チャンク分割（text splitter）、しきい値と top-k、サービス層。
- 自分で書くところ：チャンクの ORM モデルとマイグレーション、`MemoService`（分割、埋め込み、メモとチャンクの保存）、検索のクエリ、RAG の関数（検索、0 件で終了、プロンプト、回答）、検索と組み立てを別々に確かめるテスト。
- AI に任せるところ：再インデックス用のコマンドと mise のタスク、動作確認用のサンプルメモ。
- テスト：`DeterministicFakeEmbedding` を使い、保存時にチャンクができること、更新で作り直されること、削除で消えること、根拠が 0 件のときに LLM を呼ばないこと、根拠がメモ単位でまとまること。
- 完了条件：自分のメモ 10〜20 件で質問し、しきい値と top-k を決めて設定に書く。ステップ 4〜5 で登録したメモは再インデックスで検索対象になる。
- 理解度チェックの問い：メモとチャンクを 1 つのトランザクションで保存しないと、どんな状態が起こりうるか。埋め込みモデルを変えたら何をする必要があるか。

### 7. Nuxt でフロントを作る

- ゴール：メモの一覧・作成・編集の画面と、チャットの画面を作る。見た目は Nuxt UI のコンポーネントに任せる。
- 新しい用語：`ref` と `computed`（`useState` と `useMemo` に近い）、`useFetch`、composables（カスタムフックに近い）、ファイルベースルーティング（Next.js の App Router に近い）、Nitro の `routeRules`、SSR。
- 自分で書くところ：ページとコンポーネントのロジック部分、API の型、composables。
- AI に任せるところ：`nuxt.config.ts`（Nuxt UI と `routeRules` のプロキシを含む）、見た目の調整。
- テスト：書かない。
- 完了条件：ブラウザからメモの CRUD とチャットができ、回答と根拠が表示される。
- 理解度チェックの問い：`useFetch` が SSR 中とブラウザで同じ `/api/...` を呼べるのはなぜか。
- 余裕があれば：回答のストリーミング。FastAPI 側は SSE（`StreamingResponse`）で、先に根拠を送ってから回答を少しずつ送る。
- 終わりに：型のずれ（Pydantic と手書きの TS の型が食い違う問題）を体験したかを振り返る。体験していなくても、ここで openapi-typescript による自動生成の話をする。

### 8. すべてを Docker Compose で起動する

- ゴール：FastAPI と Nuxt もコンテナに入れ、`docker compose up` 1 回で全部が動くようにする。
- 新しい用語：Dockerfile とマルチステージビルド、Compose のサービス名による名前解決。
- 自分で書くところ：なし（AI が書いたものを 1 行ずつ説明できるようにする）。
- AI に任せるところ：FastAPI と Nuxt の Dockerfile、`docker-compose.yml` の更新、README の起動手順。
- テスト：コンテナ内で `pytest` が通ること。
- 完了条件：`docker compose up` だけでブラウザから使える。
- 注意：Dockerfile は mise を使わず公式イメージ（`python:3.14-slim` など）を使う。バージョンは `mise.toml` と手で合わせる。
- 理解度チェックの問い：Nitro のプロキシの転送先が、ホストで動かしていたときと変わるのはなぜか。

### 9. 発展（任意）

- 会話履歴を持たせる（LangGraph のチェックポインター）
- LangGraph でエージェント化する（メモの検索をツールとして使わせる）
- メモにタグを付け、タグで絞り込んで検索する
- `with_structured_output` で、LLM に実際に使った根拠を選ばせる
- `PGVectorStore` で同じ機能を作り、ADR 0001 の自前実装と比べる
- openapi-typescript でフロントの型を自動生成する
- SQLAlchemy を async に移行する
- フロントのテスト（Vitest、@nuxt/test-utils）
- デプロイ

## AI を使った学習ルール

`LEARNING_PLAN.md` の「AI を使った学習ルール」をそのまま適用する。
