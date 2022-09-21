---
title: ¿Que es el Patron especification?
author: Benjamin
date: 2022-09-17 18:32:00 -0500
categories: [Programacion, Java, ArquitecturaSoftware]
tags: [java, patrones, arquitectura software]
---


## ¿Que es el Patron especification?

El patrón Specification nos permite encapsular reglas de negocio, ya sean estas sencillas o complejas, de manera que 
sean reutilizables y fáciles de cambiar.

Este patron fue adoptado para la arquitectura Domain Driven design para poder realizar operaciones de filtrados sobre entidades
satisfaciendo ciertas condiciones de la entidad.

Basicamente una especificacion recibe una entidad (objeto o modelo) y este es evaluado por una condicion o regla de negocio.

### para que necesito esta wea?

el punto fuerte de specification es que cada regla (specification) puede ser reutilizable y puede componerse de otras reglas
creando un conjunto de specificaciones en una sola, la maravilla es que pasamos de lo siguiente:

```java
var productService = new ProductService();

/**
 * Multiples metodos para busquedas Objetos tipo servicios con multiples lineas de codigo
 */
List<Product> productsWithStockGreaterThan10 = productsService.findByWithStockGreaterThan10(10);

List<Product> productsWithStockGreaterThan10AndOtherCondition = productsService.findByWithStockGreaterThan10AndOtherConditions(10, otherConditions...n);

/**
 * y por debajo estos metodos con unos infernales if
 */

if (product.getStock() > 10 && (product.getOtherPropod() != null && product.getOtherPropd().equals("some f**cking condition") )){
    return "product from a complex conditions"
}


```

a esta maravilla:

```java
var productService = new ProductService();


var specStock = new WithStockSpecificationGreaterThan10(10);
/***
 * un unico metodo para filtrado de entidades ( service.findSatisfiedBy(specStock) )
 */
List<Product> productsWithStockGreaterThan10 = productsService.findSatisfiedBy(specStock);

var specStockAndOtherCondition = new OtherConditionSpecification()
                                        .and(specStock);
List<Product> productsWithStockGreaterThan10AndOtherCondition = productsService.findSatisfiedBy(spec);

/**
 * espicificaciones hechas por separadas y concatenables
 * 
 */

public class ByStockGreaterThan10Units extends CompositeSpecification<Product> {
    /**
     * Una condicion bien entendible testeable y reutilizable
     * @param candidate
     * @return
     */
    @Override
    public boolean isSatisfiedBy(Product candidate) {
        return candidate.getStock() > 10;
    }
}
```

## Beneficios
- expresivo
- Condiciones y logicas de negocios especificas encapsuladas en una sola Specificacion.
- Puedes crear specificaciones nuevas a partir de otras mediante composicion utilizando operadores AND, OR y NOT
- Specificaciones Testeables individualmente y de forma compuesta
- Un unico punto de entrada que recibe una especificacion y devuelve Las entidades que cumplan con las condiciones o reglas de negocio

## Contras
- Specification puede ser complejo de implemntar con SQL u ORM dependiendo del lenguaje o libreria de persistencia (ejemplo Java y su criteria api :fearful: )
- Tus Specificaciones compuestas pueden necesitar de alguna estrategia de creacion como un builder o un factory dependiendo de tus necesidades. 



## Testeable mantenible y reutilizables

Como dijimos con especification podremos testear nuetras condiciones y reutilizarlas. imaginemos el caso tipico de un CRUD
pero uno que verdad en tu vida laboral y no en los tutoriales de "happy path" como los del maravilloso framework nestjs con gatos.
 te dare el caso de una api de consulta de productos que empezo con busqueda de los quiero todos, los quiero por esta id ahora los quiero por categoria ahora los quiero por tienda.
parece sencillo, pero a medida que nuestro Product owner y usuarios consumieron agua llasca se pusieron muy creativos y nos piden mas weas le va aplicando condiciones y mas condiciones por propiedaes especificas calculos locos
y todo para crear la super api de productos

### como enfrentamos esto pos compadre?
 como dijimos especification nos da las condiciones y las vamos a utilizar entonces el proimer paso es crear las condiciones 

### basta de hablar como cotorra y vamos al codigo

primero definiremos las interfaces base de nuestro patron del mal:

```java
/**
 * interfaz base que cumplira nuestra condicion
 * @param <T>
 */
public interface ISpecification<T>{
    boolean isSatisfiedBy(T candidate);
}

/**
 * utilizando el patron composite para extender la funcionalidad del Specification base definimos una interfaz que hereda de nuestra Specification Base 
 * y definmos metodos que actuaran como condicionales basicos estos 
 * reciben una Specification y debuelven otra compuesta 
 * @param <T>
 */
public interface ICompositeISpecification<T> extends ISpecification<T> {
    ICompositeISpecification<T> and(ICompositeISpecification<T> other) ;
    ICompositeISpecification<T> or(ICompositeISpecification<T> other);
    ICompositeISpecification<T> not();
}
```
 Ya tenemos nuestras interfaces y definiremos las implementaciones para ICompositeISpecification<T> 
 
