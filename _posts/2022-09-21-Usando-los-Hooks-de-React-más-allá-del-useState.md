---
title: Usando los Hooks de React más allá del useState
author: Benjamin
date: 2022-09-21 18:32:00 -0500
categories: [Programacion, React, Javascript ]
tags: [typescript, javascript, frontend, hooks]
---

# Usando los Hooks de React más allá del useState


### Hack 1 - Petición asíncrona con useState() y useEffect() juntos 

El siguiente componente muestra los datos obtenidos mediante una operación asíncrona como lo puede ser una petición hacia alguna API.

Se define el Hook useState() para almacenar el estado del componente y se define el Hook useEffect() para poder implementar la lógica de actualización del estado. useEffect() recibe una función la cual será la operación deseada a realizar y recibe un array de dependencias el cual si cualquier valor del array cambia el Hook se gatilla y ejecuta la función definida. Como no existe aún alguna dependencia debemos agregar un array vacio, ya que de lo contrario el Hook no funcionara correctamente.

```typescript

import React, { useEffect, useState } from 'react';
import getPieChartData from '../../../core/application/getPieChartData';
import { PieChartData } from '../../../core/domain/model/PieChartData';
import PieChart from '../../components/PieChart';

export default function PieChartFromAPI() {
    
    const [data, setData] = useState<PieChartData | null>(null)

    useEffect(() => {
        getPieChartData()
            .then(pieChart => setData(pieChart))
    }, [])

    return <PieChart data={data} />

}
```

Falsearemos la petición asíncrona con un retraso de 1 segundo con la siguiente función.

```typescript
const awaitTimeout = (delay: number) => new Promise(resolve => setTimeout(resolve, delay));
```

Crearemos nuestra función que simula una API.

```typescript

import { PieChartData } from "../domain/model/PieChartData";
import { awaitTimeout } from "../libs/awaitTimeout";

const PIE_DATA: PieChartData = {
    labels: ['Red', 'Blue', 'Yellow', 'Green', 'Purple', 'Orange'],
    datasets: [
        {
            label: '# of Votes',
            data: [12, 19, 3, 5, 2, 3],
            backgroundColor: [
                'rgba(255, 99, 132, 0.2)',
                'rgba(54, 162, 235, 0.2)',
                'rgba(255, 206, 86, 0.2)',
                'rgba(75, 192, 192, 0.2)',
                'rgba(153, 102, 255, 0.2)',
                'rgba(255, 159, 64, 0.2)',
            ],
            borderColor: [
                'rgba(255, 99, 132, 1)',
                'rgba(54, 162, 235, 1)',
                'rgba(255, 206, 86, 1)',
                'rgba(75, 192, 192, 1)',
                'rgba(153, 102, 255, 1)',
                'rgba(255, 159, 64, 1)',
            ],
            borderWidth: 1,
        },
    ],
};

export default function getPieChartData(): Promise<PieChartData> {
    return awaitTimeout(1200)
        .then(() => PIE_DATA)
}

```

Nuestra aplicación funciona perfecto, pero esa demora implementada debemos hacerla notar mediante un aviso para nuestros usuarios agregaremos un nuevo estado de la siguiente manera.

```typescript

    const [loading, setLoading] = useState(false)

    useEffect(() => {
        setLoading(true)
        getPieChartData()
            .then(pieChart => setData(pieChart))
            .finally(() => setLoading(false))
    }, [])

    if (loading) {
        return <div>Loading motherfucking data...</div>
    }

```

Perfecto muchachos hemos usado esta pareja explosiva con un ejemplo práctico que de seguro los verás muy a menudo, ahora te haré la siguiente pregunta ¿qué pasaría si necesitamos manejar estados más complejos? ¿Qué pasaría si nuestro componente tiene una gran cantidad de valores, objetos primitivos o arreglos, y por último ¿y si un valor dependiera de otro? La solución simple es usar un objeto con useState() pero no siempre será la mejor solución especialmente cuando React usa el concepto de inmutabilidad para realizar los renderizados de componentes un objeto o un arreglo en Javascript no es un objeto completamente inmutable y debes tener ciertas precauciones cuando actualizas un arreglo o un objeto para que se gatille un useEffect() o un renderizado. Bueno en el siguiente ejemplo emplearemos el mismo ejemplo despachamos para la casa a useState() y le daremos la bienvenida a useReducer()  te parece conocido el nombre verdad si has usado Redux o la función reducer sabrás de lo que hablo

