---
title: Aplicaciones en tiempo real con programación reactiva
author: Benjamin
date: 2023-02-23 10:32:00 -0500
categories: [Programacion, Typescript, GraphQL, Arquitectura de software, Nestjs]
tags: [NestJs, Reactive, Typescript]
---

![image](https://i.ibb.co/nf88d7N/Screenshot-2023-02-23-at-16-06-51.png)

Las aplicaciones en tiempo real se definen como aquellas que ofrecen una respuesta en tiempo real
a eventos del mundo real. Piensa en aplicaciones de chat en línea, aplicaciones de juegos,
aplicaciones de seguimiento de eventos en vivo, y aplicaciones de trading en línea.
En estas aplicaciones, la información debe ser entregada y procesada en tiempo real para
brindar una experiencia de usuario fluida y efectiva.

Este tipo de aplicaciones pueden ser enfrentadas con programación reactiva que más allá de ser una técnica es un paradigma
que nos ofrece la posibilidad de crear aplicaciones complejas y altamente escalables.

## Aplicaciones en tiempo real con programación reactiva

La programación reactiva es una técnica que se utiliza para desarrollar aplicaciones en tiempo real y se basa en el uso
de flujos de datos asincrónicos y eventos. En lugar de ejecutar operaciones de forma secuencial, la programación reactiva
permite que las operaciones se realicen en paralelo, lo que permite un procesamiento más rápido y una mejor escalabilidad.

No profundizaré más allá de explicar las bases, pero en forma resumida la programación reactiva proporciona
las siguientes características:

* `Responsivos:` Tiempos de respuestas rápidos y consistentes.
* `Resilientes:` Capaces de responder adecuadamente cuando existen errores o problemas.
* `Elásticos:` Adaptación a cargas de trabajos variables
* `Orientado a mensajes:` Cada flujo o componente se comunica de manera asíncrona mediante mensajes y subscripciones a estos.

La capacidad de trabajar con flujos de datos asíncronos nos proporciona una manera efectiva de crear aplicaciones en tiempo real,
donde necesitamos priorizar el tratamiento de datos de manera eficiente. Una fuente ilimitada de datos trae los siguientes desafíos:

* Uso de memoria.
* Escucha de eventos específicos.
* Procesamiento de datos
* Asincronismo

Para aterrizar esto utilizaremos la librería RxJs y trabajaremos sobre una aplicación que simulará el registro de un
ranking de puntos de multiples jugadores.
Donde esta aplicación recibirá los puntajes de los jugadores conseguidos en una partida.

Un ejemplo de petición es el siguiente:
```json
{
    "game": "ghost-of-kiev",
    "points": 100,
    "player": "red-panda",
    "playingTime": 120,
    "submittedAt": "2023-01-27"
}
```

Entonces con base a esto nuestra aplicación hará lo siguiente:

* Registro de los puntajes
* Registro de los jugadores nuevos que transmitan puntos
* Registro de ranking de los mejores 10 jugadores
* Notificación en tiempo real de la actualización del ranking
* Notification en tiempo real del jugador que se corone como posición n.°1


## Arquitectura de una aplicación en tiempo real

La arquitectura interna de la aplicación es la siguiente:

![schema](https://i.ibb.co/55xtJBp/Screenshot-2023-02-23-at-15-20-56.png)

Cada operación que involucra código reactivo tiene su responsabilidad definida y puede emitir eventos donde otros
componentes podrán escuchar y realizar tareas de forma independiente. Implementaremos una API con GraphQL y tendremos
una base de datos MongoDB

## Ventajas de la programación Reactiva

Si intentamos desarrollar nuestros casos de uso totalmente con programación imperativa, tendremos muchos desafíos en temas de rendimiento
y lógicas complejas de mantener que no serán fácil de adaptar a nuevos requerimientos. Principalmente te daré la idea de
que si hablamos de un sistema en tiempo real y queremos trabajar con los datos lo primero que haríamos es adaptar los datos a listas
y empezar a desarrollar nuestros casos de uso, pero esa base nos proporciona complejidad en algoritmos y posible código
propenso a convertirse en un plato de tallarines. En cambio si consideramos nuestros datos como un flujo podremos realizar
operaciones adecuadas ya las implementaciones del paradigma reactivo nos proporciona las siguientes herramientas:

* Poder de escuchar flujos de datos y realizar operaciones
* Encadenamiento de operaciones a los flujos
* Operaciones asíncronas
* Reutilización de flujos u operadores


## Implementando una API con GraphQL

Utilizaremos GraphQL para exponer nuestro modelo de datos y las operaciones que podremos realizar sobre este.

GraphQL es un lenguaje de consulta y una tecnología de servidor que permite a los clientes solicitar y recibir
solo los datos que necesitan, en una sola solicitud, en lugar de múltiples solicitudes como ocurre en REST.
Esto hace que la comunicación entre el cliente y el servidor sea más eficiente y flexible, lo que permite un desarrollo
de aplicaciones más rápido y escalable. GraphQL nos permite realizar las siguientes operaciones:

* `Queries:` Permiten solicitar datos al servidor. Podemos definir solo los datos que nos interesan.
* `Mutations:` Son operaciones que permiten modificar datos en el servidor.
* `Subscriptions:` Permiten establecer una conexión en tiempo real con el servidor, donde podemos recibir actualizaciones automáticas cada vez que ocurra un cambio en los datos que se han suscrito

Estas operaciones y el modelo de datos se definen en un esquema de GraphQL. En este caso definiremos
las siguientes operaciones:

* Enviar puntaje asociado a un jugador
* Obtener Jugadores
* Subscribirse a Ranking top 10
* Suscribirse a nuevo campeón

Y nuestro esquema GraphQL es el siguiente:

```
type Response {
    message: String!
}

type Score {
    id: String!
    game: String!
    points: Int!
    player: String!
    playingTime: Int!
    submitedAt: Date!
}

type Player {
    name: String!
    points: Int!
}

type Ranking {
    place: Int!
    player: String!
    points: Int!
}

type Champion {
    player: String!
    points: Int!
    datetime: Date!
}

"""Date custom scalar type"""
scalar Date

type Query {
    getScores: [Score!]!
    getPlayers: [Player!]!
}

type Mutation {
    createScore(score: ScoreInput!): Response!
}

input ScoreInput {
    game: String!
    points: Int!
    player: String!
    playingTime: Int!
    submitedAt: Date!
}

type Subscription {
    rankingUpdated: [Ranking!]!
    currentChampion: Champion!
}
```
Internamente debemos implementar como resolver las operaciones mediantes funciones llamadas resolvers,
pero este no es el foco del artículo asi que con las bases de GraphQL explicadas vamos con los casos de uso sobre
una aplicación en tiempo real

## Desarrollo de los casos de uso

La implementación tecnológica la haremos sobre una aplicación Nestjs con MongoDB e implementaremos un servidor Graphql.
Ciertas partes de la aplicación no se explicarán a detalle, ya que tenemos muchos componentes tanto en la parte de
GraphQL y MongoDD. Nos centraremos en los casos de uso que implementan lógica reactiva.

### Registro de los puntajes

El caso de uso `createScore` lo implementa la clase `ScoreService` con el método `save()`el cual recibe un objeto DTO
y realiza las operaciones con código reactivo.

```typescript
@Injectable()
export class ScoreService {

    constructor(
        private readonly repository: ScoreMongoRepository,
        private readonly source: ScoreSubject
    ) { }

    save(dto: SaveScoreDto): Observable<Score> {
        return of(dto) // nuestro dto lo transformamos a un observable
            .pipe( // hacemos uso del operador pipe para poder encadenar operadores
                map(dto => Score.create(dto)), // transfromamos nuetsro Observable<SaveScoreDto> a Observable<Score>
                switchMap(score => this.repository.create(score)), // switchMap nos permite terminar neustra subscripcion de Observable<Score> y realizar una operacion de guardado en nuesttra base de datos
                tap(score => this.source.emit(score)), // tap se ejecutara cada vez que un valor sea emitido en este caso emitimos que un nuevo Score se ha creado
            )
    }

    findAll(): Observable<Score[]> {
        return this.repository.stream().pipe(
            toArray(),
        )
    }

    onSaved(): Observable<Score> {
        return this.source.onEmited()
    }

}
```

El objeto `source` nos permite notificar eventos a quienes se suscriban al método `onSaved()` el funcionamiento interno es el siguiente:

```typescript
import { Subject } from "rxjs";
// clase base
export abstract class SubjectBase<T> {

    private readonly subject: Subject<T> = new Subject()

    emit(value: T) {
        this.subject.next(value)
    }

    onEmited() {
        return this.subject.asObservable()
    }

}

Injectable()
export class ScoreSubject extends SubjectBase<Score> {

}

```
Simplemente heredamos de SubjectBase y hacemos uso de sus métodos `emit()`para enviar un mensaje y `onEmited()` para
poder escuchar nuestros mensajes La clase Subject es propio de `rxjs` y su objetivo es poder emitir valores y obtenerlos
existen otras variantes que puedes consultar en su documentación para otros escenarios.

### Registro de los jugadores nuevos que transmitan puntos
La clase `ScoreService` registra los puntos y asu vez podemos utilizar su método `onSaved()` para realizar operaciones sobre estos eventos
de la siguiente manera:

```typescript
@Injectable()
export class SavePlayerService implements OnModuleInit {

    constructor(
        private readonly player: PlayerService,
        private readonly score: ScoreService,
    ) {
    }

    onModuleInit() {

        this.score.onSaved() // nos suscribimos a los eventos de Score Saved
            .pipe(
                switchMap(score => { // nos suscribimos a player.findByName()
                    return this.player
                        .findByName(score.player) // buscamos el jugador asociado al puntaje
                        .pipe(
                            defaultIfEmpty(Player.create({ name: score.player, points: 0 })), // si no existe creamos un jugador por defecto con 0 puntos
                            tap(player => { // con tap por cada jugador realizamoa la operación de suma de puntos
                                player.addPoints(score.points)
                            })
                        )
                }),
                switchMap(player => this.player.saveOrUpdate(player)), // del observable de Player nos cambiamos al de Guardar o actualizar jugador
            )
            .subscribe({
                next: player => Logger.log(`${player.toString()} Saved`), // log informativo
                error: err => Logger.error(err) //cualquier error lo notificamos
            })


    }

}
```
> SavePlayerService implementa OnModuleInit lo que nos permite ejecutar código cuando inicia el servidor.



### Registro de ranking

El registro del ranking es de una manera similar en este caso hacemos uso de `PlayerService.onSaved()` para obtener
la actualización de los jugadores y realizar los cálculos para el ranking de jugadores

````typescript
@Injectable()
export class SaveRankingService implements OnModuleInit {

    ranking$: Observable<Ranking[]>

    constructor(
        private readonly player: PlayerService,
        private readonly ranking: RankingService
    ) {
    }

    onModuleInit() {

        Logger.log('save-rankings STARTED')

        this.ranking$ = this.player.onSaved() // suscripcion a los jugadroes
            .pipe(
                switchMap(() => this.calculateRanking()),// calculamos el ranking
                switchMap(rankings => this.ranking.save(rankings)) // guardamos el ranking
            )

        this.ranking$.subscribe() // nos suscribimos para poder realizar los cálculos

    }


    calculateRanking() {

        const createRanking = (players: Player[]) => {
            return [...players]
                .sort((playerA: Player, playerB: Player) => playerB.points - playerA.points)
                .map((player, idx) => new Ranking({
                    place: idx + 1,
                    player: player.name,
                    points: player.points
                }))
        }

        return this.player.findAll() // obtenemos todos los jugadores
            .pipe(
                map(players => createRanking(players)) // creamos el ranking
            )

    }

}
````
Ahora para crear las notificaciones en tiempo real haremos uso del siguiente componente creado para nuestro server GraphQL
```typescript
@Injectable()
export class PubSubService {

    constructor(@Inject(GRAPHQL_PUB_SUB) private readonly pubsub: PubSub) {}

    publishRankingUpdated(rankings: Ranking[]) {
        this.pubsub.publish('rankingUpdated', rankings)
    }

    subscribeToRankingUpdated() {
        return this.pubsub.asyncIterator('rankingUpdated')
    }

    publishCurrentChampion(champion: Champion) {
        this.pubsub.publish('currentChampion', champion)
    }

    subscribeToCurrentChampion() {
        return this.pubsub.asyncIterator('currentChampion')
    }

}
```
Utilizamos la clase PubSub propia de la librería de Graphql y creamos los métodos de suscripción y los métodos de
notificación de mensajes.

### Notificación de la actualización del ranking

Para notificar el ranking en tiempo real implementamos el siguiente código:
```typescript
@Injectable()
export class RankingUpdatedService implements OnModuleInit {

    constructor(
        private readonly ranking: RankingService,
        private readonly pubsub: PubSubService
    ) { }

    onModuleInit() {

        Logger.log('ranking-updated STARTED')

        this.ranking
            .onSaved()
            .subscribe(ranking => this.pubsub.publishRankingUpdated(ranking))
    }

}
```
Solo nos suscribimos al `RankingService.onSaved()` y `pubsub.publishRankingUpdated(ranking)` hará el resto

### Notificación del jugador que se corone como posición n.°1

Para saber quien se corona como número 1 del juego empleamos el siguiente código:

```typescript
@Injectable()
export class ChampionUpdatedService implements OnModuleInit {

    constructor(
        private readonly ranking: RankingService,
        private readonly pubsub: PubSubService
    ) { }

    onModuleInit() {

        Logger.log('champion-updated STARTED')

        this.ranking.onSaved().pipe(
            mergeMap(rankings => rankings), // aplanamos el ranking es decir Observable<Ranking[]> -> Observable<Ranking>
            filter(ranking => ranking.place === 1), // filtramos el número 1
            distinctUntilKeyChanged('player'), // hacemos un distinct por la propiedad player y ademas se emitirá el valor cuando un nuevo jugador sea campeón
            map(ranking => ({ player: ranking.player, points: ranking.points, datetime: new Date() })), // mapeamos a un dto
            tap(champion => Logger.log('Current Champion ',champion)) // imprimimos el campeon
        )
            .subscribe(champion => this.pubsub.publishCurrentChampion(champion)) // suscripcion y notificacion del campeón
    }


}
```

## Ejecución de pruebas

Para levantar la aplicación y realizar pruebas estos son los comandos:

```bash
#!/bin/bash

# run mongo db
export database_name="ranking-db"
docker run --name "$database_name" -e MONGO_INITDB_DATABASE="$database_name" -e MONGO_INITDB_ROOT_USERNAME="$database_name" \
 -e MONGO_INITDB_ROOT_PASSWORD="$database_name" -p 27017:27017 -d mongo
# show mongodb container
docker ps

# run server
npm run start:dev

# install requests library
pip3 install requests

# run app client
python3 score_client.py
```

Ejecuta el siguiente script para simular las peticiones de jugadores usando:
```bash
#!/bin/bash

# run server
npm run start:dev
```
Ahora ve a http://localhost:3000/graphql y verás el Playground de GraphQL donde puedes realizar las consultas que te salga de los cojones:

Puedes crear un score:

![score](https://i.ibb.co/m0mjSzV/Screenshot-2023-02-23-at-13-01-12.png)

Consultar los puntos:

![get-score](https://i.ibb.co/xDq269y/Screenshot-2023-02-23-at-13-10-00.png)

Consultar el ranking en tiempo real:

![rank](https://i.ibb.co/JBXSgdG/Screenshot-2023-02-23-at-13-02-56.png)

Consultar el campeón de los jugadores:

![champ](https://i.ibb.co/GF1g8xh/Screenshot-2023-02-23-at-13-06-58.png)

Puedes realizar pruebas y crear nuevos casos con programación reactiva este proyecto es una prueba de concepto
con la que puedes realizar diversas pruebas.

## [Github repository](https://github.com/nullpointer-excelsior/realtime-reactive-app)

## Conclusión

Nos acercamos al mundo reactivo con un caso de usos de datos en realtime si bien este caso es básico y en el mundo real
la solución también implicaría componentes de infraestructura podemos entender de mejor manera este paradigma en el lado del backend.
Finalmente el meme de cortesía

![meme](https://i.ibb.co/k1vbddX/Screenshot-2023-02-23-at-16-39-54.png)




