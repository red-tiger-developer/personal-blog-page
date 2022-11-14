---
title: Arquitectura hexagonal Parte III Modelando el Dominio a Fondo con Domain Driven Design
author: Benjamin
date: 2022-11-07 10:32:00 -0500
categories: [Programacion, Nestjs, Arquitectura de software, Typescript]
tags: [typescript, nestjs, hexagonal]
---

![image](https://i.ibb.co/v1HNZKR/Screen-Shot-2022-11-14-at-10-19-55.png)

En post anteriores vimos como implementar una arquitectura hexagonal y aprendimos sus principales conceptos y componentes. En esta oportunidad modelaremos en profundidad el Dominio de nuestra aplicación aplicando Domain Driven Design. 

Arquitectura Hexagonal y Domain Driven Design son clean architecture. Hexagonal y DDD se complementan muy bien pero son cosas distintas mientras hexagonal hace enfasis en separar mediante adaptadores y puertos el dominio y los casos de uso de la infraestructura DDD se centra en el diseño de la capa de dominio. En resumen básicamente la diferencia es que hexagonal se enfoca más en los adaptadores y puertos para desacoplar nuestro modelo de las tecnologías empleadas mientras que DDD es el diseño orientado al dominio de negocio y los casos de uso después de este breve resumen vamos al maldito código

## Creando un microservicio para registrar usuarios

La aplicación que crearemos será un servicio para administrar usuarios iremos iterando sobre algunos caso de uso simples e implementaremos un modelado de dominio a fondo utilizando Domain Driven Design empezaremos por el siguiente caso de uso:

`Creación de usuarios`
* Se necesita un servicio que cree usuarios donde el username debe tener el siguiente formato: la primera letra del nombre seguido de un punto y de su apellido, en caso de que exista un usuario con esta nomenclatura se debe agregar un número correlacional ejemplo : para el empleado john wick su usuario sería `j.wick00` y jordan wick sería `j.wick01` al crear el usuario debemos notificarlo mediante un email


Muy bien creamos nuestra app Nestjs y sus módulos

```bash
nest new user-micro-service
cd user-micro-service
nest generate module core
nest generate module infraestructure

```

Ahora generamos la siguiente estructura básica de de directorios

```bash
──  core
│   ├──  application
│   │   ├──  ports
│   │   └──  services
│   ├──  core.module.ts
│   ├──  domain
│   │   ├──  model
│   │   └──  ports
│   │   │   ├──  inbound
│   │   │   └──  outbound
│   │   ├──  services
│   └──  shared
│       ├──  dto
│       └──  error
├──  infraestructure
│   ├──  adapters
│   └──  infraestructure.module.ts

```
Con esta estructura empezaremos a aplicar DDD como locos 

## 1 - Definiendo Entidades 
Crearemos una clase abstract base para nuestras entidades

```typescript
export abstract class Entity<T>{

    id: Id;

    abstract equalsTo(entity: T): boolean;

}
```
Esta clase define el método abstracto equalsTo() para que sus clases hijas tengan que implementar la lógica de igualdad en entidades. Entonces la forma básica de definir nuestras entidades sería la siguiente:

```typescript
export class User<User> {
    
    username: string;
    email: string;
    confirmed: boolean;

    equalsTo(entity: User): boolean {
        return this.id.getValue() === entity.id.getValue()
    }

}
```
Definimos valores primitivos y los asignamos nada del otro mundo, como definimos nuestras clases en la mayoría de los proyectos que hemos trabajado, pero esto tiene un detalle los valores primitivos pueden representar cosas totalmente alejadas a nuestro dominio, pueden romper reglas de negocio incluso la integridad de los datos almacenados si no existen restricciones en el motor de base de datos escogido veamos este ejemplo:

```typescript
const user = new User()
user.email = 'john.carter' // this in not a email
user.username = 'a john le gusta la c...' // XD
```
Esto no representa nuestra entidad, esto no es correcto generalmente en enfoques tradicionales agregamos validaciones a nivel de peticiones HTTP o restricciones de base de datos, o podemos delegar esta responsabilidad a servicios de dominio u otra lógica de negocio.

Pero somos ingenieros de software y vamos por una solución mas cercana al dominio.

## 2 - Value Objects en el Dominio

Vamos a redefinir nuestra Entidad reemplazando nuestras variables primitivas por unos nuevos objetos llamados `ValueObjects` estos objetos son una especie de envoltura sobre valores primitivos para poder representar uno o más valores relacionados con nuestro modelo de dominio estos son objetos inmutables una vez instanciados no debemos cambiar su valor. La principal diferencia entre una entidad y un Value Object es que una entidad posee identidad es decir que contiene un valor como una id que lo identifica como único en nuestra aplicación en cambio el value object para compararlo debes comparar todas sus propiedades.

Ahora crearemos el siguiente value object

```typescript
export class Email {
    
    constructor(private email: string) { }

    getValue() {
        return this.email
    }
    
}
```
Este nuevo objeto solo es una envoltura de un valor primitivo ¿Pero de qué nos sirve crear esto que beneficios nos trae? Parece ser código de sobra. Bien si nuestro value object no sirve para nada hasta ahora. Pero como esta clase representa un Email vamos a agregar la siguiente lógica de validación:

```typescript
export class Email {
    
    constructor(private email: string) { 
        if (!this.validate(email)) {
            throw new EmailValidException(`Email value ${email} is not valid`)
        }
    }

    validate(email: string) {
        const res = /^(([^<>()\[\]\\.,;:\s@"]+(\.[^<>()\[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
        return res.test(String(email).toLowerCase());
    }

    getValue() {
        return this.email
    }
    
}
```
Ahora nuestro `ValueObject` tiene sentido entonces para ahorrar esfuerzo en crear más `ValueObject` crearemos una clase base abstracta:


```typescript
export abstract class ValueObjectBase<T> {

    protected abstract validate(value: T): boolean;

    constructor(private primitiveValue: T, errorMessage: string) {
        if (!this.validate(primitiveValue)) throw new DomainException(errorMessage)
    }

    getValue() {
        return this.primitiveValue
    }

}
```
> Aplicamos `Template method` para reutilizar lógica y extender su funcionalidad mediante sus hijos.


 y definiremos nuevos value objects de la siguiente manera:


```typescript
// Email
export class Email extends ValueObject<string> {
    
    constructor(email: string) { 
        super(email, `Invalid Email Address: ${email}`) 
    }

    validate(email: string) {
        const res = /^(([^<>()\[\]\\.,;:\s@"]+(\.[^<>()\[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
        return res.test(String(email).toLowerCase());
    }
    
}
// Id 
export class Id extends ValueObject<string> {
    
    constructor(private id: string) {
        super(id, `Invalid UUID Id:${id} `)
    }

    validate(id: string) {
        const re  =  /^[0-9a-f]{8}-[0-9a-f]{4}-[0-5][0-9a-f]{3}-[089ab][0-9a-f]{3}-[0-9a-f]{12}$/i
        return re.test(id)
    }

    static generate() {
        return new Id(uuidv4())
    }
    
}

```
Finalmente nuestro `Username ValueObject` tendra la lógica asociada a nuestro caso de uso del formato de nombre de usuario.

```typescript
export class Username extends ValueObject<string>{

    constructor(username: string) {
        super(username, `Username ${username} doesn't follow Northiwind policies`)
    }

    /**
     * Username must be first letter of name followed by dot, lastname and number Ex: "a.smith0" or "a.smith1"
     * @param username 
     */
    validate(username: string) {
        if (!username.includes('.')) {
            return false
        }
        const split = username.split('.')
        if (split[0].length !== 1) {
            return false
        }
        return true
    }

    static createUsernameBase(firstname: string, lastname: string) {
        return `${firstname[0]}.${lastname}`
    }

    static create(firstname: string, lastname: string) {
        return new Username(`${this.createUsernameBase(firstname, lastname)}00`)
    }

    static createWithIdentity(firstname: string, lastname: string, identity: number) {
        const userNumber = String(identity).padStart(2, '0')
        return new Username(`${this.createUsernameBase(firstname, lastname)}${userNumber}`)
    }

}
```
Ahora el concepto de value object toma más sentido y nos da una manera de crear reglas de negocio o validaciones de propiedades más cerca de nuestro dominio. En el desarrollo clásico estas validaciones las delegamos en librerías de terceros de validaciones o a servicios de negocio, Pero no siempre validamos un email o un largo de un texto tenemos que tener en cuenta que podemos encontrar reglas de negocio que no existirán en librerías de terceros.

Nuestra entidad queda así:
```typescript
export class User extends Entity {

