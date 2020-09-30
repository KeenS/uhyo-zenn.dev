---
title: "[...Array(n).keys()] はやめた方がいいのでは？"
emoji: "📀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: true
---

`[...Array(n).keys()]`は、JavaScriptで`0`から`n-1`までの整数が順番に並んだ配列を得る記法です。

```js
const arr = [...Array(5).keys()]
console.log(arr); // [0, 1, 2, 3, 4]
```

この記事では、**これはやめた方がいいのでは**ということを主張します。

## 理由

**読み手にかかる負荷が高い**（≒理解しにくい）からです。特に、`Array(n).keys()`の挙動を正確に把握する難易度が高くなっています。かといって、`[...Array(n).keys()]`という長いコードを中身を気にせずイディオムとして覚えるのは脳の記憶領域の無駄遣いです。

以下では、`Array(n)`と`keys()`に関して見ていき、難易度の高さを実感します。

### `Array(n)`とempty

`Array(n)`は`new Array(n)`でと同じ意味で、一言で言えば**長さ`n`の配列を作る**ということです。

ちなみに、この機能自体もあまり使うべきではありません。なぜなら、`n`として数値が渡された場合のみが特別扱いされるからです。

```js
console.log(Array(5));    // [empty × 5]
console.log(Array("5"));  // ["5"]
console.log(Array(true)); // [true]
console.log(Array(5, 5)); // [5, 5]
```

このように、数値が一つだけ与えられた場合のみ「その長さを持つ配列を作る」という意味になり、それ以外の場合は全て「与えられた値を要素に持つ配列を作る」という意味になります。

ところで、問題の`Array(5)`の結果が`[empty × 5]`となっています（Chromeの場合の表示）。これは、「中身は何も入っていないが長さが5の配列」ということです。JavaScriptでは配列のオブジェクトの一種であり配列の0番目の要素は`"0"`というプロパティの値である（なので`arr[0]`は`arr["0"]`と同じ意味）であることが知られていますが、ここでいう「中身は何も入っていない」とはプロパティすら存在しないということを意味しています。

```js
const arr = Array(5);
console.log(arr.length); // 5
console.log(arr[0]);     // undefined
console.log(arr.hasOwnProperty("0")); // false
```

念のために注意しておくと、emptyは「`undefined`が入っている」とは異なります（プロパティが存在するかしないかの点で）。

```js
const arr = Array(5);
arr[1] = undefined;

console.log(arr); // [empty, undefined, empty × 3]

console.log(arr.hasOwnProperty("0")); // false
console.log(arr.hasOwnProperty("1")); // true
```


### emptyの難解な挙動

この、「`0`〜`arr.length-1`の間にあるがプロパティが存在しない要素」のことは俗にemptyと呼ばれています。emptyの挙動はなかなか非直感的で、熟練のJavaScript使いでもその仕様を諳んじるのは簡単ではありません。

例えば、pushするとemptyの後ろにpushされます。

```js
const arr = Array(5);
arr.push(123);
console.log(arr); // [empty × 5, 123]
```

emptyは、`forEach`や`map`では無視されます（`map`ではemptyに対して関数が呼び出されず、結果の配列ではemptyのままです）。

```js
const arr = Array(5);
arr.push(123);
console.log(arr); // [empty × 5, 123]

// 123 だけ表示される
arr.forEach(x => console.log(x));

const arr2 = arr.map(x => x * 10);
console.log(arr2); // [empty × 5, 1230]

const arr3 = arr.map((_, i) => i);
console.log(arr3); // [empty × 5, 5]
```

`some`や`every`でもemptyは判定対象に入りません。

```js
const arr = Array(5);
arr.push(123);
console.log(arr.every(x => x === 123)); // true
```

なお、愚直にカウンタ関数を使う`for`文の場合は、当然ながらemptyは無視されません（emptyの位置にインデックスアクセスした場合、「オブジェクトの存在しないプロパティにアクセスすると`undefined`になる」ルールに従って`undefined`となります）。

```js
const arr = Array(5);
arr.push(123);

// undefined undefined undefined
// undefind undefined 123
// と表示される
for (let i = 0; i < arr.length; i++) {
  console.log(arr[i]);
}
```

とはいっても、`indexOf`で`undefined`を探す場合、emptyは`undefined`として扱われません。

```js
const arr = Array(5);

console.log(arr.indexOf(undefined)); // -1
```

### ES2015以降のメソッドはemptyを扱える

