---
title: Diseñando Microservicios con Domain Driven Design y NestJS
author: Benjamin
date: 2023-01-10 10:32:00 -0500
categories: [Programacion, Nestjs, Arquitectura de software, Typescript, Domain Driven Design, DDD]
tags: [typescript, NestJs, hexagonal, microservices, ddd]
---

![image](https://i.ibb.co/tL13zyh/Screen-Shot-2023-01-11-at-20-38-51.png)


Domain Driven Design o DDD para los amigos es un enfoque de diseño donde las reglas y modelo de negocio son el corazón de la aplicación. Nos olvidamos de diseños técnicos y nos centramos más en casos de uso y el modelo de contextos asociados a la problemática a resolver. DDD es ideal para proyectos complejos tanto en software como de arquitectura. Dicho esto veremos el poder de DDD en una arquitectura orientada a microservicios.

La arquitectura de microservicios no solo se trata de tomar una aplicación enorme como un monolito y dividirla en partes o módulos pequeños independientes en tecnología y lenguaje para poder obtener una independencia de funcionalidades. Esto conlleva grandes desafíos como la comunicación entre los diversos componentes, cada microservicio es independiente, pero contribuye a una misma causa, la que es integrar un sistema más grande donde este tiene un propósito que cumplir.

## Definiendo microservicios basados en DDD

Cuando modelamos un sistema con DDD dividimos nuestra problemática en Bounded Contexts (límites de contexto), estos son los que marcan los límites de una funcionalidad en nuestra aplicación y define que importancia, comportamiento responsabilidad tendrán los componentes que conformen este Bounded Context. Cada contexto es independiente y la comunicación entre estos puede darse mediante eventos de dominio. Sus componentes, aunque compartan ciertas propiedades con los de otros contextos, son totalmente diferentes, ya que tienen su responsabilidad definida por el negocio. Esta característica de diseño sobre el dominio que posee DDD nos da la ventaja de definir las tecnologías adecuadas para cada contexto. Con esto la implementación de nuestros microservicios estará guiada por el dominio y cada microservicio representará una problemática específica resolver del sistema


## El contexto define la intención y su responsabilidad

El contexto nos dice mucho 

![meme  contexto](https://i.ibb.co/hMmZYwt/The-Teachers-Copy-vs-What-They-Give-You-11012023201206.jpg)

La importancia del contexto en nuestros modelos es enorme. En aplicaciones tradicionales es típico modelar casos de negocio donde una tabla en base de datos representa distintos contextos dependiendo del caso. Por ejemplo, la tabla producto en una aplicación de ventas tiene su objetivo, pero en un contexto de bodega tiene otro y cada caso de uso diferente le dará más prioridad a ciertas propiedades sobre otras de la tabla. Cuando el sistema escale los requerimientos serán distintos y podemos también tener el problema que un cambio necesario para un contexto determinado como el de bodega afecte el contexto de ventas, esto repercute negativamente al momento de modelar nuestras lógicas de negocio, es por este motivo que una de las premisas de los microservicios es que cada microservicio es dueño de sus datos. Y esto encaja perfecto con la idea de Bounded Context en DDD.

## Modelando un servicio de viajes tipo UBER

Para levantar un stack de microservicios con DDD levantaremos un servicio de viajes tipo Uber donde definiremos unos módulos de ejemplo para poder implementar unos microservicios de ejemplo. Todo esto, ayudados por nuestro framework Gatuno NestJs. El proyecto es una implementación en Mono repositorio donde el sistema completo tendrá los siguientes contextos:

* `enrollment`: Registro de conductores y pasajeros
* `tracking`: Seguimiento del vehículo el cual podrá emitir alertas de pánico.
* `trips`: Agendamiento de viajes

Estas 3 características serán nuestros microservicios. Ahora generamos nuestro stack mediante la cli de NestJs lo importante acá es definir la librería `domain` la que contendrá la lógica del negocio donde incluye los contextos anteriormente descritos.

```bash
#!/bin/bash
nest new microservices-apps
nest generate library domain
```
> Si quieres aprender de monorepos con NestJs ven por [acá](https://nullpointer-excelsior.github.io/posts/mono-repo-with-nestjs/) muchacho

 
Entonces nuestra librería `domain` queda la siguiente manera:

```bash
#!/bin/bash
├──  context
│   ├──  enrollment
│   │   ├──  entity
│   │   │   ├──  Account.ts
│   │   │   ├──  Driver.ts
│   │   │   └──  Rider.ts
│   │   ├──  event
│   │   │   ├──  AccountCreated.ts
│   │   │   ├──  DriverCreated.ts
│   │   │   └──  RiderCreated.ts
│   │   ├──  port
│   │   │   ├──  AccountRepository.ts
│   │   │   ├──  DriverRepository.ts
│   │   │   └──  RiderRepository.ts
│   │   └──  vo
│   │       ├──  Capacity.ts
│   │       ├──  Color.ts
│   │       ├──  Email.ts
│   │       ├──  Model.ts
│   │       ├──  Vehicule.ts
│   │       └──  Year.ts
│   ├──  tracking
│   │   ├──  entity
│   │   │   ├──  Driver.ts
│   │   │   ├──  PanicAlert.ts
│   │   │   └──  Vehicule.ts
│   │   ├──  event
│   │   │   ├──  ButtonPanicActivated.ts
│   │   │   ├──  ButtonPanicDesactivated.ts
│   │   │   └──  GpsPositionUpdated.ts
│   │   ├──  port
│   │   │   ├──  DriverRepository.ts
│   │   │   └──  VehiculeRepository.ts
│   │   └──  vo
│   │       └──  GpsPosition.ts
│   └──  trips
│       ├──  entity
│       │   ├──  Driver.ts
│       │   ├──  Rider.ts
│       │   ├──  Trip.ts
│       │   └──  Vehicule.ts
│       ├──  event
│       │   └──  DriverAssigned.ts
│       ├──  GpsPosition.ts
│       └──  vo
│           ├──  MapPoint.ts
│           ├──  Pricing.ts
│           └──  TripState.ts
├──  index.ts
└──  shared
    ├──  domain
    │   ├──  PersonInfo.ts
    │   ├──  Plate.ts
    │   └──  vo
    ├──  libs
    │   └──  schemaValidator.ts
    └──  seedwork
        ├──  AggregateRoot.ts
        ├──  DomainEvent.ts
        ├──  DomainException.ts
        ├──  Entity.ts
        ├──  Identifier.ts
        ├──  port
        │   └──  DomainEventBus.ts
        ├──  UniqueEntityID.ts
        ├──  UseCase.ts
        └──  ValueObject.ts
```

Mientras tanto le echamos una mirada a al módulo de tracking y la entidad Vehicule

```typescript


export interface VehiculeState {
    plate: Plate;
    driver: Driver;
    position: GpsPosition;
    isPanicButtonActive: boolean;
    panicAlerts: PanicAlert[]
}

export class Vehicule extends Entity<VehiculeState> {

    constructor(props: EntityProps<VehiculeState>) {
        super(props)
        this.addEvent(
            new GpsPositionUpdated({
                ...this.state.position.getValue()
            })
        )
    }

    updatePosition(position: GpsPosition) {

        this.state.position = position

        this.addEvent(new GpsPositionUpdated({
            ...position.getValue()
        }))

    }

    activeButtonPanic(position: GpsPosition) {

        this.state.isPanicButtonActive = true
        this.state.position = position
        this.state.panicAlerts.push(new PanicAlert({
            position: position,
            type: 'button-panic-active'
        }))

        this.addEvent(new ButtonPanicActivated({
            vehiculeID: this.ID,
            position: position
        }))

    }

    desactiveButtonPanic(position: GpsPosition) {

        if (!this.state.isPanicButtonActive) {
            return
        }

        this.state.isPanicButtonActive = false
        this.state.position = position

        this.state.panicAlerts.push(new PanicAlert({
            position: position,
            type: 'button-panic-desactive'
        }))

        this.addEvent(new ButtonPanicDesactivated({
            vehiculeID: this.ID,
            position: position
        }))

    }

    get plate() {
        return this.state.plate
    }

    get position() {
        return this.state.position
    }

    get driver() {
        return this.state.driver
    }

    get isButtonPanicActive() {
        return this.state.isPanicButtonActive
    }

}
```
Nuestra entidad `Vehicule` muestra lógica asociada a la ubicación y alertas de pánico, pero si vemos el código de `Vehicule` en el contexto de `enrollment` vemos que esta no es una entidad sino un value object que tiene otra responsabilidad

```typescript

export interface VehiculeProps {
    plate: Plate;
    model: Model;
    color: Color;
    year: Year;
    capacity: Capacity;
}

export class Vehicule extends ValueObject<VehiculeProps> {

    constructor(props: {
        licence: string;
        email: string;
        firstname: string;
        lastname: string;
        phoneNumber: string;
        plate: string;
        model: string;
        color: string;
        year: string;
        capacity: number;
    }){
        super({
            plate: new Plate(props.plate),
            capacity: new Capacity(props.capacity),
            color: new Color(props.color),
            model: new Model(props.model),
            year: new Year(props.year)
        })
    }

    get plate() {
        return this.props.plate.getValue()
    }

    get model() {
        return this.props.model.getValue()
    }

    get color() {
        return this.props.color.getValue()
    }

    get year() {
        return this.props.year.getValue()
    }

    get capacity() {
        return this.props.year.getValue()
    }

}
```
Con esto podemos aclarar más la idea de contextos y sus límites.

## Librería application y la definición de los casos de uso

Nuestra librería `domain` esta implementada ahora crearemos una nueva librería llamada `application`.

```bash
#!/bin/bash
nest generate library application
```
Esta contendrá los casos de uso de nuestra aplicación esta capa contendrá la definición de los módulos de NestJs por cada contexto de nuestro dominio.

Caso de uso `enrollment`

```typescript
export class EnrollmentUseCasesService {

    constructor(
        private readonly driver: DriverRepository,
        private readonly rider: RiderRepository,
        private readonly account: AccountRepository,
        private readonly eventbus: DomainEventBus
    ) {}

    async createAccount(email: string) {
        
        const account = Account.create(email)
        
        await this.account.save(account)
        this.eventbus.dispatch(account.pullEvents())

    }

    async enrollDriver(dto: EnrollDriverDto) {
        
        const account = await this.account.findById(dto.accountID)
        const driver = Driver.create({ ...dto, account })
        
        await this.driver.save(driver)
        this.eventbus.dispatch(driver.pullEvents())

    }

    async enrollRider(dto: EnrollRiderDto) {
        
        const account = await this.account.findById(dto.accountID)
        const rider = Rider.create({ ...dto, account })
        
        this.rider.save(rider)
        this.eventbus.dispatch(rider.pullEvents())

    }

}
```

Caso de uso `tracking`

```typescript
export class TrackingUseCasesService {

    constructor(
        private readonly vehicule: VehiculeRepository, 
        private readonly eventbus: DomainEventBus
    ) {}

    async startTracking(dto: StartTrackingDto) {

        const driver = Driver.create({ ...dto.driver })
        const vehicule = Vehicule.create({ ..dto, vehicule })

        await this.vehicule.save(vehicule)
        this.eventbus.dispatch(vehicule.pullEvents())

    }

    async updateTracking(dto: UpdateTrackingDto) {

        const vehicule = await this.vehicule.findById(new UniqueEntityID(dto.vehiculeID))
        vehicule.updatePosition(new GpsPosition({ 
            ...dto 
        }))

        await this.vehicule.save(vehicule)
        this.eventbus.dispatch(vehicule.pullEvents())

    }
    
}
```

## Integrando el dominio con Nestjs mediante módulos dinámicos

Los módulos dinámicos que nos provee NestJs son excelentes para integrar componentes de software totalmente desacoplados.

Empezamos definiendo la estructura de las opciones de nuestro módulo, lo haremos con el contexto de tracking, pero la misma lógica se aplica a otros contextos. El parámetro `Type` es propio de NestJs, Nos ayuda a trabajar con `providers` y `modules`

```typescript
interface TrackingModuleOptions {
    modules: Type[]
    adapters: {
        vehiculeRepository: Type<VehiculeRepository>;
        eventbus: Type<DomainEventBus>
    }
}
```

`TrackingModuleOptions` recibe los siguientes parámetros:
* `modules`: módulos NestJs a importar para poder hacer uso de providers externos.
* `adapters`: providers que implementan los puertos definidos en nuestro dominio, estos providers deben estar en los módulos registros por le parámetro `modules` de `TrackingModuleOptions`.

Ahora la interfaz `TrackingModuleOptions` la integraremos al método `register()` de nuestro módulo `tracking` de esta manera nuestro casos de uso podemos inyectarlo como proveedor de NestJS mediante la ayuda e `useFactory`

```typescript
@Module({})
export class TrackingModule {

    static register(options: TrackingModuleOptions): DynamicModule {

        const { modules, adapters } = options
        const { vehiculeRepository, eventbus } = adapters

        return {
            module: TrackingModule,
            imports: [
                ...modules,
            ],
            exports: [
                TrackingUseCasesService,
            ],
            providers: [
                {
                    provide: TrackingUseCasesService,
                    useFactory(repository: VehiculeRepository, eventbus: DomainEventBus) {
                        return new TrackingUseCasesService(repository, eventbus)
                    },
                    inject:[
                        vehiculeRepository,
                        eventbus
                    ]
                }
            ]
        }

    }

}
```

Y ahora crearemos nuestro microservicio `tracking-ms`

```bash
#!/bin/bash
nest generate app tracking-ms
nest genrate module core
nest genrate module adapter
nest genrate module http-server

```
Para poder hacer uso de nuestro módulo solo debemos crear los adaptadores correspondientes al módulo `tracking` en este caso necesitamos:

* `DomianEventBus`: encargado de emitir eventos de dominio en este caso nos integraremos con RabbitMQ con el cual enviaremos nuestro evento de dominio. 
* `VehiculeRepository`: encargado de la persistencia de vehículos en este caso es una implementación ordinaria en memoria.

```typescript
@Injectable()
export class TrackingEventService implements DomainEventBus {
    
    constructor(private readonly rabbitmq: RabbitMQClientService) {}

    async dispatch(events: DomainEvent<any>[]): Promise<void> {
        events.forEach(event =>this.rabbitmq.emitTo(event.name, event.data))
    }

}

@Injectable()
export class VehiculeService implements VehiculeRepository {
    
    private vehicules: Map<UniqueEntityID, Vehicule> = new Map()

    async save(vehicule: Vehicule): Promise<void> {
        this.vehicules.set(vehicule.ID, vehicule)
    }
    
    async findById(id: UniqueEntityID): Promise<Vehicule> {
        return this.vehicules.get(id)
    }

}
```

## La comunicación entre microservicios mediante eventos de dominio 

Los eventos de dominio que generaran nuestro contextos serán la forma en que estos se comunicaran entre sí de forma asíncrona para esto podemos implementar un broker de mensajería con RabbitMQ a modo de ejemplo. Para lograr un componente encargado de los eventos reutilizable generaremos un módulo llamdo `event-queue` de NestJs en la librería `shared` de nuestro stack.

```bash
 event-queue
├──  contants.ts
├──  event-queue.module.ts
└──  rabbitmq
    ├──  rabbitmq-message.ts
    └──  services
        └──  rabbitmq-client.service.ts
```
Y el código cliente de nuestro RabbitMQ:

```typescript

export interface RabbitMQMessage<T> {
    id: string;
    pattern: string;
    timestamp: Date;
    data: T;
}


@Injectable()
export class EventQueueService {
    
    constructor(@Inject(RABBITMQ_CLIENT) private client: ClientProxy) { }
 
    emitTo<T>(pattern: string, payload: T): RabbitMQMessage<T> {
        const message: RabbitMQMessage<T> = {
            id: uuidv4(),
            pattern: pattern,
            timestamp: new Date(),
            data: payload
        }
        this.client.emit(pattern, message)
        return message
    }

}
```

## Registrando de modulos en microservicio

Con nuestros puertos definidos ahora inicializaremos nuestro módulo `tracking` en el archivo `apps/tracking-ms/src/core/core.module.ts` haciendo uso de los providers del módulo `AdaperModule`


```typescript

const trackingModule = TrackingModule.register({
    modules: [
        AdapterModule
    ],
    adapters: {
        vehiculeRepository: VehiculeService,
        eventbus: TrackingEventService
    }
})

@Global()
@Module({
    imports:[
        trackingModule
    ],
    exports: [
        trackingModule
    ]
})
export class CoreModule {}
```
Así ya tenemos nuestro microservicio listo su estructura es la siguiente:

```bash
├──  core
│   │   └──  core.module.ts
│   ├──  infraestructure
│   │   ├──  adapter
│   │   │   ├──  adapter.module.ts
│   │   │   └──  services
│   │   │       ├──  tracking-event.service.ts
│   │   │       └──  vehicule.service.ts
│   │   └──  http-server
│   │       ├──  controller
│   │       │   └──  tracking.controller.ts
│   │       ├──  http-server.module.ts
│   │       └──  model
│   │           ├──  StartTrackingRequest.ts
│   │           └──  UpdateTrackingRequest.ts
```

Nuestros otros contextos tendrán una implementación casi idéntica, diferirán en los adaptadores y otras propiedades que dependerán del caso.

Para iniciar nuestra aplicación ejecutamos lo siguiente:

```bash
docker run --rm -it --hostname DDD-rabitmq -p 15672:15672 -p 5672:5672 rabbitmq:3-management

nest start -w tracking-ms
nest start -w enrollment-ms
nest start -w trips-ms

```

## Conclusiones

Diseñamos la base para una arquitectura de microservicios utilizando las bondades de diseño que nos brinda DDD. Este enfoque no solo nos da una mirada más al negocio, sino que se complementa bien en el diseño de software en sistemas complejos y distribuidos si bien DDD puede ser complejo, empezar por utilizar sus conceptos será un buen inicio.



## [Github repository](https://github.com/nullpointer-excelsior/nestjs-microservices-with-ddd)

## Finalizando 

Para terminar nos despedimos con el meme de cortesía.

![meme](https://i.ibb.co/bJ40Znm/Zombo-Meme-27122022141407.jpg)





