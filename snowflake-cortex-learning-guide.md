# Snowflake Cortex 実務学習ガイド
### AISQL / Cortex Search / Cortex Analyst / Cortex Agents / マルチエージェント（Agent Toolset）

> 作成日: 2026-08-04
> 出典: Snowflake 公式ドキュメント（docs.snowflake.com）および Snowflake 公式ブログ
> 想定読者: Snowflake を触り始めた段階で、AI/分析活用（Cortex 系）を実務で担当するエンジニア

---

## 0. このドキュメントの使い方

読むだけでは身に付かない構成にしてあります。各章の末尾に **「手を動かすチェックリスト」** があるので、トライアルアカウントで実際に実行しながら進めてください。

| 章 | 内容 | 想定所要 |
|---|---|---|
| 1 | 全体像とレイヤ構造 | 20分 |
| 2 | 事前準備（権限・リージョン） | 30分 |
| 3 | Cortex AISQL（SQL から LLM を呼ぶ） | 1.5時間 |
| 4 | Cortex Search（非構造データ / RAG） | 2時間 |
| 5 | Cortex Analyst（構造化データ / Text-to-SQL） | 3時間 |
| 6 | Cortex Agents（エージェント基盤） | 3時間 |
| 7 | **マルチエージェント（Agent Toolset）** | 2時間 |
| 8 | コスト・運用・監視 | 1時間 |
| 9 | 学習ロードマップ | — |

---

## 1. 全体像 — Cortex は「1つの機能」ではなくレイヤ構造

初学者が最初につまずくのは「Cortex ＝ 何を指すのか」が曖昧な点です。Cortex は Snowflake の AI 機能群の総称であり、**下から積み上がる 4 層**として理解すると整理できます。

```
┌─────────────────────────────────────────────────────────┐
│ L4 UI/体験層                                             │
│    Snowflake Intelligence / Snowflake CoWork / Cortex Code│
│    （エンドユーザーがチャットで使う画面）                  │
├─────────────────────────────────────────────────────────┤
│ L3 オーケストレーション層                                  │
│    Cortex Agents                                         │
│    （Plan → Use tools → Reflect のループを回す）           │
├─────────────────────────────────────────────────────────┤
│ L2 検索・分析エンジン層（＝Agent の「道具」）               │
│    Cortex Analyst（構造化 / Text-to-SQL）                 │
│    Cortex Search（非構造 / ハイブリッド検索・RAG）         │
├─────────────────────────────────────────────────────────┤
│ L1 モデル推論層                                           │
│    Cortex AISQL（AI_COMPLETE, AI_CLASSIFY, AI_EXTRACT …） │
│    OpenAI / Anthropic / Meta / Mistral / DeepSeek のモデル │
└─────────────────────────────────────────────────────────┘
```

### 各レイヤの責務

| レイヤ | 機能名 | 何をするか | 入力 | 出力 |
|---|---|---|---|---|
| L1 | Cortex AISQL | SQL 関数として LLM 推論を実行 | テキスト/画像/音声/ドキュメント | 列の値 |
| L2 | Cortex Analyst | 自然言語 → SQL 生成（Semantic View 前提） | 質問文 | SQL + 実行結果 |
| L2 | Cortex Search | ベクトル＋キーワードのハイブリッド検索 | クエリ文字列 | 関連ドキュメント |
| L3 | Cortex Agents | 質問を分解しツールを選択・実行・再考 | 質問文 + ツール定義 | 最終回答（+ 引用/グラフ） |
| L4 | Snowflake Intelligence / CoWork | 上記をノーコードで使うチャット UI | — | — |

### 実務上の判断軸（重要）

| やりたいこと | 使うべきもの |
|---|---|
| 数百万行のレビューを一括で感情分析したい | **AISQL**（バッチ処理向けに最適化） |
| 社内規程 PDF を検索して回答させたい | **Cortex Search** |
| 「先月の東京の売上は？」に SQL 無しで答えたい | **Cortex Analyst**（＋Semantic View） |
| 構造化＋非構造化をまたいだ複合質問に答えたい | **Cortex Agents** |
| 低レイテンシの対話 | REST API（Complete / Embed / Agents API） |
| 高スループットのバッチ | AISQL の SQL 関数 |

> **設計上の注意**: 公式ドキュメントは AISQL について「多数の入力を処理するバッチ処理に適しており、レイテンシが重要な対話用途では REST API を使う」ことを明記しています。UI から 1 件ずつ AISQL を呼ぶ設計はアンチパターンです。

---

## 2. 事前準備

### 2.1 必要なロール / 権限

| 対象 | 必要な権限 |
|---|---|
| AISQL 関数 | アカウントレベル権限 `USE AI FUNCTIONS` ＋ DB ロール `SNOWFLAKE.CORTEX_USER` または `SNOWFLAKE.AI_FUNCTIONS_USER` |
| Cortex Analyst | `SNOWFLAKE.CORTEX_USER` または `SNOWFLAKE.CORTEX_ANALYST_USER`。加えてステージへの READ、Semantic View 内のテーブルへの SELECT |
| Cortex Search（作成） | `SNOWFLAKE.CORTEX_USER` または `SNOWFLAKE.CORTEX_EMBED_USER`、スキーマへの `CREATE CORTEX SEARCH SERVICE`、ソーステーブルへの SELECT、ウェアハウスへの USAGE |
| Cortex Search（利用） | サービス／DB／スキーマへの USAGE |
| Cortex Agents | `SNOWFLAKE.CORTEX_USER` または `SNOWFLAKE.CORTEX_AGENT_USER`、Agent オブジェクトへの権限、各ツールが使うオブジェクトへの権限 |

```sql
-- 学習用ロールのセットアップ例
USE ROLE ACCOUNTADMIN;

CREATE ROLE IF NOT EXISTS cortex_lab_role;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE cortex_lab_role;
GRANT ROLE cortex_lab_role TO USER <your_user>;

CREATE DATABASE IF NOT EXISTS cortex_lab;
CREATE SCHEMA  IF NOT EXISTS cortex_lab.core;
CREATE WAREHOUSE IF NOT EXISTS cortex_lab_wh WITH WAREHOUSE_SIZE = 'X-SMALL' AUTO_SUSPEND = 60;

GRANT USAGE ON DATABASE cortex_lab TO ROLE cortex_lab_role;
GRANT USAGE ON SCHEMA cortex_lab.core TO ROLE cortex_lab_role;
GRANT USAGE ON WAREHOUSE cortex_lab_wh TO ROLE cortex_lab_role;
GRANT CREATE CORTEX SEARCH SERVICE ON SCHEMA cortex_lab.core TO ROLE cortex_lab_role;
GRANT CREATE SEMANTIC VIEW        ON SCHEMA cortex_lab.core TO ROLE cortex_lab_role;
```