## Hack 2 -  Petición asíncrona con useReducer y useEffect

Tomando como base el hack anterior vamos a reemplazar el Hook useState() por useReducer() y usando sus valores como estado para mostrar los datos del gráfico del ejemplo anterior agregaremos unos cuantos valores más para poder ver el patrón reducer el cual lo podemos encontrar en aplicaciones con manejo de estado mediante Redux.

### Como diablos funciona el patron reducer

EL patrón reducer básicamente es definir una función reductora la cual recibe el estado actual de los datos y el nuevo estado que queremos definir entonces nuestra función reductora puede definir cálculos o lógicas a partir de estos 2 valores y retornar un nuevo estado. Si vamos a la programación funcional el típico ejemplo es sumar el valor anterior y el nuevo valor para generar una sumatoria entonces el resultado sería un acumulado de los datos. Ahora veamos como se implementa en React.
En React el Hook useReducer() recibe una función reductora la cual recibe 2 parámetros el estado actual del componente y un acción el cual tiene las siguientes propiedades type y payload donde type es un nombre de acción único nos dice que debe hacer el reducer y por último payload sería el nuevo estado o valores que usara nuestra función reductora. Lo sé esto suena enredado, pero vamos al maldito ejemplo.

Definiremos el modelo de nuestro reducer.

```typescript
// estado del componente
interface State {
    title: string;
    data: PieChartData | null;
    quantity: number;
    loading: boolean;
}
// modelo de la accion
interface Action {
    type: Type;
    payload?: any;
}
// tipos de aciones que puede realizar nuestra funcion reductora
enum Type {
    SET_DATA,
    LOADING,
}
```

Muy bien ahora esta es la implementación de la función reductora:

```typescript
const reducer = (state: State, action: Action) => {
    switch (action.type) {
        case Type.LOADING:
            return {
                ...state,
                loading: true
            }
        case Type.SET_DATA:
            const payload = action.payload
            return {
                ...state,
                data: payload,
                loading: false, 
                quantity: 100
            }
    }
}
```
Como ves la función recibe el estado y una acción entonces mediante un case realizaremos operaciones sobre el estado anterior y el payload suministrado como puedes apreciar no siempre necesitaremos definir un payload.

Ahora llamaremos el hook useReducer() de la siguiente manera
```typescript
const [state,  dispatch] =  useReducer(reducer, {
    data: null,
    quantity: 0,
    title: 'my fucking chart',
    loading: false,
})
```
useReducer() recibe la función reductora y un estado inicial y nos devuelve un arreglo con el estado del componente y una función llamada dispatch() esto es muy parecido a useState() con la diferencia que la función dispatch() recibe un objeto Action el cual definimos anteriormente.

Ahora para realizar cambios al estado nada usamos la funcion dispatch().

```typescript
dispatch({ type: Type.LOADING })
    getPieChartData()
        .then(pieChart => dispatch({ type: Type.SET_DATA, payload: pieChart }))
```

Vemos el primer llamado a la acción LOADING el cual cambia el estado y asigna a la variable loading el valor true y finalmente al recibir data desde nuestra petición asíncrona usamos la acción SET_DATA y el detalle es que no necesitamos una accion como STOP_LOADING para volver a asignar la variable a false, ya que lo hacemos en el propio reducer.

Terminando nuestro Hack component queda así. 
 
```jsx
import React, { useEffect, useReducer } from 'react';
import getPieChartData from '../../../core/application/getPieChartData';
import { PieChartData } from '../../../core/domain/model/PieChartData';
import PieChart from '../../components/PieChart';

interface State {
    title: string;
    data: PieChartData | null;
    quantity: number;
    loading: boolean;
}

interface Action {
    type: Type;
    payload?: any;
}

enum Type {
    SET_DATA,
    LOADING,
}

const reducer = (state: State, action: Action) => {
    switch (action.type) {
        case Type.LOADING:
            return {
                ...state,
                loading: true
            }
        case Type.SET_DATA:
            const payload = action.payload
            return {
                ...state,
                data: payload,
                loading: false, 
                quantity: 100
            }
    }
}

export default function PieChartUseReducerUseEffect() {
    
    const [state,  dispatch] =  useReducer(reducer, {
        data: null,
        quantity: 0,
        title: 'my fucking chart',
        loading: false,
    })

    const { loading, data, quantity } = state

    useEffect(() => {

        dispatch({ type: Type.LOADING })
        getPieChartData()
            .then(pieChart => dispatch({ type: Type.SET_DATA, payload: pieChart }))

    }, [])

    if (loading) {
        return <div>Loading motherfucking data...</div>
    }

    return (
        <>
            <h1>Total de datos {quantity}</h1>
            <PieChart data={data} />
        </>
    )

}
```

