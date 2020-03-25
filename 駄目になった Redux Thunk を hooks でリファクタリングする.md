# 駄目になった Redux Thunk を hooks でリファクタリングする

この記事は [React Advent Calendar 13日目](https://qiita.com/advent-calendar/2019/react)の記事です。

皆さん、react-redux の hooks を使っていますか？
react-redux は [v7.1](https://github.com/reduxjs/react-redux/releases/tag/v7.1.0) より `useDispatch` `useSelector` などいくつかの hooks が導入されました。

界隈ではどちらかというと、`mapStateToProps` の冗長さを置き換えるもの、あるいは [Container / Presentation コンポーネント](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0) の区別を不要にするものとして注目されている感があります。実際、useSelector によって可能になった設計論は重要です。

しかし本記事ではそれらと違う観点、つまり「**設計に失敗した Redux を立て直すための武器として react-redux の hooks がスゴい！**」という方向からの解説を行います。

以下、redux のミドルウェアとして [redux-thunk](https://github.com/reduxjs/redux-thunk) を使っている前提で解説します。

### レガシーな Redux、あるいはアンチパターンについて

皆さんは、完全に設計に失敗した Redux を見たことがありますか？

Redux が失敗するパターンにはいくつかありますが、ここでは特にコンポーネントと action / reducer の責務分離に失敗しているケースを扱います。

hooks はしばしば「巨大になりすぎたコンポーネント」への処方箋として紹介されますが、同じことが「巨大になりすぎた thunk」や「thunk なのにコンポーネントと密結合なケース」にも言えるという話をします。

### Case 1: コンポーネントと密結合な action

以下の thunk は私が実際に目にした例です。

```ts
type OnProgress = (progress: number) => void
type AppThunk = ThunkAction<void, RootState, { api: Api }, AppAction>

export const save = (onProgress: OnProgress): AppThunk =>
  (dispatch, getState, { api }) => {
    return new Promise(resolve => {
      api.save(getState(), onProgress).then(response => {
        dispatch({ type: 'SAVED', payload: response })
        resolve(response)
        return
      })
    })
  }
```

```tsx
// component
onSave = () => {
  this.props.save(
    progress => this.setState({ progress })
  ).then(async response => {
    if (response.hoge === ...) {
      return
    }
  })
}

const mapDispatchToProps = (dispatch: Dispatch) => {
  return bindActionCreators(AllOfTooManyActionCreators, dispatch)
}

export default connect(null, mapDispatchToProps)(YabaiComponent)
```

この実装には複数の問題点があります。

- thunk が値を返しており、コンポーネントがそれを利用している
- thunk の中で起こる出来事と、コンポーネントの状態を変更する処理の順序関係が追いづらくなっている
- 特に意味のない bindActionCreators のせいで、どれが action でどれが普通の props 経由の関数か分かりにくくなっている[^bind]

[^bind]: `bindActionCreators`、hooks と一緒に使う分にはまぁ良いのですが（ https://react-redux.js.org/api/hooks#recipe-useactions ）、クラスコンポーネントにおいては経験上殆どのケースでバッドパーツになる印象があります。

本来、コンポーネントは dispatch した action がどうなるかについて深く知るべきではありません。action が コンポーネントに値を返したらそれは「単方向データフロー」ではないですからね。

唯一、終了まで待つための `await dispatch()` は許容されえますが、その場合も返り値として許してよいのは `Promise<void>` だけです

```ts
// Good
dispatch(somethingHappened())

// OK
await dispatch(somethingHappened())

// Bad :(
const response = await dispatch(somethingHappened())
```

これまで、こういう場合の正当なリファクタリングはコンポーネントにあった状態を store に寄せる（それによって thunk を見るだけで追える処理に直す）という物でした。

しかし、事実としてコンポーネントと密結合な処理である場合に、store にすべてを入れなくとも直せる方法があったらより素敵ではないでしょうか。影響範囲が閉じるならそれに越したことはありません。

### thunk を hooks に移動する

「事実としてコンポーネントと密結合であるのだから、thunk にするのをやめよう」と判断した場合、その処理は hooks で書き直すことが可能です。

たとえば、`save` という thunk は `useSaveApi` になるでしょう。次のように。

```tsx
export function useSaveApi() {
  const [progress, setProgress] = useState<number | null>(null)

  const dispatch = useDispatch()
  const editorState = useSelector(state => state)

  async function save() {
    const response = await api.save(editorState, setProgress)
    dispatch({ type: 'SAVED', payload: response } as const)

    if (response.hoge === ...) {
      return
    }

    ....
  }

  return [save, progress] as const
}
```

どうでしょう？さっきよりはだいぶ追いやすくなったんじゃないでしょうか。
少なくとも thunk で書いていたときのような特殊さはなくなって、よくある普通の hooks になりました。

こんな感じで、駄目になった thunk action は「関数を作る hooks」に置き換えることができます。

### デメリット・懸念

react-redux の hooks を利用する場合、末端のコンポーネントが直接 store とつながるような設計が可能になります。これは、いわゆる Container コンポーネントの区別を重視する立場からすると受け入れがたいかもしれません。

その場合は、hooks を使うところを Container に分離するのが良いでしょう。
たとえば、`useSaveApi` を扱うコンポーネントでは DOM を描画せず、その子コンポーネントに save を渡すだけにします。

```tsx
export default function SaveFormContainer() {
  const [save, progress] = useSaveApi()

  // DOM のレンダリングは SaveForm に任せる
  // テストや Storybook も、SaveForm に対してのみ書く
  return <SaveForm onSave={save} progress={progress} />
}
```

参考: https://qiita.com/seya/items/700184c0d4a52bc0d32b

ただ個人的に、レガシーな Redux をリファクタリングする過程では、一時的に「すべてが Container になる」状態を許容しつつ、とにかく古いコードをなくしていくという戦略はアリだと思います。

コンポーネントをテスタブルに、あるいは Storybook に載せやすくするのは後で簡単にできるからです。

### Case 2: ダイアログを挟む thunk action

さて、先の例は極端なので「**そんなビューと密結合な thunk なんてそうそう発生しないよ**」と思った人もいるかも知れません。
そういう失敗例ではなく、thunk を hooks に移行することが積極的にメリットがなるケースが見たい人のために、もう一つ例をあげましょう。`confirm` です。

保存処理の手前で、確認ダイアログを出して API にリクエストする処理はよくあることと思います。このとき、うっかり次のようなコードがあるとします。

```ts
const save = (): AppThunk => async (dispatch, getState, { api }) => {
  ...

  if (window.confirm('本当に良い？')) {
    const params = getSaveParams(getState())

    const response = await api.save(params)
    dispatch({ type: 'SAVED', payload: response } as const)
  }
}
```

はい。

ところで、確認ダイアログをブラウザ標準のものではなく、カスタムデザインのものにしたくなったとしましょう。ダイアログの中にリンクや画像を突っ込みたくなり、 `<Dialog />` を出したくなったとします。すると、途端にこの設計では困ってきます[^dialog]。

たとえばこんな感じでカスタムダイアログを出す関数を使っているとしましょう（ [react-promise-modal](https://github.com/prezly/react-promise-modal) などはこれに近い設計を提供します ）[^dialogcomponent]。

[^dialogcomponent]: 私はよくこういうモジュールを自作することがあって、実際便利なんですが、もっといい方法をご存知でしたら知りたいです（ `<Dialog>` に Context が引き継がれないなどのデメリットが発生しています ）。この記事に出てくる `dialog.confirm` はだいたい次の実装を前提にしたものと思ってください https://gist.github.com/fsubal/f5f4bed1849b36d614f7eac82ab85568

```tsx
// ダイアログの OK ボタンを押したら `Promise<true>`、
// キャンセルボタンを押したら `Promise<false>` が返る
if (await dialog.confirm(<MyMessage />)) {
  // OK を押したら...
}
```

↑これを thunk に書くのは嫌です（ 嫌というか、無理に近いです ）。

これまでは confirm 部分だけを何とかコンポーネントのイベントハンドラに収め、保存処理は thunk という形で分離してたと思います。

```tsx
// in Component Class
handleClickSave = async () => {
  if (await dialog.confirm(<MyMessage />)) {
    this.props.dispatch(...)
  }
}
```

しかしここで save 処理をまるごと hooks に移し、confirm を一緒に書くことによって「近い関心の処理が同じファイルに書かれる」状態を実現できます。次のコードはどうでしょうか。

```tsx
export function useConditionalSave() {
  const dialog = useContext(Dialog)

  const dispatch = useDispatch()
  const saveParams = useSelector(getSaveParams)

  return async function() {
    if (saveParams.suspicious) {
      if (!await dialog.confirm(<ReallyOkay />)) {
        // 本当によろしいわけではなかった
        return
      }
    }

    const response = await api.save(saveParams)
    dispatch({ type: 'SAVED', payload: response } as const)

    await dialog.alert(<Success />)
  }
}
```

どうでしょうか！ちょっと分岐が複雑ですが、関心のまとまりに沿った hooks ができました。

`window.confirm`、グローバルにあるせいでたまに action に書いちゃう人がいますが、本来的にはビューの責務に近いものです。後々こういうリファクタリング時に困らないよう、最初からコンポーネントに近い層である hooks で使うようにするのは良い考えだと思います。

こんな感じで、コンポーネントに近い処理が非同期処理と絡み合うケースでは、thunk をやめて hooks で済ますことが一つの解決策になります。

### まとめ

- 人間は thunk とコンポーネントの責務分離に失敗することがある
- 本質的にコンポーネントと密結合な非同期処理は、まるごと hooks に移行することで見通しが良くなる
- react-redux の hooks は ↑ をサポートすることで、駄目になった Redux アプリケーションを立て直すための強力な武器になる

皆さんも、過去に書いた非同期処理で苦しくなった際は是非試してみてください。

### Next...

明日は @iku000888 さんが ClojureScript の話をしてくれるようです！お楽しみに。

[^dialog]: もちろん、コンポーネントと特に関係ないダイアログ表示/呼び出し関数をどうにか用意して、`thunk.extraArgument` で dialog 関数を突っ込む方法はないこともないかもしれません。やりたいかは別ですが…。

