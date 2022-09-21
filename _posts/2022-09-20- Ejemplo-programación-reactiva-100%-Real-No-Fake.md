---
title: Ejemplo programación reactiva 100% Real No Fake
author: Benjamin
date: 2022-09-20 18:32:00 -0500
categories: [Programacion, Angular, Rxjs]
tags: [rxjs, typescript, javascript, frontend, firebase, observables]
---

# Ejemplo programación reactiva 100% Real No Fake

La programación reactiva es un paradigma enfocado en el trabajo con flujos de datos finitos o infinitos de manera asíncrona. Su concepción y evolución ha ido ligada a la publicación del Reactive Manifesto, que establecía las bases de los sistemas reactivos, los cuales deben ser:

- Responsivos: aseguran la calidad del servicio cumpliendo unos tiempos de respuesta establecidos.
- Resilientes: se mantienen responsivos incluso cuando se enfrentan a situaciones de error.
- Elásticos: se mantienen responsivos incluso ante aumentos en la carga de trabajo.
- Orientados a mensajes: minimizan el acoplamiento entre componentes al establecer interacciones basadas en el intercambio de mensajes de manera asíncrona.

La motivación detrás de este nuevo paradigma procede de la necesidad de responder a las limitaciones de escalado presentes en los modelos de desarrollo actuales, que se caracterizan por su desaprovechamiento del uso de la CPU debido al I/O, el sobreuso de memoria (enormes thread pools) y la ineficiencia de las interacciones bloqueantes.

La arquitectura de estos sistemas se basa en los patrones observable, iterable y la integración del paradigma funcional. Todo esto crea una forma distinta de tratar los datos, ya que dejamos de verlos como variables y pasan a ser flujos de datos. Con el tiempo hemos pasado de tratar con una cantidad de datos ya definidos a fuentes de datos infinitas tales como mensajería en tiempo real, eventos de usuario, comunicación entre servicios en diversos protocolos, logs, Iot, etc.

# ¿El paradigma de programación imperativo solucionaba esto? 

¿Imagina tratar con flujo de datos que no sabes cuando termina o intermitente en el tiempo con nuestros clásicos  while data.exists() do some_operation() esto es lo más básico para tratar con datos, pero imaginate controlar si hay un error tendremos que agregar un try catch ahora si necesitamos reintentar el proceso? ¿Vamos a volver a recorrer nuestros datos? Ahora se quiere un nuevo proceso que haga cálculos con esa data, o duplicamos la lógica o la agrandamos, y si ahora los datos crecen y el algoritmo se necesita optimizar? Todo a la basura y a reimplementar, suena a complejidad y difícil de mantener.

Acordémonos del manifiesto reactivo y verás que el paradigma imperativo se vuelve complejo de usar en este contexto de sistemas.

El uso de programación reactiva no implica dejar nuestra amada clásica forma de programar es más como te había explicado la programación funcional juega un papel importante, lo que debemos tener claro es que la programación imperativa es ideal para crear funciones específicas con lógicas más pequeñas y con una responsabilidad única, no usemos esto para controlar procesos complejos cuando existen arquitecturas de software pensadas en resolver estos problemas. Con la separación de responsabilidades dentro de una solución de software con esto podemos lograr código más testeable y seguir arquitecturas como S.O.L.I.D.

# Se terminó la cháchara vamos a crear nuestra primera aplicación reactiva

En esta ocasión compartiré un pequeño ejemplo de uso de programación reactiva utilizando la librería RxJs donde crearemos un servicio que utiliza Firebase con las operaciones de lectura escritura y escucha de actualizaciones de datos. Usaremos Angular ya que es un framework full reactivo ideal para este ejemplo.

## ¿Qué es un observable?
Un observable es un objeto con el que podemos manipular un stream de datos estos pueden emir valores de forma infinita o finita de forma asíncrona y a su vez un observable nos proporciona una manera de poder estar a la escucha del flujo de datos poseen los siguientes métodos:

```bash

next() # es la llamada a emitir un valor dado
subscribe() # es un método que recibe la emisión de valores
complete() # nos índica que el observable termino de emitir valores y ya no volverá a hacerlo
error() # nos indica que un error ha ocurrido al emitir un valor

```

El uso basico de un observable es el siguiente:



```typescript

import { Observable } from 'rxjs';

const obs$ = new Observable<string>(subs => {
    subs.next("Hello");
    subs.next("World!!!");
});

obs$.subscribe(console.log);

```
nuestra salida en consola sera 

"Hello"

"World"

Si invocamos a complete()
 ```typescript

const obs$ = new Observable<string>(subs => {
    subs.next("Hello");
    subs.next("World!!!");
    subs.complete();
    subs.next("And Good Bye!!!")
});
obs$.subscribe(console.log);

```

La salida sería la misma que la anterior, ya que hemos completado el observable no emitirá más valores aunque llamemos a next()

## Creando un cliente firebase

Ahora crearemos un cliente Firebase utilizando la misma lógica. Definiremos una interfaz con las operaciones que nuestros casos de uso necesitan.

