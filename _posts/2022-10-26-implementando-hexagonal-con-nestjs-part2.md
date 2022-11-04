---
title: Arquitectura hexagonal Parte II Conectando una clean architecture con Nestjs
author: Benjamin
date: 2022-10-26 10:32:00 -0500
categories: [Programacion, Nestjs, Arquitectura de software, Typescript]
tags: [typescript, nestjs, hexagonal]
---


![logo](https://i.ibb.co/6nTjp5n/Screen-Shot-2022-11-02-at-16-17-23.png)

La segunda parte de esta serie de arquitectura hexagonal o (clean architecture) profundizaremos como configurar la interacción de un framework con el core de nuestra arquitectura, Nestjs nos ofrece un diseño adaptable para poder cubrir casos de uso avanzado. La idea es estar lo menos acoplados al framework y que este se adapte a nuestra capa `core` la cual contienen los casos de uso, las reglas de negocio y las entidades.

## ¿Qué es Nestjs? short answer

Nestjs es un framework progresivo de NodeJS desarrollado en TypeScript diseñado para facilitar el desarrollo de aplicaciones backend, aportando a los programadores una buena estructura y metodología inicial. Una de sus características es que puedes desarrollar cualquier tipo de proyecto backend, ya que las integraciones están ya disponibles en el framework y solo necesitaras instalar el módulo para poder trabajar, otro punto importante es que sigue la misma arquitectura que angular podemos desarrollar aplicaciones mediante anotaciones, módulos y podemos hacer uso de la inyección de dependencias para crear nuestras clases de servicios,
Resumiendo Nestjs es como el spring de Nodejs.

## Custom providers y módulos dinámicos en Nestjs 

### Custom Providers

Unos de los conceptos principales de nestsjs son los provider que no son nada más que clases de tipo service, objetos o valores que pueden ser inyectados en la aplicación la forma simple es mediante anotaciones, pero como nuestro proyecto necesita ser integrado al framework de una forma limpia emplearemos el uso de custom providers para lograr nuestro objetivo.

### Módulos dinámicos

Los módulos básicamente son una pieza de software que agrupa una capa o una funcionalidad de la aplicación esta contienen controllers, providers y toda pieza de software relacionada con la idea del módulo. Los componentes creados bajo el contexto de Nestjs pueden estar aislados o ser exportadas a otros módulos.
En Nestjs puedes crear módulos estáticos en los cuales definimos dependencias, providers y controllers y con esto podras reutilizarlo en otros módulos, pero en escenarios más complejos puedes definirlos de forma dinámica, esta característica nos permite crear fácilmente módulos personalizables que pueden registrar y configurar proveedores de forma dinámica. Justo lo que necesitamos para nuestra aplicación.


## Manos a la obra integrando nuestra arquitectura hexagonal con Nestjs

La estructura de nuestra aplicación actualmente es esta, ya hay definidos unos archivos `XXX.module.ts` entonces partiremos por configurar el módulo `core`.

```text
├──  core
│   ├──  application
│   │   ├──  ProductApplication.ts
│   │   └──  services
│   │       ├──  ProductApplicationService.spec.ts
│   │       └──  ProductApplicationService.ts
│   ├──  core.module.ts
│   ├──  domain
│   │   ├──  entities
│   │   │   ├──  Category.ts
│   │   │   ├──  Product.ts
│   │   │   └──  Supplier.ts
│   │   ├──  ports
│   │   │   ├──  inbound
│   │   │   │   ├──  CategoryService.ts
│   │   │   │   ├──  ProductService.ts
│   │   │   │   └──  SupplierService.ts
│   │   │   └──  outbound
│   │   │       ├──  CategoryRepository.ts
│   │   │       ├──  ProductRepository.ts
│   │   │       └──  SupplierRepository.ts
│   │   └──  services
│   │       ├──  CategoryDomainService.spec.ts
│   │       ├──  CategoryDomainService.ts
│   │       ├──  ProductDomainService.spec.ts
│   │       ├──  ProductDomainService.ts
│   │       ├──  SupplierDomainService.spec.ts
│   │       └──  SupplierDomainService.ts
│   └──  shared
│       ├──  dto
│       │   └──  NewProductDTO.ts
│       └──  error
│           ├──  ProductApplicationError.ts
│           └──  ProductServiceError.ts
├──  infraestructure
│   ├──  adapters
│   │   ├──  category.repository.adapter.ts
│   │   ├──  product.repository.adapter.ts
│   │   └──  supplier.repository.adapter.ts
│   ├──  http-server
│   │   ├──  controllers
│   │   │   ├──  product.controller.ts
│   │   │   └──  root.controller.ts
│   │   ├──  exception-filters
│   │   │   └──  product-exception.filter.ts
│   │   ├──  http-server.module.ts
│   │   ├──  model
│   │   │   ├──  app.response.ts
│   │   │   └──  create-product.request.ts
│   │   └──  utils
│   │       └──  generate-swagger-docs.ts
│   ├──  infraestructure.module.ts
│   ├──  northwind-database
│   │   ├──  entities
│   │   │   ├──  category.entity.ts
│   │   │   ├──  product.entity.ts
│   │   │   └──  supplier.entity.ts
│   │   └──  northwind-database.module.ts
│   └──  shared
│       ├──  Log.ts
│       └──  shared.module.ts

```

Para crear un módulo dinámico debemos crear un método estático que devolverá un objeto `DynamicModule` del paquete `@Nest/common` que básicamente es lo mismo que usar la anotación `@Module` la cual define las propiedades del módulo la única diferencia es que DynamicModule el parámetro module es obligatorio y hace referencia al mismo módulo. El nombre del módulo `register()` no es mandatorio por convención los nombres utilizados son `forRoot()`, `forAsyncRoot()`, `register()` y `asyncRegister()` este método puede ser tanto asíncrono como síncrono.

```typescript
import { DynamicModule, Module, Type } from '@Nestjs/common';

@Module({})
export class CoreModule {

  static register(): DynamicModule {
    return // my awesome module config
  }

}

```
El siguiente paso será definir que parámetros recibirá nuestro método, ya que acá es donde la definiremos la configuración de nuestro módulo. entonces creamos la siguiente interface

```typescript
export type CoreModuleOptions = {
  modules: Type[];
  adapters: {
    productService: Type<ProductService>;
    categoryService: Type<CategoryService>;
    supplierService: Type<SupplierService>;
  }
}
```
La propiedad `modules` recibirá los módulos dependencias que contendrán los servicios que necesitemos inyectar en nuestro módulo `core`.
Esta interface mediante su porpiedad `adapters` recibirá las implementaciones de las interfaces `ProductService`, `CategoryService` y `SupplierService` las cuales definimos como puertos en nuestra capa `core`.


Nuestro módulo va tomando esta forma.

```typescript
import { DynamicModule, Module, Type } from '@Nestjs/common';

@Module({})
export class CoreModule {

  static register(options: CoreModuleOptions): DynamicModule {
    
    const { adapters, modules } = options
    const { productService, categoryService, supplierService } = adapters

    return {
      module: CoreModule,
      providers: []
    }

  }

}
```
Muy bien el próximo paso es definir los `providers` en este caso necesitamos primero definir los token que son nada más que strings únicos que serán utilizados para identificar que `provider` queremos inyectar

definimos los siguientes providers:

```typescript
// Application service reference
export const PRODUCT_APPLICATION = 'PRODUCT_APPLICATION'
// domain services references
export const CATEGORY_SERVICE = 'CATEGORY_SERVICE'
export const PRODUCT_SERVICE = 'PRODUCT_SERVICE'
export const SUPPLIER_SERVICE = 'SUPPLIER_SERVICE'
```

Ahora crearemos una objeto que definirá nuestro custom provider este deberá tener las siguientes propiedades:

* `provide`: es la clase del provider o un string con el nombre de un servicio el cual Nestjs tendrá que inyectar en donde se solicite su uso, en nuestro caso no vamos a inyectar una clase sino una interfaz, por lo tanto, debemos definir un nombre, ya que en Nestjs para inyectar un valor que no es una clase se necesita definir un Injection token que nada más que un string identificatorio
* `useFactrory`: es una función que recibe argumentos y devuelve el objeto que queremos definir como un custom provider los argumentos que recibirá esta función en este caso son los servicios que queremos que Nestjs inyecte a nuestro ProductApplicationService (casos de uso de la entidad productos)y para poder inyectar los servicios que necesita se hace mediante la propiedad inject
* `inject`: es un array de proveedores existentes en el módulo los cuales serán los parámetros de la función useFactory()

Entonces para crear un Custom Provider para un servicio de dominio será de la siguiente manera:


Como nuestro modelo lo indica para crear el servicio de dominio ProductService

```typescript
export class ProductDomainService implements ProductService {

    constructor(private repository: ProductRepository) {}

    async save(product: Product): Promise<Product> {
        if (this.validateProductPrice(product)) {
            return this.repository.save(product)
        }
        throw new ProductServiceError('Product price cannot be negative or equal to zero')
    }

    validateProductPrice(product: Product): boolean {
        return product.unitPrice > 0
    }

}
```
necesitamos de un ProductRepository

```typescript
constructor(private repository: ProductRepository) {}
```

Donde  `productRepository` proviene del objeto `CoreModuleOptions` entonces creamos una función `ProductServiceProvider` donde injectaremos los servicios necesarios para crear nuestro `ProductDomainService`

```typescript

const ProductServiceProvider = {
  provide: PRODUCT_SERVICE, // provider token
  useFactory(repository: ProductRepository) {
    // return a service instance 
    return new ProductDomainService(repository)
  },
  inject:[
    productRepository // get this value from CoreModuleOptions
  ]
}
```

La función useFactory es sencilla solo recibe los providers inyectados por Nestjs que necesitamos y con eso podemos instanciar un Servicio de forma manual pero adecuado a nuestras necesitades.

En este otro caso nosotros devolvemos la instancia de una implementacion de la interfaz `ProductApplication`

```typescript

  function useFactory(product: ProductDomainService, category: CategoryDomainService, supplier: SupplierDomainService) {
    // ProductApplication implementation
    return new ProductApplicationService(product, category, supplier)
  }
```

La inyección de nuestro Servicio de applicación será de la siguiente manera con la diferencia que injectamos los token en vez de los adapters que recibiremos:

```typescript
const ProductApplicationProvider = {
      provide: PRODUCT_APPLICATION,
      useFactory(product: ProductDomainService, category: CategoryDomainService, supplier: SupplierDomainService) {
        return new ProductApplicationService(product, category, supplier)
      },
      inject: [
        PRODUCT_SERVICE,
        CATEGORY_SERVICE,
        SUPPLIER_SERVICE
      ]
    }
```

Finalmente nuestro módulo core es el siguiente

```typescript
/**
 * Options for core module 
 */
export type CoreModuleOptions = {
  modules: Type[];
  adapters?: {
    productRepository: Type<ProductRepository>;
    categoryRepository: Type<CategoryRepository>;
    supplierRepository: Type<SupplierRepository>;
  }
}

/**
 * Providers token for netsjs injection
 */
export const PRODUCT_APPLICATION = 'PRODUCT_APPLICATION'
export const CATEGORY_SERVICE = 'CATEGORY_SERVICE'
export const PRODUCT_SERVICE = 'PRODUCT_SERVICE'
export const SUPPLIER_SERVICE = 'SUPPLIER_SERVICE'


@Module({})
export class CoreModule {

  static register({ modules, adapters }: CoreModuleOptions): DynamicModule {

    const { categoryRepository, productRepository, supplierRepository } = adapters
    /**
     * ApplicationService Provider
     * */
    const ProductApplicationProvider = {
      provide: PRODUCT_APPLICATION,
      useFactory(product: ProductDomainService, category: CategoryDomainService, supplier: SupplierDomainService) {
        return new ProductApplicationService(product, category, supplier)
      },
      inject: [
        PRODUCT_SERVICE,
        CATEGORY_SERVICE,
        SUPPLIER_SERVICE
      ]
    }

    /**
     * DomainService Providers
     * */
    const CategoryServiceProvider = {
      provide: CATEGORY_SERVICE,
      useFactory(repository: CategoryRepository) {
        return new CategoryDomainService(repository)
      },
      inject:[
        categoryRepository
      ]
    }

    const SupplierServiceProvider = {
      provide: PRODUCT_SERVICE,
      useFactory(repository: ProductRepository) {
        return new ProductDomainService(repository)
      },
      inject:[
        productRepository
      ]
    }

    const ProductServiceProvider = {
      provide: SUPPLIER_SERVICE,
      useFactory(repository: SupplierRepository) {
        return new SupplierDomainService(repository)
      },
      inject:[
        supplierRepository
      ]
    }

    return {
      module: CoreModule,
      global: true,
      imports: [
        ...modules
      ],
      providers: [
        ProductApplicationProvider,
        CategoryServiceProvider,
        SupplierServiceProvider,
        ProductServiceProvider
      ],
      exports: [
        PRODUCT_APPLICATION
      ],
    }
  }

}
```
Ahora lo siguiente será registrar a `core` en la raiz de todos módulos en `app.module` llamando al método register().

```typescript
CoreModule.register({
  modules: [
      InfraestructureModule
  ],
  adapters: {
    productRepository: ProductRepositoryAdapter,
    categoryRepository: CategoryRepositoryAdapter,
    supplierRepository: SupplierRepositoryAdapter
  }
}),
```
El módulo principal de la aplicación registrará a `core` y shared` de forma global es decir, estará disponible para todos los módulos que definamos en nuestra aplicación sin necesidad de hacer excesivos imports en nuestras definiciones de módulos.

Para lograr esto se hace mediante la propiedad `global: true` definiendo en el decorador `@Module()` o en el objeto `DynamicModule`.

```typescript

@Module({
  global: true,
  ...otherprops
})

@Module({})
export class CoreModule {

  static register(options: CoreModuleOptions): DynamicModule {
    // ...ommited 
    return {
      module: CoreModule,
      global: true,
      // other props omitted
  }

}

```

Por último registramos el módulo `infraestructure` para terminar de conectar todas las capas de nuestra aplicación. El resultado final es el siguiente en AppModule:

```typescript
@Module({
  imports: [
    InfraestructureModule,
    SharedModule,
    CoreModule.register({
      modules: [
        InfraestructureModule
      ],
      adapters: {
        productRepository: ProductRepositoryAdapter,
        categoryRepository: CategoryRepositoryAdapter,
        supplierRepository: SupplierRepositoryAdapter
      }
    }),
  ]
})
export class AppModule { }
```

Finalmente integramos nuestra arquitectura hexagonal con Nestjs de una manera totalmente desacoplada del framework. La versatilidad de los módulos dinámicos y custom providers nos permiten crear aplicaciones mantenibles y con lógicas desconectadas de frameworks o las librerías de moda. Este mismo enfoque de los módulos dinámicos es una buena práctica al momento de definir grupos de piezas de software que queremos que sean flexibles y configurables de una forma centralizada.

## [Github repository](https://github.com/nullpointer-excelsior/nestjs-northwind-hexagonal/tree/main/clean-architecture-examples/part-2-connecting-core-infraestructure-with-nestjs)

Puedes ver los demás artículos de arquitectura hexagonal acá 😉

 * [Arquitectura hexagonal Parte I ](/posts/implementando-hexagonal-con-nestjs-part1/)

## Conclusión

No soy de conclusiones estas siempre van por parte de ti y si intentas poner en práctica lo expuesto podrás tener un concepto más amplio del tema e incluso poder aplicarlo en otros escenarios.

![meme](https://i.ibb.co/zQsBcqS/Bean-Cheating-02112022164520.jpg)
