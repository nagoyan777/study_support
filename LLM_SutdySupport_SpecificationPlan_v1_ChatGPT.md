# LLM中学受験支援アプリ — 開発用説明ドキュメント（LLM支援・CLI対応）

## 概要
このドキュメントは、Claude Code・Gemini CLI・Cursor・VSCode Copilot などのLLMコーディング支援ツールを用いて、家庭学習支援アプリを開発するための指示書（説明・プロンプト用）です。テキストベースでLLMに与えることで、各ツールが正確に設計方針を理解し、仕様書・コード・設定ファイルを自動生成できるように設計しています。

---

## プロジェクト概要
- **名称**: LLM中学受験支援アプリ（Echo Show対応）
- **目的**: 家庭での中学受験学習（算数・理科・社会）を支援するため、LLMを裏方に活用したクイズ・丸つけ・ノート管理アプリを開発する。
- **対象環境**: 
  - フロント: Web（Streamlit or Next.js） / Echo Show (Alexa Skill + APL)
  - バックエンド: FastAPI / Python / AWS Lambda / DynamoDB / S3
  - モデル: GPT-4/Claude/Gemini APIを併用可能

---

## 開発アーキテクチャの方針
```
[Parent Web App]───┐
                    │ HTTPS
                    ▼
             [FastAPI Backend]───[LLM API (Batch/Scoring/QuizGen)]
                    │
        ┌──────────┴──────────┐
        ▼                       ▼
   [OCR/Worker Pipeline]     [Alexa Skill / APL]
        │                       │
       S3  ←→  DynamoDB (Decks, Attempts, Reports)
```
- LLMはリアルタイム呼び出しではなく、**夜間バッチ（問題生成/採点テンプレート更新）**を基本とする。
- Alexaは画像/選択肢/ヒントの提示に特化し、処理はクラウドで完結。

---

## 開発構成検討
### A. 単一アプリ型
- Streamlit/Next.jsで全機能を統合。
- 利点: 短期間でMVPを構築可能。
- 欠点: Echo連携や夜間バッチの拡張が難。

### B. 連携型（推奨）
- Parent Web + Alexa Skill + Batch Worker + API共通基盤。
- モジュール分離により、LLMやOCRエンジンを柔軟に差し替え可能。

> **結論**: LLM支援で分業実装が容易な **連携型構成** を採用。

---

## モジュール構成一覧
| モジュール | 機能 | 担当AIツール | 主要入力 | 主要出力 |
|-------------|------|--------------|-----------|-----------|
| `ocr_worker` | ノート・画像→問題抽出 | Gemini / Claude Code | PDF, PNG | JSON(Item構造) |
| `scoring_api` | 丸つけ採点補助 | GPT-4 / Claude | 問題+回答 | スコア, コメント |
| `quizgen_batch` | 暗記・計算問題生成 | GPT-4 / Gemini | テキスト教材 | Deck(JSON) |
| `parent_web` | アップロード/承認/レポート | — | OCR結果 | 管理画面, CSV週報 |
| `alexa_skill` | クイズ実施 | — | Deck | APL表示/音声出題 |

---

## LLM支援ツールへの指示テンプレート
### 1. Claude Code / Gemini CLI 共通
```
あなたはこのプロジェクトのAIコーディング支援です。以下の仕様を基に、モジュール単位でコード・設定・API仕様書を生成してください。

[仕様]
- バックエンド: FastAPI (Python)
- フロント: StreamlitまたはNext.js
- データ: DynamoDBまたはSQLite
- 機能: OCR→問題抽出→採点→出題→レポート
- LLM呼び出し: 非同期 / APIキー環境変数
- セキュリティ: 署名URL, トークン認証, ローカル保存禁止

[出力形式]
- ディレクトリ構成案
- APIスキーマ（JSON形式）
- DBテーブル定義（SQLまたはPydantic）
- 各関数のdocstring（入力・出力・処理概要）
```

### 2. Claude Codeに与える追加指示例
```
目的: /items/import のエンドポイントを実装し、OCRジョブの登録と進捗照会を行う。
出力: FastAPIのルーター + OCRタスク実行スクリプト + サンプルcurlリクエスト。
```

### 3. Gemini CLIに与える指示例
```
gemini run --goal "生成AIを使って採点APIのルーブリックテンプレートを生成" --input scoring_prompt.md --output rubric.json
```

---

## ドキュメント出力方針（LLM共有用）
- 各文書を **Markdownプレーンテキスト** で保存し、Claude/Geminiへ直接貼り付けまたはファイルパス指定で渡す。
- セクション単位でプロンプト化可能：
  - `/specs/system_overview.md`
  - `/specs/api_contract.md`
  - `/specs/data_model.md`
  - `/specs/prompts/scoring.md`
  - `/specs/prompts/quizgen.md`

