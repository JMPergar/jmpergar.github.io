---
layout: post
section-type: post
title: Lleva tus Use Cases al siguiente nivel con Monads y Kotlin
headerimage: /img/posts/ayutthaya.jpg
category: tech
tags: [ 'tutorial', 'kotlin', 'programación funcional', 'monads', 'clean architecture' ]
---

El tiempo pasa volando y ya llevo mas de tres años jugando con esto que llaman [Clean Architecture](https://www.youtube.com/watch?v=x3CR39_PR_I), empece con [Hexagonal](https://www.youtube.com/watch?v=C3e3AwOTohg) y aquí sigo, si lo contara en numero de frameworks de javascript ya habría perdido la cuenta. Tres años en nuestro sector puede llegar a ser una eternidad.

En todo este tiempo he cometido muchos errores y aprendido cosas, como sufrir **sobredosis de modelos**, pasando por tener **presentadores que se creían use cases** o **repositorios que delegaban todo su trabajo**. Pero tras todo este tiempo, el punto al que seguía dándole vueltas y no terminaba de tener resuelto era la **gestión de errores y casos excepcionales** así como la **coordinación de trabajos paralelos**.

No es que no existan soluciones, que si existen, si no que todas me parecían engorrosas (Callbacks o Excepciones) o te acoplaban a librerías que o bien no seguían ningún estándar o bien se metían hasta la cocina en tu proyecto ([JDeferred](https://github.com/jdeferred/jdeferred) o [RxJava](https://github.com/ReactiveX/RxJava)).

<br><br>
> *Podéis buscar información acerca del **Callback Hell**, de la polémica **Checked Excepctions vs Unchecked**, leer un poco de como son las **Futuros/Promesas** en [Scala](https://www.scala-lang.org/) y analizar proyectos hechos con **RxJava** y ver hasta donde extiende sus tentáculos esta librería.*

> *Esta es mi opinión, informaros y construir la vuestra propia. Sed críticos y cuestionaros vuestras soluciones o las que otros os ofrezcan, es la única manera de progresar y probablemente el mejor consejo que os puedo dar en este post.*

<br><br>

Y es aquí donde entra en juego la **Programación Funcional**.

Hace poco me he puesto manos a la obra para preparar un MVP (Minimum Value Product) y como soy así y no tenía suficiente con ese reto me propuse hacerlo en [Kotlin](https://kotlinlang.org/), pero fue esta decisión la que me está permitiendo afrontar los problemas desde otra perspectiva y encontrar nuevas soluciones. Llevo unos años coqueteando con Scala y esto ya me daba alguna idea de lo que podría llegar a ser pero fue la [charla](https://www.youtube.com/watch?v=cnOA7HdNUR4) de [Raúl Raja](https://twitter.com/raulraja) en el pasado [Freakend Mobile](https://www.autentia.com/2017/02/23/un-fin-de-semana-en-la-sierra-hablando-de-mobile-charlas-del-freakend/) la que abrió mi mente a las nuevas opciones que vengo a contaros.

Es cierto que Kotlin [esta lejos de ser tan maduro como Scala](https://kotlinlang.org/docs/reference/comparison-to-scala.html) y la libreria estandar de este segundo es mucho mas completa e incluye cosas que en nuestro caso tendremos que implementar o apoyarnos en librerías de terceros para tenerlas, pero el punto importante es que Kotlin nos ofrece la base necesaría para desarrollar estas soluciones.

<br><br>
> *En este post no voy entrar a explicaros las bases de Kotlin ni los elementos de este lenguaje que vamos a usar. Si como yo sois programadores de Android os recomiendo leeros ["Kotlin for Android Developers: Learn Kotlin the easy way while developing an Android App"](https://www.amazon.es/Kotlin-Android-Developers-Learn-developing/dp/1530075610/ref=sr_1_1?s=books&ie=UTF8&qid=1488458124&sr=8-1&keywords=antonio+leiva+kotlin), un fantástico libro de introducción al desarrollo en Android con Kotlin que os proporcionara esa base necesaria. También podéis ojear el [blog](https://devexperto.com/blog/) de su autor y amigo Antonio Leiva que está plagado de posts al respecto*

> *Tampoco voy a profundizar en el concepto de Monad o sus distintos tipos, tan solo en como estos nos ayudan. Para lo primero os recomiendo una charla de **Juan Manuel Serrano** sobre [arquitecturas funcionales](https://www.youtube.com/watch?v=CT58M6CH0m4) que tuve el placer de disfrutar en la pasada Codemotion 2016 y recomiendo muy mucho. Para lo segundo podéis ver la charla de Raúl que enlacé más atrás o leeros los puntos 5, 6 y 7 de [esta](http://danielwestheide.com/scala/neophytes.html) guía de Scala.*

<br><br>

Os comentaba que era la gestión de errores y los coordinación de trabajos los puntos que me preocupaban, este post solo abarcará el primero de los problemas aunque bien es cierto que la senda que aquí empezamos es la misma que nos hará encontrar solución a lo segundo. En cuanto lo tenga más maduro publicare otro post.

<br><br><br>

## ¿Cual es el problema?

Imaginaos que necesitamos realizar dos llamadas y una vez finalizadas vamos a realizar un procesado conjunto de las respuestas de ambas llamadas. Hasta ahí fácil ¿verdad?, ¿pero qué sucede si se da algún caso excepcional y necesitamos realizar alternativas en función de estos casos?. Aquí la cosa ya se vuelve un poco más compleja.

Podríamos considerar dos implementaciones, asíncrona con callbacks o síncrona con excepciones para nuestros casos de uso:

<br>
```java
// Implementacións Asíncrona
myUseCase.execute(
  new MyUseCase.Callback() {
    @Override
    public void onSuccess(String text) {
      // do something
    }

    @Override
    public void onNetworkError() {
      // do something
    }

    @Override
    public void onServerError() {
      // do something
    }
});

// Implementación Síncrona
try {
  String result = myUseCase.execute();
  // do something
} catch (NetworkException e) {
  // do something
} catch (ServerException e) {
  // do something
}
```
<br><br>

Ambas implementaciones nos llevarían al mismo problema. Tendríamos que andar preguntando si ha ido bien o mal y en el segundo caso preguntar qué ha ido mal, esto por cada una de las llamadas. No tendríamos una manera clara y legible de escribir nuestro happy case, ¿qué es lo que queremos hacer cuando todo va bien?. A todo el código de las llamadas planteadas en cualquiera de los dos estilos anteriores habría que añadir el de esas preguntas que os comentaba. Os expongo el caso más sencillo, el síncrono:

<br>
```java
if (resultOfFirstCall != null && resultOfSecondCall != null) {
  // Process results
} else {
  // Check exceptions and process
}
```

*En el caso asíncrono sería aun más complejo por que habría que usar algún mecanismo para esperar que ambas llamadas respondan o lancen la excepción (esto es tema de próximos posts).*
<br><br>

Evidentemente este código es muy simple con el fin de servir de ejemplo, pero imaginad cómo se podría complicar si el número de llamadas sube o si también lo hace el número de casos a contemplar. Imaginaos cuánto código tendríais que leer para averiguar qué hace vuestro programa. Imaginad que queréis hacer cosas más complejas como procesar esos dos resultados y hacer una tercera llamada con él y concatenar este tercer resultado con un cuarto. Imaginad la cantidad de comprobaciones intermedias que tendréis que hacer y como poco a poco la verdadera finalidad de vuestro programa queda cada vez más oculta por el cómo por encima del qué.

Pues bien, ¿que me decís si os digo que hay una manera distinta de hacerlo que prima el qué por encima del cómo?. Os lo muestro en pseucódigo:

<br>
```
var1 = firstCall()
var2 = secondCall()

var3 = var1 + var2

var4 = otherCall(var3)

inCaseOfSuccess(var4) {
  // do something var4
} else {
  // do something with the collections of errors
}
```
<br><br>

¿Bonito verdad?

Como veis, ahora de un simple vistazo vemos lo que nuestro programa hace y sólo al final gestionamos las excepciones.
<br><br><br>

## ¿Pero como lo hacemos?

Aquí es donde entran en juego las **Monads**, en concreto el tipo `Either`, que es un tipo cuyo fin es encapsular la respuesta y todos sus casos excepcionales. Os pongo un ejemplo:

<br>
```kotlin
val myEither: Either<ExceptionsCase, String>

sealed class ExceptionsCase {

    class NetworkError : ExceptionsCase()
    class ServerError : ExceptionsCase()
}
```
<br><br>

Lo que aquí tendríamos es un tipo de respuesta que podría contener nuestro resultado esperado o el caso excepcional a contemplar (definido mediante una `sealed class` de kotlin).

Y tu me dirás: ¡pero si estamos en las mismas!, debemos preguntar por él contenido para poder trabajar con el. Y yo te diré: no, es aquí dónde, gracias al concepto de monad, podemos trabajar con estos resultados bajo el supuesto de que todo ha ido bien.

La idea es que mediante funciones como `map` o `flatMap` procesemos el resultado en caso de existir y en caso contrario devolver un `Either` que albergue el caso exceptional, esto nos permitiría ir concatenando eithers y solo al final preocuparnos de si ha dio bien o mal. Os pongo un ejemplo:

<br>
```kotlin
val result1: Either<ExceptionsCase, String> = firstCall()
val result2: Either<ExceptionsCase, String> = secondCall()

result1.flatMap {
  res1 -> result2.map {
    res2 -> // Do something with res1 y res2
  }
}
```
*La declaración de tipos en kotlin no sería necesaría y sería inferida pero la añado por claridad*
<br><br>

En este caso `res1` y `res2` serían los strings de respuesta. Pero no, esto aún no es todo lo limpio que nos gustaría y se podría complicar mucho si anidamos varios procesamientos de eithers. Lo bueno es que los lenguajes ya nos proporcionan azúcar sintáctico para lidiar con estos casos. Como las `for comprehension` de Scala que dejarían nuestro código como algo similar a lo siguiente:

<br>
```scala
for {
    res1 <- result1
    res2 <- result2
} yield (res1 + res2)
```
<br><br>

Pero he hecho un poco de trampa, por que como os comentaba al principio Kotlin no está aún al nivel de lenguaje como Scala en estos aspectos y no incluye ni `Either` ni nada similar a las `for comprehension`, pero también os comentaba que nos proporciona las base para implementar este tipo de soluciones.

En nuestro caso nos vamos a apoyar en [Funcktionale](https://github.com/MarioAriasC/funKTionale), en concreto en sus paquetes `either` y `validation`.

El primero de estos nos provee un nuevo tipo llamado `Disjunction` que vendría a ser el `Either` que os he venido presentando. Así pues ahora nuestros use case podrían implementar una interfaz similar a la siguiente:

<br>
```kotlin
interface UseCase<in I, out R, out E> {

    fun execute(input: I): Disjunction<E, R>
}
```
<br><br>

Donde `I` es el input de nuestros use case, `E` el conjunto de casos excepcionales que, como vimos más atrás, definiremos mediante una `sealed class` y `R` el tipo de retorno esperado.

Nuestro código en Kotlin, ahora ya real, quedaría tal que así:

<br>
```kotlin
val closeEvents = getCloseEventsUseCase.execute(location)
val collections = getCollectionsUseCase.execute(location)

val result = events.flatMap {
  events -> collections.map {
    collections ->
      listOf(FeedElement.Collections(collections)) + events.map { FeedElement.Event(it) }
  }
}

when (result) {
  is Disjunction.Left -> manageExceptions(result.swap().get())
  is Disjunction.Right -> result.map { view?.renderFeed(it) }
}
```
*Os pongo un caso más real para que vaya cogiendo color.*
<br><br>

En el código anterior podéis comprobar como es sólo al final cuando vamos a aplicar los **side effects** cuando contemplamos y procesamos los casos excepcionales.

Mucho más limpio, ¿no?. Quedaría mucho más limpio si encapsulara parte del código en funciones puras pero he preferido exponerlo así para mostrar mejor la diferencia entre la solución inicial y final.

Aquí tenemos otra vez esta pieza de código y nosotros sin `for comprehension`:

<br>
```kotlin
val result = events.flatMap {
  events -> collections.map {
    collections ->
      listOf(FeedElement.Collections(collections)) + events.map { FeedElement.Event(it) }
  }
}
```
<br><br>

Aquí es donde viene al rescate el segundo paquete `validation` que nos proporciona una manera más limpia y sin anidaciones de procesar varios eithers y conseguir el mismo resultado:

<br>
```kotlin
validate(events, collections) {
  events, collections ->
    listOf(FeedElement.Collections(collections)) + events.map { FeedElement.Event(it) }
}
```
<br><br>

Gracias a la función `validate` podemos procesar ese happy case y en caso contrario nos devolverá un `Disjunction` con el conjunto de los casos excepcionales.
<br><br>

Y hasta aquí todo, espero que al menos este post os muestre que las cosas se pueden hacer de otra manera distinta a como la venimos haciendo los que como yo llevamos años dedicándonos a la programación orientada a objetos y os anime a buscar nuevas soluciones, yo por mi por mi parte seguiré profundizando en el tema y compartiendo mis avances. Para ello he púlicado un [repositorio](https://github.com/JMPergar/klean) en donde iré mostrando el modo en que implemento mis apps usando estos nuevos patrones y arquitecturas. Además es un proyecto librería así que lo podréis usar si os animáis a hacer alguna prueba.
<br><br>

Future&#60;Option&#60;HastaPronto&#62;&#62;
<br><br>
