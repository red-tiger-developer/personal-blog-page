---
title: ¿Qué es y cómo funciona el Event Sourcing con un ejemplo práctico?
author: Benjamin
date: 2025-01-28 00:00:00 -0500
categories: Java Architecture Coding Backend Fullstack Microservices
tags: Java Architecture Coding Backend Fullstack Microservices
---

![intro](assets/img/intros/intro10.webp)

## ¿Qué es Event Sourcing?

Event Sourcing es un patrón de arquitectura de software que trata cada cambio en el estado de una aplicación como un evento. En lugar de almacenar el estado actual de un objeto, se almacenan los eventos que han ocurrido, los cuales se pueden reproducir para recrear el estado del objeto en cualquier momento. Esto permite tener un historial completo de las transacciones, facilitando la auditoría y la recuperación ante desastres.

## CRUD vs Event Sourcing
El enfoque tradicional de CRUD (Crear, Leer, Actualizar, Eliminar) se centra en interactuar con el estado actual de una entidad. En contraste, Event Sourcing se enfoca en la secuencia de eventos que han llevado a ese estado. 

### Ventajas y desventajas
**Ventajas:**
- **Auditoría completa:** Permite un seguimiento detallado de todas las acciones.
- **Reproducción del estado:** El estado de una entidad puede ser reconstruido en cualquier momento.
- **Facibilidad en el manejo de cambios:** Cambios en los requisitos son más fáciles de implementar, ya que solo se necesita añadir nuevos eventos.

**Desventajas:**
- **Complejidad:** Puede introducir mayor complejidad en la gestión de estados.
- **Carga de almacenamiento:** Almacenar todos los eventos puede requerir mucho espacio.
- **Performance:** La recuperación del estado puede ser más lenta que un enfoque de estado actual.
- **Consistencia eventual:** Las arquitecturas basadas en eventos deben lidiar con la consistencia eventual, donde los datos pueden no estar inmediatamente sincronizados, pero eventualmente se reflejan en todos los componentes del sistema.

## Conceptos principales de Event Sourcing
- **Events:** Son representaciones de un cambio de estado en el sistema. Cada evento indica una acción que ha tenido lugar, como "Orden Creada" o "Orden Completada".
- **Event Stream:** Es la secuencia de eventos que se han producido para una entidad específica a lo largo del tiempo.
- **Event Store:** Un almacenamiento especializado para guardar eventos, permitiendo su recuperación y consulta.
- **Entities and Aggregates:** Las entidades son objetos en el dominio que contienen datos y lógica de negocio. Los agregados son conjuntos de entidades que se tratan como una unidad para asegurar la coherencia de los cambios.
- **Projections:** Representaciones de datos derivadas de eventos, que permiten consultas eficientes sin necesidad de reconstruir el estado desde cero.
- **Snapshots:** Instantáneas del estado actual de un agregada, que ayudan a optimizar el rendimiento al evitar la necesidad de reproducir cada evento para reconstruir el estado.

Dejemos de lado la teoría y vamos con un ejemplo práctico.

## Ejemplo práctico: Manejo de órdenes de compra con Java
En este ejemplo, vamos a ver cómo implementar event sourcing para manejar el ciclo de vida de órdenes de compra. El código proporcionado abarca la creación, entrega y finalización de órdenes.

### Definición y aplicación de eventos sobre Order
En la clase `Order`, se instancian eventos como `OrderCreatedEvent`, `OrderDeliveredEvent` y `OrderCompletedEvent`, cada uno de ellos describe un cambio en el estado de la orden:

Estructura de eventos:
```java
@Getter
@ToString
@RequiredArgsConstructor
public abstract class Event {
    private final String id;
    private final LocalDateTime createdAt;

    public Event() {
        this(UUID.randomUUID().toString(), LocalDateTime.now());
    }
}

@Getter
@ToString
@AllArgsConstructor
public class OrderCreatedEvent extends Event {
    private String orderId;
    private List<Product> products;
    private Integer total;
    private OrderStatus status;
}

@Getter()
@ToString
@AllArgsConstructor()
public class OrderDeliveredEvent extends Event {
    private String orderId;
    private Shipping shipping;
    private OrderStatus status;
}

@Getter
@ToString
@AllArgsConstructor
public class OrderCompletedEvent extends Event {
    private String orderId;
    private OrderStatus status;
}

```