> `CORTEX_USER` はデフォルトで `PUBLIC` に付与されています。本番では PUBLIC から revoke し、専用ロールに絞るのが定石です。

### 2.2 リージョンの壁（実務で必ず踏む）

機能によって利用可能リージョンが異なります。日本リージョン（AWS 東京 / Azure Japan East）で全部が動くとは限りません。

- **Cortex Analyst** がネイティブ提供されるのは AWS ap-northeast-1（東京）、AWS us-east-1 / us-west-2、AWS eu-central-1 / eu-west-1、AWS ap-southeast-2、Azure East US 2、Azure West Europe など。
- 対象外リージョンでも **クロスリージョン推論（Cross-region inference）** を有効にすれば利用可能。ただしリクエストが他リージョンで処理されるため、**データレジデンシー要件があるプロジェクトでは事前にクライアント合意が必要**。
- Cortex Search は上記より広いリージョンで利用可能だが、埋め込みモデルごとに可否が異なる（例: `voyage-multilingual-2` は東京では可、Azure Japan East では不可）。

```sql
-- クロスリージョン推論の設定（ACCOUNTADMIN）
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';  -- または 'AWS_US' 等
```

### ✅ 手を動かすチェックリスト（2章）

- [ ] 学習用 DB / スキーマ / ウェアハウス / ロールを作成した
- [ ] `SELECT SNOWFLAKE.CORTEX.COUNT_TOKENS('snowflake-arctic-embed-m','test');` が通る
- [ ] 自アカウントのリージョンを確認し、Analyst がネイティブ提供かクロスリージョンが必要かを判定した

---

## 3. Cortex AISQL — SQL から LLM を呼ぶ

### 3.1 主要関数

`AI_COMPLETE` を中心に、タスク特化型の関数群が用意されています。

| 関数 | 用途 |
|---|---|
| `AI_COMPLETE` | 汎用の生成タスク。テキスト/画像を入力に補完を生成。迷ったらこれ |
| `AI_CLASSIFY` | ユーザー定義カテゴリへの分類（テキスト/画像） |
| `AI_FILTER` | True/False を返す。`WHERE` や `JOIN … ON` に直接書ける |
| `AI_AGG` | 複数行を集約してプロンプトに基づく洞察を返す（コンテキスト長制限を受けない） |
| `AI_SUMMARIZE_AGG` | 複数行を集約して要約（同上） |
| `AI_EXTRACT` | テキスト/画像/ドキュメントから構造化情報を抽出。多言語対応 |
| `AI_SENTIMENT` | 感情抽出 |
| `AI_EMBED` | 埋め込みベクトル生成 |
| `AI_SIMILARITY` | 2 入力間の類似度 |
| `AI_PARSE_DOCUMENT` | ステージ上のドキュメントから OCR / レイアウト付きテキストを抽出 |
| `AI_TRANSCRIBE` | ステージ上の音声/動画を文字起こし（タイムスタンプ・話者付き） |
| `AI_REDACT` | PII のマスキング |
| `AI_TRANSLATE` | 翻訳 |

補助関数: `TO_FILE`（ステージ上ファイルの参照生成）、`AI_COUNT_TOKENS`（トークン数計算）、`PROMPT`（プロンプトオブジェクト構築）。

> **移行メモ**: 旧 `SNOWFLAKE.CORTEX.COMPLETE` は後方互換のために残っていますが、**2026年末までに非推奨化**予定と明記されています。新規実装は `AI_COMPLETE` を使ってください。

### 3.2 ハンズオン

```sql
USE ROLE cortex_lab_role;
USE SCHEMA cortex_lab.core;
USE WAREHOUSE cortex_lab_wh;

-- サンプルデータ
CREATE OR REPLACE TABLE reviews (
  review_id   NUMBER,
  product     VARCHAR,
  body        VARCHAR
);

INSERT INTO reviews VALUES
  (1,'ルーター','届いた翌日に電源が入らなくなった。交換対応をお願いしたい。'),
  (2,'ルーター','設定が簡単で電波も安定している。価格を考えれば満足。'),
  (3,'モデム'  ,'説明書が分かりにくい。サポートに問い合わせたら丁寧に対応してもらえた。');

-- ① 単純な生成
SELECT AI_COMPLETE('claude-4-sonnet',
  '次のレビューを20字以内で要約: ' || body) AS summary
FROM reviews;

-- ② 分類（カテゴリを明示）
SELECT
  review_id,
  AI_CLASSIFY(body, ['故障・不具合','使い勝手','サポート対応','価格']) AS category
FROM reviews;

-- ③ 感情
SELECT review_id, AI_SENTIMENT(body) AS sentiment FROM reviews;

-- ④ AI_FILTER を WHERE 句に直接埋め込む（SQL ネイティブ統合の真骨頂）
SELECT review_id, body
FROM reviews
WHERE AI_FILTER('このレビューは製品の不具合を報告しているか？: ' || body);

-- ⑤ 複数行をまたいだ集約洞察（コンテキスト長を気にしなくてよい）
SELECT AI_AGG(body, '製品改善のために優先すべき課題を3点、箇条書きで日本語で挙げてください')
FROM reviews;
```

### 3.3 実務上の注意

1. **モデル選択がコストを支配する。** トークン課金であり、出力トークンも課金対象。分類・抽出のような定型処理に大型モデルを使うのは無駄です。
2. **モデルのアカウント制御。** `CORTEX_MODELS_ALLOWLIST` パラメータでアカウント全体の利用可能モデルを制御できます（アカウントレベルのみ設定可）。規制要件がある案件では必須の検討事項。
3. **バッチ前提の設計。** 大量行に対する `AI_*` 関数はウェアハウスサイズと並列度に強く影響されます。まず 100 行で試算 → 全件、という手順を守ること。

### ✅ 手を動かすチェックリスト（3章）

- [ ] 上記 ①〜⑤ を全て実行した
- [ ] `AI_COUNT_TOKENS` で入力トークン数を確認した
- [ ] `SNOWFLAKE.ACCOUNT_USAGE` 系ビューで自分の消費クレジットを確認した

---

## 4. Cortex Search — 非構造データの検索基盤

### 4.1 何をしてくれるか

