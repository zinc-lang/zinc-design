# Gestión de memoria

## Punteros ARC

zinc elige el conteo de referencias como estrategia central de gestión de memoria. ARC significa automatic reference counting. El significado de automatic es que, al incrementar o decrementar el conteo de referencias, el compilador elige automáticamente instrucciones atómicas o no atómicas.

Cada bloque de memoria asignado en el heap lleva dos valores de conteo de referencias: fuertes y débiles.
Cuando el conteo de referencias fuertes baja a 0:
* Si el valor de referencias débiles no es 0, se llama al destructor de ese bloque de memoria, pero el bloque no se libera. Hasta que el valor de referencias débiles también baje a 0, entonces se libera ese bloque.
* Si el valor de referencias débiles es 0, se llama al destructor y se libera ese bloque de memoria de forma directa.

Como el puntero de conteo de referencias es el tipo de puntero más usado, zinc asigna la sintaxis de puntero más concisa a la semántica de puntero ARC. El tipo `*T` representa un tipo de puntero de conteo de referencias ARC que apunta a un tipo `T`.

El tipo de puntero de referencia débil se expresa con `*weak T`. Un puntero de referencia débil que apunta a un tipo `T` no puede acceder a los miembros ni a las funciones miembro de `T`, ni desreferenciarse.
Un puntero de referencia débil debe convertirse en un puntero de referencia fuerte `*T` mediante la función `std::ptr::upgrade` para poder acceder a los miembros y funciones miembro de `T`.

Para asignar un objeto en el heap se usa la palabra clave `box`. Para la expresión `box <expr>`, si el tipo de `<expr>` es `T`, el tipo de la expresión `box <expr>` es `*T`.
Si la expresión `box` falla al asignar memoria, se produce un panic de forma directa.
Si el usuario realmente necesita manejar a mano el error de asignación de memoria, puede llamar él mismo a la función `std::alloc::try_box` y determinar si la asignación tuvo éxito por el valor de retorno.
Los escenarios en los que hay que manejar a mano el fallo de asignación de memoria en realidad no encajan muy bien con zinc, porque toda la implementación de la biblioteca estándar hace panic de forma directa en OOM. Juzgar solo en la capa de aplicación si la asignación de memoria tuvo éxito no basta.


Ejemplo:
```rust
let x: MyObj = MyObj::new();  // si MyObj::new() devuelve un tipo MyObj
let y: *MyObj = box MyObj::new(); // entonces box MyObj::new() devuelve un tipo *MyObj

let z = y; // z e y apuntan al mismo objeto de tipo MyObj
```

El layout de memoria de un puntero ARC que apunta a un tipo de tamaño estático (se refiere a un tipo sized; por ahora no hablamos de tipos unsized) es el siguiente. En cada asignación dinámica de memoria, primero se hace `malloc(sizeof(UInt) + sizeof(MyObj))` y luego se desplaza el puntero hasta la dirección de inicio del objeto.
Al acceder a un miembro mediante un puntero ARC, el desplazamiento necesario es el desplazamiento de ese miembro respecto a la dirección de inicio del objeto.
```
                ┌──────────────────────┐
                │ strong_count: UShort │
                │ weak_count: UShort   │
ptr: *MyObj  →  │──────────────────────│
                │ field1               │
                │ ......               │
                └──────────────────────┘
```

En caso de referencias circulares, basta con que en el ciclo haya un puntero de referencia débil para que el ciclo se recicle con normalidad y no haya fuga. Pero cuándo usar un puntero de referencia débil lo tiene que indicar el programador.

En zinc no existe el concepto de ownership de Rust, ni un tipo con semántica similar a `Box<T>`, ni “tipos de ownership” move-only.

**Nota**: no hay tipo `*mut T`. Tanto si la variable vinculada es mut como si no, el tipo `*T` siempre tiene permiso de modificación sobre sus miembros.

Ejemplo:

```rust
struct S { m: i32, n: i32 }
fn main() {
    let p1: *S = box { .m = 1, .n = 2 }:S ;
    p1.m += 10; // OK, aunque el binding de la variable p1 no esté modificado con mut, tiene derecho a modificar los miembros

    p1 = box { .m = 10, .n = 20 }:S; // error, el binding de la variable p1 no está modificado con mut, no se le puede asignar de forma directa
}
```

## Semántica de valor y semántica de referencia

