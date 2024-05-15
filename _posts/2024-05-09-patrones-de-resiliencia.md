---
title: Patrones de resiliencia
author: Benjamin
date: 2024-05-09 00:00:00 -0500
categories: Arquitectura Software, Microservicios, Patrones, Programacion, Software
tags: arquitectura software, microservicios, patrones, programacion, software
---

![intro](assets/img/intros/intro6.webp)

## ¿Qué es la resiliencia?

La resiliencia es la capacidad de un sistema de poder recuperarse de un escenario adverso. En el desarrollo de software, podemos aplicar este concepto para crear aplicaciones robustas, las cuales sean capaces de manejar errores sin comprometer el sistema entero.

Los patrones de resiliencia son sencillos de entender e implementar con ayuda de la programación reactiva. A continuación, veremos los patrones más usados tanto en sistemas críticos como en procesos sencillos que, si bien pueden fallar y aislar el problema, un sistema resiliente hará todo lo posible para recuperarse antes de arrojar el error a una capa superior del sistema.

### Timeout

Este patrón define un tiempo límite de una petición. Esto nos ayuda a eliminar latencias grandes donde el sistema dará la impresión de estar pegado o cargando infinitamente. Generalmente, se usa en peticiones HTTP donde, cuando se cumpla un tiempo límite, nos indicará que este servicio está fallando.

### Retry

Retry nos permite reintentar una petición cuando esta falla. Generalmente, un reintento viene con un tiempo de espera antes de volver a realizar la petición y generalmente se define un número finito de reintentos para no llegar a crear un loop infinito de peticiones fallidas.

### Fallback

Este patrón nos permite definir una respuesta por defecto cuando el servicio o petición falla. Nos ayuda a no romper el sistema por arrojar alguna excepción o error. Se utiliza en peticiones que su respuesta no representa algo crítico en el proceso principal. Un ejemplo sería cuando nuestra aplicación de Spotify no encuentra las letras de alguna canción y este servicio responde simplemente con un texto de "letras no encontradas".

## Las aplicaciones robustas utilizan una combinación de estos patrones

Estos sencillos patrones son fáciles de implementar, pero la complejidad crece cuando queremos combinarlos entre sí, ya que este escenario planteado es más probable de encontrar en desarrollos serios.

### ¿Cómo podemos implementar la resiliencia de forma elegante?

Cuando hablamos de combinar o componer funcionalidades, podemos hacer uso de la programación funcional, donde cada patrón sería una función y, si queremos combinarlas, nos ayudamos de la composición de funciones o funciones de orden superior. En este caso, usaremos RxJS, una librería de JavaScript que nos permite trabajar con flujos de datos de manera reactiva e implementa en su corazon la programación funcional.

### Paradigma reactivo

El paradigma reactivo combina la programación funcional, la comunicación mediante eventos y el trato de los datos mediante un flujo. Una de las características de la programación reactiva que impulsa es la resiliencia, es por eso que aplicaremos los patrones antes vistos de manera sencilla con RxJS.

### Entendiendo los observables

Un observable es un componente que emite un valor y podemos suscribirnos para escuchar los cambios de este flujo. Bajo esta definición, podremos crear un observable que escuchara las peticiones y podremos aplicar resiliencia.

Ejemplo de observable:

```typescript
import { Observable } from 'rxjs';

// Creamos un observable que emite un array de números del 1 al 5
const observable$ = of([1, 2, 3, 4, 5]);

// Nos suscribimos al observable para escuchar los cambios en el flujo de datos
observable$.subscribe({
    // La función next se ejecuta cuando se emite un nuevo valor en el observable
    next: (value) => console.log(value),
    // La función error se ejecuta cuando ocurre un error en el observable
    error: (error) => console.error(error),
    // La función complete se ejecuta cuando el observable ha completado su emisión de valores
    complete: () => console.log('complete')
});

```

Ya explicada la base de Rxjs, podemos implementar los patrones de resiliencia de manera sencilla.
En el siguiente ejemplo, crearemos una función que agregará resiliencia a un Observable. La función tomará un Observable y devolverá una nueva función que tomará un objeto de opciones de resiliencia. Las opciones incluirán un tiempo de espera, un número de reintentos y un valor de respaldo opcional.