埋め込み生成・インフラ管理・検索品質チューニング・インデックス更新を意識せずに、テキストデータ上にハイブリッド検索エンジンを構築できるマネージドサービスです。内部では以下を組み合わせています。

- **ベクトル検索**（意味的に近い文書の取得）
- **キーワード検索**（字面が近い文書の取得）
- **セマンティック再ランキング**（結果セットの並び替え）

主な用途は 3 つ。**RAG のリトリーバル層**、**エンタープライズ検索のバックエンド**、そして **Cortex Agents の非構造データ用ツール** です。

### 4.2 ハンズオン

```sql
CREATE OR REPLACE TABLE support_docs (
  doc_id    VARCHAR,
  title     VARCHAR,
  body      VARCHAR,
  category  VARCHAR
);

INSERT INTO support_docs VALUES
  ('D001','返品ポリシー','商品到着後14日以内であれば返品を受け付けます。開封済みの場合は…','policy'),
  ('D002','初期設定ガイド','ルーターの電源を入れ、LEDが緑色に点灯するまで待ちます。次に…','guide'),
  ('D003','保証規定','購入日から1年間、通常使用における故障を無償で修理します。','policy');

-- 検索サービスの作成
CREATE OR REPLACE CORTEX SEARCH SERVICE support_docs_svc
  ON body                                   -- 検索対象列
  ATTRIBUTES category                       -- フィルタに使う列
  WAREHOUSE = cortex_lab_wh
  TARGET_LAG = '1 hour'
  EMBEDDING_MODEL = 'snowflake-arctic-embed-l-v2.0'  -- 日本語なら多言語モデルを選ぶ
  AS (
    SELECT doc_id, title, body, category FROM support_docs
  );

-- 動作確認（SEARCH_PREVIEW）
SELECT PARSE_JSON(
  SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
    'cortex_lab.core.support_docs_svc',
    '{
       "query": "返品したい",
       "columns": ["doc_id","title","body"],
       "filter": {"@eq": {"category": "policy"}},
       "limit": 2
     }'
  )
)['results'] AS results;
```

### 4.3 設計上の重要ポイント

| 論点 | 内容 |
|---|---|
| **埋め込みモデル選定** | デフォルトは `snowflake-arctic-embed-m-v1.5`（**英語専用**）。日本語データでは `snowflake-arctic-embed-l-v2.0`（多言語, 512トークン）または `-8k`（8000トークン）を必ず明示指定する |
| **チャンクサイズ** | 公式推奨は **512トークン以下**。長文コンテキストモデルがあっても、小さいチャンクの方が検索精度・下流 LLM 品質ともに高いという研究結果が示されている。`SPLIT_TEXT_RECURSIVE_CHARACTER` を使う |
| **リフレッシュ** | Dynamic Table と同じ増分リフレッシュ制約に従う。ソースクエリが増分リフレッシュ対象外だとサービス作成に失敗する |
| **PRIMARY KEY** | 指定すると変更行だけを再埋め込みする最適化パスが使える。コスト・レイテンシに直結するので**実務では原則指定** |
| **オーナー権限で実行** | 検索はオーナーズライツで動く。検索サービスへの USAGE を持つロールは、元テーブルへの SELECT が無くても検索結果を見られる。**RBAC 設計時の落とし穴** |
| **AUTO_SUSPEND** | 無操作時にサービングを自動停止できる（最小 1800 秒）。検証環境のコスト削減に有効 |
| **上限** | マテリアライズ結果 400M 行、単一サービス 20 QPS / アカウント全体 140 QPS |

### ✅ 手を動かすチェックリスト（4章）

- [ ] 日本語データで多言語埋め込みモデルを指定してサービスを作成した
- [ ] `SEARCH_PREVIEW` でフィルタ付き検索が返ることを確認した
- [ ] `DESCRIBE CORTEX SEARCH SERVICE` で `INDEXING_STATE` / `SERVING_STATE` を確認した

---

## 5. Cortex Analyst — 構造化データの Text-to-SQL

### 5.1 位置づけ

自然言語のビジネス質問に対して SQL を生成し、Snowflake 上で実行するフルマネージドサービスです。REST API 中心の設計になっており、Streamlit / Slack / Teams / 自社アプリなどに組み込めます。

重要な特徴:

- 実行時に**最適なモデルの組み合わせを自動選択**する（利用者がモデルを直接指定することはできない）
- **顧客データで学習しない**。推論には Semantic View のメタデータ（テーブル名・列名・説明など）のみを使用する
- 生成された SQL は**自アカウントのウェアハウスで実行**され、既存の RBAC がそのまま効く

モデル選択の優先順位は Claude Sonnet 4.6 → Claude Sonnet 4.5 → GPT 4.1 → Arctic Text2SQL R1.5 → Mistral Large 2 と Llama 3.1 70b の組み合わせ、という順で、ロールがアクセスできる最上位モデルが使われます。

### 5.2 Semantic View が精度の 9 割を決める

**ここが Cortex Analyst の本質です。** スキーマ定義だけでは「売上とは何か」「有効顧客とは何か」といったビジネス知識が欠落するため、汎用の Text-to-SQL は精度が出ません。Semantic View がその橋渡しをします。

| 構成要素 | 意味 | 例 |
|---|---|---|
| **Logical table** | ビジネス実体 | orders, customers |
| **Relationship** | 結合パス | orders → customers |
| **Fact** | 行レベルの数量 | 割引後金額 |
| **Dimension** | カテゴリ軸 | 顧客名、受注年 |
| **Metric** | 集計 KPI | 平均受注額 |

> 旧来のステージ上 YAML ファイル（Semantic Model）は後方互換で残っていますが、**新規実装は Semantic View（スキーマレベルのオブジェクト）が推奨**です。RBAC・共有・派生メトリクスなどがネイティブに使えます。

### 5.3 ハンズオン（TPC-H サンプルデータ）