    username: Username;
    email: Email;
    confirmed: boolean;

}
```
Y el siguiente código nos arrojaría una excepción de dominio
```typescript
const user = new user()
user.email = new Email('juan.lorca.gmail') // throw DomainException() Invalid email
```
Con este enfoque controlamos la creación de objetos con mas coherencia pero también tenemos una desventaja, Con entidades grandes la instanciación de nuestros objetos se vuelve tediosa y con excesivo código en muchos casos. Para solucionar esto podemos aplicar el patrón de diseño Builder:

```typescript

export class UserBuilder {

    private user: User = new User()

    id(id: Id | string) {
        this.user.id = typeof id === 'string' ? new Id(id) : id
        return this
    }

    username(username: Username | string) {
        this.user.username = typeof username  === 'string' ? new Username(username) : username
        return this
    }

    email(email: Email | string) {
        this.user.email = typeof email === 'string' ? new Email(email) : email
        return this
    }

    confirmed(confirmed: boolean) {
        this.user.confirmed = confirmed
        return this
    }

    build() {
        return this.user
    }

}

```
> El pátron `builder` nos provee una manera de crear objetos complejos desde un objeto base y puede ir componiendose de más partes a medida que llamamos operaciones secuenciales para construir nuestro objeto final

El funcionamiento de este patrón es simple:

```typescript
// creamos instancia de UserBuilder 
const builder = new UserBuilder()
// internamente se instancia un objeto User
private user: User = new User()
// definimos metodos que reciben propiedades y las seteamos en nuestra instancia de User y devolvemos la instancia de UserBuilder con "this". De esta manera podemos encadenar los métodos de creación o devolver el mismo objeto builder para usarlo en futuras operaciones
email(email: Email | string) {
    this.user.email = typeof email === 'string' ? new Email(email) : email
    return this
}
// Cuando ya setemaos las variables deseadas de nuestro User hacemos un llamado a l metodo build() y este devolvera el objeto User
build() {
    return this.user
}
```
Lo siguiente será definir la lógica de creación o inicialización de usuarios a nuestra entidad ya definido nuestro builder lo declararemos como una clase interna de User y a su vez declararemos el constructor de user como privado para que solo User pueda instanciarse mediante las operaciones relacionadas con el dominio.

```typescript
class User {
    private constructor() {}
    // values
    private static UserBuilder = class  {
        // code...
    }
}
```

tambien definiremos 

Ahora empleamos nuestro builder de la siguiente manera en la entidad User agregando el método estático `create()` nuestra entidad quedaría así.

```typescript
export class User extends Entity {

