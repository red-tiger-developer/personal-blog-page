---
title: Nestjs tips La versatilidad de ConfigMdule 
author: Benjamin
date: 2022-10-28 10:32:00 -0500
categories: [Programacion, Nestjs, Arquitectura de software, Typescript]
tags: [typescript, nestjs, hexagonal]
---

![image](https://i.ibb.co/6JfhKfq/Screen-Shot-2022-11-07-at-14-30-39.png)

El manejo de credenciales y configuraciones en nuestras aplicaciones es generalmente algo indispensable para los distintos escenarios de desarrollo y despliegue. Algo tedioso a veces, pero nuestro gatuno framework
Nestjs Nos provee de `ConfigModule` este maravilloso módulo cubre la mayoría de los casos de uso al momento de disponer valores de configuración en nuestra aplicación en esta ocasión les tengo un ejemplo práctico de ConfigModule su inicialización y utilización. ConfigModule es perfecto para mostrar las ventajas de los módulos dinámicos, esta estrategia puede ser utilizada de una manera flexible, no importando donde lo llamemos siempre se comportara igual, pero al momento de registrarlo ahí es donde nosotros definimos que debe hacer, como debe hacerlo y que queremos hacer.

## Instalación y definición de nuestra Config

Asumiendo que ya estas trabjando con un proyecto Nestjs instalamos lo siguiente:

```bash
npm i --save @Nestjs/config

```

Ahora lo primero que haremos es crear un archivo de configuración `.env` con las siguientes variables:

```bash
# http server 
SERVER_PORT=3000
# northwind database
DATABASE_HOST="localhost"
DATABASE_PORT=5432
DATABASE_NAME="northwind"
DATABASE_USER="northwind"
DATABASE_PASSWORD="northwind"
```

> Como tip les recomiento que los nombres sean lo mas descriptivo posible. Por ejemplo `SERVER_PORT ` tiene más contexto que `PORT`. 
  

Ahora crearemos nuestras interfaces que definirán la estructura de nuestras configuraciones y su vez crearemos sus respectivos métodos `factory`

```typescript

export interface DatabaseConfig {
    host: string;
    port: number;
    name: string;
    user: string;
    password: string;
}

export default () => ({
    database: {
      host: process.env.DATABASE_HOST,
      port: parseInt(process.env.DATABASE_PORT, 10),
      name: process.env.DATABASE_NAME,
      user: process.env.DATABASE_USER,
      password: process.env.DATABASE_PASSWORD,
    }
});

```

Lo mismo para la configuracion de nuestro server http

```typescript

export interface ServerConfig {
    port: 3000
}

export default () => ({
    server: {
        port: parseInt(process.env.SERVER_PORT, 10),
    }
});

```

Ahora registramos nuestro `ConfigModule` en `SharedModule`

```typescript
@Module({
    imports: [
        ConfigModule.forRoot({
            isGlobal: true,
            expandVariables: true,
            load: [
                databaseConfig,
                serverConfig
            ],
        })
    ]
})
export class SharedModule { }
```

## Validación y configuración por defecto

Nuestro módulo ya está listo y puede obtener las variables de entorno sin ningún problema, pero ahora agregaremos una librería para poder realizar validaciones de las variables de entorno.

Instalamos `joi` para generar un esquema de validación.
```bash
npm install --save joi
```

Y agregamos lo siguiente a nuestro `SharedModule`. Como podemos observar la librería `joi` nos proporciona una manera simple y declarativa de la estructura de los valores de configuración de nuestra aplicación inclusive nos da la opción de poner valores por defecto
```typescript
@Module({
    imports: [
        ConfigModule.forRoot({
            isGlobal: true,
            expandVariables: true,
            load: [
                databaseConfig,
                serverConfig
            ],
            validationSchema: Joi.object({
                SERVER_PORT: Joi.number().default(3000),
                DATABASE_HOST: Joi.string().default('localhost'),
                DATABASE_PORT: Joi.number().default(5432),
                DATABASE_NAME: Joi.string().required(),
                DATABASE_USER: Joi.string().required(),
                DATABASE_PASSWORD: Joi.string().required(),
            }),
            validationOptions: {
                allowUnknown: true,
                abortEarly: false,
            },
        })
    ]
})
export class SharedModule { }
```

## Usando ConfigService para obtener nuestras configuraciones

Para obtener las variables de configuración lo hacemos por medio del servicio `ConfigService` y su método `get()` podemos obtener los valores inyectando ConfigService en algún servicio definido en nuestra aplicación:

```typescript

@Injectable()
export class CustomService{
    
    constructor(private config: ConfigService) {}

    getHost() {
        return this.configService.get<string>('database.host')
    }

    getDatabaseConfig(){
        return  this.configService.get<DatabaseConfig>('database')
    }

}

```

También podemos inyectar ConfigService de forma dinámica mediante CustomProviders el cual es nuestro caso. 

```typescript
@Module({
    imports: [
        ConfigModule,
        TypeOrmModule.forRootAsync({
            useFactory: (config: ConfigService) => {
                // get specific configuration
                const database = config.get<DatabaseConfig>('database')
                return {
                    type: 'postgres',
                    host: database.host,
                    port: database.port,
                    username: database.user,
                    password: database.password,
                    database: database.name,
                    entities: [
                        ProductEntity,
                        CategoryEntity,
                        SupplierEntity
                    ],
                    synchronize: false,
                    logging: ['query']
                }
            },
            inject: [ConfigService], // Injecting ConfigService (nestjs responsability)
        })
    ]
})
export class NorthwindDatabaseModule { }
```

Finalmente conseguimos una forma limpia y configurable de manejar las configuraciones de nuestra aplicación en este caso hacemos uso de variables de entorno pero dependiendo del caso podemos obtener configuraciones desde archivos o alguna API. La [documentación de ConfigModule](https://docs.nestjs.com/techniques/configuration) verás más a detalle como usarlo, pero sin duda esto nos simplifica el desarrollo.

## [Github repository](https://github.com/nullpointer-excelsior/nestjs-northwind-hexagonal/tree/main/clean-architecture-examples/part-2-connecting-core-infraestructure-with-nestjs)


## Conclusión

Que más decir esto fue corto, pero útil a sí que toma un meme de regalo:

![meme](https://i.ibb.co/NYyS4st/Zombo-Meme-07112022145026.jpg)