Estructura y creación de Order:
```java
public static Order create(String orderId, List<Product> products) {
    var order = new Order();
    var total = products.stream()
            .map(Product::getQuantity)
            .reduce(0, Integer::sum);
    var event = new OrderCreatedEvent(orderId, products, total, OrderStatus.CREATED);
    order.apply(event);
    order.events.add(event);
    return order;
}
```

En este código aplicamos la creación de una orden, realizamos todos los cálculos o validaciones que necesitamos, creamos un evento y lo aplicamos, es decir, actualizamos el estado de la entidad.

```java
private void apply(OrderCreatedEvent event) {
    this.id = event.getOrderId();
    this.products = event.getProducts();
    this.status = event.getStatus();
    this.total = event.getTotal();
}
```
De esta forma, nuestra entidad Order tendrá un estado consistente dependiendo de los eventos que se apliquen.

### Casos de uso y lógica de dominio de Order
La lógica de negocio se centraliza en la clase `OrderUseCases`, donde se manejan comandos como crear y completar órdenes:

```java
public void createOrder(CreateOrderCommand command) {
    var order = Order.create(command.orderId(), command.products());
    order.getEvents()
            .forEach(event -> this.orderEventStore.save(event));
    order.cleanEvents();
}
```
Al invocar `Order.create()` realizamos las operaciones necesarias para crear la orden. Después obtendremos los eventos que se han generado internamente en la entidad Order y los guardamos en el `EventStore`.

#### Almacenamientos de eventos
Definimos la interfaz `OrderEventStore`, que se utiliza para almacenar y recuperar eventos:

```java
public interface OrderEventStore {
    void save(Event event);
    Stream<Event> findByOrderId(String orderId);
}
```
No nos preocuparemos de la implementación del event store, ya que va más allá del alcance de este artículo.

### Consulta de un stream de eventos
Cuando se necesita recrear el estado de una orden a partir de sus eventos, se utiliza el método `fromEventStream`:

```java
ublic static Order fromEventStream(Stream<Event> events) {
    var order = new Order();
    events.forEach(event -> {
        switch (event) {
            case OrderCreatedEvent orderCreatedEvent -> order.apply(orderCreatedEvent);
            case OrderDeliveredEvent orderDeliveredEvent -> order.apply(orderDeliveredEvent);
            case OrderCompletedEvent orderCompletedEvent -> order.apply(orderCompletedEvent);
            default -> throw new IllegalStateException("Invalid event found: " + event.getClass().getName());
        }
    });
    return order;
}
```

Este método nos ayuda a recrear una `Order`; solo debemos proporcionar el stream de eventos consultando el `EventStore`:

```java
var eventStream = this.orderEventStore.findByOrderId(command.orderId());
var order = Order.fromEventStream(eventStream);
```

### Guardado de proyecciones

En nuestro caso de uso `completeOrder()` queremos persistir la orden completada y para esto nos ayudaremos de las proyecciones. Esta proyección se guarda al completar una orden. Se utiliza la clase `OrderProjection` para representar el estado actual:

Estructura de la proyección.
```java
public record ProductProjection(String sku, String name, Integer quantity) {}

public record OrderProjection(String id, List<ProductProjection> products, Integer total) { }
```

Caso de uso y guardado de la proyección.
```java
public void completeOrder(CompleteOrderCommand command) {
    var eventStream = this.orderEventStore.findByOrderId(command.orderId());
    var order = Order.fromEventStream(eventStream);
    order.complete();
    order.getEvents()
            .forEach(event -> this.orderEventStore.save(event));
    var projection = new OrderProjection(
            order.getId(),
            order.getProducts().stream()
                    .map(p -> new ProductProjection(p.getSku(), p.getName(), p.getQuantity()))
                    .toList(),
            order.getTotal()
    );
    this.orderProjectionRepository.save(projection);
    order.cleanEvents();
}
```

## Conclusión

Event Sourcing es un poderoso patrón arquitectónico que proporciona una forma efectiva de manejar el estado y la historia de las operaciones en sistemas complejos. Aunque introduce cierta complejidad y consideraciones de rendimiento, sus beneficios en términos de trazabilidad, flexibilidad y consistencia son significativos, especialmente para aplicaciones que requieren un seguimiento detallado de las interacciones y cambios en el sistema.



## [Github repository](https://github.com/nullpointer-excelsior/java-architecture-patterns)

![meme](assets/img/memes/meme8.webp)