    username: Username;
    email: Email;
    confirmed: boolean;

    static create(user: CreateNewUserDto): User {

        const { username, email } = user
        const id = Id.generate()

        return new this.UserBuilder()
            .id(id)
            .username(username)
            .email(email)
            .confirmed(false)
            .build()

    }

    equalsTo(e: User): boolean {
        return this.id.getValue() === e.id.getValue()
    }

    private static UserBuilder = class  {

        private user = new User()

        constructor() {}
    
        id(id: Id | string) {
            this.user.id = typeof id === 'string' ? new Id(id) : id
            return this
        }
    
        username(username: Username | string) {
            this.user.username = typeof username  === 'string' ? new Username(username) : username
            return this
        }
    
        email(email: Email | string) {
            this.user.email = typeof email === 'string' ? new Email(email) : email
            return this
        }
    
        confirmed(confirmed: boolean) {
            this.user.confirmed = confirmed
            return this
        }
    
        build() {
            return this.user
        }
    
    }
}
```

Muy bien ya aplicamos una solución al excesivo código boiler plate que nos puede generar entidades con un montón de `ValueObject`

## 3 - Definición de puertos y adaptadores hexagonal y Domain Driven Design Juntos

resumiendo de nuestro primer post [Arquitectura hexagonal Parte I ](/posts/implementando-hexagonal-con-nestjs-part1/) Los puertos son las entradas y salidas que nuestro dominio tiene para comunicarse con el resto de la aplicación es decir con infraestructura, tenemos 2 tipos de puertos:
* `inbound (entrada):` representan los cambios de estado que queremos realizar en nuestro dominio como puede ser la actualización de el nombre de un usuario
* `outbound (sálida):` Representan los cambios que nuestro dominio quiere realizar fuera de él. Por ejemplo actualizar una base de datos

Ahora definimos nuestros puertos casos de uso y redefinimos nuestros componentes a compartir en el directorio shared

```bash
├──  core
│   ├──  application
│   │   ├──  services
│   │   |   ├──  CreateUserService.spec.ts
│   │   |   └──  CreateUserService.ts
|   |   └──  CreateUser.ts
│   ├──  core.module.ts
│   ├──  domain
│   │   ├──  model
│   │   │   ├──  entities
│   │   │   │   └──  User.ts
│   │   │   └──  valueobjects
│   │   │       ├──  Email.spec.ts
│   │   │       ├──  Email.ts
│   │   │       ├──  Username.spec.ts
│   │   │       └──  Username.ts
│   │   ├──  ports
│   │   │   ├──  inbound
│   │   │   │   └──  UniqueUsernameGenerator.ts
│   │   │   └──  outbound
│   │   │       └──  UserRepository.ts
│   │   └──  services
│   │       ├──  UniqueUsernameGeneratorService.spec.ts
│   │       └──  UniqueUsernameGeneratorService.ts
│   └──  shared
│       ├──  application
│       │   └──  ports
│       │       └──  outbound
│       │           └──  email-service
│       │               ├──  EmailMessage.ts
│       │               └──  EmailService.ts
│       ├──  domain
│       │   ├──  Entity.ts
│       │   ├──  ValueObject.ts
│       │   └──  valueobjects
│       │       ├──  Id.spec.ts
│       │       └──  Id.ts
│       ├──  dto
│       │   ├──  CreateUserDto.ts
│       │   └──  RegisterEmailNotificationDto.ts
│       └──  error
│           └──  DomainException.ts
```

Definición de puertos de dominio

```typescript