Terminamos con nuestro ejemplos de uso de useState() vs useReducer() ahora podemos tener una noción de cuando usar useReducer() en vez del clásico useState() esto dependerá de la complejidad del componente y el modelo de datos del estado. También debemos tener la sabiduría para dividir el problema en vez de crear componentes enormes llenos de Hooks y lógicas complejas en modelos de datos enormes.




### Hack 3 - Tabla con paginado desde una API 

Ahora vamos a crear un ejemplo con la simulación de un llamado a una API con paginado usando la misma lógica anterior, pero con la maldita diferencia que usaremos el array de dependencias el Hook useEffect() para detectar la petición del usuario de cambiar de página. La misma idea puede ser aplicada para un filtro dinámico donde se necesite ejecutar la actualización de la data mediante la interacción del usuario. Crearemos nuestro nuevo caso de uso getUsersPaginated() con el siguiente código

```typescript

interface Params {
    page: number;
    size: number;
}

interface Response {
    users: User[];
    total: number;
}

export default function getUsersPaginated({ page, size }: Params): Promise<Response>{
    return awaitTimeout(1000)
        .then(() => USERS)
        .then(users => users.slice((page - 1) * size, page * size))
        .then(users => ({ users, total: USERS.length }));
}
```

La lista de usuarios la transformamos a una constante y nos ayudamos del encadenamiento de promesas para un código más simple a la vista y nos ayudamos del método slice() para el paginado.

Ahora implementamos nuestro componente con paginado queda de la siguiente manera

Definimos nuestros nuevos estados para el paginado. Las propiedades importantes acá son page (página actual de la tabla) y total (tamaño de todos los datos) las cuales serán las que irán mutando mediante las peticiones del usuario, size también puede mutar si se implementa el selector de tamaño por páginas en este caso será omitido

```typescript

const [page, setPage] = useState(2)
const [size, setSize] = useState(3)
const [total, setTotal] = useState(0)

```

A continuación definimos nuestro useEffect(). El cual invoca nuestro caso de uso getUsersPaginated() y recibe los parámetros de page y size definiendo la porción de datos a buscar desde nuestra API simulada.
Al recibir una respuesta modificamos el estado de los datos de la tabla y el estado total el cual nos indica el total de registros existente en la API, finalmente definimos las dependencias del Hook que indican que si esas dependencias cambian el Hook useEffect() debe ejecutarse estas variables  [page, size] serán actualizadas mediante las peticiones del usuario

```typescript

useEffect(() => {
    setLoading(true)
    getUsersPaginated({ page, size })
        .then(res => {
            setDatatable(res.users)
            setTotal(res.total)
        })
        .finally(() => setLoading(false))
}, [page, size])

```
El uso importante de useEffect es definir correctamente las dependencias que harán que se gatille el efecto estas dependencias podrían ser valores de un filtro con lo cual lograríamos crear una funcionalidad de consulta de datos 

Ahora definimos la función que ejecutar el usuario al cambiar el paginado de los datos. Es una simple función callback que actualizara el estado de la variable page y esta función callback lo recibirá el componente de paginado que creemos

```typescript

const onPageChange = (value: number) => {
    setPage(value)
}

```
No entraremos en detalle de la implementación de la tabla y el paginado, ya que estamos estudiando los Hooks y esta implementación estará presente el repositorio

```jsx

<Container maxWidth="md">
    <h1>Data from async api</h1>
    <UserTable
        datatable={datatable}
        page={page}
        size={size}
        total={total}
        onPageChange={onPageChange} />
</Container>
```

