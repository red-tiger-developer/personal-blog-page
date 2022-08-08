
# Creando queries dinámicas con Typeorm para flojear más. Quiero decir hacer menos código

Me encanta Typeorm su query builder permite crear complejas queries SQL al más estilo fluent API por abajo una implementación impecable de builder y otras patrañas. Vamos a un paso más adelante y crearemos una API con filtro dinámico donde la aplicación cliente pueda filtrar por el criterio o propiedades que le de la gana. Esto es bastante útil cuando NO quieres emplear un montón de condicionales Ifs, o quieres reutilizar alguna condicional en otra entidad.

## Vamos al código no más palabras que se las lleva el viento

Lo primero será definir nuestra arquitectura, así de pro master hacker de nasa.
La entidad a modo de prueba es Movement (robada de un proyecto) la cual representa movimientos de bodega imagínense que tienen miles propiedades para filtrar no coloco más por la flojera

```typescript

@Entity("movement")
export class Movement {
  @PrimaryGeneratedColumn("increment", { type: "integer", name: "movement_id" })
  movementId: number;

  @Column("character varying", { name: "sku", default: null })
  sku: string;

  @Column("character varying", { name: "description", default: "" })
  description: string;

  @Column("integer", { name: "qty", default: 0 })
  qty: number;

  @CreateDateColumn({ type: "timestamptz", name: "created_at" })
  createdAt: Date;

  @Column({ name: "warehouse_id" })
  warehouseId: number;

  @ManyToOne(() => Warehouse, (warehouse) => warehouse.warehouseId)
  @JoinColumn([{ name: "warehouse_id", referencedColumnName: "warehouseId" }])
  warehouse: Warehouse;

  @ManyToOne(() => Reason, (reason) => reason.reasonId)
  @JoinColumn([{ name: "reason_id", referencedColumnName: "reasonId" }])
  reason: Reason;

  @Column({ name: "reason_id" })
  reasonId: number;

  @Column({ name: "user_id" })
  userId: number;

  @ManyToOne(() => User, user => user.userId)
  @JoinColumn([{ name: "user_id", referencedColumnName: "userId" }])
  user: User;

  @ManyToOne(() => Detail, detail => detail.movement, { cascade: true })
  @JoinColumn({ name: "detail_id"})
  detail: Detail;
}

```

## La Arquitectura de nuestro hack

Ahora definimos nuestra interfaz Filter en la cual pondremos las propiedades disponibles a filtrar la estructura sería clave y valor de entrada para aplicar un filtro.

```typescript

export interface Filter {
    [x:string]: any;
}

```
 Nuestra segunda definición es una Condición la cual la representaremos mediante una función usando type en vez de interface. Esta función recibe un SelectQueryBuilder objeto propio de Typeorm que se encarga de construir queries y un segundo parámetro que representa la condición de filtrado.

```typescript

export type Condition<T> = (qb: SelectQueryBuilder<T>, filterValue: any) => SelectQueryBuilder<T>;

```

Ahora definiremos un tipo para el objeto que contendrá las condiciones este objeto implementara una estrategia de "clave" o nombre del filtro y su condición. Notese que la interfaz Filter y Conditions comparten la misma estructura. noten la magia de typescript ambas formas son igual de válidas.

```typescript

export type Conditions<T> = Record<string, Condition<T>>;

```

Implementaremos el filtro deseado para nuestra entidad. Pondremos solo estas 3 propiedades a modo de ejemplo

```typescript

export interface MovementFilter extends Filter {
    sku: number;
    qty: number;
    warehouseId: number;
}

```
Y ahora se viene una de las partes interesantes del post. implementaremos una condición por la propiedad SKU o Stock Keeping Unit para los bilingües esto a modo de ejemplo.
```typescript

const movementConditions: Conditions<Movement> = {
    sku: (qb: SelectQueryBuilder<Movement>, sku: number) => qb.andWhere('m.sku = :sku', { 'sku': sku })
}

```
Para entender esta lógica solo debemos poner atención a la condición "m.sku = : sku", ya que se preguntaran ¿de dónde diablos sale esa "m"? Pues bien cuando definimos una QueryBuilder con Typeorm podemos definir alias entonces "m" será el alias para la entidad Movement, ya que nuestra query base sería la siguiente:

```typescript

const qb = this.repository.createQueryBuilder('m')
        .innerJoin('m.detail', 'detail')
        .innerJoin('m.warehouse', 'warehouse')

```
Asi que ven la maldita "m".Muy bien ahora que entendemos la condición construida en la SelectQueryBuilder vemos que lo único que hace esta función de tipo Condition<ENTITY_TYPE> es recibir un SelectQueryBuilder construir la condición y agregar el valor como parámetro de la query y finalmente devolver el nuevo SelectQueryBuilder. Finalmente nuestro objeto conditions quedará de la siguiente forma:

```typescript

const movementConditions: Conditions<Movement> = {
    sku: (qb: SelectQueryBuilder<Movement>, sku: string) => qb.andWhere('m.sku = :sku', { 'sku': sku }),
    qty: (qb: SelectQueryBuilder<Movement>, qty: number) => qb.andWhere('m.qty = :qty', { 'qty': qty }),
    warehouseId: (qb: SelectQueryBuilder<Movement>, id: number) => qb.andWhere('warehouse.warehouse_id = :id', { 'id': id })
}

```

Un mónton de condiciones XD. Pero ya tenemos la lógica del filtro. Nada tan complicado. Y ahora viene la magia de que filtros aplicaremos. No es ciencia de cohetes ni nada si, pero si para mí la magia es que es lógica puramente funcional, en esta ocación a la casa la orientación a objeto o la programación imperativa XD, ya fuera de bromas este es el código y lo explicaré. esto es poesía a lo ñuñoino.


## parece enredado pero a la m**** explicación es ahora 

Nuestra magia empieza con la función build mostrada. Esta recibe un objeto queryBuilderBase: SelectQueryBuilder,(typeorm object)y un objeto conditions: Conditions<ENTITY_TYPE> El cuál habíamos hablado previamente
Esta función la explicaré pasa a paso papitos

```typescript

function  build<T>(queryBuilderBase: SelectQueryBuilder<T>, conditions: Conditions<T>, filter: Filter) {
    // filter conditions not undefined and reduce to final QueryBuilder
    const query: SelectQueryBuilder<T> = Object
        .keys(filter) //get keys from filter
        .filter(key => filter[key] !== undefined) // removing undefined filter values
        .reduce((prev: SelectQueryBuilder<T>, curr: string) => {
            const fn = conditions[curr]
            if (fn) {
                return fn(prev, filter[curr])
            }
            return prev
        }, queryBuilderBase) // reducing to a SelectQueryBuilder with all conditions
    
    return query

}

```
Nos ayudamos de Object y su función keys que si le pasamos un objeto nos retorna sus keys entonces le pasamos nuestro objeto de tipo Filter para que nos retorne las keys del filtro que queremos aplicar, Muy bien como este ciudadano nos devuelve un arreglo de keys ahora le vamos a aplicar un filter() para eliminar los valores de tipo undefined al filtro aplicado o sea los valores que no hay que filtrar porque no tienen un valor definido

```typescript

Object
    .keys(filter) //get keys from filter
    .filter(key => filter[key] !== undefined) // removing undefined filter values

```
Ahora usamos la función reduce() que muchos la ignoran como a la fea del baile, pero tiene sus gracias o si no pregúntele a redux y esas patrañas
```typescript

.reduce((prev: SelectQueryBuilder<T>, curr: string) => {
    const fn = conditions[curr]
    if (fn) {
        return fn(prev, filter[curr])
    }
    return prev
}, queryBuilderBase) 

```
Como aporta reduce a nuestro súper filtro dinámico, bueno reduce() funciona dándole como parámetros una función con un valor previo y un valor current que sería llamado como actual entonces con esos 2 valores debes devolver algo lo que tú desees una suma una resta, una comparación de objetos o lo que se te dé la gana entonces esta operación será una iteración sobre los datos o arreglo que tengas. Por último hay un tercer parámetro que es el valor inicial en nuestro caso sería nuestra query base.

Explicado esto fíjense en que recibimos un SelectQueryBuilder entonces por cada key de las condiciones filtradas y válidas preguntamos por su condición  del objeto Conditions y la si existe ejecutamos la condición que devuelve un nuevo SelectQueryBuilder y la devolvemos entonces el objeto estará con los nuevos filtro y condiciones