export interface UniqueUsernameGenerator {
    generate(firstname: string, lastname: string): Promise<Username>
}

export interface UserRepository {
    save(user: User): Promise<void>;
    findByUsernameStartWith(username: string): Promise<User[]>;
}
```

Definición de puertos de aplicación

```typescript
// Use case
export interface CreateUser {
    create(createUser: CreateUserDto): Promise<User>;
}

```

Definición de adaptadores de dominio:

En nuestro caso el puerto UserRepository será solo una implementación en memoria para simplificar el ejemplo

```typescript

export class InMemoryUserRepository implements UserRepository {

    constructor(private users: User[]) { }
    
    async save(user: User): Promise<void> {
       this.users.push(user)
    }
    
    findByUsernameStartWith(username: string): Promise<User[]> {
        const results = this.users.filter(user => user.username.getValue().startsWith(username))
        return Promise.resolve(results)
    }

}

```

Definición de servicios de dominio:

`UniqueUsernameGeneratorService` se encarga de generar el nombre de usuario de acuerdo a las políticas de nuestro fantasioso caso de uso.
```typescript
export class UniqueUsernameGeneratorService implements UniqueUsernameGenerator {
   
    constructor(private repository: UserRepository) { }

    async generate(firstname: string, lastname: string): Promise<Username> {
        
        const usernamebase = Username.createUsernameBase(firstname, lastname)
        const users = await this.repository.findByUsernameStartWith(usernamebase)

        if (users.length > 1) {
            const identity = users.length + 1
            return Username.createWithIdentity(firstname, lastname, identity)
        }
        
        return Username.create(firstname, lastname)
        
    }

}

