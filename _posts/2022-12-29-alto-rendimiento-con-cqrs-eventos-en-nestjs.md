---
title: CQRS en Soluciones de Alto Rendimiento con Nestjs
author: Benjamin
date: 2022-11-07 10:32:00 -0500
categories: [Programacion, Nestjs, Arquitectura de software, Typescript]
tags: [typescript, nestjs, hexagonal, microservices]
---

![image](https://i.ibb.co/1Tm9pVK/Screen-Shot-2022-12-30-at-00-55-44.png)

CQRS es una solución de diseño de software que separa las operaciones de lectura y escritura de nuestra aplicación. Esto no es mero capricho de la ingeniería de software. Este enfoque que proporciona CQRS nos ayuda a implementar soluciones de alto rendimiento en ambientes concurrentes. Alerta de spoiler CQRS se complementa de maravilla en arquitecturas orientadas a eventos, esta combinación nos proporciona un alto rendimiento, escalabilidad y una mejora en el diseño de las operaciones sobre nuestra aplicación.

En post anteriores aprendimos [arquitectura hexagonal](https://nullpointer-excelsior.github.io/posts/implementando-hexagonal-con-nestjs-part1/) utilizando una API backend basada en la base de datos Northwind la cual administra órdenes de compras, productos y proveedores. En esta ocasión les presentaré una solución de alto rendimiento a nivel de infraestructura con los casos de uso de crear y leer órdenes, una operación de lectura y la otra de escritura. El estado actual de esta aplicación es solo una API conectada a una base de datos relacional.

Realizaremos la siguiente prueba de carga sobre nuestra aplicación.

* Simulación de ambiente concurrente con usuarios incrementando de 10 en 10 cada segundo hasta llegar a 300 usuarios.
* 150 usuarios crearán 1 orden cada 0.5 y 1.2 segundos.
* 150 usuarios leerán 100 órdenes cada 0.5 y 1.2 segundos.

Esto en promedio realizarán entre 150 y 180 peticiones cada segundo.

Utilizaremos Locust que es una librería Python para realizar pruebas de carga sobre aplicaciones web. Los detalles de como usarlo lo encuentras [acá](https://docs.locust.io/en/stable/quickstart.html). 

Los puntos importantes a tener en cuenta para interpretar los resultados son las siguientes métricas: 

* Total Requests per Seconds: 
    * RPS: Peticiones por segundo
    * Failures: Peticiones fallidas
* Response Times (ms)
    * Media response time y 95% percentile
* Number of users:
    * Usuarios conectados en el tiempo.

## Las pruebas de cargas en nuestra API

Resumiendo las pruebas de carga realizadas en una aplicación que utiliza 1 única base de datos de lectura y escritura muestra el siguiente rendimiento:

![bad-performance](https://i.ibb.co/tZ87PD9/mal-rendimiento.png)

En promedio 17 segundos de lecturas cada segundo. Y en modo individual haciendo las pruebas obtenemos esto:

 ```bash
 #!/bin/bash

 # GET ORDERS
 time curl -s  "http://localhost:3000/purchase/order?page=1&size=10" | jq
# real	0m18.185s 18 seconds!!!
# user	0m0.026s
# sys	0m0.016s
#  CREATE ORDER
time curl -s -X POST -d "$(order.json)" -H "Content-Type: application/json" http://localhost:3000/purchase/order | jq
# real	0m0.030s
# user	0m0.020s
# sys	0m0.011s

```

La escritura de los datos no ha perdido rendimiento, pero la escritura ha subido a tiempos considerables. Como vemos estamos bloqueando nuestro servidor por las peticiones, aumentaron los tiempos dramáticamente y en esta ocasión no es la idea escalar la base de datos o las instancias de la aplicación.

## Arquitectura inicial 

La arquitectura inicial de esta aplicación era la siguiente:

![basic-architecture](https://i.ibb.co/GWMqMk8/Screen-Shot-2022-12-29-at-17-17-55.png)

Ahora para lograr escalar nuestro sistema no lo haremos dándole más recursos a nuestra base de datos (escalado vertical) y tampoco subiremos la cantidad de instancias de la aplicación (escaldo vertical), Si bien esto en muchos casos es suficiente, También tendremos escenarios donde el costo beneficio no será eficiente subiendo los recursos y para evitar elevar estos costos más de lo necesario debemos aprovechar los recursos disponibles al inicio antes de empezar el escalado de nuestra arquitectura.

## Arquitectura de alto rendimiento basada en microservicios 

Para lograr la mejora del rendimiento en nuestro sistema la nueva arquitectura tendrá los siguientes puntos:

* `Arquitectura hexagonal`: Separación del dominio de la infraestructura (opcional).
* `CQRS`: Separación de las lecturas y escrituras sobre nuestro origen de datos.
* `Arquitectura de microservicios`: Tendremos un servicio que se encarga administrar los modelos, lecturas y escrituras de los datos, y 1 servicio encargado de sincronizar nuestro modelo de lectura.
* `Broker de mensajería asíncrona`: Intermediario que comunicará nuestros microservicios de manera desacoplada.

Todo esto se plasma en el siguiente diagrama:

![performance-architecture](https://i.ibb.co/4K5FHrY/Screen-Shot-2022-12-29-at-18-07-44.png)

## Pruebas de carga sobre la nueva arquitectura

Nuestra nuevo enfoque basado en microservicios esta listo y ahora si realizamos las mismas pruebas de carga sobre nuestra nueva arquitectura obtenemos los siguientes resultados:

![good-performance](https://i.ibb.co/ZWdKHpG/buen-rendimiento.png)

Una diferencia bastante notoria en este caso nos centramos en una solución a nivel de arquitectura representando la realidad de muchos sistemas a gran escala. No nos centramos en escalar nuestra aplicación y la base de datos sin antes aprovechar ya los recursos disponibles. Tampoco no siempre es viable reescribir una aplicación desde cero en lenguajes más eficientes como Go o Rust. En definitiva, tendremos muchos casos, realidades y tecnologías disponibles y nosotros como ingenieros debemos encontrar el mejor costo beneficio de una solución. 

Les dejo el repositorio para que puedan ver la implementación del código y puedas jugar con las pruebas de carga 

## [Github repository](https://github.com/nullpointer-excelsior/high-performance-with-cqrs-events-on-nestjs)

## Finalizando 

Para terminar nos despedimos con el meme de cortesía.

![meme](https://i.ibb.co/JdbbbVH/memeawdasdc.jpg)







