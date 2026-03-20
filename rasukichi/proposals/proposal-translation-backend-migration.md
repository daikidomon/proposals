# 提案書：翻訳機能のバックエンド移行 — LLM API活用による非同期翻訳基盤の構築

| 項目 | 内容 |
|------|------|
| 作成日 | 2026年3月20日 |
| 対象Issue | [#46](https://github.com/rasukichisama/rasukichi-system/issues/46) |
| ステータス | Phase 1完了 → Phase 2実装計画 |

---

## 1. エグゼクティブサマリー

現在フロントエンド（クライアントサイド）で実行しているAWS Translateによる商品タイトル翻訳処理を、Next.js API Routeベースのバックエンドに移行し、OpenAI GPT-5 Nano（Responses API）を活用した翻訳基盤を構築する。

### 期待効果

| 項目 | 現状 | 移行後 | 改善 |
|------|------|--------|------|
| 月間翻訳コスト | $916 | 約$11 | **98.8%削減（年間約$10,860削減）** |
| セキュリティ | AWSクレデンシャルがブラウザに露出 | サーバーサイドで秘匿 | **脆弱性解消** |
| 翻訳品質 | AWS Translate（機械翻訳） | LLM（文脈理解翻訳） | **品質向上** |
| 大量処理の信頼性 | クライアント側Promise.all | サーバー側キュー＋キャッシュ | **安定性向上** |

---

## 2. 現状分析

### 2.1 現在のアーキテクチャ

```
ブラウザ（クライアントサイド）
  ├── utils/csv-export.ts : translateTitles()
  │     └── AWS SDK → AWS Translate API（ja → en）
  ├── components/import_history/api.ts : startEbayListingRequest()
  │     └── translateTitles() を呼び出し
  └── app/item_manage/import_history/[id]/page.tsx
        └── translateTitles() を呼び出し
```

### 2.2 現在の翻訳パイプライン

```
日本語タイトル（入力）
  → preprocessTitle()    … #, <, > を除去
  → AWS TranslateTextCommand（ja → en）
  → postprocessTitle()   … 現在はno-op
  → removeMultibyteChars() … 非ASCII文字の除去
  → limitTitleLength()   … 80文字に制限
  → CSV出力 / DB保存
```

### 2.3 翻訳呼び出し箇所（6箇所）

| # | ファイル | 関数 | 用途 |
|---|---------|------|------|
| 1 | `utils/csv-export.ts` L428 | `convertToEbayCSV` | eBay CSV出力時の翻訳 |
| 2 | `utils/csv-export.ts` L564 | `generateCSVData` | CSV生成時の翻訳 |
| 3 | `utils/csv-export.ts` L718 | `generateEbayCSVWithSKU` | SKU付きCSV生成時の翻訳 |
| 4 | `utils/csv-export.ts` L901 | `saveEbayOfferItems` | eBayオファー保存時の翻訳 |
| 5 | `components/import_history/api.ts` L969 | `startEbayListingRequest` | eBay出品リクエスト時 |
| 6 | `components/import_history/api.ts` L1358 | `startEbayListingRequest`（sync版） | eBay出品リクエスト時 |

> `app/item_manage/import_history/[id]/page.tsx` L102 のローカル関数 `generateEbayCSVData` も同等の翻訳処理を含む。

### 2.4 現在の問題点

1. **セキュリティリスク**: `NEXT_PUBLIC_AWS_ACCESS_KEY_ID` / `NEXT_PUBLIC_AWS_SECRET_ACCESS_KEY` がブラウザに露出
2. **高コスト**: 月間約$916（年間約$11,000）
3. **スケーラビリティ**: クライアント側 `Promise.all` による全件並列翻訳はブラウザリソースに依存
4. **キャッシュなし**: 同一タイトルの再翻訳が発生
5. **レートリミット管理なし**: AWS Translateのスロットリング制御が未実装

---

## 3. Phase 1 調査結果（完了）

### 3.1 コスト比較（1000件実測ベース）

| モデル | 方式 | 1000件費用 | 月間推定 | AWS比削減率 |
|--------|------|-----------|---------|------------|
| AWS Translate（現状） | 直接API | $0.677 | $916 | — |
| **GPT-5 Nano (minimal)** | **バッチ10件** | **$0.011** | **~$11** | **98.8%** |
| GPT-4o-mini | バッチ10件 | $0.019 | ~$19 | 97.9% |
| GPT-5 Mini (minimal) | バッチ10件 | $0.057 | ~$56 | 93.9% |

### 3.2 品質比較

| モデル | エラー率 | 品質評価 |
|--------|---------|---------|
| AWS Translate | 0% | 安定（ベースライン） |
| GPT-5 Nano (minimal) バッチ | 2%（JSONパースエラー） | 良好（フォールバックで対応可） |
| GPT-4o-mini バッチ | 0% | 最も高品質 |

### 3.3 モデル選定結果

**第一候補: GPT-5 Nano（Responses API + `reasoning: {"effort": "minimal"}`、バッチ10件）**

- 最安コスト（$0.05/M input + $0.40/M output）
- 十分な翻訳品質
- Responses API使用が必須（Chat Completions APIでは推論トークン抑制不可）

**フォールバック構成:**
```
GPT-5 Nano (minimal, バッチ10件)
  → GPT-5 Nano (minimal, 個別) ※バッチJSONパースエラー時
    → AWS Translate ※最終手段
```

---

## 4. Phase 2 実装計画

### 4.1 全体アーキテクチャ（移行後）

```
クライアント（ブラウザ）
  │
  │  POST /api/translate
  │  Body: { titles: string[] }
  ▼
Next.js API Route（サーバーサイド）
  ├── キャッシュ検索（Supabase: mst_translation_cache）
  │     ├── ヒット → キャッシュ結果を返却
  │     └── ミス → 翻訳実行
  ├── 翻訳プロバイダー（多段フォールバック）
  │     ├── GPT-5 Nano (Responses API, minimal, バッチ10件)
  │     ├── GPT-5 Nano (Responses API, minimal, 個別) ※バッチ失敗時
  │     └── AWS Translate ※最終フォールバック
  ├── 後処理（removeMultibyteChars, limitTitleLength）
  ├── キャッシュ保存
  └── レスポンス返却
```

### 4.2 実装ステップ

#### Step 1: 翻訳APIエンドポイントの作成

**新規ファイル: `app/api/translate/route.ts`**

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/utils/supabase/server';
import OpenAI from 'openai';
import {
  TranslateClient,
  TranslateTextCommand,
} from '@aws-sdk/client-translate';

// OpenAIクライアント（サーバーサイドのみ）
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY, // NEXT_PUBLIC_ なし
});

