---
title: "DateTimeImmutable::diff を返す関数のテストで DateInterval を new するとコケる"
emoji: "📆"
type: "tech"
topics: ["PHP", "PHPUnit"]
published: true
---

# DateTimeImmutable::diff を返す関数のテストで DateInterval を new すると assertEqual がコケて困る件

日付同士の差分を計算する際に `DateTimeImmutable::diff()`（あるいは `DateTime::diff()` ）を用いることがあると思います。

PHP: DateTime::diff - Manual
https://secure.php.net/manual/ja/datetime.diff.php

この関数の返り値は `DateInterval` 型で、時間の「間隔」を表現するクラスです。このクラス自体は便利なのですが、`DateInterval` 型にはある特有の挙動があり、これを返す関数をテストしたい場合に問題が起こります。

### 例: テストしたい関数

ある「〆切」を持ったオブジェクトが存在し、それに対する残り時間を返す関数があるとしましょう。設計はかなり適当です。

```php
<?php

final class ItemWithDeadline
{
    /** @var DateTimeImmutable */
    private $deadline;

    ......

    /**
     * @param DateTimeImmutable $now
     * @return DateInterval
     */
    public function getRemainingInterval(DateTimeImmutable $now)
    {
        return $now->diff($this->deadline);
    }
}
```

そしてこれに対応する PHPUnit のテストがあるとします

```php
final class GetRemainingIntervalTest extends TestCase
{
    public function test_未来の締切に対して正しい残り時間が返る ()
    {
        $now = DateTimeImmutable::createFromFormat('Y-m-d h:i:s', '2017-12-04 00:00:00');
        $item = new ItemWithDeadline([
            'deadline' => new DateTimeImmutable('2017-12-05 00:00:00'),
            ......
        ]);

        $this->assertEquals(
            DateInterval::createFromDateString('1 day'),
            $item->getRemainingInterval($now)
        ); // fails!
    }

    public function test_過ぎた締切に対して正しい残り時間が返る ()
    {
        ......
    }
}
```

結論から言うとこの上のテスト（ `test_未来の締切に対して正しい残り時間が返る` ）は失敗します。いったい何故でしょうか。

### DateInterval#days について

`DateInterval` クラスには `days` なるプロパティが存在します。

公式マニュアルの `DateInterval` のページには次のように書かれています。

> days
> DateTime::diff() で作られた DateInterval オブジェクトの場合は、開始日と終了日の間の日数。 それ以外の場合は days は FALSE となります。

何を言っているんだという気持ちですが、とにかくそのようなものが存在します。上のテストを実行すると、 `expected` と `actual` の diff に `days` が出現するのがわかると思います。PHPUnit の `assertEquals` は参照の異なるオブジェクトでも同値判定ができますが、各プロパティの中身はあくまでも同値でなければならないからです。

これを回避する方法は簡単で、たとえば `expected` として何か自明な `diff` を返す関数を作るというのがあります。

```php
<?php

private static function _get1DayInterval ()
{
    $sooner = DateTimeImmutable::createFromFormat('Y-m-d h:i:s', '2000-01-01 00:00:00');
    $later = DateTimeImmutable::createFromFormat('Y-m-d h:i:s', '2000-01-02 00:00:00');

    return $sooner->diff($later);
}

......

$this->assertEquals(
    self::_get1DayInterval(),
    $item->getRemainingInterval($now)
); // Pass!
```

おめでとうございます！良かったですね。泣いてしまいそうです。

### もう一つの罠

ところで、オブジェクト同士を比較する以外の方法も検討できるのではないかという意見があるでしょう。

たとえば仕様上 1 日単位の diff しかありえない場合（決して1週間とか1ヶ月にならず、また端数の hour なども発生しないなど）に、初めから `DateInterval#d` だけを比較する手があるのではないか、とか。

```php
<?php

public function test_未来の締切に対して正しい残り時間が返る ()
{
    $now = DateTimeImmutable::createFromFormat('Y-m-d h:i:s', '2017-12-04 00:00:00');
    $item = new ItemWithDeadline([
        'deadline' => new DateTimeImmutable('2017-12-05 00:00:00'),
        ......
    ]);

    $this->assertEquals(
        DateInterval::createFromDateString('1 day')->d,
        $item->getRemainingInterval($now)->d
    ); // Pass!
}
```