```sql
CREATE OR REPLACE SEMANTIC VIEW cortex_lab.core.sales_sv

  TABLES (
    orders AS SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
      PRIMARY KEY (o_orderkey)
      WITH SYNONYMS ('受注', '注文')
      COMMENT = '受注ヘッダ',
    customers AS SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.CUSTOMER
      PRIMARY KEY (c_custkey)
      COMMENT = '顧客マスタ',
    line_items AS SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.LINEITEM
      PRIMARY KEY (l_orderkey, l_linenumber)
      COMMENT = '受注明細'
  )

  RELATIONSHIPS (
    orders_to_customers AS orders (o_custkey)   REFERENCES customers,
    lines_to_orders     AS line_items (l_orderkey) REFERENCES orders
  )

  FACTS (
    line_items.net_price AS l_extendedprice * (1 - l_discount)
      COMMENT = '割引適用後の明細金額'
  )

  DIMENSIONS (
    customers.customer_name AS customers.c_name
      WITH SYNONYMS = ('顧客名')
      COMMENT = '顧客名',
    orders.order_date AS o_orderdate
      COMMENT = '受注日',
    orders.order_year AS YEAR(o_orderdate)
      COMMENT = '受注年',
    orders.market_segment AS customers.c_mktsegment
      COMMENT = '市場セグメント'
      SAMPLE_VALUES ('BUILDING','AUTOMOBILE','MACHINERY','HOUSEHOLD','FURNITURE')
      IS_ENUM
  )

  METRICS (
    orders.order_count     AS COUNT(o_orderkey)   COMMENT = '受注件数',
    orders.avg_order_value AS AVG(o_totalprice)   COMMENT = '平均受注金額',
    line_items.total_sales AS SUM(line_items.net_price) COMMENT = '売上高（割引後）'
  )

  COMMENT = '売上分析用セマンティックビュー'

  AI_SQL_GENERATION '金額は小数点2桁に丸めること。日付は YYYY-MM-DD 形式で返すこと。'
  AI_QUESTION_CATEGORIZATION '個人情報に関する質問は回答せず、管理者への問い合わせを促すこと。';
```

作成後は SQL から直接検証できます。

```sql
SELECT * FROM SEMANTIC_VIEW(
  cortex_lab.core.sales_sv
  METRICS    line_items.total_sales, orders.order_count
  DIMENSIONS orders.order_year, orders.market_segment
) ORDER BY 1, 2;
```

### 5.4 精度を上げる 4 つのレバー（重要）

| レバー | 効果 | 記述場所 |
|---|---|---|
| `COMMENT` / `WITH SYNONYMS` | 列の意味と言い換えを LLM に伝える。**日本語のシノニムを入れると日本語質問の精度が明確に上がる** | 各要素 |
| `SAMPLE_VALUES` / `IS_ENUM` | 取りうる値を提示。`IS_ENUM` を付けるとその値だけでフィルタするようになる | Dimension / Fact |
| `AI_SQL_GENERATION` / `AI_QUESTION_CATEGORIZATION` | SQL 生成方針・質問の受付ポリシーを自然言語で指示 | View 直下 |
| `AI_VERIFIED_QUERIES` | 「この質問にはこの SQL」という検証済みペアを登録。**最も効果が高い** | View 直下 |

```sql
-- 検証済みクエリの登録例
AI_VERIFIED_QUERIES (
  yearly_sales AS (
    QUESTION '年度ごとの売上高を教えて'
    VERIFIED_AT 1785000000
    ONBOARDING_QUESTION TRUE
    VERIFIED_BY '(STEWARD = data_stewards)'
    SQL 'SELECT o.order_year, SUM(li.net_price) AS total_sales
         FROM orders o JOIN line_items li ON li.l_orderkey = o.o_orderkey
         GROUP BY o.order_year ORDER BY o.order_year'
  )
)
```

### 5.5 REST API と マルチターン

```bash
curl -X POST "$SNOWFLAKE_ACCOUNT_BASE_URL/api/v2/cortex/analyst/message" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PAT" \
  -d '{
    "semantic_view": "cortex_lab.core.sales_sv",
    "messages": [
      {"role":"user","content":[{"type":"text","text":"2023年の市場セグメント別売上は？"}]}
    ]
  }'
```

会話履歴を `messages` にそのまま積むことでフォローアップ質問に対応します（「じゃあ北米は？」→ 直前の文脈を補完して解釈）。

**既知の制限（設計時に必ず考慮）:**

- 前回の SQL の**実行結果**は参照できない（「2番目の商品の売上は？」は答えられない）
- SQL で解けない質問には答えない（「どんな傾向が見える？」のような洞察生成は範囲外）
- ターン数が増えたり意図が頻繁に変わると解釈精度が落ちる → 会話をリセットする UI 設計が必要
- LLM はステートレスなため、**毎ターン全履歴を処理する ＝ ターンが進むほどコストが上がる**

### ✅ 手を動かすチェックリスト（5章）

- [ ] Semantic View を作成し `SEMANTIC_VIEW()` で直接クエリできた
- [ ] `SHOW SEMANTIC METRICS IN <view>` で定義を確認した
- [ ] REST API で自然言語質問を投げ、生成 SQL を確認した
- [ ] シノニム／`SAMPLE_VALUES` を追加して精度が変わることを体感した
- [ ] `CORTEX_ANALYST_USAGE_HISTORY` ビューでメッセージ課金を確認した

---

## 6. Cortex Agents — エージェント基盤

### 6.1 概要

Cortex Agents は、**Snowflake のガバナンス境界内で動くフルマネージドのエージェント基盤**です。オーケストレーションループ・ランタイム・サンドボックスを自前で構築せずに、推論 → 計画 → ツール実行 → 応答生成を行えます。

推論ループは 3 ステップ:

1. **Plan** — 曖昧な質問の明確化、複雑な質問のサブタスク分解、各サブタスクへのツール割り当て
2. **Use tools** — 選択したツールを実行
3. **Reflect and respond** — 結果を評価し、追加質問・別ツール呼び出し・最終回答のいずれかを決定

このループが 1 リクエスト内で必要に応じて繰り返されます。

### 6.2 主要概念

| 概念 | 説明 |
|---|---|
| **Agent** | モデル・ツール・オーケストレーション設定・指示をまとめたスキーマレベルオブジェクト |
| **Tools** | エージェントがデータやシステムに作用する手段 |
| **Orchestration** | Plan → Use tools → Reflect のループ。自然言語の指示で制御する |
| **Thread** | ターンをまたぐ会話コンテキストの永続化。クライアント側で状態管理が不要になる |
| **Run** | `agent:run` API への 1 リクエスト。推論・ツール呼び出し・再考のイベントがストリームされる |

### 6.3 利用できるツール一覧

