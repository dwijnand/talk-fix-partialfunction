
```scala
xs.collect { case s if s.startsWith("FOO_") => s }

def receive = {
  case Deposit(amt)  => ???
  case Withdraw(amt) => ???
  // missing: CheckBalance
}
```

```scala
scala> List("FOO_A", "BAR_B").map { case s if s.startsWith("FOO_") => s }
scala.MatchError: BAR_B (of class java.lang.String)
  at .$anonfun$res1$1(<console>:1)
  at scala.collection.immutable.List.map(List.scala:226)
  ... 52 elided
```

```scala
scala> Try[Int](???).recover(_ => 1)
                               ^
       error: type mismatch;
        found   : Throwable => Int
        required: PartialFunction[Throwable,?]

scala> Try[Int](???).recover { case _ => 1 }
res1: scala.util.Try[Int] = Success(1)
```

"Synthesize a PartialFunction from function literal" [scala/scala#8172][], coming in 2.13.1.

[scala/scala#8172]: https://github.com/scala/scala/pull/8172

```scala
trait PartialFunction[-A, +B] {
  def isDefinedAt(a: A): Boolean = ...

  /** Must not be called if `isDefinedAt` isn't `true` for `a`! */
  def unsafeApply(a: A): B = ...

  def andThen[C](g: B ?=>: C): A ?=>: C
  def compose[Z](g: R ?=>: A): R ?=>: B
  def orElse[A1 <: A, B1 >: B](g: A1 ?=>: B1): A1 ?=>: B1
  def lift: A => Option[B]

  def applyOrElse[A1 <: A, B1 >: B](a: A1, default: A1 => B1): B1 =
    if (isDefinedAt(a)) unsafeApply(a) else default(a)

  def runWith[U](action: B => U): A => Boolean =
    (a: A) => {
      val b: B = applyOrElse(a, checkFallback[B])
      if (fallbackOccurred(b)) false else {
        action(b)
        true
      }
    }
}

trait Function1[-A, +B] extends PF[A, B] {
  final def isDefinedAt(a: A) = true
  final def apply(a: A): B    = unsafeApply(a)

  def andThen[C](g: B => C): A => C = a => g(apply(a))
  def compose[R](g: R => A): R => B = r => apply(g(r))
}
```
