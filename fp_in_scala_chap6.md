# 第6章 純粋関数の状態

乱数生成 (random number generation, RNG) を例に， `状態` を操作する純粋関数型のプログラムを記述する方法を考える．


## 6.1 副作用を使った乱数の生成
標準ライブラリの乱数生成器を見てみる．

```
scala > val rng = new scala.util.Random 　

scala > rng.nextDouble 
res 0: Double = 0.9867076608154569 

scala > rng.nextDouble 
res 1: Double = 0.8455696498024141 

scala > rng.nextInt 
res 2: Int = -623297295 

scala > rng.nextInt(10)
res 3: Int = 4
```

`nextInt` などが呼ばれるたびに `res` の内容が変わっていることから、
`nextInt` には内部状態が存在し、それが変化しているのだとわかる 
(→ 参照透過ではない → テストむずい → 死)．

本当にテストが難しいのか見てみる．

```
// rollDiceじゃねーのと思ったら、dieはdiceの単数形 (最近は単複どっちもdiceだけど)
def rollDie: Int = { 
    val rng = new scala.util.Random 
    rng.nextInt(6) // 0〜5 の乱数を返す 
}
```

この関数は、`[0, 5]` を返すので、本来期待する `[1, 6]` を返す関数にはなっていない．
しかし、テストケースのうち，1, 2, 3, 4, 5 を出すことは確認でき (テストケースをパスし)、
`0を出力しないこと` 、 `6を出力すること` のケースで失敗する．
この状況を再現させること (つまり、 `全く同じ状態` の rng を用意すること) は容易ではない．


## 6.2 純粋関数型の乱数の生成
参照透過な乱数生成器を作ることを考える．

```
trait RNG {
    def nextInt: (Int, RNG)  // 乱数と、新しい状態を持った乱数生成器を返す
}
```

上記のように、新しい状態を、乱数と一緒に返すようにすれば、
たとえば、テストの時に、ケースをパスしない状態を再現することが容易になる．

