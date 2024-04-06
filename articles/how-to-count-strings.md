---
title: "文字数のカウントに Intl.Segmenter は本当に必要なのか？"
emoji: "🧶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "ソフトウェアテスト"
published: true
---

A. ユースケース次第だし、Intl.Segmenter を使ってはいけないケースもある

([クソ最悪な小バズ](https://x.com/le_panda_noir/status/1776241771766526372?s=46)をかましてしまったので、贖罪も兼ねて記事を書きます)

# 文字数を数えるのは難しい
文字数を数えるのは意外と難しいです。 `str.length` ではアルファベットや数字は正しく数えられますが、**ちょっと込み入った文字列は見た目どおりに数えられません。**

以下で文字列を数えるときに使える実装を3つ見てみます。

## 実装1: str.length
一番カンタンなのは `str.length` です。しかし、**サロゲートペアをうまくカウントできません。**

```ts
"abc123あいう".length === 9 // 🙆‍♂ カウントできてる
"𠮷野屋".length === 4 // 🙅‍♂ 𠮷 が2とカウントされる
```

## 実装2: [...str].length

`[...str].length` とすると unicode の code point 単位で数えられるので、サロゲートペアなどの問題を回避できます。**ただ、異字体セレクタなどがあるとうまく数えられません。**

```ts
[..."𠮷野屋"].length === 3 // 🙆‍♂ サロゲートペアは回避できてる
[..."葛󠄀城市"].length === 4 // 🙅‍♂ 葛󠄀 が2とカウントされてしまっている
```

## 実装3: Intl.Segmenter
`Intl.Segmenter` を使って書記素単位で数えれば、見た目のとおりにカウントできます。

```ts
const segmenter = new Intl.Segmenter("ja-JP", { granularity: "grapheme" })
[...segmenter.segment("葛󠄀城市")].length === 3 // 🙆‍♂ 想定どおりカウントされている
[...segmenter.segment("👪")].length === 1 // 🙆‍♂ 想定どおりカウントされている
```

このように、**文字列を数えるというのは意外と単純ではありません。**

...しかし、本当に毎回 Intl.Segmenter を使うのが万能な方法なのでしょうか?

# なぜ文字を数えたいのか?を考える

そもそも、**絵文字や漢字で文字数がおかしくなるのはダメなのでしょうか？** 実は問題ないのではないでしょうか?

たとえば、ユーザー名の入力フォームのバリデーションを考えます。

![](https://storage.googleapis.com/zenn-user-upload/a36c04de5a0c-20240406.png)

- ユーザー名はアルファベット、数字、アンダースコア、ハイフンのみ許容
- ユーザー名は10文字以内

この条件であれば、**そもそも絵文字が入った時点で弾けます。** 文字数を数えるまでもありません。

また、アルファベット、数字、アンダースコア、ハイフンのカウントなら `str.length` で十分にカウントできます。よってコードは以下のようになります。

```ts
const isValidUsername = (username: string) => {
  return /^[a-zA-Z0-9-_]+$/.test(username) && username.length <= 10;
}
```

# 仕様を満たせているか確認する

先程のコードはまだテストしてないので、本当に求める仕様どおり動くのかは不明です。Intl.Segmenter を使わなくていいのでしょうか?テストして確認してみましょう。

仕様をさらに細かく分解すると、以下の3つを満たせていればバリデーションがちゃんと機能していると言えるでしょう。

- ひらがななどの無効な文字が入ってたら弾けるか
- 10文字を超えてたら弾けるか
- 複数種類混じっていても判定できているか

テストを書いてみます。

```ts
test('無効な文字が入ってたら弾けるか', () => {
  expect(isValidUsername('あ')).toBe(false);
  expect(isValidUsername('ア')).toBe(false);
  expect(isValidUsername('阿')).toBe(false);
  expect(isValidUsername('😃')).toBe(false);
  expect(isValidUsername('[')).toBe(false);
});
test('10文字を超えてたら弾けるか', () => {
  expect(isValidUsername('aaaaaaaaaa')).toBe(true); // 10文字
  expect(isValidUsername('aaaaaaaaaaa')).toBe(false);
  expect(isValidUsername('0000000000')).toBe(true); // 10文字
  expect(isValidUsername('00000000000')).toBe(false);
  expect(isValidUsername('----------')).toBe(true);
  expect(isValidUsername('-----------')).toBe(false);
  expect(isValidUsername('__________')).toBe(true);
  expect(isValidUsername('___________')).toBe(false);
});
test('複数種類混じっていても判定できているか', () => {
  expect(isValidUsername('123456-_aA')).toBe(true);
  expect(isValidUsername('1234567-_aA')).toBe(false); // 11文字
  expect(isValidUsername('123456-_aあ')).toBe(false);
});
```

実際に動かしてみます。

![](https://storage.googleapis.com/zenn-user-upload/28291ab5b377-20240406.png)

すべてパスしましたので、これで **今回のユースケースであれば `str.length` でも十分であると保証できました。**

# Intl.Segmenter が欲しいケース/使ってはいけないケース

もちろん、Intl.Segmenter が欲しいケースもあります。例えばブラウザでトランケート処理[^1]をするときには、見た目の文字数をちゃんとカウントする必要があります。

また、**Intl.Segmenter を使ってはいけないパターンもあります。** たとえば「データベースのカラムに収まるか判定する」ときには、Intl.Segmenter でカウントしてはいけません。

# まとめ: 自身のユースケースに合わせた仕様を決めよう

「文字数を数える」というのはあまりに一般的すぎます。実際の自身のユースケース次第で何を使うべきかを考えましょう。

また、ちゃんと仕様を考えて、テストを行い、 **求められる仕様の範疇で正常に動くことを保証をしましょう。** 仕様外のパターンについては「それは保証範囲外です」としっかりいえる、それが肝心です。


# 参考

- [JavaScript における文字コードと「文字数」の数え方 | blog.jxck.io](https://blog.jxck.io/entries/2017-03-02/unicode-in-javascript.html)

[^1]: トランケート処理とは、はみ出した部分を「...」で置き換える処理のこと
