# How to totally fix PartialFunction

```scala
trait Function1[-T1, +R] {
  def apply(v1: T1): R
}
```

```scala
trait PartialFunction[-A, +B] extends (A => B) {
  def apply(v1: A): B
  def isDefinedAt(x: A): Boolean
}
```

```scala
def compose[A](g: A => T1): A => R = { x => apply(g(x)) }
def andThen[A](g: R => A): T1 => A = { x => g(apply(x)) }
```

```scala
def orElse(that: A ?=> B): A ?=> B
def orElse[A1 < A, B1 >: B](that: A1 ?=> A2): A1 ?=> B1

def andThen[C](k: B  => C): A ?=> C
def andThen[C](k: B ?=> C): A ?=> C

def compose[R](k: R ?=> A): R ?=> B

def lift: A => Option[B]
def unapply(a: A): Option[B]          = lift(a)
def elementWise: ElementWiseExtractor = new ElementWiseExtractor(this)

def applyOrElse(x: A, default:   => B): B = if (isDefinedAt(x)) apply(x) else default
def applyOrElse(x: A, default: A => B): B = if (isDefinedAt(x)) apply(x) else default(x)

def applyOrElse                  (x: A,  default:    => B ): B  = if (isDefinedAt(x)) apply(x) else default
def applyOrElse                  (x: A,  default: A  => B ): B  = if (isDefinedAt(x)) apply(x) else default(x)
def applyOrElse[A1 <: A, B1 >: B](x: A1, default: A1 => B1): B1 = if (isDefinedAt(x)) apply(x) else default(x)

def runWith[U](action: B => U): A => Boolean = { x =>
  val z = applyOrElse(x, checkFallback[B])
  if (fallbackOccurred(z)) false else {
    action(z)
    true
  }
}
```

```scala
object PartialFunction {
  private[this] val fallback_fn: Any => Any   = _ => fallback_fn
  private       def checkFallback[B]          = fallback_fn.asInstanceOf[Any => B]
  private       def fallbackOccurred[B](x: B) = fallback_fn eq x.asInstanceOf[AnyRef]
}
```

[F1-scaladoc]: https://www.scala-lang.org/api/2.13.0/scala/Function1.html
[PF-scaladoc]: https://www.scala-lang.org/api/2.13.0/scala/PartialFunction.html

[F1-source]: https://github.com/scala/scala/blob/v2.13.0/src/library/scala/Function1.scala#L65
[PF-source]: https://github.com/scala/scala/blob/v2.13.0/src/library/scala/PartialFunction.scala

