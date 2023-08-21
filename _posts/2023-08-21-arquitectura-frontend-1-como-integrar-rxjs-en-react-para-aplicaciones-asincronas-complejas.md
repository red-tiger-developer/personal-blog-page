---
title: Arquitectura Frontend 1. Cómo integrar RxJS en React para aplicaciones asíncronas complejas
author: Benjamin
date: 2023-08-21 12:48:40 -0500
categories: [Frontend, React, Rxjs, Arquitectura Frontend, Javascript, Typescript]
tags: [frontend, react, rxjs, arquitectura frontend, javascript, typescript]
---

![image](https://i.ibb.co/vcf5t0f/Screenshot-2023-08-21-at-14-12-34.png)

React es una excelente librería para crear componentes visuales. También nos brinda los hooks, con los cuales podemos trabajar con estados y efectos secundarios en situaciones asíncronas. Si bien para casos sencillos esto puede ser suficiente, manejar flujos asíncronos complejos puede hacer que nuestros componentes se conviertan en una ensalada de código. Afortunadamente, podemos integrar fácilmente RxJS para enfrentar estas situaciones. RxJS es una poderosa librería que nos permite trabajar con flujos asíncronos y eventos, permitiendo la reutilización de código y la creación de aplicaciones con alto flujo de datos y eventos.

Si no conoces sobre RxJS o la programación reactiva, te invito al siguiente [artículo](https://nullpointer-excelsior.github.io/posts/Ejemplo-programaci%C3%B3n-reactiva-100-Real-No-Fake/) donde se explica en qué consiste este enfoque. Ahora bien, si estás interesado en crear aplicaciones más robustas, puedes seguir leyendo este artículo.

## Custom Hooks para trabajar con RxJS

React nos permite crear nuestros propios hooks, lo que resulta en componentes más limpios y fáciles de mantener. Para ello, haremos uso de `useState()` y `useEffect()`, los clásicos de siempre.

### useObservable

Este hook nos permite trabajar con suscripciones de forma efectiva, sin que tengamos que preocuparnos por las fugas de memoria debido a la falta de desuscripción de los observables que estamos empleando.

```jsx
import { useEffect } from "react";
import { Observable, Observer } from "rxjs";

/**
 * Hook personalizado para suscribirse a un observable y ejecutar una acción cuando se emiten valores.
 * @template T El tipo de valor emitido por el observable.
 * @param {Observable<T>} observable$ El observable al que se desea suscribir.
 * @param {Partial<Observer<T>> | ((value: T) => void)} callback La función que se ejecutará cuando se emitan valores.
 */
export default function useObservable<T>(observable$: Observable<T>, callback: Partial<Observer<T>> | ((value: T) => void)) {
    
    useEffect(() => {
        // Se crea una suscripción al observable proporcionado
        const subscription = observable$.subscribe(callback);
        
        // La función de limpieza que se ejecutará al desmontar el componente
        return () => {
            // Se desuscribe de la suscripción para evitar fugas de memoria
            subscription.unsubscribe();
        };
    }, []);
}

```

### useObservableValue 

Este hook nos permite usar los valores emitidos por un observable. Podremos escuchar cada valor emitido, los errores y también cuando el observable se completa.

```jsx
import { useEffect, useState } from "react";
import { Observable } from "rxjs";

/**
 * Una tupla que representa el valor, el error y el estado de completado de un observable.
 * @template T El tipo de valor emitido por el observable.
 */
type ObservableValue<T> = [
    /** El valor actual emitido por el observable. */
    value: T,
    /** El error, si se produce alguno, o `false` si no hay error. */
    error: any,
    /** Un indicador de si el observable ha sido completado (`true`) o no (`false`). */
    isCompleted: boolean
];

/**
 * Hook para trabajar con valores emitidos por un observable.
 * @template T El tipo de valor emitido por el observable.
 * @param {Observable<T>} observable$ El observable del que se escucharán los valores.
 * @param {T} defaultValue El valor por defecto.
 * @returns {ObservableValue<T>} Una tupla con el valor actual, el error (si existe) y un indicador de si el observable está completado.
 */
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

## Cómo utilizar nuestros Hooks

Con estas sencillas implementaciones, analicemos los siguientes ejemplos.

### Eventos de teclado

```typescript
/**
 * Crea un observable que emite eventos de teclado filtrados por la tecla especificada.
 * @param {'arrowUp' | 'ArrowDown' | 'ArrowLeft' | 'ArrowRight'} key La tecla de flecha a escuchar.
 * @returns {Observable<KeyboardEvent>} El observable que emite eventos de teclado filtrados.
 */
const keydown$ = (key) => fromEvent(document, "keydown").pipe(
    filter((event: KeyboardEvent) => event.key === key), // filtramos por la tecla especifica.
    takeUntil(game$.gameOver()) // terminamos la emisión de valores hasta que el observable game$ termine
);

// Suscripción a eventos de teclado
useObservable(keydown$('arrowUp'), () => movePlayer('up'));
useObservable(keydown$('ArrowDown'), () => movePlayer('down'));
useObservable(keydown$('ArrowLeft'), () => movePlayer('left'));
useObservable(keydown$('ArrowRight'), () => movePlayer('right'));


```
Simulamos un juego donde escuchamos los eventos de las flechas de navegación y emitimos los eventos correspondientes.

Equivalente en React con hooks:

```jsx
import React, { useEffect } from 'react';

function App() {
  const movePlayer = (direction) => {
    // Lógica para mover al jugador en la dirección especificada
  };

  useEffect(() => {
    const handleKeyDown = (event) => {
      switch (event.key) {
        case 'ArrowUp':
          movePlayer('up');
          break;
        case 'ArrowDown':
          movePlayer('down');
          break;
        case 'ArrowLeft':
          movePlayer('left');
          break;
        case 'ArrowRight':
          movePlayer('right');
          break;
        default:
          break;
      }
    };

    document.addEventListener('keydown', handleKeyDown);

    return () => {
      document.removeEventListener('keydown', handleKeyDown);
    };
  }, []);

  return (
    <div>
      {/* Contenido de tu aplicación */}
    </div>
  );
}

export default App;
```
> Es importante mencionar que en ciertos escenarios, se puede simplificar este código utilizando el evento onKeyDown del componente deseado, sin necesidad de manipular el DOM directamente.

### Elegante y Coqueto StateManager

```typescript
export interface Book {
    title: string
    pages: number
    genre: string
    cover: string
    synopsis: string
    year: number
    ISBN: string
    author: string
  }

export class BookStoreState {

  constructor(private book$: BehaviorSubject<Book[]>) {}

  setBooks(books: Book[]): void {
    this.book$.next(books);
  }

  addBook(book: Book): void {
    const books = this.book$.getValue();
    if (books.find(b => b.ISBN === book.ISBN) === undefined) {
      this.book$.next([...books, book]);
    }
  }

  getBooks(): Observable<Book[]> {
    return this.book$.asObservable();
  }

  getAuthors(): Observable<string[]> {
    return this.book$.pipe(
      map(books => books.map(book => book.author)),
      switchMap(authors => of([...new Set(authors)]))
    );
  }
}

```

Podemos crear un state manager reactivo con un enfoque más orientado al dominio, ideal para aplicaciones frontend con lógicas de negocio complejas. Este enfoque no tiene nada que envidiarle a bibliotecas más elaboradas e incluso resulta más sencillo que utilizar Redux o Context de React.

Generamos un archivo con la instancia del estado:

```typescript
export const bookStoreState = new BookStoreState(new BehaviorSubject(books));
```

Y hacemos llamados en el componente para obtener un estado global:

```tsx

const books$ = bookStoreState.getBooks()
const authors$ = bookStoreState.getAuthors()

function BooksComponent() {
  
  const [books, booksError] = useObservableValue(books$, [])
  const [authors, autorsError] = useObservableValue(authors$, [])

  return (
    <div>
      {/* Contenido de tu aplicación */}
    </div>
  );
}

```

Si deseamos que este State Manager sea específico de un solo componente, podemos hacer uso de `useRef()`:

```tsx

function BooksComponent() {

  const bookStoreRef = useRef(new BookStoreState(new BehaviorSubject(books)))
  
  const [books, booksError] = useObservableValue(bookStoreRef.current.getBooks(), [])
  const [authors, autorsError] = useObservableValue(bookStoreRef.current.getAuthors(), [])

  return (
    <div>
      {/* Contenido de tu aplicación */}
    </div>
  );
}


```

Estos ejemplos de uso nos proporcionan componentes más simples de implementar, ya que trasladamos la lógica compleja fuera de los hooks. Esto nos brinda varias ventajas:

* Componentes visuales más limpios y fáciles de mantener.
* Lógica compleja de negocio reutilizable sin atarse a una librería o framework en particular.
* Flujos asíncronos reutilizables.
* Uso de variables reactivas y en tiempo real.
* Pruebas unitarias de lógicas de negocio más sencillas.

Por supuesto, hay algunas desventajas:

* Curva de aprendizaje alta para RxJS.
* Flujos complejos pueden generar observables difíciles de entender.
* Pruebas unitarias difíciles de crear con observables complejos, donde se necesita seguir minuciosamente el flujo.


# Conclusión

RxJS nos proporciona un enfoque avanzado para trabajar con flujos de datos, eventos y código reactivo. La facilidad de integración con React se debe a los hooks básicos proporcionados por esta librería. Con RxJS, podemos crear aplicaciones más complejas tanto en términos de código asíncrono como en el ámbito de lógicas de negocio.


![meme](https://i.ibb.co/XjjZFkX/Screenshot-2023-08-21-at-13-58-59.png)
    