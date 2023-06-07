---
title: Conociendo el patrón de diseño state machine.
author: Benjamin
date: 2023-06-07 10:32:00 -0500
categories: [Programacion, Typescript, Arquitectura de software, Patrones de diseño ]
tags: [Design Patterns, Typescript]
---

![image](https://i.ibb.co/BrRJpjL/Screenshot-2023-06-07-at-12-17-16.png)

Sí correcto este es otro post de patrones de diseño. lo distinto es que es uno de los que nadie habla pero se usa muy a menudo y no nos damos ni cuenta. Hoy voy a hablarles del patrón de diseño conocido como "state machine" (o máquina de estados). ¡Sí, también esta también relacionado con esos diagramas llenos de flechas y círculos que nos ayudan a modelar el comportamiento de nuestros sistemas! Pero no se preocupen, no vamos a sumergirnos en una telenovela de diagramas aburridos; en su lugar, exploraremos un ejemplo práctico y util para ilustrar cómo funciona este patrón en el mundo real.

## ¿Qué es una State Machine?

El patrón State Machine se utiliza para modelar y controlar el comportamiento de un objeto en función de su estado interno. Consiste en definir una serie de estados posibles y las transiciones permitidas entre ellos. Cada estado representa una configuración y comportamiento específico del objeto, y las transiciones definen cómo se puede cambiar de un estado a otro.

### Ventajas del patrón de diseño State Machine:

* Claridad y estructura: Proporciona una forma clara y estructurada de modelar el comportamiento de un sistema, facilitando la comprensión de las transiciones de estado y las acciones asociadas.
* Mantenibilidad y extensibilidad: Al separar el comportamiento en diferentes estados y transiciones, la máquina de estados se vuelve modular, lo que facilita el mantenimiento y la extensión del sistema sin afectar otras partes.
* Control de flujo: Permite un control de flujo preciso y determinista, ya que cada estado representa un conjunto definido de acciones y condiciones.

### Desventajas del patrón de diseño State Machine:

* Complejidad inicial: Requiere un esfuerzo adicional y puede resultar más complejo en comparación con enfoques lineales, ya que implica definir estados, transiciones y acciones, lo que aumenta la complejidad del código.
* Escalabilidad limitada: A medida que el sistema crece y las interacciones entre estados se vuelven más complejas, mantener y escalar la máquina de estados puede volverse difícil, ya que agregar nuevos estados y transiciones implica modificar múltiples partes del código.
* Sobrecarga de memoria: Dependiendo de la implementación, puede requerir más memoria para almacenar el estado actual y los datos asociados, lo cual puede ser una preocupación en sistemas con restricciones de recursos o dispositivos con capacidades limitadas.

## Implementacion de machine state

Implementaremos el Patrón State Machine patrón State Machine utilizando TypeScript y la biblioteca RxJS.

```typescript
import { filter, map } from 'rxjs/operators';


export interface State<T> {
    name: T;
    data: any;
}

export interface Transition<T> {
    from: T;
    to: T[];
}

export interface StateMachineConfig<T> {
    initialState: State<T>;
    transitions?: Transition<T>[];
}

export class InvalidStateError extends Error {
    constructor(message: string) {
        super(message)
        this.name = 'InvalidStateError'
        this.stack = ''
    }
}

```

Este código define las siguientes interfaces y clases que actuan como base de nuestro modelo de StateMachine:

### State<T>
La interfaz `State<T>` representa un estado en la máquina de estados. Tiene dos propiedades:

name: representa el nombre del estado.
data: almacena cualquier dato adicional asociado con el estado.

### Transition<T>
La interfaz `Transition<T>` representa una transición entre dos estados. Tiene dos propiedades:

`from`: representa el estado desde el cual se realiza la transición.
`to`: es un arreglo que contiene los estados a los que se puede transicionar desde el estado de origen.

### StateMachineConfig<T>
La interfaz `StateMachineConfig<T>` define la configuración inicial de la máquina de estados. Tiene dos propiedades:

`initialState`: representa el estado inicial de la máquina de estados.
`transitions (opcional)`: es un arreglo de objetos Transition<T> que define las transiciones permitidas entre los estados.

### InvalidStateError
La clase `InvalidStateError` es una subclase de Error y se utiliza para representar errores relacionados con transiciones de estados no válidas. Se personaliza con un mensaje específico de error.

### StateMachine 

En esta parte de la implementación, se define la clase `StateMachine<T>` que implementa la lógica principal de la máquina de estados. Incluyendo el manejo de estados y la validacion entre transiciones.

```typescript
export class StateMachine<T> {

    private currentState: BehaviorSubject<State<T>>;
    private transitions: Transition<T>[];

    constructor(config: StateMachineConfig<T>) {
        this.currentState = new BehaviorSubject(config.initialState)
        this.transitions = config.transitions ? config.transitions : []
    }

    state() {
        return this.currentState.asObservable()
    }

    stateValue(){
        return this.currentState.getValue()
    }

    stateFor(stateName: T) {
        return this.currentState.asObservable().pipe(
            filter(state => state.name === stateName),
            map(state => state.data)
        )
    }

    transition(state: State<T>) {
        const currentState = this.currentState.getValue().name
        const transitionToValidate = this.transitions
            .filter(t => t.from === currentState)
            .map(t => t.to)
            .reduce((result, element) => result.concat(element), [])
        if (!transitionToValidate.includes(state.name)) {
            throw new InvalidStateError(`The transition from "${currentState}" to "${state.name}" is invalid. The valid transitions for "${currentState}" are "${transitionToValidate}".`)
        }
        this.currentState.next(state)
    }

}
```



Nuestra clase `StateMachine<T>` Tiene los siguientes miembros y métodos:

### Miembros
`currentState`: Un objeto `BehaviorSubject<State<T>>` que almacena el estado actual de la máquina de estados. Es un BehaviorSubject de la biblioteca RxJS, que permite observar cambios en el estado.
`transitions`: Un array de objetos `Transition<T>` que almacena las transiciones definidas para la máquina de estados.
### Constructor
El constructor de `StateMachine<T>` acepta un objeto `StateMachineConfig<T>` como parámetro. Inicializa el `currentState` con el estado inicial proporcionado en la configuración y asigna las transiciones si se proporcionaron. Si no se proporcionan transiciones, se asigna un array vacío.

### Métodos
`state()`: Devuelve un observable que emite el estado actual de la máquina de estados. Los observadores pueden suscribirse a este observable para recibir actualizaciones sobre cambios de estado.

`stateValue()`: Devuelve el valor actual del estado sin la funcionalidad de observación. Proporciona acceso directo al estado actual sin la necesidad de suscribirse a un observable.

`stateFor(stateName: T)`: Devuelve un observable que filtra el estado actual por el nombre proporcionado. Solo emite el estado si coincide con el nombre especificado y mapea el resultado al valor de data del estado.

`transition(state: State<T>)`: Realiza una transición de estado. Comprueba si la transición proporcionada es válida, verificando si existe una transición definida desde el estado actual hacia el estado proporcionado. Si la transición es inválida, se lanza un `InvalidStateError` con un mensaje de error descriptivo.

## Ejemplo de Control de calidad de una manufactura de un producto.

Para implementar un ejemplo práctico, crearemos una máquina de estados para controlar el proceso de control de calidad de un producto. El objetivo es modelar y gestionar los diferentes estados por los que pasa un producto durante el proceso de control de calidad.

La máquina de estados se define utilizando el tipo `QualityControlState`, que es una unión de literales de cadena que representan los posibles estados del control de calidad. Estos estados son: 

* `factory-proccess`: proceso de fabricación.
* `visual-inspection`: inspección visual.
* `functional-inspection`: inspección funcional. 
* `approved`: aprobado.
* `rejected`: rechazado.

```typescript
export type QualityControlState = 'factory-proccess' | 'visual-inspection' | 'functional-inspection' | 'approved' | 'rejected'
```

La configuración de la máquina de estados se define en la variable config. El estado inicial se establece como `factory-proccess` y se proporciona una descripción de datos asociada al estado inicial. A continuación, se definen las transiciones permitidas entre los estados. Por ejemplo, se puede realizar una transición desde `factory-proccess` a `visual-inspection`, desde `visual-inspection` a `functional-inspection` o `rejected`, etc.

```typescript
const config: StateMachineConfig<QualityControlState> = {
    initialState: {
        name: 'factory-proccess',
        data: 'Building product'
    },
    transitions: [
        { from: 'factory-proccess', to: ['visual-inspection'] },
        { from: 'visual-inspection', to: ['functional-inspection', 'rejected'] },
        { from: 'functional-inspection', to: ['rejected', 'approved'] },
        { from: 'rejected', to: ['factory-proccess'] },
    ]
}
```

Se crea una instancia de la clase `StateMachine<QualityControlState>` llamada productState utilizando la configuración definida. Esta instancia representa el estado actual del producto durante el control de calidad.

```typescript
let productState = new StateMachine<QualityControlState>(config)
```

La función `executeTransitions` se utiliza para ejecutar las transiciones con un intervalo de 1 segundo. Recibe un arreglo de estados y devuelve un observable que emite los estados uno por uno en el intervalo de tiempo especificado. En este ejemplo, se utiliza `executeTransitions` para simular el progreso del control de calidad.

```typescript
const executeTransitions = (states: State<QualityControlState>[]) => {
    return interval(1000).pipe(
        take(states.length),
        map((index) => states[index])
    )
}
```

La función `successTransitions` simula una serie de transiciones exitosas en el control de calidad. Define un arreglo de estados que representan las transiciones a realizar, como la inspección visual, la inspección funcional y la aprobación del producto. Luego, se suscribe al observable devuelto por `executeTransitions` y cada vez que se emite un estado, se realiza la transición correspondiente utilizando el método transition de la instancia `productState`.

```typescript
const successTransitions = () => {
    const transitions: State<QualityControlState>[] = [
        { name: 'visual-inspection', data: 'Performing visual inspection' },
        { name: 'functional-inspection', data: 'Performing functional inspection' },
        { name: 'approved', data: 'Product approved' },
    ];
    executeTransitions(transitions)
        .subscribe((state: State<QualityControlState>) => productState.transition(state));
}
```

La función `failedTransition` simula una transición fallida en el control de calidad. Define un arreglo de estados que representa la inspección visual y el rechazo del producto. Al igual que en `successTransitions`, se realiza la transición correspondiente utilizando el método transition.

```typescript
const failedTransition = () => {
    const transitions: State<QualityControlState>[] = [
        { name: 'visual-inspection', data: 'Performing visual inspection' },
        { name: 'rejected', data: 'Product rejected' },
    ];
    executeTransitions(transitions)
        .subscribe((state: State<QualityControlState>) => productState.transition(state));
}
```

La función `invalidTransitions` simula una serie de transiciones inválidas en el control de calidad. Define un arreglo de estados que intenta realizar una transición desde `visual-inspection` a `approved`, lo cual no es una transición permitida. Se suscribe al observable devuelto por `executeTransitions` y cuando se intenta realizar la transición inválida, se captura el error utilizando el bloque error en la suscripción.

```typescript
const invalidTransitions = () => {
    const transitions: State<QualityControlState>[] = [
        { name: 'visual-inspection', data: 'Performing visual inspection' },
        { name: 'approved', data: 'Product approved' },
    ];
    executeTransitions(transitions)
        .subscribe({
            next: (state: State<QualityControlState>) => productState.transition(state),
            error: (err) => console.error(err.message)
        });
}
```

Finalmente, se realiza la suscripción al estado actual del producto utilizando el método `state()` de `productState`. Cada vez que el estado cambia, se imprime el nuevo estado en la consola.

```typescript
productState
    .state()
    .subscribe(state => console.log('Product state', state))

successTransitions()
```

Al llamar a `successTransitions()`, se ejecutan las transiciones exitosas simuladas y se observa cómo el estado del producto cambia a medida que avanza el proceso de control de calidad.

Finalmente el código completo es el siguiente:

```typescript
export type QualityControlState = 'factory-proccess' | 'visual-inspection' | 'functional-inspection' | 'approved' | 'rejected'

const config: StateMachineConfig<QualityControlState> = {
    initialState: {
        name: 'factory-proccess',
        data: 'Building product'
    },
    transitions: [
        { from: 'factory-proccess', to: ['visual-inspection'] },
        { from: 'visual-inspection', to: ['functional-inspection', 'rejected'] },
        { from: 'functional-inspection', to: ['rejected', 'approved'] },
        { from: 'rejected', to: ['factory-proccess'] },
    ]
}

let productState = new StateMachine<QualityControlState>(config)

/**
 * Execute transitions by 1 second interval.
 * @param states 
 * @returns 
 */
const executeTransitions = (states: State<QualityControlState>[]) => {
    return interval(1000).pipe(
        take(states.length),
        map((index) => states[index])
    )
}


const successTransitions = () => {
    const transitions: State<QualityControlState>[] = [
        { name: 'visual-inspection', data: 'Performing visual inspection' },
        { name: 'functional-inspection', data: 'Performing functional inspection' },
        { name: 'approved', data: 'Product approved' },
    ];
    executeTransitions(transitions)
        .subscribe((state: State<QualityControlState>) => productState.transition(state));
}

const failedTransition = () => {
    const transitions: State<QualityControlState>[] = [
        { name: 'visual-inspection', data: 'Performing visual inspection' },
        { name: 'rejected', data: 'Product rejected' },
    ];
    executeTransitions(transitions)
        .subscribe((state: State<QualityControlState>) => productState.transition(state));
}

const invalidTransitions = () => {
    const transitions: State<QualityControlState>[] = [
        { name: 'visual-inspection', data: 'Performing visual inspection' },
        { name: 'approved', data: 'Product approved' },
    ];
    executeTransitions(transitions)
        .subscribe({
            next: (state: State<QualityControlState>) => productState.transition(state),
            error: (err) => console.error(err.message)
        });
}


// Subscripcion al estado de un producto
productState
    .state()
    .subscribe(state => console.log('Product state', state))

successTransitions()
```

## Conclusiones

Vimos un caso simple para visualizar los posibles estados de un producto. El uso del patrón de máquina de estados en la gestión de estados de software ofrece varias ventajas. Proporciona una forma clara de modelar el comportamiento del sistema y facilita la comprensión y el mantenimiento del código al encapsular la lógica en estados individuales. Además, permite la extensibilidad al agregar o modificar estados y transiciones. Sin embargo, es importante tener cuidado con la complejidad y asegurarse de cubrir todos los estados y transiciones de manera coherente para evitar comportamientos impredecibles o inconsistentes.

En situaciones más complejas, es mejor utilizar alguna biblioteca o framework que permita manipular el estado de una aplicación para poder cubrir la coherencia, los eventos y la trazabilidad del proceso.

## [Github repository](https://github.com/nullpointer-excelsior/advanced-design-patterns-with-typescript/tree/master/src/state-machine)


🙂 Meme de cortesía:

![meme](https://i.ibb.co/hX25RZw/Screenshot-2023-06-07-at-12-01-32.png)