```typescript
import { Observable, catchError, iif, of, retry, throwError, timeout } from "rxjs";

type ResilenceOptions<T> = {
    timeout: number,
    retry: {
        count: number,
        delay: number
    },
    fallback?: T
}

export function addResilence<T = any>(source$: Observable<T>) {
    return function <T = any>(options: ResilenceOptions<T>) {
        const { timeout: timeoutMilliseconds, retry: retryOptions, fallback } = options;
        return source$.pipe(
            timeout({
                each: timeoutMilliseconds,
            }),
            retry({
                count: retryOptions.count,
                delay: retryOptions.delay
            }),
            catchError(err => iif(() => fallback !== undefined, of(fallback), throwError(() => err)))
        )
    }
}

```

Este código define una función `addResilence` que agrega resiliencia a un Observable. La función `addResilence` toma un Observable `source$` y devuelve una función que toma un objeto de opciones `ResilenceOptions`. Las opciones incluyen un tiempo de espera `timeout`, un objeto de reintento `retry` con un conteo y un retraso, y un valor de respaldo opcional `fallback`.

La función devuelta configura el Observable para que se agote después de un cierto tiempo de espera `timeout`, que se reintente un número específico de veces con un cierto retraso `retry`, y que, en caso de error, devuelva el valor de respaldo si se proporciona uno `fallback`, o lance el error si no se proporciona un valor de respaldo.

El operador `catchError` se utiliza para manejar los errores que pueden ocurrir durante la ejecución del Observable. Si se proporciona un valor de respaldo, se devuelve un Observable del valor de respaldo. Si no se proporciona un valor de respaldo, se lanza el error.


## Ejemplos concretos

Implementaremos los siguientes ejemplos:

- **Fallback sobre errores:** Simularemos un error controlado y manejaremos el error con un valor de respaldo
- **Timeout y reintentos:** Simularemos una operación con un tiempo de espera y reintentos

Definiremos las opciones de resilencia para todos los ejemplos:


```typescript
const options: ResilenceOptions<string> = {
    timeout: 1000,
    retry: {
        count: 3,
        delay: 3000
    },
    fallback: "Ops, something went wrong"
}
```

### Fallback para tratar errores

Crearemos la siguiente función que simula un error

```typescript
const failedRequest$ = throwError(() => {
    logger.error('throwing a controlled error 😨')
    return new Error('controlled error')
})
```
Ahora al agregar la resilencia al observable `failedRequest$`

```typescript
const fallbackRequest = addResilence(failedRequest$)
const fallbackObservable$ = fallbackRequest(options)

fallbackObservable$.subscribe({
    next: (value) => logger.info(`fallback-response: ${JSON.stringify(value, null, 2)}`),
    error: (error) => logger.error(`resilence-error: ${error.message}`, error),
})
```

Recibiremos la siguiente salida:

![ex1](assets/img/examples/resilence-ex1.webp)

### Timeout y reintentos

Para simular una operación con timeot, creamos el siguiente observable:

```typescript
let intent = 0
const calculateDelay = (intent: number) => intent === 3 ? 100 : 3000

const retriedRequest$ = from(Promise.resolve()).pipe(
    tap(() => intent++),
    tap(() => {
        if (intent > 0) logger.info(`executing retry number: ${intent}`)
    }),
    switchMap(() => of("Retry example 🧐").pipe(delay(calculateDelay(intent))))
);
```

Y al observable `retriedRequest$` le agregamos resilencia

```typescript
const retryRequest = addResilence(retriedRequest$)
const retryObservable$ = retryRequest(options)

retryObservable$.subscribe({
    next: (value) => logger.info(`retry-response: ${JSON.stringify(value, null, 2)}`),
    error: (error) => logger.error(`resilence-error: ${error.message}`, error),
})

```
Obtendremos la siguiente salida:

![ex2](assets/img/examples/resilence-ex2.webp)

Y si queremos simular una operacion sin errores lo hacemos así:

```typescript

const successRequest$ = of('success request 😎')
const successRequest = addResilence(successRequest$)
const successObservable$ = successRequest(options)

successObservable$.subscribe({
    next: (value) => logger.info(`success-response: ${JSON.stringify(value, null, 2)}`),
    error: (error) => logger.error(`resilence-error: ${error.message}`, error),
})
```

![ex3](assets/img/examples/resilence-ex3.webp)

## Conclusiones

Los patrones de resiliencia nos permiten crear aplicaciones robustas y preparadas para actuar ante cualquier error o intermitencia que algun servicio u operacion presente en el flujo de una aplicacion. RXjs nos provee una manera adecuada de tratar flujos complejos con la posibilidad de extender las capacidades actuales de un codigo basado en el paradigma reactivo


## [Github Repository](https://github.com/nullpointer-excelsior/advanced-design-patterns-with-typescript/tree/master/src/resilience)

### Meme de regalo

![meme](assets/img/memes/meme4.webp)