Nuestro componente final seria el siguiente:

```jsx

import React, { useEffect, useState } from 'react';
import { Container } from '@mui/material';
import getUsersPaginated from '../core/application/getUsersPaginated';
import { User } from '../core/domain/model/User';
import UserTable from '../components/UserTable';

export default function DatatableFromAPIPaginated() {

    const [datatable, setDatatable] = useState([] as User[])
    const [loading, setLoading] = useState(false)
    const [page, setPage] = useState(2)
    const [size, setSize] = useState(3)
    const [total, setTotal] = useState(0)

    useEffect(() => {
        setLoading(true)
        getUsersPaginated({ page, size })
            .then(res => {
                setDatatable(res.users)
                setTotal(res.total)
            })
            .finally(() => setLoading(false))
    }, [page, size])

    const onPageChange = (value: number) => {
        setPage(value)
    }

    if (loading) {
        return <div>Loading data...</div>
    }

    return (
        <Container maxWidth="md">
            <h1>Data from async api</h1>
            <UserTable
                datatable={datatable}
                page={page}
                size={size}
                total={total}
                onPageChange={onPageChange} />
        </Container>
    )
}

```

Nuestro componente esta de putas maravillas funciona correctamente en su funcionalidad básica, pero en un componente real debemos tener un control de errores el cual puede ser incluso una variable de estado al igual que loading implementando la misma estrategia, pero dejame decirte que existen opciones más desacopladas como usar interceptores en las peticiones y controlar desde ahí los errores, pero eso es mejor explicarlo en otra instancia con mas detalle.

Nuestro siguiente paso será refactorizar el componente con un Hook personalizado.

Este componente es pequeño y nuestro deber es seguir manteniéndolo así para que nuestra aplicacion sea mantenible y resiliente al cambio esto nos suena a S.O.L.I.D. verdad, en el frontend se olvidan a menudo de este concepto principalmente el de RESPONSABILIDAD ÚNICA así que vamos a aplicar un reactor.

Bienvenido Custom Hook

### Hack 4 - Tabla con paginado con custom hook

En nuestro componente anterior utilizamos un montón de useState() y useEfffect() para crear una API mínima de usuarios, ahora moveremos toda la lógica de estos Hooks a uno personalizado.

Para crear un Hook personalizado solo basto con mover toda la lógica de los Hooks usados en una simple función, esta función puede recibir parámetros y devolver valores de estado o funciones que puedan realizar código definido dentro del Hook creado entonces el código lo movemos a la siguiente función


```typescript

import { useEffect, useState } from "react"
import getUsersPaginated from "../../core/application/getUsersPaginated"
import { User } from "../../core/domain/model/User"

interface Hook {
    users: User[];
    loading: boolean;
    page: number;
    size: number;
    total: number;
    requestGetUsers: (page: number, size: number) => void;
}

export default function useGetUsers(): Hook {
    
    const [users, setUsers] = useState([] as User[])
    const [loading, setLoading] = useState(false)
    const [page, setPage] = useState(2)
    const [size, setSize] = useState(3)
    const [total, setTotal] = useState(0)

    useEffect(() => {
        setLoading(true)
        getUsersPaginated({ page, size })
            .then(res => {
                setUsers(res.users)
                setTotal(res.total)
            })
            .finally(() => setLoading(false))
    }, [page, size])

    const requestGetUsers = (page: number, size: number) => {
        setPage(page)
        setSize(size)
    }

    return {
        users,
        loading,
        page,
        size,
        total,
        requestGetUsers
    }
}
```

y finalmente nuestro componente queda de esta forma

```jsx

import React from 'react';
import { Container } from '@mui/material';
import UserTable from '../../components/UserTable';
import useGetUsers from '../../hooks/useGetUsers';

export default function DatatablePaginatedCustomHook() {

    const { users, page, total, size, loading, requestGetUsers } = useGetUsers()

    const onPageChange = (value: number) => {
        requestGetUsers(value, size)
    }

    if (loading) {
        return <div>Loading data...</div>
    }

    return (
        <Container maxWidth="md">
            <h1>Data from async api</h1>
            <UserTable
                datatable={users}
                page={page}
                size={size}
                total={total}
                onPageChange={onPageChange} />
        </Container>
    )
}

```