| ツール | `tool_spec.type` | 説明 |
|---|---|---|
| Cortex Analyst | `cortex_analyst_text_to_sql` | Semantic View 経由の Text-to-SQL |
| Cortex Search | `cortex_search` | 非構造データ検索。フィルタ・取得列・件数などをクエリに応じて動的調整 |
| Analytical search | （Search のオプション） | 大規模文書集合に対する集計・件数・傾向分析。複数インデックスを横断し AISQL を全結果に適用 |
| Code execution | `code_execution` | 隔離サンドボックスで Python 実行 |
| Data to Chart | `data_to_chart` | 他ツールの結果からグラフ生成 |
| Custom tools | `generic` | ストアドプロシージャ / UDF による独自ロジック・外部システム連携 |
| Agent skills | （`skills` セクション） | 指示とスクリプトをパッケージ化した再利用可能な能力 |
| MCP connectors | — | リモート MCP サーバのツール（Jira, Salesforce, 自社アプリ等） |
| **Agent toolsets** | **`agent_toolset`** | **他エージェントのツールを実行時に継承（← 7章で詳述）** |
| Web search | `web_search` | Brave Web Search API による公開 Web 検索。アカウントレベルで有効化が必要 |

### 6.4 ハンズオン — エージェントの作成

```sql
CREATE OR REPLACE AGENT cortex_lab.core.sales_agent
  COMMENT = '売上分析＋社内文書検索エージェント'
  PROFILE = '{"display_name": "売上アシスタント", "color": "blue"}'
  FROM SPECIFICATION
$$
models:
  orchestration: auto        # 推奨。アカウントで利用可能な最高品質モデルが自動選択される

orchestration:
  budget:
    seconds: 60
    tokens: 32000

instructions:
  response: "回答は日本語で、結論を先に述べ、根拠となる数値を必ず添えること。"
  orchestration: |
    売上・件数・金額に関する質問は sales_analyst ツールを使うこと。
    返品・保証・規程に関する質問は policy_search ツールを使うこと。
    数値の比較や推移を示す場合は data_to_chart でグラフ化すること。
  sample_questions:
    - question: "2023年の市場セグメント別売上を教えて"
    - question: "返品はいつまで受け付けていますか？"

tools:
  - tool_spec:
      type: "cortex_analyst_text_to_sql"
      name: "sales_analyst"
      description: "受注・売上・顧客に関する数値をSQLで集計する"
  - tool_spec:
      type: "cortex_search"
      name: "policy_search"
      description: "社内規程・ポリシー文書を検索する"
  - tool_spec:
      type: "data_to_chart"
      name: "data_to_chart"
      description: "データからグラフを生成する"

tool_resources:
  sales_analyst:
    semantic_view: "cortex_lab.core.sales_sv"
    execution_environment:
      type: "warehouse"
      warehouse: "cortex_lab_wh"
      query_timeout: 60
  policy_search:
    name: "cortex_lab.core.support_docs_svc"
    max_results: 5
    title_column: "title"
    id_column: "doc_id"
    columns_and_descriptions:
      body:
        description: "文書の本文"
        type: "string"
        searchable: true
        filterable: false
      category:
        description: "文書区分。policy / guide のいずれか"
        type: "string"
        searchable: false
        filterable: true
$$;
```

確認・更新:

```sql
SHOW AGENTS IN SCHEMA cortex_lab.core;
DESCRIBE AGENT cortex_lab.core.sales_agent;

-- 仕様の更新（LIVE バージョンの差し替え）
ALTER AGENT cortex_lab.core.sales_agent
  MODIFY LIVE VERSION SET SPECIFICATION = $$ ... $$;
```

### 6.5 呼び出し（REST API）

```bash
# ① スレッド作成（会話コンテキストの永続化）
curl -X POST "$SNOWFLAKE_ACCOUNT_BASE_URL/api/v2/cortex/threads" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PAT" \
  -d '{"origin_application": "my_app"}'
# → thread_id が返る

# ② エージェント実行
curl -X POST "$SNOWFLAKE_ACCOUNT_BASE_URL/api/v2/databases/CORTEX_LAB/schemas/CORE/agents/SALES_AGENT:run" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PAT" \
  -d '{
    "thread_id": "<thread_id>",
    "parent_message_id": 0,
    "messages": [
      {"role":"user","content":[{"type":"text","text":"2023年の売上上位セグメントとその推移をグラフで"}]}
    ]
  }'
```

- デフォルトは **SSE（Server-Sent Events）ストリーミング**。単一 JSON が欲しい場合は `"stream": false`
- SQL からは `DATA_AGENT_RUN` 関数でも呼べる（非ストリーミング）。ただし公式は **REST API を推奨**
- デフォルトのタイムアウトは 15 分。`"background": true` で最大 6 時間のバックグラウンド実行が可能（スレッド利用が前提）

### 6.6 制限と運用

- **Streamlit in Snowflake（ウェアハウスランタイム）からは Cortex Agents API を呼べない**。SiS から使う場合は **コンテナランタイム**を選択する必要がある（実務でよく踏む制約）
- Agent は**呼び出しユーザーのデフォルトロール**でセッション権限を判定する
- 監視は `Monitor Cortex Agent requests`、品質改善は `Cortex Agent evaluations`（評価データセットを作り、システムメトリクスの推移を Snowsight で追跡）
- **Versioning UI（パブリックプレビュー）** により、LIVE バージョンを本番に据えたまま候補バージョンを検証し、比較・ロールバック・昇格が YAML 手編集なしで行える

### ✅ 手を動かすチェックリスト（6章）

- [ ] Analyst + Search の 2 ツールを持つエージェントを SQL で作成した
- [ ] Snowsight の Agent Playground で構造化・非構造化の両方の質問を投げた
- [ ] REST API + Thread で 2 ターンの会話を成立させた
- [ ] オーケストレーション指示を変更してツール選択が変わることを確認した

---

## 7. マルチエージェント — Agent Toolset

### 7.1 何が発表されたか（2026年7月時点）

2026年7月21日の Snowflake 公式ブログ「Your Agents Are in Production. Now What?」で、Cortex Agents の運用系機能群が一括で発表されました。マルチエージェントに直結するのは **Agent Toolset** です。

| 機能 | ステータス（発表時点） | 概要 |
|---|---|---|
| **Agent Toolset** | パブリックプレビュー予定 | **他エージェントのツールを自エージェントの仕様から参照し、実行時に継承する** |
| Skills Package | パブリックプレビュー予定 | スキル群を1オブジェクトにまとめ、単一 URI で参照 |
| Tool Search | パブリックプレビュー予定 | 全ツール定義を事前ロードせず、推論時に必要な定義だけを検索・ロード |
| Coding Agent | パブリックプレビュー予定 | CoCo と同じランタイムのコーディングエージェントを自アプリに組み込む |
| Async Agent API | GA 予定 | 長時間ジョブをバックグラウンド実行し、`run_id` で後から結果取得 |
| Code Execution Tool | パブリックプレビュー予定 | サンドボックス Python（3.12 / numpy・pandas 同梱）で PDF・PPT・グラフを生成 |
| Interrupt and Resume | GA 予定 | 実行中エージェントの停止・軌道修正・再開 |
| Partial Access | パブリックプレビュー予定 | 単一エージェントで権限レベルの異なるユーザーを同時にサーブ |
| Versioning UI | パブリックプレビュー | 構成の世代管理・比較・ロールバック・昇格 |

