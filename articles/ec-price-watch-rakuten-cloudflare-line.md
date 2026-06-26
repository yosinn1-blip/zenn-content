---
title: "楽天市場の競合価格監視をCloudflare Workers + LINE Botで無料で作った【OSS公開】"
emoji: "📊"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "linebot", "rakuten", "oss"]
published: true
---

## はじめに

楽天市場に出品していると、こんな悩みがあります。

> 「競合が値下げしたのを気づかず、気が付いたら自分の商品が最安値じゃなくなっていた」

既存の価格監視ツールを調べると…

| ツール | 料金 |
|---|---|
| プライスサーチ | ¥27,500/月 |
| らくらく最安更新 | ¥20,000/月 |

**高すぎる。**

そこで Cloudflare Workers + LINE Bot を使って、**月0円（個人利用）** で動く価格監視ツールを作りました。OSSとして公開しています。

## 作ったもの

LINE公式アカウントに友達追加するだけで使えます。

**登録フロー（LINE上で完結）:**
```
ユーザー: 登録
Bot:      📦 監視したい商品のJANコード（13桁の数字）を送ってください。
ユーザー: 4984824719866
Bot:      ✅ 商品を確認しました:
          パナソニック アルカリ乾電池 単3形 2本パック LR6XJ/2B
          あなたの販売価格（円）を数字で送ってください。
ユーザー: 500
Bot:      🎉 登録完了！次回チェック時（6・12・18・24時）から監視を開始します。
```

**競合が安くなったら即通知:**
```
🔔 価格アラート
商品: パナソニック アルカリ乾電池 単3形 2本パック LR6XJ/2B

あなたの価格: ¥500
楽天最安値:  ¥175（いーでん楽天市場店）
            → ¥325 下回られています

楽天で確認: https://search.rakuten.co.jp/search/mall/4984824719866/
```

## 技術スタック

| 役割 | 技術 | 費用 |
|---|---|---|
| サーバー | Cloudflare Workers | 無料（10万req/日） |
| DB | Cloudflare Workers KV | 無料（1GB） |
| 通知 | LINE Messaging API | 無料（月200通） |
| 価格取得 | 楽天 Ichiba Item Search API | 無料 |

全部無料枠で動きます。

## アーキテクチャ

```
[Cron: 毎時0分]
    ↓
[Cloudflare Worker]
    ↓ KVから全ユーザーの監視商品を取得
[楽天 Ichiba Item Search API]
    ↓ JANコードで最安値を検索
[価格比較]
    ↓ 自分の価格 > 楽天最安値 ならアラート
[LINE Messaging API]
    → プッシュ通知

[LINE Webhook]
    ↓ ユーザーの「登録」メッセージ
[対話式登録フロー]
    → KVに保存
```

## 実装のポイント

### 1. 楽天APIの新エンドポイント（2026年4月〜）

2026年4月から楽天APIのエンドポイントが変わり、`applicationId`（UUID形式）に加えて`accessKey`（pk_形式）が必須になりました。

```js
// 新エンドポイント
const RAKUTEN_SEARCH_URL =
  'https://openapi.rakuten.co.jp/ichibams/api/IchibaItem/Search/20260401';

const url = new URL(RAKUTEN_SEARCH_URL);
url.searchParams.set('applicationId', applicationId);  // UUID形式
url.searchParams.set('accessKey', accessKey);           // pk_形式（NEW）
url.searchParams.set('keyword', jan);                   // JANコードで検索
url.searchParams.set('sort', '+itemPrice');             // 安い順
```

旧エンドポイント（`app.rakuten.co.jp`）にUUID形式のIDを渡すと400エラーになります。

### 2. LINE Webhookの署名検証（Web Crypto API）

Cloudflare WorkersにはNode.jsの`crypto`モジュールがないため、Web Crypto APIを使います。

```js
export async function verifyLineSignature(secret, body, signature) {
  const key = await crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['verify'],
  );
  const expected = await crypto.subtle.sign(
    'HMAC', key, new TextEncoder().encode(body),
  );
  const expectedB64 = btoa(String.fromCharCode(...new Uint8Array(expected)));
  return expectedB64 === signature;
}
```

### 3. LINE Botの状態管理

ユーザーの会話状態（JAN待ち・価格待ち）をWorkers KVに保存することで、サーバーレスでもステートフルな対話を実現しています。

```js
// KVのユーザーデータ構造
{
  lineUserId: "Uxxxx",
  state: "IDLE" | "WAITING_JAN" | "WAITING_PRICE",
  items: [
    { jan: "4984824719866", myPrice: 500, label: "商品名", addedAt: "..." }
  ]
}
```

### 4. cronは毎時実行、JST判定はWorker内で

Cloudflare Workersのcronは `0 * * * *`（毎時0分）で動かし、Worker内でJST時刻を判定して実際のチェックは6・12・18・24時のみ実行します。

```js
// wrangler.toml
[triggers]
crons = ["0 * * * *"]

// price-checker.mjs
const CHECK_JST_HOURS = new Set([6, 12, 18, 0]);

export function getJstHour(date) {
  return new Date(date.getTime() + 9 * 60 * 60 * 1000).getUTCHours();
}
```

## セットアップ方法

### AIに任せる（おすすめ）

以下のプロンプトを [Claude.ai](https://claude.ai)（無料）に貼るだけです。

```
あなたはCloudflare Workers + LINE Botのセットアップを手伝うアシスタントです。
ec-price-watchのセットアップを対話形式で進めてください。

【STEP 1】楽天アプリID取得
https://webservice.rakuten.co.jp/ で「新規アプリ登録」

【STEP 2】LINE公式アカウント作成
https://manager.line.biz/ でアカウント作成（無料・審査なし）
→ Messaging API > チャネルアクセストークン発行

【STEP 3】Cloudflareセットアップ
1. https://cloudflare.com でアカウント作成
2. npx wrangler login
3. npx wrangler kv namespace create ec-price-watch
4. wrangler.toml の id を差し替え
5. npx wrangler deploy

詰まったところを教えてください。
```

### 手動セットアップ

```bash
git clone https://github.com/yosinn1-blip/ec-price-watch
cd ec-price-watch

# KV作成
npx wrangler kv namespace create ec-price-watch
# → 出力されたIDをwrangler.tomlに記入

# シークレット登録
npx wrangler secret put RAKUTEN_APP_ID
npx wrangler secret put RAKUTEN_ACCESS_KEY
npx wrangler secret put LINE_CHANNEL_TOKEN
npx wrangler secret put LINE_CHANNEL_SECRET

# デプロイ
npx wrangler deploy
```

デプロイ後、LINE Messaging APIの設定画面でWebhook URLを `https://<your-worker>.workers.dev/webhook/line-bot` に設定すれば完成です。

## 使い方

| コマンド | 動作 |
|---|---|
| `登録` | 商品の監視を追加（JAN→価格を対話式で入力） |
| `リスト` | 登録中の商品一覧を表示 |
| `削除` | 削除したい商品を番号で選ぶ |
| `削除 1` | 1番の商品を削除 |

## 注意事項

- LINE無料プランのプッシュ通知は**月200通まで**。出品者として本格運用する場合はLINE有料プラン（月3,000円〜）への切り替えを推奨
- 楽天API利用規約に従い、取得データは価格監視目的のみに使用してください

## リポジトリ

https://github.com/yosinn1-blip/ec-price-watch

スター・フォーク・PRお待ちしています。