---

## 今後の成果物出力プラン
1. **API契約書（OpenAPI雛形）** — Claudeに自動生成させる
2. **DBスキーマ（ERD/DDL）** — Gemini CLIでSQL/Pydantic生成
3. **プロンプト集** — GPTで採点・生成テンプレート整備
4. **ワイヤーフレーム（APL/Streamlit）** — ClaudeにJSON構造→APL生成を依頼

---

## 今後のタスク（LLM連携開発）
| フェーズ | タスク | 使用ツール |
|-----------|---------|-------------|
| W1 | 設計レビュー・API雛形出力 | Claude Code / Gemini CLI |
| W2 | データモデル定義・LLMプロンプト設計 | GPT / Gemini |
| W3 | OCR・採点・出題連携コード生成 | Claude Code |
| W4 | 統合テスト・デプロイ手順書生成 | Gemini CLI / GPT |

---

この文書をそのままClaude CodeやGemini CLIへ読み込ませることで、設計理解・APIスキーマ・ディレクトリ構成案などを自動出力できます。



---

# 付録A｜API契約書（OpenAPI 3.1 雛形）
> このまま `openapi.json` として保存し、Claude Code / Gemini CLI に読み込ませてエンドポイントのスタブ生成や型定義に利用できます。

```json
{
  "openapi": "3.1.0",
  "info": {
    "title": "LLM中学受験支援API",
    "version": "0.1.0",
    "description": "家庭用 学習支援MVPのAPI。親Web・Alexaスキル・バッチ処理が共有。"
  },
  "servers": [
    { "url": "https://api.example.com", "description": "prod" },
    { "url": "https://staging-api.example.com", "description": "staging" }
  ],
  "security": [{ "bearerAuth": [] }],
  "components": {
    "securitySchemes": {
      "bearerAuth": {
        "type": "http",
        "scheme": "bearer",
        "bearerFormat": "JWT"
      }
    },
    "schemas": {
      "User": {
        "type": "object",
        "properties": {
          "id": {"type": "string"},
          "role": {"type": "string", "enum": ["child","parent","admin"]},
          "locale": {"type": "string"},
          "tz": {"type": "string"},
          "created_at": {"type": "string", "format": "date-time"}
        },
        "required": ["id","role","created_at"]
      },
      "Material": {
        "type": "object",
        "properties": {
          "id": {"type": "string"},
          "title": {"type": "string"},
          "source": {"type": "string", "enum": ["note","summary"]},
          "subject": {"type": "string"},
          "unit": {"type": "string"},
          "owner_id": {"type": "string"}
        },
        "required": ["id","title","source","owner_id"]
      },
      "Item": {
        "type": "object",
        "properties": {
          "id": {"type": "string"},
          "stem": {"type": "string"},
          "media": {"type": "array", "items": {"type": "string", "format": "uri"}},
          "type": {"type": "string", "enum": ["mc","num","short"]},
          "choices": {"type": "array", "items": {"type": "string"}},
          "answer": {"oneOf": [{"type": "string"},{"type": "number"}]},
          "explain": {"type": "string"},
          "tags": {"type": "array", "items": {"type": "string"}},
          "difficulty": {"type": "integer", "minimum": 1, "maximum": 5}
        },
        "required": ["id","stem","type","answer"]
      },
      "Deck": {
        "type": "object",
        "properties": {
          "id": {"type": "string"},
          "name": {"type": "string"},
          "item_ids": {"type": "array", "items": {"type": "string"}},
          "policy": {"type": "string", "enum": ["today","weak"]},
          "owner_id": {"type": "string"}
        },
        "required": ["id","name","item_ids","policy"]
      },
      "Attempt": {
        "type": "object",
        "properties": {
          "id": {"type": "string"},
          "user_id": {"type": "string"},
          "item_id": {"type": "string"},
          "input": {"oneOf": [{"type": "string"},{"type": "number"}]},
          "score": {"type": "number", "enum": [0,0.5,1]},
          "hint_used": {"type": "boolean"},
          "ai_conf": {"type": "number", "minimum": 0, "maximum": 1},
          "human_verified": {"type": "boolean"},
          "ts": {"type": "string", "format": "date-time"}
        },
        "required": ["id","user_id","item_id","input","score","ts"]
      },
      "Session": {
        "type": "object",
        "properties": {
          "id": {"type": "string"},
          "user_id": {"type": "string"},
          "deck_id": {"type": "string"},
          "started_at": {"type": "string", "format": "date-time"},
          "finished_at": {"type": "string", "format": "date-time"}
        },
        "required": ["id","user_id","deck_id","started_at"]
      },
      "Job": {
        "type": "object",
        "properties": {
          "id": {"type": "string"},
          "status": {"type": "string", "enum": ["queued","running","succeeded","failed"]},
          "progress": {"type": "integer", "minimum": 0, "maximum": 100},
          "result": {"type": "object"},
          "error": {"type": "string"}
        },
        "required": ["id","status"]
      },
      "Error": {
        "type": "object",
        "properties": {"message": {"type": "string"}}
      }
    }
  },
  "paths": {
    "/materials/import": {
      "post": {
        "summary": "ノート/PDFを登録しOCRジョブを作成",
        "requestBody": {"required": true, "content": {"multipart/form-data": {"schema": {"type": "object","properties": {"file": {"type": "string","format": "binary"}, "subject": {"type": "string"}, "unit": {"type": "string"}}}}}},
        "responses": {"202": {"description": "Accepted", "content": {"application/json": {"schema": {"type": "object","properties": {"job_id": {"type": "string"}}}}}}, "400": {"description": "Bad Request","content": {"application/json": {"schema": {"$ref": "#/components/schemas/Error"}}}}}
      }
    },
    "/jobs/{id}": {
      "get": {
        "summary": "ジョブ進捗の取得",
        "parameters": [{"name": "id","in": "path","required": true,"schema": {"type": "string"}}],
        "responses": {"200": {"description": "OK","content": {"application/json": {"schema": {"$ref": "#/components/schemas/Job"}}}}, "404": {"description": "Not Found"}}
      }
    },
    "/items/batch": {
      "post": {
        "summary": "抽出済み問題をドラフト登録",
        "requestBody": {"required": true, "content": {"application/json": {"schema": {"type": "array","items": {"$ref": "#/components/schemas/Item"}}}}},
        "responses": {"201": {"description": "Created"}}
      }
    },
    "/items/{id}/approve": {
      "post": {
        "summary": "親/管理者による問題の公開承認",
        "parameters": [{"name":"id","in":"path","required": true,"schema": {"type": "string"}}],
        "responses": {"200": {"description": "OK"}, "403": {"description": "Forbidden"}}
      }
    },
    "/decks/today": {
      "get": {
        "summary": "今日の10問を配信",
        "parameters": [{"name": "policy","in": "query","schema": {"type": "string","enum": ["today","weak"]}}],
        "responses": {"200": {"description": "OK","content": {"application/json": {"schema": {"$ref": "#/components/schemas/Deck"}}}}}
      }
    },
    "/attempts": {
      "post": {
        "summary": "回答送信→AI採点→結果返却",
        "requestBody": {"required": true, "content": {"application/json": {"schema": {"type": "object","properties": {"item_id": {"type": "string"},"input": {"oneOf": [{"type": "string"},{"type": "number"}]},"hint_used": {"type": "boolean"}}, "required": ["item_id","input"]}}}},
        "responses": {"200": {"description": "OK","content": {"application/json": {"schema": {"$ref": "#/components/schemas/Attempt"}}}}, "409": {"description": "Conflict(未承認問題)"}}
      }
    },
    "/reports/weekly": {
      "get": {
        "summary": "週報の取得",
        "responses": {"200": {"description": "OK","content": {"application/json": {"schema": {"type": "object","properties": {"study_minutes": {"type": "integer"}, "by_tag": {"type": "array","items": {"type": "object","properties": {"tag": {"type": "string"},"acc": {"type": "number"}}}}, "recommend": {"type": "array","items": {"type": "string"}}}}}}}
      }
    }
  }
}
```

