---
title: "GBP API 60日待ちに完成させたもの全部: LLMの癖・通知設計・グローバル対応"
emoji: "🌍"
type: "tech"
topics: ["cloudflareworkers", "googlebusinessprofile", "ai", "line", "oss"]
published: true
---

[前回の記事](https://zenn.dev/yosiki/articles/gbp-api-rejected-60day-rule)でGBP APIを0日で却下された経緯を書きました。次の再申請は2026年9月以降です。

その間、「API承認後に動かすだけ」の状態にするために、依存しない部分を全部作り切りました。この記事はその技術的な記録です。

## 作ったもの一覧

| モジュール | ファイル | 状態 |
|---|---|---|
| AI返信エンジン | `src/reply-engine.mjs` | ✅ テスト11件 |
| LINE通知 | `src/line-notify.mjs` | ✅ テスト15件 |
| 通知チャネル抽象化 | `src/notify.mjs` | ✅ LINE/Telegram両対応 |
| Cloudflare Worker | `worker/index.mjs` | ✅ 本番デプロイ済み |
| Cron補助 | `src/cron.mjs` | ✅ タイムゾーン対応 |
| Webhook HMAC検証 | `src/hmac.mjs` | ✅ Yelp/Trustpilot対応 |
| レビューパーサー | `src/review-parser.mjs` | ✅ GBP/Yelp/Trustpilot |
| **テスト合計** | `test/` | **89件パス** |

以下、設計上の判断が面白かった部分をピックアップして書きます。

---

## LLMの「癖」と向き合う: サニタイザの設計

Groqで口コミ返信を生成していると、ごくたまに想定外の出力が来ます。

**① 韓国語文字が混入する**

Groq（Llama-3.3-70b）を使っていると、日本語の返信の途中に唐突にハングルが1〜2文字混ざることがあります。体感では100件に1〜2回程度。

```
ありがとうございます。またのご来店를 스心よりお待ちしております。
```

店主がこれをそのまま投稿するとブランドが傷つくので、検出して自動的に1回だけ再生成します。

```js
const HANGUL_RE = /[가-힣]/;
const shouldRetry = warnings.includes("hangul-contamination");
if (!shouldRetry) break;
```

**② 前置きラベルを付けてくる**

「返信の下書き：〜」「Draft: 〜」のような前置きが付くことがあります。プロンプトで禁止しても完全には防げないので、正規表現で除去します。

```js
const PREAMBLE_RE = /^\s*(返信(の下書き)?|下書き|Reply|Draft|回答)\s*[:：]\s*/i;
text = text.replace(PREAMBLE_RE, "").trim();
```

**③ 名前のプレースホルダーを入れてくる**

`〇〇様`や`（お客様のお名前）様`のような穴埋め文字が出ることがあります。そのまま投稿すると当然おかしいので、検出してwarningとして上位に通知します（再生成はしない。文章自体は使えるため）。

---

## 言語を自動判定して英語・韓国語の口コミに対応

GBPは世界共通プロダクトです。インバウンド需要のある飲食店や観光地では、英語・韓国語・中国語の口コミが普通に届きます。

口コミのテキストを見て言語を推定し、返信もその言語で生成します。

```js
export function detectLang(text) {
  if (!text) return 'en';
  if (/[぀-ゟ゠-ヿ]/.test(text)) return 'ja';  // ひらがな・カタカナ
  if (/[가-힣]/.test(text)) return 'ko';         // ハングル
  return 'en';                                     // それ以外は英語
}
```

中国語（繁体・簡体）は今のところ英語システムプロンプトに流し込む設計です。モデルはそのまま中国語で返答するので、「レビューと同じ言語で返す」というルールを守ればおおむね問題ありません。

---

## 医療・治療系の自動免責: 薬機法・景表法への配慮

整体院・歯科・エステなどを対象にしているとき、AIが「必ず治ります」「確実に痩せられます」という断定表現を生成してしまうと、薬機法・景表法の問題になります。

店舗の業種名を判定して、自動でシステムプロンプトに制約を追加します。

```js
const HEALTH_BIZ_RE = /整体|接骨|整骨|鍼灸|治療院|クリニック|歯科|medical|clinic|dental.../i;
```

業種カテゴリを3段階に分け、それぞれ追加する制約文を変えています。

| カテゴリ | 例 | 追加される制約 |
|---|---|---|
| medical | クリニック・歯科・病院 | 診断・治癒を約束する表現を禁止 |
| therapy | 整体・鍼灸・接骨 | 医療的な回復を断言しない |
| beauty-medical | エステ・脱毛・痩身 | 効果を断定しない（個人差あり前提） |

この判定はビジネスタイプの文字列に対して実行するので、`口コミ本文`を見ているわけではありません。店舗登録時に`businessType`を設定しておくだけで機能します。

---

## LINE通知: 1口コミ1通は送らない

LINE Messaging API の無料枠は月200通です。口コミが届くたびに1通送っていると、店舗によってはすぐ枯れます。

設計は**1日1通のダイジェスト**にしました。

```
【本日の新着口コミ】ソフィア美容室

⭐⭐⭐⭐⭐ (5) — 山田 太郎様
カラーもカットも大満足でした！スタッフさんが親切で...
→ AI返信案: 山田様、大変嬉しいお言葉をいただき...

⭐⭐ (2) — 匿名様
待ち時間が長すぎました
→ AI返信案: お待たせしてしまい大変申し訳ございません...
```

1日に口コミが何件届いても、その日の分をまとめて1通に収めます。月30日で30通。無料枠の15%で収まります。

---

## Telegram対応: LINEがない国でも使えるように

LINEは日本・東南アジア中心のアプリです。GBPは世界共通なので、最初からグローバルを意識して設計しました。

通知チャネルを抽象化して、設定値1つで切り替えられます。

```js
// store.notificationChannel の値で自動切替
if (channel === 'line')     return sendViaLine({ store, reviews });
if (channel === 'telegram') return sendViaTelegram({ store, reviews });
```

TelegramはBot APIの`sendMessage`を使います。LINE Notifyが2025年に終了したのと同様、将来的に別のチャネルが必要になっても、このレイヤーに追加するだけで済む設計にしてあります。

---

## Yelp・Trustpilot のWebhook対応: HMAC署名検証

GBP以外のレビュープラットフォームからも口コミを受け取れるようにしました。現時点でYelpとTrustpilotのWebhookに対応しています。

外部WebhookはHMACで署名検証しないと、誰でもリクエストを送り込めてしまいます。

```js
// crypto.subtle ベース（Workers/Node.js両対応）
export async function verifyHmacSignature({ secret, payload, signature }) {
  const key = await crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['verify']
  );
  // timing-safe compare で長さ情報の漏れを防ぐ
  return crypto.subtle.verify('HMAC', key, expected, payload);
}
```

プラットフォームごとに署名ヘッダーの形式が異なるため、`src/review-parser.mjs`でペイロードを正規化した上で共通パイプラインに流します。

---

## タイムゾーン別Cron: 毎日現地9時に届ける

ダイジェストを「毎日同じ時間に届ける」には、タイムゾーンを考慮する必要があります。CloudflareのCron TriggerはすべてUTC基準です。

解決策は**毎時実行にして、Workerの中でフィルタ**することです。

```js
// wrangler.toml
crons = ["0 * * * *"]  // 毎時0分

// scheduled ハンドラ内
const utcHour = new Date(event.scheduledTime).getUTCHours();
// store.utcOffset (+9=JST, +1=BST, -5=EST) から現地時刻を計算
const localHour = ((utcHour + store.utcOffset) % 24 + 24) % 24;
if (localHour !== 9) return; // 現地9時でなければスキップ
```

日本の店舗は`utcOffset: 9`、UKなら`+1`、東海岸アメリカなら`-5`を設定します。これで「現地の朝9時にダイジェストが届く」が全タイムゾーンで成立します。

---

## 公開ウィザードのセキュリティ設計: ADMIN_KEYを渡さない

設置ウィザード（`wizard.html`）を作ったとき、当初「ADMIN_KEYをフォームに埋めて店舗登録するAPI呼ぶ」設計を考えました。

これは**明らかに間違いでした**。

公開ページのJavaScriptに秘密鍵を入れると、誰でもブラウザのDevToolsで見られます。そのキーで他の店舗の設定を上書きできてしまいます。

代わりに`POST /signup`という公開エンドポイントを新設しました。

- 受け取るのはLINEのcredentialのみ（channelAccessToken, userId）
- まずLine Profile APIで実在するcredentialか検証する
- 検証が通ったら、Server側で`storeId`と`apiKey`を`crypto.randomUUID()`で生成してKVに保存
- フロントにはその2つだけ返す

ADMIN_KEYはこのフローに一切登場しません。公開ページに秘密鍵を出さない設計です。

発見したのはZenn記事を書く直前だったので、公開前に修正できてよかったです。

---

## データの7日TTL: GDPRを最初から考慮する

GBP APIが通ったあとは、ユーザーのレビュー本文がWorkerのKVを経由します。レビューには個人名や連絡先が含まれることがあるので、保持期間を7日に制限しました。

```js
await env.STORES.put(pendingKey, JSON.stringify(buffer), {
  expirationTtl: 7 * 24 * 3600,
});
```

処理済みのデータは送信後に即削除、店舗ごとのバッファは7日で自動消滅します。グローバル展開を見据えてGDPR/CCPA対応を最初から入れておく方針です。

---

## 現在の状態

```
口コミ（Yelp/Trustpilot/GBP）
  → POST /webhook/:platform  or  POST /review
  → generateReply（Groq既定・言語自動検出・サニタイザ）
  → KVにバッファ（7日TTL）
  → 毎時Cron → 現地9時に sendDigest（LINE or Telegram）
```

このループはGBP API承認なしの今でも動いています。Yelp/TrustpilotはWebhookで受け取れるので、日本以外の店舗は今すぐ使い始めることができます。

GBP側は承認待ち（2026年9月再申請予定）。承認が出れば、このパイプラインの入口にGBP口コミポーリングが加わるだけです。

---

## まとめ

| 課題 | 解決策 |
|---|---|
| Groqのハングル混入 | 検出して最大1回再生成 |
| 前置きラベル除去 | 正規表現でstrip |
| 医療系の断定表現 | 業種判定で自動免責文 |
| 多言語口コミ | `detectLang`でja/ko/en自動切替 |
| LINE無料枠 | 日次ダイジェスト（月30通） |
| グローバル通知 | LINE/Telegram切替可能な抽象層 |
| タイムゾーン | 毎時Cron＋Worker内フィルタ |
| Webhook改ざん防止 | HMAC-SHA256署名検証 |
| 公開ページのADMIN_KEY漏れ | `/signup`で鍵を渡さない設計 |
| 個人情報の保持 | KVの7日TTL＋送信後即削除 |

GBP API承認を待つ間に、わりといろんな問題を解いていました。

- GitHub: [yosinn1-blip/meo-harness](https://github.com/yosinn1-blip/meo-harness)（MIT License）
- 触れるデモ（登録不要）: https://yosinn1-blip.github.io/yoshiki-apps/demo.html
