# App Router でよくある罠・注意点

## 罠1: `page.tsx` を置かないとURLが存在しない

ディレクトリを作っただけではページになりません。

```
app/
└── about/          ← フォルダだけ作っても
                       /about にアクセスすると404
```

```
app/
└── about/
    └── page.tsx    ← これがないとページとして認識されない
```

---

## 罠2: Server Component と Client Component の混同

App Router のコンポーネントはデフォルトで **Server Component**（サーバー側で実行）です。

`useState` や `useEffect`、クリックイベントはブラウザ側の機能なので、使おうとするとエラーになります。

```tsx
// ❌ これはエラー。useStateはクライアントでしか動かない
export default function Counter() {
  const [count, setCount] = useState(0);  // エラー！
}
```

```tsx
// ✅ ファイルの先頭に "use client" を書けばOK
"use client";

export default function Counter() {
  const [count, setCount] = useState(0);  // OK
}
```

**判断基準：**

| やりたいこと | どちらか |
|---|---|
| DBからデータ取得、重い処理 | Server Component（デフォルト） |
| ボタン・フォーム・アニメーション | `"use client"` が必要 |

---

## 罠3: `layout.tsx` の中で状態管理しようとする

`layout.tsx` は Server Component なので `useState` が使えません。  
ログイン状態などをレイアウトで管理したい場合は、Client Component を別ファイルに切り出してラップします。

```tsx
// ❌ layout.tsx に直接 useState は書けない
export default function RootLayout({ children }) {
  const [isOpen, setIsOpen] = useState(false); // エラー！
}
```

---

## 罠4: 動的ルートのフォルダ名を間違える

`[id]` の角カッコを忘れると、動的URLではなく固定URLになります。

```
app/blog/id/page.tsx    ← /blog/id という固定URLになってしまう
app/blog/[id]/page.tsx  ← /blog/123 や /blog/hello などが動的に対応
```

---

## 罠5: `layout.tsx` で `<html>` と `<body>` を二重に書いてしまう

サブの `layout.tsx` で `<html>`・`<body>` を書くとHTMLが壊れます。

```tsx
// ❌ app/dashboard/layout.tsx でこれはダメ
export default function DashboardLayout({ children }) {
  return (
    <html>   {/* <html>はapp/layout.tsxにだけ書く */}
      <body>{children}</body>
    </html>
  );
}
```

```tsx
// ✅ サブレイアウトはdivなどで包むだけでOK
export default function DashboardLayout({ children }) {
  return (
    <div className="flex">
      <aside>サイドバー</aside>
      <main>{children}</main>
    </div>
  );
}
```

---

## まとめ

| 罠 | 対策 |
|---|---|
| ページが404になる | `page.tsx` を置き忘れていないか確認 |
| `useState` 等がエラー | ファイル先頭に `"use client"` を追加 |
| 動的URLが動かない | フォルダ名が `[id]` になっているか確認 |
| HTMLが壊れる | `<html><body>` は `app/layout.tsx` にだけ書く |

> 最初のうちは **「インタラクティブな操作が必要なら `"use client"`」** だけ覚えておくと、多くのエラーを回避できます。

---

# Server Component と Client Component を深く理解する

## Server Component と Client Component の技術的な違い

### 「どこで実行されるか」が根本的な違い

```
【Server Component】
  サーバー ──→ HTML を生成 ──→ ブラウザに送る
  （ブラウザには完成したHTMLが届く）

【Client Component】
  サーバー ──→ JS を送る ──→ ブラウザ上でコンポーネントを実行してHTMLを生成
```

### 具体的に何が違うか

#### 1. JavaScript のダウンロード量

**Server Component** はサーバーで実行済みなので、**そのコンポーネントの JS コードはブラウザに送られません**。