---

# 付録B｜DBスキーマ（ERD要約 + SQL/Pydantic）

## ERD要約
- **User(1) — (n) Session**, **User(1) — (n) Attempt**
- **Deck(1) — (n) Item(through deck_items)**
- **Material(1) — (n) Item**（抽出元）

## SQL DDL（PostgreSQL想定）
```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  role TEXT CHECK (role IN ('child','parent','admin')) NOT NULL,
  locale TEXT,
  tz TEXT,
  created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);

CREATE TABLE materials (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  source TEXT CHECK (source IN ('note','summary')) NOT NULL,
  subject TEXT,
  unit TEXT,
  owner_id TEXT REFERENCES users(id)
);

CREATE TABLE items (
  id TEXT PRIMARY KEY,
  stem TEXT NOT NULL,
  media JSONB DEFAULT '[]',
  type TEXT CHECK (type IN ('mc','num','short')) NOT NULL,
  choices JSONB,
  answer JSONB NOT NULL,
  explain TEXT,
  tags TEXT[],
  difficulty INT CHECK (difficulty BETWEEN 1 AND 5) DEFAULT 2,
  material_id TEXT REFERENCES materials(id),
  approved BOOLEAN DEFAULT FALSE
);

CREATE TABLE decks (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  policy TEXT CHECK (policy IN ('today','weak')) NOT NULL,
  owner_id TEXT REFERENCES users(id)
);

CREATE TABLE deck_items (
  deck_id TEXT REFERENCES decks(id) ON DELETE CASCADE,
  item_id TEXT REFERENCES items(id) ON DELETE CASCADE,
  position INT,
  PRIMARY KEY (deck_id, item_id)
);

CREATE TABLE attempts (
  id TEXT PRIMARY KEY,
  user_id TEXT REFERENCES users(id),
  item_id TEXT REFERENCES items(id),
  input JSONB NOT NULL,
  score NUMERIC CHECK (score IN (0,0.5,1)) NOT NULL,
  hint_used BOOLEAN DEFAULT FALSE,
  ai_conf NUMERIC,
  human_verified BOOLEAN DEFAULT FALSE,
  ts TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  user_id TEXT REFERENCES users(id),
  deck_id TEXT REFERENCES decks(id),
  started_at TIMESTAMPTZ NOT NULL,
  finished_at TIMESTAMPTZ
);

CREATE INDEX idx_attempts_user_ts ON attempts(user_id, ts);
CREATE INDEX idx_items_tags ON items USING GIN (tags);
```

