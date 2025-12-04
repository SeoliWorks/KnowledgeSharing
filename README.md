----

▷reactとNext.jsでなんらかのサービスを始めるとして、<br>
１番手軽に試せるサービスを考えてからそれを実装するソースコードを提供してほしい。<br>
ただし、ソースコードの生成後にはレビューと改善を３回繰り返してね。

----

▶ReactとNext.jsを使って「一番手軽に試せて、かつ実用的なサービス」を作るなら、「迷った時の決断サポーター（ランチ何食べる？）」というアプリをおすすめします。

これは、ユーザーがいくつかの選択肢（例：「カレー」「ラーメン」「ハンバーグ」）を入力すると、アプリがランダムで1つを選んでくれるシンプルなツールです。
データベース不要（ブラウザだけで完結）で実装でき、Next.jsの基本（State管理、リスト表示、イベントハンドリング）を学ぶのに最適です。

では、まずは\*\*「初期バージョンのソースコード」**を作成し、そこから指示通り**「レビューと改善を3回」\*\*繰り返して、完成度を高めていきます。

-----

### ステップ1：初期構想とベースコード (v0.1)

まずは最低限の機能（入力して、ボタンを押すと、結果が出る）だけを実装します。

**ファイル構成:** `app/page.tsx` (Next.js App Router)

```tsx
"use client";
import { useState } from 'react';

export default function Home() {
  const [input, setInput] = useState('');
  const [options, setOptions] = useState<string[]>([]);
  const [result, setResult] = useState<string | null>(null);

  const addOption = () => {
    setOptions([...options, input]);
    setInput('');
  };

  const decide = () => {
    const randomIndex = Math.floor(Math.random() * options.length);
    setResult(options[randomIndex]);
  };

  return (
    <div className="p-10 max-w-md mx-auto">
      <h1 className="text-2xl font-bold mb-4">決断サポーター</h1>
      
      <div className="flex gap-2 mb-4">
        <input 
          type="text" 
          value={input}
          onChange={(e) => setInput(e.target.value)}
          className="border p-2 flex-grow text-black"
          placeholder="選択肢を入力"
        />
        <button onClick={addOption} className="bg-blue-500 text-white p-2 rounded">追加</button>
      </div>

      <ul className="mb-4 list-disc pl-5">
        {options.map((opt, i) => <li key={i}>{opt}</li>)}
      </ul>

      <button 
        onClick={decide} 
        disabled={options.length === 0}
        className="bg-green-500 text-white w-full p-3 rounded font-bold disabled:opacity-50"
      >
        決める！
      </button>

      {result && (
        <div className="mt-6 p-4 bg-yellow-100 border-2 border-yellow-400 text-center text-xl font-bold text-black">
          答えは... {result} です！
        </div>
      )}
    </div>
  );
}
```

-----

### サイクル1：レビューと改善 (UXの向上)

**🤖 AIレビュアーの視点:**

> 「動くことは動きますが、使い勝手が悪いです。
>
> 1.  空文字でも追加できてしまう。
> 2.  入力欄でEnterキーを押しても追加されない（いちいちボタンを押す必要がある）。
> 3.  同じ選択肢を重複して登録できてしまう。」

**🛠️ 改善の実装 (v0.2):**

  * **バリデーション追加:** 空文字と重複を禁止。
  * **キーボード操作:** `onKeyDown` でEnterキー対応。

<!-- end list -->

```tsx
// 変更点のみ抜粋
const addOption = () => {
  if (!input.trim()) return; // 空文字防止
  if (options.includes(input.trim())) return; // 重複防止
  setOptions([...options, input.trim()]);
  setInput('');
};

const handleKeyDown = (e: React.KeyboardEvent) => {
  if (e.key === 'Enter') addOption();
};

// JSX部分のinputに追加
<input 
  // ...略
  onKeyDown={handleKeyDown}
/>
```

-----

### サイクル2：レビューと改善 (演出と管理機能)

**🤖 AIレビュアーの視点:**

> 「機能は良くなりましたが、結果があっさり出すぎて『決断した感』が薄いです。
> また、間違えて追加した項目を削除できません。リストが増えると不便です。」