```
## 4 - Servicios de aplicación implementación de caso de uso

 En la implementación de nuestro caso de uso hacemos inyección de los puertos `UserRepository` y el servicio de dominio `UniqueUsernameGenerator`
```typescript

export class CreateUserService implements CreateUser {
    
    constructor(
        private repository: UserRepository,
        private uniqueUsernameGenerator: UniqueUsernameGenerator,
    ) { }
    
    async create(createuser: CreateUserDto): Promise<void> {
        
        const username = await this.uniqueUsernameGenerator.generate(createuser.firstname, createuser.lastname)
        
        const user = User.create({
            username: username,
            email: createuser.email
        })

        return this.repository.save(user)
        
    }

}

```

Para completar nuestro caso de uso debemos implementar el envío del correo electrónico notificando la creación de usuario, pero en esta ocasión implementaremos un nuevo concepto de Domain Driven Design. Este caso de uso puede ser divido en 2:
* Crear usuario
* Notificar usuario por correo

En un enfoque tradicional implementaríamos un servicio y al crear el usuario invocaríamos este componente mediante la coordinación de nuestro servicio de aplicación en ciertos escenarios esto nos bastaría, pero como nuestro sistema es algo mas ligado a un proceso de negocio es mejor pensar en un enfoque más desacoplado. Para lograrlo implementaremos Eventos de Dominio.

## 5 - Eventos de Dominio

Como su nombre lo indica estos son sucesos que han ocurrido en el dominio se describen en verbo pasado como `UserCreated`, `UserUpdated`. Esto nos permite notificar a otros componentes de nuestra aplicación los cambios que han ocurrido en el dominio de forma explícita. Los eventos de dominio nos permite declarar tanto el evento como el subscriptor o manejador del evento de esta manera podemos lograr una aplicación más desacoplada al implementar casos de usos más específicos y con responsabilidad más acotada.

Para implementar los eventos de dominio debemos crear un componente llamado `EventBus` esto es quien coordinara el envío del evento y la distribución de este a los componentes que necesitan realizar una acción determinada. Para crear este `EventBus` nos basaremos en el patrón de diseño `Observer`

> El Patrón de diseño observer nos ofrece la posibilidad de definir una dependencia uno a uno entre dos o más objetos para transmitir todos los cambios de un objeto concreto de la forma más sencilla y rápida posible.

Crearemos una clase abstracta llamada `EventBase` y el tipo `EventName`. Esta clase declara el método `getName()` que identificara el tipo de evento a su vez declaramos una segunda clase abstracta llamada DomainEvent haciendo uso de genericos para que el método `getData()` devuelva de forma tipada los datos que queremos enviar al `EventBus`.

 ```typescript
export type EventName = string;

export abstract class EventBase {

    constructor(readonly eventId: string, readonly ocurredOn: Date) { }

    abstract getName(): EventName;

}

export abstract class DomainEvent<T> extends EventBase {

    constructor(private readonly data: T) {
        super(Id.string(), new Date())
    }

    getData(): T {
        return this.data
    }

}

```

Ya definido nuestra estructura de eventos debemos hacer cambios en nuestra clase `Entity`. En domain driven design la idea es que nuestras lógicas de negocio este lo mas cerca de las entidades de nuestro dominio es por eso que `User` almacenará los eventos relacionados a él.

Declararemos un arreglo vacío de EventBase donde iremos almacenando los eventos que pueda producir nuestra entidad `User`. Acá nuestra clase base nos ayuda a ignorar problemas de tipado cuando tengamos distintos eventos con tipo de data distintos entre sí.
Definimos también los siguientes métodos:
* `record():` registra un nuevo evento
* `pullevents():` devolverá todos los eventos y los eliminará de nuestra entidad.

```typescript
export abstract class Entity<T>{

    id: Id;
    private events: EventBase[] = []

    abstract equalsTo(entity: T): boolean;

    record(event: EventBase) {
        this.events.push(event)
    }

