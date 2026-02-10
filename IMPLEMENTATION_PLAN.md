# Nuxt Content v3 個人ブログ 実装計画

## 技術スタック

| レイヤー | 技術 | 備考 |
|---------|------|------|
| フレームワーク | Nuxt 3 | |
| コンテンツ管理 | @nuxt/content v3 | Markdown + SQL ベース |
| ホスティング | Cloudflare Pages | |
| コンテンツ DB | Cloudflare D1 | Nuxt Content が自動利用 |
| 閲覧数 DB | Cloudflare D1 | 閲覧数カウント用（同一 D1 インスタンス or 別途作成） |
| スタイリング | 未定（Tailwind CSS 推奨） | |

---

## フェーズ 1: プロジェクト初期セットアップ

### 1-1. Nuxt 3 プロジェクト作成

```bash
npx nuxi@latest init blog
cd blog
```

### 1-2. Nuxt Content v3 インストール

```bash
npx nuxi module add content
```

### 1-3. content.config.ts の作成

ブログ記事用のコレクションを定義する。

```ts
// content.config.ts
import { defineCollection, defineContentConfig } from '@nuxt/content'
import { z } from 'zod'

export default defineContentConfig({
  collections: {
    blog: defineCollection({
      type: 'page',
      source: 'blog/**/*.md',
      schema: z.object({
        title: z.string(),
        description: z.string(),
        date: z.date(),
        tags: z.array(z.string()).optional(),
        image: z.string().optional(),
        draft: z.boolean().default(false),
      }),
    }),
  },
})
```

### 1-4. サンプル記事の作成

```
content/
  blog/
    hello-world.md
```

```md
---
title: "Hello World"
description: "最初のブログ記事です"
date: 2026-02-10
tags: ["nuxt", "blog"]
draft: false
---

## はじめに

Nuxt Content v3 を使ったブログへようこそ。
```

---

## フェーズ 2: ページ実装

### 2-1. ブログ一覧ページ (`pages/blog/index.vue`)

```vue
<script setup lang="ts">
const { data: posts } = await useAsyncData('blog-list', () =>
  queryCollection('blog')
    .where('draft', '=', false)
    .order('date', 'DESC')
    .all()
)
</script>

<template>
  <div>
    <h1>Blog</h1>
    <ul>
      <li v-for="post in posts" :key="post.id">
        <NuxtLink :to="post.path">
          <h2>{{ post.title }}</h2>
          <p>{{ post.description }}</p>
          <time>{{ post.date }}</time>
        </NuxtLink>
      </li>
    </ul>
  </div>
</template>
```

### 2-2. ブログ記事ページ (`pages/blog/[...slug].vue`)

```vue
<script setup lang="ts">
const route = useRoute()
const { data: post } = await useAsyncData(route.path, () =>
  queryCollection('blog').path(route.path).first()
)
</script>

<template>
  <article v-if="post">
    <h1>{{ post.title }}</h1>
    <time>{{ post.date }}</time>
    <ContentRenderer :value="post" />
  </article>
</template>
```

### 2-3. トップページ (`pages/index.vue`)

最新記事の一覧を表示し、ブログ一覧へのリンクを設置。

---

## フェーズ 3: Cloudflare デプロイ設定

### 3-1. Cloudflare Pages 用のプリセット設定

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/content'],

  nitro: {
    preset: 'cloudflare-pages',
  },
})
```

### 3-2. D1 データベースの作成（Cloudflare ダッシュボード or CLI）

```bash
# Wrangler CLI で D1 データベースを作成
npx wrangler d1 create blog-db
```

### 3-3. wrangler.toml の設定

```toml
name = "blog"
compatibility_date = "2026-02-10"
pages_build_output_dir = ".output/public"

