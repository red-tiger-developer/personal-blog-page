---
title: Arquitectura Frontend 2. Cómo implementar Clean Architecture en el Frontend.
author: Benjamín
date: 2023-10-11 00:38:47 -0500
categories: [Frontend, React, Rxjs, Arquitectura Frontend, Javascript, Typescript]
tags: [frontend, react, rxjs, arquitectura frontend, javascript, typescript]
---

![img](https://i.ibb.co/LCChSkp/Screenshot-2023-10-12-at-11-54-36.png)

La aplicación de los principios de la arquitectura limpia (Clean Architecture) no se limita al ámbito del backend; de hecho, pueden ser igualmente beneficiosos cuando se aplican al frontend, ofreciendo una serie de ventajas sustanciales:

* `Separación de Lógica de Negocio y Interfaz de Usuario:` Al implementar la Clean Architecture en el frontend, podemos lograr la separación efectiva de la lógica empresarial compleja de la interfaz de usuario. Esta clara división permite una gestión más efectiva de ambos componentes, simplificando el mantenimiento y la escalabilidad.

* `Independencia Tecnológica:` La arquitectura limpia garantiza que el código de negocio no dependa de tecnologías específicas de interfaz de usuario, frameworks o bibliotecas. Esta independencia facilita la adaptación a nuevas tecnologías o la actualización de las existentes sin afectar la lógica central del negocio.

* `Pruebas Unitarias sobre la Lógica Central:` La estructura de Clean Architecture permite realizar pruebas unitarias de manera eficiente en la lógica central de la aplicación, asegurando su funcionamiento correcto y confiable. Esto reduce los errores y mejora la calidad del software.

* `Resistencia a Cambios Disruptivos en las Interfaces de Usuario:` La arquitectura limpia minimiza el impacto de cambios drásticos en las interfaces de usuario, como actualizaciones de librerías o frameworks, en la lógica de negocio. Esto resulta en una mayor estabilidad y reducción de riesgos.

* `Simplificación de Migraciones de Código:` La estructura ordenada y modular de la Clean Architecture facilita las migraciones de código, ya sea a nuevas tecnologías, plataformas o arquitecturas. Esto es especialmente valioso a medida que la aplicación crece y evoluciona.

Clean Architecture, particularmente en aplicaciones complejas, es una estrategia poderosa. Aquí, exploraremos un ejemplo concreto en el contexto de React, la cual es una potente librería para crear UI. Sin embargo, en mi experiencia personal, es muy propensa a convertirse en código espagueti. Si bien siempre podremos separar las funciones de negocio de las de UI, hay casos donde las lógicas asíncronas ligadas a nuestra lógica de negocio no nos permitirán separarnos totalmente de los Hooks de React.


## No abusaremos de useEffect, useState y useContext en el código principal de negocio.

¡Nos volvimos locos!. Trataremos de usar React solo en la parte de componentes visuales, y nos ayudaremos del paradigma de programación reactiva con RxJs. Comunicaremos la capa de negocio con la UI con 2 custom hooks: `useObservable()` y `useObservableValue()`. Estos son los intermediarios de nuestra lógica. Con este enfoque, nuestros componentes podrán suscribirse al estado de nuestra lógica de negocio de forma sencilla y limpia al componente.

### Creando el custom hook `useObservable()`

Este hook nos permite usar la subscripción hacia un observable y también terminará las subscripciones sin que nos preocupemos nosotros.

```tsx
import { useEffect } from "react";
import { Observable, Observer } from "rxjs";

export default function useObservable<T>(observable$: Observable<T>, callback: Partial<Observer<T>> | ((value: T) => void)) {
    
    useEffect(() => {
        const subscription = observable$.subscribe(callback)
        return () => subscription.unsubscribe()
    },[])    

}
```
### Escuchando estados globales o locales con `useObservableValue()`.

Este otro hook simulará a `useState()` y nos permitirá usar cualquier observable como estado de un componente.

```tsx
import { useEffect, useState } from "react";
import { Observable } from "rxjs";

type ObservableValue<T> = [
    value: T,
    error: any,
    isCompleted: boolean
]

export default function useObservableValue<T>(observable$: Observable<T>, defaultValue: T): ObservableValue<T> {

    const [value, setValue] = useState<T>(defaultValue)
    const [error, setError] = useState<false | Error>(false)
    const [isCompleted, setIsCompleted] = useState(false)
    
    useEffect(() => {
        const subscription = observable$.subscribe({
            next: (nextValue: T) => setValue(nextValue),
            error: err => {
                console.error("useObservableValueError", err)
                setError(err)
            },
            complete: () => setIsCompleted(true)
        })
        return () => subscription.unsubscribe()
    },[])    

    return [
        value, 
        error,
        isCompleted
    ]

}
```

Explico estos hooks de mejor manera en este post [arquitectura front 1](https://nullpointer-excelsior.github.io/posts/arquitectura-frontend-1-como-integrar-rxjs-4-react-para-aplicaciones-asincronas-complejas/) si bien por debajo aquí usamos useState y useEffect, solo serán aquí, ya nos olvidamos de estos hooks para las lógicas de negocio.

## Prueba de concepto: Juego interactivo con un laberinto, el clásico ratón atrapado y el ingrediente X.

Nuestra aplicación será un juego donde podemos generar un laberinto aleatorio y la misión del jugador será ayudar a un ratoncito a salir del laberinto. Es un juego clásico pero que tendrá un ingrediente especial. Para sacar al ratoncito debemos hacerlo con código, ya sea con código TypeScript o JavaScript.

Nuestra aplicación ahora suena compleja por los siguientes motivos:

- Generación del laberinto aleatorio.
- Control del ratoncito y definición de restricciones de negocio del juego.
- Transpilar TypeScript a JavaScript.
- Interpretar y controlar la ejecución de código JavaScript.
- Creación de sandbox de ejecución.
- Integración con editor de código.
- Medir puntajes del jugador.
- Código asíncrono por todos lados.

En pocas palabras, esto resume la complejidad del proyecto. Inicialmente, se sitúa en un nivel de complejidad moderado, pero debemos estar preparados para garantizar que el código sea sostenible y adaptable a medida que el proyecto crezca en el futuro.

## Estructura de directorios 

La estructura de directorio diferirá de la que puede ser hecha en un backend. Si quieres saber cómo es, puedes ver este artículo: [arquitectura hexagonal](https://nullpointer-excelsior.github.io/posts/implementando-hexagonal-con-nestjs-part1/). En el frontend tendremos otras prioridades en cuanto a los componentes de software de nuestra aplicación. Serán los siguientes:

* `application`:
    * casos de uso
    * puertos:
        * Browser api
        * utilidades varias
* `domain`:
    * puertos:
        * apis
        * serverless sdks
    * modelo
    * estado
* `infrastructure`:
    * adapters: implementaciones de puertos cuando corresponda
    * UI: componentes visuales, libs, frameworks, etc.
    * librerías de terceros

Si comparamos el frontend con el backend, el dominio del backend contendrá entidades y repositorios, pero en el frontend tendríamos los modelos de datos representados del backend y, en vez de la existencia de repositorios, tendremos el manejador de estados.

## MicroMouse Feature

No explicaremos la totalidad de las funcionalidades del proyecto, ya que alargaríamos mucho este artículo. Si deseas, puedes verlo y estudiarlo en el siguiente [repositorio](https://github.com/nullpointer-excelsior/micromouse-challenger). Me enfocaré en explicar la funcionalidad de Micromouse, la cual lleva la lógica principal del juego.


La estructura del módulo `micromouse` es la siguiente:

```bash
 micromouse
├──  application
│   ├──  MicroMouse.ts
│   └──  MicromouseGame.ts
├──  domain
│   ├──  Cell.ts
│   ├──  CellPosition.ts
│   ├──  Events.ts
│   ├──  Mouse.ts
│   ├──  MouseMaze.ts
│   ├──  MoveMouseResponse.ts
│   └──  state
│       └──  MicromouseState.ts
├──  dto.ts
├──  exceptions.ts
└──  infrastructure
    ├──  services.ts
    └──  ui
        └──  components
            ├──  maze
            │   ├──  CellContent.tsx
            │   └──  Maze.tsx
            └──  start-code-challenge-button
                └──  StartCodeChallengeButton.tsx
```

Esta estructura contiene lo siguiente:

* `application`: casos de uso de la aplicación
* `domain`: 
    * estructura del estado del módulo
    * lógica de negocio principal 
* `infraestructure`: 
    * componentes UI
    * instancia de servicios de uso global


### MicroMouseGame: casos de uso del juego del laberinto

Este componente se encarga de controlar el estado del juego. También actualizará el estado global del módulo.

```typescript
export type MicromouseGameOptions = {
    stopwatch: Stopwatch,
    gameTime: `${number}:${number}`,
    state: MicromouseState
}

export class MicromouseGame {

    private stopwatch: Stopwatch
    private gameTime: `${number}:${number}`
    private state$: MicromouseState

    constructor(options: MicromouseGameOptions) {
        this.stopwatch = options.stopwatch
        this.gameTime = options.gameTime
        this.state$ = options.state
    }

    start() {
        this.stopwatch.start()
        this.stopGameAt(this.gameTime)
    }

    finish() {
        this.stopwatch.stop()
        this.state$.resetMousePosition()
    }

    reset() {
        this.stopwatch.stop()
        this.state$.reset()
    }

    time() {
        return this.stopwatch.time()
    }

    stopGameAt(time: `${number}:${number}`) {
        this.stopwatch
            .time()
            .pipe(filter(timeValue => timeValue === `${time}:00`))
            .subscribe((time: string) => {
                this.stopwatch.stop()
                this.state$.updatePlayerResult({
                    time: time,
                    isWinner: false
                })
                this.state$.resetMousePosition()
            })

    }

    onGameOver(callback: () => void) {
        this.stopwatch.onStop().subscribe({
            next: () => {
                callback()
            },
            error: err => console.log(err)
        })
    }

    gameOver(): Observable<GameOverResponse> {
        return this.stopwatch.onStop().pipe(
            map(() => ({
                isWinner: this.state$.getIsWinner()
            }))
        )
    }

    updateScore(params: { message: string, position: string }) {
        this.state$.updateMousePosition(params.position)
        this.state$.updateMessage(params.message)
        this.state$.incrementMovements()
    }

    movements() {
        return this.state$.onMovement()
    }

    setup(code: string) {
        this.state$.setCode(code)
    }

    getSetup(): SetupGame {
        return {
            code: this.state$.getCode(),
            matrix: this.state$.getMaze()
        }
    }

    win() {
        this.state$.setIsWinner(true)
        this.finish()
    }

}

```

### MicroMouse: casos de uso de control del ratoncito dentro del laberinto

Este servicio se encargará de enviar las instrucciones al ratoncito del laberinto para moverse y se encargará de enviar eventos de dominio de forma global. También servirá de fachada del estado de la clase `Mouse` (ratoncito), la cual tiene la responsabilidad de conocer el laberinto y poder realizar los movimientos cuando corresponda.

```typescript
import { EventBus } from "../../utils/eventbus";
import { eventbus } from "../../utils/infrastructure";
import { Cell } from "../domain/Cell";
import { MicromouseEvent, MouseMoveEvent, MouseWinEvent } from "../domain/Events";
import { Mouse } from "../domain/Mouse";
import { MouseMaze } from "../domain/MouseMaze";
import { MoveMouseResponse } from "../domain/MoveMouseResponse";

export class MicroMouse {

    constructor(
        private readonly eventbus: EventBus<MicromouseEvent>,
        private readonly mouse: Mouse,
        private readonly moveDelay = 0,
    ) { }

    async move(position: 'up' | 'down' | 'left' | 'right'): Promise<MoveMouseResponse> {

        await new Promise(resolve => setTimeout(resolve, this.moveDelay));

        const response = this.mouse.move(position)
        
        this.eventbus.dispatch(new MouseMoveEvent({
            isMoved: response.mouseMoved,
            message: response.message,
            position: response.cellPosition.getCurrentPosition()
        }))

        if (response.mouseMoved && response.cellPosition.value.isExit()) {
            this.eventbus.dispatch(new MouseWinEvent("Felicitaciones ganaste maldito bastardo!!"))
        }

        return response

    }

    static create(params: { matrix: string[][], moveDelay: number }) {
        const maze = MouseMaze.create({
            matrix: params.matrix
        })
        const mouse = new Mouse(maze, maze.getPosition('A0'))
        return new MicroMouse(
            eventbus,
            mouse,
            params.moveDelay
        )
    }

    getCurrentPosition(): string {
        return this.mouse.getCurrentPosition()
    }

    getCurrentCell(): Cell {
        return this.mouse.getCurrentCell()
    }

    getUpCell(): Cell | null {
        return this.mouse.getUpCell();
    }

    getDownCell(): Cell | null {
        return this.mouse.getDownCell()
    }

    getLeftCell(): Cell | null {
        return this.mouse.getLeftCell()
    }

    getRightCell(): Cell | null {
        return this.mouse.getRightCell()
    }

}

```

### Mouse y Maze: objetos de dominio que contienen la lógica principal del juego.

Estos 2 componentes de dominio contienen la lógica principal del juego. `Mouse` tendrá la información de ubicación y de cómo moverse por el laberinto, y `MouseMaze` tendrá la lógica del laberinto y su estructura.

```typescript
import { Cell } from "./Cell";
import { CellPosition } from "./CellPosition";
import { MouseMaze } from "./MouseMaze";
import { MoveMouseResponse } from "./MoveMouseResponse";

export class Mouse {

    constructor(
        private readonly maze: MouseMaze,
        private currentPosition: CellPosition
    ) { }

    move(position: string): MoveMouseResponse {
        const navigateTo = this.currentPosition.getCell(position);

        if (!navigateTo) {
            return new MoveMouseResponse(
                '⚠️  Raton culiao No puedes moverte fuera de tu universo 🪐',
                this.currentPosition,
                false
            );
        }

        if (!navigateTo.canWalk()) {
            return new MoveMouseResponse(
                '✋ 🐁 no puedes ir por aca raton culiao.',
                this.currentPosition,
                false
            );
        }

        const cellPosition = this.maze.getPosition(navigateTo.position);
        this.currentPosition = cellPosition;

        return new MoveMouseResponse(
            `Me he movido 🐁 -> (${this.currentPosition.getCurrentPosition()})`,
            cellPosition,
            true
        );
    }

    getCurrentPosition(): string {
        return this.currentPosition.getCurrentPosition();
    }

    getCurrentCell(): Cell {
        return this.currentPosition.value;
    }

    getUpCell(): Cell {
        return this.currentPosition.up;
    }

    getDownCell(): Cell {
        return this.currentPosition.down;
    }

    getLeftCell(): Cell {
        return this.currentPosition.left;
    }

    getRightCell(): Cell {
        return this.currentPosition.right;
    }

}

export class MouseMaze {

    constructor(public readonly maze: Cell[][]) { }

    private findIndexes(valor: string): [number, number] | null {
        for (let i = 0; i < this.maze.length; i++) {
            for (let j = 0; j < this.maze[i].length; j++) {
                if (this.maze[i][j].position === valor) {
                    return [i, j];
                }
            }
        }
        return null;
    }

    getPosition(position: string): CellPosition | null {
        const loc = this.findIndexes(position);
        return loc ? this.showPositionMaze(loc) : null;
    }

    private showPositionMaze(loc: [number, number]): CellPosition {
        const maze = this.maze;
        const row = loc[0];
        const col = loc[1];

        if (row < 0 || col < 0) {
            throw new InvalidMazeLocationException('No se admiten coordenadas negativas');
        }
        if (row >= maze.length) {
            throw new InvalidMazeLocationException('No se admiten coordenadas fuera de rango');
        }
        if (col >= maze[0].length) {
            throw new InvalidMazeLocationException('No se admiten coordenadas fuera de rango');
        }

        const currentLocation = maze[row][col];

        const up = row > 0 ? maze[row - 1][col] : null;
        const down = row < maze.length - 1 ? maze[row + 1][col] : null;
        const left = col > 0 ? maze[row][col - 1] : null;
        const right = col < maze[0].length - 1 ? maze[row][col + 1] : null;

        return new CellPosition(
            currentLocation,
            up,
            down,
            left,
            right
        );
    }

    static create(props: MouseMazeProps): MouseMaze {
        const {matrix } = props
        const matrixCells: Cell[][] = [];
        for (let i = 0; i < matrix.length; i++) {
            const row: Cell[] = [];
            for (let j = 0; j < matrix[i].length; j++) {
                const position = String.fromCharCode('A'.charCodeAt(0) + i) + String(j);
                const cell = new Cell(position, matrix[i][j]);
                row.push(cell);
            }
            matrixCells.push(row);
        }
        return new MouseMaze(matrixCells);
    }

    static mapMatrixToCells(matrix: string[][]): Cell[][] {
        const matrixCells: Cell[][] = [];
        for (let i = 0; i < matrix.length; i++) {
            const row: Cell[] = [];
            for (let j = 0; j < matrix[i].length; j++) {
                const position = String.fromCharCode('A'.charCodeAt(0) + i) + String(j);
                const cell = new Cell(position, matrix[i][j]);
                row.push(cell);
            }
            matrixCells.push(row);
        }
        return matrixCells
    }
}

```

### Eventos de dominio

Nuestro dominio contiene eventos definidos, los cuales podremos escuchar de forma global para saber qué está ocurriendo en el juego. Esto es de utilidad para desacoplar distintas lógicas o módulos de la aplicación. Por ejemplo, no queremos mezclar la lógica de puntajes con la lógica de movimientos o de tiempo. Los eventos de dominio nos son de gran utilidad en arquitecturas limpias.

```typescript
import { DomainEvent } from "../../utils/eventbus";

export type MicromouseEvent = "micromouse.mouse-move" | "micromouse.mouse-win" | "micromouse.mouse-timeout"

export interface MouseEventProps {
    isMoved: boolean;
    message: string;
    position: string;
}

export class MouseMoveEvent extends DomainEvent<MouseEventProps, MicromouseEvent>{
    name: MicromouseEvent = "micromouse.mouse-move";
}

export class MouseWinEvent extends DomainEvent<string, MicromouseEvent> {
    name: MicromouseEvent = "micromouse.mouse-win";
}

export class MouseTimeoutEvent extends DomainEvent<string, MicromouseEvent> {
    name: MicromouseEvent = "micromouse.mouse-timeout"
}
```

Estos eventos son orquestados por un eventbus, una clase implementada con Rxjs. Si quieres saber más sobre este enfoque, te invito al siguiente artículo: [Arquitectura orientadas a eventos](https://nullpointer-excelsior.github.io/posts/como-implementar-un-eventbus-y-rxjs-en-conjunto/)

```typescript
export class ReactiveEventBus<E extends string> implements EventBus<E> {

    private events$ = new Subject<DomainEvent<any, E>>()

    onEvent<T extends DomainEvent<any, E>>(eventName?: E): Observable<T> {
        return iif(
            () => eventName !== undefined,
            this.events$.pipe(filter(event => eventName === event.name)),
            this.events$.asObservable()
        ) as Observable<T>;
    }

    dispatch<T extends DomainEvent<any, E>>(event: T): void {
        this.events$.next(event)
    }

}

export const eventbus = new ReactiveEventBus<MicromouseEvent>()
```
### StateManager sin las ultraconocidas librerías.

Nuestro estado global del módulo es manejado con RxJs. Podríamos haber empleado alguna otra alternativa con más características y definir puertos y adaptadores para no acoplarnos a una librería de terceros, pero RxJs nos da la posibilidad de crear un state manager de forma sencilla y entender qué realmente debemos considerar al crear un state manager por nosotros mismos.

```typescript

export interface State {
    message: string;
    mousePosition: string;
    maze: string[][];
    time: string
    movements: number
    isWinner: boolean
    code: string
}

export class MicromouseState {

    private defaultState: State 
    private message: BehaviorSubject<string>
    private maze: BehaviorSubject<string[][]>
    private mousePosition: BehaviorSubject<string>
    private time: BehaviorSubject<string>
    private movements: BehaviorSubject<number>
    private isWinner: BehaviorSubject<boolean>
    private code: BehaviorSubject<string>

    constructor(props: State) {
        this.defaultState = props
        this.maze = new BehaviorSubject(props.maze)
        this.message = new BehaviorSubject(props.message)
        this.mousePosition = new BehaviorSubject(props.mousePosition)
        this.time = new BehaviorSubject(props.time)
        this.movements = new BehaviorSubject(props.movements)
        this.isWinner = new BehaviorSubject(props.isWinner)
        this.code = new BehaviorSubject(props.code)
    }

    setMaze(maze: string[][]) {
        this.maze.next(maze)
    }

    updateMousePosition(position: string) {
        this.mousePosition.next(position)
    }

    updateMessage(message: string) {
        this.message.next(message)
    }

    onMessage() {
        return this.message.pipe(distinctUntilChanged())
    }

    onMaze() {
        return this.maze.pipe(distinctUntilChanged())
    }

    getMaze() {
        return this.maze.value
    }

    onMousePosition() {
        return this.mousePosition.pipe(distinctUntilChanged())
    }

    reset() {
        this.maze.next(this.defaultState.maze)
        this.message.next(this.defaultState.message)
        this.mousePosition.next(this.defaultState.mousePosition)
    }

    resetMousePosition() {
        this.mousePosition.next(this.defaultState.mousePosition)
        this.movements.next(0)
    }

    updatePlayerResult(payload: { time: string, isWinner: boolean}){
        this.time.next(payload.time)
        this.isWinner.next(payload.isWinner)
    }

    getIsWinner() {
        return this.isWinner.value
    }

    setIsWinner(payload: boolean) {
        this.isWinner.next(payload)
    }

    incrementMovements() {
        this.movements.next(this.movements.value + 1)
    }

    onMovement() {
        return this.movements.pipe(distinctUntilChanged())
    }

    setCode(payload: string){
        this.code.next(payload)
    }

    getCode(){
        return this.code.value
    }
}

```
Cabe destacar el siguiente código:

```typescript
onMousePosition() {
    return this.mousePosition.pipe(distinctUntilChanged())
}
```

El operador `distinctUntilChanged()` nos permitirá que quienes escuchen el estado de `MicromouseState` no reciban actualizaciones innecesarias, lo cual evitaría un problema de renderizados innecesarios. RxJs nos permitirá realizar operaciones más complejas con el estado si es necesario, pero con esta implementación nos basta.

### Instancia de servicios

Ciertos componentes deben estar en un scope global, por lo que lo único que debemos hacer es instanciarlos en un único lugar, en este caso en el archivo `src/micromouse_challenger/micromouse/infraestructure/services.ts`.

```typescript

const configuration = getGameConfiguration()


export const micromouseState = new MicromouseState({
    message: "Micromouse challenger iniciando 🏆 <- 🐁",
    mousePosition: 'A0',
    maze: [
        [' ', 'X', 'X', 'X', 'X'],
        [' ', 'X', ' ', ' ', ' '],
        [' ', 'X', ' ', 'X', ' '],
        [' ', ' ', ' ', 'X', ' '],
        ['X', 'X', 'X', 'X', 'S']
    ],
    isWinner: false,
    movements: 0,
    time: "00:00:00",
    code: "// you must to code a solution for Micromouse Challenge",
})


export const micromouseGame = new MicromouseGame({
    stopwatch: new Stopwatch(),
    gameTime: configuration.gameTimeout,
    state: micromouseState
})

```

> Si necesitáramos una instancia a nivel de componentes podemos hacer uso del hook useRef() de React.

## Conectando la UI con nuestra aplicación

Para hacer uso del estado de nuestra aplicación, solo debemos hacer uso de nuestros hooks previamente definidos:

```typescript
const [time] = useObservableValue(micromouseGame.time(), "00:00:00")
const [movements] = useObservableValue(micromouseGame.movements(), 0)
const [gameOver] = useObservableValue(micromouseGame.gameOver(), { isWinner: false })
```
De esta manera, las variables `time`, `movements` y `gameOver` actuarán como estado global, y React renderizará los componentes cuando cualquiera de esos cambie.

Mostraré como ejemplo el componente `<Sanbox/>`, el cual debe ejecutar el juego dentro de un `WebWorker`. Para lograr esto, utilizamos el caso de uso `executeCodeRunnerWorker`, el cual pertenece al módulo `code-runner`. Este caso de uso ejecuta el flujo del juego y se comunicará con el `WebWorker` que controlará el servicio `Micromouse`.

### Caso de uso: executeCodeRunnerWorker

Este caso de uso ejecutará el juego e internamente el código del jugador para poder mover el ratoncito en el laberinto. Todo este proceso debe ser ejecutado en un `WebWorker` para evitar interferir en el hilo principal del navegador.

```typescript

export default function executeCodeRunnerWorker(options: ExecuteCodeRunnerWorkerOptions) {

    const worker = createCodeRunnerWorker()

    micromouseGame.start()

    micromouseGame.onGameOver(() => {
        worker.terminate()
        options.onGameOver()
    })

    worker.sendMessage(new StartMicromouseMessage(micromouseGame.getSetup()))

    worker
        .onMessage<MicromouseMoveMessage>("MICROMOUSE_MOVE")
        .pipe(map(event => event.payload.micromouseEvent))
        .subscribe({
            next: (micromouseEvent) => {
                micromouseGame.updateScore({
                    message: micromouseEvent.data.message,
                    position: micromouseEvent.data.position
                })
            },
            error: (err) => console.log(err)
        })

    worker.onMessage<MicromouseMoveMessage>("MICROMOUSE_WIN").subscribe({
        next: () => {
            micromouseGame.win()
        },
        error: (err) => console.log(err)
    })

    return worker

}
```

Esta función debe retornar un worker, el cual debe ser terminado al desmontar el componente de forma obligatoria. Para lograrlo, podemos hacer uso del clásico `useEffect()`.

```typescript
useEffect(() => {
    const worker = executeCodeRunnerWorker({ onGameOver: () => setOpen(true) })
    return () => { 
        worker.terminate()
    }
}, [])
```

O mandamos a casa al `useEffect()` y lo hacemos con Observables de RxJs.

```typescript
export const codeRunnerWorker$ = new Observable(observer => {
    const worker = executeCodeRunnerWorker({ onGameOver: () => observer.next() })
    return () => { 
        worker.terminate()
    }
})
```

Ya creado nuestro observable `codeRunnerWorker$`, lo usamos con el hook `useObservable()` en nuestros componentes.

```typescript
useObservable(codeRunnerWorker$, () => {
    console.log("GameOver")
})
```

Finalmente, nuestros componentes de UI quedan así:

```typescript

export default function SandBox() {
    //  libraries and React hooks
    const [, navigate] = useLocation();
    const [open, setOpen] = useState(false)
    // game logic 
    const [time] = useObservableValue(micromouseGame.time(), "00:00:00")
    const [movements] = useObservableValue(micromouseGame.movements(), 0)
    const [gameOver] = useObservableValue(micromouseGame.gameOver(), { isWinner: false })
    
    // we need to execute a worker for code execution
    useObservable(codeRunnerWorker$, () => {
        // game over showing Modal with game results
        setOpen(true)
    })

    const handleAcceptModal = () => {
        setOpen(false)
        navigate(Paths.MICROMOUSE_CODERUNNER)
    }

    return (
        <div className="rounded w-[800px] h-[500px] bg-gray-900">
            <div className="flex flex-col items-center">
                <ScoreDashboard time={time} movements={movements} />
                <Maze />
                <Modal title={gameOver.isWinner ? "Felicitaciones" : "Que penita"} open={open} onAccept={handleAcceptModal} onClose={handleAcceptModal}>
                    {gameOver.isWinner ? <WinnerModalContent movements={movements} time={time} /> : <GameOverModalContent />}
                </Modal>
                {gameOver.isWinner ? <ConfettiExplosion zIndex={1000} /> : null}
            </div>
        </div>
    )
}
```

# Conclusión

Clean architecture en el frontend nos brinda la capacidad de abordar los casos de uso de manera independiente de la interfaz de usuario. En este proyecto, se han logrado los siguientes hitos en el diseño de aplicaciones:

* `Separación de Lógicas Complejas:` Hemos logrado aislar las lógicas de negocio complejas de la interfaz de usuario.

* `Conexión UI-Core Eficiente:` Implementamos una conexión efectiva entre la interfaz de usuario y el núcleo de la aplicación mediante dos sencillos custom hooks.

* `Programación Reactiva:` Utilizamos programación reactiva para permitir que cada componente de la aplicación se comunique de manera desacoplada.

* `State Manager Eficiente:` Creamos un gestor de estado sin recurrir a soluciones más pesadas como React Redux u otras alternativas altamente acopladas a React.

* `Mantenibilidad y Migración Transparente:` Nos hemos asegurado de que la mantenibilidad y migración de características sean lo más fluidas posible.

Es importante destacar que no se pretende desacreditar el uso de los hooks de React. Cada caso puede requerir soluciones particulares, y React ofrece una amplia variedad de hooks diseñados para abordar escenarios específicos. Este proyecto aún no ha enfrentado todos los posibles desafíos donde React y sus hooks pueden abordar, por lo que es fundamental adaptar la solución a las necesidades específicas de cada contexto.

Ahora, si quieres probar el desafío y saber de qué se trata `Micromouse Challenger`, puedes intentarlo en el siguiente link: [MicroMouse Challenger Page](https://nullpointer-excelsior.github.io/micromouse-challenger/).


## [Github repository](https://github.com/nullpointer-excelsior/micromouse-challenger)

Meme de cortesía

![meme](https://i.ibb.co/gRYDMpS/Screenshot-2023-10-12-at-11-40-34.png)
    