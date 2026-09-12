# Genéricos y trait

## Genéricos

Tanto las definiciones de tipos como las definiciones de funciones pueden declarar parámetros genéricos.

### Tipos genéricos

Los tipos definidos por el usuario pueden llevar parámetros genéricos.
Hay dos tipos de parámetros genéricos: parámetros genéricos de lifetime y parámetros genéricos de tipo. En la lista de parámetros genéricos, el orden es: los parámetros de lifetime delante de los parámetros de tipo.

todo: más adelante hay que considerar si se debería añadir la característica de genéricos constantes; hay que tener en cuenta las razones por las que swift no admite esa característica

```rust
struct S<'a, T> {
    p: &'a T
}

enum E<T> {
    A(T), B(Int)
}
```

La definición de tipos admite una cláusula `where` opcional. La cláusula `where` puede expresar: la relación de supervivencia entre lifetimes, y si un tipo cumple una restricción de trait.

El uso de un tipo genérico necesita proporcionar argumentos genéricos.

```rust
fn main() {
    let v = 1;
    let x: S<Int> = S { .p = &v };
}
```

### Funciones genéricas

Las funciones también admiten genéricos. Igual que los tipos genéricos, los parámetros genéricos también admiten parámetros genéricos de lifetime y parámetros genéricos de tipo.
Las funciones genéricas también admiten una cláusula `where` opcional.

```
fn swap<'a, T>(lhs: &'a mut T, rhs: &'a mut T) {}
```

Nota: en zinc la sintaxis de llamada a funciones genéricas cambia, para evitar la sintaxis turbo fish de rust.

<details>

<summary>Qué es la sintaxis turbo fish</summary>

<div style="border: 1px solid black; padding: 10px;">

En el diseño de sintaxis de Rust, si una llamada a función genérica usa de forma directa la misma sintaxis que la definición de la función genérica, hay ambigüedad sintáctica, como en este ejemplo:
```rust
fn main() {
    let (the, guardian, stands, resolute) = ("the", "Turbofish", "remains", "undefeated");
    let _: (Bool, Bool) = (the<guardian, stands>(resolute)); // ¿qué significa esta línea?
}
```

El núcleo de esta ambigüedad es que el operador menor-que y los ángulos genéricos reutilizan el mismo símbolo, lo que hace que la línea anterior tenga dos interpretaciones:
1. Es una llamada a función genérica dentro de paréntesis; el nombre de la función es `the`, los parámetros genéricos son `guardian` `stands` y el parámetro de la función es `resolute`
2. Es una tuple de dos elementos. El primer elemento es la comparación `the<guardian`; el segundo elemento es la comparación `stands>(resolute)`.

Para resolver este conflicto de sintaxis, Rust exige que, al llamar a una función genérica, se ponga `::` detrás del nombre de la función; por tanto, la sintaxis de llamada correcta es:
```
the::<guardian, stands>(resolute)
```

Esa es la sintaxis turbo fish. Por ejemplo `Vec::<i32>::with_capacity(16)`.

</div>
</details>

<br/>

La sintaxis turbo fish de Rust elimina el problema de ambigüedad del análisis sintáctico, pero introduce inconsistencia sintáctica y no es estética.
Zinc adoptó otro diseño, que evita el problema de ambigüedad y también mantiene la consistencia de la sintaxis.

1. En fully-qualified-call-syntax, si aparece un tipo genérico, hay que empezar siempre con la palabra clave `type`. Ejemplos:
    * `(type Vec<Int>)::with_capacity(4)`  
      Como `Vec` lleva de forma explícita un argumento genérico, `Vec<Int>::with_capacity(4)` es un error de sintaxis.
    * `Vec::with_capacity(4)`  
      Se permite no indicar el parámetro genérico de `Vec`; el parámetro genérico puede inferirse del contexto, y entonces no hace falta el prefijo type.
    * `(type Vec<String> as Default)::default()`  
      Se permite indicar de forma explícita el nombre del tipo y el nombre del trait correspondiente, convirtiéndose en una llamada de sintaxis completamente calificada a una función concreta. Eso significa que las funciones miembro de distintos trait pueden tener el mismo nombre e implentarse para el mismo tipo, sin conflicto.

