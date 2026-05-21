---
title: "「うっかり課金」を防ぐだけのiOSアプリをReact Nativeで作ってApp Storeに出した"
emoji: "🛡️"
type: "tech"
topics: ["reactnative", "expo", "ios", "appstore", "個人開発"]
published: true
---

## 作ったもの

**Subscription Guard**（サブスクリプション ガード）というiOSアプリです。

https://apps.apple.com/jp/app/subscription-guard/id6770502246

「無料トライアルを登録したまま、うっかり課金された」経験がある人に向けたアプリです。機能はシンプルで、**終了日を登録すると前日の朝9時に通知が来る**、それだけです。

## なぜ作ったか

無料トライアルの管理ができるアプリはいくつかあります。ただ、ほとんどは「全サブスクの管理」「月額合計の可視化」「家計簿との連携」といった機能を持つ多機能アプリです。

自分が欲しかったのはそこではなく、「今トライアル中のやつ、終わる前に教えてくれ」だけでした。

管理画面もグラフも要らない。通知さえ来れば、解約するかどうかは自分で判断できます。

## 技術スタック

| 項目 | 採用技術 |
|------|---------|
| フレームワーク | React Native + Expo (SDK 52) |
| 言語 | TypeScript |
| ストレージ | AsyncStorage（ローカルのみ） |
| 通知 | expo-notifications |
| ビルド / 申請 | EAS Build / EAS Submit |

外部サーバーは一切使っていません。データはすべてデバイス内に保存されます。

## 機能一覧

- サービス選択（Netflix・Spotify・iCloud+ など25種類以上のプリセット、カスタム登録も可）
- 終了日の設定
- 前日朝9時の通知
- ホーム画面に「あと何日」表示
- 残り日数が3日以内になるとハイライト

機能を絞ることを意識しました。「全サブスクを管理するアプリ」ではなく、「うっかり課金を防ぐだけのアプリ」です。

## ディレクトリ構成

```
subscription-guard/
├── app/
│   ├── (tabs)/
│   │   ├── index.tsx        # ホーム（トライアル一覧）
│   │   └── add.tsx          # 追加ウィザード
│   └── _layout.tsx
├── src/
│   ├── components/          # TrialCard, ServicePicker など
│   ├── hooks/               # useTrials, useNotifications
│   ├── utils/
│   │   ├── dates.ts         # daysUntil, formatDate など
│   │   └── notifications.ts
│   └── types.ts
├── assets/
└── eas.json
```

## ハマったポイント

### 1. タイムゾーンバグ（daysUntil）

`daysUntil` 関数で「今日なのに -1 が返る」バグがありました。

```ts
// ❌ バグあり（JST +09:00 環境で09:00前に-1になる）
export function daysUntil(isoDate: string): number {
  const now = new Date();
  now.setHours(0, 0, 0, 0); // ← ローカル時刻で切り捨て
  const target = new Date(isoDate); // ← UTC 00:00 として解釈
  return Math.round((target.getTime() - now.getTime()) / 86400000);
}
```

`new Date('2026-06-01')` はUTC 00:00:00として解釈されますが、`setHours(0,0,0,0)` はローカル時刻（JST）で切り捨てます。JSTは+9時間なので、JST朝9時より前は両者に9時間のズレが生じて -1 になっていました。

```ts
// ✅ 修正後（Date.UTC で統一）
export function daysUntil(isoDate: string): number {
  const now = new Date();
  const todayUTC = Date.UTC(now.getUTCFullYear(), now.getUTCMonth(), now.getUTCDate());
  const [y, m, d] = isoDate.split('-').map(Number);
  const targetUTC = Date.UTC(y, m - 1, d);
  return Math.round((targetUTC - todayUTC) / 86400000);
}
```

ローカル時刻を一切使わず、UTC同士で比較するようにしました。

### 2. EAS Submit の後に配信設定が必要

`eas submit` は「App Reviewに提出する」だけです。審査通過後、**App Store Connect で配信先の設定を別途行わないと App Store に並びません**。

具体的には審査通過後に App Store Connect の「価格および配信状況」→「配信状況の設定」から全175か国を選択する手順が必要です。これを知らずに「審査通ったのに見つからない」になりました。

### 3. credentials.json の管理

EAS BuildにはApple Developer Programの証明書が必要です。`eas credentials` で管理するか、`credentials.json` にローカル保存するかを選べます。チーム開発では注意が必要ですが、個人開発なら `credentials.json` をローカルに置いてリポジトリに含めないのが楽でした。

## App Store 審査

初回申請から審査通過まで約4日でした（2026年5月）。

リジェクトはなく、一発通過でした。審査メモに「ログイン不要・+ボタンでトライアル追加できる」とシンプルに書いておいたのが良かったかもしれません。

## まとめ

React Native + Expo + EAS の組み合わせは、個人でiOSアプリを出す用途に向いていると思います。

- ネイティブコードをほぼ書かずにApp Storeに出せる
- EAS Build/Submit でビルド〜申請を自動化できる
- TypeScriptで書けるので型の恩恵を受けられる

機能を絞って「これだけをやる」アプリとして作ったので、コードベースもそれほど大きくならず、最後まで把握した状態で完成できました。

アプリはこちら（無料です）：
https://apps.apple.com/jp/app/subscription-guard/id6770502246
