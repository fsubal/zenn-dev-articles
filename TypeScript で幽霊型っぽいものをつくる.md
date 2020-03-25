# TypeScript で幽霊型っぽいものをつくる

TypeScript で幽霊型（Phantom Type）っぽいものを作る場合に必要なテクニックを紹介する。

### TL;DR

- TypeScript の型検査の仕組み上、普通の方法で幽霊型はできない
- が、 intersection type などをうまく用いることで、幽霊型と同じような目的を達成することはできる

### モチベーション

たとえば URL のエンコード。次の `encode()` 関数は、まだエンコードしてない文字列だけを受け取りたいとする。このとき、

```ts
const encode(decoded: string) => encodeURLComponent(decoded)
```

この実装だと当然エンコード済みの文字列も渡せてしまい、多重エンコードが起きる。未エンコードの文字列だけが渡ってくることをコンパイラレベルで検出したい。

### 幽霊型（Phantom Type）

こういうとき、他の言語ではよく幽霊型が用いられる。次のリンク先は Scala での実装例を紹介している

- https://www.slideshare.net/AkinoriAbe1/aja-2016623

```scala
// 上記スライドの 6 枚目より

class Str[T] (val str: String)

trait Normal
trait Encoded

def encode(x: Str[Normal]) = new Str[Encoded](...)
```

内部では利用されない型パラメータ（ `Normal` `Encoded` ）を使って、`Str[T]` にはそういう種類があること、 `Str[Normal]` と  `Str[Encoded]` は違う型なので互いに assign してはいけないことが表現されている。

このように、実装内部では一切使われない型パラメータを用いて、ある種の値をコンパイラレベルでのみ区別する方法が**幽霊型（Phantom Type）**である。

### 構造的型と名前的型

これと同じことを TypeScript でやりたいとする。が、いざ始めるとすぐに問題が生じる。

TypeScript はいわゆる structural type system を用いていて、構造（メンバーのキーとその値の型）に互換性があれば、たとえ名前の上では違う型でも通る。

なので、以下のコードは普通に通る。

```ts
class A {
    property: number
    method(s: string) { return null }
}

class B {
    property: number
    method(s: string) { return null }
}

const a: A = new B // えっ？？？？？？

const doSomething = (a: A) => a.method('foo')
doSomething(new B) // えっ？？？？？？？
```

この挙動を踏まえて、さきほどの Scala コードを真似してみる。

```ts
interface Str<T> {
    value: string
}

let a: Str<Encoded> // Encoded は何でも良い。class でも、interface でも…
let b: Str<Normal>

const encode = (s: Str<Normal>) => ...

encode(a) // 通ってしまう… ><
```

TypeScript にとっては、 `Str<Normal>` も `Str<Encoded>` も `value: string` を持つオブジェクトであることに変わりはないので、区別がされない。これでは先の方法での幽霊型は実装できない。