[線形合同法](https://ja.wikipedia.org/wiki/%E7%B7%9A%E5%BD%A2%E5%90%88%E5%90%8C%E6%B3%95) 
の乱数生成器の実装を以下に示す．
(おまけ 線形合同法のPros/Cons: Pros: 計算が軽い(高速 \& 使用メモリが少ない (けどもっといいやつある)). Cons: 出力は乱数として残念 → 良い子は別のアルゴリズムを使おうな！)

```
case class SimpleRNG(seed: Long) extends RNG { 
    def nextInt: (Int, RNG) = { 
        // 線形合同法の式漸化式 ax + b mod M
        val newSeed = (seed * 0x5DEECE66DL + 0xBL) & 0xFFFFFFFFFFFFL
    
        val nextRNG = SimpleRNG(newSeed)
        val n = (newSeed >>> 16).toInt
        (n, nextRNG)
    } 
}
```

これを数回実行して、得られる出力列は、同様に実行する限り、常に同様である．

## 6.3 ステートフルAPIの純粋化
APIが何かを変化させる (状態を持ってしまって辛い) → APIに `次の状態を計算させる` (状態がなくなってハッピー)は、
一般に使える解決策である．

これを

```
class Foo { 
    private var s: FooState = ... 
    def bar: Bar 
    def baz: Int 
}
```

こう！

```
trait Foo { 
    def bar: (Bar, Foo) 
    def baz: (Int, Foo) 
}
```

(ただし、APIの呼び出し元が、ちゃんと、返された次の状態をプログラムの他の部分に渡す責任を帯びるけどな！)


### Exercise 6.1
i in [0, Int.Max] (iは整数) を返す関数を作る．
```
def nonNegativeInt(rng: RNG): (Int, RNG) = {
    val (i, newRng) = rng.nextInt
    val nni = if (i < 0) -(i + 1) else i  // 負の数だったら、符号ビットを逆転させる
    (nni, newRng)
}
```

参考
```
scala> Int.MaxValue
res0: Int = 2147483647

scala> Int.MinValue
res1: Int = -2147483648
```

### Exercise 6.2
[0, 1) の範囲のDouble型を返す関数を作る．
```
def double(rng: RNG): (Double, RNG) = {
    val (i, newRng) = nonNegativeInt(rng)
    val zeroToOne = i / (Int.MaxValue.toDouble + 1)  // 生成された整数をintMax + 1で割る
    (zeroToOne, newRng)
}
```

### Exercise 6.3
(Int, Double), (Double, Int), (Double, Double, Double)を返す関数を作る．
```
def intDouble(rng: RNG): ((Int, Double), RNG) = {
    val (i, rng1) = rng.nextInt
    val (d, rng2) = double(rng1)
    ((i, d), rng2)
}

def doubleInt(rng: RNG): ((Int, Double), RNG) = {
    val ((i, d), rng1) = intDouble(rng)
    ((d, i), rng1)
}

def double3(rng: RNG): ((Double, Double, Double), RNG) = {
    val (d1, rng1) = double(rng)
    val (d2, rng2) = double(rng1)
    val (d3, rng3) = double(rng2)
    ((d1, d2, d3), rng3)
}
```

### Exercise 6.4
ランダムな整数のリストを返す関数を作る
```
// 普通にやるver.
def ints(count: Int)(rng: RNG): (List[Int], RNG) = {
    if (count == 0) {
        (List(), rng)
    } else {
        val (x, rng1) = rng.nextInt
        val (xs, rng2) = ints(count - 1)(rng1)
        ((x::xs), rng2)
    }
}

// tailrec ver.
def ints_tail_rec(count: Int)(rng: RNG): (List[Int], RNG) = {
    @tailrec
    def go(count: Int, r:RNG, xs: List[Int]): (List[Int], RNG) = {
        if (count == 0) {
            (xs, r)
        } else {
            val (x, r1) = r.nextInt // 整数を取得
            go(count - 1, r1, x::xs)  // カウントを減らして、整数これまでのリストに今取得した整数をくっつける
        }
    }
    go(count, rng, List())
}
```


## 6.4 状態の処理に適したAPI
これまでに定義した純粋関数は全て、型Aに対して、RNG → (A, RNG) という型が使われていた (こういう関数を状態アクションと呼ぶ)．
このとき、ユーザにとって本質的ではない「状態を渡す部分」を無くしたい．
状態の受け渡しを `コンビネータ` を使って自動化することを考える．

まず、状態アクションデータ型の型エイリアスを作る
```
type Rand[+A] = RNG => (A, RNG)
```
この型の値は、「RNGを使ってAを作り、RNGを新しい状態に遷移させるもの」と考えることができる．

これを使ってnextIntをこの型の値にする．
```
val int: Rand[Int] = _.nextInt
```

`rng` を変更せず、引数で与えられた数値を返し続ける関数．
```
def unit[A](a: A): Rand[A] = 
    rng => (a, rng)
```

`f` によって出力を変換する `map`．
```
def map[A, B](s: Rand[A])(f: A => B): Rand[B] = 
    rng => {
        val (a, rng2) = s(rng)
        (f(a), rng2)
    }
```

これを使って0以上の偶数を出力する関数を作る．
```
def nonNegativeEven: Rand[Int] = 
    // i % 2 はiの2の剰余なので、奇数の場合は1引けば良い
    map(nonNegativeInt)(i => i - (i % 2))
```

### Exercise 6.5
mapを使ってExercise6.2のdouble ([0,1)を返すやつ) をいい感じにする．

```
val _double: Rand[Double] =
    map(nonNegativeInt)(_ / (Int.MaxValue.toDouble + 1))
```

### 6.4.1 状態アクションの結合
mapは1つしか引数をとれないので、2つ引数をとれるmap2をつくる．


### Exercise 6.6
```
def map2[A, B, C](ra: Rand[A], rb: Rand[B])(f: (A, B) => C): Rand[C] =
    rng => {
        val (a, r1) = ra(rng)
        val (b, r2) = rb(r1)
        (f(a, b), r2)
    }
}
```

上記を使えば、任意の状態アクションを結合できるようになる．
例えば、A型を出力するアクションと, B型を出力するアクションをまとめて、
A, Bのペアを生成するアクションを作る．

```
def both[A, B](ra: Rand[A], rb: Rand[B]): Rand[A, B] = 
    map2(ra, rb)((_, _))
```

これを使ってExercise6.3のintDouble, doubleIntを書き換えると

```
val randIntDouble: Rand[(Int, Double)] = both(int, double)
val randDoubleInt: Rand[(Double, Int)] = both(double, int)
```

### Exercise 6.7
```
def sequence[A](fs: List[Rand[A]]): Rand[List[A]] =
    // 初期値: A型の空のリストのunit
    // 適用する関数: fとaccを引数として、fとaccを結合する
    fs.foldRight(unit(List[A]()))((f, acc) => map2(f, acc)(_ :: _))
    
def ints(count: Int): Rand[List[Int]] =
    sequence(List.fill(count)(int))
```

### 6.4.2 入れ子状態のアクション
map, map2を使って，rngを明示的に渡さなくてもすむようになってきた．
ただ、これでもまだうまく記述できない関数がある．
[0, n)を返すnonNegativeLessThanを考える．

まずは、自然数についてnの剰余をとることを考えてみる．
```
def nonNegativeLessThan(n: Int): Rand[Int] =
    map(nonNegativeInt) { _ % n }
```

これだと、Int.MaxValueのnの剰余mが0以外であった場合に、[0, m]が、(m, n]に比べて多く出てしまうため、
RNGとして良くない性質を帯びる．
そこで、以下のような変更を加える．

```
def nonNegativeLessThan(n: Int): Rand[Int] = 
    map(nonNegativeInt) {i => 
        val mod = i % n
        if (i + (n - 1) - mod >= 0) {  // ここ負にならなくない？
            mod
        } else {
            nonNegativeLessThan(n)(???) 1
        }
    }
```

nonnNegativeLessThanにRNGを渡すように変更する
```
def nonNegativeLessThan(n: Int): Rand[Int] = { rng => 
    val (i, rng2) = nonNegativeInt(rng)
    val mod = i % n
    if (i + (n - 1) - mod >= 0) {
        (mod, rng2)
    } else {
        nonNegativeLessThan(n)(rng2)  // retryなので、rng2を渡す (本の間違い)
    }
}
```

RNGを毎回渡すのはだるいので、flatMapが欲しくなる

### Exercise 6.8
flatMapを作る
```
def flatMap[A,B](f: Rand[A])(g: A => Rand[B]) : Rand[B] = 
    rng => {
        val (a, r1) = f(rng)
        g(a)(r1)
    }

def nonNegativeLessThan(n: Int): Rand[Int] = {
    flatMap(nonNegativeInt) { i => 
        val mod = i % n
        if (i + (n - 1) - mod >= 0) unit(mod)
        else nonNegativeLessThan(n)
    }
}
```

### Exercise 6.9
mapとmap2をflatMapを使って実装する
```
def map[A, B](s: Rand[A])(f: A => B): Rand[B] = 
    flatMap(s)(a => unit(f(a)))
def map2[A, B, C](ra: Rand[A], rb: Rand[B])(f: (A, B) => C): Rand[C] =
    flatMap(ra)(a => map(rb))(b => f(a, b))
```

冒頭で示したサイコロの例をnonNegativeLessThanを使ってもう一度示してみると、

```
def rollDie: Rand[Int] = nonNegativeLessThan(6)
```
と実装したとして、0 を返す場合のRNGをSimpleRNG(5)とすることで再現できる．
そして、バグの修正は、

```
def rollRie: Rand[Int] = map(nonNegativeLessThan(6))(_ + 1)
```

とすること．


## 6.5 状態アクションデータ型の一般化

