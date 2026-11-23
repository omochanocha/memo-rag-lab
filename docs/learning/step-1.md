# ステップ 1：最小の FastAPI サーバー

mise と uv で環境を作り、`GET /health` とそのテストを書いた。`mise run check` と `mise run test` が通り、`/docs` で API ドキュメントを確認できた。

## 理解度チェック

### Q1. mise と uv は、それぞれ何を管理しているか

**自分の答え**

> miseは特定の言語に関係なくどんな言語をインストールするかを決めるツール。uvはpython関係のどんなツールを入れるかを決めるパッケージマネージャ。

**添削**

- mise は「何を入れるか」に加えて「どのバージョンを使うか」をプロジェクトごとに固定する（`mise.toml` の `python = "3.14.10"` など）。`mise run check` のようなタスクも管理する。
- uv が入れるのは、アプリが使うライブラリ（fastapi、pytest など）。`pyproject.toml` に書いた依存を `uv.lock` で固定し、`.venv` にインストールする。`uv run` は `.venv` の中でコマンドを実行する。
- 関係：mise が Python 本体と uv を入れ、uv がライブラリを `.venv` に入れる。JS/TS では mise が Volta と npm scripts、uv が pnpm と `pnpm-lock.yaml` に当たる。

### Q2. `/docs` の API ドキュメントは、どこから作られているか

**自分の答え**

> fast APIが裏側でメインファイルのデコレータや関数名から作っている。

**添削**

- 材料には型ヒント（`-> dict[str, str]` など）も含まれる。リクエストやレスポンスの形は型ヒントから作られる。
- 対象は、メインファイルではなく `app = FastAPI()` のインスタンスに登録されたルートすべて。`APIRouter` で別ファイルに分けても、`include_router` で取り込めば仕様書に載る。
- 流れは 2 段階。FastAPI が仕様書を `/openapi.json` で返し、`/docs` の Swagger UI がそれを読み込んで描画する。

### Q3. モジュールとパッケージの違い。`app/__init__.py` で mypy のエラーが直ったのはなぜか

**自分の答え**

> モジュールは一つのファイルのイメージ。パッケージは一つのディレクトリのイメージ。あるディレクトリに__init__.pyを置くとそのディレクトリをパッケージとして置くという規約になるので名前解決で一意に決まりエラーが直った。

**添削**

- 「パッケージとして置く」は「パッケージとして扱う」が正確。
- Python の import は相対パスではなく、起点ディレクトリ（`sys.path`）からの名前で探す。`__init__.py` があると、mypy は `app/` の 1 つ上の `backend/` を起点と判断できる。その結果、`main` と `app.main` の 2 つあった候補が `app.main` に絞られた。

## 新しく学んだこと

### OpenAPI と `/docs`

- FastAPI は、デコレータ（パスと HTTP メソッド）、型ヒント、関数名、docstring から OpenAPI の仕様書を作る。
- 仕様書は `/openapi.json` で返される。`/docs`（Swagger UI）と `/redoc`（ReDoc）は、同じ JSON を別の見た目で表示している。
- 関数名は `/docs` の表示名に使われる。そのため `root` ではなく `health` のように、中身が分かる名前にしておく。

### ルーティングの構成

- `@app.get(...)` の `app` はディレクトリの `app/` ではなく、`app = FastAPI()` で作ったインスタンス。デコレータで、このオブジェクトにルートを登録している。
- ルートが増えたら `APIRouter` で別ファイルに分け、`main.py` で `app.include_router(router)` を呼んで取り込む。Express の `express.Router()` と `app.use()` に近い。
- `main.py` は、ルートの中身を全部書くファイルではなく、`app` を作って各ルーターを取りまとめる入口。
- 1 ファイルにまとめるのは Python 全体の慣習ではない。小さいうちは `main.py` 1 つで十分で、大きくなったら分けるのが FastAPI の定番（公式ドキュメントの「Bigger Applications - Multiple Files」）。

### Laravel との比較

| | Laravel | FastAPI |
|---|---|---|
| ルート定義 | `routes/web.php` にまとめて書く | 処理を書く関数の真上にデコレータで書く |
| 処理本体 | Controller（別ファイル） | デコレータを付けた関数そのもの |
| ルートファイルの登録 | `bootstrap/app.php` で `web.php` や `api.php` を読み込む | `main.py` で `app.include_router(...)` を呼ぶ |

FastAPI は、`web.php` に Closure で処理を直接書くスタイルに近い。

Laravel のルートグループ（`Route::prefix()->middleware()->group()`）に当たるのは、`APIRouter` の引数。

```python
router = APIRouter(
    prefix="/memos",                    # Route::prefix
    tags=["memos"],                     # /docs での見出し（Laravel にはない）
    dependencies=[Depends(認証の関数)],  # ->middleware に近い
)
```

- `prefix` は、`include_router(router, prefix="/memos")` のように取り込む側で指定することもできる。
- `router.include_router(別のrouter)` で、グループを入れ子にできる。
- `app.add_middleware(...)` のミドルウェアはアプリ全体にかかる ASGI ミドルウェアで、グループ単位には付けられない。グループ単位の前処理は `dependencies` で書く（`Depends` はステップ 3 で学ぶ）。

### テストの書き方

- pytest は、`test_*.py` ファイルの中にある `test_` で始まる関数を実行する。ファイル直下に書いた `assert` はテストとして扱われない。
- `TestClient(app)` を使うと、サーバーを起動せずにリクエストを送れる。
- `response.status_code` でステータスコード、`response.json()` で本文を dict として取り出せる。
- `response.json()` は、本文が JSON でなければ例外を投げる。そのため dict と比べる 1 行で「JSON であること」と「中身が正しいこと」の両方を確認できる。

### モジュールとパッケージ

- モジュールは `.py` ファイル 1 つ、パッケージは `__init__.py` を置いたディレクトリ。パッケージ内のモジュールは `app.main` のようにドットでつないだ名前で呼ぶ。
- JS/TS の import は相対パス（`../app/main`）で探すが、Python は起点ディレクトリからの名前（`app.main`）で探す。このプロジェクトでは pytest の `pythonpath = ["."]` 設定で `backend/` が起点になっている。
- 「パッケージ」にはもう 1 つ、npm パッケージに当たる配布パッケージの意味がある。`pyproject.toml` の `package = false` はこちらの意味で、このプロジェクトを配布用にビルドしないという設定。

## つまずいたところと解決方法

| 症状 | 原因 | 解決方法 |
|---|---|---|
| テストが常に失敗する | `response.json` に `()` がなく、メソッドそのものと dict を比べていた | `response.json()` と呼び出す |
| `mise run check` で mypy が `Source file found twice under different module names: "main" and "app.main"` を出す | `app/` に `__init__.py` がなく、mypy が同じファイルを `main` と `app.main` の 2 通りに名付けた | `app/__init__.py`（空ファイル）を置き、`app/` をパッケージにした |
| `from main import app` に書き換えると IDE で赤線が出る | 起点の `backend/` の直下に `main.py` はない（`app/` の中にある） | `from app.main import app` に戻した |