> **注意**: ブログ末尾の「マルチエージェント宣言的オーケストレーションガイド」へのリンク（`.../cortex-agents/multi-agent`）は 2026-08-04 現在 404 です。**正式な仕様は `user-guide/snowflake-cortex/cortex-agents-toolsets` を参照**してください。プレビュー機能のため、自アカウントで利用可能かは Snowsight / `SHOW PARAMETERS` で必ず確認すること。

### 7.2 Agent Toolset の実体 — 「ツール継承」であって「サブエージェント委譲」ではない

**ここが最重要の理解ポイントです。** 名前から「親エージェントが子エージェントに仕事を丸投げする」構成を想像しがちですが、実際の挙動は異なります。

呼び出し側エージェントが `agent_toolset` ツールを持つリクエストを受けたとき、Snowflake は次を実行します。

1. `tool_resources[<tool_name>].agent_name` を読み、参照先エージェントを特定
2. 完全修飾名で解決し、**呼び出しユーザーのロール**で USAGE を認可
3. 参照先エージェントの保存済み仕様から `tools` と `tool_resources` を抽出
4. 抽出したツールを呼び出し側の実効構成に **union（合成）**
5. **展開済みのフラットなツール一覧**をオーケストレーターに渡す。`agent_toolset` エントリ自体は除去される

```
【誤解しているイメージ】              【実際の挙動】
  親Agent                              呼び出し側Agent
    └─▶ 子Agent（自分のLLM・指示で           tools: [local_tool,
          考えて回答を返す）                          (継承)analyst_x,
                                                     (継承)search_y]
                                       ↑ 参照先のモデル・instructions は
                                         継承されない。ツールだけが平坦化される
```

**設計上の含意（実務で効く）:**

- 参照先エージェントの **`instructions`（planning / response）や `models` は継承されない**。ドメイン固有の振る舞いを共有したい場合、Agent Toolset では実現できない
- したがって Agent Toolset の正しい使いどころは **「共通ツールライブラリの一元管理」**。データアクセス層、標準検索連携、共通 MCP 設定などを「ツールキットエージェント」として定義し、複数エージェントから参照する
- 参照先のツールが更新されると、参照している全エージェントに自動的に反映される（＝定義の重複とドリフトを排除できる）

### 7.3 実装方法

#### パターン A: 共通ツールキットエージェントを定義して参照する（推奨）

```sql
-- ① 共通ツールキットエージェント（それ自体は直接使わなくてよい）
CREATE OR REPLACE AGENT cortex_lab.core.toolkit_agent
  COMMENT = '全社共通のデータアクセスツール群'
  FROM SPECIFICATION
$$
tools:
  - tool_spec:
      type: "cortex_analyst_text_to_sql"
      name: "sales_analyst"
      description: "受注・売上・顧客の数値集計"
  - tool_spec:
      type: "cortex_search"
      name: "policy_search"
      description: "社内規程・ポリシー文書の検索"

tool_resources:
  sales_analyst:
    semantic_view: "cortex_lab.core.sales_sv"
    execution_environment:
      type: "warehouse"
      warehouse: "cortex_lab_wh"
  policy_search:
    name: "cortex_lab.core.support_docs_svc"
    max_results: 5
$$;

-- ② 業務エージェントから参照
CREATE OR REPLACE AGENT cortex_lab.core.cs_agent
  COMMENT = 'カスタマーサポート向けエージェント'
  FROM SPECIFICATION
$$
models:
  orchestration: auto

instructions:
  response: "顧客対応担当者向けに、丁寧かつ簡潔な日本語で回答すること。"
  orchestration: "規程に関する質問は必ず検索ツールを使い、根拠文書を引用すること。"

tools:
  - tool_spec:
      type: "generic"
      name: "create_ticket"
      description: "サポートチケットを起票する"
  - tool_spec:
      type: "agent_toolset"
      name: "shared_tools"

tool_resources:
  create_ticket:
    type: "procedure"
    identifier: "CORTEX_LAB.CORE.SP_CREATE_TICKET"
    execution_environment:
      type: "warehouse"
      warehouse: "cortex_lab_wh"
  shared_tools:
    agent_name: CORTEX_LAB.CORE.TOOLKIT_AGENT
$$;
```

REST API 版:

```bash
curl -X POST "$SNOWFLAKE_ACCOUNT_BASE_URL/api/v2/databases/CORTEX_LAB/schemas/CORE/agents" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PAT" \
  -d '{
    "name": "CS_AGENT",
    "spec": {
      "models": { "orchestration": "claude-4-sonnet" },
      "tools": [
        {"tool_spec": {"type": "generic",       "name": "create_ticket"}},
        {"tool_spec": {"type": "agent_toolset", "name": "shared_tools"}}
      ],
      "tool_resources": {
        "shared_tools": { "agent_name": "CORTEX_LAB.CORE.TOOLKIT_AGENT" }
      }
    }
  }'
```

#### パターン B: 複数エージェントを参照する

```yaml
tools:
  - tool_spec:
      type: agent_toolset
      name: analytics_tools
  - tool_spec:
      type: agent_toolset
      name: communication_tools

tool_resources:
  analytics_tools:
    agent_name: CORTEX_LAB.CORE.ANALYTICS_AGENT
  communication_tools:
    agent_name: CORTEX_LAB.CORE.COMMS_AGENT
```

`analytics_tools` が先に展開されます。名前が衝突した場合は**先に定義されたものが勝ちます**。

### 7.4 仕様の詳細（テスト観点になる箇所）

