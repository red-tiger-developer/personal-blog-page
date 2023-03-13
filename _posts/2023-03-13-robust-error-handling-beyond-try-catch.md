---
title: Robusto control de errores más allá del Try Catch
author: Benjamin
date: 2023-03-13 10:32:00 -0500
categories: [Programacion, Arquitectura de software, Typescript]
tags: [Typescript, Javascript, Patterns Designs]
---

![img](https://i.ibb.co/yFqbm4x/Screenshot-2023-03-13-at-15-33-52.png)

El control de errores en aplicaciones tradicionalmente los manejamos con `trycatch` si bien esto nos proporciona una
manera efectiva y simple de controlar errores y definir lógicas un poco más elaboradas de cara al cliente, existe otra
alternativa proveniente de la programación funcional.

## Either Monad

La `Either Monad` es una estructura de datos en programación funcional que se utiliza para manejar valores que pueden
tener dos posibles estados: "éxito" o "falla". Básicamente creamos una respuesta donde puede ser uno de estos 2 valores:

* `Right value`: lo que queremos retornar cuando nuestro código realiza una operación exitosa.
* `Left value`: retornamos un objeto que representa un error.

Este simple enfoque nos puede proveer una manera de control de errores mucho más robusta.


## Try catch vs Either

Dependiendo del escenario `trycatch` puede dejar de ser una manera efectiva de controlar errores, ya que una excepción
se refiere a algo excepcional que ha ocurrido en el sistema y este debe interrumpirse o tratar de recuperarse.
Una excepción es adecuada para los siguientes casos:

* Problemas de red o conexión
* Errores en librerías de bajo nivel
* Errores correspondiente al ambiente o sistema operativo

En cambio `Either` es ideal para un control de errores más específicos relacionados con lógicas de dominio, ya que nos obliga
a definir la respuesta correcta a ciertos errores, si bien la implementación de Either por si sola nos da la posibilidad
entre una respuesta exitosa y una fallida, Este enfoque a menudo se implementa junto a `pattern matching` una estructura encontrada
muy a menudo en la programación funcional. Esta combinación nos permite tener el control total de un flujo relacionado con la lógica principal
del programa, ya que estaremos obligados a implementar todas las posibles respuestas fallidas incluyendo el caso exitoso
esto separa totalmente las excepciones de los errores de lógicas. En typescript no disponemos de pattern matching,
pero podemos hacer algo interesante con el tipado.

## Implementando un control de errores avanzados con typescript

La implementación básica de Either está dada por el siguiente código:

```typescript
export type Result<T, E> = Success<T, E> | Failure<T, E>;

export class Success<T, E> {

    readonly success: T;

    constructor(success: T) {
        this.success = success;
    }

    isSuccess(): this is Success<T, E> {
        return true;
    }

    isError(): this is Failure<T, E> {
        return false;
    }

}

export class Failure<T, E> {

    readonly error: E;

    constructor(error: E) {
        this.error = error;
    }

    isSuccess(): this is Success<T, E> {
        return false;
    }

    isError(): this is Failure<T, E> {
        return true;
    }

}

```

El funcionamiento de estos componentes es el siguiente:

* `Success`: Representa una respuesta exitosa y contendrá un valor para ser tratado.
* `Failure`: Representa un error específico este también contiene un valor el cual puede ser el detalle del error ocurrido en la lógica
* `Result`: este objeto representa una respuesta exitosa o fallida,

Si `Result` es exitoso puede devolver el valor de `Success` pero no puede devolver el error definido del objeto
`Failure`, en cambio si `Result` es de tipo `Failure` podremos obtener el valor de `Failure` pero no el valor de `Success`.
Para consultar si el objeto `Result` es exitoso lo haremos por medio del método `isError()` o `isSuccessful()` a su vez al
invocar alguno de estos 2 métodos typescript automáticamente hará un casting de `Result` a `Success` o `Failure`
dependiendo de si la operación fue existosa o errónea

```typescript
// example 1
result = failureOperation()
result.error // typing error: error no existe mientras no se llame a isError()
if(result.IsError()) {
  result.error // casting automatico
}

//example 2
result = succesOperation()
result.success // typing error: success no existe mientras no se llame a !isError()
result.error // typing error: error no existe mientras no se llame a isError()
if(!result.IsError()) {
  result.success // casting automatico
}

```
El uso de `Either` es bastante sencillo y poderoso ahora implementaremos un control de errores avanzado con esta clase de ayuda
```typescript
export class ErrorHandler<T extends string> {

    constructor(private error: T, private message: string = 'No provided error message') {}

    match(handler: Record<T, (message: string) => void>) {
        if (handler.hasOwnProperty(this.error)) {
            handler[this.error](this.message); // si hay una coincidencia ejecutará el método callback
        }
    }

}
```
Esta clase recibe un tipo generico que representará los errores que pueden ocurrir, el método match se encarga de invocar
una función callback asociada al error. Para entender mejor este código crearemos nuestro `ErrorHandler` basados en una API de productos

```typescript

// definimos nuetsrso custom errors
export type ProductErrors = 'unavailableStock' | 'serverError' | 'otherError'

// heredamos una clase de ErrorHandler que representará nuestros errores
export class ProductErrorHandler extends ErrorHandler<ProductErrors> {

}

```
El siguiente ejemplo nos permite entender el uso del patron `Either`, el método `udpateStock()` dependiendo del caso
devolverá una respuesta exitosa o un error específico el cual puede tratarse de una manera más personalizada.
```typescript
export class ProductService {

    constructor(
        private productRepository: ProductRepository,
        private stockRepository: StockService
    ) {}

    updateStock(id: string, stock: number): Result<any, UpdateStockErrorHandler> {

        const product = this.productRepository.findById(new ProductID(id))

        if (!product) {
            return new Failure(new UpdateStockErrorHandler('productNotFound', `Product(id=${id}) Not found`))
        }

        const stockAvailable = this.stockRepository.queryStock(product.ID)

        if (stockAvailable < 0) {
            return new Failure(new UpdateStockErrorHandler('unavailableStock', `Product(id=${id}) stock unavailable!!!`))
        }

        if (stockAvailable < 40) {
            return new Failure(new UpdateStockErrorHandler('insuffisientStock', `Product(id=${id}) stock cannot be updated`))
        }

        this.stockRepository.updateStock(product.ID, stock)

        return new Success({
            productId: product.ID,
            newStock: stock
        })

    }

}
```
Y ahora cuando invocamos el servicio de productos Obtendremos nuestro objeto Result (patron Either)

```typescript

const productId = 'ab351bc97d'
const result = productService.updateStock(productId, 20)

if (result.isSuccess()) {
    result.success
} else {
    result.error.match({
        insuffisientStock:(msg: string) => {
            alertService.sendAlert(msg)
        },
        productNotFound: (msg: string) => {
            // some actions...
        },
        unavailableStock: (msg: string) => {
            // some actions...
        }
    })
}

```
Al invocar el método `match()` de Error nos obligará a implementar métodos callbacks basados en el type error definido previamente.

```typescript
// my custom errors
export type ProductErrors = 'unavailableStock' | 'serverError' | 'otherError'
```

## Conclusiones

Implementamos Either en typescript para un control de errores más robusto para la lógica principal de la aplicación.
`Either` Puede ser interesante para abordar casos bordes de una manera sencilla con la posibilidad de cubrir de manera
obligatoria los errores referentes al negocio, mientras que `trycatch` nos puede tratar las excepciones como eventos
o errores externos a la lógica de negocio principal.

## Meme de cortesía
![meme](https://i.ibb.co/rsm5f96/Screenshot-2023-03-13-at-15-27-29.png)