```tsx
// Server Component
// このコードはサーバーで実行され、結果のHTMLだけがブラウザに届く
// marked（重いライブラリ）のJSはブラウザにダウンロードされない
import { marked } from "marked"; // 重いライブラリ

export default function Article({ content }) {
  return <div>{marked(content)}</div>;
}
```

**Client Component** はブラウザに JS ごと送られるので、ライブラリも含めてダウンロードされます。

#### 2. アクセスできるものが異なる

| できること | Server Component | Client Component |
|---|---|---|
| DB に直接アクセス | ✅ | ❌ |
| ファイルシステム読み込み | ✅ | ❌ |
| `useState` / `useEffect` | ❌ | ✅ |
| ブラウザAPI (`window`, `document`) | ❌ | ✅ |
| クリック・入力イベント | ❌ | ✅ |
| 環境変数（シークレット）を安全に使う | ✅ | ❌（漏洩リスク） |

#### 3. レンダリングのタイミング

```
Server Component:
  リクエスト発生 → サーバーで実行 → HTMLとして返す
  ※ページを開くたびにサーバーで動く

Client Component:
  ① 初回: サーバーでHTMLの骨格だけ生成（SSR）
  ② ブラウザで JS をダウンロード・実行（Hydration）
  ③ 以降: ブラウザ内でインタラクションに応じて再レンダリング
```

---

## Hydration（ハイドレーション）とは

Client Component には「乾いたHTML（見た目だけ）に水（JS）を注いで動くようにする」工程があります。

```
サーバー → <button>+1</button>（見た目だけ、クリックしても何も起きない）
     ↓ JSダウンロード・Hydration
ブラウザ → <button>+1</button>（クリックできるようになる）
```

Hydration が完了するまでの間、ボタンが押せない「ガワだけの状態」が一瞬存在します。  
JSが重い場合はこの期間が長くなり、**TTI（Time To Interactive）遅延** と呼ばれます。

---

## Client Component でないとダメなパターン

### ユーザーの操作に反応する必要があるとき

```tsx
"use client";
// ボタンクリック・入力・トグルなど状態が変わるもの
const [isOpen, setIsOpen] = useState(false);
const [value, setValue] = useState("");
```

具体例：ハンバーガーメニューの開閉、タブ切り替え、モーダルの表示・非表示、入力フォームのリアルタイムバリデーション

### ブラウザの機能を使うとき

```tsx
"use client";
// これらはブラウザにしか存在しない
window.scrollTo(0, 0);
localStorage.setItem("key", "value");
navigator.geolocation.getCurrentPosition(...);
document.title = "新しいタイトル";
```

### ライフサイクル・副作用が必要なとき

```tsx
"use client";
useEffect(() => {
  const timer = setInterval(...);
  return () => clearInterval(timer); // クリーンアップ
}, []);
```

具体例：ページ表示時にアニメーションを起動、WebSocket接続の管理、外部ライブラリ（地図・グラフ等）の初期化

### サードパーティライブラリが `"use client"` を要求するとき

```tsx
"use client";
import { motion } from "framer-motion";   // アニメーションライブラリ
import { useForm } from "react-hook-form"; // フォームライブラリ
import { Toaster } from "react-hot-toast"; // トースト通知
```

ブラウザ前提で作られたライブラリはほぼ全てClient Component必須です。

---

## Server Component でないとダメなパターン

### DBやファイルに直接アクセスするとき

```tsx
// "use client" なし = Server Component
import { db } from "@/lib/db";

export default async function UserList() {
  const users = await db.user.findMany(); // DBに直接アクセス
  return <ul>{users.map(u => <li>{u.name}</li>)}</ul>;
}
```

Client Componentでやろうとするとシークレットキーがブラウザに漏れます。

### APIキー・シークレットを使うとき

```tsx
// Server Component なら環境変数のシークレットが漏れない
const data = await fetch("https://api.example.com/data", {
  headers: {
    Authorization: `Bearer ${process.env.SECRET_API_KEY}`, // ブラウザに送られない
  },
});
```

