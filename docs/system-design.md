# RAGシステム 基本設計（v1.1）

## 1. アーキテクチャ概要
```text
[Document Source]
   -> [Ingestion]
   -> [Chunking/Metadata]
   -> [Embedding]
   -> [Vector DB]

[User Query]
   -> [API]
   -> [Retriever]
   -> [Prompt Builder]
   -> [LLM]
   -> [Answer + Citations]
```

## 2. コンポーネント設計
### 2.1 Ingestion Service
- 入力: ファイルパス or アップロード
- 処理: 形式判定、本文抽出、正規化
- 出力: `Document` エンティティ

### 2.2 Indexing Service
- 処理: チャンク分割、埋め込み生成、ベクトルDB登録
- 要件: 差分更新（doc_id単位）

### 2.3 Retrieval Service
- 処理: クエリ埋め込み、Top-K類似検索、任意再ランキング
- 出力: `RetrievedChunk[]`

### 2.4 Generation Service
- 処理: コンテキスト付きプロンプト構築、LLM呼び出し
- 制御: 根拠外推論の抑制、引用整形

### 2.5 API Service
- エンドポイント提供、入力バリデーション、監査ログ出力

## 3. API最小仕様（MVP）
### `POST /v1/index`
- 概要: 文書取り込み/再取り込みを実行
- Request: `{ "source": "./data", "reindex": true }`
- Response: `{ "accepted": 12, "failed": 1 }`

### `POST /v1/query`
- 概要: 質問に対して回答を返す
- Request: `{ "question": "...", "top_k": 5, "filters": {"type": "md"} }`
- Response:
```json
{
  "answer": "...",
  "citations": [
    {"doc_id": "d1", "title": "運用手順", "chunk_id": "c10"}
  ],
  "latency_ms": 1520
}
```

### `GET /v1/health`
- 概要: ヘルスチェック
- Response: `{ "status": "ok" }`

## 4. データモデル
### documents
- `doc_id` (PK)
- `title`
- `source_path`
- `checksum`
- `updated_at`

### chunks
- `chunk_id` (PK)
- `doc_id` (FK)
- `chunk_index`
- `chunk_text`
- `token_count`
- `metadata` (JSON)

### query_logs
- `query_id` (PK)
- `question`
- `retrieved_chunk_ids`
- `answer`
- `latency_ms`
- `created_at`

## 5. 主要パラメータ初期値
- チャンクサイズ: 500 tokens
- オーバーラップ: 100 tokens
- Top-K: 5
- 温度: 0.2（回答の安定性重視）

## 6. エラーハンドリング方針
- 外部LLM障害: リトライ（指数バックオフ最大3回）
- 検索結果ゼロ件: 「根拠不足」メッセージを返す
- 取り込み失敗: 失敗ファイルをログに記録して継続

## 7. ディレクトリ設計（実装時）
- `app/ingestion/`
- `app/indexing/`
- `app/retrieval/`
- `app/generation/`
- `app/api/`
- `tests/`