Hemos definido la interfaz DocumentProvider con operaciones de manipulación de documentos, Firebase no es la única solución serverless a sí que si en un futuro queremos enviar a la casa a Firebase podremos hacerlo sin complicaciones.

 ```typescript

import { Observable } from "rxjs";

export interface DocumentProvider {
    saveDocument(collectionName: string, entity: any): Observable<any>;
    onDocumentAdded(collectionName: string, entityAdapter: (data: any) => any): Observable<any>;
}

```

Implementamos nuestra interfaz y definimos sus métodos, agregamos la inicialización de Firebase en este caso usamos la mala práctica de inicializar un. Objeto dentro del constructor, porque se me da la gana y en este caso veo más óptimo guardar la lógica de Firebase en este componente, ya que la inyección de dependencias es más importante en los servicios core de la aplicación como los caso de uso. 
Implementaremos el método onDocumentAdded() ya que acá usaremos la clase Observable Rxjs.

 ```typescript

import { initializeApp } from "firebase/app"
import { Firestore, getFirestore } from "firebase/firestore"
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import { DocumentProvider } from './providers/document.provider';

initializeApp({
  apiKey: environment.firebase.apiKey,
  authDomain: environment.firebase.authDomain,
  projectId: environment.firebase.projectId
});

export class FirebaseClientService implements DocumentProvider {

  private db: Firestore = null;

  constructor() {
    this.db = getFirestore();
  }

  saveDocument(collectionName: string, entity: any) {
    
  }

  onDocumentAdded(collectionName: string, entityAdapter: (data: DocumentData) => any) {

    
  }

}

```

Lo primero es definir  el queryDocument para poder estar a la escucha de nuevos documentos agregados en Firebase.

Llamamos al objeto query y a su vez a collection de las librerías de Firebase le pasamos como parámetro el nombre de una colección y el objeto ya inicializado de FireStore, con esto obtenemos el queryDocument con el cual hacer operaciones sobre Firebase.
 ```typescript

 onDocumentAdded(collectionName: string, entityAdapter: (data: DocumentData) => any) {
    const queryDocument = query(collection(this.db, collectionName));
 }

```

Ahora definimos las siguientes funciones callback para crear nuestro objeto observable.
Estas funciones son necesarias por Firebase para poder crear una escucha en tiempo real de las actualizaciones de los documentos  mediante una función onSnapshot() propia de Firebase
 ```typescript

    onDocumentAdded(collectionName: string, entityAdapter: (data: DocumentData) => any) {
        
        const queryDocument = query(collection(this.db, collectionName));

        const onSnapshotNext = (snapshot: QuerySnapshot<DocumentData>, observer: Observer<any>) => {
        
        }

        const onSnapshotError = (error: FirestoreError, observer: Observer<any>) => {
        
        }

        const onSnapshotComplete = (observer: Observer<any>) => {

        }

    }

```
onSnapshotNext(): Esta función llama a observer.next() cuando recibe un documento de Firebase y con esto nuestro observable emitirá el valor obtenido.
Cuando nos ponemos en escucha a los cambios de un documento en Firebase con snapshot.docChanges() recibimos un objeto de tipo QuerySnapshot el cual nos permite realizar operaciones para obtener una respuesta más limpia adecuada a nuestras necesidades.

así que haremos lo siguiente:

- filtraremos por él el change.type de "added"
- haremos un map() a la propiedad "data"
- usaremos la función callback entityAdapter() para convertir los datos recibidos de Firebase a un objeto de tipo Entidad de nuestra aplicación. Con esto centralizamos la inicialización de una entidad en un único punto
- Por cada documento agregado llamamos a observer.next(entityValue)

 ```typescript

const onSnapshotNext = (snapshot: QuerySnapshot<DocumentData>, observer: Observer<any>) => {
      snapshot.docChanges()
        .filter(change => change.type === 'added')
        .map(change => change.doc.data())
        .map(data => entityAdapter(data))
        .forEach(entity => observer.next(entity))
    }

```
La implementación de observer.error() y observer.complete() son las siguientes:
 ```typescript

    const onSnapshotError = (error: FirestoreError, observer: Observer<any>) => {
      observer.error(error)
    }

    const onSnapshotComplete = (observer: Observer<any>) => {
      observer.complete()
    }

```
Ahora definimos nuestro objeto Observable y dentro de su función callback llamamos a onSnapshot() quien es el encargado de poder tener nuestros documentos de Firebase en tiempo real, a esta función le debemos pasar como parámetros las funciones que definimos onSnapshoNext() onSnapshotError() y onSnapshotComplete() y con esto ya tendríamos la posibilidad de suscribirnos a nuevos documentos agregados.
 ```typescript

 const documentAdded$ = new Observable((observer: Observer<any>) => {
    // función Firebase para poder estar a la escucha de eventos en tiempo real
    onSnapshot(
        queryDocument, // collecion de documentos a escuchar por nuevos cambios
        (snapshot) => onSnapshotNext(snapshot, observer), // funcion callback para emitir valores
        err => onSnapshotError(err, observer), // error
        () => onSnapshotComplete(observer) // termino de emision de eventos
    );
})

```
Finalmente devolvemos nuestro observable, pero fíjense en que el observable lo devolvemos con un llamado a una función llamada pipe()
y ¿para qué sirve pipe? Pipe nos sirve para poder encadenar operadores o funciones que se ejecutaran sobre los flujos de datos de nuestro observable y en esta ocasión lo usamos para agregar el operador share() es cuál nos permite que el observable que retorna es único para todos los que se suscriban a él, si share() no estuviera cada vez que un cliente se suscriba a Firebase se crearía una nueva conexión a Firebase lo cual no es óptimo en este caso, a posibilidad de pipe es que nos puede devolver nuevos observables a partir de otros. Que maravilla lógicas complejas pueden ser reutilizables.

 ```typescript

return documentAdded$.pipe(
      share()
)

```