Una gran diferencia con nuestro componente anterior. Este es mucho más limpio, con lógica encapsulada, separamos las responsabilidades y podemos escalar de mejor manera

## Hack 5 Custom Hook para cancelar peticiones http

Muchass veces hemos perdido la paciencia con páginas lentas y nos ponemos en modo loco y a darle al click como 20 mil veces como si eso solucionar el asunto y la página la muy cabrona se vuelve más lenta. Cuantas veces nos ha pasado y me imagino como desarrolladores cuantas veces hemos creado páginas con el mismo problema en donde en cierto comportamiento ocurre esto

Bueno en este post vamos a abordar una solución a uno de los problemas que causa esto.

## Como manejar peticiones http cancelables

Cuando lanzamos una petición http al servidor desde un cliente generalmente el "happy path" es realizar la petición recibir los datos mostrarlos y si existe un error tratarlo pero muchas veces existen escenarios donde una petición http puede ser lanzada múltiples veces sin que el desarrollador haya predicho que el usuario haciendo ciertas acciones lograría mandar un montón de peticiones entonces el backend puede quedar en espera tramitando la petición dando la sensación de que estamos en un Internet Explorer generalmente para evitar eso se bloquean botones o se pone el típico loading, pero ojo que hay interacciones complejas en el frontend donde no todos los bloqueos o validaciones cubrirán los casos

Un caso típico es el de las pestañas o un menú de navegación en una aplicación, imaginate crear un menú y que cuando el usuario hace click en un módulo del menú empieza a cargarse el módulo, pero después el usuario hace click en otro módulo y empieza otra petición y así puede hacerlo si la página es lenta lo más probable que se ponga en modo loco y haga click por todos lados en estos caso lo ideal es que las peticiones se puedan cancelar entonces cada vez que cambie de módulo o haga click la petición que estaba en curso sea cancelada de lo contrario veríamos una respuesta más lenta en servidores que no aguanten mucha carga por el lado de React también puede ocurrir un warning de que no podemos actualizar el estado en un componente desmontado lo cual ocurre porque una petición se procesó y el resultado se quiere escribir en un componente inexistente porque ahora se está tratando de cargar otro, React nos advierte que puede ser un posible "memory leak" y es mejor evitarlo así que vamos a crear el maldito ejemplo que es mucha cháchara para un par de líneas de código

## El caso de ejemplo

Crearemos un módulo que muestra las aves chilenas en esta ocasión están no nombre en inglés, la funcionalidad más ridícula que se me ocurrió, pero sirve para mostrar el problema

Instalaremos axios para manejar las peticiones http

```bash
npm install axios
```