**🛠️ 改善の実装 (v0.3):**

  * **ドラムロール演出:** 結果が出る前に少しパラパラと選択肢が切り替わるアニメーションを追加してワクワク感を出す。
  * **削除機能:** リストの各項目に「削除」ボタンを追加。

<!-- end list -->

```tsx
// 変更・追加ロジック
const [isThinking, setIsThinking] = useState(false);

const decide = () => {
  setIsThinking(true);
  let count = 0;
  // パラパラ演出（簡易ルーレット）
  const interval = setInterval(() => {
    const randomIndex = Math.floor(Math.random() * options.length);
    setResult(options[randomIndex]);
    count++;
    if (count > 10) { // 10回切り替わったらストップ
      clearInterval(interval);
      setIsThinking(false);
    }
  }, 100);
};

const removeOption = (indexToRemove: number) => {
  setOptions(options.filter((_, i) => i !== indexToRemove));
};
```

-----

### サイクル3：レビューと改善 (UI/UXの洗練と永続化)

**🤖 AIレビュアーの視点:**

> 「だいぶ良くなりました。しかし、ページをリロードするとデータが消えてしまうのは不便です（毎回ランチの候補を入力するのは面倒）。
> あと、UIが少し質素すぎます。もう少し現代的なデザイン（Tailwind CSSの活用）と、空の状態の表示（Empty State）があると親切です。」

**🛠️ 改善の実装 (v1.0 Final):**

  * **ローカルストレージ保存:** 次回アクセス時も選択肢を覚えておく。
  * **UIブラッシュアップ:** カードデザイン、アイコン、空の状態の案内を追加。
  * **全消去機能:** リセットボタンの追加。

-----

### 🚀 完成版ソースコード

以下が、3回の改善サイクルを経た完成版コードです。
これを `app/page.tsx` にコピペするだけで動作します。（※Lucideアイコンを使用するため、事前に `npm install lucide-react` を実行してください。もしインストールしたくない場合は、アイコン部分を文字に置き換えてください）

