# How to totally fix PartialFunction

```scala
trait Function1[-A, +B] {
  def apply(x: A): B
}

// aka: A => B
// Note: A => B => C means A => (B => C)  (right associativity)
```

```scala
trait PartialFunction[-A, +B] extends (A => B) {
  def apply(x: A): B
  def isDefinedAt(x: A): Boolean
}
```

```scala
trait Function1[-A, +B] {
  def apply(x: A): B

  def andThen[C](g: B => C): A => C = { x => g(apply(x)) }
  def compose[R](g: R => A): R => B = { x => apply(g(x)) }

  def unlift[T](implicit ev: B <:< Option[T]): A ?=> T = ??? // ext. method
}
```

```scala
trait PartialFunction[-A, +B] extends (A => B) {
  def apply(v1: A): B
  def isDefinedAt(x: A): Boolean

  def andThen[C](k: B  => C): A ?=> C
  def andThen[C](k: B ?=> C): A ?=> C

  def compose[R](k: R ?=> A): R ?=> B

  def orElse(that: A ?=> B): A ?=> B
//def orElse[A1 < A, B1 >: B](that: A1 ?=> A2): A1 ?=> B1

  def lift: A => Option[B]

  def applyOrElse(x: A, default: A => B): B = if (isDefinedAt(x)) apply(x) else default(x)
//def applyOrElse[A1 <: A, B1 >: B](x: A1, default: A1 => B1): B1

  def runWith[U](action: B => U): A => Boolean = { x =>
    val z = applyOrElse(x, checkFallback[B])
    if (fallbackOccurred(z)) false else {
      action(z)
      true
    }
  }

  def unapply(a: A): Option[B]                = lift(a)
  def elementWise: ElementWiseExtractor[A, B] = new ElementWiseExtractor(this)
}
```

```scala
object PartialFunction {
  def empty[A, B]: A ?=> B

  def fromFunction[A, B](f: A => B): A ?=> B = { case x => f(x) }

  def cond[T](x: T)(pf: T ?=> Boolean): Boolean    = pf.applyOrElse(x, constFalse)
  def condOpt[T, U](x: T)(pf: T ?=> U]): Option[U] = pf.lift(x)

  private[this] val fallback_fn: Any => Any   = _ => fallback_fn
  private       def checkFallback[B]          = fallback_fn.asInstanceOf[Any => B]
  private       def fallbackOccurred[B](x: B) = fallback_fn eq x.asInstanceOf[AnyRef]
}
```

```scala
trait Function2[-T1, -T2, +R] {
  def apply(v1: T1, v2: T2): R

  def curried:  T1 => T2  => R = { (x1: T1) => (x2: T2) => apply(x1, x2) }
  def  tupled: ((T1, T2)) => R = { case ((x1, x2))      => apply(x1, x2) }
}
```

```scala
object Function {
  def chain[T](fs: sc.Seq[T => T]): T => T = { x => fs.foldLeft(x)((x, f) => f(x)) }
  def const[T, U](x: T)(y: U): T = x
  def unlift(A => Option[B]): A ?=> B

  def  untupled[T1, T2, R](f: ((T1, T2)) => R):  (T1, T2)  => R = {       (x1, x2)  => f((x1, x2)) }
  def uncurried[T1, T2, R](f:  T1 => T2  => R):  (T1, T2)  => R = {       (x1, x2)  =>  f(x1)(x2)  }
  // ... up to 5 params (T1-T5)

  // Also tupled, slotted for deprecation.. once type inferrence of tupled anonymous functions improves
  def tupled[T1, T2, R](f: (T1, T2) => R): ((T1, T2)) => R = { case ((x1, x2)) => f(x1, x2) }
}
```


[F1-scaladoc]: https://www.scala-lang.org/api/2.13.0/scala/Function1.html
[PF-scaladoc]: https://www.scala-lang.org/api/2.13.0/scala/PartialFunction.html

[F1-source]: https://github.com/scala/scala/blob/v2.13.0/src/library/scala/Function1.scala#L65
[PF-source]: https://github.com/scala/scala/blob/v2.13.0/src/library/scala/PartialFunction.scala

