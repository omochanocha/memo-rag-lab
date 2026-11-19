# チャンクのベクトルは langchain-postgres を使わず、自前のテーブルに保存する

メモの作成・更新と同時に埋め込みを作り直すため、メモとチャンクの保存が食い違わないことを優先する。チャンクのテーブルは Alembic で管理し、`memo_id` に外部キー（`ON DELETE CASCADE`）を張る。保存と検索は SQLAlchemy と `pgvector` で書き、メモとチャンクを 1 つのトランザクションで保存する。LangChain は埋め込みモデル・text splitter・プロンプト・チャットモデルでは使い続ける。

## Considered Options

- `PGVectorStore`（langchain-postgres）を自前のテーブルの上で使う：独自の接続（`PGEngine`）を持つため SQLAlchemy のセッションと別のトランザクションになり、失敗時の後始末を自分で書く必要がある。
- `PGVectorStore` にテーブルも任せる：上の問題に加え、メモとの関連がメタデータ（JSON）頼みになり、外部キーで守れない。
- 旧 `PGVector`：langchain-postgres 0.0.14 で deprecated。