| 項目 | 挙動 |
|---|---|
| **名前衝突** | 呼び出し側エージェント自身の定義が最優先。継承ツールをローカルで上書きできる（参照先を変更せずにオーバーライド可能） |
| **再帰展開** | 参照先がさらに `agent_toolset` を持つ場合、再帰的に展開される |
| **循環検出** | 訪問済み集合を用いた DFS で検出。A → B → A の循環は**実行エラー** |
| **最大ネスト深度** | 5 階層（アカウントパラメータ `CORTEX_AGENT_TOOLSET_MAX_DEPTH` で変更可） |
| **最大参照数** | 1 実行あたり 25 エージェント（`CORTEX_AGENT_TOOLSET_MAX_AGENTS` で変更可） |
| **権限不足 / 参照先不在** | **サイレントにスキップされる。エラーも注釈も出ず、呼び出し側自身のツールだけで実行が成功する** |
| **部分展開なし** | 参照先が解決できない場合、そのエージェントのツールは**全て**落ちる。一部だけ継承されることはない |
| **作成時バリデーション** | 構造のみ検証（`name` があるか、`agent_name` が空でないか）。参照先の解決・認可は行わない |
| **解決タイミング** | **実行時**。参照先はエージェント作成時に存在していなくてよい。作成順序を気にせず権限を後付けできる |

**必要な権限:**

| 権限 | 対象 | 用途 |
|---|---|---|
| USAGE | 呼び出し側エージェント | エージェントの実行 |
| USAGE | 参照先エージェント | ツールセットの展開（無い場合はサイレントスキップ） |
| USAGE | DB / スキーマ | 各エージェントが存在する DB・スキーマへのアクセス |

### 7.5 実務で必ず設計に織り込むべきリスク

> ⚠️ **サイレントスキップが最大の落とし穴です。**
> 参照先が削除された、または権限が revoke された場合、エージェントは**エラーを返さずに、ツールが減った状態で動き続けます**。ユーザーから見ると「昨日まで答えられていた質問に、今日は答えられない（しかし失敗はしていない）」という不可解な挙動になります。

対策として実装すべきこと:

1. **デプロイ時のプリフライトチェック** — CI/CD で参照先エージェントの存在と USAGE 付与を検証する
   ```sql
   SHOW AGENTS LIKE 'TOOLKIT_AGENT' IN SCHEMA CORTEX_LAB.CORE;
   SHOW GRANTS ON AGENT CORTEX_LAB.CORE.TOOLKIT_AGENT;
   ```
2. **カナリアクエリの定期実行** — 継承ツールを必ず使うはずの質問を定期投入し、ツール呼び出しイベントが発生しているかを監視する
3. **評価（Evaluations）の CI 組み込み** — Cortex Agent evaluations でシステムメトリクスの回帰を検知する
4. **参照先の DROP を禁止する運用** — `CREATE OR REPLACE` ではなく `ALTER AGENT ... MODIFY LIVE VERSION` を使い、オブジェクトの同一性を保つ

### 7.6 「真の」マルチエージェント委譲が必要な場合の代替パターン

Agent Toolset はツール共有のための機能であり、**子エージェントに独自の指示・モデルで推論させたい**場合は別のパターンが必要です。

| パターン | 概要 | 使いどころ |
|---|---|---|
| **A. UDF 経由の階層型オーケストレーション** | 子エージェントの `agent:run` を呼ぶ Python UDF を作り、それを親エージェントの Custom tool として登録する。マスターエージェントがドメイン別サブエージェントにルーティングする | 各ドメインで指示・モデル・トーンを変えたい場合。Snowflake の Developer Guides に公式のテンプレートあり |
| **B. MCP 経由** | 子エージェントを MCP サーバとして公開し、親が MCP connector で呼ぶ | Snowflake 外のオーケストレーター（例: Microsoft AI Foundry）と組み合わせる場合 |
| **C. 外部フレームワーク** | LangGraph などの Supervisor アーキテクチャから Cortex Agents を呼ぶ | 複雑な状態遷移・条件分岐が必要な場合 |
| **D. A2A プロトコル** | Cortex Agent を A2A ラッパー経由で公開し、他ベンダーのエージェントと相互運用 | マルチベンダー環境 |

**パターン A の骨格（UDF による子エージェント呼び出し）:**

```sql
-- 1) 外部アクセス設定
CREATE OR REPLACE NETWORK RULE cortex_agent_egress_rule
  MODE = EGRESS TYPE = HOST_PORT
  VALUE_LIST = ('<your_account>.snowflakecomputing.com');

CREATE OR REPLACE SECRET cortex_agent_token
  TYPE = GENERIC_STRING SECRET_STRING = '<PAT>';

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION cortex_agent_eai
  ALLOWED_NETWORK_RULES = (cortex_agent_egress_rule)
  ALLOWED_AUTHENTICATION_SECRETS = (cortex_agent_token)
  ENABLED = TRUE;

-- 2) 子エージェントを呼ぶ UDF
CREATE OR REPLACE FUNCTION ask_finance_agent(user_query VARCHAR)
RETURNS STRING
LANGUAGE PYTHON RUNTIME_VERSION = '3.12'
PACKAGES = ('requests')
EXTERNAL_ACCESS_INTEGRATIONS = (cortex_agent_eai)
SECRETS = ('token' = cortex_agent_token)
HANDLER = 'run_agent'
AS $$
import _snowflake, requests, json

def run_agent(user_query):
    token = _snowflake.get_generic_secret_string('token')
    url = ("https://<your_account>.snowflakecomputing.com"
           "/api/v2/databases/CORTEX_LAB/schemas/CORE/agents/FINANCE_AGENT:run")
    resp = requests.post(
        url,
        headers={"Authorization": f"Bearer {token}",
                 "Content-Type": "application/json"},
        json={"stream": False,
              "messages": [{"role": "user",
                            "content": [{"type": "text", "text": user_query}]}]},
        timeout=300,
    )
    resp.raise_for_status()
    return json.dumps(resp.json(), ensure_ascii=False)
$$;

-- 3) 親エージェントに Custom tool として登録
--    tools: [{tool_spec: {type: "generic", name: "finance_agent", ...}}]
--    tool_resources: {finance_agent: {type: "function",
--                     identifier: "CORTEX_LAB.CORE.ASK_FINANCE_AGENT", ...}}
```

> ⚠️ このパターンでは PAT をシークレットとして保持するため、**呼び出し元ユーザーの RBAC ではなく PAT 所有者の権限で子エージェントが動きます**。行レベルセキュリティを前提とした案件では、Partial Access やマルチテナンシー機能の GA を待つか、テナント値をセッション属性として渡す設計に切り替えてください。

### 7.7 判断フローチャート