// AWS Translateクライアント（フォールバック用）
const translateClient = new TranslateClient({
  region: process.env.AWS_REGION || 'ap-northeast-1',
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID || '',
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY || '',
  },
});

const BATCH_SIZE = 10;

export async function POST(request: NextRequest) {
  try {
    const { titles } = await request.json();

    if (!Array.isArray(titles) || titles.length === 0) {
      return NextResponse.json(
        { error: 'titles must be a non-empty array of strings' },
        { status: 400 }
      );
    }

    // 1. 前処理
    const preprocessed = titles.map(preprocessTitle);

    // 2. キャッシュ検索
    const { cached, uncached } = await checkCache(preprocessed);

    // 3. 未キャッシュ分を翻訳
    const translated = await translateWithFallback(uncached);

    // 4. キャッシュ保存
    await saveCache(uncached, translated);

    // 5. 結果をマージ（キャッシュ + 新規翻訳）
    const results = mergeResults(preprocessed, cached, uncached, translated);

    // 6. 後処理
    const processed = results.map((title) =>
      limitTitleLength(removeMultibyteChars(title))
    );

    return NextResponse.json({
      success: true,
      translations: processed,
    });
  } catch (error) {
    console.error('Translation API error:', error);
    return NextResponse.json(
      { error: 'Translation failed' },
      { status: 500 }
    );
  }
}
```

**主要関数の実装概要:**

```typescript
// GPT-5 Nano バッチ翻訳（Responses API）
async function translateBatchWithGPT(
  titles: string[]
): Promise<string[] | null> {
  const prompt = `Translate the following Japanese product titles to English.
Return a JSON array of translated titles in the same order.
Only return the JSON array, no other text.

${JSON.stringify(titles)}`;

  try {
    const response = await openai.responses.create({
      model: 'gpt-5-nano',
      reasoning: { effort: 'minimal' },
      input: prompt,
    });

    const parsed = JSON.parse(response.output_text);
    if (Array.isArray(parsed) && parsed.length === titles.length) {
      return parsed;
    }
    return null; // パースエラー → フォールバック
  } catch {
    return null;
  }
}

