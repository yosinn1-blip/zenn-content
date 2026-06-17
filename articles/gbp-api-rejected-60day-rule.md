---
title: "Google ビジネスプロフィール API に申請したら翌日却下された話と「60日ルール」の罠"
emoji: "🚫"
type: "tech"
topics: ["googlebusinessprofile", "meo", "googleapi", "gcp", "oss"]
published: true
---

[前回の記事](https://zenn.dev/yosiki/articles/meo-harness-oss-launch)で「GBP APIを使ったOSSを作っている」と書いた翌日、申請が却下されました。正直に記録します。

## 何が起きたか

2026年6月16日にGoogle Business Profile APIの利用申請（A-3フォーム）を提出しました。翌6月17日、以下のメールが届きました。

> We will not be able to move forward with your application to use the GBP API as your account did not pass our internal quality checks.
> 
> We recommend reviewing our eligibility criteria and ensuring that your Business Profile and your company's official website are fully up to date before reapplying in the future if you choose to do so.

「内部の品質チェックを通過しなかった」——理由の詳細は書いてありません。

## 原因：60日ルールの罠

申請後に各種フォーラムを調べると、**GBP確認済みから60日以上経っていないと自動却下される**という情報が複数出てきました。

私のタイムライン：

- 2026年6月12日：GBPオーナー確認の動画を提出
- 2026年6月16日：オーナー確認の承認を確認（business.google.com/locationsで「確認済み」に変化）
- 2026年6月16日：同日にA-3申請フォームを提出
- 2026年6月17日：却下メール

確認済みになってから**わずか0日**で申請したので、60日条件を大きく割り込んでいます。

:::message alert
この「60日ルール」は、[公式の申請要件ページ](https://developers.google.com/my-business/content/prereqs#request-access)に明確に書いていません。コミュニティフォーラムや開発者の体験談に散らばっているだけです。
:::

## 申請までに準備したこと（無駄ではない）

却下はされましたが、準備自体は今後の再申請にそのまま使えます。

**GCPプロジェクト設定**
- プロジェクト作成（yoshiki-apps-gbp）
- Business Information・Notifications・Q&A 等のAPIを有効化
- OAuth同意画面の構成（外部・テスト中、プライバシーポリシーURL設定、`business.manage`スコープ追加）
- OAuthクライアントID作成（ウェブアプリ）

**申請フォームに書いた内容**
- GCPプロジェクト番号
- 公式サイトURL
- 用途の説明（口コミへのAI返信下書き生成・投稿は人間が承認する前提）
- 順位スクレイピングはしないこと

---

一点、申請時に失敗したことがあります。Google Auth Platformでスコープを追加した際、サブパネルの「更新」を押しただけでページ下部の「**Save**」を押し忘れていました。`Save`を押さないとスコープは実際には保存されません。今回の却下原因ではないかもしれませんが、再申請時は必ず確認します。

## 60日待つ間にやったこと

API承認を待たずに実装できる部分は全部作りました。

**① AI返信エンジン（`src/reply-engine.mjs`）**

GBP APIなしでも、口コミのテキストがあればAI返信は生成できます。Groq（無料・クレカ不要）を既定にして、ハングル混入や前置きラベルを除去するサニタイザも実装済み。整体・接骨・クリニック系は効果保証の文言を自動で回避します。

**② LINE通知モジュール（`src/line-notify.mjs`）**

新着口コミをLINEにダイジェスト通知します。LINE Notifyは2025年に終了しているので、Messaging API pushで実装。1日1通まとめて送る設計で、無料枠200通/月に余裕で収まります。

**③ Cloudflare Worker（`worker/index.mjs`）**

`POST /review`（口コミ→AI下書き→LINE通知）・`POST /signup`（店舗登録）・`GET /health` の3エンドポイントを本番デプロイ済み。各店舗がBYOトークン（自分のLINEアカウント）を持つ設計なので、Yoshiki側に課金が集中しません。

**④ メール監視（暫定）**

GBP APIがなくても、GBPが送ってくる「新しいクチコミ」メールをGmail APIで検知する回路を作りました。AIが下書きを作ってLINEに流すまでのループは、すでに実LINE送信で動作確認済みです。

```
口コミ通知メール → Gmail API検知 → AI下書き生成（Groq） → LINEにダイジェスト送信
```

GBP APIが通ったら、このメール検知をAPIポーリングに差し替えるだけで済む設計にしてあります。

## 再申請のスケジュール

オーナー確認済みが2026年6月16日なので、60日後は**2026年8月15日**。保守的に数週間バッファを取って、**2026年9月初旬に再申請**する予定です。

申請フォームは同じURLから同内容で再提出するだけです。GCPプロジェクト・OAuth設定・コードはすでに完成しているので、再申請後に承認が出たら動かすだけの状態です。

## まとめ

| やること | 状態 |
|---|---|
| GBPオーナー確認 | ✅ 承認済み（2026-06-16） |
| GCPプロジェクト・OAuth設定 | ✅ 完了 |
| A-3 API利用申請 | ❌ 却下（60日未達） |
| AI返信エンジン | ✅ 実装・テスト済み |
| LINE通知 | ✅ 実LINE送信で動作確認済み |
| Cloudflare Worker（本番） | ✅ デプロイ済み |
| 再申請 | 📅 2026年9月予定 |

却下は想定の範囲内でしたが、「60日ルールがどこにも書いていない」のは開発者が詰まりやすいポイントだと思うので記録しました。同じ状況の人の参考になれば。

- GitHub: [yosinn1-blip/meo-harness](https://github.com/yosinn1-blip/meo-harness)（MIT License）
- 触れるデモ（登録不要）: https://yosinn1-blip.github.io/yoshiki-apps/demo.html
