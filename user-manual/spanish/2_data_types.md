
# Tipos de datos básicos

Convención de nombres: todos los tipos con nombre usan UpperCamelCase. (excepto la palabra clave `fn`, porque se usa principalmente en definiciones de funciones.)

## Tipo Bool

El tipo `Bool` tiene dos valores posibles, `true` y `false`.

La negación de `Bool` usa el operador `!`.

## Enteros

|  Longitud |  Con signo  |  Sin signo  |
|-------|---------|----------|
| 8 bit | Int8      |  UInt8      |
| 16 bit | Int16      |  UInt16      |
| 32 bit | Int32      |  UInt32      |
| 64 bit | Int64      |  UInt64      |
| arch-dependent | Int      |  UInt  |
| arch-dependent | Short    | UShort  |

El tamaño de Int/UInt depende de la plataforma objetivo. En plataformas de 64 bit son de 64 bit; en plataformas de 32 bit son de 32 bit. Equivalen a `intptr_t` / `uintptr_t` de C.
El tamaño de Short/UShort depende de la plataforma objetivo y siempre es la mitad de Int/UInt. En plataformas de 64 bit son de 32 bit; en plataformas de 32 bit son de 16 bit.


Los literales enteros permiten `_` como separador.

Se permiten prefijos para distinguir la base: `0o` representa octal, `0x` hexadecimal y `0b` binario.

Los literales enteros permiten sufijo de tipo. El sufijo de tipo va en minúsculas, por ejemplo `123_i8`  `100_i`  `0x64_u`.
Si no hay otra información de contexto, un literal entero no puede inferir su tipo. Se recomienda que el usuario añada el sufijo de tipo al literal entero para indicar el tipo de forma explícita.

En las operaciones aritméticas enteras, el overflow siempre es en modo wrapping. Si se necesita otro comportamiento, hay que llamar a la biblioteca estándar. La biblioteca estándar ofrece a los enteros funciones miembro como `checked_add` (aún no implementadas).

zinc no ofrece operadores de "operaciones de bits", incluidos desplazamiento a la izquierda, desplazamiento a la derecha, AND bit a bit, OR bit a bit, NOT bit a bit y XOR bit a bit.
En su lugar, la biblioteca estándar ofrece a los tipos enteros funciones miembro para las operaciones de bits, por ejemplo `bit_and`, suficientes para cubrir la funcionalidad.

Motivo: las operaciones de bits se usan con poca frecuencia y solo aplican a tipos enteros; diseñar operadores específicos es un tanto desperdicio.
Además, las operaciones de bits no se limitan a las anteriores: también hay rotate, reverse, count_ones y otras decenas de operaciones de bits poco habituales. Esas, de todas formas, solo pueden ofrecerse como funciones miembro. Personalmente, me parece más coherente ofrecer todas las operaciones de bits como funciones miembro.
Los símbolos del código ASCII ya escasean; es más razonable reservarlos para otras funciones más importantes.


## Números de punto flotante

Los números de punto flotante se dividen en los tipos `F32` y `F64`.

Los literales de punto flotante permiten `_` como separador.
Los literales de punto flotante permiten sufijo de tipo, `f32` y `f64` respectivamente.

Los literales de punto flotante no permiten omitir el 0 delante del punto decimal. `.123` o `-.1` dan error.

## Tipo de carácter Char

El tipo `Char` es de 32 bit y representa un unicode code point.

Los literales `Char` van entre comillas simples. Por ejemplo `'好'`.

Los literales `Char` admiten escape. `'\u{3456}'`.

Se admiten literales Byte, `b'a'`. El valor de un literal Byte solo puede ser un carácter ascii y su tipo es `UInt8`.

En la biblioteca estándar hay un alias de tipo `type Byte = UInt8;`.

## Arrays

Los elementos de un array son todos del mismo tipo. La longitud del array debe ser una constante en tiempo de compilación.

Un array cuyos elementos son de tipo `T` y cuya longitud es `N` se escribe `[T; N]`. Los arrays son tipos valor; al asignar, pasar como parámetro o devolver un array se copian todos los elementos.

Ejemplo:

```rust
fn main() {
    let x: [Int; 3] = [1, 2, 3];
}
```

Los literales de array tienen dos formas:

1. Enumerar todos los elementos entre corchetes, separados por comas.

    `[1, 2, 3]`

2. Separar el elemento y el recuento con un punto y coma entre corchetes, lo que representa varios elementos repetidos.

    `[1; 3]`

Para acceder a un elemento del array se usa una expresión de índice; si se sale de rango se produce un panic. Ejemplo:

```rust
fn main() {
    let mut x: [Int; 3] = [1, 2, 3];
    x[1] = 10;
    println(x[5]); // la compilación pasa; en ejecución se produce un panic
}
```

