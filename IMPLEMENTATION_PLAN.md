# Nuxt Content v3 個人ブログ 実装計画

## 技術スタック

| レイヤー | 技術 | 備考 |
|---------|------|------|
| フレームワーク | Nuxt 3 | |
| コンテンツ管理 | @nuxt/content v3 | Git ベース CMS — 原本は Markdown ファイル |
| ホスティング | Cloudflare Pages | |
| D1 (Content 内部用) | Cloudflare D1 | Nuxt Content がビルド成果物のクエリ用に内部利用（自分で設計不要） |
| D1 (閲覧数用) | Cloudflare D1 | 閲覧数カウント用（自前で設計。同一 D1 インスタンスに別テーブル） |
| スタイリング | 未定（Tailwind CSS 推奨） | |

> **補足: Nuxt Content v3 のデータフロー（公式ドキュメントより）**
>
> Nuxt Content は **git-based CMS** であり、コンテンツの原本は Git リポジトリ内の Markdown ファイルです。
> DB はコンテンツの原本ではなく、以下の 3 ステップで内部的に利用されるクエリキャッシュです:
>
> 1. **ビルド時**: 各コレクションの Markdown を解析 → AST に変換 → スキーマに基づくテーブルに格納 → ダンプファイルとして保存
> 2. **ランタイム (コールドスタート)**: 最初のクエリ実行時にダンプを DB に復元（整合性チェック付き）
> 3. **ブラウザ (クライアント側)**: 最初のクエリ時にダンプをダウンロード → ブラウザ内 WASM SQLite で以降のクエリをローカル実行
>
> コンテンツ用の DB 設計やマイグレーションを自分で行う必要はありません。
> 自前で D1 のテーブル設計が必要なのは **閲覧数カウント (`page_views`) のみ** です。

---

## フェーズ 1: プロジェクト初期セットアップ

### 1-1. Nuxt 3 プロジェクト作成

```bash
npx nuxi@latest init blog
cd blog
```

### 1-2. Nuxt Content v3 インストール

```bash
# パッケージマネージャーでインストール
pnpm add @nuxt/content

# nuxt.config.ts の modules に追加
# (nuxi module add content でも可)
```

`nuxt.config.ts`:

```ts
export default defineNuxtConfig({
  modules: ['@nuxt/content'],
})
```

> **注意 (pnpm v10+)**: `better-sqlite3` のネイティブビルドが必要。`pnpm approve-builds` を実行するか、
> `package.json` に以下を追加:
> ```json
> { "pnpm": { "onlyBuiltDependencies": ["better-sqlite3"] } }
> ```

### 1-3. app.vue の更新

`pages/` ディレクトリを使用するために `app.vue` を更新:

```vue
<!-- app.vue -->
<template>
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

### 1-4. content.config.ts の作成

ブログ記事用のコレクションを定義する。`type: 'page'` はコンテンツファイルとページが 1:1 対応することを意味する。

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
        date: z.date(),
        tags: z.array(z.string()).optional(),
        image: z.string().optional(),
        draft: z.boolean().default(false),
      }),
    }),
  },
})
```

> **補足**: `title` と `description` は `type: 'page'` のビルトインフィールドとして自動提供される。
> `content.config.ts` が存在する場合、定義されたパターンに一致するファイルのみがインポートされる。

### 1-5. サンプル記事の作成

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
    preset: 'cloudflare_pages',
  },
})
```

### 3-2. D1 データベースの作成（Cloudflare ダッシュボード or CLI）

```bash
# Wrangler CLI で D1 データベースを作成
npx wrangler d1 create blog-db
```

### 3-3. Cloudflare ダッシュボードで D1 を Pages プロジェクトにバインド

Cloudflare ダッシュボード → Pages プロジェクト → Settings → Bindings から、
作成した D1 データベースをバインディング名 **`DB`** で接続する。

> **注意**: Nuxt Content v3 はデフォルトで `DB` というバインディング名を使用する。

### 3-4. ローカルプレビュー用の wrangler.jsonc

`nuxi dev` と `nuxi build` は追加設定不要。
`nuxi preview` でビルド結果をローカルテストする場合のみ、Wrangler 設定が必要:

```jsonc
// wrangler.jsonc
{
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "blog-db",
      "database_id": "local-dev-id"  // ローカル用なので任意の値でOK
    }
  ]
}
```

```bash
nuxi build
npx wrangler pages dev .output/public
```

---

## フェーズ 4: 閲覧数カウント機能

Nuxt Content が内部利用する D1 にはコンテンツのクエリ用テーブルが自動生成される。
閲覧数カウントは **同じ D1 インスタンスに自前のテーブルを追加** して実装する。
（Content の内部テーブルとは独立しており、干渉しない）

### 4-1. 方式の選択

| 方式 | メリット | デメリット |
|------|---------|-----------|
| **A: 同一 D1 に別テーブル + Nitro API** | シンプル、追加コスト無し、バインディング追加不要 | Content 内部テーブルと同居 |
| **B: 別 D1 インスタンス + Nitro API** | 完全な分離 | バインディングが増える |
| **C: Cloudflare Analytics Engine** | 集計に特化、高パフォーマンス | API が別体系 |

**推奨: 方式 A**（Content の内部テーブルとは干渉しないため、初期段階はこれで十分）

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
  const { DB } = event.context.cloudflare.env

  if (event.method === 'POST') {
    // UPSERT: 閲覧数をインクリメント
    await DB.prepare(`
      INSERT INTO page_views (path, count, updated_at)
      VALUES (?, 1, datetime('now'))
      ON CONFLICT(path) DO UPDATE SET
        count = count + 1,
        updated_at = datetime('now')
    `).bind(path).run()

    return { success: true }
  }

  // GET: 閲覧数を取得
  const row = await DB.prepare(
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
├── app.vue
├── content.config.ts
├── nuxt.config.ts
├── wrangler.jsonc          # ローカルプレビュー用（本番はダッシュボードで設定）
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