2. La inicialización de estructuras y la coincidencia de patrones adoptan siempre sintaxis de tipo pospuesta. Ejemplos:
    * Inicialización de estructura: `let v = { .x = 1 } : S<Int>;`
    * Coincidencia de patrones de estructura: `expr is { .x = 1 } : S<Int>`

3. En las llamadas a funciones genéricas, la lista de parámetros genéricos debe escribirse dentro de los paréntesis. Ejemplos:
    * `std::mem::size_of(<Int>)`

La lógica central de estas modificaciones es hacer que todos los nombres de tipo con genéricos aparezcan siempre en un contexto que empieza con un identificador concreto.
Así, cuando el compilador hace el análisis sintáctico, al encontrar el símbolo o la palabra clave correspondiente sabe que lo que viene después es necesariamente un tipo y no una expresión, y que los símbolos `<` `>` que aparecen ahí deben entenderse como ángulos y no como menor-que y mayor-que.

A continuación se explica con detalle la sintaxis de llamada a funciones genéricas:
```
// la sintaxis de definición de la función no cambia
fn f<T>(arg: T) {}

// la sintaxis de llamada a la función sí cambia
fn main() {
    f(<Int>, 1); // se indica de forma explícita el tipo del argumento genérico
    f(1); // forma de inferencia de tipos, omitiendo el argumento genérico
}
```

El motivo principal de diseñar así la sintaxis es, primero, evitar una sintaxis tan fea como turbo fish, y segundo, encajar con la forma de implementar las funciones genéricas de zinc.

Sin considerar las optimizaciones del compilador en escenarios especiales, las funciones genéricas de zinc, desde el punto de vista de la implementación, se implementan por defecto mediante table passing. El compilador traduciría la función f anterior a:
```
// pseudocódigo C correspondiente a la definición de la función f:
void f(ZnTypeMeta * typeof_T, void * arg) { }
```

Al ejecutar una llamada como `f(<Int>, 1)`, quien llama realmente obtiene el ZnTypeMeta correspondiente a `Int` y lo pasa como argumento de función a f.
Por eso escribir la lista de argumentos genéricos de la función dentro de los paréntesis encaja mejor con la semántica de ejecución de las funciones genéricas de zinc.

El tipo de puntero a función de zinc puede contener genéricos: `let pf: fn<T>(T)->() = f;`.
zinc no permite currificación (currying) de funciones; escribir `let pf: fn(Int)->() = f(<Int>);` fallará al compilar.
Para este escenario de instanciación de funciones, úsese una lambda; esto sí es válido: `let pf: fn(Int)->() = .{ (arg: Int)->() => f(<Int>, arg); };` .

La principal ventaja de implementar así las funciones genéricas es que las funciones genéricas también se pueden distribuir mediante bibliotecas de enlace dinámico. Esa es una ventaja de diseño del lenguaje Swift.

Esto es muy útil en este escenario:
Supongamos que diseñamos una biblioteca de enlace dinámico `a.so` que expone una interfaz de función genérica y es usada por la aplicación `b.exe`.
Entonces podemos, al actualizar la implementación interna de `a.so` (por supuesto, sin hacer cambios destructivos de API), no recompilar `b.exe` y que el programa siga funcionando automáticamente con normalidad.
Si los genéricos se implementan por instanciación, este escenario es difícil de admitir. Al actualizar la biblioteca upstream, la aplicación downstream debe recompilarse y redesplegarse.

Además, este diseño trae otras ventajas:
1. Admite que las funciones `virtual` lleven genéricos. Sobre las funciones `virtual` se puede consultar el capítulo siguiente.
2. Reduce el code size y reduce el tiempo de compilación.

El coste, por supuesto, es una menor eficiencia de ejecución. Esto impide muchas optimizaciones del compilador, tampoco es amigable con la caché y perjudica el tiempo de ejecución.
Pero aún podemos, a nivel de implementación, en ciertos escenarios concretos, mediante opciones de compilación extra, permitir usar la instanciación de genéricos como medio de optimización de rendimiento, y lograr un mejor equilibrio entre code size y eficiencia de ejecución.
1. Al compilar un programa ejecutable, y no una biblioteca, se puede usar por completo la instanciación para implementar los genéricos
2. Incluso al compilar una biblioteca, siempre que no haya necesidad de distribuir y desplegar por separado una biblioteca de enlace dinámico, se puede usar la instanciación para implementar los genéricos

