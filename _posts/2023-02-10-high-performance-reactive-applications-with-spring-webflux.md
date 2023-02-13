---
title: Aplicaciones Reactivas de Alto Rendimiento con Spring WebFlux
author: Benjamin
date: 2023-02-10 10:32:00 -0500
categories: [Programacion, Java, Arquitectura de software, Spring, Springboot, Reactive]
tags: [Java, Spring, Reactive, Reactor]
---

![image](https://i.ibb.co/TLTm5Jw/Screen-Shot-2023-02-12-at-23-31-52.png)

## ¿Qué es Spring WebFlux 

Spring WebFlux es un proyecto del Framework Spring que permite el desarrollo de aplicaciones web asíncronas siguiendo el paradigma de programación reactivo. Ofrece una alternativa al modelo síncrono de Spring Web MVC donde cada solicitud es atendida por un hilo separado. 

La principal característica de WebFlux es su enfoque asíncrono y no bloqueante basado en la gestión de hilos similar al Event Loop de NodeJs, lo que permite procesar solicitudes de manera eficiente y escalable, especialmente en aplicaciones web con un alto tráfico de datos. 


### Entendiendo FLux y Mono

Para crear aplicaciones reactivas debemos conocer a Flux y Mono que son objetos en el marco de trabajo Reactor que 
representan flujos de datos y eventos asíncronos. 

 * `Flux:` representa flujos con múltiples valores 
 * `Mono:` representa flujos con un solo valor. 

Tanto Flux y Mono Ofrecen operadores para manipular y transformar los flujos de datos en aplicaciones reactivas.
A su vez nos ofrece una manera simple de crear flujos a partir de otros objetos. 

### Subscripciones 

Para utilizar nuestros flujos reactivos debemos subscribirnos a estos ya que los objetos Flux son lazy. También nos da la posibilidad que puedan existir múltiples subscriptores a un flujo. 

```java
var flux = Flux.fromIterable(Arrays.asList(10,20,30,40,50)); // creating a Integer flux
flux.subscribe(System.out::println); // subscribe to get values
```
Entendiendo las bases implementaremos una API simple de ejemplo basados en el paradigma reactivo utilizado por WebFLux.

## ¿Qué es la programación reactiva?

Es un paradigma de programación que se centra en la gestión de flujos de datos y eventos asíncronos.
Se basa en la idea de que las aplicaciones deben responder de forma eficiente a los cambios en los flujos de datos
y eventos, incluso cuando estos cambios son frecuentes y volátiles.

Este paradigma se describe en el [Manifiesto Reactivo](https://www.reactivemanifesto.org/pdf/the-reactive-manifesto-2.0-es.pdf) esto quiere decir que los sistemas reactivos son:

* `Responsivos:` tiempos de respuesta rápidos y consistentes.
* `Resilientes:` permite que el sistema siga funcionando incluso en caso de fallos.
* `Elasticicos:` permite al sistema adaptarse a cambios durante las cargas de trabajo.
* `Orientados a mensajes:` cuando un valor es emitido todo aquel componente que se halla suscrito obtendrá ese valor. 

El intercambio asíncrono de mensajes nos permite establecer fronteras entre componentes y mejorar la gestión de la carga, la elasticidad y el control de flujo.


## Implementación de API reactiva

Implementaremos nuestra aplicación reactiva basada en el artículo acerca de [Java Stream](https://nullpointer-excelsior.github.io/posts/tranajando-con-1millon-de-registros-con-java-stream) básicamente trabajaremos sobre una tabla producto. Organizaremos nuestro repositorio como monorepositorio transformamos nuestro ejemplo de Stream de java en una librería llamada `products` y creamos las siguientes aplicaciones:

* `api-webflux:` API Server que implementará `spring-webflux`.
* `api-blocking:` API Server bloqueante que usaremos para comparar el rendimiento contra una aplicación reactiva.
* `client-reactive:` Cliente http que consumirá nuestra API Reactiva.

Declaramos la dependecia necesaria en `gradle.build`:

```bash
# gradle.build

# ... other configs
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation project(":libs:products")
}
```

Nuestra aplicación `api-webflux` solo contendrá una clase Main que inicia una aplicación Spring hacemos uso de nuestra 
librería `products` importada en el archivo de configuración `gradle.build`  

```java

@Log4j2
@RestController 
@RequestMapping("product")
@SpringBootApplication
@Import(ProductConfiguration.class) // import spring configuration from products lib
public class WebFluxApplication {

    @Autowired
    private ProductService products;

    public static void main(String[] args) {
        SpringApplication.run(WebFluxApplication.class, args);
    }

    @GetMapping
    public Flux<Product> getAllProducts() {
        log.info("All products request");
        return Flux.fromStream(products.getAllProducts());
    }

}

```
Para crear nuestro endpoint reactivo solo debemos devolver un objeto Flux y spring por debajo hará su magia. 
Como nuestro caso de uso retorna un objeto Stream hacemos uso del método `fromStream()` de la clase `Flux`.
Con esto nuestra api reactiva esta lista.

## Implementación de cliente reactivo

En nuestro proyecto cliente `client-reactive` haremos uso de `WebClient` (librería incorporada dentro de WebFlux) con el cual podremos obtener los datos en forma asíncrona.  `WebClient` nos permite trabajar con flujos asíncronos proporcionando todas las bondades de la librería Reactor, A diferencia del clásico cliente de Spring `RestTemplate` el cual es bloqueante.


El siguiente código muestra como utilizar `WebClient` de forma básica. Podremos crear nuestro cliente dependiendo de nuestras necesidades.

```java
WebClient client = WebClient.create("http://localhost:8080");

Flux<Product> products = client.get()
        .uri("/product")
        .retrieve()
        .bodyToFlux(Product.class);

products.subscribe(System.out::println);

```

Ahora Definimos nuestro `WebClient` como un bean de Spring disponible en toda la aplicación y lo inyectamos en nuestro componente `ProductAPI`

```java
// App Configuration
@Configuration
@SpringBootApplication
public class AppClientReactive {
    // ..more code
    @Bean
    public WebClient getWebClient() {
        return WebClient.create("http://localhost:8080");
    }
}

// Product API Component
@Component
public class ProductAPI {

    @Autowired
    private WebClient client;

    public Flux<Product> getProducts() {
        return client.get()
                .uri("/product")
                .retrieve()
                .bodyToFlux(Product.class);

    }

}

```

Finalmente creamos nuestro `ProductService` que contendrá nuestros casos de uso utilizando las funciones que nos ofrece el paradigma reactivo.

```java
@Service
public class ProductService {

    @Autowired
    private ProductAPI api;

    public Flux<String> getBrands() {
        return api.getProducts()
                .map(Product::getBrand)
                .distinct();
    }

    public Flux<Product> getAllProducts() {
        return api.getProducts();
    }

    public Flux<Product> getProductWithoutStock() {
        return api.getProducts()
                .filter(product -> product.getStock() <= 0);
    }

    public Mono<Long> countSkuWithoutStock() {
        return api.getProducts()
                .filter(product -> product.getStock() <= 0)
                .count();
    }

    public Mono<Integer> sumStockDepartment604() {
        return api.getProducts()
                .filter(product -> product.getDepartment().equals("604"))
                .map(Product::getStock)
                .reduce(0, Integer::sum);
    }

    public Flux<Brand> groupByDepartment604Brand() {

        Function<Map<String, Collection<Product>>, Stream<Brand>> mapToBrand = (Map<String, Collection<Product>> map) -> map
                .entrySet()
                .stream()
                .map(Brand::fromEntrySet);

        return api.getProducts()
                .filter(product -> product.getDepartment().equals("604"))
                .groupBy(p -> p.getBrand() != null ? p.getBrand() : "No brand")
                .flatMap(group -> group.collectMultimap(Product::getBrand, item -> item))
                .map(mapToBrand)
                .flatMap(Flux::fromStream);

    }

    public Mono<Long> getCount() {
        return api.getProducts().count();
    }
}
```

Las operaciones realizadas son muy similares a lo que lograríamos con Java Stream, pero esta se basa en un modelo de "pull", 
en el que los datos se consumen poco a poco, a medida que se los va solicitando.
En cambio ReactiveStream (En Reactor Flux) es un modelo "push-pull", que permite que los datos se produzcan
y se consuman de forma asíncrona. Se puede considerar a Flux como una mezcla de Stream + CompletableFuture con las siguientes características:

* muchos operadores aplicables a los flujos de datos.
* soporte de BackPressure (velocidad de producción VS la velocidad de consumo de flujos).
* control sobre el comportamiento del publicador y suscriptor.
* control sobre la noción de tiempo (ventanas de almacenamiento de valores, agregar tiempos de espera y alternativas, etc.)

Ya entendiendo las diferencias podemos aplicar operaciones más complejas a nuestro flujo reactivo como pueden ser: 

múltiple subscripciones:
```java
var products = productService
                .getProductWithoutStock()
                .share(); // Nos permite compartir una subscripcion a un unico flujo
        
products.subscribe(System.out::println);

products.subscribe(p -> stockService.updateStock(p));

products.subscribe(p -> emailService.notify(p));

```

Operaciones sobre el tiempo
```java
var products = productService
        .getProductWithoutStock()
        .delayElements(Duration.ofMillis(100));
products.subscribe(System.out::println);
```

Reintentos dependiendo de las condiciones que queramos contemplar:

```java
var products = productService
                .getProductWithoutStock()
                .retryWhen(Retry.fixedDelay(3L, Duration.ofSeconds(10))); // Retry nos permitira construir la estrategia de retry mas adecuada a nuestro contexto

products.subscribe(System.out::println);

```

También podemos realizar otras operaciones más complejas que necesitan ser explicadas más en profundidad, tales como:

* Control sobre quien va a ejecutar las tareas permitiéndonos el control de los hilos de manera pragmática
* crear subscripciones internas sobre cada elemento de un flujo.
* terminar una subscripción y unirse a otra.
* manejar múltiples flujos dentro de un mismo flujo. 

WebFlux utiliza el proyecto [Reactor](https://projectreactor.io/) para poder trabajar con el paradigma reactivo. su documentación es bien completa a nivel conceptual y en el uso de su API, pero el punto de partida ideal sería probar [Spring WebFlux](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#spring-webflux) el cual nos permitirá crear aplicaciones reactivas de manera sencilla. Este tipo de enfoque de desarrollo es ideal para microservicios.

## Comparando WebMvc vs WebFlux

Compararemos una API desarrollada con Spring WebMvc bloqueante y otra diseñada con WebFLux no bloqueante. Definimos nuestra dependecia en la aplicación `api-blocking`:

```bash
# gradle.build

# ... other configs
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation project(":libs:products")
}
```

Y montamos un clásico web server con spring

```java
@Log4j2
@RestController
@RequestMapping("product")
@SpringBootApplication
@Import(ProductConfiguration.class)
public class WebApplication {

    @Autowired
    private ProductRepository repository;

    public static void main(String[] args) {
        SpringApplication.run(WebApplication.class, args);
    }

    @GetMapping
    public Stream<Product> getProducts(
            @RequestParam(value= "limit", defaultValue="100") Integer limit,
            @RequestParam(value="offset", defaultValue="0") Integer offset
    ) {
        log.info("Web-app Request limit: {}, offset: {}", limit, offset);
        return repository.findByPaginated(limit, offset).stream();
    }

}

```

Y Ahora podemos hacer el siguiente ejercicio con `curl` para ver la diferencia entre una api bloqueante y asíncrona.

`Petición bloqueante`

![curl1](https://i.ibb.co/4pJpCXs/Screen-Shot-2023-02-10-at-13-17-26.png)

Nuestra petición bloqueante esperará a terminar la operación de la API y nos devolverá los datos.

`Petición no bloqueante`

![curl2](https://i.ibb.co/XY9pGvP/Screen-Shot-2023-02-10-at-13-20-39.png)


Nuestra petición a la API reactiva va obteniendo los datos en forma parcial, no necesitamos que el servidor termine de realizar las operaciones
para poder trabajar, nuestro cliente no se quedara bloqueado esperando una respuesta ya los datos vienen en un flujo de manera asíncrona.
En este caso, la respuesta de nuestra API nos permite trabajar de forma reactiva.

Resumiendo las diferencias entre una API síncrona como Spring Web y el enfoque asíncrono que nos brinda WebFlux serían:

* `Spring Web` utiliza un modelo de programación síncrono y basado en hilos, donde un hilo se bloquea hasta que se recibe una respuesta del servidor. Este modelo es efectivo para muchos casos, pero puede ser limitante en términos de escalabilidad y rendimiento en entornos de alto tráfico.

* `Spring WebFlux` utiliza un modelo de programación asíncrono y no bloqueante basado en un modelo concurrente EventLoop, donde el servidor puede manejar muchas solicitudes simultáneamente sin bloquear los hilos. Este modelo es más escalable y eficiente en entornos de alto tráfico y ofrece mejores tiempos de respuesta y una mejor gestión de recursos.


## Conclusiones

Este es la primera mirada que le damos a Spring WebFlux y El modelo Reactivo la verdad este paradigma da para muchos ejemplos prácticos y casos de uso. Este enfoque es ideal para crear microservicios resilientes y escalables cuando necesitamos una comunicación síncrona podemos facilmente pasar a un modelo asíncrono con las ventajas que nos da la programación reactiva. Si estás desarrollando una aplicación web de tamaño medio con una cantidad moderada de tráfico, Spring Web probablemente sea suficiente. Pero si estás desarrollando una aplicación web de alta escalabilidad con una gran cantidad de tráfico, Spring WebFlux es la opción más adecuada debido a su enfoque reactivo y no bloqueante.  



## [Github repository](https://github.com/nullpointer-excelsior/java-high-performance-api)

Meme de cortesía

![meme](https://i.ibb.co/mGncR8v/Screen-Shot-2023-02-12-at-23-57-49.png)

