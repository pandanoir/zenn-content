---
title: "なぜ document.title = 'title' は useEffect でラップする必要があるのか"
emoji: "⛩️"
type: "tech"
topics:
  - "react"
published: false
---

A. レンダリングフェーズを純粋にするため (結論)

# useEffect でラップする意味ってなくない?
以下の2つのコードはどちらもレンダリングすると Hello world と表示され、ページタイトルが Hello world になります。

```tsx
const App1 = () => {
  useEffect(() => {
    document.title = 'Hello world';
  });
  return <h1>Hello world</h1>
};
```

```tsx
const App2 = () => {
  document.title = 'Hello world';
  return <h1>Hello world</h1>
};
```

同じ動作をするのであれば、**なぜ useEffect でラップする必要があるのでしょうか?**

# 問題1: レンダリング時に react 以外を考慮する必要が出てくる

useEffect に渡す関数は、[DOM の更新まで完全に終わったあとに呼ばれます](https://github.com/donavon/hook-flow/blob/master/hook-flow.png)。そのため、レンダリング中には useEffect で囲まれた部分について考える必要がありません。つまり **useEffect を使うとレンダリングとエフェクトを完全に分離できるということです。**

たとえば以下のようにキー入力を受け付けるコンポーネントも、useEffect を無視すると DOM 操作などを一切してないことが分かります。レンダリングとエフェクトが完全に分離しているため、レンダリングをシンプルに考えられます。

```tsx
const App = () => {
  const [num, setNum] = useState(0);

  // ↓ レンダリング中は実行されない = 無視して良い
  useEffect(() => {
    const keydownListener = () => {}
    document.addEventListener('keydown', keydownListener);
    return () => document.removeEventListener('keydown', keydownListener);
  }, []);
  // ↑ここまで

  return <div>{num}</div>;
};
```

もし useEffect でラップせずに DOM　を操作したりすると、**書いたエフェクトがreact 内部での処理とかち合う可能性があります。** react の DOM 操作を考慮しながらエフェクトを書くのはかなりハードです(というか無理です)。

まとめ: **react のレンダリングの内部動作を完全完璧に理解できていないなら、大人しく useEffect のなかにエフェクトを書きましょう。** (理解できていてもやめたほうがいいと思う)

# 問題2: クリーンアップがないとバグの原因になりうる

useEffect を使っていないと、**クリーンアップ(エフェクトのリセット)ができません。** これが2つ目の問題です。

たとえば複数ページを持った SPA を考えます。

```tsx
const NewsPage = () => {
  document.title = 'news';
  return <h1>News</h1>
};
const AboutPage = () => {
  // document.title = 'about' の実装が抜けてる!
  return <h1>About</h1>
};
const Layout = ({children}) => (
  <div>
    <Nav />
    {children}
  </div>
);
```

AboutPage は document.title の実装が漏れてしまっています。この時、NewsPage から AboutPage へ遷移すると、タイトルが news のままになります。**バグです。**

この時、NewsPage 側でクリーンアップをしておけば、ある程度被害を抑えられます。

```tsx
const NewsPage = () => {
  useEffect(() => {
    const prevTitle = document.title;
    document.title = 'news';
    return () => {
      document.title = prevTitle;
    }
  });
  return <h1>News</h1>
};
```

これなら、AboutPage に遷移した時タイトルは空のままになります。news のままになるよりはマシです。

このように、クリーンアップが漏れていると、 **直前にレンダリングしていたもののエフェクトが残り続けてしまいます。** この例の規模ならば問題は少ないです。しかし、実際のアプリ開発のデバッグ時には大きな妨げとなります。 想像してください。「AboutPage のバグは、直前に NewsPage を訪問してた時にのみ起きる」のです。 **再現条件が複雑なので絶対に調査が難航します。**

このように、クリーンアップができていないと、あるレンダリングの影響が残り続けてしまうので、レンダリングを気軽にできなくなります。「これまでにどのようなコンポーネントをレンダリングしてたか」を考えずに済むようにするにはエフェクトのクリーンアップが必須です。

# まとめ

- `useEffect` 内にエフェクトを書くと、**レンダリングとエフェクトが完全に分離され、コンポーネントがシンプルになります**
- ReactのDOM操作と競合せずに済むので、reactの内部動作を知らずにコーディングできます
- エフェクトをクリーンアップすることで、前のレンダリングの影響が残り続けるバグを防ぎ、アプリケーションのデバッグを容易にします。

必ず useEffect で囲みましょう。
