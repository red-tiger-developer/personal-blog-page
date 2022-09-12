---
title: 5 Arquitecturas de software para machos alfa
author: Benjamin
date: 2022-09-12 14:10:00 +0400
categories: [backend,fullstack,software-engineer, hackcode]
tags: [arquitectura, hackcode]
render_with_liquid: false
---


# 5 arquitecturas de software para proyectos serios

Cuando iniciamos a crear nuestros proyectos lo primero que nos enseñaron fue el patrón MVC o la arquitectura por capas con el tiempo
Todos los proyectos se acercaron a más una arquitectura por capas, ya que que nuestros frameworks favoritos por debajo implementaban
el patrón MVC. Pero a medida de tantos ejemplos con controller ->  service -> repository -> entities quedaron como un estándar en muchos proyectos de carácter profesional y a medida que crecía el proyecto te ibas llenando de clases services extremadamente complejas y un montón de entidades


A continuación les presentaré 5 enfoques al momento de estructurar los componentes de nuestras aplicaciones, desde un pequeño proyecto (como un microservicio) hasta una aplicación más compleja que necesite mayor mantenibilidad y facilidad de entendimiento.

## Modelo N capas (Común en tutoriales)

```
 1-basic-layers
├──  controllers
│   ├──  OrderController.java
│   └──  ProductController.java
├──  entities
│   ├──  Order.java
│   └──  Product.java
├──  repositories
│   ├──  OrderRepository.java
│   └──  ProductRepository.java
└──  services
    ├──  OrderService.java
    └──  ProductService.java
```

El primer ejemplo es el que más hemos usado al momento de crear aplicaciones básicas tanto productivas como para aprender el error que se comente al usar esta arquitectura en aplicaciones reales, es que si el proyecto crece seguimos agregando componentes como servicios repositorios o controladores y muchas veces las lógicas de servicios pueden a llegar a ser complejas y nos hará códigos intesteables si agregamos lógicas en los servicios como locos porque lo primero que nos enseñaron es que los servicios deben tener toda la lógica de negocio. Con esta premisa terminas teniendo servicios enormes llenos de responsabilidades y acoplado a otros componentes, lo cual a la larga te traera más dificultad de deducir que hace específicamente el servicio


## Modelo N capas mejorado

```
 2-n-layers
├──  application
│   ├──  OrderCreator.java
│   ├──  OrderFinder.java
│   └──  ProductCreator.java
├──  controllers
│   ├──  OrderController.java
│   └──  ProductController.java
├──  entities
│   ├──  Order.java
│   └──  Product.java
├──  repositories
│   ├──  OrderRepository.java
│   └──  ProductRepository.java
└──  services
    ├──  OrderService.java
    └──  ProductService.java
```

En este enfoque creamos capas adicionales acordes a las necesidades de nuestra aplicación, en este ejemplo se agrega la capa `application` la cual representará nuestros casos de uso y se comunicarán con las clases services entonces de esta manera separamos responsabilidades y le damos un poco más de sentido a la arquitectura N capas, ya que podríamos crear otra capa para resolver alguna nueva necesidad de la aplicación, por ejemplo que en un futuro necesitemos implementar un sistema de eventos.

## Modelo N Capas con enfoque al dominio

Este ejemplo empieza a modelar la aplicación de una forma en que refleja mas el dominio del negocio de la aplicación definiendo como principales capas:
 
 * `domain`: Modelo de negocio de nuestra aplicación dentro de esta capa consideramos repositorios entidades y nuestras clases de servicios como "servicios de dominios" son operaciones sobre las entidades
 * `application`:  Representará los casos de uso del negocio y contiene "servicios de aplicación" que son las lógicas más cercanas a la interacción de la aplicación con sus clientes. Adicionalmente, agregaremos componentes de software más alejados del dominio del negocio y más cercanos a implementaciones tecnológicas

