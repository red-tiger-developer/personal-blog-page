---
title: Repository pattern simple escalable y para todo tipo de proyectos
author: Benjamin
date: 2024-05-29 00:00:00 -0500
categories: Software,Arquitectura,Patrones,ORM
tags: software,arquitectura,patrones,orm
---

![intro](assets/img/intros/intro7.webp)

El patrón de diseño Repository se utiliza para abstraer la lógica de acceso a datos de la lógica de negocio. Su objetivo es encapsular el acceso a los datos y ocultarlo a cualquier componente que utilice el repositorio. De esta manera, la capa de negocio no necesita preocuparse por cómo se accede a los datos, sino que simplemente interactúa con el repositorio para obtener la información que requiere.

## ¿Por qué usar el Repository pattern?

El Repository pattern es una buena opción cuando se quiere desacoplar la lógica de negocio de la lógica de acceso a datos. Esto permite que la lógica de negocio sea más fácil de entender y de mantener.

Más que permitirnos tener un acceso a los datos desacoplados, la realidad es que el Repository pattern nos permite tener las siguientes ventajas:

- Lógica de negocio desacoplada de la lógica de acceso a datos nos permite crear un dominio más limpio y más relacionado al negocio que a implementaciones tecnológicas.
- Facilita la implementación de pruebas unitarias y de integración, ya que podemos simular el acceso a datos mediante mocks.
- Facilita la implementación de código limpio y patrones de diseño como el patrón de inyección de dependencias.

## Ejemplo sencillo y elegante

Nos basaremos en el proyecto `spotify-clone` tomando la entidad `Album`, la cual representa un álbum de música. 

### Componente Entidad

Una entidad es un objeto del dominio que tiene una identidad única y distintiva, que persiste a lo largo del tiempo y puede cambiar su estado. Las entidades poseen atributos y métodos que permiten manipular su estado.

```typescript

export class Album {

    @IsUUID()
    @IsNotEmpty()
    @ApiProperty({ description: 'The ID of the album' })
    id: string;

    @IsNotEmpty()
    @IsString()
    @ApiProperty({ description: 'The title of the album' })
    title: string;

    @IsNotEmpty()
    @IsString()
    @ApiProperty({ description: 'The photo of the album' })
    photo: string;

    @IsNumber()
    @ApiProperty({ description: 'The year the album was released' })
    year: number;


}
```
En este caso, nuestra entidad `Album` no posee comportamiento, solo atributos y lógica de validaciones.

### Componente Repositorio

El repositorio es el encargado de acceder a los datos de la entidad `Album`. En este caso, el repositorio se encarga de acceder a los datos de los álbumes desde una base de datos. El repositorio es definido a través de una interfaz que define los métodos que se pueden utilizar para acceder a los datos de los álbumes. No nos preocuparemos de cómo se accede a los datos, simplemente el repositorio nos indica las operaciones que podemos realizar sobre nuestra entidad. 

```typescript

export abstract class AlbumRepository {
    abstract findAll(): Promise<Album[]>;
    abstract findById(id: string): Promise<Album>;
    abstract findByArtistId(id: string): Promise<Album[]>;
}

```

### Implementaciones del Repositorio

Como hemos mencionado, el repositorio nos abstrae del acceso de datos, esto nos permitirá definir una implementación dependiendo del caso. También nos permitirá cambiar la fuente de datos sin tener que modificar la lógica de negocio en caso de que se requiera cambiar el tipo de almacenamiento por algún motivo.


#### Implementación productiva con TypeORM y Postgres

En esta implementación utilizamos Postgres como base de datos y TypeORM como librería para acceder a los datos. Esta es nuestra implementación productiva:

Definimos una Entidad de TypeORM que mapea la tabla `albums` en la base de datos. Entidad ORM (`AlbumEntity`) es distinta a la entidad del dominio `Album`, ya que la entidad de TypeORM es específica para el acceso a datos. Mientras que la entidad del dominio es específica para la lógica de negocio.