[[d1_databases]]
binding = "DB"
database_name = "blog-db"
database_id = "<作成時に取得した ID>"
```

> **注意**: Nuxt Content v3 はデフォルトで `DB` というバインディング名を使用する。

### 3-4. ローカルプレビュー

```bash
npx nuxi build
npx wrangler pages dev .output/public
```

---

## フェーズ 4: 閲覧数カウント機能

Cloudflare D1 を使って閲覧数を記録する API を実装する。

### 4-1. 方式の選択

| 方式 | メリット | デメリット |
|------|---------|-----------|
| **A: 同一 D1 + Nitro API** | シンプル、追加コスト無し | Content 用 DB と混在 |
| **B: 別 D1 + Nitro API** | 関心の分離 | バインディングが増える |
| **C: Cloudflare Analytics Engine** | 集計に特化、高パフォーマンス | API が別体系 |

**推奨: 方式 A**（初期段階はシンプルに同一 D1 を利用）

### 4-2. テーブル設計

```sql
CREATE TABLE IF NOT EXISTS page_views (
  path TEXT NOT NULL,
  count INTEGER NOT NULL DEFAULT 0,
  updated_at TEXT NOT NULL DEFAULT (datetime('now')),
  PRIMARY KEY (path)
);
```

### 4-3. API エンドポイント実装

**閲覧数の記録・取得** (`server/api/views/[...path].ts`)

```ts
export default defineEventHandler(async (event) => {
  const path = '/' + (getRouterParam(event, 'path') || '')
  const db = hubDatabase() // NuxtHub 利用時
  // または: const { DB } = event.context.cloudflare.env // 直接バインディング利用時

  if (event.method === 'POST') {
    // UPSERT: 閲覧数をインクリメント
    await db.prepare(`
      INSERT INTO page_views (path, count, updated_at)
      VALUES (?, 1, datetime('now'))
      ON CONFLICT(path) DO UPDATE SET
        count = count + 1,
        updated_at = datetime('now')
    `).bind(path).run()

    return { success: true }
  }

  // GET: 閲覧数を取得
  const row = await db.prepare(
    'SELECT count FROM page_views WHERE path = ?'
  ).bind(path).first()

  return { path, count: row?.count ?? 0 }
})
```

### 4-4. フロントエンド連携

記事ページで閲覧数をインクリメントし、表示する。

```ts
// composables/usePageViews.ts
export function usePageViews(path: string) {
  const count = ref(0)

  onMounted(async () => {
    // 閲覧数をインクリメント
    await $fetch(`/api/views${path}`, { method: 'POST' })
    // 最新の閲覧数を取得
    const res = await $fetch(`/api/views${path}`)
    count.value = res.count
  })

  return { count }
}
```

### 4-5. D1 マイグレーション

本番デプロイ前にテーブルを作成する。

```bash
npx wrangler d1 execute blog-db --command "CREATE TABLE IF NOT EXISTS page_views (path TEXT NOT NULL, count INTEGER NOT NULL DEFAULT 0, updated_at TEXT NOT NULL DEFAULT (datetime('now')), PRIMARY KEY (path));"
```

---

## フェーズ 5: スタイリング・UI

### 5-1. Tailwind CSS の導入（推奨）

```bash
npx nuxi module add @nuxtjs/tailwindcss
```

### 5-2. レイアウト設計

```
layouts/
  default.vue      # ヘッダー・フッター共通レイアウト
components/
  AppHeader.vue     # ナビゲーション
  AppFooter.vue     # フッター
  BlogCard.vue      # 記事一覧用カード
```

---

## フェーズ 6: SEO・メタデータ

### 6-1. useSeoMeta の活用

```ts
useSeoMeta({
  title: post.value?.title,
  description: post.value?.description,
  ogTitle: post.value?.title,
  ogDescription: post.value?.description,
  ogImage: post.value?.image,
})
```

### 6-2. RSS フィード（オプション）

`nitro-plugin` または `server/routes/rss.xml.ts` で生成。

---

## 最終的なディレクトリ構成

```
blog/
├── content/
│   └── blog/
│       ├── hello-world.md
│       └── ...
├── pages/
│   ├── index.vue
│   └── blog/
│       ├── index.vue
│       └── [...slug].vue
├── layouts/
│   └── default.vue
├── components/
│   ├── AppHeader.vue
│   ├── AppFooter.vue
│   └── BlogCard.vue
├── composables/
│   └── usePageViews.ts
├── server/
│   └── api/
│       └── views/
│           └── [...path].ts
├── content.config.ts
├── nuxt.config.ts
├── wrangler.toml
└── package.json
```

---

## 実装順序のまとめ

1. **Nuxt 3 + Content v3 のセットアップ** — プロジェクト作成、モジュール追加、コレクション定義
2. **ページ実装** — 一覧・記事詳細・トップページ
3. **ローカルで動作確認** — `nuxi dev` で Markdown 記事が表示されることを確認
4. **Cloudflare デプロイ設定** — D1 作成、wrangler.toml、ビルド＆プレビュー
5. **閲覧数カウント機能** — D1 テーブル、API エンドポイント、フロントエンド連携
6. **スタイリング** — Tailwind CSS、レイアウト、コンポーネント
7. **SEO 対応** — メタタグ、OGP、RSS（オプション）
8. **本番デプロイ** — Cloudflare Pages にデプロイ

---

## 参考リンク

- [Nuxt Content v3 公式ドキュメント](https://content.nuxt.com/docs/getting-started)
- [Nuxt Content インストールガイド](https://content.nuxt.com/docs/getting-started/installation)
- [Nuxt Content コレクション定義](https://content.nuxt.com/docs/collections/define)
- [Nuxt Content Cloudflare Pages デプロイ](https://content.nuxt.com/docs/deploy/cloudflare-pages)
- [Nuxt Content 設定](https://content.nuxt.com/docs/getting-started/configuration)
- [Cloudflare D1 ドキュメント](https://developers.cloudflare.com/d1/)
- [Cloudflare Workers + D1 閲覧数カウンター例](https://mrinalcs.github.io/cloudflare-workers-d1-view-counter)