    pullEvents() {
        const domainEvents = this.events.slice();
        this.events = [];
        return domainEvents;
    }

}
```
Ahora debemos generar el evento en el método `create()` de nuestra entidad `User`.

```typescript
export class User extends Entity<User> {

    // Entity props

    static create(user: CreateNewUserDto): User {

        const { username, email } = user
        const id = Id.generate()

        const userCreated = new this.UserBuilder()
            .id(id)
            .username(username)
            .email(email)
            .confirmed(false)
            .build()
        // generate a domain event
        userCreated.record(
            new UserCreated(userCreated)
        )

        return userCreated

    }
    // More code...

}
```
Finalizamos los cambios sobre nuestra entidad `User` y declararemos nuestros componentes subscriptores. Creamos una interfaz base `EventSubscriber` con los siguientes métodos:  
* `subscribeTo():` Indicaremos a que evento queremos subscribirnos
* `onEvent():` Acá implementaremos la lógica que queremos realizar cuando llegue un evento de dominio.

```typescript
export interface EventSubscriber {
    suscribeTo(): EventName;
    onEvent(event: EventBase): void;
}
```
Definimos la siguiente interfaz hija de `EventSubscriber` esta interfaz sobreescribe el método `onEvent()` con el parámetro event de tipo `DomainEvent` el cual nos permite crear un `EventSubscriber` de forma tipada.

```typescript
export interface DomainEventSubscriber<T> extends EventSubscriber {
    onEvent(event: DomainEvent<T>): void;
}
```
Creamos la interfaz `DomainEventBus` la cual nos permite agregar subscritores, eliminar subscriptores y enviar eventos de dominio.
```typescript
export interface DomainEventBus {
    subscribe(subscriber: EventSubscriber): void;
    unsubscribe(subscriber: EventSubscriber): void;
    publish(event: EventBase): void;
}
```
La implementación de nuestro `EventBus` será en memoria en este caso no necesitamos más. La lógica más importante se centra en método `publish()` donde cuando recibe un evento este filtrara a todos los subscriptores que coincidan con en el nombre de evento y notificara el mensaje a los subscriptores correspondientes

```typescript
export class InMemoryEventBus implements DomainEventBus {

    private subscribers: EventSubscriber[] = []

    subscribe(subscriber: EventSubscriber): void {
        this.subscribers.push(subscriber)
    }

    unsubscribe(subscriber: EventSubscriber): void {
        const subscriberIndex = this.subscribers.indexOf(subscriber);
        this.subscribers.splice(subscriberIndex, 1);
    }

    publish(event: EventBase): void {
        this.subscribers
            .filter(subscriber => subscriber.suscribeTo() === event.getName())
            .forEach(subscriber => subscriber.onEvent(event))
    }

}
```
Nuestro patrón observer está terminado nos resta implementar las clases concretas que tendrán la lógica de dominio.

Implementamos nuestro evento de dominio donde la data será la misma entidad User, pero puedes enviar los valores que estimes conveniente.

```typescript
export class UserCreated extends DomainEvent<User>{

    static EVENT_NAME = 'user-ms.user-created'
    
    constructor(user: User) {
        super(user)
    }

    getName(): string {
        return UserCreated.EVENT_NAME
    }

}

```

Implementamos nuestro caso de uso `NotifyUserCreatedByEmail`. La lógica no es compleja solo hace uso del puerto `EmailService`. lo último a tomar en cuenta es devolver el nombre de evento al que queremos suscribirnos en el método `suscribeTo()`.

```typescript
export class NotifyUserCreatedByEmail implements DomainEventSubscriber<User> {

    constructor(private readonly service: EmailService) { }

    async onEvent(event: DomainEvent<User>) {
        
        const user = event.getData()

        await this.service.send({
            to: user.email,
            message: `Congratulations your username is ${user.username}. you must to complete the register on ....`,
            sent: new Date()
        })

    }