```typescript
/**
 * creamos la variable fn que es la que aplica la funcion de la condicion recordemos que curr es la key del filtro entonces si exite el filtro aplicamos la funcion de creacion de SelectQueryBuilder pasanodel el valor del filtro
 **/

const fn = conditions[curr]
if (fn) {
    return fn(prev, filter[curr])
}

```
Ahora creamos una clase con un método estático para poder hacer uso de nuestra utilidad y adjuntamos la estructura de Conditions

```typescript

import { SelectQueryBuilder } from "typeorm";
/**
 * Las interfaces que definan la estructura de un filtro para una query deben heredar de esta interface
 */
export interface Filter {
    [x:string]: any;
}

/**
 * Esta funcion recibe un SelectQueryBuilder<T> y un valor de filtro para ser aplicado en el objeto SelectQueryBuilder<T>
 */
export type Condition<T> = (qb: SelectQueryBuilder<T>, filterValue: any) => SelectQueryBuilder<T>;
/**
 * Representa un objeto clave Condicion donde la clave es el nombre del filtro
 */
export type Conditions<T> = Record<string, Condition<T>>;

/**
 * Clase con metodos de ayuda para crear Queries sobre la librería Typeorm 
 */
export class DynamicQueryBuilder {
    
    /**
     * Construye una nueva query a partir de una query base una serie de condiciones, un filtro con los valores 
     * y un paginado opcional.
     * @param queryBuilderBase objeto SelectQueryBuilder<T> el que debe contener la sentenica basica de select * from de acuerdo a typeorm
     * @param conditions objeto clave Condition el cual contendra las condiciones opcionales de busqueda
     * @param filter objeto clave valor que contiene los campos a filtrar y su valor
     * @param page Representa un paginado opcional para las queries
     * @returns 
     */
    static build<T>(queryBuilderBase: SelectQueryBuilder<T>, conditions: Conditions<T>, filter: Filter) {
        // filter conditions not undefined and reduce to final QueryBuilder
        const query: SelectQueryBuilder<T> = Object
            .keys(filter) //get keys from filter
            .filter(key => filter[key] !== undefined) // removing undefined filter values
            .reduce((prev: SelectQueryBuilder<T>, curr: string) => {
                const fn = conditions[curr]
                if (fn) {
                    return fn(prev, filter[curr])
                }
                return prev
            }, queryBuilderBase) // reducing to a SelectQueryBuilder with all conditions
 
        return query
    
    }

}

```
Como nuestra función nos devuelve un objeto SelectQueryBuilder podremos agregar algún ordenamiento, límite y offset o condición obligatoria esto va a depender de nuestros casos de uso.Bueno vamos a juntar todo en un servicio con Nestjs que haga las consultas dinámicas

```typescript

@Injectable()
export class MovementService {
    /**
     * Condiciones para el filtro 
     **/
    private movementConditions: Conditions<Movement> = {
        sku: (qb: SelectQueryBuilder<Movement>, sku: string) => qb.andWhere('m.sku = :sku', { 'sku': sku }),
        qty: (qb: SelectQueryBuilder<Movement>, qty: number) => qb.andWhere('m.qty = :qty', { 'qty': qty }),
        warehouseId: (qb: SelectQueryBuilder<Movement>, id: number) => qb.andWhere('warehouse.warehouse_id = :id', { 'id': id })
    }

    constructor(
        @InjectRepository(Movement) private readonly repository: Repository<Movement>
    ) { }

    async findByFilter(filter: MovementFilter) {
        // build query base
        const qb = this.repository.createQueryBuilder('m')
            .innerJoin('m.detail', 'detail')
            .innerJoin('m.warehouse', 'warehouse')
        // building a dynamic SelectQueryBuilder
        const query = DynamicQueryBuilder.build<Movement>(qb, this.movementConditions, filter)
        // aditional conditions if you want  
        return query.getMany()

    }

}

```

```typescript

const filter: Filter = {
    sku: 9997687667;
    warehouseId: 230;
}

const movements = await this.movementService.findByFilter(filter)

```

## Conclusiones

Nada de conclusiones acá hay un meme