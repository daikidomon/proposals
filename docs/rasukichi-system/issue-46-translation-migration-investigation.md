# 調査記録：翻訳機能のバックエンド移行 — LLM API活用による非同期翻訳基盤の構築

| 項目 | 内容 |
|------|------|
| Issue | [rasukichisama/rasukichi-system#46](https://github.com/rasukichisama/rasukichi-system/issues/46) |
| 記録日 | 2026年3月20日 |
| フェーズ | Phase 1（現状調査・コスト試算）完了 |

---

## 1. 調査の背景と目的

現在、らす吉システムの商品タイトル翻訳処理はフロントエンド（クライアントサイド）の `utils/csv-export.ts` にてAWS Translate APIを直接呼び出している。この構成には以下の問題がある。

1. **セキュリティリスク**: AWSクレデンシャルが `NEXT_PUBLIC_` 環境変数でブラウザに露出
2. **高コスト**: AWS Translateの月間費用が約$916（年間約$11,000）
3. **スケーラビリティの限界**: クライアント側 `Promise.all` による並列翻訳はブラウザリソースに依存
4. **キャッシュなし**: 同一タイトルの再翻訳が発生

本調査では、LLM APIへの移行可能性・コスト削減効果・翻訳品質を実データで検証した。

---

## 2. 現状の翻訳処理

### 2.1 処理フロー

```
日本語タイトル（入力）
  → preprocessTitle()         … #, <, > を除去
  → AWS TranslateTextCommand  … ja → en 翻訳
  → postprocessTitle()        … 現在はno-op
  → removeMultibyteChars()    … 非ASCII文字の除去
  → limitTitleLength()        … 80文字に制限
  → CSV出力 / DB保存
```

### 2.2 翻訳関数の定義

**ファイル:** `utils/csv-export.ts` L141-167

```typescript
export const translateTitles = async (titles: string[]) => {
  try {
    const translatedTitles = await Promise.all(
      titles.map(async (title) => {
        try {
          const preprocessedTitle = preprocessTitle(title);
          const command = new TranslateTextCommand({
            Text: preprocessedTitle,
            SourceLanguageCode: 'ja',
            TargetLanguageCode: 'en',
          });
          const response = await translateClient.send(command);
          const translatedText = response.TranslatedText || title;
          return postprocessTitle(translatedText);
        } catch (error) {
          console.error('タイトル翻訳エラー:', error);
          return title;
        }
      })
    );
    return translatedTitles;
  } catch (error) {
    console.error('一括翻訳中にエラーが発生しました:', error);
    return titles;
  }
};
```

- 入力: `string[]`（日本語タイトルの配列）
- 出力: `Promise<string[]>`（英語タイトルの配列、同じ長さ）
- エラー時は原文をそのまま返却（例外を投げない）

### 2.3 AWS Translateクライアント設定

**ファイル:** `utils/csv-export.ts` L1-21

```typescript
export const translateClient = new TranslateClient({
  region: process.env.NEXT_PUBLIC_AWS_REGION || 'ap-northeast-1',
  credentials: {
    accessKeyId: process.env.NEXT_PUBLIC_AWS_ACCESS_KEY_ID || '',
    secretAccessKey: process.env.NEXT_PUBLIC_AWS_SECRET_ACCESS_KEY || '',
  },
});
```

`NEXT_PUBLIC_` プレフィックスにより、これらの認証情報はブラウザのJavaScriptバンドルに含まれ、DevToolsから閲覧可能。

### 2.4 翻訳呼び出し箇所（全6箇所）

| # | ファイル | 行 | 関数名 | 用途 |
|---|---------|-----|--------|------|
| 1 | `utils/csv-export.ts` | L428 | `convertToEbayCSV` | eBay CSV出力時の翻訳 |
| 2 | `utils/csv-export.ts` | L564 | `generateCSVData` | CSV生成時の翻訳 |
| 3 | `utils/csv-export.ts` | L718 | `generateEbayCSVWithSKU` | SKU付きCSV生成時の翻訳 |
| 4 | `utils/csv-export.ts` | L901 | `saveEbayOfferItems` | eBayオファー保存時の翻訳 |
| 5 | `components/import_history/api.ts` | L969 | `startEbayListingRequest` | eBay出品リクエスト時 |
| 6 | `components/import_history/api.ts` | L1358 | `startEbayListingRequest`（sync版） | eBay出品リクエスト時 |

> `app/item_manage/import_history/[id]/page.tsx` L102 にもローカル関数 `generateEbayCSVData` 内で同等の翻訳処理が存在する。

### 2.5 ヘルパー関数

```typescript
// 前処理: 特殊文字除去
const preprocessTitle = (title: string) => {
  return title.replace(/#/g, '').replace(/</g, '').replace(/>/g, '');
};

// 後処理: 現在はno-op
const postprocessTitle = (translatedTitle: string) => {
  return translatedTitle;
};

// 非ASCII文字除去
export const removeMultibyteChars = (text: string): string => {
  if (!text) return '';
  return text.replace(/[^\x00-\x7F]/g, '');
};

// タイトル長制限（デフォルト80文字）
export const limitTitleLength = (title: string, maxLength: number = 80): string => {
  if (!title) return '';
  return title.length > maxLength ? title.substring(0, maxLength) : title;
};
```

---

## 3. AWS Translate 実績データ

### 3.1 月別費用実績

| 月 | AWS Translate実費用 |
|----|-------------------|
| 2025年12月 | $979.01 |
| 2026年1月 | $853.18 |
| **月平均** | **$916.10** |

### 3.2 翻訳量（CSVタイトルデータから計測）

| 月 | タイトル数 | 総文字数 | 平均文字数/タイトル |
|----|-----------|---------|-------------------|
| 2025年12月 | 970,996 | 4,587万文字 | 47.2 |
| 2026年1月 | 993,561 | 4,624万文字 | 46.5 |

---

## 4. LLM API 1000件実翻訳テスト

2026年1月のデータからランダム1000件を抽出し、各翻訳手段で実翻訳を実施。

### 4.1 コスト比較

| 手段 | 方式 | 入力トークン | 出力トークン | うち推論 | 処理時間 | 費用 |
|------|------|-------------|-------------|---------|---------|------|
| **AWS Translate** | - | (45,123文字) | - | - | 81秒 | **$0.6768** |
| **GPT-5 Nano (minimal)** | 個別 | 47,061 | 44,271 | 0 | 1,340秒 | **$0.0201** |
| **GPT-5 Nano (minimal)** | バッチ10件 | 35,736 | 24,080 | 0 | 276秒 | **$0.0114** |
| **GPT-5 Mini (minimal)** | 個別 | 48,773 | 38,788 | 0 | 1,180秒 | **$0.0898** |
| **GPT-5 Mini (minimal)** | バッチ10件 | 36,436 | 23,763 | 0 | 348秒 | **$0.0566** |
| **GPT-4o-mini** | 個別 | 62,505 | 19,109 | - | 921秒 | **$0.0208** |
| **GPT-4o-mini** | バッチ10件 | 37,436 | 22,073 | - | 477秒 | **$0.0189** |
| **Grok 4.1 Fast NR** | 個別 | 217,918 | 19,824 | - | 942秒 | **$0.0535** |
| **Grok 4.1 Fast NR** | バッチ10件 | 51,927 | 20,836 | - | 286秒 | **$0.0208** |
| GPT-5 Nano (デフォルト) | 個別 | 61,505 | 1,417,492 | 1,388,736 (98%) | 14,227秒(3.9h) | $0.5701 |
| GPT-5 Nano (デフォルト) | バッチ10件 | 37,336 | 591,132 | 567,296 (96%) | 5,435秒(1.5h) | $0.2383 |
| GPT-5 Mini (デフォルト) | 個別 | 61,505 | 531,419 | 501,696 (94%) | 9,633秒(2.7h) | $1.0782 |
| GPT-5 Mini (デフォルト) | バッチ10件 | 37,336 | 210,250 | 186,944 (89%) | 3,935秒(1.1h) | $0.4298 |

### 4.2 翻訳品質

| 手段 | エラー率 | 誤翻訳率 | 品質評価 |
|------|---------|---------|---------|
| **AWS Translate** | 0/1000 (0%) | ベースライン | 安定 |
| **GPT-5 Nano (minimal) 個別** | 0/1000 (0%) | - | 良好 |
| **GPT-5 Nano (minimal) バッチ** | 2/100バッチ (2%) JSONパースエラー | - | 良好（フォールバックで対応可） |
| **GPT-5 Mini (minimal) 個別** | 0/1000 (0%) | - | 良好 |
| **GPT-5 Mini (minimal) バッチ** | 0/100バッチ (0%) | - | 良好 |
| **GPT-4o-mini 個別** | 0/1000 (0%) | - | 最も高品質 |
| **GPT-4o-mini バッチ** | 0/1000 (0%) | 5/1000 (0.5%) | 良好 |
| **Grok 個別** | 4/1000 (0.4%) | - | 良好（403エラーあり） |
| **Grok バッチ** | 0/1000 (0%) | **362/1000 (36.2%)** | **重大な品質問題** |

> **Grokバッチ翻訳は36%が元タイトルと無関係な翻訳を返すため、採用不可。**

---

## 5. GPT-5 Nano / GPT-5 Mini の推論トークン問題

### 5.1 問題

両モデルはreasoningモデルであり、デフォルト設定では：

- 出力トークンの89〜98%が推論トークンに消費される
- `temperature` パラメータ非対応
- GPT-4o-miniの12〜23倍のコスト、8〜30倍の処理時間

### 5.2 解決策

**OpenAI Responses API + `reasoning: {"effort": "minimal"}` の使用で推論トークンを0に抑制。**

```python
from openai import OpenAI
client = OpenAI(api_key=API_KEY)

resp = client.responses.create(
    model="gpt-5-nano",
    reasoning={"effort": "minimal"},  # 推論トークン0
    input="Translate this Japanese product title to English: " + title,
)
```

> `reasoning: {"effort": "none"}` はGPT-5 Nano/Miniで未対応（minimal/low/medium/highのみ）。Chat Completions APIの `reasoning_effort` では抑制不可のため、**Responses APIの使用が必須**。

### 5.3 設定別コスト比較

| モデル | 設定 | バッチ費用 | GPT-4o-mini比 | 処理時間 | 推論トークン |
|--------|------|-----------|-------------|---------|------------|
| **GPT-5 Nano** | **minimal** | **$0.011** | **0.6x（最安）** | 276秒 | **0** |
| GPT-5 Mini | minimal | $0.057 | 3.0x | 348秒 | 0 |
| GPT-4o-mini | - | $0.019 | 1x | 477秒 | - |
| GPT-5 Nano | デフォルト | $0.238 | 12.6x | 5,435秒 | 567,296 |
| GPT-5 Mini | デフォルト | $0.430 | 22.7x | 3,935秒 | 186,944 |

---

## 6. 月間換算コスト比較

| 手段 | 月額推定 | AWS比削減率 | 備考 |
|------|---------|------------|------|
| **AWS Translate（実績）** | **$916** | — | 現状 |
| **GPT-5 Nano (minimal) バッチ** | **~$11** | **98.8%** | 最安・採用候補 |
| GPT-4o-mini バッチ | ~$19 | 97.9% | 安定性高い |
| Grok 4.1 Fast NR バッチ | ~$20 | 97.8% | バッチ品質問題あり |
| GPT-5 Mini (minimal) バッチ | ~$56 | 93.9% | Nanoより安定（エラー0） |
| GPT-5 Nano (デフォルト) バッチ | ~$236 | 74.2% | 推論トークン消費大 |
| GPT-5 Mini (デフォルト) バッチ | ~$427 | 53.4% | 推論トークン消費大 |

### 年間削減効果

| 項目 | 金額 |
|------|------|
| 現状 月平均（AWS Translate） | $916 |
| LLM移行後 月平均（GPT-5 Nano minimal バッチ） | 約$11 |
| **月間削減額** | **約$905** |
| **年間削減額** | **約$10,860** |

---

## 7. モデル選定結果

### 7.1 採用モデル

**GPT-5 Nano（Responses API + `reasoning: {"effort": "minimal"}`、バッチ10件）**

| 項目 | 結果 |
|------|------|
| コスト | $0.05/M input + $0.40/M output（全モデル中最安、AWS Translateの約60分の1） |
| 品質 | バッチ時JSONパースエラー2%（フォールバックで対応可） |
| 速度 | 1000件あたり276秒（バッチ）— 最速 |
| API | Responses API必須（Chat Completions APIでは推論トークン抑制不可） |
| 注意点 | `reasoning: {"effort": "minimal"}` の指定が必須 |

### 7.2 フォールバック構成

```
GPT-5 Nano (minimal, バッチ10件)
  → GPT-5 Nano (minimal, 個別)  ※バッチJSONパースエラー時
    → AWS Translate              ※最終手段
```

> GPT-4o-miniはバッチエラー率0%でより安定しているため、GPT-5 Nanoで問題が頻発する場合はGPT-4o-miniへの切り替えも選択肢。

---

## 8. レートリミット検証

### 8.1 必要処理量（2026年1月データ）

| 項目 | 値 |
|------|-----|
| 月間タイトル数 | 993,561 |
| バッチ数（10件/バッチ） | 99,356 |
| 月間総トークン | 約5,900万 |
| 30日均等分散時の1日あたり | 3,312リクエスト / 197万トークン |
| 1分あたり（30日均等） | 2.3リクエスト / 1,369トークン |

### 8.2 Tier別レートリミット判定

| Tier | RPM上限 | TPM上限 | TPD上限 | 判定 |
|------|---------|---------|---------|------|
| Tier 1 | 500 | 200,000 | 200万 | ギリギリ（ピーク日にオーバーの可能性） |
| **Tier 2** | 5,000 | 2,000,000 | 2,000万 | **OK（10倍の余裕）** |
| **Tier 3** | 5,000 | 4,000,000 | 4,000万 | **OK（20倍の余裕）** |
| Tier 4+ | 10,000+ | 10,000,000+ | 10億+ | 余裕 |

> 30日均等分散であればTier 2以上で安定運用可能。

### 8.3 OpenAI Tier昇格条件

| Tier | 必要入金額 | 必要経過日数（初回入金から） |
|------|-----------|--------------------------|
| Tier 1 | $5 | 7日 |
| Tier 2 | $50 | 7日 |
| Tier 3 | $100 | 7日 |
| Tier 4 | $250 | 14日 |
| Tier 5 | $1,000 | 30日 |

- **$100入金 → 7日後にTier 3に自動昇格**（TPD 4,000万、20倍の余裕）
- $100は月間コスト約$11の約9ヶ月分のクレジットとして利用可能
- 既にOpenAIに支払い履歴がある場合は即座にTier 3以上になる可能性あり

---

## 9. 調査結論

### 9.1 移行の妥当性

LLM API（GPT-5 Nano）への移行は**技術的・経済的に妥当**である。

| 評価軸 | 判定 | 根拠 |
|--------|------|------|
| コスト | ◎ | 月額$916 → $11（98.8%削減） |
| 品質 | ○ | バッチエラー率2%はフォールバックで吸収可能 |
| セキュリティ | ◎ | AWSクレデンシャルのブラウザ露出を解消 |
| 実装リスク | ○ | `translateTitles` のインターフェース維持により呼び出し元の変更不要 |
| 運用リスク | ○ | 多段フォールバック＋AWS Translate残存で可用性確保 |

### 9.2 次のアクション

1. **Phase 2**: バックエンドAPI設計・実装（`/api/translate` エンドポイント作成）
2. **Phase 3**: フロントエンド側の移行・テスト

---

## 参考リンク

- [AWS Translate Pricing](https://aws.amazon.com/translate/pricing/)
- [OpenAI API Pricing](https://platform.openai.com/docs/pricing)
- [OpenAI Responses API](https://platform.openai.com/docs/api-reference/responses)
- [OpenAI Rate Limits](https://platform.openai.com/docs/guides/rate-limits)
- [xAI Models and Pricing](https://docs.x.ai/developers/models)