```
複数のエージェント構成が必要か？
  │
  ├─ ツール定義の重複を解消したいだけ
  │     → Agent Toolset（agent_toolset）  ★ 最もシンプル・低コスト
  │
  ├─ スキル（指示＋スクリプト）を共有したい
  │     → Skills Package
  │
  ├─ ドメインごとに指示・モデル・トーンを変えたい
  │     → UDF 経由の階層型オーケストレーション（パターン A）
  │
  ├─ Snowflake 外のエージェント／プラットフォームと連携したい
  │     → MCP connector（パターン B）／ A2A（パターン D）
  │
  └─ 複雑な状態遷移・条件分岐・評価ループが必要
        → LangGraph 等の外部フレームワーク（パターン C）
```

### ✅ 手を動かすチェックリスト（7章）

- [ ] 自アカウントで `agent_toolset` が利用可能か確認した（プレビュー状況の確認）
- [ ] ツールキットエージェント＋参照側エージェントの 2 つを作成した
- [ ] 参照側から継承ツールが実際に呼ばれることを Playground で確認した
- [ ] **参照先の USAGE を revoke し、エラーにならずツールが消えることを再現した**（サイレントスキップの体感）
- [ ] 同名ツールをローカル定義してオーバーライドされることを確認した
- [ ] `CORTEX_AGENT_TOOLSET_MAX_DEPTH` / `_MAX_AGENTS` の現在値を確認した

---

## 8. コスト・運用・監視

### 8.1 課金モデル

| 機能 | 課金軸 |
|---|---|
| AISQL | 処理トークン数（入力＋出力）。モデルごとに単価が異なる |
| Cortex Analyst | **処理メッセージ数**（成功=HTTP 200 のみ）。単体利用ではトークン数は課金に影響しないが、**Cortex Agents 経由で呼ばれた場合はトークン数が影響する** |
| Cortex Search | ①ウェアハウス（リフレッシュ）②埋め込みトークン ③サービング（インデックスデータ GB/月）④ストレージ ⑤クラウドサービス |
| Cortex Agents | ①オーケストレーション（トークン）②各ツールのコスト（Analyst=トークン、Search=インデックスサイズ×保持期間、Custom tool=ウェアハウス） |
| 生成 SQL の実行 | 別途ウェアハウスコスト |

> **見落としやすい点**: Cortex Search の**サービングコストはクエリが 0 件でも発生します**（インデックスが常時提供状態のため）。検証環境では `AUTO_SUSPEND` を設定するか、使い終わったサービスを DROP すること。

### 8.2 監視クエリ

```sql
-- Cortex Analyst の利用履歴
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_ANALYST_USAGE_HISTORY
ORDER BY start_time DESC LIMIT 100;

-- Cortex Search の日次消費
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_SEARCH_DAILY_USAGE_HISTORY
ORDER BY usage_date DESC;

-- AI サービス全体のクレジット消費
SELECT service_type, DATE_TRUNC('day', start_time) AS d, SUM(credits_used) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY
WHERE service_type = 'AI_SERVICES'
GROUP BY 1, 2 ORDER BY 2 DESC;
```

### 8.3 ガバナンス設計チェックリスト

- [ ] `CORTEX_USER` を PUBLIC から revoke し、専用ロールに限定したか
- [ ] クロスリージョン推論を有効化する場合、データ越境についてクライアント合意を取ったか
- [ ] Semantic Model YAML をステージに置く場合、**ステージへのアクセス権がテーブルの SELECT 権を迂回しないか**を確認したか
- [ ] Cortex Search がオーナーズライツで動くことを前提に、検索サービス単位の権限設計をしたか
- [ ] Agent の budget（秒数・トークン数）を設定し、暴走コストを抑制したか
- [ ] 評価（Evaluations）データセットを用意し、リリース前の回帰検知を仕組み化したか

---

## 9. 学習ロードマップ（実務投入まで）

| Day | 目標 | 成果物 |
|---|---|---|
| 1 | 環境構築・権限理解・リージョン確認 | 学習用 DB / ロール / WH |
| 2 | AISQL を一通り実行 | 分類・抽出・集約の SQL 集 |
| 3 | Cortex Search サービス構築（日本語データ） | 検索サービス 1 本 |
| 4-5 | Semantic View 設計（自分でモデリングする） | Semantic View 1 本 + 検証済みクエリ 5 件 |
| 6 | Cortex Analyst の REST 呼び出し・精度改善 | 精度改善の Before/After 比較 |
| 7 | Cortex Agents 構築（Analyst + Search） | エージェント 1 体 |
| 8 | Thread / ストリーミング / 監視 | REST クライアント実装 |
| 9 | **Agent Toolset でマルチエージェント構成** | ツールキット + 業務エージェント |
| 10 | 評価・コスト分析・ガバナンス整理 | 設計レビュー資料 |

### 案件投入前に自問すべき 5 つの問い

1. クライアントのデータは **Semantic View に落とせる粒度でモデリングされているか**（できていなければ Analyst の精度は出ない）
2. 非構造データのチャンク設計とリフレッシュ頻度は決まっているか
3. **リージョンとクロスリージョン推論**について、データレジデンシー要件をクリアしているか
4. コストの上限（budget / AUTO_SUSPEND / モデル許可リスト）は設定されているか
5. プレビュー機能（Agent Toolset 等）に依存する設計にする場合、**GA 前の仕様変更リスクをクライアントと合意しているか**

---

## 10. 参考リンク（公式）

| ドキュメント | URL |
|---|---|
| Cortex AI Functions（AISQL） | https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql |
| Cortex Search | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview |
| Cortex Analyst | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst |
| Semantic Views（DDL） | https://docs.snowflake.com/en/user-guide/views-semantic/sql |
| Cortex Agents | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents |
| エージェントの作成・管理 | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-manage |
| **Agent toolsets（マルチエージェント）** | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-toolsets |
| Agent skills | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-skills |
| Code execution tool | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-code-execution-tool |
| Cortex Agents Run API | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-run |
| Cortex Agent evaluations | https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-evaluations |
| 公式ブログ（2026/7/21 新機能発表） | https://www.snowflake.com/en/blog/snowflake-cortex-agents-enterprise-ai-scale/ |
| 階層型マルチエージェントの実装ガイド | https://www.snowflake.com/en/developers/guides/multi-agent-orchestration-snowflake-intelligence/ |

---

> **メンテナンス注記**: Cortex の機能追加ペースは非常に速く、本ドキュメントの内容（特に 7 章のプレビュー機能）は数週間で陳腐化する可能性があります。実装前に必ず公式ドキュメントの最新版と、自アカウントでの実際の利用可否を確認してください。
