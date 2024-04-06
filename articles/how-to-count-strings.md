---
title: "文字数のカウントはどれが正解なのか?"
emoji: "🧶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "ソフトウェアテスト"
  - "javascript"
  - "typescript"
published: true
---

A. ユースケース次第でどう実装すべきかは変わる。Intl.Segmenter が万能というわけでもない。

([クソ最悪な小バズ](https://x.com/le_panda_noir/status/1776241771766526372?s=46)をかましてしまったので、贖罪も兼ねて記事を書きました)

# 「文字数を数える」のは難しい
「文字数を数える」実装は意外と難しいです。というもの、アルファベットや数字だけなら `str.length` でも正しく数えられますが、絵文字や異体字などが入った文字列は見た目どおりに数えられません。

```
"👨‍👩‍👧‍👦".length === 11 // 1じゃない！！
```

そこで、以下で文字列を数えるときに使える実装を3つ見てみます。

## 実装1: str.length
一番カンタンなのは `str.length` です。アルファベット程度ならこれで十分です。しかし、**サロゲートペアをうまくカウントできません。**

```ts
"abc123あいう".length === 9 // 🙆‍♂ カウントできてる
"𠮷野屋".length === 4 // 🙅‍♂ 𠮷 が2とカウントされる
```

## 実装2: [...str].length

`[...str].length` は unicode の code point 単位でカウントします。これならサロゲートペアなどの問題を回避できます。**ただ、異字体セレクタなどがあるとうまく数えられません。**

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

...しかし、本当に毎回 Intl.Segmenter を使うのが万能な方法なのでしょうか?

# そもそも絵文字で文字数がおかしくなるのはダメなのか？

そもそも、絵文字や漢字が入ったときに文字数がおかしくなるのはダメなのでしょうか？ **実はユースケース次第では `str.length` でも問題ありません。**

たとえば、ユーザー名の入力フォームのバリデーションを考えます。

![](https://storage.googleapis.com/zenn-user-upload/a36c04de5a0c-20240406.png)

- ユーザー名はアルファベット、数字、アンダースコア、ハイフンのみ許容
- ユーザー名は10文字以内

この条件であれば、**そもそも絵文字が入った時点で弾けます。** 文字数を数えるまでもありません。

アルファベット、数字、アンダースコア、ハイフンだけカウントするなら `str.length` で十分できます。よって、今回のユースケースでは以下のような関数になります。

```ts
const isValidUsername = (username: string) => {
  return /^[a-zA-Z0-9-_]+$/.test(username) && username.length <= 10;
}
```

(もちろんこの場合 Intl.Segmenter でもいいですが、大事なのは str.length でも良いということです)

## 求める仕様を満たせているか確認すれば str.length でも OK

先程のコードが本当に求める仕様どおり動くのか、テストして確認してみましょう。

仕様をさらに細かく分解して、以下の3つを満たせればバリデーションが機能していると言えるでしょう。

- 無効な文字(ひらがな等)が入ってたら弾けるか
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

このように、自身のユースケースと仕様について考えて実装を行い、テストをして仕様の範疇で正常に動くことを確認するのが肝要です。逆に、仕様の範囲外で正常に動かなくてもそれは知ったこっちゃありません。

## Intl.Segmenter が欲しいケース/使ってはいけないケース

もちろん、`str.length` ではなく Intl.Segmenter が欲しいケースもあります。例えばブラウザでトランケート処理[^1]をするときには、見た目の文字数をちゃんとカウントする必要があります。

また逆に、 **Intl.Segmenter を使ってはいけないパターンもあります。** たとえば「データベースのカラムに収まるか判定する」ときには、Intl.Segmenter でカウントしてはいけません。

このように、Intl.Segmenter も文字数カウントの銀の弾丸ではありません。自身が求めている仕様をしっかりと詰め、ちゃんとテストして動作を確認するようにしましょう。

# まとめ: 自身のユースケースに合わせた仕様を決めよう

「文字数を数える」というのはあまりに一般的すぎます。**実際の自身のユースケース次第で何を使うべきかを考えましょう。**

また、ちゃんと仕様を考えて、テストを行い、 **求められる仕様の範疇で正常に動くことを保証をしましょう。** 仕様外のパターンについては「それは保証範囲外です」としっかりいえる、それが肝心です。


# 参考

- [JavaScript における文字コードと「文字数」の数え方 | blog.jxck.io](https://blog.jxck.io/entries/2017-03-02/unicode-in-javascript.html)

[^1]: トランケート処理とは、はみ出した部分を「...」で置き換える処理のこと