> **TODO**: por ahora no se permite que un tipo UnSized se use como argumento genérico, ni que un trait se use como argumento genérico.

> **Funcionalidad prevista**: más adelante se podrá admitir una condición "o" como condición de la cláusula where, como azúcar sintáctico de sobrecarga del tipo de parámetro de función:
```
fn f<T>(arg: T) where T: Int32 | UInt32 {}
// equivale a:
trait AnonymousTR {}
impl AnonymousTR for Int32 {}
impl AnonymousTR for UInt32 {}
fn f<T>(arg: T) where T: AnonymousTR {}
```

## trait

Ejemplo de sintaxis de definición de un trait:

```
trait TrName<T> : SuperTrait1 + SuperTrait2 where T: Condition {
    type AssocType;

    virtual fn method(&self);

    fn static_method() -> Self;
}
```

La definición de un trait admite:
1. Parámetros genéricos opcionales
2. Trait padre opcionales; se admiten varios trait padre, separados por `+`
3. Cláusula de condición where opcional

El cuerpo de la definición de un trait puede incluir:
1. Tipos asociados
2. Constantes asociadas
3. Funciones asociadas

### impl trait

Un trait puede usarse para hacer una abstracción unificada de distintos tipos. Ejemplo de sintaxis para indicar que un tipo concreto `SomeType` implementa un trait `TrName`:

```
impl TrName for SomeType {

}
```

Bloque impl
1. Las funciones internas no pueden indicar pub por separado
2. Tampoco pueden indicar virtual

No se permite impl Trait for Trait.

**Regla del huérfano (orphan rule)**

El principio básico es:

### Punteros a trait

Los dos escenarios de uso de un trait son:
1. Como cota superior de una restricción genérica
2. Como tipo de puntero que apunta a un trait

Las variables globales, las variables locales y los miembros no pueden usar de forma directa un trait como tipo. Pero sí pueden usar como tipo un puntero que apunta a un trait.

Si el tipo `S` implementa `trait R`, un puntero que apunta a `S` puede hacer upcast a un puntero que apunta a `R`. Ejemplo:

```rust
struct S {}
trait R {
    virtual fn f(&self);
}
impl R for S {
    fn f(&self) {}
}

fn test(p: *S) {
    let p1: *R = p; // ok, upcast
    p1.f();
}
```

### Funciones virtual

Un puntero que apunta a un trait puede llamar a las funciones miembro del trait. Pero hay una restricción: solo las funciones miembro marcadas con virtual pueden llamarse a través de un puntero que apunta a un trait.

> Nota:
> Rust permite usar la sintaxis `where Self: Sized` para marcar las funciones miembro de un trait e indicar que son "non-virtual". Esa sintaxis es muy extraña y difícil de entender. zinc introduce de forma directa una palabra clave para marcarlas, lo cual es más legible.
>

Ejemplo:

```rust
trait TR {
    virtual fn f1(&self);
    fn f2(&self);
}

fn test(p: &TR) {
    p.f1(); // OK
    p.f2(); // error de compilación, f2 no es una función virtual; no se permite llamarla a través de un puntero que apunta a un trait
}

fn test2<T>(arg: T) where T: TR {
    arg.f2(); // OK. Las funciones no virtual siguen pudiendo llamarse de forma genérica.
}
```

Al mismo tiempo, fíjate en que no todas las funciones escritas en un trait pueden modificarse con virtual. Las funciones que pueden modificarse con virtual deben cumplir los siguientes requisitos:
1. El nombre del primer parámetro debe ser `self`, y el tipo debe ser un tipo de puntero que apunta a `Self`. Incluye `&self` `&mut self` `*self` `*weak self`, todos válidos.
2. Salvo el primer parámetro, los demás parámetros no pueden usar el tipo `Self` ni tipos asociados
3. El tipo de retorno de la función no puede usar el tipo `Self` ni tipos asociados