```typescript

@Entity({ name: "albums" })
export class AlbumEntity {

    @PrimaryGeneratedColumn('uuid')
    id: string;

    @Column()
    title: string;

    @Column()
    photo: string;

    @Column()
    year: number;

    @ManyToOne(() => ArtistEntity, artist => artist.albums)
    artist: ArtistEntity;

}
```

Definimos un repositorio que implementa la interfaz `AlbumRepository`. En este caso, utilizamos NestJs para injectar el repositorio de TypeORM mediante el decorador `@InjectRepository`.

```typescript
@Injectable()
export class PostgresAlbumRepository extends AlbumRepository {

    constructor(@InjectRepository(AlbumEntity) private repository: Repository<AlbumEntity>) {
        super()
    }

    findAll(): Promise<Album[]> {
        return this.repository.find();
    }

    findById(id: string): Promise<Album> {
        return this.repository.findOneBy({ id });
    }

    findByArtistId(id: string): Promise<Album[]> {
        return this.repository.findBy({ artist: { id } });
    }

}

```

#### Implementación de pruebas con un mock en memoria

En este caso, utilizamos un mock en memoria para simular el acceso a datos. Esta implementación es útil para pruebas unitarias y de integración.

```typescript
@Injectable()
export class InMemoryAlbumRepository extends AlbumRepository {

    private albums: Album[] = [
        { id: '1', title: 'Album 1', photo: 'photo1.jpg', year: 2021 },
        { id: '2', title: 'Album 2', photo: 'photo2.jpg', year: 2021 },
        { id: '3', title: 'Album 3', photo: 'photo3.jpg', year: 2021 },
    ];

    findAll(): Promise<Album[]> {
        return Promise.resolve(this.albums);
    }

    findById(id: string): Promise<Album> {
        return Promise.resolve(this.albums.find(album => album.id === id));
    }

    findByArtistId(id: string): Promise<Album[]> {
        return Promise.resolve(this.albums.filter(album => album.artist.id === id));
    }

}

```

### Usando repository pattern en lógica de negocio

En este caso, `AlbumService` es el encargado de la lógica de negocio relacionada con los álbumes. Este se comunica con el repositorio para obtener los datos que necesita. `AlbumService` no tiene que preocuparse de cómo se accede a los datos, simplemente se comunica con el repositorio para obtener los datos que necesita.

```typescript
@Injectable()
export class AlbumService {

    constructor(private repository: AlbumRepository) {
    }

    async getAlbums(): Promise<Album[]> {
        return this.repository.findAll();
    }

    async getAlbum(id: string): Promise<Album> {
        const album = await this.repository.findById(id);
        if (!album) {
            throw new NotFoundException(`Album with id ${id} not found`);
        }
        return album;
    }

    async getAlbumsByArtist(id: string): Promise<Album[]> {
        return this.repository.findByArtistId(id);
    }

}
```

### Inyectando dependencias

Para usar Repository Pattern generalmente se utiliza junto a la inyección de dependencias. En este caso, utilizamos NestJS como framework y el módulo TypeORM para la inyección de dependencias.

```typescript
@Module({
    imports: [
        TypeOrmModule.forFeature([AlbumEntity])
    ],
    providers: [
        AlbumService,
        {
            provide: AlbumRepository,
            useClass: PostgresAlbumRepository
        }
    ],
    exports: [AlbumService]
})
```

Y si queremos probar nuestro servicio sin framework o librería, por ejemplo en el contexto de pruebas unitarias, podemos crear nuestro servicio de la siguiente manera:

```typescript

// in memory repository
const repository: AlbumRepository = new InMemoryAlbumRepository();
const service = new AlbumService(repository);

// json repository
const repository: AlbumRepository = new JsonAlbumRepository();
const service = new AlbumService(repository);

```

## Conclusión

El verdadero poder del patrón Repository no es permitirnos cambiar la fuente de datos de manera sencilla. Este nos permite tener una lógica de negocio totalmente desacoplada de la implementación tecnológica del acceso de datos. Este sencillo enfoque, nos permite definir un dominio más limpio y testeable.

## [Github repository](https://github.com/nullpointer-excelsior/microservices-architecture-nestjs/)

### Meme de despedida

![meme](assets/img/memes/meme5.webp)