```
3-self-explain-domain-layers
├── application
│   ├── events
│   │   └── OrderNotificationService.java
│   ├── restcontrollers
│   │   ├── OrderController.java
│   │   └── ProductController.java
│   └── services
│       ├── OrderCreator.java
│       ├── OrderFinder.java
│       └── ProductCreator.java
└── domain
    ├── entities
    │   ├── Order.java
    │   └── Product.java
    ├── repositories
    │   ├── OrderRepository.java
    │   └── ProductRepository.java
    └── services
        ├── OrderService.java
        └── ProductService.java

```


Sus ventajas más admirables son que viendo la estructura del proyecto entenderemos de mejor manera la necesidad del negocio y el propósito de nuestra aplicación. A su vez observa que agregamos en application la capa events donde agregamos un nuevo servicio llamado OrderNotificationService.java el cual se encargará de las notificaciones y este podrá ser llamado por nuestros servicios de aplicación o casos de uso OrderCreator.java



## Arquitectura basada en módulos y capas orientadas al dominio


Los anteriores ejemplos modelan la aplicación separadas en capas pero este enfoque tiene sus desventajas cuando el proyecto empieza a crecer.

* Acoplamiento de distintas funcionalidades entre entidades del dominio.
* Dificultad de entender partes específicas del sistema por la no separación de funcionalidades.
* La reutilización de entidades, servicios o repositorios evita modelar una funcionalidad basándonos en el caso de uso y te obliga a adecuarte a los componentes existentes.
* Cualquier cambio en componentes ya usados por otros pueden fallar, por lo que la cobertura de testing por más completa que sea siempre existirá el riesgo de comportamientos indeseados en producción.

para solucionar esto tomaremos el modelo anterior y lo convertiremos a una aplicacion modular

```
 4-modules-and-layers
├──  modules
│   ├──  order
│   │   ├──  application
│   │   │   ├──  events
│   │   │   │   └──  OrderNotificationService.java
│   │   │   ├──  restcontrollers
│   │   │   │   └──  OrderController.java
│   │   │   └──  services
│   │   │       ├──  OrderCreator.java
│   │   │       └──  OrderFinder.java
│   │   └──  domain
│   │       ├──  entities
│   │       │   ├──  Order.java
│   │       │   └──  OrderProduct.java
│   │       ├──  repositories
│   │       │   ├──  OrderProductRepository.java
│   │       │   └──  OrderRepository.java
│   │       └──  services
│   │           └──  OrderService.java
│   └──  product
│       ├──  application
│       │   ├──  restcontrollers
│       │   │   └──  ProductController.java
│       │   └──  services
│       │       └──  ProductCreator.java
│       └──  domain
│           ├──  entities
│           │   └──  Product.java
│           ├──  repositories
│           │   └──  ProductRepository.java
│           └──  services
│               └──  ProductService.java
└──  shared
    ├──  config
    │   └──  Constants.java
    └──  utils
        ├──  DatetimeUtils.java
        └──  TextUtils.java
```
Como puedes apreciar cada módulo tiene sus propias capas, esto nos permite incluso dentro de un módulo escoger otra estructura de capas si lo estimas conveniente a su vez si observas el módulo `order` este posee su propia entidad `Product` la cual esta desacoplada del módulo `product` 

Si te preguntas ¿Qué consigo duplicando entidades de un ORM si puedo reusarla? La respuesta es que cuando modelas tu dominio debes hacerlo de acuerdo al objetivo de negocio y no de acuerdo al modelo de datos exacto, ya que cuando usas un ORM en algunos casos necesitaras ciertas configuraciones o relaciones que no encajaran con otros módulos de la aplicación con este enfoque te ahorras problemas de mapeo de la entidad a la tabla, ya que hay casos donde necesitas una carga perezosa de alguna relación, pero en otro caso de uso de la aplicación necesitara una carga temprana de esa entidad, entonces la independencia de los módulos es una ventaja enorme al modelar una nueva funcionalidad. En cuanto a la reutilización de módulos estos deberían exponer sus servicios mediante la capa aplicación  y la definicion de DTOs. esta arquitectura es ideal para proyectos medianos o grandes

## Arquitectura Hexagonal