Ejemplo:
```
trait TR {
    virtual fn clone()->Self; // error de compilación, no se permite modificarla con virtual. No hay parámetro self y el tipo de retorno usa Self.
    virtual fn g(arg: Int); // error de compilación, no se permite modificarla con virtual. No hay parámetro self.
    virtual fn h(self);  // error de compilación, no se permite modificarla con virtual. El parámetro self no es un tipo puntero.
}
```

**Nota**: las funciones virtual admiten introducir parámetros genéricos nuevos. Ejemplo:
```
trait TR {
    virtual fn f<T>(&self, arg: T) where T: Hash + Ord + Default; // OK
}
```

> **Funcionalidad prevista**: más adelante se podrá admitir la escritura `* Trait1 + Trait2 + 'a`:
1. Al usar `+`, no puede ser un tipo concreto; solo se pueden sumar trait y trait
2. Solo puede haber un tipo non-marker trait. Sumarse a sí mismo tampoco vale.
3. `* Trait1 + Trait2` puede convertirse de forma implícita a `*Trait1` o `*Trait2`

Esta característica puede ser una función necesaria en ciertos escenarios; por ejemplo, queremos `* MyTrait + Sync` para expresar que el tipo al que se apunta no solo cumple la restricción MyTrait, sino también la restricción Sync.
Con otros métodos no se puede expresar.

### Tipos asociados

Un tipo asociado es un placeholder de tipo definido en un trait; al implementar el trait hay que indicar el tipo concreto.

Ejemplo:
```rust
trait Container {
    type Item;
    
    fn add(&mut self, item: Self::Item);
    fn get(&self) -> Option<Self::Item>;
}

struct IntContainer {
    items: Vec<Int>,
}

impl Container for IntContainer {
    type Item = Int;
    
    fn add(&mut self, item: Int) {
        self.items.push(item);
    }
    
    fn get(&self) -> Option<Int> {
        self.items.last().copied()
    }
}
```

El tipo `Self` representa el tipo concreto que implementa el trait.

> **TODO**: por ahora no se admiten tipos asociados de orden superior (Higher-Kinded Types).

### Herencia de trait

En zinc los trait admiten herencia. Al definir un trait se pueden indicar cero o más trait padre.

En trait con relación de herencia, un puntero que apunta al trait hijo puede convertirse de forma implícita a un puntero que apunta al trait padre (tanto punteros ARC como BORROW). Ejemplo:
```
trait TR: TR1 + TR2 + TR3 {}

fn test(p: *TR) {
    let p1: *TR1 = p; // OK
    let p2: *TR2 = p; // OK
    let p3: *TR3 = p; // OK
}
```

Un puntero que apunta al trait padre también puede, mediante coincidencia de patrones, hacer downcast a un puntero al trait hijo. Por ejemplo, tanto `is` como `match` pueden hacerlo.

```
trait TR: TR1 + TR2 + TR3 {}

fn test(p: *TR1) {
    if (p is p1: *TR) { // se intenta el downcast; puede fallar
        // se usa la variable p1
    }

    match (p) {
        p2: *TR => {}
        _ => {}
    }
}
```

Otras funciones incluyen:
1. En el trait hijo se permite indicar el valor de associated type / const
2. En el trait hijo se permite override el cuerpo de las funciones del trait padre
3. Se admite herencia múltiple; hay que considerar el problema de conflicto de nombres de funciones en escenarios de herencia múltiple
    1. En herencia múltiple, se prohíbe el conflicto de nombres
    2. En herencia múltiple, añadir un tipo padre o ajustar el orden de herencia afecta al ABI
    3. `trait TR1 : TR2 {}`  y `trait TR1 where Self: TR2 {}` tienen significados distintos.

### reuse

El propósito del mecanismo reuse es reutilizar miembros y funciones miembro. reuse es un azúcar sintáctico que facilita al usuario delegar una implementación a otro tipo. Ejemplo:

