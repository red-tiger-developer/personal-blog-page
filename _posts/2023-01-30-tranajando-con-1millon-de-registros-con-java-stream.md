---
title: Trabajando con 1 millón de registros con Java Stream
author: Benjamin
date: 2023-01-30 10:32:00 -0500
categories: [Programacion, Java, Arquitectura de software, Spring, Springboot]
tags: [Java, Spring, Stream,]
---

![image](https://i.ibb.co/wwrWnGq/Screen-Shot-2023-01-30-at-18-14-41.png)

# Trabajando con 1 millón de registros con Java Stream

Cuando trabajamos con grandes cantidades de datos lo primero que hacemos es definir un filtrado de la fuente de datos para poder 
trabajar con lo que realmente nos interesa, pero cuando el filtrado no basta y nuestro universo de datos resultante es considerable 
en Java podemos hacer uso de su API Streams.

Java streams nos proporciona una manera eficiente de recorrer y realizar operaciones sobre una secuencia de elementos.
Para demostrar las bondades y la facilidad de uso de esta API montaremos un pequeño ejemplo donde trabajaremos con 
un universo de `1 millón de registros` rescatados de una base de datos.
 
## Streams vs Arrays

Java Stream nos trae el concepto de flujo de datos donde cada valor del stream es obtenido de forma secuencial 
con la posibilidad de realizar operaciones individuales sobre cada elemento, sobre un subconjunto de ellos,
o sobre la totalidad de los mismos. Estas operaciones pueden ser ejecutadas de forma secuencial o paralela. 
Este enfoque promueve la programación funcional y la posibilidad de encadenamiento de estas operaciones a modo de pipelines.

Diferencia entre Streams y Arreglos:

* No son almacenes ni estructura de datos.
* Los valores son obtenidos desde una fuente y pasan a través de una cadena de operaciones.
* Están orientados a la programación funcional.
* No modifican la fuente de la que se obtienen datos.
* No realizan side effects.
* Promueven la inicialización y evaluación perezosa (lazy) e implementan sus operaciones de forma perezosa cuando sea posible.
* Un elemento es visitado solo una vez.
* No tienen un tamaño fijo. Un stream puede alimentarse de una fuente de datos infinita.

## Programación imperativa vs Funcional

La capacidad de utilizar programación funcional nos da la posibilidad de crear código más legible y mantenible cuando 
las lógicas de negocio van cambiando y el volumen de datos empieza a ser un problema y necesitamos agregar nuevas operaciones.

Un pequeño ejemplo de programación imperativa es este:

```java
List<Integer> ages = Arrays.asList(25, 30, 45, 28, 32);

var totalAges = 0;
for(int age: ages) {
    totalAges += age;
}
System.out.println(totalAges);


```
 Y el enfoque funcional es el siguiente:
```java
List<Integer> ages = Arrays.asList(25, 30, 45, 28, 32);

int computedAges = ages
        .stream()
        .reduce(0, (a, b) -> a + b, Integer::sum);

System.out.println(computedAges);

```
Ambos códigos hacen lo mismo, pero no se comportarán de igual manera cuando la cantidad de registros sea más grande 
y aumenten los requerimientos de lógicas de negocio. El rendimiento también será comprometido intentaremos evitar 
realizar más bucles de los necesarios sobre nuestros registros, pero esto hará el código menos legible. 
Con Streams nuestros registros serán recorridos de forma secuencial nuestras operaciones podrán ser encadenadas y cada 
una de ellas tendrá su responsabilidad definida mejorando el testing y la posibilidad de reutilización. 

Basta de explicaciones esto lo veremos con código y sus correspondientes hacks 😉, a sí que manos a la obra.

## Levantando 1 millón de datos

Puedes descargar este [repositorio](https://github.com/nullpointer-excelsior/java-high-performance-api.git) y ver en detalle el código y si quieres 
realizar las pruebas a tu gusto, Ahora levantamos nuestra base de datos haciendo uso del script `product.sql` el que contiene 
nuestro universo.

```bash
#!/bin/bash

unzip database.zip

docker run --name streamdata -e POSTGRES_USER=streamdata -e POSTGRES_PASSWORD=streamdata -e POSTGRES_DATABASE=streamdata -p 5432:5432 -v "$PWD/product.sql:/docker-entrypoint-initdb.d/product.sql" -d postgres

```

## Arquitectura de software

Nuestro proyecto implementa una arquitectura hexagonal para un mejor entendimiento de nuestros componentes. Además hacemos uso de spring-boot.

### Capa de dominio 
nuestro dominio es simple:


`Entidad: Product`
```java
@Getter
@Builder
@ToString
public class Product {
    private Integer sku;
    private String name;
    private Double price;
    private Integer stock;
    private String department;
    private String brand;
}
```
`Puerto: ProductRepository`

```java
public interface ProductRepository {

    List<Product> findByPaginated(int limit, int offset);

}
```

Nuestro repositorio tiene un único método que se encarga de leer los productos de forma paginada este enfoque nos 
permitirá crear un Stream de Products de forma eficiente sin tener que leer toda la tabla.

### Capa de Infraestructura

Nuestra base de datos empleada es postgres la configuración de las credenciales está dada en el archivo `application.yml`

```yaml
spring.jpa:
  database: POSTGRESQL
  hibernate:
    ddl-auto: none
  show-sql: false

spring.datasource:
  platform: postgres
  driverClassName: org.postgresql.Driver
  url: jdbc:postgresql://localhost:5432/streamdata
  username: streamdata
  password: streamdata
```
La definición de nuestra entidad y repositorio JPA son las siguientes:

```java
@Getter
@Setter
@ToString
@Entity
@Table(name = "product")
public class ProductEntity {
    @Id
    @Column(name = "sku")
    private Integer sku;

    @Column(name = "name")
    private String name;

    @Column(name = "price")
    private Double price;

    @Column(name = "stock")
    private Integer stock;

    @Column(name = "department")
    private String department;

    @Column(name = "brand")
    private String brand;

}

public interface JpaProductRepository extends JpaRepository<ProductEntity, Integer>  {

    @Query(value="SELECT * FROM product p ORDER BY p.sku LIMIT :limit OFFSET :offset ", nativeQuery = true)
    List<ProductEntity> findByPaginated(@Param("limit")int limit, @Param("offset") int offset);

}
```
Para integrar la persistencia con el dominio implementamos nuestro adaptador `ProductRepositoryAdapter`

```java
@Repository
public class ProductRepositoryAdapter implements ProductRepository {

    @Autowired
    private JpaProductRepository repository;

    @Override
    public List<Product> findByPaginated(int limit, int offset) {
        return repository.findByPaginated(limit, offset)
                .stream()
                .map(this::mapToProduct)
                .toList();
    }


    private Product mapToProduct(ProductEntity entity) {
        return Product.builder()
                .sku(entity.getSku())
                .stock(entity.getStock())
                .brand(entity.getBrand())
                .department(entity.getDepartment())
                .name(entity.getName())
                .price(entity.getPrice())
                .build();
    }

}
```
Nuestra primera aproximación con Streams nos la da el llamado al método `findByPaginated()` el cual devuelve un Array que nos proporciona el método `stream()` el cual transforma nuestro objeto Array a Stream lo primero que hacemos es hacer llamado al método `map()` en el 
cual recibiremos uns Entidad ORM y devolveremos una Entidad de dominio.

## Generación de Stream de productos

### Paginado infinito

Ahora teniendo la base de nuestro proyecto podremos hacer uso de Stream para una proceso de alto rendimiento. Lo que 
haremos será leer nuestra tabla product y obtener su universo completo (1 millón de registros) lo haremos eficientemente 
seleccionando los datos en Arrays de 10.000 registros e iremos transformando la lista obtenida en un stream de datos.
Para leer nuestra tabla lo que haremos es ir leyendo los registros basados en limit y offset para lograr esto crearemos 
un Stream infinito que nos devuelva la siguiente secuencia:

 - limit=10000 , offset=0
 - limit=10000 , offset=10000
 - limit=10000 , offset=20000
 - limit=10000 , offset=30000
 - ...
 - limit=10000, offset= ...N

Nuestro Stream generador será la siguiente clase:
````java
@Getter
@ToString
@AllArgsConstructor
public class StreamPaginated {

    private int limit;
    private int offset;

    public static Stream<StreamPaginated> paginate(int size) {
        return IntStream
                .iterate(0, i -> i + size)
                .map(skip -> new StreamPaginated(size, skip));
    }

}
````

### Definición de nuestro Servicio de Dominio

Este código generará en base a un tamaño dado nuestro stream nos ayudamos de `IntStream` una clase que nos ayuda a generar 
Streams de Integers que generará una secuencia de 10.000 en 10.000. Finalmente llamamos a `map()` para transformar nuestra secuencia de enteros en un objeto `StreamPaginated`.

Ahora creamos nuestro servicio de dominio y utilizamos nuestro `StreamPaginated` para crear una secuencia infinita de este objeto.

```java
@Service
@AllArgsConstructor
public class ProductService {

    @Autowired
    private ProductRepository repository;

    @Transactional
    public Stream<Product> getStream() {
        return StreamPaginated
                .paginate(10000)
                .map(page -> this.repository.findByPaginated(page.getLimit(), page.getOffset()))
                .takeWhile(records -> !records.isEmpty())
                .flatMap(Collection::stream);

    }
}
```

Ahora la lógica de nuestro Stream se divide en lo siguiente:
* `.paginated(10000)`: creamos un paginado de 10.000 en 10.000 de forma infinita para obtener todos los registros de nuestra tabla productos. Con esto nos aseguramos de leer todos los registros.
* `.map(page -> this.repository.findByPaginated(page.getLimit(), page.getOffset())))`: nuestro paginado lo transformaremos en un Array de Product haciendo llamado a nuestro repositorio y su método de búsqueda de productos.
* `.takeWhile(records -> !records.isEmpty())`: Acá está la magia que detendrá nuestra secuencia de paginado, Consumiremos nuestro Stream Mientras los registros que obtengamos no estén vacíos. Es decir cuando venga un Array vacío el Stream terminará.
* `.flatMap(Collection::stream)`: Aplanaremos nuestro Array de Productos de una lista de productos a una secuencia de productos con esto nuestro quién haga llamado del método getStream() obtendrá los registros de 1 en 1 no importando que estemos obteniendo los datos de 10.000 en 10.000.

En resumen el flujo generado realizo las siguientes cambios:

```bash
# StreamPaginated -> List<Product> -> Stream<Product>
```
Nuestro Stream de product está listo para poder ser consumido y realizar las operaciones que queramos.

## Definición de Nuestros casos de uso 

Ahora crearemos el siguiente servicio que representará nuestros casos de uso

```java
@Service
public class ProductUseCases {

    @Autowired
    private ProductService products;

    // ... Your imagination making code

}

```
Crearemos casos de uso para realizar operaciones sobre nuestro Stream

### Obtener las marcas de nuestros productos

```java
public List<String> getBrands() {
        return products.getStream()
                .map(Product::getBrand) // seleccionamos solo el campo brand de nuestro objeto Product 
                .distinct() // de las marcas obtenidas obtneemos los objetos únicos
                .toList(); // transformamos nuestro Stream<String> a List<String>
    }
```
### productos sin stock

```java
public Stream<Product> getSkuWithoutStock() {
        return this.products.getStream()
                .filter(product -> product.getStock() <= 0); // filtraremos todos los productos con stock igual o menor a zero
    }
```
### conteo de productos sin stock
```java
public long countSkuWithoutStock() {
        return this.products.getStream()
                .filter(product -> product.getStock() <= 0) // filtraremos todos los productos con stock igual o menor a zero
                .count(); // de los productos filtrados realizaremos un conteo y devolveremos el valor numerico
    }
```
### Suma de Stock del departamento 604 
```java
public long sumStockDepartment604() {
        return this.products.getStream()
                .filter(product -> product.getDepartment().equals("604")) // filtramos productos del depto 604
                .map(Product::getStock) // mapeamos de Product a Integer
                .reduce(0, Integer::sum) // realizamos una suma con el tradicional reduce()
                .longValue(); // devolvemos el valor como un long
    }
```
### Agrupar por marca productos del departamento 604
```java
public Map<String, List<Product>> groupByDepartment604Brand() {
        return this.products.getStream()
                .filter(product -> product.getDepartment().equals("604")) // filtramos productos del depto 604
                .collect(Collectors.groupingBy(p -> p.getBrand() != null ? p.getBrand() : "No brand")); // agrupamos por marca si el campo marca es nulo le damos el valor por defecto "No brand"
    }
```

Esta es la magia de Stream poder realizar operaciones sobre una gran cantidad de datos de forma eficiente, ya que estamos obteniendo valores
de forma secuencial sin necesidad de obtener el universo completo y después realizar operaciones. Basado en programación 
funcional nos hace más fácil crear código más legible claro que esto es con operaciones más comunes lamentablemente 
hay ocasiones donde el código es más difícil de entender por la nomenclatura de los genéricos de Java, pero ahi solo queda 
dividir y vencer.

Para probar por ti mismo realiza las pruebas en la clase `src/main/java/App.java`:

```java
@Log4j2
@Component
@SpringBootApplication
public class App {

    @Autowired
    private ProductUseCases productUseCases;

    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }

    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        productUseCases
                .getSkuWithoutStock()
                .forEach(product -> log.info("Product {}", product));
    }

}
```

## Conclusiones

No hay conclusiones prueba por ti mismo el verdadero poder de API Stream de Java 




## [Github repository](https://github.com/nullpointer-excelsior/java-high-performance-api.git)


![meme](https://i.ibb.co/HHmdD5Z/efc4edaf-eb8e-4b92-a5fb-2b6283e8e85d.jpg)





