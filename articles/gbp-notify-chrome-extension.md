---
title: "Googleビジネスプロフィールのクチコミが来たらLINEに通知するChrome拡張を作った【無料・Chrome Store公開済み】"
emoji: "🔔"
type: "tech"
topics: ["chrome拡張", "googlebusinessprofile", "linebot", "cloudflareworkers", "manifestv3"]
published: true
---

## 背景

MEO Harness（GBPの口コミ管理OSSツール）を作っている中で、Google Business Profile APIの申請が「60日ルール」で弾かれました。

> [Google ビジネスプロフィール API に申請したら翌日却下された話と「60日ルール」の罠](https://zenn.dev/yosinn1/articles/gbp-api-rejected-60day-rule)

APIが使えないので、**Chrome拡張でDOM解析して口コミを取得→LINEに通知する**という迂回策を作りました。Chrome Web Storeに公開しています。

## 作ったもの

**GBP Notify — 口コミ通知**

インストール後にLINEと連携するだけで、`business.google.com` の口コミページを30分ごとに自動チェックして新着口コミをLINEに通知します。

**通知イメージ（LINE）:**
```
🔔 新着クチコミ 1件（Yoshiki Apps）

1. ★★★★★ 田中 太郎さん
「スタッフの対応がとても丁寧で…」
```

**ストア:**
https://chromewebstore.google.com/detail/gbp-notify/hkhedcmicnfbiejpamcjonmclbfhococ

## 技術スタック

| 役割 | 技術 | 費用 |
|---|---|---|
| フロントエンド | Chrome拡張（Manifest V3） | 無料 |
| バックエンド | Cloudflare Workers | 無料（10万req/日） |
| 状態管理 | Cloudflare Workers KV | 無料（1GB） |
| 通知 | LINE Messaging API | 無料（月200通） |

全部無料枠で動きます。

## アーキテクチャ

```
[Chrome拡張 Service Worker]
  ↓ 30分アラーム
  ↓ business.google.com/reviews をContent Scriptで解析
  ↓ 新着口コミのみ抽出
[POST /notify → Cloudflare Workers]
  ↓
[LINE Messaging API]
  → LINEにプッシュ通知
```

GBP APIは一切使いません。`business.google.com` のページをすでに開いている（または拡張が開く）ことで、ログイン済みのDOMから口コミを取得します。

## 実装のポイント

### 1. GBP APIなしでDOM解析

GBP API（`mybusinessreviews.googleapis.com`）は申請→審査が必要で、デフォルトのクォータは **0**（使えない）です。

そこでDOM解析を使いました。

```js
// content.js
const reviewEls = doc.querySelectorAll('[data-review-id], [jsdata*="review"]');

reviewEls.forEach(el => {
  const rating = el.querySelector('[aria-label*="star"], [aria-label*="星"]')
    ?.getAttribute('aria-label')?.match(/\d/)?.[0];
  const text = el.querySelector('[class*="review-text"], .Jtu6Td')?.textContent;
  // ...
});
```

GoogleのUIが変わったらセレクターの更新が必要ですが、APIが使えない間の現実的な解決策です。

### 2. 30分アラームで自動ポーリング（MV3対応）

Manifest V3ではBackground Pageが廃止され、Service Workerになりました。常駐できないので `chrome.alarms` で定期実行します。

```js
// background.js
chrome.runtime.onInstalled.addListener(async () => {
  await chrome.alarms.create('gbp-poll', { periodInMinutes: 30 });
});

chrome.alarms.onAlarm.addListener(async (alarm) => {
  if (alarm.name === 'gbp-poll') await triggerContentScriptCheck();
});
```

### 3. 初回はベースライン保存・通知しない

「インストール直後に過去の口コミが大量通知される」問題を防ぐため、最初のスキャンは `knownReviewIds` を保存するだけです。

```js
// 初回: ベースラインを保存して通知しない
if (state.knownReviewIds.length === 0) {
  await saveState({ knownReviewIds: reviews.map(r => r.id), ... });
  return { ok: true, new: 0, note: 'baseline-set' };
}

// 2回目以降: 新着のみ通知
const newReviews = reviews.filter(r => !state.knownReviewIds.includes(r.id));
```

### 4. LINE連携は6桁コードでペアリング

APIキー入力不要。拡張のポップアップに6桁コードが表示され、それをLINEボットに送るだけで連携完了します。

```
[拡張ポップアップ]  →  コード: 6Y6WJC
[LINEボット]        ←  ユーザーが「6Y6WJC」を送信
[Cloudflare Workers] →  コードとLINE UserIDを紐付けてKVに保存
[拡張がポーリング]  ←  /verify?code=6Y6WJC → lineUserId返却
[chrome.storage.local] → lineUserId保存 → 連携完了
```

## インストール方法

1. [Chrome Web Store](https://chromewebstore.google.com/detail/gbp-notify/hkhedcmicnfbiejpamcjonmclbfhococ) からインストール
2. [LINEボット（@477byprh）](https://lin.ee/477byprh) を友達追加
3. 拡張のポップアップを開いて表示される6桁コードをLINEボットに送る
4. 「連携完了」が表示されたらOK

以降は `business.google.com` を開くたびに（または30分ごとに）自動チェックが走ります。

## 注意事項

- **LINE無料プランは月200通まで。** ユーザーが増えた場合は有料プラン（月3,000円〜）への切り替えが必要
- GBP APIを使わずDOMを解析しているため、GoogleのUIが変わるとセレクターの更新が必要になる場合があります
- `business.google.com` にログイン済みのGoogleアカウントで動作します

## GBP API が使えるようになったら

60日ルールをクリアしてGBP APIのクォータが増えたら、DOMスクレイピングからAPI経由に切り替える予定です。APIが使えれば `business.google.com` を開かなくてもバックグラウンドで口コミ取得できます。

## リポジトリ

https://github.com/yosinn1-blip/gbp-notify

スター・フォーク・PRお待ちしています。
