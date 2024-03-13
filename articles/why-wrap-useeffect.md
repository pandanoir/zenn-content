---
title: "なぜ document.title = 'title' は useEffect でラップする必要があるのか"
emoji: "⛩️"
type: "tech"
topics:
  - "react"
published: true
published_at: 2024-03-13 09:30
---

答え(結論):

- レンダリングとエフェクトを分離するため
- クリーンアップを設定するため

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

# 理由1: レンダリング時に react の内部動作を考慮しなくて済む

useEffect を使っていない App2 は、react が DOM 更新している最中に `document.title = 'Hello world'` が実行されます。**App2 のタイトル変更がレンダリングのどのタイミングで実行されるか、正確に言えますか?** 僕は言えません。

一方、useEffect を使っている App1 では、[DOM の更新が完全に終わったあとにエフェクトが実行されます](https://github.com/donavon/hook-flow/blob/master/hook-flow.png)。そのため、**App1 のレンダリング中に useEffect で囲まれた部分について考える必要がありません。**


このように、**useEffect でラップしていれば、書いたエフェクトがreact 内部での処理とかち合う可能性がありません。** useEffect なしで react の DOM 操作を考慮しながらエフェクトを書くのはかなりハードです(というか無理です)。useEffect でラップすれば、react のレンダリングが完了したあとにエフェクトが実行されるので、レンダリング中の挙動を意識する必要がありません。

例をもう一つあげます。以下はキー入力を受け付けるコンポーネントです。もし useEffect を使わないと、react の DOM 操作の最中にイベントリスナーを貼っていいか一々調べる必要があります。

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

useEffect を使っていれば、レンダリングとエフェクトが完全に分離できるので、レンダリングフェーズをシンプルに考えられます。

まとめ: **react のレンダリングの内部動作を完全完璧に理解できていないなら、大人しく useEffect のなかにエフェクトを書きましょう。** (理解できていてもやめたほうがいいと思う)


# 理由2: クリーンアップを設定できる

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

このように、クリーンアップが漏れていると、 **直前にレンダリングしていたもののエフェクトが残り続けてしまいます。** この例の規模ならば問題は少ないです。しかし、実際のアプリ開発のデバッグ時には大きな妨げとなります。 想像してください。「AboutPage のタイトルが news になるバグは、直前に NewsPage を訪問してた時にのみ起きる」のです。 **再現条件が複雑なので絶対に調査が難航します。**

このように、クリーンアップができていないと、あるレンダリングの影響が残り続けてしまい、気軽にレンダリングできなくなります。「これまでにどのようなコンポーネントをレンダリングしてたか」を考えずに済むようにするには、エフェクトのクリーンアップが必須です。

# 理由3: SSR 時にエフェクトが実行されない

useEffect で定義したエフェクトはサーバーレンダリング中には実行されません。これは公式ドキュメントにも書かれています。

> エフェクトはクライアント上でのみ実行されます。サーバレンダリング中には実行されません。

[useEffect – React](https://ja.react.dev/reference/react/useEffect#useeffect)

もし `document.title = 'title'` を useEffect なしで書いてサーバーレンダリングすると、サーバー上で `document.title = 'title'` が実行されます。しかし、サーバーの実行環境には document が存在しないのでエラーが起こります。

公式ドキュメントにもあるように、useEffect は「コンポーネントを外部システムと同期させるための React フック」です。なので、外部システム(この場合ブラウザのタイトル部分)と同期する処理は useEffect の中で行いましょう。

# まとめ

- `useEffect` 内にエフェクトを書くと、**レンダリングとエフェクトが完全に分離され、コンポーネントがシンプルになります**
- ReactのDOM操作と競合せずに済むので、**reactの内部動作を知らずにコーディングできます**
- エフェクトをクリーンアップすることで、前のレンダリングの影響が残り続けるバグを防ぎ、アプリケーションのデバッグを容易にします。

**必ず useEffect で囲みましょう。**


# 参考

- [useEffect – React](https://ja.react.dev/reference/react/useEffect#useeffect)
- [Flow Diagram - hook-flow](https://github.com/donavon/hook-flow/blob/master/hook-flow.png)
- [Why doesn't React.useEffect run on React server-side renders (SSR)?](https://codewithhugo.com/react-useeffect-ssr/)