これは通ります。両方とも `1` ですから何の問題もありません。

しかしこれには罠があります。例えば締め切りを過ぎた場合のケースのテストを追加しましょう。

```diff
 <?php

 public function test_過去の締切に対して正しい残り時間が返る ()
 {
-    $now = DateTimeImmutable::createFromFormat('Y-m-d h:i:s', '2017-12-04 00:00:00');
+    $now = DateTimeImmutable::createFromFormat('Y-m-d h:i:s', '2017-12-06 00:00:00');
     $item = new ItemWithDeadline([
         'deadline' => new DateTimeImmutable('2017-12-05 00:00:00'),
         ......
     ]);

     $this->assertEquals(
-        DateInterval::createFromDateString('1 day')->d,
+        DateInterval::createFromDateString('-1 day')->d,
         $item->getRemainingInterval($now)->d
     ); // fail!
 }
```

これが失敗します。いったい何ででしょうか。

### DateImmutable#invert について

`DateImmutable` にはもう一つ興味深い `invert` というプロパティがあります。こちらは公式では以下のように説明されます。

> invert
> 間隔が負の数になっている場合は 1、そうでない場合は 0。 DateInterval::format() を参照ください。

https://secure.php.net/manual/ja/class.dateinterval.php

すなわち、たとえば以下のようにして、どちらが後の日付かを判定するのに使えます。便利ですね。

```php
<?php

$interval = $today->diff($deadline);

if ($interval->invert == 1) {
    throw new Exception('締め切りを過ぎています');
}
```

### `new DateInverval` したときの invert の挙動

さてこちらの `invert` ですがなかなか面白い挙動をしてくれます。先ほど、 `$today->diff($deadline)` によって得た期間の `invert` は `1` になることを見ました。

ところで同じ「負の向きに1日分」を表す `DateInterval` を直接作るとどうなるでしょうか。

```console
$ php -a

php > var_dump(DateInterval::createFromDateString('-1 day'));
object(DateInterval)#4 (15) {
  ["y"]=>
  int(0)
  ["m"]=>
  int(0)
  ["d"]=>
  int(-1)
  ["h"]=>
  int(0)
  ["i"]=>
  int(0)
  ["s"]=>
  int(0)
  ["weekday"]=>
  int(0)
  ["weekday_behavior"]=>
  int(0)
  ["first_last_day_of"]=>
  int(0)
  ["invert"]=>
  int(0)
  ["days"]=>
  bool(false)
  ["special_type"]=>
  int(0)
  ["special_amount"]=>
  int(0)
  ["have_weekday_relative"]=>
  int(0)
  ["have_special_relative"]=>
  int(0)
}
```

何と `invert` は `0` です。代わりに日数を表す `d` が `-1` になりました。

### 回避策

さきほど `days` の回避策として、自明な日付の差分を取ってその結果を返す関数を作る方法を紹介しました。こちらに一工夫を加えます。

```diff
 <?php

-private static function _get1DayInterval ()
+private static function _get1DayInterval ($invert = false)
 {
+    // https://github.com/beberlei/assert
+    Assersion::boolean($invert);

     $sooner = DateTimeImmutable::createFromFormat('Y-m-d h:i:s', '2000-01-01 00:00:00');
     $later = DateTimeImmutable::createFromFormat('Y-m-d h:i:s', '2000-01-02 00:00:00');

-    return $sooner->diff($later);
+    return $invert ? $later->diff($sooner) : $sooner->diff($later);
 }
```

こうすることで負方向の `DateInterval` も正しくテストに使えます。良かったですね。

### あるいは

あきらめてこう書いたほうが安全かもしれません

```php
<?php

$this->assertEquals(0, $actual->y);
$this->assertEquals(0, $actual->m);
$this->assertEquals(1, $actual->d);
$this->assertEquals(0, $actual->h);
$this->assertEquals(0, $actual->i);
$this->assertEquals(0, $actual->s);
```

### まとめ

`DateInterval` 本当に厳しいので覚えておいたほうが良いです。