    suscribeTo(): string {
        return UserCreated.EVENT_NAME
    }

}
```
Nuestro ubscriptor debemos agregarlo al `EventBus`. Dependiendo del framework que estés usando la estrategia de creación de componentes cambiará en esta ocasión no nos enfocaremos en ese punto:

```typescript
this.eventbus.suscribe(
    new NotifyUserCreatedByEmail(emailService)
)
```

Ahora debemos hacer uso de nuestro `EventBus` en nuestro caso de uso `CreateUser` y hacer unos pequeños cambios:


```typescript
export class CreateUserService implements CreateUser {
    
    constructor(
        private repository: UserRepository,
        private uniqueUsernameGenerator: UniqueUsernameGenerator,
        private eventbus: DomainEventBus
    ) { }
    
    async create(createuser: CreateUserDto): Promise<void> {
        
        const username = await this.uniqueUsernameGenerator.generate(createuser.firstname, createuser.lastname)
        
        const user = User.create({
            username: username,
            email: createuser.email
        })

        await this.repository.save(user)
        // pull events and publish
        user
            .pullEvents()
            .forEach(event => this.eventbus.publish(event))

    }

}

```
Hemos implementado nuestros casos de uso de una manera totalmente desacoplada sobre una entidad simple. Esto no termina aca, ya que tendremos escenarios donde nuestras entidades estarán compuestas por otras encontrándonos con relaciones uno a muchos, muchos a uno y muchos a muchos. Domain Driven Design nos introduce el concepto de `AgregateRoot` a esta relación de entidades

## 6 - AggregateRoot y entidades complejas

Un `Aggregate` no es nada más que una agrupación de entidades, value objects y propiedades a simples palabras representa un concepto de negocio que mantiene la coherencia entre sus entidades y objetos. Los agregados son un componente que orquesta todas las operaciones relacionadas con las entidades de forma transaccional para poder mantener su integridad.
Para acceder a los valores de nuestro agregado debemos definir una entidad raiz por la que sería como la puerta de entrada de nuestro agregado esto se le conoce como `AggregateRoot`.

Crearemos un `AggregateRoot` esta tendrá la misma estructura que nuestra clase `Entity`
```typescript
export abstract class AggregateRoot<T> {
    abstract equalsTo(e: T): boolean;
}
```
En este ejemplo promoveremos nuestra entidad `User` a un agregado el cual tendrá los roles que tiene el usuario.

```typescript
export enum Permission {
    READ = 'READ'
    EDIT ='EDIT'
    DELETE ='DELETE'
}

export class Module extends Entity<Module> {
    name: string;
    permissions: Permission[];
}

export class Role extends Entity<Role> {
    name: string;
    modules: Module[]
}

export class UserId extends Id {

}

export class User extends AggregateRoot<User> {

    userId: UserId;
    username: Username;
    email: Email;
    confirmed: boolean;
    roles: Role[];

    private constructor() {
        super()
    }

    static create(user: CreateNewUserDto): User {
        // code here...
    }  
    // more code...
}


```

Ahora el acceso de las propiedades de nuestro agregado y operaciones con los datos los realizaremos mediante nuestro agregado como podemos apreciar en este caso el agregado no hace nada diferente a lo que haríamos con una entidad. pero si la idea principal es mantener la coherencia entre sus objetos
```typescript

export class User extends AggregateRoot<User> {

    userId: UserId;
    username: Username;
    email: Email;
    confirmed: boolean;
    roles: Role[];

    private constructor() {
        super()
    }

    static create(user: CreateNewUserDto): User {
        // ... code here
    }

    addRole(role: Role) {
        // code here
    }

    removeRole(role: Role) {
        // code here
    }

