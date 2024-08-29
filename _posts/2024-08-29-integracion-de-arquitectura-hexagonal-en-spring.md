---
title: Integración de arquitectura hexagonal en Spring
author: Benjamin
date: 2024-08-29 00:00:00 -0500
categories: Arquitectura,Arquitectura Software,Spring,Microservicios,Backend
tags: arquitectura,arquitectura software,spring,microservicios,backend
---

![intro](assets/img/intros/intro8.webp)

La arquitectura hexagonal busca separar el dominio de las implementaciones tecnológicas. El modelado de dominio se puede hacer mediante DDD, pero no es obligatorio. Sin embargo, hay patrones muy útiles que pueden ayudarte a crear un código más mantenible.

**RECUERDA:** el diseño y la aplicación de patrones o ciertas arquitecturas dependerán del problema que quieres solucionar y si este aporta valor al objetivo planteado.

## Ejemplo práctico de una arquitectura hexagonal aplicada en Spring Boot

Implementaremos una aplicación base CRUD para un clon de LinkedIn. Este ejemplo es simple, con una sola entidad que representa a un profesional. Si quieres ver el ejemplo más a fondo, revisa el siguiente repositorio [Java Architecture Patterns](https://github.com/nullpointer-excelsior/java-architecture-patterns/tree/master/hexagonal)

## ¿Qué es la arquitectura hexagonal?

La arquitectura hexagonal, también conocida como arquitectura de puertos y adaptadores, permite desacoplar el núcleo de la lógica de negocio de las implementaciones tecnológicas, haciendo que la aplicación sea más fácil de mantener, probar y escalar.

A continuación, se explica la estructura y los componentes principales de esta aplicación de ejemplo:

## Capa de Aplicación
- **ProfessionalUseCases:** Define los casos de uso que el sistema expone. En este caso, hay un método `createProfessional` que permite crear un nuevo profesional. Este método se encarga de coordinar la creación de un nuevo `Professional` en la capa de dominio, guardar ese profesional a través del repositorio y publicar un evento de dominio (`ProfessionalCreatedEvent`) en el bus de eventos.

## Capa de Dominio
### Entidades:

- **Professional:** Es la entidad central en el dominio. Se encarga de representar a un profesional con atributos como `id`, `firstname` y `lastname`. Incluye una validación mediante el uso de Jakarta Bean Validation (`@NotBlank`, `@UUID`) para asegurar que los datos son correctos. La creación de un `Professional` se realiza a través del método estático `create`.

### Eventos de Dominio:

- **ProfessionalCreatedEvent:** Representa un evento que ocurre cuando un nuevo profesional es creado. Este evento se propaga a través del bus de eventos de dominio (`DomainEventBus`).

### Excepciones:

- **DomainException:** Se lanza cuando hay un problema en la validación o en la lógica del dominio.

### Interfaces (Puertos):

- **ProfessionalRepository:** Define las operaciones que deben implementarse para interactuar con la persistencia de datos de `Professional`.
- **DomainEventBus:** Define el contrato para publicar eventos de dominio.

## Capa de Infraestructura

### Persistencia:
- **ProfessionalEntity:** Es la representación de la entidad `Professional` en la base de datos, mapeada usando JPA. Define cómo se almacenan los datos en la tabla `professionals`.
- **JpaProfessionalRepository:** Extiende `JpaRepository` para proporcionar métodos CRUD para `ProfessionalEntity`.
- **PostgresProfessionalRepository:** Implementa el puerto `ProfessionalRepository` usando el repositorio JPA subyacente. Se encarga de mapear los datos entre las entidades de dominio (`Professional`) y las entidades de base de datos (`ProfessionalEntity`).

### Adaptadores:
- **InMemoryDomainEventBus:** Es una implementación del bus de eventos que se utiliza para publicar eventos de dominio, aunque en este caso es una implementación en memoria que podría usarse en pruebas o desarrollo.

### Configuración:

#### CoreConfig: 
Configura los beans de la aplicación, como el caso de uso (`ProfessionalUseCases`) y el bus de eventos (`DomainEventBus`), usando Spring.

## Flujo General:
- **Crear Profesional:** El cliente invoca el caso de uso `createProfessional` pasando los datos necesarios.
- **Validación:** En la capa de dominio, se valida la creación del `Professional`.
- **Persistencia:** El `Professional` creado se guarda en la base de datos a través de `PostgresProfessionalRepository`.
- **Publicación de Evento:** Se publica un evento de dominio (`ProfessionalCreatedEvent`) usando el bus de eventos configurado.

Este diseño modular permite que la aplicación sea flexible y fácilmente extensible, como por ejemplo, cambiar el sistema de almacenamiento o la implementación del bus de eventos sin afectar el núcleo de la lógica de negocio.

## Integración con frameworks

Como hemos dicho antes, este enfoque de arquitectura nos permite separar el dominio y las lógicas de negocio de cualquier implementación externa tecnológica. Esto nos da la ventaja de poder crear aplicaciones más mantenibles y agrega más simplicidad al testing unitario. En este ejemplo, integramos la capa de dominio y la capa de aplicación con Spring, pero perfectamente este enfoque se puede hacer con cualquier otro framework.

### Creando adaptadores

Para poder instanciar los componentes de tipo servicio o repository en una aplicación de Spring con arquitectura hexagonal, debemos definir las implementaciones de los puertos, es decir, los adaptadores. En este caso, tenemos los siguientes puertos:

```java
public interface DomainEventBus {
    void publish(DomainEvent<?> event);
}

public interface ProfessionalRepository {
    void save(Professional p);
    void update(Professional p);
    Stream<Professional> findAll();
}
```

Crearemos las implementaciones:

```java
public class InMemoryDomainEventBus implements DomainEventBus {
    @Override
    public void publish(DomainEvent<?> event) {
        // ... send events code
    }
}
```

Crearemos un repository de Spring y su entidad correspondiente:

```java
@Entity
@Table(name = "professionals")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProfessionalEntity {

    @Id
    @GeneratedValue(generator = "UUID")
    private UUID id;

    @NotBlank
    @Column
    private String firstname;

    @Column
    @NotBlank
    private String lastname;
}

public interface JpaProfessionalRepository extends JpaRepository<ProfessionalEntity, UUID> {
}
```

Y ahora implementamos el puerto `ProfessionalRepository`:

```java
@Repository
@AllArgsConstructor
public class PostgresProfessionalRepository implements ProfessionalRepository {

    private final JpaProfessionalRepository repository;

    @Override
    public void save(Professional p) {
        var entity = ProfessionalEntity.builder()
                .id(UUID.fromString(p.getId()))
                .firstname(p.getFirstname())
                .lastname(p.getLastname())
                .build();
        repository.save(entity);
    }

    @Override
    public void update(Professional p) {
        this.save(p);
    }

    @Override
    public Stream<Professional> findAll() {
        return repository.findAll()
                .stream()
                .map(e -> Professional.builder()
                        .id(e.getId().toString())
                        .firstname(e.getFirstname())
                        .lastname(e.getLastname())
                        .build());
    }
}
```

Debemos hacer hincapié en que la entidad ORM es distinta a una entidad de dominio. Muchos confunden ambos conceptos y los tratan de igual manera, y lo único que logran es acoplar las lógicas de negocio con implementaciones tecnológicas de persistencia. Las entidades ORM representan un mapeo de base de datos y pueden tener comportamientos inesperados si se tratan como lógicas de negocio.

## Inyección de Servicios de Aplicación

En la capa de aplicación nos encontramos con los casos de uso o, propiamente dicho, la entrada y control del programa. Ahora debemos inyectar de manera manual el servicio `ProfessionalUseCases`. Para lograrlo, haremos uso de un componente de tipo `@Configuration` de Spring.

En esta clase inyectamos el puerto `ProfessionalRepository`, el cual es una interfaz. Spring, por debajo, encuentra el componente `PostgresProfessionalRepository` y lo inyectará, ya que esta es la implementación de `ProfessionalRepository`. Ahora solo debemos definir a `ProfessionalUseCases` mediante la anotación `@Bean`, la cual le indica a Spring que el componente puede ser llamado desde otros puntos de la aplicación. También creamos el bean `DomainEventBus`.

```java
@Configuration
@AllArgsConstructor
public class CoreConfig {

    ProfessionalRepository professionalRepository;

    @Bean
    public ProfessionalUseCases getProfessionalUseCase() {
        return new ProfessionalUseCases(professionalRepository, getDomainEventBus());
    }

    @Bean
    public DomainEventBus getDomainEventBus() {
        return new InMemoryDomainEventBus();
    }
}
```

Ahora, para hacer uso de los casos de uso de la aplicación, lo hacemos de manera tradicional con Spring:

```java
@Autowired
private ProfessionalUseCases useCases;
```

Esta es la manera básica de implementar la inyección de dependencias en una arquitectura hexagonal. Puede que nos dé un poco de código extra comparado con la manera tradicional de hacer aplicaciones en Spring, pero obtendrás las siguientes ventajas:

- **Código portable:** La lógica y las reglas de negocio serán portables entre frameworks, librerías e integraciones tecnológicas como bases de datos, integraciones con eventos, etc.
- **Test unitario más simple:** El testing unitario es el más simple de realizar; sin embargo, si tu código no sigue buenas prácticas, será complejo de igual manera. Enfoques de clean architecture te permitirán realizar testing de forma sencilla.
- **Código de dominio se autoexplica:** Si implementas hexagonal o cualquier arquitectura limpia, tenderás a crear código con mejores prácticas. Estas incluirán una definición de nombres y acciones más orientadas a comportamiento que a tecnologías.
- **Mantenibilidad:** Un código que separa el dominio de implementaciones tecnológicas podrá llevar a cabo cambios menos drásticos a los que pueden realizarse cuando ambos conceptos están acoplados.


## Conclusión

La integración de la arquitectura hexagonal en aplicaciones Spring ofrece una sólida separación entre el dominio y las implementaciones tecnológicas, permitiendo un diseño modular y flexible. Aunque requiere un esfuerzo adicional en la configuración inicial, los beneficios a largo plazo en términos de mantenibilidad, portabilidad, y simplicidad en las pruebas unitarias son significativos. Al adoptar esta arquitectura, se fomenta un enfoque más limpio y organizado en el desarrollo de software, facilitando la evolución y escalabilidad de las aplicaciones con un menor costo técnico.

## [Github repository](https://github.com/nullpointer-excelsior/java-architecture-patterns/tree/master/hexagonal)

### Meme de despedida

![meme](assets/img/memes/meme7.webp)