```tsx
"use client";

import { useState, useEffect, useRef } from 'react';
import { Trash2, Plus, Sparkles, RefreshCcw, X } from 'lucide-react';

export default function DecisionMaker() {
  // State管理
  const [input, setInput] = useState('');
  const [options, setOptions] = useState<string[]>([]);
  const [result, setResult] = useState<string | null>(null);
  const [isThinking, setIsThinking] = useState(false);
  
  // 初回マウント時にローカルストレージから読み込み
  useEffect(() => {
    const saved = localStorage.getItem('decisionOptions');
    if (saved) {
      setOptions(JSON.parse(saved));
    }
  }, []);

  // optionsが変わるたびに保存
  useEffect(() => {
    localStorage.setItem('decisionOptions', JSON.stringify(options));
  }, [options]);

  // 選択肢を追加
  const addOption = () => {
    const trimmed = input.trim();
    if (!trimmed) return;
    if (options.includes(trimmed)) {
      alert('その選択肢は既にあります');
      return;
    }
    setOptions([...options, trimmed]);
    setInput('');
    setResult(null); // 新しい候補を入れたら結果をリセット
  };

  // Enterキー対応
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter') addOption();
  };

  // 選択肢を削除
  const removeOption = (indexToRemove: number) => {
    setOptions(options.filter((_, i) => i !== indexToRemove));
  };

  // 全消去
  const clearAll = () => {
    if (confirm('全ての選択肢を削除しますか？')) {
      setOptions([]);
      setResult(null);
    }
  };

  // 決断ロジック（ルーレット演出付き）
  const decide = () => {
    if (options.length === 0) return;
    
    setIsThinking(true);
    setResult(null);
    
    let count = 0;
    const maxCount = 15; // ルーレットの回転数
    
    const interval = setInterval(() => {
      const randomIndex = Math.floor(Math.random() * options.length);
      setResult(options[randomIndex]);
      count++;
      
      if (count >= maxCount) {
        clearInterval(interval);
        setIsThinking(false);
        // 最終決定（もう一度ランダムにするか、最後に見えたものにするか。ここでは最後に改めて抽選）
        const finalIndex = Math.floor(Math.random() * options.length);
        setResult(options[finalIndex]);
      }
    }, 80); // 回転速度
  };

  return (
    <div className="min-h-screen bg-gray-50 flex items-center justify-center p-4">
      <div className="bg-white max-w-md w-full rounded-xl shadow-lg overflow-hidden border border-gray-100">
        
        {/* ヘッダー */}
        <div className="bg-indigo-600 p-6 text-center">
          <h1 className="text-2xl font-bold text-white flex items-center justify-center gap-2">
            <Sparkles className="w-6 h-6" />
            迷い解消くん
          </h1>
          <p className="text-indigo-100 text-sm mt-1">AI...ではなくランダム運任せ</p>
        </div>

        <div className="p-6">
          {/* 結果表示エリア */}
          <div className={`mb-6 p-6 rounded-lg text-center transition-all duration-300 ${
            result 
              ? 'bg-indigo-50 border-2 border-indigo-200 scale-100' 
              : 'bg-gray-50 border border-gray-100'
          }`}>
            <p className="text-gray-500 text-sm mb-1">
              {isThinking ? '抽選中...' : result ? '運命の選択は...' : 'ここに結果が出ます'}
            </p>
            <p className={`text-2xl font-bold ${isThinking ? 'text-gray-400' : 'text-indigo-600'}`}>
              {result || '???'}
            </p>
          </div>

          {/* 入力エリア */}
          <div className="flex gap-2 mb-6">
            <input 
              type="text" 
              value={input}
              onChange={(e) => setInput(e.target.value)}
              onKeyDown={handleKeyDown}
              className="flex-grow border border-gray-300 rounded-lg px-4 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500 text-gray-800"
              placeholder="選択肢（例：カレー）"
            />
            <button 
              onClick={addOption}
              disabled={!input.trim()}
              className="bg-gray-900 text-white p-2 rounded-lg hover:bg-gray-700 disabled:opacity-50 transition"
            >
              <Plus className="w-6 h-6" />
            </button>
          </div>

          {/* リスト表示エリア */}
          <div className="mb-6">
            <div className="flex justify-between items-center mb-2">
              <span className="text-sm font-bold text-gray-500">エントリーリスト ({options.length})</span>
              {options.length > 0 && (
                <button onClick={clearAll} className="text-xs text-red-400 hover:text-red-600">
                  全削除
                </button>
              )}
            </div>
            
            <ul className="space-y-2 max-h-48 overflow-y-auto pr-1">
              {options.length === 0 && (
                <li className="text-center py-4 text-gray-400 text-sm border-2 border-dashed border-gray-200 rounded-lg">
                  選択肢を追加してください
                </li>
              )}
              {options.map((opt, i) => (
                <li key={i} className="flex justify-between items-center bg-gray-50 px-3 py-2 rounded border border-gray-100 group hover:border-indigo-100 transition">
                  <span className="text-gray-800">{opt}</span>
                  <button 
                    onClick={() => removeOption(i)} 
                    className="text-gray-400 hover:text-red-500 opacity-0 group-hover:opacity-100 transition"
                  >
                    <X className="w-4 h-4" />
                  </button>
                </li>
              ))}
            </ul>
          </div>

          {/* アクションボタン */}
          <button 
            onClick={decide} 
            disabled={options.length < 2 || isThinking}
            className={`w-full py-4 rounded-xl font-bold text-lg flex items-center justify-center gap-2 shadow-md transition-all ${
              options.length < 2 
                ? 'bg-gray-200 text-gray-400 cursor-not-allowed' 
                : 'bg-indigo-600 text-white hover:bg-indigo-700 hover:shadow-lg active:scale-95'
            }`}
          >
            {isThinking ? (
              <RefreshCcw className="w-5 h-5 animate-spin" />
            ) : (
              <Sparkles className="w-5 h-5" />
            )}
            {isThinking ? '選び中...' : '決める！'}
          </button>
        </div>
      </div>
    </div>
  );
}
```

-----

### 次にできること

このコードはNext.jsの基礎（状態管理、副作用、条件付きレンダリング）が詰まっています。このコードをベースにして、次は以下のような機能追加に挑戦してみませんか？

1.  **履歴機能:** 過去に選ばれた結果をリストで下に表示する。
2.  **シェア機能:** 「今日のランチは〇〇に決まりました！」とX (Twitter) に投稿するボタンを作る。

ご希望があれば、これらの機能追加のコードも作成します。
