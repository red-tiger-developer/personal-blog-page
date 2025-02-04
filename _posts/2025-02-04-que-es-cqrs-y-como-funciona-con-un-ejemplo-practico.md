---
title: ¿Qué es CQRS y cómo funciona? Con un ejemplo práctico.
author: Benjamin
date: 2025-02-04 00:00:00 -0500
categories: Java Architecture Coding Backend Fullstack
tags: Java Architecture Coding Backend Fullstack
---

![intro](assets/img/intros/intro11.webp)


## ¿Qué es CQRS y qué problema soluciona?

CQRS (Command Query Responsibility Segregation) es un patrón arquitectónico que separa las operaciones de lectura y escritura en una aplicación. En lugar de usar un solo modelo para ambas tareas, CQRS define modelos independientes: uno para procesar comandos (modificaciones de datos) y otro para ejecutar consultas (lectura de datos).

## Escalabilidad de aplicaciones mediante arquitectura
Esta separación mejora el rendimiento al evitar conflictos entre lecturas y escrituras, facilita la escalabilidad ajustando cada parte según sus necesidades y simplifica el mantenimiento en sistemas complejos. Es especialmente útil en aplicaciones con alta concurrencia o grandes volúmenes de datos, donde optimizar cada operación por separado marca la diferencia.