## Otra forma de crear observables

Ahora implementaremos el método saveDocument() este deberá guardar un documento en Firebase y devolver un Observable para crear observables Rxjs nos facilita operadores creacionales para facilitar este proceso, no siempre crear un Observable a la vieja escuela es lo ideal, pero a veces debemos hacerlo en esta ocasión usaremos el operador from() con el cual podemos crear un observable a partir de una promesa y así de simple cuando necesitamos observables a partir de primitivos u objetos. Existen más operadores creacionales, pero no se abordan acá.

 ```typescript

saveDocument(collectionName: string, entity: any) {
    const promise = setDoc(doc(this.db, collectionName, entity.id), entity)
    return from(promise).pipe(
      map(() => entity)
    )
}

```
Finalmente nuestro servicio queda de la siguiente manera:
 ```typescript

import { initializeApp } from "firebase/app"
import { doc, setDoc, DocumentData, Firestore, FirestoreError, getDoc, getFirestore, QuerySnapshot, collection, query, onSnapshot } from "firebase/firestore"
import { from, map, Observable, Observer, share } from 'rxjs';
import { environment } from '../../../environments/environment';
import { DocumentProvider } from './providers/document.provider';


initializeApp({
  apiKey: environment.firebase.apiKey,
  authDomain: environment.firebase.authDomain,
  projectId: environment.firebase.projectId
});

export class FirebaseService implements DocumentProvider {

  private db: Firestore = null;
  constructor() { this.db = getFirestore(); }

  saveDocument(collectionName: string, entity: any) {
    const promise = setDoc(doc(this.db, collectionName, entity.id), entity)
    return from(promise).pipe(
      map(() => entity)
    )
  }

  onDocumentAdded(collectionName: string, entityAdapter: (data: DocumentData) => any) {

    const queryDocument = query(collection(this.db, collectionName));

    const onSnapshotNext = (snapshot: QuerySnapshot<DocumentData>, observer: Observer<any>) => {
      snapshot.docChanges()
        .filter(change => change.type === 'added')
        .map(change => change.doc.data())
        .map(data => entityAdapter(data))
        .forEach(entity => observer.next(entity))
    }

    const onSnapshotError = (error: FirestoreError, observer: Observer<any>) => {
      observer.error(error)
    }

    const onSnapshotComplete = (observer: Observer<any>) => {
      observer.complete()
    }

    const documentAdded$ = new Observable((observer: Observer<any>) => {
      onSnapshot(
        queryDocument,
        (snapshot) => onSnapshotNext(snapshot, observer), 
        err => onSnapshotError(err, observer), 
        () => onSnapshotComplete(observer));
    })

    return documentAdded$.pipe(
      share()
    )
    
  }

}

```

Uso de FirebaseService

Inyección de FirebaseService en el módulo de la aplicación. esto es importante ya que definimos una interfaz como servicio. no utilizamos el decorador.

 ```typescript

import { InjectionToken, NgModule } from '@angular/core';
import { AppComponent } from './app.component';

export const DOCUMENT_SERVICE = new InjectionToken<DocumentProvider>('app.document.service');

@NgModule({
  declarations: [
    AppComponent,
  ],
  imports: [
    RouterModule.forRoot(routes),
    RegisterModule
    AppRoutingModule,
    SharedModule,
  ],
  providers: [{
    provide: DOCUMENT_SERVICE,
    useClass: FirebaseService
  }],
  bootstrap: [AppComponent]
})
export class AppModule { }

  ```

Uso de FirebaseService 

 ```typescript

@Injectable({
  providedIn: 'root'
})
export class MessageService {

  constructor(@Inject(DOCUMENT_SERVICE) private firebase: DocumentProvider) { }

  sendMessage(msg: Message){
    return this.firebase.saveDocument('messages', msg)
  }

  getRealtimeMessages(): Observable<Message> {
    
    const adapter = (data: any): Message => ({ 
      id: data['id'],
      message: data['message'],
      createdAt: data['createdAt'],
      images: data['images']
    })

    return this.firebase.onDocumentAdded('messages', adapter).pipe(
      filter(attack => attack.message !== null)
    )

  }

}

  ```

  ## Conclusión

  Aca no hay concluciones hay memes.

  ![meme](https://i.ibb.co/1JFL4MC/memejavascript.jpg)