// GPT-5 Nano 個別翻訳（バッチ失敗時のフォールバック）
async function translateSingleWithGPT(title: string): Promise<string | null> {
  try {
    const response = await openai.responses.create({
      model: 'gpt-5-nano',
      reasoning: { effort: 'minimal' },
      input: `Translate this Japanese product title to English: ${title}`,
    });
    return response.output_text.trim();
  } catch {
    return null;
  }
}

// AWS Translate（最終フォールバック）
async function translateWithAWS(title: string): Promise<string> {
  try {
    const command = new TranslateTextCommand({
      Text: title,
      SourceLanguageCode: 'ja',
      TargetLanguageCode: 'en',
    });
    const response = await translateClient.send(command);
    return response.TranslatedText || title;
  } catch {
    return title; // 全フォールバック失敗時は原文返却
  }
}

// 多段フォールバック翻訳
async function translateWithFallback(titles: string[]): Promise<string[]> {
  const results: string[] = new Array(titles.length);
  const failedIndices: number[] = [];

  // Step 1: バッチ翻訳
  for (let i = 0; i < titles.length; i += BATCH_SIZE) {
    const batch = titles.slice(i, i + BATCH_SIZE);
    const batchResult = await translateBatchWithGPT(batch);

    if (batchResult) {
      batchResult.forEach((title, j) => {
        results[i + j] = title;
      });
    } else {
      // バッチ失敗 → 個別翻訳にフォールバック
      batch.forEach((_, j) => failedIndices.push(i + j));
    }
  }

  // Step 2: 個別翻訳（バッチ失敗分）
  const awsFallbackIndices: number[] = [];
  await Promise.all(
    failedIndices.map(async (idx) => {
      const result = await translateSingleWithGPT(titles[idx]);
      if (result) {
        results[idx] = result;
      } else {
        awsFallbackIndices.push(idx);
      }
    })
  );

  // Step 3: AWS Translate（最終フォールバック）
  await Promise.all(
    awsFallbackIndices.map(async (idx) => {
      results[idx] = await translateWithAWS(titles[idx]);
    })
  );

  return results;
}
```

#### Step 2: 翻訳キャッシュテーブルの作成

**Supabase SQL:**

```sql
CREATE TABLE mst_translation_cache (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  source_text TEXT NOT NULL,
  source_hash TEXT NOT NULL,  -- SHA-256ハッシュ（検索高速化）
  translated_text TEXT NOT NULL,
  provider TEXT NOT NULL DEFAULT 'gpt-5-nano',  -- 翻訳プロバイダー
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- 高速検索用インデックス
CREATE UNIQUE INDEX idx_translation_cache_hash ON mst_translation_cache (source_hash);

-- キャッシュのヒット率を計測できるように統計情報を有効化
COMMENT ON TABLE mst_translation_cache IS '翻訳結果キャッシュテーブル';
```

**キャッシュ検索・保存の実装:**

```typescript
import { createHash } from 'crypto';

function hashTitle(title: string): string {
  return createHash('sha256').update(title).digest('hex');
}

async function checkCache(
  titles: string[]
): Promise<{ cached: Map<string, string>; uncached: string[] }> {
  const supabase = await createClient();
  const hashes = titles.map(hashTitle);

  const { data } = await supabase
    .from('mst_translation_cache')
    .select('source_hash, translated_text')
    .in('source_hash', hashes);

  const cacheMap = new Map(
    (data || []).map((row) => [row.source_hash, row.translated_text])
  );

  const cached = new Map<string, string>();
  const uncached: string[] = [];

  titles.forEach((title) => {
    const hash = hashTitle(title);
    const cachedResult = cacheMap.get(hash);
    if (cachedResult) {
      cached.set(title, cachedResult);
    } else {
      uncached.push(title);
    }
  });

  return { cached, uncached };
}

async function saveCache(
  titles: string[],
  translations: string[]
): Promise<void> {
  const supabase = await createClient();
  const rows = titles.map((title, i) => ({
    source_text: title,
    source_hash: hashTitle(title),
    translated_text: translations[i],
    provider: 'gpt-5-nano',
  }));

  await supabase
    .from('mst_translation_cache')
    .upsert(rows, { onConflict: 'source_hash' });
}
```

#### Step 3: クライアント側の翻訳関数を置き換え

**変更ファイル: `utils/csv-export.ts`**

現在の `translateTitles` 関数を、API Route呼び出しに置き換える。

```typescript
// Before（削除）
import {
  TranslateClient,
  TranslateTextCommand,
} from '@aws-sdk/client-translate';

export const translateClient = new TranslateClient({ ... });

export const translateTitles = async (titles: string[]) => {
  // AWS Translate直接呼び出し
};

// After（置き換え）
export const translateTitles = async (titles: string[]): Promise<string[]> => {
  try {
    const response = await fetch('/api/translate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ titles }),
    });

    if (!response.ok) {
      console.error('Translation API error:', response.status);
      return titles; // フォールバック: 原文返却
    }

    const data = await response.json();
    return data.translations;
  } catch (error) {
    console.error('翻訳APIの呼び出しに失敗しました:', error);
    return titles; // フォールバック: 原文返却
  }
};
```

**変更のポイント:**
- `translateTitles` のインターフェース（引数・戻り値）は変更なし
- 呼び出し元（6箇所）のコード変更は不要
- 後処理（`removeMultibyteChars`, `limitTitleLength`）はAPI側で実行するため、呼び出し元での重複処理を後に整理

#### Step 4: 環境変数の整理

**追加する環境変数（サーバーサイドのみ）:**

```env
# OpenAI API（NEXT_PUBLIC_ なし = サーバーサイドのみ）
OPENAI_API_KEY=sk-...