## Pydanticモデル（抜粋）
```python
from pydantic import BaseModel, Field, HttpUrl
from typing import List, Optional, Union

class Item(BaseModel):
    id: str
    stem: str
    media: List[HttpUrl] = []
    type: str # 'mc' | 'num' | 'short'
    choices: Optional[List[str]] = None
    answer: Union[str, float]
    explain: Optional[str] = None
    tags: List[str] = []
    difficulty: int = Field(ge=1, le=5, default=2)
    material_id: Optional[str] = None
    approved: bool = False

class Attempt(BaseModel):
    id: str
    user_id: str
    item_id: str
    input: Union[str, float]
    score: float # 0 | 0.5 | 1
    hint_used: bool = False
    ai_conf: Optional[float] = None
    human_verified: bool = False
```

---

# 付録C｜プロンプト集（抽出/採点/生成）
> そのまま `.md` に保存し、Claude/Geminiへ投入してください。変数は `{{ }}` を置換。

## C-1 抽出（ノート→Item JSON）
```
あなたは中学受験の問題抽出アシスタントです。以下のノート本文だけを根拠に、問題アイテムをJSON配列で返してください。外部知識の追加は禁止。不明は "UNSURE" とし、該当箇所を `notes` に列挙します。

[出力スキーマ]
Item = { stem, media: [], type: 'mc'|'num'|'short', choices?, answer, explain?, tags[], difficulty }

[入力メタ]
subject={{subject}} unit={{unit}} 最大件数={{n}}

[本文]
{{note_text}}
```

## C-2 採点（丸つけ補助・部分点あり）
```
役割: 記述最小・高信頼の採点補助。模範解答とルーブリックに従い、0/0.5/1 を返す。信頼度<{{threshold}} は human_review=true とし、根拠を簡潔に要約。

[入力]
問題: {{stem}}
解答形式: {{type}}
児童回答: {{student_input}}
模範解答: {{gold_answer}}
ルーブリック: {{rubric}}

[出力(JSON)]
{"score": 0|0.5|1, "ai_conf": 0.0–1.0, "reason": "…", "tag": "割合/比/…", "human_review": true|false}

[禁止]
- 推測での加点/減点
- ルーブリック外の判断
```

## C-3 クイズ生成（暗記・計算）
```
目的: 家庭の自作ノートのみを根拠に、暗記/計算クイズを生成。難度★{{d_min}}–★{{d_max}}、各{{k}}問。

[制約]
- 外部知識を持ち込まない
- 出力はItem配列（explainは100字以内）
- 計算問題は整数解優先、単位を明記

[入力]
単元: {{unit}} / 分野: {{subject}}
ノート本文:
{{note_text}}
```

## C-4 ルーブリック雛形（算数 短文/数値/選択）
```
# ルーブリック: 算数（割合/比）
- 数値: 完全一致=1, 単位欠落=0.5, 誤り=0
- 選択: 正答=1, その他=0
- 短文: キーワード2個(各0.5): 「比の基本」「同倍率」
- 共通: 桁区切り/前後空白は正規化、分数は既約分数へ
```

## C-5 安全弁プロンプト（共通）
```
- 不明は "UNSURE" を返す
- 教材本文の引用は50語以内
- 外部知識の追加は禁止
- 出力に個人情報を含めない
```

---

これで、API契約・DBスキーマ・プロンプト一式が揃いました。LLMツールに順次投入して、スタブ生成→実装→評価に進めてください。