配列のメソッドの中でも、ES2015で追加されたメソッドはemptyを感知できる（`undefined`が入っている要素と同じ扱いをする）傾向にあります。

```js
const arr = Array(5);

console.log(arr.findIndex(x => x === undefined)); // 0
console.log(arr.includes(undefined)); // true
```

また、`fill`を使うとempty部分を他の値で上書きできます。

```js
const arr = Array(5);
arr.fill(123);

console.log(arr); // [123, 123, 123, 123, 123]
```

以上の情報から、問題の`[0, 1, 2, 3, 4]`を作るのに次のようなステップを踏むという方法もあります（これもやめるべきだと思いますが）。

```js
const arr = Array(5).fill(0).map((_, i) => i);
console.log(arr); // [0, 1, 2, 3, 4]
```

これは、`[empty × 5]`を`fill`で`[0, 0, 0, 0, 0]`に変換し、それを`map`で`[0, 1, 2, 3, 4]`に変換しています（先述の通り、`[empty × 5]`に直接`map`を使っても`[empty × 5]`のままです）。

### イテレータ系のメソッド

配列のイテレータ系のメソッド（`keys`, `values`, `entries`）もやはりES2015で追加されたもので、emptyを無視しません（`undefined`が入っている要素と同じように扱います）。

```js
const arr = Array(5);

console.log([...arr.values()]); // [undefined, undefined, undefined, undefined, undefined]

// [[0, undefined], [1, undefined], [2, undefined],
//  [3, undefined], [4, undefined]]
console.log([...arr.entries()]);

console.log([...arr.keys()]); // [0, 1, 2, 3, 4]
```

ここで`keys()`が登場しましたね。これは配列の各要素のキー名を順に出力するイテレータを返しますが、emptyであっても無視しないので、`[empty × 5]`であっても`[undefined, undefined, undefined, undefined, undefined]`と同じように扱い、0〜4の要素が存在しているかのように振舞います。これが`[...arr.keys()]`が`[0, 1, 2, 3, 4]`となる理由です。

ここで表題の`[...Array(5).keys()]`の意味が分かりましたね。まず`Array(5)`で`[empty × 5]`を作り、keys()でそこから`0, 1, 2, 3, 4`を出力するイテレータを得て、`...`でそれを配列に変換して`[0, 1, 2, 3, 4]`を得ています。

以上の挙動を理解するには、`Array(5)`がemptyから成る配列を作ることを理解する必要があり、その上で`keys()`はemptyも`undefinde`と同様に扱うことを知っている必要があります。これは非常にややこしいので、避けるべきです。

## ではどうするのが良いか

筆者のお勧めの方法はこれです。

```ts
console.log([...range(0, 5)]); // [0, 1, 2, 3, 4]

/**
 * Returns an iterator that iterates integers in [start, end).
 */
function* range(start, end) {
  for (let i = start; i < end; i++) {
    yield i;
  }
}
```

`Array(n).keys()`は上で定義した`range`を使って`range(0, n)`と書くことができます。
こちらのほうが`range`にコメントを書くこともできるし、上述のような機構を一々思い出す必要もないので理解の負荷が低いと考えられます。
`range`がイテレータではなく配列を返すようにすれば`[... ]`も省いてさらに簡潔にすることもできますが、それは`range`の汎用性を下げるし本題とは関係ないのでやりませんでした。

## `Number.range`

ちなみに、`Number.range`をECMAScript仕様に追加しようというプロポーザルもあります。

- [Number.range & BigInt.range](https://github.com/tc39/proposal-Number.range)

これが導入されれば、次のように書けるでしょう。`Array(5).keys()`より微妙に長いですが、読みやすさのためなら長さなど些細な問題です（コードゴルフをしたい場合は別ですが）。

```js
console.log([...Number.range(0, 5)]);
```

## まとめ

`[...Array(5).keys()]`のような書き方は、意味を認識するために知っているべき事項が多くてややこしいので、読み手の負荷を下げるためには避けるべきです。代わりに、あらかじめ`range`のような関数を定義しておいて`[...range(0, 5)]`とするのがよいと思います。

### 余談: パフォーマンスの話

この記事で扱ったようなemptyを含む配列は、emptyを含まない配列に比べてパフォーマンス上不利であることが知られています（[例えばV8のブログ記事をご覧ください](https://v8.dev/blog/elements-kinds)）。この記事では理解しやすさの観点から議論しますが、パフォーマンスの観点からもemptyを含む配列を無闇に扱うべきではないことが分かります。