# AWS認証情報もNEXT_PUBLIC_プレフィックスを除去
AWS_REGION=ap-northeast-1
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

**削除する環境変数（セキュリティ修正）:**

```env
# 以下を削除（ブラウザに露出しているため）
NEXT_PUBLIC_AWS_REGION=...
NEXT_PUBLIC_AWS_ACCESS_KEY_ID=...
NEXT_PUBLIC_AWS_SECRET_ACCESS_KEY=...
```

#### Step 5: 不要な依存関係の整理

`@aws-sdk/client-translate` のフロントエンド側インポートを削除し、API Route内のみで使用する。最終的にAWS Translateが不要になった場合は依存関係ごと削除可能。

---

### 4.3 大量翻訳対応（非同期キューイング）

月間約100万件の翻訳が発生するため、大量リクエスト時の信頼性を確保する。

#### 同期API（少量用）

```
POST /api/translate
Body: { titles: string[] }  ※ 100件以下を推奨

Response: { success: true, translations: string[] }
```

- 100件以下の翻訳に使用
- 即座に翻訳結果を返却

#### 非同期API（大量用）

```
POST /api/translate/async
Body: { titles: string[], taskId: string }

Response: { success: true, jobId: string, status: "queued" }
```

```
GET /api/translate/status?jobId=xxx

Response: {
  status: "processing" | "completed" | "failed",
  progress: { total: 1000, completed: 450 },
  translations?: string[]  // completedの場合のみ
}
```

