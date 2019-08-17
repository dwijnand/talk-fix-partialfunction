# How to totally fix PartialFunction

[F1-scaladoc]: https://www.scala-lang.org/api/2.13.0/scala/Function1.html
[PF-scaladoc]: https://www.scala-lang.org/api/2.13.0/scala/PartialFunction.html

[F1-source]: https://github.com/scala/scala/blob/v2.13.0/src/library/scala/Function1.scala#L65
[PF-source]: https://github.com/scala/scala/blob/v2.13.0/src/library/scala/PartialFunction.scala

```scala
trait Function1[-A, +B] {
  def apply(a: A): B
}
```

```scala
trait PartialFunction[-A, +B] extends (A => B) {
  def apply(a: A): B
  def isDefinedAt(a: A): Boolean
}
```

```scala
type A => B => Function1[A, B]
// `=>` is right-associative,
// so `A => B => C` means `A => (B => C)`

type A ?=> B  = PartialFunction[A, B] // ideally...
type A ?=>: B = PartialFunction[A, B] // for today...
```

```scala
trait Function1[-A, +B] {
  def apply(a: A): B

  def andThen[C](g: B => C): A => C = a => g(apply(a))
  def compose[R](g: R => A): R => B = r => apply(g(r))

  def unlift[T](implicit ev: B <:< Option[T]): A ?=>: T = ??? // ext. method
}
```

```scala
trait PartialFunction[-A, +B] extends (A => B) {  
  def apply(a: A): B             = if (isDefinedAt(a)) { ... } else throw new MatchError(...)
  def isDefinedAt(a: A): Boolean = ...

  def andThen[C](g: B  =>  C): A ?=>: C
  def andThen[C](g: B ?=>: C): A ?=>: C
  def compose[R](g: R ?=>: A): R ?=>: B
  def orElse[A1 < A, B1 >: B](g: A1 ?=>: A2): A1 ?=>: B1
  def lift: A => Option[B]

  def applyOrElse[A1 <: A, B1 >: B](a: A1, default: A1 => B1): B1 =
    if (isDefinedAt(a)) apply(a) else default(a)

  def runWith[U](action: B => U): A => Boolean =
    (a: A) => {
      val b: B = applyOrElse(a, checkFallback[B])
      if (fallbackOccurred(b)) false else {
        action(b)
        true
      }
    }

  def unapply(a: A): Option[B]                = lift(a)
  def elementWise: ElementWiseExtractor[A, B] = new ElementWiseExtractor(this)
}
```

```scala
object PartialFunction {
  def empty[A, B]: A ?=>: B

  def fromFunction[A, B](f: A => B): A ?=>: B = { case a => f(a) }

  def cond[A](a: A)(pf: A ?=>: Boolean): Boolean    = pf.applyOrElse(a, constFalse)
  def condOpt[A, B](a: A)(pf: A ?=>: B]): Option[B] = pf.lift(a)

  private val fallback_fn: Any => Any            = _ => fallback_fn
  private def checkFallback[B]: Any => B         = fallback_fn.asInstanceOf[Any => B]
  private def fallbackOccurred[B](b: B): Boolean = fallback_fn eq b.asInstanceOf[AnyRef]
}
```

```scala
trait Function2[-A, -B, +R] {
  def apply(a: A, B: B): R

  def curried: A => B => R   = { a => b        => apply(a, b) }
  def  tupled: ((A, B)) => R = { case ((a, b)) => apply(a, b) }
}
```

```scala
object Function {
  def chain[A](fs: sc.Seq[A => A]): A => A = a => fs.foldLeft(a)((a, f) => f(a))
  def const[A, U](a: A)(_: U): A           = a
  def unlift(A => Option[B]): A ?=>: B

  def  untupled[A, B, R](f: ((A, B)) => R): (A, B) => R = (a, b) => f((a, b))
  def uncurried[A, B, R](f:  A => B  => R): (A, B) => R = (a, b) =>  f(a)(b)
  // ... up to 5 params (A-T5)

  // Also tupled, slotted for deprecation.. once type inferrence of tupled anonymous functions improves
  def tupled[A, B, R](f: (A, B) => R): ((A, B)) => R = { case ((a, b)) => f(a, b) }
}
```

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