    getModules() {
        // code here
    }


}
```
Nuestra estructura de carpetas quedaría asi 

```bash
├──  core
│   ├──  application
│   │   ├──  CreateUser.ts
│   │   ├──  event-subscribers
│   │   │   └──  NotifyUserCreatedByEmail.ts
│   │   └──  services
│   │       ├──  CreateUserService.ts
│   │       └──  UserCreatorService.spec.ts
│   ├──  core.module.ts
│   ├──  domain
│   │   ├──  events
│   │   │   └──  UserCreated.ts
│   │   ├──  model
│   │   │   ├──  entities
│   │   │   │   └──  User.ts
│   │   │   └──  valueobjects
│   │   │       ├──  Email.spec.ts
│   │   │       ├──  Email.ts
│   │   │       ├──  Username.spec.ts
│   │   │       └──  Username.ts
│   │   ├──  ports
│   │   │   ├──  inbound
│   │   │   │   └──  UniqueUsernameGenerator.ts
│   │   │   └──  outbound
│   │   │       └──  UserRepository.ts
│   │   └──  services
│   │       ├──  UniqueUsernameGeneratorService.ts
│   │       └──  UsernameGeneratorService.spec.ts
│   └──  shared
│       ├──  application
│       │   └──  ports
│       │       └──  outbound
│       │           └──  email-service
│       │               ├──  EmailMessage.ts
│       │               └──  EmailService.ts
│       ├──  domain
│       │   ├──  AggregateRoot.ts
│       │   ├──  DomainEvent.ts
│       │   ├──  DomainEventSubscriber.ts
│       │   ├──  Entity.ts
│       │   ├──  EventBus.ts
│       │   ├──  ValueObject.ts
│       │   └──  valueobjects
│       │       ├──  Id.spec.ts
│       │       └──  Id.ts
│       ├──  dto
│       │   ├──  CreateUserDto.ts
│       │   └──  RegisterEmailNotificationDto.ts
│       └──  error
│           └──  DomainException.ts
├──  infraestructure
│   ├──  adapters
│   │   ├──  in-memory-event-bus.service.ts
│   │   ├──  in-memory-user.repository.spec.ts
│   │   └──  in-memory-user.repository.ts
│   └──  infraestructure.module.ts
```

Tenemos cubierto como modelar nuestro dominio pero nuestro dominio debe estar dentro de un contexto definido para representar los caso de uso del negocio.



## 7 - Indentificando Bounded Context de nuestra aplicación

Los Bounded Context nos da los límites de nuestros modelos para poder separar en los distintos dominios y contextos estableciendo las relaciones necesarias para poder resolver la problemática de nuestro negocio de una manera más modular y adecuado al contexto de lo que está solucionando. Los casos de usos que realizamos para la creación de un usuario podemos englobarlos en un Bounded Context llamado "users" nuestra aplicación después puede integrar otros Bounded Context como puede ser el caso de "user-logs" o "authorization" pero cada uno engloba su propio contexto y si necesitamos que estos Bounded Context se comuniquen podemos realizarlo mediante los eventos de dominio. 

Para finalizar la estructura de directorios para representar un Bounded Context sería nada más que definir nuestros dominios en forma modular.

```bash
├──  core
│   ├──  auth
│   │   ├──  application
│   │   ├──  domain
│   │   └──  shared
│   ├──  shared
│   │   ├──  application
│   │   └──  domain
│   ├──  user-logs
│   │   ├──  application
│   │   ├──  domain
│   │   └──  shared
│   └──  users
│       ├──  application
│       ├──  domain
│       └──  shared
├──  infraestructure
└──  shared
```

Finalizando los Bounded Context pueden ser una característica del software o puede representar un microservicio.

## Finalizando con Domain Driven Design

Modelamos el dominio de nuestra aplicación con Domain Driven Design espero que este post pueda servirte si quieres emplear clean architectures. Conceptualmente puede abrumarnos ciertos conceptos, pero creo que la decisión de que arquitectura emplear en una solución de software dependerá de que es lo que necesitamos resolver es muy distinto resolver un problema específico del negocio a modelar un proceso entero de negocio que sea vital para la organización. Ten esto en mente para aplicar la arquitectura correcta.



Puedes ver los demás artículos de arquitectura hexagonal acá 😉

 * [Arquitectura hexagonal Parte I ](/posts/implementando-hexagonal-con-nestjs-part1/)
 * [Arquitectura hexagonal Parte II ](/posts/implementando-hexagonal-con-nestjs-part2/)

## [Github repository](https://github.com/nullpointer-excelsior/nestjs-northwind-hexagonal/tree/main/clean-architecture-examples/part-3-domain-full-design-with-ddd)