## Slices

Un slice de array es un puntero borrow que apunta a una parte de un array. Hay dos tipos de slice de array, `Slice<'a, T>` y `SliceMut<'a, T>`, que representan permiso de solo lectura y permiso de lectura y escritura respectivamente.

```rust
fn main() {
    let x: [Int; 3] = [1, 2, 3];
    let s: Slice<Int> = &x;
    println(s.len());
    println(s[1]);
}
```

Un slice de array es una estructura con lifetime; puede verse como un puntero borrow con metadatos adicionales. En el ejemplo anterior, la variable `s` ocupa el tamaño de dos punteros, que guardan la dirección de inicio y la longitud respectivamente.

Nota: el motivo de no usar la sintaxis `&[T]` de rust es querer simplificar el sistema de tipos y la dificultad de implementación.
En rust, `[T]` es un tipo válido, se puede usar de forma independiente y es dyn sized. Eso introduce un caso especial en el sistema de tipos y complica mucho la implementación de genéricos. zinc espera que el usuario solo pueda usar los tipos `&[T]/&mut [T]`, sin un tipo `[T]` dyn sized independiente, y tampoco tipos como `Box<[T]>` `Rc<[T]>`, etc.
Por eso se sustituyó la sintaxis `&[T]/&mut[T]` y solo se ofrecen en la biblioteca estándar los tipos `Slice/SliceMut`. Su semántica equivale a exigir en rust que `[T]` solo se use junto con un puntero borrow. El `Slice<'a, T>` de Zinc equivale al `&'a [T]` de Rust.

> **TODO**: el tipo Range aún no está implementado, pero el operador ya está reservado, con la sintaxis de Swift. Hay dos tipos de Range, intervalo abierto `..<` e intervalo cerrado `..=`; se elimina `RangeFull`.
Por ahora las operaciones de slice aún no se pueden implementar con el operador de índice `[]` junto con el tipo Range, porque la sobrecarga de operadores aún no está implementada.
```
let s: Slice<Int> = &x[1..<3]; // se admitirá más adelante
```

## Cadenas

Hay dos tipos de cadena, `Str` y `String`.

`Str` es una estructura con lifetime definida en la biblioteca estándar, `struct Str<'a>`. El tipo `String` es una estructura sin lifetime definida en la biblioteca estándar e implementada sobre `Vec<UInt8>`.

Los literales de cadena son de tipo `Str<'static>`.

Nota: de forma similar a los slices de array, el motivo de no usar la palabra clave `str` de rust es querer simplificar el sistema de tipos y la dificultad de implementación.
En rust, `str` es un tipo válido, se puede usar de forma independiente y es dyn sized. Eso introduce un caso especial en el sistema de tipos y complica mucho la implementación de genéricos. zinc espera que el usuario solo pueda usar el tipo `&str`, sin un tipo `str` dyn sized independiente, y tampoco tipos como `Box<str>` `Rc<str>`, etc.
Por eso se sustituyó la palabra clave `str` y solo se ofrece en la biblioteca estándar el tipo `Str`. Su semántica equivale a exigir en rust que `str` solo se use junto con un puntero borrow de solo lectura. El `Str<'a>` de Zinc equivale al `&'a str` de Rust.

El interior de las cadenas está codificado en utf8.

Pensándolo con mentalidad de C++, el `String` de zinc corresponde al `std::string` de C++, solo que la implementación interna usa un mecanismo de copy-on-write. El `Str` de zinc corresponde al `std::string_view` de C++, con comprobación de lifetime en tiempo de compilación adicional.

Los literales de cadena admiten escape.

todo: admitir raw string literal. `r##" content "##`.

todo: admitir interpolación de cadenas (string interpolation), es decir, expresiones anidadas dentro de literales de cadena. Ejemplo:

```
let name: Str = "Amy";
let s: String = f"hello $(name), how are you?";
```
Se eligen paréntesis en lugar de llaves porque en zinc los paréntesis se usan generalmente en escenarios de expresiones. Las expresiones tienen tipo y valor. Las llaves se usan generalmente en escenarios de sentencias; las sentencias no tienen tipo ni valor.

TODO: por ahora las expresiones embebidas solo admiten el caso más simple de un único nombre de variable; más adelante se admitirán todas las expresiones.

## Tuplas (tuple)

Una tupla de 0 elementos se llama tipo unidad (unit type). El tipo unidad se escribe `()`. El tipo de la expresión de tupla vacía `()` es el tipo unidad.
Si se omite el tipo de retorno de una función, significa que la función devuelve el tipo `()`.

Una tupla de 1 elemento necesita una coma extra al final; de lo contrario el compilador la interpreta como “expresión entre paréntesis” y no como tupla. `(1,)` es una tupla, `(1)` no lo es.