Ahora creamos la estructura de la respuesta de la API y la función fecthAllChileanBirds() que hace un llamado a la api [Aves ninja](https://aves.ninjas.cl/api/birds)

```typescript
import axios, { AxiosResponse } from 'axios';

export interface ChileanBirdResponse {
    uid: string;
    name: {
        spanish: string;
        english: string;
        latin:   string;
    };
    images: {
        main:  string;
        full:  string;
        thumb: string;
    },
    _links: {
        self:   string;
        parent: string;
    },
    sort: number;
}

export function fecthAllChileanBirds(): Promise<AxiosResponse<ChileanBirdResponse[]>> {
    return axios.get('https://aves.ninjas.cl/api/birds')
}

```

Ahora crearemos nuestro caso de uso que hará un llamado a la API y nos devolverá las aves chilenas con su nombre en ingles y su nombre en latín para que nuestro post parezca más intelectual me vea la nasa y me saque de Latinoamérica

Modelo
```typescript
export interface Bird {
    id: string;
    name: string;
    latinname: string;
}
```
Función Adapter
```typescript
import { ChileanBirdResponse } from "../../api/ChileanBirdsApi";
import { Bird } from "../model/Bird";

export function ChileanBirdToEnglishBirdAdapter(response: ChileanBirdResponse): Bird {
    return {
        id: response.uid,
        name: response.name.english,
        latinname: response.name.latin,
    }
}
```

Caso de uso 
```typescript
import { fecthAllChileanBirds } from "../api/ChileanBirdsApi";
import { ChileanBirdToEnglishBirdAdapter } from "../domain/adapter/ChileanBirdToEnglishBirdAdapter";
import { Bird } from "../domain/model/Bird";

export default function getEnglishNameBirds(): Promise<Bird[]> {
    return fecthAllChileanBirds()
        .then(res => res.data)
        .then(birds => birds.map(b => ChileanBirdToEnglishBirdAdapter(b)))
}
```

Ojo nótese la función ChileanBirdToEnglishBirdAdapter() la cual es usada como patrón Adapter donde recibirá un objeto externo al "Dominio" o lógica "core" de la aplicación y nos devolverá un objeto de tipo Modelo o entidad la cual será usado por nuestros casos de usos, ¿que ventajas hay usando esto? Bueno principalmente desacopla él la lógica de negocio frente a implementaciones u objetos externos los cuales podrían ser una respuesta de una API o cualquier implementación de una librería de terceros imagina que después cambiemos la API de aves chilenas por otra cuya respuesta sea distinta entonces centralizamos el cambio en un solo punto que sería la función adaptadora.

## Hook personalizado para funciones asíncronas

Ahora crearemos un Hook personalizado para hacer uso de funciones asíncronas con lo cual usaremos para obtener las aves chilenas con su nombre en ingles y pintarlos sobre un componente de React

```jsx
import { useEffect, useState } from "react"

type Callback<T> = () => Promise<T>

interface Hook<T> {
    loading: boolean;
    data: T | undefined;
    error: any;
}

export default function useAsync<T>(callback: Callback<T>): Hook<T>{
    
    const [data, setData] = useState<T>()
    const [loading, setLoading] = useState(false)
    const [error, setError] = useState(null)

    useEffect(() => {
        setLoading(true)
        callback()
            .then(response => setData(response))
            .catch(err => setError(err))
            .finally(() => setLoading(false))

    }, [])

    return {
        loading,
        data,
        error
    }
}
```

Este Hook no tien mayor gracia usa el useState() y useEffect() de toda la vida tiene un loading un estado de posibles errores. Ahora crearemos otro componente que reflejará el caso de uso

```jsx
import getEnglishNameBirds from "../../../../../core/application/getEnglishNameBirds";
import { Bird } from "../../../../../core/domain/model/Bird";
import useAsync from "./useAsync";

export default function useEnglishBirds() {
    return useAsync<Bird[]>(getEnglishNameBirds)
}
```
Este hook importa la función getEnglishNameBirds() como caso de uso y la agrega como parámetro al Hook useAsync() y es todo 


## El componente de Aves en inglés


```jsx
import React from 'react';
import useEnglishBirds from '../hooks/useEnglishBirds';
import BirdContainer from '../../../shared/components/BirdContainer';

export default function EnglishTranslateBirds() {
    
    const { loading, data } = useEnglishBirds()

    return (
        <BirdContainer datatable={data} loading={loading} />
    )

}
```

Ahora crearemos el componete que hará uso del hook useEnglishBirds() para mostrar las aves chilenas en ingles la implementación del componente es lo de menos, ya que es una clase de Hooks


Nuestro frontend luce así

FOTO_FRONT

Ahora si hacemos múltiples clicks sobre el menú de aves y sobre el cualquier otro

FOTO_MENU Y CILCK

Ahora vamos a inspeccionar página y nos vamos a networking y si hacemos múltiples clicks sobre el menú de aves y sobre el cualquier otro obtendremos las siguientes peticiones

FOTO_NETWORKING

Como ves quedaron pendientes muchas peticiones y todas terminaran, pero es total ineficiente y más cuando hay apis que son síncronas que lanzan un hilo nuevo por cada petición necesitamos cancelar las petición. Para lograr esto necesitamos crear un objeto AbortController propio de la API nativa del navegador esta solución que te presentaré funciona tanto en axios como con la Fetch opción nativa del navegador para hacer peticiones.

Cambiaremos la implementación de la petición de las aves chilenas agregando a la petición de axios un parámetro llamado signal el cual es de tipo AbortSignal con el cual axios se dará cuenta de qué la petición debe ser cancelada

```typescript

export function fecthCancellableChileanBirds(signal?: AbortSignal): Promise<AxiosResponse<ChileanBirdResponse[]>> {
    return axios.get('https://aves.ninjas.cl/api/birds', { signal })
}

```
Nuestro caso de uso ahora no solo devolverá una petición para obtener las aves sino que también devolverá una función para cancelar la petición, esta vez obtendremos las aves chilenas en idioma español


Definimos la estructura de la respuesta del caso de uso
```typescript

interface GetSpanishBirdServices {
    fetchBirds: () => Promise<Bird[]>;
    cancelFetchBirds: () => void;
}

```

Y ahora definimos la lógica de la petición y de cancelación

```typescript

export default function getSpanishBirdsServices(): GetSpanishBirdServices {
    
    const controller = new AbortController()
    
    const request = () => fecthCancellableChileanBirds(controller.signal)
        .then(res => res.data)
        .then(birds => birds.map(b => ChileanBirdToSpanishBirdAdapter(b)))

    return {
        fetchBirds: request,
        cancelFetchBirds: () => controller.abort()
    }

}

```
Ya ahora nuestro caso de uso devuelve un objeto que simula un típico componente "Service"

## Entendiendo el ciclo de vida en React

Es momento de entender el ciclo de vida de los Hooks en React para poder implementar nuestra cancelación de petición cuando usamos el Hook useEffect() este recibe una función callback que se ejecutara cuando se actualiza alguna dependencia definida en un array definido como segundo parámetro del Hook en la mayoría de los casos necesitamos solo definir una función que haga una acción y termine ahí, pero la función callback que definimos en useEffect puede devolver un función y ¿para qué rayos sirve esta función? Esta función es llamada "cleanup" y se ejecuta cada vez que el componente se desmonta es ideal para ejecutar acciones como es el cerrado de una conexión por websocket por ejemplo o un clearInterval si se usa la función useInterval() en Hooks entonces esta función es ideal para nuestro propósito de cancelar un petición http

Entonces nuestro Hook quedara de la siguiente manera

```jsx

import { useEffect, useState } from "react"

interface Hook<T> {
    loading: boolean;
    data: T | undefined;
    error: any;
}

export interface CancellableRequest<T> {
    execute: () => Promise<T>;
    cancel: () => void;
}

export default function useCancelableAsync<T>(cancellablerequest: CancellableRequest<T>): Hook<T>{
    
    const { execute, cancel } = cancellablerequest;

    const [data, setData] = useState<T>()
    const [loading, setLoading] = useState(false)
    const [error, setError] = useState(null)
   
    useEffect(() => {
        
        setLoading(true)
        
        execute()
            .then(response => setData(response))
            .catch(err => setError(err))
            .finally(() => setLoading(false))

        return () =>{
            cancel()
        }
        
    }, [])

    return {
        loading,
        data,
        error
    }
}

```

Los únicos cambios son las propiedades de nuestro Hook serán la petición asíncrona que debe realizar una función que cancel la operación así quienes implementen el uso de hook se preocuparan de realizar las operaciones de limpieza del hook finalmente en el useEffec() la función callback definida devolverá una función que se encargara de realizar la cancelación de la petición

## useAsyncCancellable en acción 

Ahora definimos el componente que usara nuestro custom Hook

```jsx

import React from 'react';
import BirdContainer from '../../../shared/components/BirdContainer';
import getSpanishBirdsServices from '../../../../../core/application/getSpanishNameBirdsCancellable';
import useCancelableAsync from '../hooks/useCancelableAsync';
import { Bird } from '../../../../../core/domain/model/Bird';

export default function SpanishTranslateBirdsCancellable() {
    
    const { fetchBirds, cancelFetchBirds } = getSpanishBirdsServices()

    const { data, loading } = useCancelableAsync<Bird[]>({
        execute: fetchBirds,
        cancel: cancelFetchBirds
    })

    return (
        <BirdContainer datatable={data} loading={loading} />
    )

}

```

Y así de simple usamos nuestro Hook definimos nombres de funciones y variables claras que expresan totalmente la intención de  la aplicación. Finalmente vemos los resultados de múltiples clicks en distintos módulos y como en la sección networking apreciamos los benditos canceled. Un respiro para el backend y una optimización excelente para nuestro frontend

FOTOS RESULTADOS

## Conclusión

 Las conclusiones debes sacarlas por ti mismo leer implementar y seguir aprendiendo de otros recursos. por mientras un meme