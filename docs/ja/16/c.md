---
out: Effect-system.html
---

### エフェクトシステム

[Lazy Functional State Threads](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.144.2237&rep=rep1&type=pdf) において John Launchbury さんと Simon Peyton-Jones さん曰く:

> Based on earlier work on monads, we present a way of securely encapsulating stateful computations that manipulate multiple, named, mutable objects, in the context of a non-strict purely-functional language.

Scala には `var` があるので一見すると不必要にも思われるけど、stateful な計算をカプセル化するという考え方は役に立つことがある。並列に実行される計算など特殊な状況下では、状態が共有されないかもしくは慎重に共有されているかどうかが正誤を分ける。

### ST

この論文で説明される `ST` に対応するものとして Scalaz には `ST` モナドがある。Rúnar さんの [Towards an Effect System in Scala, Part 1: ST Monad](http://apocalisp.wordpress.com/2011/03/20/towards-an-effect-system-in-scala-part-1/) も参照。以下が [`ST`]($scalazBaseUrl$/effect/src/main/scala/scalaz/effect/ST.scala) のコントラクトだ:

```scala
sealed trait ST[S, A] {
  private[effect] def apply(s: World[S]): (World[S], A)
}
```

`State` モナドに似ているけども、違いは状態が可変で上書きされていることと、その引き換えとして状態が外から観測できないことにある。

### STRef

LFST:

> What, then is a "state"? Part of every state is a finite mapping from *reference* to values. ... A reference can be thought of as the name of (or address of) a *variable*, an updatable location in the state capable of holding a value.

`STRef` は `ST` モナドのコンテキストの内部でしか使えない可変変数だ。`ST.newVar[A]` によって作られ以下の演算をサポートする:

```scala
sealed trait STRef[S, A] {
  protected var value: A

  /**Reads the value pointed at by this reference. */
  def read: ST[S, A] = returnST(value)
  /**Modifies the value at this reference with the given function. */
  def mod[B](f: A => A): ST[S, STRef[S, A]] = ...
  /**Associates this reference with the given value. */
  def write(a: => A): ST[S, STRef[S, A]] = ...
  /**Synonym for write*/
  def |=(a: => A): ST[S, STRef[S, A]] = ...
  /**Swap the value at this reference with the value at another. */
  def swap(that: STRef[S, A]): ST[S, Unit] = ...
}
```

自家版の Scalaz 7 を使う:

```scala
\$ sbt
scalaz> project effect
scalaz-effect> console
[info] Compiling 2 Scala sources to /Users/eed3si9n/work/scalaz-seven/effect/target/scala-2.9.2/classes...
[info] Starting scala interpreter...
[info]

scala> import scalaz._, Scalaz._, effect._, ST._
import scalaz._
import Scalaz._
import effect._
import ST._

scala> def e1[S] = for {
         x <- newVar[S](0)
         r <- x mod {_ + 1}
       } yield x
e1: [S]=> scalaz.effect.ST[S,scalaz.effect.STRef[S,Int]]

scala> def e2[S]: ST[S, Int] = for {
         x <- e1[S]
         r <- x.read
       } yield r 
e2: [S]=> scalaz.effect.ST[S,Int]

scala> type ForallST[A] = Forall[({type λ[S] = ST[S, A]})#λ]
defined type alias ForallST

scala> runST(new ForallST[Int] { def apply[S] = e2[S] })
res5: Int = 1
```

Rúnar さんのブログに [Paul Chiusano さん (@pchiusano)](http://twitter.com/pchiusano) が皆が思っていることを言っている:

> 僕はこれの Scala での効用について決めかねているんだけど - わざと反対の立場をとってるんだけど - もし (例えば quicksort) なんらかのアルゴリズムを実装するためにローカルで可変状態が必要なら、関数に渡されたものさえ上書き変更しなければいいだけだ。これを正しくやったとコンパイラをわざわざ説得する意義はある? 別にここでコンパイラの助けを借りなくてもいいと思う。

30分後に帰ってきて、自分の問に答えている:

> もし僕が命令形の quicksort を書いているなら、インプットの列を配列にコピーしてソートの最中に上書き変更して、結果をソートされた配列にたいする不変な列のビューとして返すだろう。STRef を使うと、可変配列に対する STRef を受け取ってコピーを一切避けることができる。さらに、僕の命令形のアクションは第一級となるのでいつものコンビネータを使って合成することができるようになる。

これは面白い点だ。可変状態が漏洩しないことが保証されているため、データのコピーをせずに可変状態の変化を連鎖して合成することができる。可変状態が必要な場合の多くは配列が必要な場合が多い。そのため `STArray` という配列へのラッパーがある:

```scala
sealed trait STArray[S, A] {
  val size: Int
  val z: A
  private val value: Array[A] = Array.fill(size)(z)
  /**Reads the value at the given index. */
  def read(i: Int): ST[S, A] = returnST(value(i))
  /**Writes the given value to the array, at the given offset. */
  def write(i: Int, a: A): ST[S, STArray[S, A]] = ...
  /**Turns a mutable array into an immutable one which is safe to return. */
  def freeze: ST[S, ImmutableArray[A]] = ...
  /**Fill this array from the given association list. */
  def fill[B](f: (A, B) => A, xs: Traversable[(Int, B)]): ST[S, Unit] = ...
  /**Combine the given value with the value at the given index, using the given function. */
  def update[B](f: (A, B) => A, i: Int, v: B) = ...
}
```

これは `ST.newArr(size: Int, z: A` を用いて作られる。エラトステネスのふるいを使って 1000 以下の全ての素数を計算してみよう...

### 速報

`STArray` にバグを見つけたので、さっさと直すことにする:

```
\$ git pull --rebase
Current branch scalaz-seven is up to date.
\$ git branch topic/starrayfix
\$ git co topic/starrayfix
Switched to branch 'topic/starrayfix'
```

`ST` にスペックが無いので、バグを再現するためにもスペックを書き起こすことにする。これで誰かが僕の修正を戻してもバグが捕獲できる。

```scala
package scalaz
package effect

import std.AllInstances._
import ST._

class STTest extends Spec {
  type ForallST[A] = Forall[({type λ[S] = ST[S, A]})#λ]

  "STRef" in {
    def e1[S] = for {
      x <- newVar[S](0)
      r <- x mod {_ + 1}
    } yield x
    def e2[S]: ST[S, Int] = for {
      x <- e1[S]
      r <- x.read
    } yield r
    runST(new ForallST[Int] { def apply[S] = e2[S] }) must be_===(1)
  }

  "STArray" in {
    def e1[S] = for {
      arr <- newArr[S, Boolean](3, true)
      _ <- arr.write(0, false)
      r <- arr.freeze
    } yield r
    runST(new ForallST[ImmutableArray[Boolean]] { def apply[S] = e1[S] }).toList must be_===(
      List(false, true, true))
  }
}
```

これが結果だ:

```
[info] STTest
[info] 
[info] + STRef
[error] ! STArray
[error]   NullPointerException: null (ArrayBuilder.scala:37)
[error] scala.collection.mutable.ArrayBuilder\$.make(ArrayBuilder.scala:37)
[error] scala.Array\$.newBuilder(Array.scala:52)
[error] scala.Array\$.fill(Array.scala:235)
[error] scalaz.effect.STArray\$class.\$init\$(ST.scala:71)
...
```

Scala で NullPointerException?! これは以下の `STArray` のコードからきている:

```scala
sealed trait STArray[S, A] {
  val size: Int
  val z: A
  implicit val manifest: Manifest[A]

  private val value: Array[A] = Array.fill(size)(z)
  ...
}
...
trait STArrayFunctions {
  def stArray[S, A](s: Int, a: A)(implicit m: Manifest[A]): STArray[S, A] = new STArray[S, A] {
    val size = s
    val z = a
    implicit val manifest = m
  }
}
```

見えたかな? Paulp さんがこれの [FAQ](https://github.com/paulp/scala-faq/wiki/Initialization-Order) を書いている。`value` が未初期化の `size` と `z` を使って初期化されている。以下が僕の加えた修正:

```scala
sealed trait STArray[S, A] {
  def size: Int
  def z: A
  implicit def manifest: Manifest[A]

  private lazy val value: Array[A] = Array.fill(size)(z)
  ...
}
```

これでテストは通過したので、push して [pull request](https://github.com/scalaz/scalaz/pull/155) を送る。

### Back to the usual programming

[エラトステネスのふるい](http://en.wikipedia.org/wiki/Sieve_of_Eratosthenes#Implementation) は素数を計算する簡単なアルゴリズムだ。

```scala
scala> import scalaz._, Scalaz._, effect._, ST._
import scalaz._
import Scalaz._
import effect._
import ST._

scala> def mapM[A, S, B](xs: List[A])(f: A => ST[S, B]): ST[S, List[B]] =
         Monad[({type λ[α] = ST[S, α]})#λ].sequence(xs map f)
mapM: [A, S, B](xs: List[A])(f: A => scalaz.effect.ST[S,B])scalaz.effect.ST[S,List[B]]

scala> def sieve[S](n: Int) = for {
         arr <- newArr[S, Boolean](n + 1, true)
         _ <- arr.write(0, false)
         _ <- arr.write(1, false)
         val nsq = (math.sqrt(n.toDouble).toInt + 1)
         _ <- mapM (1 |-> nsq) { i =>
           for {
             x <- arr.read(i)
             _ <-
               if (x) mapM (i * i |--> (i, n)) { j => arr.write(j, false) }
               else returnST[S, List[Boolean]] {Nil}
           } yield ()
         }
         r <- arr.freeze
       } yield r
sieve: [S](n: Int)scalaz.effect.ST[S,scalaz.ImmutableArray[Boolean]]

scala> type ForallST[A] = Forall[({type λ[S] = ST[S, A]})#λ]
defined type alias ForallST

scala> def prime(n: Int) =
         runST(new ForallST[ImmutableArray[Boolean]] { def apply[S] = sieve[S](n) }).toArray.
         zipWithIndex collect { case (true, x) => x }
prime: (n: Int)Array[Int]

scala> prime(1000)
res21: Array[Int] = Array(2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97, 101, 103, 107, 109, 113, 127, 131, 137, 139, 149, 151, 157, 163, 167, 173, 179, 181, 191, 193, 197, 199, 211, 223, 227, 229, 233, 239, 241, 251, 257, 263, 269, 271, 277, 281, 283, 293, 307, 311, 313, 317, 331, 337, 347, 349, 353, 359, 367, 373, 379, 383, 389, 397, 401, 409, 419, 421, 431, 433, 439, 443, 449, 457, 461, 463, 467, 479, 487, 491, 499, 503, 509, 521, 523, 541, 547, 557, 563, 569, 571, 577, 587, 593, 599, 601, 607, 613, 617, 619, 631, 641, 643, 647, 653, 659, 661, 673, 677, 683, 691, 701, 709, 719, 727, 733, 739, 743, 751, 757, 761, 769, 773, 787, 797, 809, 811, 821, 823, 827, 829, 839, 853, 857, 859, 863, 877, 881, 883, 887, 907, 911, 919, 929, 937, 941, ...
```

[最初の 1000個の素数のリスト](http://primes.utm.edu/lists/small/1000.txt)によると、結果は大丈夫みたいだ。`STArray` の反復を理解するのが一番難しかった。ここでは `ST[S, _]` のコンテキスト内にいるから、ループの結果も ST モナドである必要がある。リストを投射して配列に書き込むとそれは `List[ST[S, Unit]]` を返してしまう。

`ST[S, B]` へのモナディック関数を受け取ってモナドを裏返して `ST[S, List[B]]` を返す `mapM` を実装した。`sequence` と基本的には同じだけど、こっちの方が感覚的に理解しやすいと思う。`var` を使うのと比べると苦労無しとは言い難いけど、可変のコンテキストを渡せるのは役に立つかもしれない。

続きはまた後で。