Una tupla de 2 o más elementos puede tener elementos de distintos tipos.

## Estructuras (struct)

Se usa la palabra clave `struct` para definir una estructura. Ejemplo:

```rust
struct User {
    active: Bool,
    username: String,
    email: String,
}
```

Las estructuras admiten genéricos.
Tanto la estructura como sus miembros pueden modificarse con pub.
Se permite que no haya miembros dentro de las llaves.

Los miembros de una estructura admiten valores por defecto; el valor por defecto debe ser una expresión constante.
```rust
struct User {
    active: Bool = true, // valor por defecto
    username: String = f"",
    email: String = f"",
}
```

La expresión de inicialización de una estructura es similar a la de C:

```rust
let user: User = {  // se indica el tipo de la expresión de inicialización anotando el tipo del binding de la variable
    .active = false,
    .username = "John".to_string(),
    .email = "john@gmail.com".to_string(),
};
// o bien
let user = {  // se indica el tipo de la expresión de inicialización con un sufijo de tipo
    .active = false,
    .username = "John".to_string(),
    .email = "john@gmail.com".to_string(),
} : User;
```

Las estructuras no admiten la inicialización anónima de miembros en orden, similar a C:
```
let user: User = { false, "John".to_string(), "john@gmail.com".to_string() };  // error
```

Los miembros con valor por defecto pueden no inicializarse de forma explícita.

Se admite inicializar una estructura con otra. Fíjate en que aquí se usan tres puntos `...`, y no los dos puntos `..` de rust.
En zinc se usan de forma unificada, tres puntos `...` para representar elipsis. La sintaxis de elipsis se usa en varios escenarios. Después de la elipsis puede haber o no una expresión.

```rust
// active puede no escribirse; significa que usa el valor por defecto
let user1: User = { .username = "smith".to_string(), .email = String::new() };

// se usa user1 para inicializar user2; solo cambia el valor del miembro active
let user2: User = { .active = true, ...user1 }; 

// se usan los valores por defecto de la definición del tipo User para inicializar user3; solo cambia el valor del miembro active
let user3: User = { .active = true, ... }; 

// esto equivale a inicializar todo con los valores por defecto de la definición del tipo User.
// hace falta la elipsis; de lo contrario el análisis léxico no puede distinguir una inicialización de estructura vacía de un bloque de sentencias vacío
let user4: User = { ... }; 
```

Las estructuras no admiten herencia. zinc piensa admitir el paradigma de programación orientada a objetos mediante herencia de trait y el mecanismo `reuse`.

zinc no piensa admitir tuple struct, es decir, estructuras cuyos miembros no tienen nombre. Se recomienda que, para estructuras simples, se ofrezca por separado una función global con nombre snake_case correspondiente, para usarla como “constructor”.
Así las reglas del lenguaje son más concisas y unificadas, y desde el punto de vista del uso las convenciones de nombres también son más unificadas. No aparecen cosas “parecidas a funciones” que empiezan por mayúscula.
Por ejemplo:

```rust
struct Weight {
    w: UInt32
}

fn weight(w: UInt32) -> Weight {
    return { .w = w };
}

fn main() {
    // el lado de la definición es un poco más trabajoso; el lado del uso es básicamente igual que un tuple struct
    let w = weight(1); 
}
```

zinc admite “estructuras unidad”. Una estructura sin miembros puede terminar en punto y coma, o tener las llaves vacías:
```rust
// las dos escrituras siguientes son iguales
struct Unit{} 
struct Unit;
```
Pero fíjate en que la inicialización de una estructura unidad sigue necesitando llaves:
```
let x: Unit = { ... };
```

## Enumeraciones (enum)

Las enumeraciones se definen con la palabra clave `enum`:

```rust
enum IpAddrKind {
    V4,
    V6,
}

fn main() {
    let k: IpAddrKind = IpAddrKind::V4;
}
```

Los miembros de una enumeración admiten valores asociados.
```rust
enum IpAddr {
    V4(Byte, Byte, Byte, Byte),
    V6(String),
}
```

No se admiten valores asociados con llaves. Si el usuario necesita nombrar los parámetros, se recomienda envolver todos los parámetros en una estructura; el efecto es el mismo:

```rust
enum E {
    A{ x:Byte, y: String } // error
}
// escritura alternativa:
struct EA {
    x: Byte,
    y: String,
}
enum E {
    A(EA),
}
```

Se admite sobrecarga de miembros de enumeración; las reglas son las mismas que las de sobrecarga de funciones: solo se admite sobrecarga con distinto número de parámetros.

```rust
enum Arg {
    Integer(UInt),
    Integer(UInt, UInt),
    Integer(UInt, UInt, UInt),
    Float(F64),
    Float(F64, F64),
    Float(F64, F64, F64),
}
```