実際 [TypeScript FAQ](https://github.com/Microsoft/TypeScript/wiki/FAQ#why-is-astring-assignable-to-anumber-for-interface-at--) でも、利用されない型パラメータは受け取らないでくれとはっきり言っている。

> **Why is `A<string>` assignable to `A<number>` for interface `A<T> { }`?**
> TypeScript uses a structural type system. When determining compatibility between Something<number> and Something<string>, we examine each member of each type. If each member of the types are compatible, then the type are compatible as well. Because Something<T> doesn't use T in any member, it doesn't matter what type T is.

> In general, **you should never have a type parameter which is unused.** The type will have unexpected compatibility (as shown here) and will also fail to have proper generic type inference in function calls.

ところで、Flow は TypeScript とは異なり、全面的に構造的型検査を採用しているわけではない。（関数とobject は構造的型で、クラスには名前的型が用いられる。つまり、全く同じメソッドとプロパティが生えていても、名前が違うクラスは違う型として扱われる）。


> For example, Flow uses structural typing for objects and functions, but nominal typing for classes.
> https://flow.org/en/docs/lang/nominal-structural/

Flow はあまり詳しくないが、だとすると Flow では素直な幽霊型実装ができるかもしれない。

### 代わりの方法で幽霊型っぽくする

TypeScript は構造的型システムで、名前が違う型であってもプロパティが同じならば同じ型として扱ってしまう。

ならTypeScriptでは幽霊型は不可能なのか？と思いきや、実は代わりの方法でそれっぽいことができる。

- [https://medium.com/@dhruvrajvanshi/advanced-typescript-patterns-6cf8826c7944](https://medium.com/@dhruvrajvanshi/advanced-typescript-patterns-6cf8826c7944)

```ts
interface Iso8601 extends String {
    __Iso8601: never
}

const pattern = /^ 長いので省略 $/

export const iso8601 = (str: String) => {
    if (!str.match(pattern)) {
        throw new Error('not ISO8601')
    }
    return str as Iso8601
}

iso8601('2018-08-16')
```

以下、解説。

### 「必須だけど絶対に値が存在しないキー」を作る

特殊な文字列を表現するのに `String` を継承するということ自体はともかく、問題は中にいる `__Iso8601: never` だ。

`never` 型とは「**1つも取りうる値がない（null や undefined ですらない）**」ことを表す型である。これが必須のキーである `__Iso8601 ` の型となることで、「**ランタイム時には絶対に存在しないが、キーとしては必須なメンバー**」が表現できる。これがあるおかげで、 `Iso8601` は単なる `String` 型から区別される（ランタイムでは普通の文字列と同じように振る舞う）[^type]。

[^type]: もちろん厳密には `String` にないキーさえ生えていれば区別はされるのだが、普通の意味での継承（挙動やプロパティを追加したい）ではないことを示すために、値の型を `never` にしておくのがここでは適当である。

あとは普通の文字列を `Iso8601` 型にすべく、専用の関数を用意する。バリデーションが通ったら `Iso8601` にキャスト、失敗したら例外。`Iso8601` 型を `iso8601()` 関数の返り値のみに用いれば、`Iso8601` 型の値はすなわち `iso8601()` を通過済みであることが保証される。

このように、「**ある関数を通過済みであることを型で保証する**」というのは幽霊型を用いる典型的なパターンである。

### より汎用的な解法

とはいえしかし、この実装にも欠点がある。たとえば、 `extends String` は `extends string` と書くことはできない。そもそも小文字の `string` は継承可能ではないからだ。

この欠点を避けるには、先の記事のように `extends` を使うのではなく、 `&` を使った以下のように書くほうが良い。

```ts
type Iso8601 = string & { __Iso8601: never }
```

Angular の内部では（実験的にではあるが）この方法で実装した幽霊型が用いられている（情報提供してくれた @printf_moriken に感謝）。

https://github.com/angular/angular/blob/41ef75869c86752e7089f74a9f01b0cf06c6a6e4/tools/public_api_guard/platform-browser/platform-browser.d.ts#L122-L125

```ts
/** @experimental */
export declare type StateKey<T> = string & {
    __not_a_string: never;
};
```

さて、上の例では `never` にされるキーが決めうちになっていたが、複数の幽霊型を作って運用することを考えると、それらが互いに区別可能になっていたほうが良いだろう。

とすると、最終的には以下のような実装になる。

```ts
type Phantomic<T, U extends string> = T & { [key in U]: never }

type UserId = Phantomic<number, 'UserId'>
const userId = (n: number) => n as UserId

const doSomething = (id: UserId) => alert(id.toString())

// ok
doSomething(userId(10))

// not ok
doSomething(11)
```

これを「幽霊型」と称するのが言葉の使い方として正しいかは微妙だが、似たような目的が TypeScript でも実現可能なことがわかった。

### まとめ

- 幽霊型を使うと、ランタイムへの影響なしに、コンパイル時のみ区別される値が表現できる
- TypeScript は構造的型システムを採用してるので、他言語で普通に使われる幽霊型の実装ができない
- しかし intersection types と never を利用することで、TypeScript でも同様の目的を達成することはできる

### 追記（2018-09-09）

このようなテクニックは TypeScript 本体のコードベースでも利用されているようだ。キー名として `_〇〇Brand: ` を追加する規約を用いるようなので、こちらに従っておくと良いかもしれない。

https://basarat.gitbook.io/typescript/main-1/nominaltyping

一般化した型を用意する場合も、 `Phantomic<T>` より `Branded<T>` のような名前にした方が、このテクニックを知っている人に伝わりやすい実装にできそうだ。