- 100件超の翻訳に使用
- バックグラウンドでキュー処理
- ポーリングで進捗確認

**実装方法:**
- Supabaseの `trn_translation_job` テーブルでジョブ管理
- API Route内でバッチ分割＋逐次処理
- ジョブステータスをDB経由でポーリング

---

### 4.4 レートリミット対策

| 対策 | 内容 |
|------|------|
| Tier昇格 | OpenAIに$100入金 → 7日後にTier 3（TPD 4,000万、20倍の余裕） |
| バッチ処理 | 10件/リクエストでAPIコール数を1/10に削減 |
| キャッシュ | 同一タイトルの再翻訳を防止（月間タイトルの重複率次第で大幅削減） |
| 分散処理 | 大量翻訳は非同期キューで均等分散 |
| バックプレッシャー | Tier 1期間（初週）はリクエストレートを制限 |

---

## 5. 実装スケジュール

| フェーズ | 作業内容 | 見積もり |
|---------|---------|---------|
| **Step 1** | 翻訳APIエンドポイント作成（`/api/translate`） | — |
| **Step 2** | 翻訳キャッシュテーブル作成（Supabase） | — |
| **Step 3** | `translateTitles` 関数のAPI呼び出し置き換え | — |
| **Step 4** | 環境変数の整理（`NEXT_PUBLIC_` → サーバーサイド） | — |
| **Step 5** | 動作検証・テスト | — |
| **Step 6** | 非同期キューイング（大量翻訳対応） | — |
| **Step 7** | AWS Translateフォールバック確認＋不要依存の整理 | — |

---

## 6. リスクと対策

| リスク | 影響 | 対策 |
|--------|------|------|
| OpenAI API障害 | 翻訳不可 | 多段フォールバック（GPT-5 Nano → 個別 → AWS Translate） |
| GPT-5 Nano品質劣化 | 翻訳品質低下 | GPT-4o-miniへの切り替え（コスト微増だが品質安定） |
| Tier 1期間のレートリミット | 大量翻訳が制限される | 初週は処理ペースを抑える / キャッシュを活用 |
| JSONパースエラー（バッチ翻訳） | バッチ翻訳失敗 | 個別翻訳へ自動フォールバック（実測エラー率2%） |
| キャッシュ肥大化 | DB容量増加 | 定期的な古いキャッシュの削除（90日以上など） |

---

## 7. 成果物一覧

| # | 成果物 | 種別 |
|---|--------|------|
| 1 | `app/api/translate/route.ts` | 新規作成 |
| 2 | `app/api/translate/async/route.ts` | 新規作成（非同期用） |
| 3 | `app/api/translate/status/route.ts` | 新規作成（ステータス確認用） |
| 4 | `mst_translation_cache` テーブル | Supabase新規作成 |
| 5 | `trn_translation_job` テーブル | Supabase新規作成（非同期ジョブ管理） |
| 6 | `utils/csv-export.ts` | 既存改修（translateTitles置き換え） |
| 7 | `.env` | 既存改修（環境変数整理） |
| 8 | `package.json` | 既存改修（openai パッケージ追加） |

---

## 8. 費用まとめ

| 項目 | 金額 |
|------|------|
| 現状 月間コスト（AWS Translate） | $916/月 |
| 移行後 月間コスト（GPT-5 Nano） | 約$11/月 |
| OpenAI初期入金（Tier 3昇格用） | $100（APIクレジットとして消費可） |
| **月間削減額** | **約$905/月** |
| **年間削減額** | **約$10,860/年** |

> 初期入金$100は約9ヶ月分のAPIクレジットに相当し、無駄にならない。
