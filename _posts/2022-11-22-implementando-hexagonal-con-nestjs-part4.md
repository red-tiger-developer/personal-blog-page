---
title: Arquitectura hexagonal Parte IV Patrones de arquitectura sobre la capa Application
author: Benjamin
date: 2022-11-07 10:32:00 -0500
categories: [Programacion, Nestjs, Arquitectura de software, Typescript]
tags: [typescript, nestjs, hexagonal]
---

![image](https://i.ibb.co/G9rmWpC/Screen-Shot-2022-12-04-at-23-48-50.png)


En nuestro post anterior modelamos nuestro dominio y nuestras reglas de negocio con Domain Driven Design implementamos los conceptos y patrones más usados en este enfoque de desarrollo. En esta ocasión nos enfocaremos en nuestra capa de aplicación en la cual definimos nuestros casos de uso. En este post definiremos patrones de arquitectura que nos permitirán crear un código mantenible y escalable.
## ¿Qué componentes se encuentran en nuestra Capa Application?

Los componentes que podemos encontrar en esta capa son los siguientes:
* `Application Services`: Son nuestros casos de uso o los llamados `features` de nuestra aplicación.
* `Ports`: componentes que deben interactuar con servicios externos o componentes de carácter más técnicos que no se relacionan con el dominio, por ejemplo un componente que necesite enviar emails
* Event Subscribers: Componentes que se suscriben a eventos y deben ejecutar algún caso de uso o `feature`

La principal diferencia entre los servicios de aplicación y servicios de dominio es que los servicios de dominio contienen lógicas y reglas de negocio asociadas a las entidades, mientras que los servicios de aplicación hacen uso de los componentes de dominio coordinando el uso de los servicios de dominio u los otros componentes existentes para cumplir con el objetivo del caso de uso o `feature` asociado.

## Estructura básica de la capa Application

La estructura de la capa application:

```bash
 core
├──  application
│   ├──  events
│   │   └──  subscribers
│   │       └──  StockUpdaterSubscriber.ts
│   ├──  ports
│   │   └──  EmailSender.ts
│   └──  services
│       ├──  CatalogUseCases.ts
│       └──  PurchaseUseCases.ts
└──  domain
    ├──  entities
    ├──  repositories
    └──  services
```
Nuestros casos de uso:
 ```typescript
export class PurchaseUseCases {

    constructor(private order: OrderService) { }

    async createOrder(createorder: CreateOrderDto): Promise<OrderCreatedDto>  {

        return this.order.create(createorder) // creating an order instance
            .then(order => this.order.save(order)) // save to database
            .then(order => order.getSummary()) // return summary

    }

    async getOrders(getorder: GetOrdersRequest) {
        
        const offset = getorder.page - 1 // define offset for query
        const orders = await this.order.getOrdersSlice(getorder.size, offset) // get orders slice
        const totalRecords = await this.order.getOrdersCount() // data count
        
        // creating a paginated
        return Paginated.create({
            ...getorder,
            count: totalRecords,
            data: orders,
        })

    }

}

export class CatalogUseCase {
  
    constructor(private product: ProductService) { }

    async getProductsByFilter(filter: filtersRequest) {
        
        const offset = filter.page - 1 // define offset for query
        const products = await this.products.getProductsByFilter(filter) // get products
        const totalRecords = await this.order.countProductByFilter(filter) // data count
        
        // creating a paginated
        return Paginated.create({
            ...filter,
            count: totalRecords,
            data: products,
        })

    }
}
 ```
En este enfoque nos enfocamos en definir los casos de uso, pero en algunas implementaciones de arquitectura hexagonal se definen los controladores dentro de la capa de aplicación es un enfoque válido ya que muchas veces nuestra aplicación es solo un API CRUD donde los casos de uso y la lógica de dominio no son tan complejas. Estas implementación generalmente utilizan librerías de terceros o frameworks en sus controladores y hacen llamado de los servicios de aplicación o incluso solo de los servicios de aplicación.

```bash
 core
├──  application
│   ├──  controllers
│   │   ├──  catalog.controller.ts
│   │   └──  purchase.controller.ts
│   └──  services
│       ├──  CatalogUseCases.ts
│       └──  PurchaseUseCases.ts
└──  domain
    ├──  entities
    ├──  repositories
    └──  services
```

## Exponiendo los servicios de aplicación

Para hacer uso de nuestros casos de uso solo debemos inyectar nuestros servicios de aplicación en componentes de tipo presentacional como lo puede ser un Controlador Rest y hacer uso de los métodos expuestos por nuestros casos de uso. 

```typescript
// using Get products use case 
const products = await = this.catalog.getProductByFilter(filter)
```
En casos simples, este enfoque puede ser suficiente como una API CRUD. Pero cuando nuestra aplicación contiene lógicas de dominio complejas, las que pueden evolucionar en el tiempo y no queremos que estos cambios rompan las interacciones entre clientes y servicios y a su vez queremos escalabilidad Necesitamos otro enfoque.

## Usando CQRS dentro de nuestra capa application

Dentro del diseño de nuestra capa de dominio y aplicación podemos implementar CQRS
el cual significa segregación de responsabilidades de comandos y consultas, Un elegante significado en el mundo de las arquitecturas limpias. Básicamente, CQRS es un patrón de diseño el cual separa las operaciones de lectura y escritura sobre un almacén de datos. La implementación de CQRS en una aplicación puede maximizar el rendimiento y la escalabilidad. La flexibilidad que nos entrega este enfoque nos permite que nuestro sistema evolucione mejor con el tiempo y evita que procesos de actualización y lectura provoquen conflictos de combinación en nuestra capa de dominio.

En resumen, CQRS No solo mejora la arquitectura de las operaciones sobre nuestro dominio, sino que nos provee una manera efectiva para **mejorar el rendimiento y escalabilidad de nuestra aplicación**. 

## ¿Por qué o cuando debería implementar CQRS?

CQRS nos da una interfaz común para que nuestros componentes de tipo cliente o ui puedan comunicarse a través de comandos y queries. CQRS también se utiliza en aplicaciones de alto rendimiento. Al separar las consultas de los comandos, podemos separar nuestras fuentes de datos en una de lectura y otra de escritura, con lo que podríamos escoger un motor de base datos adaptado a nuestras necesidades de dominio y no al revés, así evitamos adaptar nuestro código a estrategias de performance que de una u otra manera pueden contaminar nuestro dominio. También Fomenta el uso del asincronismo en acciones más lentas, podemos emplear comandos y definir quien atiende a esa petición sin bloquear el proceso actual.

## CQRS y arquitecturas orientadas a eventos

Este enfoque se complementa perfecto con patrones de arquitectura orientados a eventos donde definimos subscriptores que estarán a la escucha sobre eventos específicos pueden ser tanto eventos de dominio como eventos de integración cuando un evento se produzca nuestros subscriptores ejecutaran lógicas de negocio a través de los servicios de aplicación entonces teniendo estos 3 componentes:

* `Comandos`: acciones que causan efectos secundarios sobre el dominio (escritura)
* `Queries`: acciones que devuelven datos relacionados al dominio (lectura)
* `Eventos`: eventos relacionados con sucesos ocurridos dentro nuestro dominio o eventos enviados por otras aplicaciones.

Podemos definir una arquitectura para exponer nuestros casos de uso mediante una interfaz común, esta sería nuestra API de la copa core (application y domain)

## CQRS con Nestjs 

En esta ocasión implementaremos CQRS con Nestjs y su módulo `cqrs` este módulo es muy comodo, ya que nos provee una forma limpia de implementar comando, queries e incluso eventos mediante anotaciones e inyectando los componentes `CommandBus`, `QueryBus` y `EventBus`.

> CQRS por debajo implementa los siguientes patrones de diseño: Mediator y Command

Instalamos nuestro módulo:
```bash
npm install --save @nestjs/cqrs

```
Para hacer uso de comandos en cqrs debemos crear una clase que represente los datos de entrada de nuestro comando. Al definir un comando debemos también definir un `handler` que no es nada más que una clase con la anotación `@CommandHandler` finalmente debemos agregarlo como `provider` dentro de nuestro módulo `core`.

```typescript
// command to create a order
export class CreateOrderCommand {
    constructor(public readonly order: CreateOrderDto) { }
}
// handler for create order command
@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {

    constructor(private purchase: PurchaseUseCases) { }

    async execute(command: CreateOrderCommand) {
        this.purchase.createOrder(command.order)
    }

}
```
Podemos implementar múltiples handlers para nuestro comando en algunos escenarios, puede ser útil como un log de los datos como de auditoria.

Ahora implementaremos nuestras cqrs queries y sus handler:

```typescript
// query for orders
export class OrdersQuery {

    constructor(public readonly page: number, public readonly  size: number){}

}
// handler
@QueryHandler(OrdersQuery)
export class OrdersQueryHandler implements IQueryHandler<OrdersQuery>{

    constructor(private purchase: PurchaseUseCases) { }

    execute(query: OrdersQuery): Promise<Paginated<Order>> {
        return this.purchase.getOrders(query)
    }

}
```

Ahora para poder realizar nuestros comandos y queries debemos inyectar en nuestro `PurchaseController` los componentes `QueryBus` y `CommandBus`.

```typescript
@Controller('/purchase')
export class PurchaseController {

    constructor(
        private command: CommandBus,
        private query: QueryBus
    ) {}
  
}
```
Ahora desde nuestros endpoints hacemos uso de nuestros `CommandBus` y `QueryBus`

```typescript

    @Post('/order')
    async create(@Body() order: CreateOrderRequest): Promise<OrderCreatedDto> {
        return await this.command.execute(new CreateOrderCommand({
            ...order
        }))
    }

    @Get('/order')
    async getOrders(@Query('page')page: number, @Query('size') size: number) {
        return this.query.execute(new OrdersQuery(page, size))
    }

}
```
CQRS dentro de nuestra aplicación está lista. Ahora configuraremos la escucha de eventos. Nuestra aplicación ya posee una implementación de EventBus y subscripción de eventos, pero utilizaremos las herramientas que nos provee nestjs para mostrar la utilidad del módulo `cqrs`

Implementamos un nuevo servicio de aplicación para contener los casos de usos asociados al contexto de Stock.

```typescript
@Injectable()
export class StockUseCase {

    constructor(private product: ProductService) {}

    async updateStockProducts(order: Order) {
        
        for (let detail of order.details) {
            await this.product.updateProductStock(detail.product.productId, detail.quantity)
        }
        
    }

}
```

Implementación de eventos, reutilizaremos nuestro evento de dominio ya definido anteriormente `OrderCreated` y haremos una reimplementación de nuestro puerto `DomainEventBus` utilizando el servicio `EventBus` del módulo `@nest/cqrs`


```typescript
// port 
export interface EventBusPublisher {
    publish(event: EventBase): void 
}
// adapter
@Injectable()
export class EventBusPublisherDomain implements EventBusPublisher {
    
    constructor(private eventbus: EventBus){}

    async publish(event: EventBase): Promise<void> {
        await this.eventbus.publish(event)
    }

}

```
En esta ocasión vemos el poder de los puertos y adaptadores, nos permite una migración de tecnologías sin mayores contratiempos, solo debemos modificar nuestro módulo core para realizar las inyecciones correspondientes. Finalmente implementamos nuestro `EventHandler`.

```typescript

// handler
@EventsHandler(OrderCreatedEvent)
export class OrderCreatedHandler implements IEventHandler<OrderCreatedEvent> {

    constructor(private stock: StockUseCase) { }

    async handle(event: OrderCreatedEvent) {
        const order = event.orderCreated.getData()
        await this.stock.updateStockProducts(order)
    }
    
}
```
Publicación de evento `OrderCreated`:

```typescript
export class OrderService {
  
  constructor(
        private readonly order: OrderRepository,
        private readonly eventbus: EventBusPublisher
    ) { }

    async save(order: Order): Promise<Order> {
        return this.order
            .save(order)
            .then(orderId => {
                order.orderId = orderId
                return order
            })
            .then(order => {
                this.eventbus.publish(new OrderCreated(order))// publish domain event
                return order
            })
    }

  // ...hidenn code
}
```

## Estructura final de la aplicación:

Nuestra aplicación queda de la siguiente manera agregamos el directorio `entrypoint` dentro de nuestra capa `application` la cual define la manera de interactuar con nuestros casos de uso
este enfoque nos da claridad de las acciones de lectura, escritura y escucha de eventos.

```bash
 core
├──  application
│   ├──  entrypoint
│   │   ├──  commands
│   │   │   ├──  CreateOrderCommand.ts
│   │   │   └──  handlers
│   │   │       └──  CreateOrderHandler.ts
│   │   ├──  events
│   │   │   └──  handlers
│   │   │       └──  OrderCreatedHandler.ts
│   │   └──  queries
│   │       ├──  handlers
│   │       │   └──  OrdersQueryHandler.ts
│   │       └──  OrdersQuery.ts
│   ├──  services
│   │   ├──  CatalogUseCases.ts
│   │   ├──  CompanySuppliersUseCases.ts
│   │   ├──  CompanyUseCases.ts
│   │   ├──  CustomerPortfolioUseCases.ts
│   │   ├──  PurchaseUseCases.ts
│   │   └──  StockUseCases.ts
│   └──  utils
│       └──  Paginated.ts
├──  core.module.ts
```

## Conclusiones

Básicamente, CQRS nos permite separar las escrituras y lecturas de nuestro dominio para poder crear sistemas escalables. Este post no terminará aca, ya que no hemos visto los beneficios de escalabilidad que nos entrega CQRS y las arquitecturas orientadas a eventos. Pero esos será en unos próximos posts.

Meme de cortesía 

![meme](https://i.ibb.co/TMg1drj/memeback.jpg")

Puedes ver los demás artículos de arquitectura hexagonal acá 😉

 * [Arquitectura hexagonal Parte I ](/posts/implementando-hexagonal-con-nestjs-part1/)
 * [Arquitectura hexagonal Parte II ](/posts/implementando-hexagonal-con-nestjs-part2/)
 * [Arquitectura hexagonal Parte III ](/posts/implementando-hexagonal-con-nestjs-part3/)

## [Github repository](https://github.com/nullpointer-excelsior/nestjs-northwind-hexagonal/tree/main/clean-architecture-examples/part-4-application-layer-patterns)

 