Una enumeración es en esencia una unión etiquetada (tagged union). Se permite que un enum definido por el usuario indique de qué tipo entero es el tag y el valor de etiqueta de cada miembro.
El valor de etiqueta debe ser una expresión constante, debe poder expresarse en el entero correspondiente y el valor de etiqueta de cada miembro no puede repetirse.

Solo se permite que un enum sin valores asociados indique el valor del tag.

```rust
enum IpAddr : UInt8 {
    V4 = 1,  //
    V6 = 2,
}
```

Se permite que un enum sin valores asociados se convierta de tipo de forma directa a un entero con el operador `as`.

La definición de un tipo enumeración admite genéricos. Los miembros de la enumeración no pueden introducir genéricos nuevos.

> **Funcionalidad prevista**: admitir la definición de enumeraciones no exhaustivas (non-exhaustive enum); la sintaxis es usar elipsis después del último miembro:
```rust
enum Puctuation {
    Comma,
    Dot,
    Semi,
    ...  // elipsis, equivalente a modificar el enum con #[non_exhaustive] en Rust
}
```

### Tipos puntero

Se explican en el capítulo 4.

### Algunos tipos enumeración especiales definidos en la biblioteca estándar

1. `Option`

    ```rust
    enum Option<T> { None, Some(T) }
    ```

    El tipo Option tiene una optimización especial de layout de memoria.

2. `Result`

    ```rust
    enum Result<T, E> { Ok(T), Err(E) }
    ```

    Result admite el operador `?`, que facilita el manejo de errores.

## Uniones (union)

Las uniones se definen con la palabra clave `union`. El tamaño total de una unión depende del tamaño del miembro más grande.

```
union U {
    data1: Byte,
    data2: UInt,
}
```

El tipo union es inseguro; leer y escribir miembros de un tipo union debe hacerse en un contexto unsafe.

La sintaxis de inicialización de una union es similar a la de inicialización de un struct. Pero se exige que solo se elija uno de los miembros para inicializar.

Los tipos non-trivial no pueden usarse como miembros de un tipo union.
Los tipos non-trivial incluyen:
1. Tipos con destructor, por ejemplo Arc/Weak, etc.
2. Tipos con parámetros de lifetime, por ejemplo &/&mut, etc.
3. Si un miembro de un struct es de tipo non-trivial, ese struct es non-trivial
4. Si un valor asociado de un enum es de tipo non-trivial, ese enum es non-trivial
5. Si el elemento de un tipo array es de tipo non-trivial, ese array es non-trivial
6. Si un elemento de un tipo tuple es de tipo non-trivial, esa tuple es non-trivial

La definición de un tipo union no puede llevar parámetros genéricos.

Las union tienen principalmente dos escenarios de uso:
1. Escenarios de interacción con C
2. Escenarios de conversión de tipo bit a bit (similar a `std::mem::transmute` de rust)

Por ejemplo, si quiero convertir bit a bit de tipo `F32` a tipo `UInt32`, puedo usar una union para hacerlo:
```rust
union FloatOrInt {
    f: F32,
    i: UInt32,
}

fn bit_cast_f32_to_uint32(f: F32) -> UInt32 {
    let u: FloatOrInt = { .f = f };
    return unsafe(u.i);
}
```

## Tamaño de los tipos

Los tipos se dividen en Sized y UnSized. Los tipos Sized pueden usarse como variables locales, variables globales y miembros; los tipos UnSized no, y solo pueden accederse de forma indirecta mediante punteros.

Los tipos UnSized incluyen:
1. Tipos declarados con `extern type`
    * `extern type` se usa principalmente en escenarios de interoperabilidad. Su significado es declarar que la definición de un tipo está en otro lenguaje y que en zinc no se puede obtener esa definición. Por tanto no se conoce su tamaño y no se puede acceder de forma directa a sus miembros.

2. Todos los trait
    * Supongamos que se define `trait R`: no se puede declarar que el tipo de una variable es `R`, pero sí `&R / &mut R / *R`, etc.

3. Tipos personalizados `S` con `impl UnSized for S {}` explícito
    * `UnSized` es un trait integrado que permite al usuario indicar que un tipo personalizado también es Unsized, mediante `impl UnSized for MyType {}`.
        El propósito es prohibir que ese tipo se use como “tipo con semántica de valor” y solo permitir usarlo mediante un puntero; ese tipo solo necesita ser un “tipo con semántica de referencia”.
        Un escenario típico es el tipo `File` de la biblioteca estándar. Ese tipo no permite que el usuario lo use como “tipo con semántica de valor”; el usuario solo puede obtener un tipo puntero que apunte a `File`.

En escenarios genéricos, por ahora no se permite que un tipo UnSized sea argumento genérico.
