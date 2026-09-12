
# Coincidencia de patrones

La coincidencia de patrones se usa principalmente en expresiones `is` y sentencias `match`.

El propósito de la coincidencia de patrones es comprobar si la estructura de una expresión es la esperada, y también permite extraer valores internos y vincularlos a variables nuevas.

Ejemplo:

```
// la variable x es una tuple de 3 elementos; se vincula el segundo elemento a la variable middle y se comprueba si middle > 5
if (x is (_, middle, _)) && middle > 5 {

}

// el tipo de la variable y es un puntero a trait; de esta forma podemos comprobar cuál es el tipo concreto al que apunta y. Si la comprobación tiene éxito, ese puntero se vincula a p.
if y is p: *S {
    // p can be used inside block
}

// la variable z es de tipo Option<S>; a continuación se procesan por separado los escenarios Some y None
match z {
    Some(s) => {

    }
    _ => {}
}
```

Al igual que las expresiones, los patrones también pueden anidarse.

## WildcardPattern

Patrón comodín.

`_` el guion bajo puede ocupar una posición.

`...` la elipsis puede ocupar varias posiciones.

## LiteralPattern

Se permite que literales de tipos como enteros, punto flotante, Bool, Char, etc. se usen de forma directa como patrones.

El patrón literal de `Str` aún no está implementado.

## TypePattern

El patrón de tipo se usa principalmente para comprobar si una expresión es un subtipo concreto. El patrón de tipo empieza con la palabra clave `type`.

Ejemplo:
```
// supongamos que TR es un nombre de trait y S es un nombre de struct.
// la siguiente sentencia comprueba si un puntero a trait puede hacer downcast con éxito a un puntero a S.
fn test(p: &TR) {
    let b = p is type &S;  // se usa la palabra clave type al principio para evitar un conflicto de sintaxis con el patrón posterior modifier+identifier
}
```

## StructPattern

El patrón de estructura se usa para coincidir con valores de tipo estructura y puede desestructurar sus miembros.

Ejemplo:
```rust
struct Point { x: Int, y: Int }

fn test(p: Point) {
    // coincidir con la estructura y vincular los miembros a variables nuevas
    if p is { .x = a, .y = b }: Point {
        println(f"x = $(a), y = $(b)");
    }
    
    // usar elipsis para ignorar los demás miembros
    if p is { .x = 10, ... }: Point {
        println("x is 10");
    }
    
    // coincidencia de patrones anidada
    match p {
        { .x = 0, .y = 0 } => println("origin");
        { .x = 0, .y = y } => println(f"on y-axis at $(y)");
        { .x = x, .y = 0 } => println(f"on x-axis at $(x)");
        { .x = x, .y = y } => println(f"point ($(x), $(y))");
    }
}
```

## TuplePattern

El patrón de tupla se usa para coincidir con valores de tipo tupla y puede desestructurar sus elementos.

Ejemplo:
```rust
fn test(t: (Int, String, Bool)) {
    // coincidir con la tupla y vincular los elementos a variables nuevas
    if t is (num, str, flag) {
        println(f"num = $(num), str = $(str), flag = $(flag)");
    }
    
    // usar guion bajo para ignorar algunos elementos
    if t is (1, _, true) {
        println("first element is 1 and third is true");
    }
    
    // patrón de tupla anidado
    let nested = ((1, 2), (3, 4));
    if nested is ((a, b), (c, d)) {
        println(f"($(a), $(b)), ($(c), $(d))");
    }
}
```

## EnumVariantPattern

El patrón de variante de enumeración se usa para coincidir con una variante concreta de un tipo enumeración y puede desestructurar sus valores asociados.

Ejemplo:
```rust
enum Message {
    Quit,
    Move(Int, Int),
    Write(String),
}

fn test(msg: Message) {
    match msg {
        Message::Quit => println("quit");
        Message::Move(x, y) => println(f"move to ($(x), $(y))");
        Message::Write(text) => println(f"write: $(text)");
    }
}
```

## ModifierPattern

El patrón de modificador se usa para controlar la forma de vinculación de variables en la coincidencia de patrones.

- `mut`: marca la variable vinculada como mutable
- `&`: coincide con una referencia
- `&mut`: coincide con una referencia mutable
- `ref`: vincula por referencia, no por movimiento
- `ref mut`: vincula por referencia mutable

Ejemplo:
```rust
fn test(s: String) {
    // modificador mut: hace mutable la variable vinculada
    let mut x = s;
    x.push_str(" modified");
    
    // modificador ref: vincula por referencia, evitando el movimiento

    // modificador ref mut: vincula por referencia mutable

}
```

## IdentifierPattern

El patrón de identificador se usa para vincular el valor coincidente a un nombre de variable.

Ejemplo:
```rust
fn test(x: Int) {
    // vinculación simple por identificador
    if x is y {
        println(f"y = $(y)");
    }
    
    // uso en match
    match x {
        0 => println("zero");
        n => println(f"other: $(n)");
    }
    
    // combinado con otros patrones
    let tuple = (1, 2, 3);
    if tuple is (first, ...) {
        println(f"first = $(first)");
    }
}
```



## Aún no admitidos
StrPattern
GroupedPattern
MacroInvocationPattern
RangePattern
SlicePattern
Pattern Guard
Or pattern
