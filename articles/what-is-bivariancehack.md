---
title: "bivarianceHack とは何か、なぜ必要なのか"
emoji: "🔧"
type: "tech"
topics: ["TypeScript"]
published: false
---


TypeScript に bivarianceHack と呼ばれるテクニックがある。
これは、関数を意図的に双変（bivariant） にするテクニックだ。著名なところだと React の型定義で使われている

```typescript
  type EventHandler<E extends SyntheticEvent<any>> = { bivarianceHack(event: E): void }["bivarianceHack"];
```

実は自分はこのテクニックをとあるコードベースで使ったことがあるのだが（そしてそこにはある程度やむを得ない事情があったのだが）、当然初めて見たメンバーにとっては意味がわからない箇所となってしまった。

実際 bivarianceHack が必要になる事情を説明すると結構話が長い上に込み入ってしまうので、この記事でできるだけ噛み砕いてその背景を説明しようと思う。

### TL;DR

- TypeScript は配列を共変にするために型システムに矛盾が生じており、それを飲むためにメソッドが双変になっている
- `--strictFunctionTypes` 以前は通常の関数も双変として扱われた経緯がある
  - このため、`strict: true` への移行時など、双変であることを意識しないといけないケースがある
-	TypeScript はメソッドと関数を区別しており、その違いを悪用すると任意の関数を意図的に双変にすることができる
  -	`--strictFunctionTypes` 移行時の脱出ハッチ他、いくつかこれが必要なユースケースが存在する。これが bivarianceHack

### 双変（bivariance）とは何か
そもそも「双変」とは何か。これは共変（covariance）・反変（contravariance）との対比で使われる型システム上の用語である。

（※ 以下、共変・反変・双変の概念を知っている人は次の節に行って良い）

共変とは、たとえば型 A が型 B のサブタイプであるとき、ジェネリクス型 P<A> もまた P<B> のサブタイプになるような関係のことを指す（型 P<T> は共変であるという）。
次の例では、`List<T>` は共変である。`1` は `number` のサブタイプなので、`1` しか入らないリストもまた、`number` しか入らないリストのサブタイプというわけである。

```typescript
 // List<number> に、1 しか入り得ない List を代入するのは問題ない
 const l: List<number> = new List<1>(1, 1, 1)
```

反変は共変の逆で、型 A が型 B のサブタイプであるとき、逆に型 P<B> が P<A> のサブタイプになるような関係のことを指す。
典型的には関数の引数は反変の関係になる。`1` は `number` のサブタイプだが、`1` しか受け取れない関数は `number` 全部受け取れる関数のサブタイプではない。より具体的には、コールバック関数であらゆる `number` が来るかもしれないときに `1` しか受け取れない関数を渡すことはできない（が、逆は許される）。

```typescript
 declare const getNumber: (url: string, handleNumber: (response: number) => void) => void
 
 // 1 以外も来るかもしれないので、これはエラー
 getNumber('/mynumber', (hoge: 1) => {
 
 })
 
 // これを踏まえると、下の代入が失敗する意味も理解できるはず
 const fn: (response: number) => void = (response: 1) => {}
```

###  TypeScript はメソッドを双変として扱う
ところで TypeScript では（実は他の言語とは異なり）、例えば配列が共変になっている。

```typescript
 const l: Array<1> = [1, 1, 1]
 const ll: Array<number> = l // OK!
```

関数の引数は反変の関係にあるので、Array に生えているメソッドは当然反変の関係になるはずである。
具体的には、`Array<1>` が `Array<number>` のサブタイプであるなら、逆に `Array<number>.push()` は `Array<1>.push()` のサブタイプになるだろう。`1` しか入らない配列に任意の `number` が push できて良いはずがないと考えられるからだ。

```typescript
 const a1: Array<1> = [1, 1, 1]
 const push1 = a1.push
 
 // これはエラーになる
 const anyNumberCanBePushed: (...n: number[]) => number = push1
```

ところでしかし、TypeScript は構造的型システムである。
つまり TypeScript において、型 A が型 B に代入できるということは、型 A のプロパティを1個1個見ていって、その全てが型 B のプロパティに代入できるということを意味する。

```typescript
 interface YabaiHuman {
 	body: string
 	face: number
 	legs(): void
 }
 
 const you: YabaiHuman = {
 	body: '', // body あってる
 	face: 1, // face あってる
 	legs: () => console.log(800000000) // legs あってる
 } // ということは、you の型はあってる！
```

ここまでの前提を組み合わせると、TS では配列が共変なので `Array<1>` は `Array<number>`の変数に代入できるべきだが、`Array<1>` のプロパティである `push` メソッドは `Array<number>.push` に代入できないはずので話が矛盾していることになる。

```typescript
 const l: Array<1> = [1, 1, 1]
 
 // 代入できてるけど、本来なら .push の型に互換性がないということで怒られるべきなのでは…？ 🤔
 const ll: Array<number> = l
```

構造的型チェックの環境で配列を共変にすると（メソッドの型によって）型システムに矛盾が生じる。
それでも配列を共変として扱いたい場合、型システム上に何らかのごまかしというか、一種の妥協が必要になる。
（ちなみに、型システム上にこの意味でのごまかしが存在しないことを「健全性（soundness）」と呼ぶと理解している）

そこで双変（bivariance）が登場する。
双変とは、型 A が型 B のサブタイプであるとき、P<A> を P<B> に代入しても許されるし、逆に P<B> を P<A> に代入しても許されるといった関係のことを指す。
何を代入しても許すほどゆるくはないが（たとえば全く親子関係にない null を P<A> に代入したらエラーになるとかは当然起きる）、親子関係にあれば何でもいいという点で厳密性がない。