### SEOのためにサーバー側でHTMLを確定させたいとき

```tsx
// OGPやmeta情報はサーバーで生成しないと検索エンジンに読まれない
export async function generateMetadata({ params }) {
  const post = await getPost(params.id);
  return {
    title: post.title,
    openGraph: { images: [post.thumbnail] },
  };
}
```

---

## 判断フローチャート

```
コンポーネントを作るとき
        ↓
useState / useEffect を使う？ → YES → "use client"
        ↓ NO
クリック・入力イベントがある？ → YES → "use client"
        ↓ NO
window / localStorage を使う？ → YES → "use client"
        ↓ NO
DBやシークレットキーを使う？ → YES → Server Component（デフォルト）
        ↓ NO
        ↓
   どちらでもOK（Server Componentのままが推奨）
```

---

## 実際のページでの分担イメージ

```tsx
// app/blog/[id]/page.tsx（Server Component）
export default async function BlogPage({ params }) {
  const post = await db.post.findById(params.id); // DB直接アクセス

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
      <LikeButton postId={post.id} />  {/* ← ここだけClient Component */}
    </article>
  );
}
```

```tsx
// components/LikeButton.tsx
"use client"; // いいねボタンはクリックが必要なのでClient
export function LikeButton({ postId }) {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(true)}>❤️</button>;
}
```

**ページの大部分はServer Component、動く部分だけClient Component** に切り出すのが理想の構成です。

---

## ファイル名のルール

Next.js が認識する特殊なファイル名は決まっています。それ以外の名前は URLになりません。

| ファイル名 | 役割 |
|---|---|
| `page.tsx` | URLとして公開されるページ本体 |
| `layout.tsx` | そのディレクトリ以下の共通外枠 |
| `loading.tsx` | データ取得中に表示するUI |
| `error.tsx` | エラー時に表示するUI |
| `not-found.tsx` | 404時に表示するUI |
| `route.ts` | API エンドポイント（Route Handlers） |

### コンポーネントの置き場所

```
app/
├── page.tsx              ← / のページ（URLになる）
└── dashboard/
    └── page.tsx          ← /dashboard のページ（URLになる）

components/               ← 自作コンポーネントはここ（URLにならない）
├── Button.tsx
└── UserCard.tsx

lib/                      ← ユーティリティ関数・DBクライアントなど
└── db.ts
```

> `lib/` に置くのは JSX を返さない関数・クラス・設定です。Server/Client Componentという分類は当てはまらず、**呼び出し元がServer ComponentならサーバーでClientComponentならブラウザで動きます。**

---

## 静的export（output: 'export'）での Server Component

静的exportを設定した場合、Server Component は**ビルド時に1回だけ実行**されます。

```
【通常のNext.js（サーバーあり）】
  ユーザーがアクセス → サーバーでServer Componentを実行 → HTMLを返す

【静的export（output: 'export'）】
  npm run build のとき（1回だけ）Server Componentを実行
  → .html ファイルとして保存
  → ユーザーがアクセス → 保存済みのHTMLをそのまま返す
```

### 静的exportで使えなくなる機能

| 機能 | 理由 |
|---|---|
| `cookies()` / `headers()` | リクエストのたびに変わる情報はビルド時に確定できない |
| `revalidate`（定期的な再生成） | サーバーが常駐していないと動かない |
| Route Handlers（`route.ts`） | サーバーがいないのでAPIエンドポイントを作れない |
| Server Actions | サーバーがいないので実行できない |

### 通常Next.js vs 静的exportの使い分け

| | 通常Next.js | 静的export |
|---|---|---|
| Server Component | リクエストのたびに実行 | ビルド時に1回だけ実行 |
| Client Component | 変わらず動く | 変わらず動く |
| Server Actions / Route Handlers | 使える | **使えない** |
| 向いているサービス | ECサイト・SNS等 | ブログ・ランディングページ等 |