> En el siguiente artículo [Alto rendimiento con CQRS y NestJs](https://nullpointer-excelsior.github.io/posts/alto-rendimiento-con-cqrs-eventos-en-nestjs/) Podrás ver cómo CQRS mejora el rendimiento de una aplicación.

## Ventajas y desventajas

**Ventajas:**

*   **Escalabilidad:** Permite escalar las partes de lectura y escritura de forma independiente, optimizando los recursos.
*   **Simplicidad:** Simplifica el modelo de dominio al separar las responsabilidades de escritura y lectura.
*   **Rendimiento:** Mejora el rendimiento al optimizar las bases de datos para cada tipo de operación.
*   **Flexibilidad:** Facilita la implementación de diferentes modelos de datos para lectura y escritura.

**Desventajas:**

*   **Complejidad:** Introduce complejidad arquitectónica al separar las responsabilidades.
*   **Consistencia eventual:** Puede generar inconsistencia eventual entre los datos de escritura y lectura.
*   **Sobrecarga:** Requiere la implementación de componentes adicionales como buses de comandos y consultas.

## Componentes principales

*   **commands:** Representan intenciones de cambiar el estado del sistema.  Son objetos que contienen información sobre la operación a realizar, pero no el resultado de la operación.
*   **queries:** Representan solicitudes de información del sistema.  Son objetos que especifican qué datos se necesitan, pero no cómo obtenerlos.
*   **command handlers:** Son responsables de ejecutar los comandos.  Reciben un comando, validan la información y modifican el estado del dominio.
*   **query handlers:** Son responsables de ejecutar las consultas.  Reciben una consulta y devuelven los datos solicitados.
*   **command bus:** Es el componente encargado de enrutar los comandos a sus respectivos handlers.
*   **query bus:** Es el componente encargado de enrutar las consultas a sus respectivos handlers.
*   **read model:** Es una representación optimizada de los datos para lectura.  Puede ser una base de datos diferente o una vista materializada.
*   **write model:** Es el modelo de dominio principal donde se realizan las operaciones de escritura.

## Ejemplo práctico con Java

### Sistema de reseñas de productos con CQRS

El ejemplo práctico implementa un sistema de reseñas de productos utilizando el patrón CQRS.  El sistema permite a los usuarios agregar reseñas, comentarios y reacciones a las reseñas, así como consultar las reseñas y comentarios de un producto.

### Descripción del modelo de dominio

El modelo de dominio incluye entidades como `Product`, `Review`, `ReviewComment`, `User` y `ReviewReaction`, que representan los elementos del sistema de reseñas.  También se utilizan *value objects* como `Content` para encapsular el contenido de las reseñas y comentarios.


```java
@Getter
@ToString
@Builder
public class Review {
    private Double score;
    private Product product;
    private Content content;
    private List<ReviewComment> comments;
    private List<ReviewReaction> reactions;

    public void addReaction(ReviewReaction reaction) {
        this.reactions.add(reaction);
    }

    public void addComment(ReviewComment comment) {
        this.comments.add(comment);
    }
}

@Getter
@ToString
@AllArgsConstructor
@Builder
public class ReviewComment {
    private String id;
    private Content content;
    private List<ReviewReaction> reactions;

    public void addReaction(ReviewReaction reaction) {
        this.reactions.add(reaction);
    }
}

@Getter
@ToString
@AllArgsConstructor
public class Product {
    private String sku;
    private String name;
}

@Getter
@ToString
@AllArgsConstructor
public class User {
    private String id;
    private String fullname;
    private String email;
}

public enum ReactionType {
    LIKE,DISLIKE,HELPFUL
}

@Getter
@ToString
@AllArgsConstructor
public class ReviewReaction {
    private User user;
    private ReactionType type;
}

@Getter
@ToString
@AllArgsConstructor
public class Content {
    private User user;
    private String content;
}
```

### Casos de uso

Los casos de uso incluyen:

*   Agregar una reseña a un producto.
*   Agregar un comentario a una reseña.
*   Agregar una reacción a una reseña o comentario.
*   Obtener las reseñas de un producto.
*   Obtener los comentarios de una reseña.


```java
@AllArgsConstructor
public class ReviewUseCases {

    private ReviewWriteRepository reviewWriteRepository;
    private UserReadRepository userReadRepository;
    private ProductReadRepository productReadRepository;
    private ReviewReadRepository reviewReadRepository;
    private EventBus eventBus;

    public void addReview(AddReviewCommand command) {
        var user = this.userReadRepository.findById(command.userId())
                .orElseThrow(() -> new NoSuchElementException("User not found"));
        var product = this.productReadRepository.findBySku(command.sku())
                .orElseThrow(() -> new NoSuchElementException("Product not found"));
        var review = Review.builder()
                .score(command.score())
                .content(new Content(user, command.content()))
                .product(product)
                .build();
        this.reviewWriteRepository.save(review);
        var event = new ReviewCreatedEvent(UUID.randomUUID().toString(), LocalDateTime.now(), review);
        this.eventBus.dispatch(event);
    }

    public void addReviewReaction(AddReviewReactionCommand command) {
        var user = this.userReadRepository.findById(command.userId())
                .orElseThrow(() -> new NoSuchElementException("User not found"));
        var review = this.reviewReadRepository.findById(command.reviewId())
                .orElseThrow(() -> new NoSuchElementException("Review not found"));
        var reaction = new ReviewReaction(user, command.reaction());
        review.addReaction(reaction);
        this.reviewWriteRepository.save(review);
        var event = new ReviewUpdatedEvent(UUID.randomUUID().toString(), LocalDateTime.now(), review);
        this.eventBus.dispatch(event);
    }

    public void addReviewComment(AddReviewCommentCommand command) {
        var user = this.userReadRepository.findById(command.userId())
                .orElseThrow(() -> new NoSuchElementException("User not found"));
        var review = this.reviewReadRepository.findById(command.reviewId())
                .orElseThrow(() -> new NoSuchElementException("Review not found"));
        var comment = ReviewComment.builder()
                .id(UUID.randomUUID().toString())
                .content(new Content(user, command.content()))
                .build();
        review.addComment(comment);
        this.reviewWriteRepository.save(review);
        var event = new ReviewUpdatedEvent(UUID.randomUUID().toString(), LocalDateTime.now(), review);
        this.eventBus.dispatch(event);
    }

    public List<ReviewResult> getReviewsByProduct(GetReviewsByProductQuery query) {
        return this.reviewReadRepository.findByProductSku(query.sku())
                .stream()
                .map(ReviewResult::map)
                .toList();
    }

}

@AllArgsConstructor
public class ReviewCommentUseCases {

    private UserReadRepository userReadRepository;
    private ReviewCommentReadRepository reviewCommentReadRepository;
    private ReviewCommentWriteRepository reviewCommentWriteRepository;
    private EventBus eventBus;

    public void addReviewCommentReaction(AddReviewCommentReactionCommand command) {
        var user = this.userReadRepository.findById(command.userId())
                .orElseThrow(() -> new NoSuchElementException("User not found"));
        var reviewComment = this.reviewCommentReadRepository.findById(command.reviewCommentId())
                .orElseThrow(() -> new NoSuchElementException("Review comment not found"));
        var reaction = new ReviewReaction(user, command.reaction());
        reviewComment.addReaction(reaction);
        this.reviewCommentWriteRepository.save(reviewComment);
        var event = new ReviewCommentCreatedEvent(UUID.randomUUID().toString(), LocalDateTime.now(), reviewComment);
        this.eventBus.dispatch(event);
    }

    public List<ReviewCommentResult> getReviewComments(GetReviewCommentsQuery query) {
        return this.reviewCommentReadRepository.findByReviewId(query.reviewId())
                .stream()
                .map(ReviewCommentResult::map)
                .toList();
    }
}

```



### Repositorios Read Model and Write Model

Se utilizan repositorios separados para lectura (`ReviewReadRepository`, `ReviewCommentReadRepository`, etc.) y escritura (`ReviewWriteRepository`, `ReviewCommentWriteRepository`, etc.).  Esto permite optimizar cada repositorio para su respectiva función.

```java

public interface ReviewReadRepository {
    Optional<Review> findById(String id);
    List<Review> findByProductSku(String sku);
}

public interface ReviewWriteRepository {
    void save(Review review);
}

public interface ReviewCommentWriteRepository {
    void save(ReviewComment comment);
}

public interface ReviewCommentReadRepository {
    Optional<ReviewComment> findById(String id);
    List<ReviewComment> findByReviewId(String id);
}

public interface UserReadRepository {
    Optional<User> findById(String id);
}

public interface ProductReadRepository {
    Optional<Product> findBySku(String sku);
}

```

> La implementación técnica puede incluir distintas tecnologías de almacenamiento optimizadas para lectura o escritura, dependiendo del caso; pero esto va más allá de este artículo.

### Ejecución de comandos

Los comandos se ejecutan a través del `CommandBus`. Por ejemplo, para agregar una reseña, se crea un objeto `AddReviewCommand` y se envía al `CommandBus`. El `CommandBus` enruta el comando al `AddReviewCommandHandler`, que ejecuta la lógica de negocio y modifica el modelo de dominio.

```java
// Command
@Builder
public record AddReviewCommand (String sku, Double score, String userId, String content) implements Command {}

// Ejemplo de ejecución de un comando
AddReviewCommand command = AddReviewCommand.builder()
    .sku("SKU123")
    .score(4.5)
    .userId("user123")
    .content("Buena reseña")
    .build();
commandBus.dispatch(command);

// CommandBus
public interface CommandBus {
    void dispatch(Command command);
}

// CommandHandler
public interface CommandHandler<T extends Command> {
    void onCommand(T command);
}

// Implementación de CommandBus
@Builder
public class ConcurrentCommandBus implements CommandBus {

    private final AddReviewCommandHandler addReviewCommandHandler;
    private final AddReviewCommentCommandHandler addReviewCommentCommandHandler;
    private final AddReviewReactionCommandHandler addReviewReactionCommandHandler;
    private final AddReviewCommentReactionCommandHandler addReviewCommentReactionCommandHandler;

    @Override
    public void dispatch(Command command) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            switch (command) {
                case AddReviewCommand addReviewCommand -> executor.submit(
                        () -> addReviewCommandHandler.onCommand(addReviewCommand));
                case AddReviewCommentCommand addReviewCommentCommand -> executor.submit(
                        () -> addReviewCommentCommandHandler.onCommand(addReviewCommentCommand));
                case AddReviewReactionCommand addReviewReactionCommand -> executor.submit(
                        () -> addReviewReactionCommandHandler.onCommand(addReviewReactionCommand));
                case AddReviewCommentReactionCommand addReviewCommentReactionCommand -> executor.submit(
                        () -> addReviewCommentReactionCommandHandler.onCommand(addReviewCommentReactionCommand));
                default -> throw new IllegalArgumentException("Unknown command: " + command.getClass().getSimpleName());
            }
        }
    }
}

// Implementación CommandHandler
@AllArgsConstructor
public class AddReviewCommandHandler implements CommandHandler<AddReviewCommand> {
    private final ReviewUseCases review;
    @Override
    public void onCommand(AddReviewCommand command) {
        this.review.addReview(command);
    }
}
```

### Ejecución de queries

Las consultas se ejecutan a través del QueryBus.  Para obtener las reseñas de un producto, se crea un objeto GetReviewsByProductQuery y se envía al QueryBus.  El QueryBus enruta la consulta al GetReviewsByProductQueryHandler, que consulta el Read Model y devuelve los resultados.

```java
// Query
public record GetReviewsByProductQuery(String sku) implements Query {}

// Ejemplo de ejecución de una consulta
GetReviewsByProductQuery query = new GetReviewsByProductQuery("SKU123");
CompletableFuture<List<ReviewResult>> future = queryBus.query(query, ReviewResult.class);
List<ReviewResult> reviews = future.join();

// QueryBus
public interface QueryBus {
    <R extends Result> CompletableFuture<List<R>> query(Query query, Class<R> queryType);
}

// QueryHandler
public interface QueryHandler<Q extends Query, R> {
    List<R> onQuery(Q query);
}

// Implementación QueryBus
@Builder
public class ConcurrentQueryBus implements QueryBus {

    private final GetReviewCommentsQueryHandler getReviewCommentsQueryHandler;
    private final GetReviewsByProductQueryHandler getReviewsByProductQueryHandler;

    @Override
    @SuppressWarnings("unchecked")
    public <R extends Result> CompletableFuture<List<R>> query(Query query, Class<R> queryType) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            return switch (query) {
                case GetReviewCommentsQuery getReviewCommentsQuery -> CompletableFuture.supplyAsync(
                        () -> (List<R>) getReviewCommentsQueryHandler.onQuery(getReviewCommentsQuery), executor);
                case GetReviewsByProductQuery getReviewsByProductQuery -> CompletableFuture.supplyAsync(
                        () -> (List<R>) getReviewsByProductQueryHandler.onQuery(getReviewsByProductQuery), executor);
                default -> throw new IllegalArgumentException("Unknown query:" + query.getClass().getSimpleName());
            };
        }
    }
}

//Implementación QueryHandler
@AllArgsConstructor
public class GetReviewsByProductQueryHandler implements QueryHandler<GetReviewsByProductQuery, ReviewResult> {
    private final ReviewUseCases reviewUseCases;
    @Override
    public List<ReviewResult> onQuery(GetReviewsByProductQuery query) {
        return this.reviewUseCases.getReviewsByProduct(query);
    }
}
```

## Sincronización del read model con el write model.

La separación del modelo de datos en uno de lectura y otro de escritura conlleva considerar que nuestro sistema tendrá que lidiar con la consistencia eventual y estos 2 modelos deben sincronizarse, por lo que tenemos las siguientes opciones:

- **CDC (change capture data) solucion de infraestructura:** Una forma de sincronizar el Read Model con el Write Model es utilizando Change Data Capture (CDC).  CDC captura los cambios en el Write Model (generalmente una base de datos) y los propaga al Read Model.  Esto se puede implementar con herramientas como Debezium, Apache Kafka Connect, Etc.
- **Réplicas de lectura de AWS:** AWS ofrece la posibilidad de crear réplicas de lectura de las bases de datos. Estas réplicas se pueden utilizar como Read Model, ya que replican los cambios del Write Model de forma asíncrona. Esta opción es ideal para evitar preocuparnos de la infraestructura, solo debemos considerar los costos asociados.
- **Vistas materializadas:** Una vista materializada es una tabla que almacena el resultado de una consulta predefinida y se actualiza periódicamente para reflejar los cambios en los datos subyacentes. Esta técnica permite mejorar el rendimiento de las consultas de lectura sin afectar la base de datos principal. Este enfoque es utilizado cuando el Write Model y el Read Model están en el mismo motor de base de datos.
- **Eventos de integración:** Otra forma es utilizando eventos de integración.  Cuando se realiza un cambio en el Write Model, se genera un evento que se publica en un bus de eventos (EventBus).  Los componentes que mantienen el Read Model se suscriben a estos eventos y actualizan el Read Model en consecuencia.  Este es el enfoque utilizado en el ejemplo de código.  Cuando se crea una nueva review, se publica un evento ReviewCreatedEvent que es consumido por los event handlers que actualizan el Read Model.

En el siguiente ejemplo, podemos ver la sincronización mediante eventos de integración. Estos pueden ser consumidos por componentes de la misma aplicación u otra aplicación que actúa como un worker o job.
```java
// Event
public record ReviewCreatedEvent(String eventId, LocalDateTime createdAt, Review payload) implements Event {

}
// Ejemplo de publicación de un evento
ReviewCreatedEvent event = new ReviewCreatedEvent(UUID.randomUUID().toString(), LocalDateTime.now(), review);
eventBus.dispatch(event);

// Ejemplo de un event handler
public class ReviewCreatedEventHandler implements EventHandler<ReviewCreatedEvent> {
    @Override
    public void onEvent(ReviewCreatedEvent event) {
        System.out.println("saving Review on read model:" + event.payload());
    }
}
```

## Conclusión

La separación de modelos de lectura y escritura en CQRS permite mejorar significativamente la escalabilidad y la organización del código. Al dividir estas responsabilidades, el sistema puede optimizar consultas y comandos de manera independiente, evitando bloqueos en la base de datos y permitiendo ajustar los recursos según las necesidades de cada operación. Esto resulta especialmente útil en aplicaciones con alta concurrencia o grandes volúmenes de datos, donde mantener un solo modelo puede generar cuellos de botella.

Sin embargo, esta separación introduce el desafío de sincronizar el modelo de lectura con el de escritura, asegurando que los datos reflejen cambios recientes sin comprometer el rendimiento. Estrategias como CDC, réplicas de lectura y eventos de integración permiten mantener la consistencia eventual, cada una con sus propias ventajas y consideraciones. Elegir el enfoque adecuado dependerá de los requisitos del sistema y su infraestructura, garantizando un balance entre rendimiento, consistencia y costos operativos.

## [Github repository](https://github.com/nullpointer-excelsior/java-architecture-patterns/tree/master/app/src/main/java/com/benjamin/cqrs)

![meme](assets/img/memes/meme9.webp)

