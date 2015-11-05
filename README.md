```scala
type Intent[-A, -B] =
  PartialFunction[HttpRequest[A], ResponseFunction[B]]

// A == f.eks HttpServletRequest
// B == f.eks HttpServletResponse

~=

type Intent = Request => Response

```

---

```scala
def intent = {
  case req @ POST(Path("/example"))) &
      Accepts.Json(RequestContentType("application/json")) =>
    Ok ~> JsonContent ~> ResponseBytes(Body.bytes(req))
}
```

---

## Problem
* hva er statuskode for /foo ?
* hva er statuskode for GET ?
* hva er statuskode for Accept: text/xml ?
* hva er statuskode for Content-Type: text/xml ?

---

## Løsning
```scala
def intent = {
  case req @ Path("/example") => req match {
    case POST(_) => req match {
      case RequestContentType("application/json") => req match {
        case Accepts.Json(_) =>
          Ok ~> JsonContent ~> ResponseBytes(Body.bytes(req))
        case _ => NotAcceptable
      }
      case _ => UnsupportedMediaType
    }
    case _ => MethodNotAllowed
  }
}
```
---

## Urk..
* `req` må sendes over alt
* vanskelig
* stygt
* ingen mulighet for gjenbruk / komposisjon

---

## hvorfor ?

```scala
object POST {
  // hva med MethodNotAllowed ?
  def unapply[A](r:Request):Option[Request] =
    if(r.method == "POST") Some(r) else None
}
```

---

## send request videre
```scala
def fix(f:PartialFunction[Request, Request => Response]):Intent =
  case r if f.isDefinedAt(r) => f(r)(r)

// r == Request
// f(r) == Request => Response
// f(r)(r) == Response  
```

---

## either
```scala
def fix(f:PartialFunction[Request, Request => Either[Response, Response]):Itent =
  case r if f.isDefinedAt(r) => f(r)(r).merge
```

---

```scala
case class Directive[A](run:Request => Either[Response, A]){
  def map[B](f:A => B):Directive[B] =
    Directive(r => run(r).map(f))

  def flatMap[B](f:A => Directive[B]):Directive[B] =
    Directive(r => run(r).flatMap(a => f(a).run(r)))
}
```

---

```scala
val POST = Directive(r =>
  if(r.method == "POST") Right(()) else Left(MethodNotAllowed))
val request = Directive(r => Right(r))
val bytes = request.map(r => Body.bytes(r))

def intent = fix{
  case Path("/example") => for {
    _ <- POST
    _ <- Accepts.Json
    b <- bytes
  } yield Ok ~> JsonContent ~> ResponseBytes(b)
}
```  

---

## Directives 2 - Hva med Async ?

```scala
type Intent = PartialFunction[Request, Future[Response]]
```

---

* jeg liker scalaz.concurrent.Task
* andre liker scala.concurrent.Future
* og noen ganger vil jeg ha blocking-io

---

```scala
type Intent[F[_]] = PartialFunction[Request, F[Response]]

F = Future | Task | Id | ...

PartialFunction[Request, Request => F[Either[Response, Response]]]
```

---

```scala
import scalaz._
import syntax.monad._

case class Directive[F[_], A](run:Request => F[Either[Response, A]]){
  def map[B](f:A => B)(implicit F:Functor[F]):Directive[F, B] =
    Directive[F, B](r => run(r).map(_.map(f)))

  def flatMap[B](f:A => Directive[F, B])(implicit F:Monad[F]):Directive[F, B] =
    Directive[F, B](r => run(r).flatMap{
        case Left(l) => Monad[F].point(Left(l))
        case Right(r) => f(a).run(r)
    })
}
```

---

## Directives 3 (eksisterer ikke)
MethodNotAllowed er fortsatt ikke håndtert 100% korrekt
Det skulle egentlig vært med en `Allow` header med støttede metoder

---

## Monader er for kraftige
```scala
trait Monad[F[_]]{
  def flatMap[A, B](a:F[A])(f:A => F[B]):F[B]
}
```

---

## Applicative Functor ?
```scala
trait Applicative[F[_]]{
  def apply2[A, B, C](a:F[A], b: => F[B])(f:(A, B) => C):F[C]
}
```