que serviran de operadores

```java

/**
 * Representa un AND operator
 * @param <T>
 */
public class AndSpecification<T> extends CompositeSpecification<T> {

    private ICompositeISpecification<T> left;
    private ICompositeISpecification<T> right;

    public AndSpecification(ICompositeISpecification<T> left, ICompositeISpecification<T> right) {
        super();
        this.left = left;
        this.right = right;
    }

    /**
     * recibe un candidato y evalua las operaciones mediante un and con las especificaciones contenidas en esta clase
     * @param candidate
     * @return
     */
    @Override
    public boolean isSatisfiedBy(T candidate) {
        return this.left.isSatisfiedBy(candidate) && this.right.isSatisfiedBy(candidate);
    }
}
```
 lo mismo aplicado para los operadores OR y NOT

```java
/**
 * Not specification
 * @param <T>
 */
public class NotSpecification<T> extends CompositeSpecification<T> {

    private ICompositeISpecification<T> spec;

    public NotSpecification(ICompositeISpecification<T> spec) {
        super();
        this.spec = spec;
    }

    @Override
    public boolean isSatisfiedBy(T candidate) {
        return !this.spec.isSatisfiedBy(candidate);
    }
}

/**
 * OR specification
 * @param <T>
 */
public class OrSpecification<T> extends CompositeSpecification<T> {

    private ICompositeISpecification<T> left;
    private ICompositeISpecification<T> right;

    public OrSpecification(ICompositeISpecification<T> left, ICompositeISpecification<T> right) {
        super();
        this.left = left;
        this.right = right;
    }

    @Override
    public boolean isSatisfiedBy(T candidate) {
        return this.left.isSatisfiedBy(candidate) || this.right.isSatisfiedBy(candidate);
    }
}
```
Ahora nos queda implementar otra interfaz de CompositeSpecification<T> la cual sera una clase abstracta e implementara 
la logica de los operadores 

```java
public abstract class CompositeSpecification<T> implements ICompositeISpecification<T> {

    @Override
    public ICompositeISpecification<T> and(ICompositeISpecification<T> other) {
        return new AndSpecification<T>(this, other);
    }

    @Override
    public ICompositeISpecification<T> or(ICompositeISpecification<T> other) {
        return new OrSpecification<T>(this, other);
    }

    @Override
    public ICompositeISpecification<T> not() {
        return new NotSpecification<T>(this);
    }

}
```
 ya tenemos nuestro patron listo para ser utilizado

implementaremos unas especificaciones heredando de CompositeSpecification<T> e implementando el metodo 

```java
boolean isSatisfiedBy(T candidate)
```

ahora este metodo contendra las logicas sobre una entidad Producto

```java
/**
 * Stocke mayor a 10
 */
public class ByStockGreaterThan10Units extends CompositeSpecification<Product> {
    @Override
    public boolean isSatisfiedBy(Product candidate) {
        return candidate.getStock() > 10;
    }
}

/**
 * productos de cierto departamento
 */
public class ByDepartmentSpecification extends CompositeSpecification<Product> {

    private Department department;

    public ByDepartmentSpecification(Department department) {
        this.department = department;
    }

    @Override
    public boolean isSatisfiedBy(Product candidate) {
        return candidate
                .getDepartment()
                .getId()
                .equals(this.department.getId());
    }
}

```

Ahora implementaremos la logica de filtrado de nuestros productos solo crearemos 
el metodo findSatisfiedBy(ISpecification<Product> spec) el cual recibe una especification y devolvera nuestros productos

```java
@AllArgsConstructor
public class ProductService {

    private List<Product> products;

    public List<Product> findSatisfiedBy(ISpecification<Product> spec) {
        return products
                .stream()
                .filter(spec::isSatisfiedBy)
                .collect(Collectors.toList());
    }
}
```

Con estas lineas es suficiente para utiliar las specification
nuestro filter se basara en la specificacion obtenida e internamente esta especificacion puede estar compuesta por 1 
o mas especificaciones utilizando operadores como and, or y not

```java

    var filtered = products
                .stream()
                .filter(spec::isSatisfiedBy)
                .collect(Collectors.toList());

```


Martin flower quien ideo esto 
https://www.martinfowler.com/apsupp/spec.pdf


# Conclusión

Naaa de conclusiones solo memes

![meme](https://i.ibb.co/4SPny7K/Zombo-Meme-05082022223013.jpg)