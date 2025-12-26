----
▷eslint-plugin-simple-import-sortとは？
----
<br>

▷`eslint-plugin-simple-import-sort` は、ファイル上部にある **`import` 文（と `export` 文）を、自動的かつ強制的に並べ替えて整理してくれる ESLint のプラグイン**です。

一言で言えば、**「import文の整頓係」**です。

開発が進むと、以下のようにimport文が散らかってしまうことがよくあります。

### ❌ 導入前の状態（Before）

ライブラリ、自作コンポーネント、画像ファイル、CSSなどが無秩序に並んでいます。

```typescript
import { Button } from "@/components/ui/button"; // 自作コンポーネント
import { useState } from "react"; // フレームワーク標準
import { z } from "zod"; // 外部ライブラリ
import { cn } from "@/lib/utils"; // ユーティリティ
import "./global.css"; // スタイル
import { Icon } from "lucide-react"; // 外部ライブラリ

```

これを、`eslint-plugin-simple-import-sort` を導入して保存（または `--fix` 実行）すると、以下のようになります。

### ✅ 導入後の状態（After）

ルールに基づいて、種類ごとにグループ化され、アルファベット順に整列します。

```typescript
// 1. フレームワーク・外部ライブラリ (react, next, zodなど)
import { Icon } from "lucide-react";
import { useState } from "react";
import { z } from "zod";

// 2. 絶対パスインポート (@/...)
import { Button } from "@/components/ui/button";
import { cn } from "@/lib/utils";

// 3. スタイル (css)
import "./global.css";

```

---

### なぜこれが重要なのか？（メリット）

1. **コードの可読性向上**
* 「どのライブラリを使っているか」「どのコンポーネントに依存しているか」がひと目で分かります。


2. **チーム開発での無駄な議論の排除**
* 「Reactのimportは一番上に書くべきか？」「アルファベット順にするべきか？」といった、本質的でない議論やレビューの指摘を自動化できます。


3. **マージ競合（コンフリクト）の軽減**
* 常に同じルールで並び替えられるため、複数人が同じファイルにimportを追加した際、行が入り乱れてコンフリクトするリスクを減らせます。



### 補足：なぜ "Simple" なのか？

他にも `eslint-plugin-import` という有名なプラグインがありますが、設定が非常に複雑で難しいという欠点がありました。
この `simple-import-sort` は、**「細かい設定は不要。プラグインが推奨する『一番良い並び順』に全部任せる」** という思想で作られているため、導入が非常に簡単です。

---

今回のようなモダンなスタック（Next.js + TypeScript）では、ファイルの先頭にimportが数十行並ぶことも珍しくないため、**ほぼ必須レベルで導入されているツール**です。

----
▷eslint-plugin-simple-import-sortの導入方法をまとめて
----
<br>

▷`eslint-plugin-simple-import-sort` を、今回の技術スタック（Next.js + TypeScript）に導入する手順をまとめました。

手順は大きく分けて **「1. インストール」「2. 設定ファイルの編集」「3. 自動整形の有効化（VS Code）」** の3ステップです。

---

### ステップ 1: パッケージのインストール

お使いのパッケージマネージャーに合わせて、以下のコマンドを実行してください。

```bash
# npm の場合
npm install --save-dev eslint-plugin-simple-import-sort

# pnpm の場合
pnpm add -D eslint-plugin-simple-import-sort

# yarn の場合
yarn add -D eslint-plugin-simple-import-sort

```

---

### ステップ 2: ESLint 設定ファイルの編集

プロジェクトルートにある `.eslintrc.json`（または `.eslintrc.js`）を開き、以下の **`plugins`** と **`rules`** の設定を追加します。

Next.jsのデフォルト設定を残しつつ、追加する形になります。

```json
{
  "extends": [
    "next/core-web-vitals"
    // 他の設定があればそのまま維持
  ],
  "plugins": [
    "simple-import-sort" // 👈 これを追加
  ],
  "rules": {
    // 👇 以下の2行を追加（importとexportの並び替えをエラーとして扱う）
    "simple-import-sort/imports": "error",
    "simple-import-sort/exports": "error"
  }
}

```

> **解説:** これで、「並び順が整理されていない状態」が「エラー（赤い波線）」として扱われるようになります。

---

### ステップ 3: 保存時に自動修正されるようにする（推奨）

これだけだと「並び順が違う！」と怒られるだけなので、**ファイル保存時（Ctrl+S / Cmd+S）に勝手に直してくれる設定** をVS Codeに追加します。

プロジェクト直下の `.vscode` フォルダ（なければ作成）の中に `settings.json` を作成し、以下を記述します。

**ファイルパス:** `.vscode/settings.json`

```json
{
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  }
}

```

> **解説:** これで、ファイルを保存するたびに ESLint が走り、`simple-import-sort` のルールに従って import 文がシャキッと整列するようになります。

---

### 動作確認

適当な `.ts` や `.tsx` ファイルを開き、import文の順番をわざとぐちゃぐちゃにして保存してみてください。

**Before (保存前):**

```typescript
import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { z } from 'zod';

```

**After (保存後):**

```typescript
import { useState } from 'react';
import { z } from 'zod'; // 外部ライブラリがまとまる

import { Button } from '@/components/ui/button'; // 内部コンポーネントが空行を挟んで続く

```

このように自動で並び変わり、グループ分けされれば導入成功です。

### 補足: エラーが出る場合

もし `eslint-plugin-simple-import-sort` 導入後にすべての import 文がエラーになり、修正もされない場合は、VS Codeの ESLint 拡張機能が有効になっているか、または VS Code を再起動してみてください。