TypeScript は配列をはじめ、いろいろな型のメソッドを双変として扱う。これにより、`Array<1>` の `push` メソッドが `Array<number>.push` に代入できない問題が解消される。

### TypeScript は「関数」と「メソッド」を区別している

しかし、最初の説明では関数の引数は反変だったのに、Array の push メソッドの例で双変だったのはどういうことか。
先のような問題はあらゆる関数で起こるのか？と思うかもしれないが、そうではない。
実は TypeScript は「関数」と「メソッド」を内部的に区別しており、双変になるのはメソッドのときだけである。

具体的には、TypeScript は次の2つの記法を違う意味で解釈している。

```typescript
 type Props = {
   onSelect: (value: string) => void
 }
 
 type Props = {
   onSelect(value: string): void
 }
```

https://www.typescriptlang.org/play?#code/C4TwDgpgBAygrgYwRAzigCgJwPZhVAXigG8BYAKCimwDsYIAbCBYACgDcBDBuCALigpgmAJY0A5gEoB7bCIAmFAL4UKoSFABinEQyy58RMpWp1GzYAI7deAoaImTCAPiiyFy1eQS0hp+kwshFDWPPxQAOScEVAAPpEARhFOBK7EUD40KNhMAHQM2OKhvE4q5BSZfmAAjALwSKgYOHjBxlS0ARae5d6+wFBgAEwC2rr6LUYU7WaBwN0UQA

アロー関数で書いた場合は「関数」扱い、そうでない場合は「メソッド」扱いになる。
それはつまり、前者で書いた場合は反変、後者の場合は双変になるということを意味する。
（ ただし `--strictFunctionTypes` が有効なときのみ。これを解除した場合は両方とも双変になる ）

https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-6.html

時折、メソッドと関数の区別はクラス記法や `interface` を使ったときのみ発生すると勘違いしている人がいるが、そんなことはなく `type` 記法を使っていてもこの問題は存在する。

実は `strictFunctionTypes` 導入にあたり、メソッドを含めあらゆる関数を反変にするか議論が行われたことがあるらしい（残念ながらどの Issue か見つけられてません！どなたかご存知でしたらお寄せください）。
が、それをやるとクラスの継承を行っている箇所が軒並みエラーになるなどの理由から頓挫し、「メソッド」と「関数」で変性が異なるというこの奇妙な挙動が生まれることとなった。

### 関数の型を意図的に双変にする方法としての bivarianceHack

さて、`onChange(next: number): void` と `onChange: (next: number) => void` の意味が違うなら、
前者の書き方をした上で onChange の型を取り出すことができれば、好き勝手に双変の関数が作れることになる。

```typescript
 type BivariantOnChange = {
   onChange(next: number): void
 }['onChange']
```

そして、これが冒頭の `bivarianceHack` の正体である。

これに汎用性をもたせ、任意の関数型を双変にする型が作りたければ次のように定義ができる。

```typescript
 type Bivariant<M extends (...args: any[]) => any> = {
   bivarianceHack(...args: Parameters<M>): ReturnType<M>
 }['bivarianceHack']
```

### bivarianceHack のユースケース

しかしこんなもの一体いつ使うのか？
そもそも（あまり健全ではないが） bivariance に依存したいユースケースとして次のようなイベントハンドラのケースがある。
（冒頭の React の件も EventHandler に関わっていたことを思い出すと良い）

`--strictFunctionTypes` が無効な状態で次の React コンポーネントが定義されているとする。

```typescript
 interface Props {
 	value: string
 	onChange?: (nextValue: string) => void
 }
 
 export const SomeInput: React.FC<Props> = () => { ... }
```

ここで、`strictFunctionTypes` が無効なので `onChange` は双変である。
双変であるということは、親コンポーネントは次のようなことをしても許されてしまう。

```typescript
 function handleChange(nextValue: 'a' | 'b' | 'c' ) {}
 
 return (
 	<SomeInput value={...} onChange={handleChange}>
 )
```

この定義は健全ではない。コンパイラからすれば、`nextValue` として `a` でも `b` でも `c` でもない文字列が渡ってくるかもしれないからだ。それでもこれは（ `strictFunctionTypes` がなかった頃の TypeScript では ）通ってしまっていた。

この状態で、あなたはコードベース全体を `strict: true` に移行するリファクタリングを行いたいとする。
そして `strictFunctionTypes` を有効にした途端、このあたりのコードはコケるようになった。

本当は Props の型定義を健全になるように書き換えていくのが筋ではあるが、これの利用箇所（あるいは似たようなことをしているコンポーネント）があまりにも多く、それが厳しかったとしよう。

```typescript
 interface Props {
 	value: string
 	
 	// FIXME: `strict: true` にする過程で、むりやり双変を維持している
 	onChange?: Bivariant<(nextValue: string) => void>
 }
 
 export const SomeInput: React.FC<Props> = () => { ... }
```

これで一旦通るようになった。もちろん最終的にここは直すべき印ではあるが、機械的な移行過程で一時的にこういう変更をすることができる（ 治すべきところを探すときは `Bivariant<T>`  の利用箇所を探れば良い ）というわけだ。

### まとめ

- TypeScript は配列を共変にするために型システムに矛盾が生じており、それを飲むためにメソッドが双変になっている
- `--strictFunctionTypes` 以前は通常の関数も双変として扱われた経緯がある
  - このため、`strict: true` への移行時など、双変であることを意識しないといけないケースがある
-	TypeScript はメソッドと関数を区別しており、その違いを悪用すると任意の関数を意図的に双変にすることができる
  -	`--strictFunctionTypes` 移行時の脱出ハッチ他、いくつかこれが必要なユースケースが存在する。これが bivarianceHack