Introducción a [semántica de valor y semántica de referencia](https://isocpp.org/wiki/faq/value-vs-ref-semantics#val-vs-ref-semantics):
1. Semántica de valor significa que, al copiar, se replica el valor; la variable nueva y la antigua no tienen ninguna relación entre sí. Una modificación de cualquiera de ellas no se refleja en la otra.
2. Semántica de referencia significa que, al copiar, se copia el puntero; la variable nueva y la antigua comparten datos, y una modificación de esos datos compartidos afecta a ambas variables.

El criterio de juicio es la lógica de este pseudocódigo:
```rust
let mut y: T = x;
modify(&mut y); // se modifica y
print(x);
print(y);
// si la modificación de y no puede afectar de ninguna forma a x, se puede decir que el tipo T es un tipo con semántica de valor. En caso contrario, el tipo T es un tipo con semántica de referencia.
```

zinc anima al usuario a, al definir un tipo, diseñarlo como un tipo con “semántica de valor”.

Porque si un tipo `S` se define con semántica de valor, el tipo `*S` correspondiente es de semántica de referencia, y quien lo usa puede elegir con facilidad `S` o `*S` según el escenario.
Si se diseña directamente como tipo de “semántica de referencia” al definirlo, es difícil obtener el tipo de “semántica de valor” correspondiente, lo cual es incómodo para quien lo usa.

## Borrow

¿Por qué necesitamos punteros borrow?

Porque si solo hay punteros ARC/WEAK, la capacidad de expresión es insuficiente.
El tipo `*T` solo puede apuntar siempre a la cabecera de un “objeto asignado de forma dinámica”. Un objeto asignado de forma dinámica siempre lleva un valor de conteo de referencias. Porque tiene que operar el valor de conteo de referencias y tiene que garantizar que ese valor está a un desplazamiento fijo de la dirección a la que apunta.
Si necesitamos un puntero que apunte al interior de un objeto, no podemos usar un puntero de tipo `*T`.

Si todos los punteros seguros de este lenguaje solo pueden apuntar a la cabecera del objeto, el layout de memoria del objeto no puede ser compacto. Este diseño no favorece el aprovechamiento de los tipos valor.

Si permitimos tomar la dirección de una parte intermedia de un objeto y obtener un puntero nuevo, como muestra la figura:
```
                ┌───────────────────┐
                │ strong_count: u32 │
                │ weak_count: u32   │
ptr: *MyObj  →  │───────────────────│
                │ field1: i32       │
                │ field2: MyStruct  │ ← borrow_ptr: &MyStruct
                └───────────────────┘
```

Entonces ese puntero borrow_ptr necesariamente tiene las siguientes características:
1. En runtime no tiene información suficiente para encontrar el conteo de referencias que lleva el objeto al que apunta
2. En runtime no tiene capacidad de controlar de forma activa el lifetime del objeto al que apunta

Por tanto, el lifetime de un puntero borrow necesita necesariamente una comprobación estática en tiempo de compilación que asegure que el puntero borrow vive menos que el objeto prestado; de lo contrario aparecería el problema de punteros colgantes.

Así que zinc también conserva los punteros borrow de rust y los parámetros de lifetime. Un puntero borrow solo se puede obtener mediante la operación de “tomar dirección”; el compilador rastrea en tiempo de compilación el rango de vida del puntero borrow.
Para distinguir distintos permisos de lectura y escritura, los punteros borrow se dividen en dos tipos: el tipo `&mut T` de lectura y escritura y el tipo `&T` de solo lectura. Las expresiones de operador correspondientes son `&mut <expr>` y `&<expr>`.

Los punteros borrow son un muy buen complemento a la capacidad de expresión de los punteros ARC:
1. Un puntero ARC solo puede apuntar a objetos asignados de forma dinámica en el heap; un puntero borrow puede apuntar tanto a objetos en el heap como a objetos en el stack.
2. Un puntero ARC solo puede apuntar a la cabecera de un objeto asignado de forma dinámica en el heap y no admite aritmética de punteros; un puntero borrow puede apuntar a la cabecera o al interior del objeto, y se puede lograr el desplazamiento de puntero tomando borrow de un miembro.
3. El tipo `*T`, al copiarse o pasarse como parámetro, siempre necesita incrementar o decrementar el conteo de referencias, con un overhead de rendimiento adicional. Asignar, pasar como parámetro y devolver un puntero borrow no necesita operar el valor del conteo de referencias. Usar de forma razonable los punteros borrow ayuda a reducir operaciones redundantes de incremento y decremento del conteo.

Los puntos en común de los punteros borrow de zinc y rust son:
* Ambos tienen comprobación de lifetime. Es decir, el compilador debe comprobar en tiempo de compilación que el puntero borrow en sí vive menos que el objeto prestado.
* Ambos tienen comprobación de permisos de lectura y escritura. Un borrow de tipo `&T` solo tiene permiso de lectura, no de escritura; un borrow de tipo `&mut T` tiene permiso de lectura y escritura.

Las diferencias de los punteros borrow de zinc y rust son:
* Las reglas de mutabilidad son distintas. En zinc, el tipo de puntero ARC `*T`, tanto si el binding de la variable es mutable como si no, siempre puede obtener a través de esa variable ambos borrows, `&T` y `&mut T`.
* zinc no tiene “tipos de ownership” ni “reglas de comprobación de borrow”. El borrow de lectura y escritura no es exclusivo. Las reglas semánticas son mucho más laxas.


### Ejemplo uno: un borrow que apunta a una dirección en el stack
```rust
struct S { m: Int, n: Int }
fn main() {
  // se permite apuntar de forma directa a variables temporales y literales. En ese caso el compilador genera automáticamente una variable local anónima y hace que el puntero borrow apunte a ella.
  let mut p2: &Int = &1;

  {
    let x1 = { .m = 1, .n = 2 }: S;
    let p1 = &mut x1; // error de compilación, no se puede obtener un borrow de lectura y escritura de la variable de solo lectura x1
    p2 = &x1.m; 
  }
  print(p2); // error de compilación, x1 vive menos que p2

  let mut x2 = { .m = 3, .n = 3 }: S;
  let p3 = &mut x2; // correcto
  let p4 = &x2.n; // correcto
  p3.m += 1;    // correcto, p3 tiene permiso de escritura
  println(p4); // correcto, que p3 y p4 existan a la vez no es problema; p3 no tiene exclusividad
}
```

### Ejemplo dos: un borrow que apunta a una dirección en el heap

Ejemplo de código zinc:

```rust
fn main() {
  let mut b: *Int = box 1_i;

  let p1: &Int = &*b; // permitido, apunta a una dirección en el heap
  let p2: &mut Int = &mut *b; // permitido, apunta a una dirección en el heap

  // p1/p2 pueden existir a la vez; un borrow de tipo mut no tiene exclusividad
  println(f"$(p1)");
  *p2 += 1;
  println(f"$(p2)"); 

  b = box 10_i; // se reasigna b

  // leer y escribir p1/p2/b no es problema; p1/p2 no se invalidan
  println(f"$(p1)");
  *p2 += 1;
  println(f"$(p2)"); 
  println(f"$(b)");
}
```
El ejemplo de código anterior, si se escribiera en Rust cambiando `*Int` por `Box<i32>`, daría error de compilación. Porque el borrow mutable de Rust es exclusivo y no puede coexistir con otros borrows. Pero zinc puede compilarlo y no hay problema de seguridad de memoria.
La razón es que, en zinc, en el escenario de tomar borrow de un miembro a través de un puntero ARC, el compilador genera siempre una variable temporal extra que copia b una vez, asegurando que el valor del conteo de referencias +1, y luego toma borrow del miembro. Esa variable temporal se libera al terminar el block actual, y entonces el conteo de referencias se decrementa automáticamente en 1.
Por tanto, la asignación posterior a b no provoca que la memoria originalmente apuntada se libere de inmediato; p1/p2 siguen siendo válidos hasta que termina la función, momento en el que se libera ese bloque de memoria original.

A continuación, el escenario de punteros ARC anidados:
```rust
struct Obj { p: *Int } // zinc no tiene tipos de puntero con semántica move; solo se puede usar el puntero *.

fn main() {
  let mut o: *Obj = box { .p = box 1_i }:Obj;
  let q: &mut Int = &mut *o.p; // aquí se copia temporalmente el puntero p, no se copia o
  *q = 2; // q tiene permiso de modificación

  // crear un puntero borrow a través de un puntero ARC
  let r: &mut Obj = &mut *o;
  r.p = box 10_i; // permitido, no hay error de compilación ni puntero colgante.

  o = box { .p = box 100_i }:Obj; // permitido, no hay error de compilación ni puntero colgante.

  println(*q); // q sigue apuntando al valor original; el resultado impreso es 2
  println(*r.p); // r.p apunta al valor nuevo 100
}
```

En resumen, cuando se crea un puntero borrow nuevo a través de un puntero Arc, la forma en que zinc evita que el puntero borrow se convierta en un puntero colgante es incrementar de forma protectora el conteo de referencias y retrasar la liberación de memoria.
Al compilador le basta con hacer análisis estático local dentro de una función para garantizar la seguridad de memoria.

Desde el punto de vista del rendimiento de ejecución: este diseño sacrifica necesariamente una parte del rendimiento, pero no debería influir demasiado.

1. Lo anterior solo explica el comportamiento esperado del código desde el punto de vista semántico, no describe las instrucciones concretas desde el punto de vista de la optimización.

   Desde el punto de vista de la optimización, no hace falta incrementar el conteo de referencias cada vez que se crea un puntero borrow nuevo. Solo cuando el compilador no puede garantizar en tiempo de compilación que el objeto al que apunta el puntero borrow sobrevive en esa región, por precaución incrementa automáticamente el conteo de referencias para evitar una liberación prematura de memoria.
   En muchos escenarios el compilador tiene información suficiente para optimizar y eliminar esas operaciones extra de incremento y decremento del conteo.

2. Comparado con Rust, el momento de liberación de algunos objetos asignados de forma dinámica en el heap se retrasa.

   Desde el punto de vista de garantizar la seguridad de memoria, un GC de mark-and-sweep sigue en realidad un enfoque similar: mientras haya un puntero que apunte a ese bloque de memoria, el runtime garantiza que no se liberará.
   La diferencia es solo que un GC de mark-and-sweep puede tratar las referencias circulares, y ARC no.
   Nótese que un puntero borrow creado dentro de una función no puede vivir más que esa función, así que si un bloque de memoria se libera con retraso porque existe un puntero borrow, cuando termine el bloque de código en el que está ese puntero borrow, ese bloque de memoria ya se puede liberar; el retraso no será demasiado largo.

3. Siempre que usemos de forma razonable los punteros borrow, comparado con un diseño que usa ARC en todos los sitios, en muchos casos se puede reducir la frecuencia del conteo de referencias.
   1. Un puntero borrow, al asignarse, pasarse como parámetro o devolverse, no necesita modificar el conteo de referencias
   2. A través de un puntero borrow, obtener un puntero borrow de un miembro, siempre que el miembro no sea de tipo puntero ARC, no necesita modificar el conteo de referencias
   3. Incluso al tomar borrow de un miembro de tipo puntero ARC, se puede eliminar por optimización una parte de las modificaciones redundantes del conteo. Esa optimización depende de si el compilador tiene información suficiente para saber que el objeto al que apunta cierto puntero ARC no tiene riesgo de liberación en cierto lifetime.


Desde el punto de vista de la facilidad de uso: este diseño es una gran simplificación de Rust. **Se elimina el borrow checker y desaparece la restricción de que un borrow de tipo &mut es exclusivo**. Se logra, a nivel de tipos, eliminar los tipos move only.


Desde el punto de vista del layout de memoria: la cooperación de ARC y los punteros borrow puede garantizar el control del programador sobre el layout de memoria.
1. El layout de memoria de todos los tipos es determinista; el compilador no inserta de forma implícita ningún miembro oculto.
2. Solo cuando usamos de forma explícita la palabra clave `box` para asignar memoria en el heap se registra el conteo de referencias en la cabecera de la memoria asignada de forma dinámica; entonces hay un overhead extra de memoria por el conteo. Las variables locales usadas en el stack no tienen overhead extra de memoria por conteo de referencias.
3. Cualquier tipo, al usarse como miembro, se coloca de forma directa, sin overhead extra de memoria por conteo de referencias. A menos que el programador indique que el miembro es de tipo ARC, el compilador no insertará de forma implícita una operación box ni un valor de conteo de referencias.

## Operaciones de tomar borrow un poco más complejas

Los tipos contenedor de Zinc, por ejemplo `Vec<T>`, también admiten tomar borrow de un elemento. Ejemplo:

```rust
fn main() {
    let mut vec = (type Vec<Int>)::from(&[1i,2i,3i,4i] as Slice<Int>);
    let p: &Int = vec.index(1); // más adelante, al añadir sobrecarga de operadores, se podrá escribir &vec[1]
    vec.push(5); // aquí puede provocarse una ampliación de vec
    println(*p); // ¿cómo se garantiza que p es legal?
}
```

Aquí nuestra estrategia sigue siendo, al crear el borrow `p: &Int`, incrementar el conteo de referencias del array subyacente, y al morir `p` decrementar el conteo.

La pregunta es: si `p` es un borrow, ¿cómo sabe el compilador que, al morir esa variable borrow, hay que hacer una operación de conteo de referencias, y en qué dirección hacerla?

Respuesta: para que el código anterior compile hace falta la cooperación del compilador y de la biblioteca estándar. La operación de tomar borrow `vec.index(1)` en realidad no devuelve un tipo borrow nativo `&Int`, sino una estructura que lleva datos extra; es un “tipo de puntero inteligente similar a un tipo borrow”.
Luego, a través de esa variable temporal, se hace una operación `deref` y entonces se obtiene el puntero borrow final `p: &Int`.

```rust
// este tipo implementa el Deref trait y, mediante el auto-deref insertado por el compilador, se obtiene el tipo &T; luego se accede al elemento a través del tipo &T.
// este tipo implementa el Drop trait y, al destruirse, puede decrementar el conteo de referencias del array correspondiente
struct ItemRef<'a, T> {
    ptr: *Shared<T>, // apunta a la cabecera del array
    item: &'a T, // el borrow real
}
```

Tras el desazucarado del compilador, el código anterior significa en realidad esto:

```rust
fn main() {
    let mut vec = (type Vec<Int>)::from(&[1i,2i,3i,4i] as Slice<Int>);
    let _temp: ItemRef<Int> = vec.index(1); // variable local añadida de forma implícita por el compilador; es un “puntero borrow gordo”
    let p: &Int = _temp.deref(); // operación de “auto-deref” añadida de forma implícita por el compilador
    // nota: si no vinculamos el valor de retorno de index a una variable local, este _temp debería destruirse en cuanto termine la sentencia index, no al terminar el bloque de sentencias.

    vec.push(5); // al ampliar, se dispara la condición de copy-on-write, se asigna un espacio nuevo, pero el espacio antiguo tiene conteo de referencias mayor que 0, no se libera, p sigue siendo legal
    println(*p);
    // el compilador destruye de forma implícita la variable _temp; entonces el conteo de referencias del espacio antiguo llega a 0 y se libera el contenido del array original
}
```

Nota: la regla de “auto-deref” es que, en escenarios de coincidencia de patrones, paso de parámetros y retorno, si existe `T: Deref<U>`, entonces `T` puede convertirse al tipo `U` llamando automáticamente a la función miembro `deref`.

Para alcanzar el objetivo anterior, el Index trait de la biblioteca estándar de zinc se define así:
```
pub trait Index<Idx>
{
    type Output<'a>;

    fn index<'s>(&'s self, index: Idx) -> Self::Output<'s>;
}
```

Fíjate en que el tipo de retorno de la función index ya no es el tipo borrow nativo fijo `&Item`, sino un tipo estructura `struct ItemRef<'a, Item>` que encapsula `&Item` junto con otros datos.
Podemos verlo como un tipo de puntero inteligente personalizado: un tipo que “hace que el tipo borrow nativo lleve metadatos extra”, una especie de “puntero borrow gordo”.
Por tanto, este tipo asociado `Output` también debe definirse como un constructor de tipos, es decir, un generic associated type de Rust.
Este puntero no solo contiene el borrow del elemento concreto, sino también un puntero a la cabecera del array interno, y este puntero gordo tiene destructor; tiene información suficiente para mantener correctamente en runtime el valor del conteo de referencias.

Solo desde el punto de vista del análisis semántico, esto parece afectar mucho al rendimiento. Pero si consideramos inlinear las funciones clave `index` `deref` `drop` usadas aquí, es totalmente posible eliminar en la fase de optimización de rendimiento las operaciones extra de conteo de referencias:
1. Si en la función main no se ha llamado a funciones de modificación como push, ni se ha tomado un borrow mutable de vec, el compilador tiene información suficiente para saber que ese bloque de memoria no tiene riesgo de liberación, y puede eliminar por completo todas las operaciones extra de incremento y decremento del conteo y las variables locales superfluas;
2. Si en la función main se toma un borrow mutable de vec y se pasa a otras funciones, el compilador no está seguro de si vec puede modificarse; entonces la operación protectora de conteo de referencias es indispensable y no se considera un desperdicio de rendimiento.

## Cómo resuelve zinc el clásico problema de invalidación de iteradores

Al explicar las características de seguridad de memoria de Rust, el siguiente es un ejemplo clásico:

```rust
///// Rust code
fn main() {
    let mut vec: Vec<i32> = Vec::from(&[1i,2i,3i]);
    for ref _i in &vec {
        vec.push(4);
    }
}
```

Rust dará error de compilación en este ejemplo, porque `vec.iter()` crea un borrow de solo lectura de `vec` que se guarda en el iterador; y `vec.push` necesita crear un borrow de lectura y escritura de `vec`.
Según las reglas de comprobación de borrow de Rust, ambos entran en conflicto, así que el compilador da error, previene de forma efectiva el problema de invalidación de iteradores y evita un tipo de comportamiento indefinido habitual en C++. Esta es una virtud de Rust digna de elogio.

Zinc garantiza la seguridad de otra forma. A continuación está el código correspondiente de Zinc; a nivel de código fuente se parece mucho, pero el efecto de ejecución real es muy distinto del de Rust:

```rust
fn main() {
    let mut vec: Vec<Int> = (type Vec<Int>)::from(&[1i,2i,3i,4i] as Slice<Int>);
    for ref _i in vec {
        vec.push(4);
    }
}
```

1. Como en zinc no hay “tipos de ownership”, `Vec<T>` no tiene semántica move, sino semántica copy. Podemos hacer `let vec2 = vec;` a voluntad y seguir usando la variable `vec`.
2. Aunque `Vec<T>` tiene semántica copy, eso no significa que, al copiar, copie de inmediato todos los miembros; implementa una optimización de copy-on-write.
Al copiar un `Vec<T>` solo se hace una copia superficial: se incrementa en 1 el conteo de referencias del array asignado internamente en el heap; el contenido de los elementos no se copia de verdad.
3. La copia profunda ocurre al modificar una variable `Vec<T>`: si al modificar se descubre que el conteo de referencias del array interno es mayor que 1, se copian todos los elementos y luego se hace la modificación.
4. En el ejemplo anterior, al crear el iterador se incrementa en 1 el conteo de referencias del array interno. Si durante la iteración no hay operaciones de escritura sobre `vec`, la copia profunda nunca ocurrirá;
si durante la iteración ocurre un `vec.push`, entonces se descubre que hay que disparar una copia profunda: `vec` crea internamente un array nuevo y añade un 4 al final.
5. Pero el array al que apunta el iterador no ha cambiado: no se ha liberado ni modificado, y la iteración sigue. El iterador y `vec` ya apuntan a dos arrays completamente distintos, así que no hay ningún comportamiento indefinido.
6. Cuando termina el bucle for, termina el lifetime del iterador; al destruirse decrementa en 1 el conteo de referencias del array original, y entonces se dispara la destrucción y liberación del array original.

Así que zinc también evita el “comportamiento indefinido” y no introduce “inseguridad de memoria”, pero paga un cierto coste de rendimiento: al ejecutar la operación push, el tipo `Vec<T>` original hace de forma implícita una copia profunda.
Ese coste es aceptable: no queremos comportamiento indefinido como en C++, ni comprobaciones en tiempo de compilación excesivamente estrictas como en Rust; bajo la premisa de garantizar “seguridad” y “facilidad de uso”, la pérdida de rendimiento de ejecución ya es el menor coste que hay que pagar.
Si el usuario quiere ahorrarse esa copia profunda, entonces debe cuidar de no modificar el vec original durante el bucle.

Para Rust, la seguridad de memoria y la thread-safety son **objetivos de diseño**. El “principio de exclusión mutua entre mutabilidad y compartición” (alias XOR mutation principle) es la **base teórica** hacia el objetivo.
El ownership y el borrow checker son **medios de implementación**. Zinc eligió usar la técnica de copy-on-write para alcanzar el mismo objetivo; caminos distintos, mismo destino.
O, dicho de otra forma, este diseño de Zinc convierte la comprobación estática en tiempo de compilación de “mutabilidad y compartición mutuamente excluyentes” (alias XOR mutation) en una garantía en tiempo de ejecución.
copy-on-write, en esencia, garantiza que al modificar la memoria ese bloque solo tiene un alias; es del mismo tenor que ownership + borrow checker. La comparación detallada es la siguiente:

* La comprobación de ownership + borrow checker en tiempo de compilación de Rust garantiza en tiempo de compilación que el objeto solo tiene un único mutable alias. Pero pueden existir a la vez varios immutable alias.

  Si se viola el principio alias XOR mutation, se da error de compilación.

* El `RefCell<T>` de Rust se usa mediante las funciones miembro `borrow` `borrow_mut`. Garantiza en runtime que el objeto solo tiene un único mutable alias, pero pueden existir a la vez varios immutable alias.

  Si se viola el principio alias XOR mutation, se produce un panic en runtime y se rechaza seguir ejecutando.

* El `RwLock<T>` de Rust se usa mediante las funciones miembro `read` `write`. Garantiza en runtime que el objeto solo tiene un único mutable alias, pero pueden existir a la vez varios immutable alias.

  Si se viola el principio alias XOR mutation, se bloquea el hilo actual en runtime hasta que se cumple la condición y se sigue ejecutando. `Mutex<T>` es similar: garantiza que un bloque de datos solo puede tener un accedente a la vez, tanto para lectura como para escritura.

* El copy-on-write de Zinc se implementa así: en un immutable method no se comprueba el ref-count; en un mutable method se comprueba el ref-count, garantizando que al modificar el objeto solo tiene un único alias.

  Si se viola el principio alias XOR mutation, se copia el objeto asignado en el heap. Se garantiza que al modificar/liberar la memoria solo hay un alias.

Todas las técnicas de implementación anteriores pueden satisfacer el mismo principio "alias XOR mutation" y todas pueden alcanzar el objetivo de “seguridad de memoria”. El diseño de Rust es solo una de las opciones para implementar “seguridad de memoria”, en absoluto el único diseño viable.

## Lifetimes

El llamado lifetime se refiere al intervalo en el que una variable está viva durante la ejecución del programa. El lifetime de una variable empieza cuando se crea y termina cuando se destruye.

### Lifetime de las variables globales

Una variable global vive durante toda la ejecución del programa, así que su lifetime existe desde el inicio del programa hasta el final.

> **TODO**: en la implementación actual, ninguna variable global llama al destructor. Más adelante hay que discutir si este diseño es razonable y si necesitamos destruir las variables globales antes de salir del proceso.

### Lifetime de las variables temporales

Una variable temporal es una variable generada por una expresión que no está vinculada a un nombre concreto.

El momento en que termina el lifetime de una variable temporal tiene dos casos: uno es al terminar la sentencia actual; el otro es al terminar el bloque de sentencias actual. Depende de si ha sido prestada por una variable de lifetime más largo.

Ejemplo:
```rust
fn f() -> S { ... }
// supongamos que S tiene el método miembro method
impl S {
  fn method(&self) -> &S { return self; }
}

fn test() {
  f(); // la variable que devuelve f() es una variable temporal; no está vinculada a un nombre de variable. El lifetime de esta variable temporal termina al terminar la sentencia.

  let p1: &S = &f(); // la variable temporal que devuelve f() ha sido prestada, y el lifetime del borrow supera esta sentencia. Entonces el lifetime de esta variable temporal se extiende hasta el final del bloque de sentencias; al terminar la función, la variable temporal se destruye.

  let p2: &S = f().method(); // la variable temporal que devuelve f() ha sido prestada, y el lifetime del borrow supera esta sentencia. Entonces el lifetime de esta variable temporal se extiende hasta el final del bloque de sentencias; al terminar la función, la variable temporal se destruye.
  // se usan p1 p2
}
```

### Lifetime de las variables locales

En general, el lifetime de una variable local, incluidos los parámetros de función, empieza cuando se crea y termina cuando termina el bloque de código actual.

No existe algo como non-lexical-lifetime; no hace falta. Rust introdujo eso porque la exclusividad de los tipos mut borrow, si el análisis no es lo bastante preciso, provoca muchos errores de compilación innecesarios y limita la capacidad de expresión del usuario. Si no tenemos la regla de comprobar en tiempo de compilación la exclusividad del mut borrow, tampoco hace falta buscar formas de relajar las reglas de comprobación del compilador.

* Para un tipo personalizado, podemos implementarle el `Drop trait`; las variables de ese tipo, al terminar el lifetime, llamarán automáticamente al destructor correspondiente.
* Para un tipo ARC, al terminar el lifetime se decrementará automáticamente en 1 el conteo de referencias del espacio de heap al que apunta.
* Para un tipo borrow, su destructor no hace nada.

Ejemplo:
```rust
struct S { m: i32 }
impl Drop for S {
  fn drop(&mut self) {
    println("drop S");
  }
}

fn test() {
  let s1 = { .m = 1 }:S;
  let s2 = s1; // aquí ocurre una copia

  // s1 y s2 se destruyen al terminar test
}
```

Antes se dijo que, “en general, el lifetime de una variable local termina al terminar el bloque de código”. Es decir, todavía hay “casos especiales”. El caso especial es la expresión `move`.

En el lenguaje zinc, tras la palabra clave move puede ir una expresión, llamada expresión move. Entonces ocurre semántica de movimiento, no semántica de copia.

```rust
fn test() {
  let x = { .m = 1 }:S;
  let y = move x;
  println(x.m); // error de compilación, el lifetime de x ya ha terminado; después no se puede volver a usar x
}
```

Si lo que se hace move es un puntero ARC, podemos usar esta función para reducir en algunos escenarios las operaciones de incremento y decremento del conteo de referencias:
```rust
fn f(arg: *S) { }

fn test() {
  let s: *S = box { .m = 1 }:S;
  f(s); // si se llama así, al pasar el parámetro ocurre una copia, el conteo de referencias +1, al final del cuerpo de f el conteo de referencias -1. Al salir de test el conteo de referencias vuelve a -1,
  f(move s); // si se llama así, se puede asegurar que al pasar el parámetro el conteo de referencias no se incrementa automáticamente, ni se decrementa al final del cuerpo de test. El lifetime de s se transfiere al cuerpo de f para que lo mantenga.
  // tras hacer move de s, volver a usar s provoca un error de compilación

  // ...
}
```

Si queremos que la variable `x` termine su lifetime de forma anticipada, podemos usar la sentencia `move x;`, de modo que el resultado de la expresión move no se vincule a ninguna variable; entonces esa x se destruirá en el acto, y no al final del bloque de sentencias.

En resumen, la semántica move de zinc no está en la “definición del tipo”; todos los tipos son de forma natural copiables o movibles.
Si se usa semántica move, la elección está en la “expresión”. zinc ofrece “expresiones” con semántica move, no “tipos” con semántica move.

La sentencia `return` tiene por defecto semántica move; `return expr;` equivale a `return move expr;`, no hace falta escribir de forma explícita la palabra clave `move`.

## Marcadores de lifetime

Un marcador de lifetime puede verse como un parámetro genérico. Los tipos borrow necesitan este parámetro genérico.
Los marcadores de lifetime explícitos se usan generalmente en la firma de una función, para expresar la relación de lifetime entre los parámetros y el tipo de retorno.

```rust
fn find<'a, 'b>(strings: Slice<'a, Str<'b>>) -> Str<'b> {

}
```

> **TODO**: las reglas para omitir marcadores de lifetime en la firma de una función aún no están completamente implementadas.

## Destructores y `Drop`

Un destructor es una función miembro del objeto. Cuando termina el lifetime del objeto, el compilador inserta automáticamente instrucciones para llamar al destructor.

El destructor hace estas cosas:
1. Llama a la función `Drop::drop` implementada por ese tipo; si no la hay, no la llama
2. Llama a la función `Drop::drop` de los miembros
3. Decrementa en 1 el conteo de referencias correspondiente de todos los miembros de tipo puntero Arc y puntero Weak que contiene ese tipo. Si el conteo de referencias del objeto al que apunta un puntero ARC baja a 0, se llama automáticamente al destructor de ese objeto.

El destructor lo genera siempre de forma automática el compilador; el usuario solo puede controlar el comportamiento de la función `Drop::drop`, que es solo una parte del proceso de destrucción.
Si un tipo y todos sus miembros no tienen función `Drop::drop` ni punteros de tipo conteo de referencias, el destructor de ese tipo está vacío.

En general, el usuario no debería llamar de forma activa al destructor de un objeto. Pero al escribir código unsafe, se permite al usuario forzar la llamada al destructor de un objeto mediante la función `unsafe fn destruct_in_place`.

La función `Drop::drop` nunca puede ser llamada de forma explícita por el usuario.

Ejemplo de un tipo personalizado que hace `impl std::mem::Drop` del trait:

```rust
struct S {
  p: *i32,
  m: String,
}

impl Drop for S {
  fn drop(&mut self) {
    println("S is dropped.")
  }
}
```

Nota: una función `Drop::drop` personalizada no necesita ocuparse de la destrucción y liberación de memoria de los miembros; el destructor de los miembros se llama automáticamente.
