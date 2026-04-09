---
title: "技術解説：快適なUIの裏側"
category: "Pantry Quest"
description: "「指に吸い付く」ようなUXを実現するための、TanStack Queryによる楽観的更新の実装。"
date: 2026-04-09
order: 4
---

## Pantry Questの技術スタック

Pantry QuestはモダンなWeb技術を駆使して、ネイティブアプリに負けないUXを目指しています。

- **Frontend**: Next.js (App Router), Tailwind CSS
- **Backend & DB**: Prisma, SQLite (Cloudflare D1への移行見込み)
- **State Management**: TanStack Query (React Query)
- **Deployment**: Vercel

### 最重要課題：「待たせない」こと

在庫管理ツールとしての最大の敵は「ネットワーク遅延」です。
スーパーの店内やキッチンの奥など、必ずしも電波状況が良いとは限らない場所で使うアプリだからです。

そこで、Pantry Questでは**「楽観的更新（Optimistic Updates）」**を全面的に採用しています。

### TanStack Query による楽観的更新

ユーザーが「スワイプして消費」のアクションを行った際、内部では以下の順序で処理が走ります。

```typescript
// 簡略化したイメージコード
const consumeMutation = useMutation({
  mutationFn: consumeItemAPI,
  onMutate: async (consumedItem) => {
    // 1. 通信前に、既存のキャッシュ取得をキャンセル（競合防止）
    await queryClient.cancelQueries({ queryKey: ['inventory'] })
    
    // 2. 現在の状態をバックアップ（エラー時のロールバック用）
    const previousInventory = queryClient.getQueryData(['inventory'])

    // 3. ✨ ネットワーク通信の完了を待たずに、UI上のデータだけ先にお肉を「消費済み」に書き換える（楽観的更新）
    queryClient.setQueryData(['inventory'], (old) => overrideData(old, consumedItem))

    return { previousInventory }
  },
  onError: (err, newItem, context) => {
    // 万が一、ネットワークエラーが出た場合はバックアップから復元する
    queryClient.setQueryData(['inventory'], context.previousInventory)
  },
  onSettled: () => {
    // 最終的に正しいデータを取り直す
    queryClient.invalidateQueries({ queryKey: ['inventory'] })
  },
})
```

この実装により、ユーザーから見ると**「タップした瞬間にアニメーションとともにアイテムが消費される」**ため、サーバーとの通信ラグ（200ms~1000ms）を一切感じさせない体験を実現しています。
