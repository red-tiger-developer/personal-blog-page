---
title: Mono Repositorios con Nestjs Para una Arquitectura Orientada a Eventos
author: Benjamin
date: 2022-12-13 10:32:00 -0500
categories: [Programacion, Nestjs, Typescript, Events]
tags: [typescript, nestjs, rabbitmq]
---

![image](https://i.ibb.co/Pgt2j53/Screen-Shot-2022-12-13-at-18-24-41.png)

Definir una arquitectura distribuida puede llegar a ser complejo cuando manejas muchos proyectos o servicios y estos tienen que interactuar entre sí. Cada servicio tendrá su propio repositorio y nos dará la ventaja de usar la tecnología adecuada al problema específico a solucionar, pero cuando tu stack tecnológico comparte el mismo lenguaje o framework podrías pensar en usar Monorepositorios.
Monorepo es enfoque de desarrollar múltiples aplicaciones dentro de un solo repositorio. Esto podemos verlo muchas veces en arquitecturas de microservicios centralizando todas las aplicaciones dentro de un mismo proyecto donde podremos reutilizar ciertas piezas de software entre aplicaciones sin necesidad de desplegar librerías asociadas a gestores de dependencias. Si bien esto trae ventajas también trae desafíos al momento de desplegar y de definir una arquitectura escalable y mantenible. Pero como todo enfoque este debe ser evaluado según tu caso y necesidades.

Nestjs nos provee una forma fácil de implementar monorepositorios, su mágica e útil cli nos permite transformar nuestro proyecto con una sola aplicación a múltiples aplicaciones y la posibilidad de definir librerías compartidas.

## Creando un Monorepositorio con Nestjs

Podemos empezar creando una simple aplicación con NestJs

```bash
#!/bin/bash
nest new main-application
```

Hemos creado una aplicación llamada `main-application` nada nuevo donde el código fuente esta situado en el directorio `src`, Pero esta estructura de proyecto puede ser transformada a monorepositorio simplemente agregando una nueva aplicación ingresamos dentro del raíz del proyecto y ejecutamos:

```bash
#!/bin/bash
nest generate app other-application
```

Nuestra aplicación cambio su estructura, Se creo un nuevo directorio en la raíz del proyecto llamado `apps` dentro del cual se ubicaran las aplicaciones. Nuestra carpeta `src` es movida a `main-application`.

```bash
 apps
├──  main-application
│   ├──  src
│   │   ├──  main.ts
│   │   ├──  producer.controller.spec.ts
│   │   ├──  producer.controller.ts
│   │   ├──  producer.module.ts
│   │   └──  producer.service.ts
│   ├──  test
│   │   ├──  app.e2e-spec.ts
│   │   └──  jest-e2e.json
│   └──  tsconfig.app.json
└──  other-application
    ├──  src
    │   ├──  app.controller.spec.ts
    │   ├──  app.controller.ts
    │   ├──  app.module.ts
    │   ├──  app.service.ts
    │   └──  main.ts
    ├──  test
    │   ├──  app.e2e-spec.ts
    │   └──  jest-e2e.json
    └──  tsconfig.app.json
```
Ahora también podemos crear librerías donde podremos compartir código entre aplicaciones.

```bash
#!/bin/bash
nest generate library shared
```
Esto creará un directorio en la raíz del proyecto llamado `libs`.

```bash
#!/bin/bash
 libs
└──  shared
    ├──  src
    │   ├──  index.ts
    │   ├──  shared.module.ts
    │   ├──  shared.service.spec.ts
    │   └──  shared.service.ts
    └──  tsconfig.lib.json
```

Esto es todo, ya podemos trabajar con monorepositorios dentro de NestJs.

## Arquitectura orientada a eventos utilizando Monorepositorio

Para ver las ventajas que nos dará los monorepositorios implementaremos una arquitectura orientada a eventos utilizando RabbitMQ con una cola de mensajes y una `dead-letter` para mensajes fallidos, todo esto mediante la utilización de un custom módulo con NestJs definido como una librería compartida.

### ¿Qué es RabbitMQ? 

RabbitMQ es un broker de mensajería de código abierto, distribuido y escalable, que sirve como intermediario para la comunicación eficiente entre productores y consumidores.

RabbitMQ implementa el protocolo mensajería de capa de aplicación AMQP (Advanced Message Queueing Protocol), el cual está enfocado en la comunicación de mensajes asíncronos con garantía de entrega, a través de confirmaciones de recepción de mensajes desde el broker al productor y desde los consumidores al broker.

### ¿Qué es una Dead letter?

Una Dead Letter es una cola donde los mensajes que no pudieron ser procesados por los consumidores llegan, con esto podemos generar una estrategia de reintento o de registro de que el mensaje no pudo ser procesado.

En el siguiente repositorio tendremos un stack tecnológico que implementa una arquitectura orientada a eventos con la cual podemos implementar el envío, consumo y reintento de mensajes asíncronos mediante un servidor RabbitMQ. Este proyecto es ideal para generar una estrategia de dead-letter-queue para la recuperación de operaciones fallidas siguiendo la arquitectura modular de nestjs y sus buenas prácticas.


## Módulo Rabbitmq-queue

Este módulo fue diseñado con el paquete `@nestjs/microservices` de Nestjs y contiene las siguientes características:

* `Consumer de Mensajes:` Factory de creación de microservicio Worker 
* `Dead Letter Consumer de mensajes no procesados por error en el consumer:` Factory de creación de microservicio Recovery 
* `Cliente Productor de mensajes:` Servicio Nesjs Injectable importando `RabbitmqModule`

Nuestro módulo puede ser importado y utilizado por las aplicaciones que definamos en el directorio `apps/` y desde cualquier librería o módulo compartido que definamos en `libs/`

Este proyecto requiere Node 14 o superior. instala sus dependencias y empezaremos a definir una estructura básica de prodctor, consumidor y control de errores

```bash
#!/bin/bash
npm install
```

## Levantando un servidor RabbitMQ

Debes tener instalado docker y ejecutar la siguiente instrucción:

```bash
#!/bin/bash
docker run --rm -it --hostname rabbit-server -e RABBITMQ_DEFAULT_VHOST=mono-repo-example -p 15672:15672 -p 5672:5672 rabbitmq:3-management
```

## Stack de Aplicaciones 

Para configurar las aplicaciones que estarán en la misma cola RabbitMQ escuchando los eventos introduciré 3 conceptos:
* `Producer:` aplicación encargada de enviar mensajes a una cola
* `Worker:` aplicación encargada de realizar una tarea especifica dependiendo del mensjae. Será quien consuma los mensajes de una cola
* `Recovery:` "Dead Letter Queue" aplicación encargada de consumir mensajes que no pudieron ser procesados por algún error en la aplicación `Worker`


Estas 3 aplicaciones deben compartir una configuración en comúm para que puedan funcionar en conjunto. Y esta es definida por la interface `RabbitmqQueueModuleOptions`

Ejemplo de configuración Base:

```typescript
const options: RabbitmqQueueModuleOptions = {
    credentials: {
      host: 'localhost',
      password: 'guest',
      port: 5672,
      vhost: 'mono-repo-example',
      user: 'guest'
    },
    queue: {
      name: 'my-queue',
      deadLetter: {
        exchange: 'dlx',
        patterns: ['SEND_MESSAGE'] // dead-letter for specific message pattern
      }
    }
  }

```

También deben compartir la estructura del mensaje que serán enviados a RabbitMQ. Esta estructura puede ser definida de acuerdo a tus necesidades.

Ejemplo de una estructura de mensaje
```typescript
interface Data {
    name: string
    message: string
}
```

Ya definida nuestra configuración y estructura de mensaje, Podemos empezar a levantar nuestras aplicaciones

### Iniciar Worker

Para iniciar una aplicaión `Worker` debes ir a tu proyecto Nestjs en este caso sería `apps/worker` y en el archivo `apps/worker/main.ts` debes invocar la función `createWorkerMicroserviceOptions` el cual devolverá un objeto `ClientProviderOptions` el cual es necesario para iniciar un microservicio de Nestjs

Ejemplo main.ts
```typescript

async function bootstrap() {
  
  const options = {
    credentials: {
      host: 'localhost',
      password: 'guest',
      port: 5672,
      vhost: 'javel',
      user: 'guest'
    },
    queue: {
      name: 'my-queue',
      deadLetter: {
        exchange: 'dlx',
        patterns: ['SEND_MESSAGE']
      }
    }
  }
  // Build ClientProviderOptions for Worker Microservice
  const workerMicroservice = await RabbitmqQueueModule.createWorkerMicroserviceOptions(options)
  // Init Microservice
  const app = await NestFactory.createMicroservice(WorkerModule, workerMicroservice);
  
  await app.listen()

}
bootstrap();

```
### Definir Worker controller para obtener mensajes.

Ahora para definir los controladores que consumirán los mensajes es de igual forma, manera que nos indica la documentación de Nestjs. El valor de `MessagePattern` debe coincidir con las configuraciones base si se necesita usar una Dead letter desde la App `Recovery`.

```typescript
@Controller('consumer')
export class ConsumerController {

    @EventPattern('SEND_MESSAGE', Transport.RMQ)    
    consume(@Payload() data: RabbitmqMessage<Data>, @Ctx() context: RmqContext) {
        
        try {
            // Make some operations with message
            context.getChannelRef().ack(context.getMessage())
        } catch (error) {
            Logger.warn(`An error occured with mnessage: ${data.id}`);
              // reject message and set reque = false
              // this will dead letter our message
              context.getChannelRef().reject(context.getMessage(), false);
        }
    }

}
```

Nuestro controlador debe recibir el siguiente `@Payload()`. donde el Tipo Data es nuestra estructura de mensaje definida.

```typescript
@Payload() data: RabbitmqMessage<Data>
```

la interface RabbitmqMessage es la siguiente:
```typescript
interface RabbitmqMessage<T> {
    id: string;
    pattern: string;
    timestamp: Date;
    data: T;
}
```
Nosotros nos debemos preocupar por solo el tipo de data los otros valores son definidos por la librería RabbitmqQueue.


### Iniciar Recovery

Lo mismo para iniciar la aplicaión `Recovery` debes ir a tu proyecto Nestjs en este caso sería apps/recovery y en el archivo `apps/recovery/main.ts` debes invocar la función `createRecoveryMicroserviceOptions` el cual devolverá un objeto `ClientProviderOptions` el cual es necesario para iniciar un microservicio de Nestjs

```typescript
async function bootstrap() {

  const options = {
    credentials: {
      host: 'localhost',
      password: 'guest',
      port: 5672,
      vhost: 'javel',
      user: 'guest'
    },
    queue: {
      name: 'my-queue',
      deadLetter: {
        exchange: 'dlx',
        patterns: ['SEND_MESSAGE']
      }
    }
  }

  const microservice = await RabbitmqQueueModule.createRecoveryMicroserviceOptions(options)
  const app = await NestFactory.createMicroservice(RecoveryModule, microservice);
  app.listen()
  
}
bootstrap();
```
### Definir Recovery controller para obtener mensajes que no pudieron ser procesados por `Worker`.

Ahora para definir los controladores que consumirán los mensajes fallidos es de igual manera que nos indica la documentación de Nestjs. El valor de MessagePattern debe estar incluido en los valores de `queue.deadLetter.patterns`


```typescript
@Controller('dead-letter-queue')
export class DeadLetterController {

    @EventPattern('SEND_MESSAGE', Transport.RMQ)
    async consume1(@Payload() data: RabbitmqMessage<Data>, @Ctx() context: RmqContext) {
        Logger.log('Dead-letter SEND_MESSAGE 1: ', data)
    }

    @EventPattern('SEND_MESSAGE2', Transport.RMQ)
    async consume2(@Payload() data: RabbitmqMessage<Data>, @Ctx() context: RmqContext) {
        Logger.log('Dead-letter SEND_MESSAGE 2: ', data)
    }

}
```
### Iniciar Producer 
Para iniciar nuestra aplicación solo debemos hacer un registro del RabbitmqQueueModule proporcionando nuestra configuración base en el módulo Nestjs que necesitemos producir mensajes

```typescript
@Module({
  imports: [
    RabbitmqQueueModule.register({
      credentials: {
        host: 'localhost',
        password: 'guest',
        port: 5672,
        vhost: 'javel',
        user: 'guest'
      },
      queue: {
        name: 'my-queue',
        deadLetter: {
          exchange: 'dlx',
          patterns: ['SEND_MESSAGE']
        }
      }
    })
  ],
  controllers: [ProducerController],
})
export class ProducerModule { }

```

Ahora solo debemos injectar nuestro servicio productor de mensajes:

```typescript
import { RabbitmqProducerClient } from '@app/shared/rabbitmq-queue/services/rabbitmq-producer-client.service';

@Injectable()
export class MyService {
  
  constructor(private rabbitmq: RabbitmqProducerClient) { }

}

```
Ahora para enviar mensajes lo hacemos de la siguiente manera

```typescript
 const data: Data = {
    name: 'rabbitmq-message',
    message: 'Simple message for testing'
  }
  const payload = this.rabbitmq.emitTo<Data>('SEND_MESSAGE', data)

```
el metódo `emitTo()` nos devolverá el mensaje que enviará a Rabbitmq

```json
{
  "id": "bb322090-578d-4717-9d12-a38eadbd0311",
  "pattern": "SEND_MESSAGE",
  "timestamp": "2022-11-09T13:18:45.704Z",
  "data": {
    "name": "rabbitmq-message",
    "message": "Simple message for testing"
  }
}
```
Generamos un controlador de pruebas para probar nuestra arquitectura orientada a eventos

```typescript
@Controller('producer')
export class ProducerController {

  constructor(private rabbitmq: RabbitmqProducerClient) { }

  @Post()
  emitMessage(@Body() data: any) {

    const payload = this.rabbitmq.emitTo<Data>('SEND_MESSAGE2', data)
    Logger.log(`Producer: message sent ${payload.id}`)

    return {
      message: 'OK',
      messageSent: payload
    }
  }
}

```
Y generamos la siguiente instrucción en curl para realizar la petición:

```bash
#!/bin/bash
curl -s -X POST -d '{"name": "rabbitmq-message","message": "testing message on event architecture"}' -H 'Content-type: application/json' http://localhost:3001/producer | jq
```
Adicional a este comando puedes hacer uso de Make para realizar las siguientes operaciones:



```bash
#!/bin/bash
# start a fucking rabbitmq server
make rabbit
# start worker
make worker
# start recovery
make recovery
# start producer
make producer
# send a request
make produce
# testing dead-letter
make produce; sleep 1; make produce; sleep 1; make produce
```

## Fin del Post

Esta es la forma más simple de utilizar una arquitectura orientada a eventos. Generamos un proyecto de tipo mono Repositorio para poder reutilizar nuestras piezas de software y seguir una misma implementación con Nestjs, La estrategia de mono repositorio debes analizarla bien, ya que dependerá de tus necesidades y tipo de proyecto pero para empezar a jugar no está mal.

Acá el meme de despedida

![meme](https://i.ibb.co/899qQr1/Zombo-Meme-01122022233002.jpg)