La arquitectura hexagonal o arquitectura de cebolla se enfoca en la definición del modelo de negocio como el corazón de nuestra aplicación capa "domain" esta capa es seguida por la encargada de representar los casos de uso "application" y finalmente tendremos nuestra capa de "infraestructura" la cual implementara todo el mundo externo de nuestra aplicación como librerías de terceros, frameworks, apis, protocolos, comunicación, orm, etc es decir todo lo que no sea nuestra lógica de negocio.


La comunicación de las capas de domain, application e infraestructure se hace mediante componentes llamados "ports" y "adapters"que son básicamente interfaces y sus implementaciones para comunicar las capas de la arquitectura hexagonal.

```
hexagonal
├──  infraestructure
│   ├──  order
│   │   ├──  adapters
│   │   │   ├──  events
│   │   │   │   └──  RabbitMQOrderNotificator.java
│   │   │   └──  repositories
│   │   │       ├──  OrderEntity.java
│   │   │       ├──  OrderProductEntity.java
│   │   │       ├──  PostgresOrderProductRepository.java
│   │   │       └──  PostgresOrderRespository.java
│   │   └──  restcontrollers
│   │       └──  OrderController.java
│   ├──  product
│   │   ├──  adapters
│   │   │   └──  repositories
│   │   │       ├──  PostgresProductRespository.java
│   │   │       └──  ProductEntity.java
│   │   └──  restcontrollers
│   │       └──  ProductController.java
│   └──  shared
│       └──  utils
│           └──  ThirdPartyLibraryDatetimeUtilImplementation.java
├──  modules
│   ├──  order
│   │   ├──  application
│   │   │   ├──  OrderCreator.java
│   │   │   ├──  OrderFinder.java
│   │   │   └──  ports
│   │   │       └──  events
│   │   │           └──  OrderNotificator.java
│   │   └──  domain
│   │       ├──  entities
│   │       │   ├──  Order.java
│   │       │   └──  OrderProduct.java
│   │       ├──  ports
│   │       │   └──  repositories
│   │       │       ├──  OrderProductRepository.java
│   │       │       └──  OrderRepository.java
│   │       └──  services
│   │           └──  OrderService.java
│   └──  product
│       ├──  application
│       │   └──  ProductCreator.java
│       └──  domain
│           ├──  entities
│           │   └──  Product.java
│           ├──  ports
│           │   └──  repositories
│           │       └──  ProductRepository.java
│           └──  services
│               └──  ProductService.java
└──  shared
    ├──  config
    │   └──  Constants.java
    └──  utils
        ├──  DatetimeUtils.java
        └──  TextUtils.java

```


Por ejemplo acá en nuestra capa de dominio tenemos las interface ProductRepository la cual es un puerto comunica nuestro dominio con una fuente de datos entonces en nuestra capa de infraestructura definimos una implementación de este puerto que sería nuestro adaptador en este caso seria PostgresProductRepository si te fijas nuestra implementación es bastante autoexplicativa ya sabes que hay metido un postgres en nuestra aplicación y si quieres puedes crear otro adaptador porque en un futuro se desea utilizar NoSql solo implementas un nuevo adapter o si en tus pruebas unitarias necesitas un mock de ProductRepository lo puedes crear implementando una clase de prueba sin necesidad de usar una librería de terceros para hacer un Mock del componente

La arquitectura hexagonal es ideal para proyectos grandes y serios, ya que podrás implementar otras arquitecturas o estrategias dentro de la estructura, también es algo que debemos manejar a un nivel más abstracto y conceptual para entender como aplicarlo, encontraras un montón de implementaciones de hexagonal y ninguna estructura o nombres se parecen a esta o a las que veas, pero si debes entender los conceptos principales


Con la separación de la lógica del negocio, casos de usos, componentes externos y la definición de puertos y adaptadores lograrás mayor testeabilidad y mantenibilidad de tu maravillosa aplicación además de que el objetivo y las tecnologías empleadas se entienden a simple vista.

Mi principal recomendación son estas 3 ultimas arquitecturas
son mas autoexplicativas y pueden escalar a arquitecturas mas complejas sin perder mantenibilidad o testeabilidad de tu aplicación


Conclusión para la casa solo memes
