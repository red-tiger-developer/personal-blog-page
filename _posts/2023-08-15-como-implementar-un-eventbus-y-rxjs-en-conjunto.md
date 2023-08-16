---
title: Arquitecturas Orientadas a Eventos. Implementando un EventBus y RxJS en Conjunto
author: Benjamin
date: 2023-08-16 08:32:00 -0400
categories: [Programacion, Typescript, Microservicios, Eventos ]
tags: [Programacion, Typescript, Arquitectura de software ]
---

![img](https://i.ibb.co/F3C2gcc/Screenshot-2023-08-16-at-10-14-58.png)


En el mundo de la arquitectura de software, especialmente en entornos orientados a microservicios, la comunicación y la coordinación entre componentes son cruciales. Una forma efectiva de lograrlo es mediante el uso de patrones de diseño basados en eventos. Uno de estos patrones es el uso de un EventBus, que actúa como un intermediario entre los distintos componentes, permitiendo que se comuniquen y se mantengan desacoplados.

En este artículo, exploraremos cómo implementar un EventBus utilizando RxJS, una librería de JavaScript que proporciona un conjunto poderoso de herramientas para trabajar con secuencias de eventos asincrónicos. Analizaremos un ejemplo de código que muestra cómo implementar y utilizar un EventBus en un contexto de arquitectura de microservicios.

## Parte 1: Contratos y Definiciones

En la primera parte del código, encontramos las definiciones de contratos y eventos de dominio. Estos contratos establecen la base para nuestro EventBus.

```typescript
import { Observable, Subject, filter, iif } from 'rxjs';
import { v4 as uuid } from 'uuid';

// Representa un evento de dominio con una carga de datos específica.
export abstract class DomainEvent<D, E extends string> {
    readonly datetime: Date = new Date();
    readonly id = uuid()
    abstract name: E
    constructor(public readonly data: D) { }
}

// Define el contrato para un despachador de eventos.
export interface EventDispatcher<E extends string> {
    dispatch<T extends DomainEvent<any, E>>(event: T): void;
}

// Define el contrato para un oyente de eventos.
export interface EventListener<E extends string> {
    onEvent<T extends DomainEvent<any, E>>(name?: E): Observable<T>
}

// Define el contrato para un bus de eventos que actúa como despachador y oyente.
export interface EventBus<E extends string> extends EventListener<E>, EventDispatcher<E> {

}

```

## Parte 2: Implementación de Eventos

En esta sección, se definen eventos específicos que heredan de DomainEvent, proporcionando detalles sobre datos y nombres de eventos. Nos centraremos en un ejemplo relacionado al retail.

```typescript

export type WarehouseEvents = "PRODUCT_CREATED" | "CATEGORY_CREATED" | "STOCK_UPDATED"

class ProductCreated extends DomainEvent<{ sku: string }, WarehouseEvents> {
    name: WarehouseEvents = "PRODUCT_CREATED"
}

class StockUpdated extends DomainEvent<{ sku: string, stock: number }, WarehouseEvents> {
    name: WarehouseEvents = "STOCK_UPDATED"
}

class CategoryCreated extends DomainEvent<string, WarehouseEvents> {
    name: WarehouseEvents = "CATEGORY_CREATED"
}

```
## Parte 3: Implementación del EventBus
Aquí es donde se implementa la lógica principal del EventBus utilizando RxJS.

```typescript
/**
 * Una implementación de EventBus utilizando RxJS.
 */
export class RxjsEventBus<E extends string> implements EventBus<E> {

    // Creación de un flujo de eventos usando un Subject
    private events$ = new Subject<DomainEvent<any, E>>()

    // Método para suscribirse a eventos específicos o todos los eventos
    onEvent<T extends DomainEvent<any, E>>(eventName?: E): Observable<T> {
        // Usando iif para filtrar eventos basado en el nombre del evento
        return iif(
            () => eventName !== undefined,
            this.events$.pipe(filter(event => eventName === event.name)),
            this.events$.asObservable()
        ) as Observable<T>;
    }

    // Método para despachar eventos al flujo
    dispatch<T extends DomainEvent<any, E>>(event: T): void {
        this.events$.next(event);
    }
}

```

 La implementación de `RxjsEventBus` utiliza un flujo de eventos `events$` implementado con `Subject` de RxJS para manejar la comunicación entre componentes. El método `onEvent` permite a los componentes suscribirse a eventos específicos o todos los eventos. Se utiliza el operador `iif` para filtrar eventos según el nombre del evento y devuelve un flujo de eventos observable. El método `dispatch` se utiliza para enviar eventos al flujo. En conjunto, esta implementación facilita la comunicación asincrónica entre componentes utilizando el patrón Observable/Observer de RxJS.


## Parte 4: Utilizando el EventBus

Finalmente, se crea una instancia del EventBus, se suscribe a los eventos que deseamos y se despachan algunos eventos de ejemplo.

```typescript
const eventbus: EventBus<WarehouseEvents> = new RxjsEventBus<WarehouseEvents>()

eventbus.onEvent()
    .subscribe({
        next: event => console.log("all-events", event),
        error: error => console.log(error)
    })

eventbus.onEvent("CATEGORY_CREATED")
    .subscribe({
        next: event => console.log("category-event", event.data),
        error: error => console.log(error)
    })

eventbus.dispatch(new ProductCreated({ sku: "9978363737" }))
eventbus.dispatch(new CategoryCreated("ELECTRONICS"))
eventbus.dispatch(new StockUpdated({ sku: "6453993", stock: 7 }))

```


## Importancia del Código en Arquitecturas Orientadas a Eventos
La implementación del EventBus presentada en este código tiene un impacto significativo en las arquitecturas de microservicios orientadas a eventos. Aquí hay algunas razones clave:

* `Desacoplamiento`: El EventBus permite que los componentes se comuniquen sin tener conocimiento directo entre sí. Cada microservicio puede despachar eventos y suscribirse a eventos de interés, lo que reduce el acoplamiento entre los servicios.

* `Escalabilidad`: En un entorno de microservicios, donde los componentes pueden aumentar o disminuir dinámicamente, el EventBus facilita la adición de nuevos servicios y la adaptación a cambios en la demanda.

* `Flexibilidad`: Los eventos encapsulan acciones significativas dentro del sistema. Si se requiere una acción adicional en respuesta a un evento, es posible agregar oyentes sin afectar otros componentes.

* `Rastreabilidad`: Los eventos registran acciones y cambios en el sistema. Esto permite la trazabilidad de acciones y la auditoría, lo que es esencial para aplicaciones críticas y sistemas complejos.

* `Mantenibilidad`: Cambiar o extender el comportamiento de un componente se vuelve más sencillo, ya que los cambios pueden realizarse en el contexto del evento correspondiente sin afectar otros componentes.

El EventBus en arquitecturas de microservicios es una pieza crucial para establecer comunicación efectiva y desacoplada entre componentes. Permite la construcción de sistemas altamente escalables, flexibles y mantenibles.

Con EventBus, podremos comunicar desde componentes internos de una aplicación hasta microservicios totalmente separados. Solo debemos definir las implementaciones específicas dependiendo del caso que tengamos.

## Conclusión

En este artículo, hemos explorado cómo implementar un EventBus utilizando RxJS en el contexto de arquitecturas de microservicios orientadas a eventos. Hemos desglosado el código paso a paso y hemos discutido la importancia de esta implementación en la construcción de sistemas flexibles y escalables.

Esperamos que esta inmersión en la implementación de EventBus te haya brindado una comprensión sólida de cómo puedes utilizar esta herramienta para mejorar la comunicación y la coordinación entre los componentes de tus aplicaciones orientadas a eventos. Te animamos a experimentar con esta implementación y a explorar más a fondo cómo puede beneficiar tu propia arquitectura de software.


## [Github repository](https://github.com/nullpointer-excelsior/advanced-design-patterns-with-typescript/tree/master/src/event-architecture)

Meme de cortesía

![meme](https://i.ibb.co/QM0dT9d/Screenshot-2023-07-28-at-16-13-08.png)



