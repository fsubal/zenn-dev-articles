# CSV から PHP の連想配列を作って実運用する話

[ピクシブ株式会社 AdventCalendar 2017 11日目の記事](https://qiita.com/advent-calendar/2017/pixiv)です。

エンジニアの @fsubal です。普段は[イラストコミュニケーションサービス pixiv](https://www.pixiv.net/)で主に投稿者向けの機能開発を行っています。
最近は絵の描き方で困っているユーザーのために「[**描き方ページ**](https://www.pixiv.net/howto)」というのを実装しました。

### 巨大なベタ打ちデータを作る

例えば「カテゴリ」のように、基本的に運営側の人間しか変更しないデータはベタ打ちで管理したいことがあると思います。

こういうデータはリリース後頻繁に変わるものでもないので、管理画面がどうしても必要とかでなければDBを使わずに済ませたいものです。万が一スキーマ変更したくなった場合のコストも低く済みますからね。

ちょっとしたデータなら yaml なり連想配列なりを手書きする手もあるでしょうが、カテゴリ の種類が200個ぐらいあるとこの方法では厳しい感じがしてきます。

pixiv の描き方ページは 10個 の大カテゴリの下に 32個の中カテゴリ、その下に **232個の小カテゴリ** がありました。こういう時にうまいことベタ打ちデータを作るにはどうするのが良いでしょうか。

### 実装方針

- Googleスプレッドシートでいい感じのシートを書き、CSV出力する
- パースして PHP の連想配列として成形する
- `var_export()` で PHP で実行可能なコードを出力する
- 出来上がったデータに対するテストを書く

### 前提

以下、「**ユーザー投稿されたお絵かき講座が簡単に探せるサービス**」を作っているという想定で説明します。

講座のカテゴリは「**大カテゴリ**」「**中カテゴリ**」「**小カテゴリ**」の3階層でできています、

たとえば `人物 > 服装 > ゴシックロリータの描き方` といった具合です。これの仕様書を以下のようなスプレッドシートで管理していると想定します。

|大カテゴリ名|大カテゴリサムネイル|中カテゴリ|小カテゴリ|検索ワード|検索モード|
|---|---|---|---|---|---|
|人物|`https://some.com/category/01.jpg`|服装|ゴシックロリータ|ゴスロリ OR ゴシック OR ロリータ OR ロリィタ OR ゴシックロリータ|完全一致|
|人物|`https://some.com/category/01.jpg`|服装|スーツ|スーツ OR ネクタイ|完全一致|
|背景|`https://some.com/category/02.jpg`|建物|学校|学校 OR 校舎 OR 体育館|部分一致|

親要素の情報が行をまたいで被ることは許容します。
（上の表で言うと、`ゴシックロリータ` と `スーツ` の列の内、大カテゴリ名や中カテゴリ名はかぶっているので冗長です。しかしこの形のほうが生成スクリプトが単純になるのでこれで行きます）

予めこの形に正規化してもらえるようチームメンバーには頼んでおきましょう。このスプレッドシートが言わばデータの管理画面です。

### 出来上がりのイメージ

```php
<?php

// 出来上がるクラスのイメージ
final class Category
{
	const CATEGORY_ID_HUMANBODY = 1;
	......
	
	const CATEGORY_CONF = [
		// 大カテゴリ
		Category::CATEGORY_ID_HUMANBODY => [
			'name' => '人物',
			'thumbnail' => 'https://some.com/category/01.jpg',
			'children' => [
				// 中カテゴリ
				'服装' => [
					'children' => [
						// 小カテゴリ
						'ゴシックロリータ' => [
						    'query' => 'ゴスロリ OR ゴシック OR ロリータ OR ロリィタ OR ゴシックロリータ',
						    'search_mode' => SearchMode::EXACT,
						],
						......
					]
				],
				......
			],
		],
		......
	];
}
```

### CSV をパースする

スプレッドシートからシートを CSV でエクスポートしておきます。
まずはこれを PHP でパースしましょう。

PHPなら `SplFileObject` を用いるのが簡単です。 `SplFileObject::READ_CSV` をセットすることで、一行一要素という形でイテレートできるようになります。

PHP: SplFileObject - Manual
https://secure.php.net/manual/ja/class.splfileobject.php

```php
<?php

$csv = new SplFileObject('./from_spreadsheet.csv');
$csv->setFlags(SplFileObject::READ_CSV);

$records = [];
foreach ($csv as $line) {
    ...
}
```

安全のため予め必須のキーを配列で定義しておき、 `foreach` の中で `array_combine` しましょう。この時点で一行あたりのカラム数がおかしければエラーで止まります。

```php
<?php

$columns = [
	'parent_category_name', // 大カテゴリ名
	'parent_thumbnail_url', // 大カテゴリサムネイルURL
	'group_name', // 中カテゴリ名
	'child_category_name', // 小カテゴリ名
	'query', // 検索ワード
	'search_mode', // 完全一致 OR 部分一致
];

$records = [];
foreach ($csv as $line) {
	// $line の要素数と $columns の要素数が違ってたら例外
	$records[] = array_combine($columns, $line);
}
```

PHP: array_combine - Manual
https://secure.php.net/manual/ja/function.array-combine.php

### 連想配列を組み立てる

できあがったものを再び `foreach` で回して連想配列に組み立てます。この時一行ごとに中身のバリデーションを行うと良いでしょう。ここでは [beberlei/assert](https://github.com/beberlei/assert) を例に説明します[^assert]

[^assert]: 実際の業務では、社内で用いている型チェック用ユーティリティクラスがありそちらを使いました。だいたい `beberlei/assert` と似たインターフェースのものです。

```php
<?php

use Assert\Assertion;

$categories = [];
foreach ($records as $record) {
	try {
		$parent_category_name = trim($records['parent_category_name']);
 		$parent_category_key = md5($parent_category_name);

		$parent_thumbnail_url = trim($records['parent_thumbnail_url']);
		$group_name = trim($records['group_name']);
		$child_category_name = trim($records['child_category_name']);
		$query = trim($records['query']);
		$search_mode = trim($records['search_mode']);

		Assertion::minLength($parent_category_key, 1); // 空じゃない string
		Assertion::url($parent_thumbnail_url);
		Assertion::minLength($group_name, 1);
		Assertion::minLength($child_category_name, 1);
		Assertion::minLength($query, 1);
		Assertion::inArray($search_mode, ['完全一致', '部分一致']);

		$categories[$parent_category_key]['thumbnail'] = $parent_thumbnail_url;
		$categories[$parent_category_key]['name'] = $parent_category_name;
		$categories[$parent_category_key]['children'][$group_name]['children'][$child_category_name] = [
		    'query' => $query,
		    'search_mode' => $search_mode,
		];
	catch (Assert\InvalidArgumentException $e) {
		var_dump('Failed: ', $record);
		exit(1);
	}
}
```

`$parent_category_key` がMD5で表現されている理由は後ほど説明します。

### var_export() について

PHP には `var_export` という関数があります。

PHP: var_export - Manual
https://secure.php.net/manual/ja/function.var-export.php

これは `var_dump` などと異なり、引数に渡した変数を PHP のコードとして valid な文字列として出力することができます。

```php
// $ php -a

php > var_dump(['a' => 1]);
array(1) {
  ["a"]=>
  int(1)
}

php > var_export(['a' => 1]);
array (
  'a' => 1,
)
```

`var_export` はデフォルトでは成形結果が標準出力に出されますが、第2引数に true を渡すことで関数の返り値として渡すことできます。

```php
// $ php -a

php > $exported = var_export(['a' => 1], true);
php > var_dump($exported);
string(21) "array (
  'a' => 1,
)"
```

ということは、先に生成した連想配列はこんな感じでファイルに吐き出せます。簡単ですね。

```php
<?php

$output = var_export($categories, true);

......

$content = <<<EOM
<?php

final class Category
{
    const CATEGORY_ID_HUMANBODY = 1;
    const CATEGORY_ID_LANDSCAPE = 2;

    const CATEGORY_CONF = {$output};
}
EOM;

file_put_contents('output.php', $content);
```

メソッドの生えたクラスを文字列で作るのは流石に危険と判断してやめました。出力する際はプロパティだけのクラス or 変数宣言のみのPHPファイルにするのを強くオススメします。

インデントが崩壊した状態で出てくると思うのでそこはお使いのエディタで適宜フォーマットして下さい。

私はMac の VSCode に phpfmt をインストールして `Shift + option + F` を押して成形しました。

phpfmt - Visual Studio Marketplace
https://marketplace.visualstudio.com/items?itemName=guodf.vscode-phpfmt

### 出力を人間らしく整える

とりあえずこれで動くコードはできますが、`var_export` の出力をそのままコミットすると良いコードになりません。lint があるプロジェクトならきっと怒られるコードができるでしょう。

これを最低限整えるTIPSも以下に記しておきます。

#### `array()` 記法を消す

`var_export` で配列を出力すると、かの `array()` 記法で出力されます。2017年にこれはよろしくありません。まずはこれを直します。

```php
// $ php -a

php > var_export(['a' => 1]);
array (
  'a' => 1,
)
```

pixiv はかつて PHP 5.4+ 対応の際に `array()` を `[]` に置換する大規模なリファクタリングを行いました。その時 [thomasbachem/php-short-array-syntax-converter](https://github.com/thomasbachem/php-short-array-syntax-converter) というライブラリを用いていたので、同じものを使ってみます。出来上がったファイルにこれをかけてみましょう。

```diff
-file_put_contents('output.php', $content);
+$output_filename = 'output.php';
+file_put_contents($output_filename, $content);

+exec(__DIR__ . "/convert.php -w ${output_filename}");
```

これで `array()` が消えて全部 `[]` に変わります。

#### 添字なし配列が勝手に連想配列になるのを直す

サンプルコードにはいませんが、例えば次のようなプロパティを「人物」の大カテゴリに持たせたくなったとします。

```php
// 人物
// 大カテゴリの説明文に用いるタグの一覧。仮に <li>${word}</li> のように出力するとします。
'description_words' => ['人体', '顔', '目', '手', '足', '筋肉', '資料']
```

ところで、添字なしの配列を `var_export` で出力すると次のようになります。

```php
php > var_export(['人体', '顔', '目', '手', '足', '筋肉', '資料']);
array (
  0 => '人体',
  1 => '顔',
  2 => '目',
  ......
)
```

勝手に添え字が生えました。PHP の配列は連想配列と区別されませんから、値を正確に出力しようと思うとこうなるというのは納得できる話です。

ちょっと悔しいですがいい方法が思いつかなかったので正規表現で消します。 `/^\t+[0-9]+ =\> /` を削除してやります。

```php
array (
'人体',
'顔',
'目',
......
)
```

先ほどやった `array()` => `[]` の変換と合わせることで整います。

#### 定数を参照する

たとえば検索の際に完全一致か部分一致か、といった設定は pixiv.git 内では定数で表現されます。コーディング規約上これは後者の書き方が推奨されます（これは一例です）。

```php
× 'search_mode' => '完全一致'
○ 'search_mode' => SearchMode::EXACT
```

これまた悔しいですがいい方法が思いつかなかったので正規表現で置換します。メタプログラミング最高という気持ちになってきました。

```php
<?php

$output = preg_replace('/\'完全一致\'/', 'SearchMode::EXACT', $output);
```

同じ方法で大カテゴリのキーも置換しておきます。先ほど連想配列を生成した際に、大カテゴリのキーをわざわざMD5化していました。

これはキーを最終的にクラス内の定数に置換するため、コード内の他の場所に現れない文字列にしたかったからです。

```php
<?php

$output = preg_replace('/\'eec124abdcfa233b76d95d873ec4ac12\'/', 'Category::CATEGORY_ID_HUMANBODY', $output);
```

これでだいたい完成です。

```php
<?php

final class Category
{
	const CATEGORY_ID_HUMANBODY = 1;
	const CATEGORY_ID_LANDSCAPE = 2;
	
	const CATEGORY_CONF = [
		Category::CATEGORY_ID_HUMANBODY => [
			'name' => '人物',
			'thumbnail' => 'https://some.com/category/01.jpg',
			'children' => [
				'服装' => [
					'children' => [
						'ゴシックロリータ' => [
						    'query' => 'ゴスロリ OR ゴシック OR ロリータ OR ロリィタ OR ゴシックロリータ',
						    'search_mode' => SearchMode::EXACT,
						],
						......
					]
				],
				......
			],
		],
		......
	];
}
```


### テストを書く

ひとまずこれで当初イメージしたコードがcsvから生成されました。しかしこれだけあれこれやっていると出力結果の正しさを保証する必要が出てきます。

できあがったデータに対するテストをPHPUnitで書いていきましょう。とりあえず

- すべての大カテゴリがもれなく存在し、順番も正しい
- 大カテゴリの型が正しい
- 中カテゴリの型が正しい
- 小カテゴリの型が正しい

ことはテストしたいところです。

```php
<?php

namespace Category;

final class CategoryConfTest extends \TestCase
{
    public function test_全ての大カテゴリがもれなく存在し、順番も正しい()
    {
        $actual_category_ids = array_keys(\Category::CATEGORY_CONF);
        $this->assertSame([
            \Category::CATEGORY_ID_HUMANBODY,
            \Category::CATEGORY_ID_LANDSCAPE,
            ......
        ], $actual_category_ids);
    }

    public function test_大カテゴリが正しい形式をしている()
    {
        foreach (\Category::CATEGORY_CONF as $parent_category_id => $parent_category) {
            $this->assertInternalType('integer', $parent_category_id);
            $this->assertInternalType('string',  $parent_category['name']);
            $this->assertInternalType('string',  $parent_category['thumbnail']);
            $this->assertInternalType('array',   $parent_category['children']);
        }
    }

    ......
```

大カテゴリまではできました。一方ここで注意するべき点が、あります。

小カテゴリの型チェックを行う際に素朴にループで舐めようとすると三重ループになってしまってテストのパフォーマンスが悪くなります。ここでは中カテゴリを `dataProvider` 経由で吐かせて対処しておきます

```php
<?php

public function groupDataProvider()
{
    foreach(\Category::CATEGORY_CONF as $parent_category) {
        foreach($parent_category['children'] as $group_name => $group) {
            yield [$group_name, $group];
        }
    }
}
```

```php
<?php

/**
 * @dataProvider groupDataProvider
 */
public function test_中カテゴリが正しい形式をしている($group_name, array $group)
{
    $this->assertInternalType('string', $group_name);
    $this->assertInternalType('array',  $group['children']);
}

/**
 * @dataProvider groupDataProvider
 */
public function test_小カテゴリが正しい形式をしているか($_group_name, array $group)
{
    foreach ($group['children'] as $child_category_name => $child_category) {
        $this->assertInternalType('string', $child_category_name);
        $this->assertInternalType('string', $child_category['query']);
        $this->assertContains($child_category['search_mode'], [
            \SearchMode::EXACT,
            \SearchMode::PARTIAL,
        ]);
    }
}
```

これにより、万が一変なデータが入ってもCIでコケます。


### やってみた結果

実際これを pixivの [**描き方ページ**](https://www.pixiv.net/howto) を作る際に行いました（データは説明のために簡易化しているので本物はもう少し複雑です）。

カテゴリの仕様は実際リリース前にちょくちょく変更がありましたが、スプレッドシートを更新するたびにコード生成するフローを用意していたことで、素早くそれに対応することができました。

またCSVからのコード生成というといかにも危険な感じがしますが、生成時の型チェックと、ベタ打ちデータに対するテストを書くことで安全に作ることができました。

大規模ベタ打ちデータの運用はプロジェクトごとに色々あると思いますが、PHP で行う際の一つの方法ではあると思います。`var_export` 思ったよりも強力でした。ぜひお試しください。

### 告知

**[ピクシブ株式会社](https://www.pixiv.co.jp/)** ではクリエイターに向けてサービスを作っていきたいエンジニアを随時募集しています！

明日は @kana1 が Vim script を用いた大規模なリファクタリングの知見を書いてくれます！お楽しみに。


