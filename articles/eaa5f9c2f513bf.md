---
title: "Dexie.js のテストは fake-indexeddb におまかせ"
emoji: "🗄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dexie", "indexeddb", "vitest", "jest", "testing"]
published: true
---

## はじめに

ブラウザの IndexedDB を手軽に扱えるライブラリ [Dexie.js](https://dexie.org/) は、PWA やオフライン対応アプリでよく使われます。しかし「Dexie のロジックをユニットテストしたい」となったとき、**Node.js 環境には IndexedDB API が存在しない**という壁にぶつかります。

この記事では、その問題をすっきり解決してくれる `fake-indexeddb` を中心に、Dexie.js のテスト戦略を紹介します。

---

## fake-indexeddb とは

[fake-indexeddb](https://github.com/dumbmatter/fakeIndexedDB) は、IndexedDB API を純粋な JavaScript で再実装したポリフィルです。Node.js 上でも IndexedDB と同じ API が使えるようになるため、Vitest や Jest によるユニットテストが可能になります。

データはディスクには保存されず、メモリ上にのみ存在します。テストごとにクリーンな状態を保てるため、テストの独立性も担保しやすいのが特徴です。

---

## セットアップ

### インストール

```bash
npm install -D fake-indexeddb
```

### Vitest の場合

`vitest.config.ts` の `setupFiles` に追加するだけで、全テストへ自動適用されます。

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    setupFiles: ["fake-indexeddb/auto"],
  },
});
```

### Jest の場合

同様に `jest.config.ts` の `setupFiles` に追加します。

```ts
// jest.config.ts
export default {
  setupFiles: ["fake-indexeddb/auto"],
};
```

---

## Dexie との組み合わせ方

### パターン① auto import（グローバル適用）

setupFiles への追加だけで済むため、最もシンプルな方法です。

```ts
import "fake-indexeddb/auto";
import Dexie from "dexie";

const db = new Dexie("MyDatabase");
```

### パターン② コンストラクタに明示的に渡す

テストファイル単位で細かく制御したい場合に有効です。

```ts
import Dexie from "dexie";
import { IDBKeyRange, indexedDB } from "fake-indexeddb";

const db = new Dexie("MyDatabase", { indexedDB, IDBKeyRange });
```

---

## テストの書き方

テストごとに DB を再生成することで、データが汚染されない独立したテストを実現できます。

```ts
// db.ts
import Dexie, { type EntityTable } from "dexie";

interface Item {
  id: number;
  name: string;
}

const db = new Dexie("MyDatabase") as Dexie & {
  items: EntityTable<Item, "id">;
};

db.version(1).stores({ items: "++id, name" });

export { db };
```

```ts
// db.test.ts
import { beforeEach, describe, expect, it } from "vitest";
import { db } from "./db";

beforeEach(async () => {
  await db.delete();
  await db.open();
});

describe("items テーブル", () => {
  it("アイテムを追加して取得できる", async () => {
    const id = await db.items.add({ name: "テストアイテム" });
    const item = await db.items.get(id);
    expect(item?.name).toBe("テストアイテム");
  });

  it("アイテムを削除できる", async () => {
    const id = await db.items.add({ name: "削除対象" });
    await db.items.delete(id);
    const item = await db.items.get(id);
    expect(item).toBeUndefined();
  });
});
```

---

## E2E テストしたい場合

`liveQuery` のリアクティビティや、実ブラウザ上の動作を確認したい場合は、Playwright や Cypress で本物の IndexedDB を使ったテストが適しています。

| 目的 | 推奨手段 |
|---|---|
| ユニットテスト（CRUD ロジック） | Vitest / Jest + `fake-indexeddb` |
| リアクティビティ・UI 統合テスト | Playwright / Cypress |

---

## まとめ

- Node.js 環境での Dexie.js テストには **`fake-indexeddb`** を使う
- `setupFiles` に `fake-indexeddb/auto` を追加するだけで導入できる
- `beforeEach` で `db.delete()` → `db.open()` することでテストを独立させる
- E2E が必要な場合は Playwright / Cypress で実ブラウザを使う

シンプルな設定でしっかりテストできるので、ぜひ取り入れてみてください。