```
struct S1 { }
impl TR1 for S1 {
    fn f1(&self, arg: Int) {}
}

struct S2 {
    base: S1  // el nombre del miembro no tiene restricción; puede ser cualquiera
}
impl TR1 for S2 {
    reuse self.base;  // el significado es que las funciones miembro de este bloque impl reutilizan todas las funciones miembro del mismo trait implementadas por self.base
}

// el código anterior equivale a:
impl TR1 for S2 {
    // se implementan todas las funciones miembro, y el cuerpo de cada una llama a la función miembro correspondiente de `self.base`.
    fn f1(&self, arg: Int) {
        self.base.f1(arg);
    }
}

```

### Trait integrados habituales

`Any` trait

`Fn` trait

`Send` trait

`Sync` trait

### Paradigma de programación orientada a objetos

Mediante la combinación de las características de lenguaje anteriores, zinc también puede admitir el paradigma de programación orientada a objetos, pero de forma distinta al estilo habitual del sector (como C++/Java/C#/Swift).
La característica principal de Zinc es: no admite `class` ni herencia de class. En Zinc solo un trait puede heredar, y solo un puntero que apunta a un trait puede hacer despacho dinámico.

A continuación se explica, desde varios aspectos, por qué se diseña así.

1. El tipo `class` de semántica de referencia no tiene un layout de memoria lo bastante flexible

    Desde el punto de vista del layout de memoria, se puede ver el class de semántica de referencia de otros lenguajes como la unión de `puntero ARC + estructura` en Zinc. Solo se permite usar semántica de referencia, no la estructura de semántica de valor correspondiente.
    Eso significa que, cuando una variable de tipo class se usa como variable local o miembro, siempre hay asignación dinámica de memoria y siempre hay una capa extra de indirección de puntero. En realidad eso es una degeneración de la capacidad de expresión, no un refuerzo.
    Si ofrecemos al usuario por separado el `puntero ARC` y la `estructura`, por ejemplo `Vec<S>` y `Vec<*S>` son tipos distintos y se pueden elegir según el escenario, eso es más flexible y el usuario tiene más libertad.

    Un class de semántica de referencia, en cualquier escenario, lleva dentro de cada objeto un puntero de “cabecera de objeto”. Este diseño de zinc, en cambio, no afecta al layout de memoria del objeto en sí.

2. Añadir un tipo `class` de semántica de referencia haría inconsistente el sistema de tipos de Zinc; muchos escenarios necesitarían reglas especiales y la implementación sería compleja

    Un class de semántica de referencia traería al sistema de tipos de Zinc una serie de reglas extra; no compensa. Sobre todo cuando se combinan distintos punteros con class.
    En zinc ya existen varios tipos de puntero, y un “class de semántica de referencia” en sí puede entenderse como la combinación de “puntero implícito + estructura”.
    Esta función no es ortogonal con otras características del lenguaje, porque ese “puntero implícito” debería ser ARC, weak o borrow, y el usuario no tiene capacidad de elección.

    Consideremos este escenario: cuando `T` es un class, ¿qué significa `&T`? ¿Ese borrow apunta a la variable puntero en sí, o a la dirección de inicio del objeto?
    Como class fuerza a unir el `puntero ARC` y la `estructura`, cualquier diseño elegido traerá problemas.
    Separar el puntero y la estructura hace la expresión semántica más clara. El tipo `&S` es un borrow de `S`; el tipo `&*S` es un borrow de `*S`. Semántica clara e implementación simple.

    Consideremos el siguiente ejemplo de escenario genérico; se verá que unificar en genéricos un struct de semántica de valor y un class de semántica de referencia es muy problemático:
    ```
    fn test<T>(arg: T) {
        // si introducimos un class de semántica de referencia, las expresiones de tomar borrow y de desreferenciar se comportan de forma distinta para tipos de semántica de valor y de semántica de referencia.
        // en un escenario genérico, necesitaríamos hacer en runtime algunos juicios extra para lograrlo y unificar ambas situaciones con genéricos.
        let p: &T = &arg;
        let c: T = *p;
    }
    ```

3. Si, como en C++, introducimos class y herencia, pero class no es de semántica de referencia, ¿vale?

    En C++ la diferencia semántica entre class y struct no es grande. Admite tipos valor y no está vinculado a un puntero.
    Pero la herencia de class significa que el tipo hijo debe aceptar todas las interfaces implementadas por el tipo padre, y eso es problemático.

    En el diseño tradicional de herencia, el class hijo necesariamente implementa todas las interfaces (interface/protocol) del class padre.

    Consideremos un trait como `Send` en Zinc, que exige que todos los miembros cumplan `Send`. Si introdujéramos herencia, aparecería esta situación: el tipo padre implementa `Send`,
    el tipo hijo hereda del padre pero añade un miembro non-`Send`; queremos que el tipo hijo reutilice todos los miembros y métodos del padre, pero no implemente `Send`; con el diseño tradicional de herencia eso no se puede.

    En un diseño que usa herencia, el tipo hijo no puede elegir implementar solo una parte de las interfaces del padre. Para alcanzar el objetivo anterior, también hay que abandonar la herencia y pasar a composición. Así que la herencia solo facilita la reutilización de código en algunos escenarios; no se adapta a todos.

4. La herencia de class necesita diseñar reglas especiales para escenarios especiales

    1. Un escenario típico es la herencia múltiple.
    Según el diseño tradicional que admite herencia, si admitimos que un class herede de varios class, hay que considerar el escenario de “herencia en diamante”.
    C++ diseñó reglas sintácticas y semánticas especiales para este escenario, y muchas normas de programación también imponen muchas restricciones a esta característica.
    Lenguajes como Java/C#/Swift no admiten de forma directa la herencia múltiple, lo que limita la capacidad de reutilización de código y simplifica las reglas semánticas.

    2. Otro escenario típico es el de los constructores. Las reglas de cada lenguaje son distintas. En C++, llamar a una función virtual desde un constructor no tiene efecto de función virtual.
    Los constructores de Java/C# no acceden a variables no inicializadas; dependen de la “inicialización a valor 0” de los campos.
    Swift no puede admitir “inicialización a valor 0” y además quiere que un constructor nunca acceda a un miembro no inicializado, así que sus reglas son las más complejas.
    El constructor restringe el nombre de la función, lo que significa que admitir constructores implica admitir sobrecarga de funciones basada en el tipo; el constructor restringe el tipo de retorno, lo que significa que no se puede hacer manejo de errores mediante el tipo de retorno.
    Estas ideas de diseño no encajan con zinc.

    En resumen, los lenguajes tradicionales admiten herencia de class, pero esa característica trae algunas reglas semánticas especiales extra; las reglas de cada lenguaje son distintas y no es muy estético.

Zinc sigue la línea de diseño de Rust y adopta un diseño más conciso: solo se permite herencia de trait.

El diseño de Zinc es más fácil de entender y usar para el usuario:
* Toda reutilización de miembros adopta el modo de “composición”;
* Toda reutilización de métodos miembro adopta el “mecanismo reuse”;
* Todos los escenarios que necesitan unificar una interfaz pública para varios tipos se resuelven con “trait+impl”;
* Todos los escenarios que necesitan despacho dinámico se resuelven con un “puntero que apunta a un trait”.

Las reglas anteriores aplican a todos los tipos (tipos integrados, enum, struct, tuple, cadenas, arrays, etc.), sin casos especiales. Las reglas anteriores bastan para admitir por completo el paradigma OOP. No hay problema de capacidad de expresión; las funciones que admiten los lenguajes tradicionales se pueden expresar, solo que en algunos casos la sintaxis es un poco más tediosa.

Las reglas semánticas son concisas, consistentes y ortogonales.

### Layout de memoria

Un puntero que apunta a un trait es un puntero gordo. Los tipos de puntero mencionados en esta sección incluyen estas seis: `*T`, `*weak T`, `&T`, `&mut T`, `*raw T`, `*raw mut T`.

```rust
trait TRBase { virtual fn f1(&self); }
trait TRSub : TRBase { virtual fn f2(&self); }

struct S { i: Int }
impl TRBase for S { fn f1(&self) {} }
impl TRSub for S { fn f2(&self) {} }

fn main() {
    let p1: *S = box S{.i = 1}; // p1 es un puntero delgado
    let p2: *TRSub = p1; // p2 es un puntero gordo
    let p3: *TRBase = p2; // upcast

    if p3 is psub: *TRSub {
        // downcast
    }
    if p2 is ps: *S {
        // downcast
    }
}
```

El layout del puntero gordo de Zinc es distinto del de Rust. Un puntero que apunta a un trait contiene 3 miembros:
```
      p2
┌───────────────────┐
│ object_ptr        │→ apunta a la cabecera del objeto. El objeto puede ser un tipo integrado, un struct definido por el usuario, un enum, etc.
│───────────────────│
│ typemeta_ptr      │→ apunta al TypeMeta del tipo concreto del objeto; contiene name, size y demás información del tipo, y también punteros al destructor, a la función de copia, etc.
│───────────────────│
│ vtable_ptr        │→ apunta a la tabla de funciones virtuales de este tipo concreto para este trait; es un array de punteros a función
└───────────────────┘
```

* Un puntero que apunta a un trait admite llamar a las funciones miembro virtual del trait. Al llamar a una función miembro, se toma el puntero a función de la posición correspondiente de la vtable y se llama; el desplazamiento se determina en tiempo de compilación.

* Un puntero que apunta a un trait admite, mediante coincidencia de patrones, hacer conversión a punteros de otros tipos, porque el puntero gordo guarda la información de tipo en runtime del objeto. Podemos juzgar en runtime a qué tipo concreto apunta este puntero a trait; si la coincidencia tiene éxito, se toma el object_ptr que contiene y se usa como puntero al tipo concreto. También se admite downcast: `*TRBase` puede juzgar en runtime si es de tipo `*TRSub`.

* Un puntero que apunta a un trait admite upcast. El tipo `*TRSub` puede convertirse al tipo `*TRBase`.

> Nota:
> En Rust, un puntero que apunta a un trait se llama trait object; también es un puntero gordo, pero no admite downcast. Solo el trait especial `Any` puede admitir downcast; los demás trait no.
> Porque solo tiene el tamaño de dos punteros: es un poco gordo, pero no lo bastante gordo como para admitir todos los paradigmas de programación orientada a objetos que Zinc espera admitir.
>
> Tomando como referencia el layout de memoria del Protocol de Swift, se puede pensar que solo hace falta añadir un puntero más. Los dos punteros a metadatos que Zinc diseñó para el puntero gordo pueden compararse con la value witness table y la protocol witness table de swift.
>

## Problemas pendientes:

1. ¿No permitir impl trait para tipos puntero tendrá algún problema de capacidad de expresión? (se piensa introducir más adelante una sintaxis especial en los parámetros de función, equivalente a un AsRef/AsMut trait integrado en el lenguaje)

2. ¿No permitir impl trait para un tipo arbitrario tendrá algún problema de capacidad de expresión?
    ```
    impl<T> ToString for T where T: Display {} // esta escritura por ahora no se admite. Se recomienda usar herencia de trait en su lugar.
    ```

3. ¿Cómo permitir que un UnSized type se use como argumento genérico? Por ahora no se puede implementar un tipo como `Mutex<File>`; un `MutexFile` fijo tiene una composabilidad demasiado débil.

4. ¿Hay que admitir impl TR como tipo de retorno?

5. ¿Hay que admitir const generics? Fíjate en que swift no lo admite; esto probablemente plantea un gran desafío al runtime y al ABI.

## Covarianza y contravarianza

|           |   'a   |   T   |   U   |
|-----------|--------|-------|-------|
| &'a T     |  covariante  |  covariante  |       |
| &'a mut T |  covariante  |  invariante  |       |
| *T        |        |  covariante  |       |
| Mutable<T>|        |  invariante  |       |
| Vec<T>    |        |  covariante  |       |
| fn(T)->U  |        |  contravariante  |  covariante  |
| *raw T    |        |  covariante  |       |
| *raw mut T|        |  invariante  |       |

Más adelante se podrá seguir admitiendo las funciones de covarianza y contravarianza de tipos.
Las palabras clave `in` `out` ya están reservadas.

Ejemplos de escenarios de uso:
1. Convertir `Option<*Sub>` a `Option<*Base>`
2. Convertir `Result<*SubR, *SubE>` a `Result<*BaseR, *BaseE>`
3. Convertir `(*Sub, *Sub)` a `(*Base, *Base)`
4. Convertir `fn(*Base)->*Sub` a `fn(*Sub)->*Base`
