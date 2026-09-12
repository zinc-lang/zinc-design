## Expresiones, sentencias y funciones

## Expresiones

zinc no es un lenguaje de expresiones.
1. Palabras clave como `break` `continue` `return` `goto` `panic` solo pueden usarse en sentencias, no de forma directa en expresiones.
2. Solo las expresiones pueden producir un valor. Las sentencias no.
3. Una sentencia puede contener expresiones, pero una expresión no puede contener sentencias de forma directa. La excepción es la expresión lambda, que sí puede contener sentencias, pero el return/break/continue de esas sentencias solo afecta al interior de la lambda; hacia fuera la lambda sigue comportándose como una expresión y no puede transferir el flujo de control.

Si el usuario necesita, dentro de una expresión, ejecutar en orden sentencias relativamente complejas y producir al final un valor, debe usar una lambda e invocarla de inmediato.
Las sentencias de control de flujo del interior de la lambda solo actúan en el cuerpo de la lambda y no afectan a la expresión externa.

### Expresiones de operador

#### Tipo `Result`

Zinc usa `Result<R, E>` como mecanismo principal de manejo de errores. Es un enum definido en la biblioteca estándar:
```
enum Result<R, E> {
    Ok(R), Err(E)
}
```

La `E` de `Result<R, E>` representa la información de error. El usuario puede usar cualquier tipo para describir la información de error.

* Si las posibilidades de error son muy limitadas, diseñar un enum como información de error es muy razonable.

* Si solo necesitamos la información de error para imprimir un log, usar simplemente una cadena como tipo de error tampoco es problema.

* Si necesitamos clasificar los tipos de error en varios niveles, diseñar un conjunto `trait MyBussinessError` y su jerarquía de herencia, y usar `*MyBussinessError` como tipo de error de retorno, también es un buen diseño.

* Si quien define la función realmente no necesita que quien la llama se preocupe del tipo de error concreto, basta con devolver de forma unificada `Result<R, *Any>`. Cualquier tipo de error, tras hacerle box, puede convertirse a `*Any`.
Quien llama puede hacer downcast según necesite y procesar solo la parte de tipos de error que le interesa.

* Si queremos que quien llama pueda obtener la pila de llamadas en el momento del error, se puede envolver el tipo `BackTrace` de la biblioteca estándar en un tipo personalizado y usarlo como tipo de error.
El tipo `BackTrace` puede describir la pila de llamadas actual; basta con pasarlo hacia fuera como parte de la información de error.

#### Operador `?`

El operador `?` es un operador sufijo; se usa así: `expr?`.

El operador `?` está pensado principalmente para usarse junto con el tipo `Result`. El operador `?` exige que el tipo de la expresión que lo precede sea `Result`.

> **Funcionalidad prevista**: más adelante se admitirá la sobrecarga del operador `?` para que pueda admitir tipos personalizados.

La semántica de ejecución de la sentencia `let x = expr?;` es:

```
match(expr) {
    Ok(r) => {
        let x = r;
    }
    Err(e) => {
        return Err(e);
    }
}
```

Como provoca el retorno anticipado de toda la función, impone requisitos al tipo de retorno de esa función.
Si el tipo de `expr` es `Result<R1, E1>` y el tipo de retorno de la función es `Result<R2, E2>`, se exige que `E2` pueda convertirse de forma implícita y natural a `E1`, o que `E2` implemente el trait `From<E1>`.
Solo así el compilador es capaz de convertir `E1` a `E2` y devolverlo.

### Expresiones entre paréntesis

Se permite `unsafe(<expr>)`; en ese caso la expresión `<expr>` está en un contexto unsafe. Y el valor de esa expresión es igual al valor de la expresión `<expr>`.

Se permite `const(<expr>)`; en ese caso la expresión `<expr>` está en un contexto const. Y esa expresión debe evaluarse en tiempo de compilación.

### Expresiones de índice

### Expresiones de llamada a función

Llamada a función

Llamada a método

### Expresiones de acceso a miembros

miembros de struct

miembros de tuple

### Expresiones de cierre (closure)

La sintaxis de closures de zinc es distinta de la de rust. Al diseñar la sintaxis de closures, zinc tuvo principalmente estas consideraciones:
1. La sintaxis de closures debe encajar con el azúcar sintáctico de “llamada trailing”. La llamada trailing permite diseñar DSL muy estéticos y elegantes en muchos escenarios. Funciones como lock / spawn / lazy encajan muy bien con la sintaxis de closures trailing.
2. Los closures deben tener tanto una forma completa como una forma abreviada
3. La sintaxis completa de closures debe ser funcionalmente completa: admitir lista de captura explícita, declaración de genéricos, lista completa de parámetros y tipo de retorno explícito
4. La forma abreviada debe ser lo bastante breve, permitiendo omitir todos los elementos sintácticos innecesarios, pero también debe parecerse a la forma completa

Ejemplo de la sintaxis completa de closures:
```
.{
    async [move x, weak y, ref z]<T>(arg: T) -> ReturnTy where T: Constrait
    =>
    statements(x, arg);
    return y;
}
```

La sintaxis completa de closures empieza por `.{` y termina en `}`, para encajar con la “llamada trailing”. El contenido interno incluye:
1. Lista de calificadores (qualifier), incluidos `async` `const`, etc.
2. Lista de captura explícita entre corchetes; los modos de captura admitidos son `move` `ref` `ref mut` `weak`
3. Lista de parámetros genéricos entre ángulos
4. Lista de parámetros de función entre paréntesis
5. Tipo de retorno explícito tras `->`
6. Condición where opcional
7. Cuerpo de la closure tras `=>`

Ejemplo de la forma más abreviada:
```
(x) => x + x
```
Dentro de los paréntesis está la lista de parámetros; los parámetros pueden omitir la declaración de tipo. Tras `=>` está el cuerpo de la closure; si no hay llaves, ese cuerpo solo puede ser una expresión y no puede contener sentencias.




### Expresiones de ramificación

if cond then expr else expr

### Expresiones yield/await



### Expresión is

La palabra clave `is` sustituye a `if-let` `while-let` `matches!`

### Expresión as


### Expresiones constantes


## Sentencias

Las sentencias suelen terminar en punto y coma. Se permiten sentencias vacías.

### Bloques de sentencias (block)

Las llaves contienen varias sentencias. Un bloque de sentencias introduce un scope nuevo; las variables locales del scope interno pueden tener el mismo nombre que las del scope externo.

Se permiten blocks vacíos.

### Bloques de sentencias unsafe

Un block modificado con `unsafe`.

### Sentencia let

```
let <pattern> = <expr> ;
```

> **TODO**: por ahora se exige que la declaración de variables se inicialice, para simplificar la implementación y evitar comprobar por CFG que la variable se inicializa antes de usarse. Más adelante se puede relajar el requisito y solo comprobar que se inicializa antes de leerla.

### Sentencia de expresión

```
<expr> ;
```

### Sentencia if

```rust
fn test(cond: Bool) {
    if cond {
        println("then branch");
    } else {
        println("else branch");
    }
}
```

### Sentencia match


### Sentencias de bucle

#### `while`

1. La condición tras while debe ser una expresión de tipo Bool
2. Tras las llaves de while no hace falta escribir punto y coma

#### `do-while`

1. La condición tras while debe ser una expresión de tipo Bool
2. do-while debe escribir punto y coma al final de la sentencia

#### `for`

`for  <pattern> in <expr>  { <statements> }`

1. for admite coincidencia de patrones
2. expr debe ser un tipo que cumpla el trait `Iterable` o `IterableMut`, o un puntero a ese tipo.

Hay tres formas de desazucarar la sentencia `for-in`, según cómo se escriba `<pattern>`:

1. pattern es identifier: `for i in v { <body> }`; en este caso el tipo de `i` es el tipo `Iterable::Item` de `v`. Se desazucara así:

    ```
    {
        let mut iter = v.iter();
        while (iter.next() is Some(p)) { // p es un borrow de solo lectura que apunta a Item
            let i = *p;                // i es un Item nuevo copiado desde el borrow de solo lectura
            <body>
        }
    }
    ```

2. pattern es ref identifier: `for ref i in v { <body> }`; en este caso el tipo de `i` es el tipo `&Iterable::Item` de `v`. Se desazucara así:

    ```
    {
        let mut iter = v.iter();
        while (iter.next() is Some(i)) { // i es un borrow de solo lectura que apunta a Item
            <body>
        }
    }
    ```

3. pattern es ref mut identifier: `for ref mut i in v { <body> }`; en este caso el tipo de `i` es el tipo `&mut Iterable::Item` de `v`. Se desazucara así:

    ```
    {
        let mut iter = v.iter_mut();
        while (iter.next() is Some(i)) { // i es un borrow de lectura y escritura que apunta a Item
            <body>
        }
    }
    ```

### Sentencias de salto

#### `return`

1. Tras return puede haber o no una expresión.
2. El tipo de la expresión tras return debe ser compatible con el tipo de retorno de la función.

#### `break`

1. break solo puede usarse en bloques de bucle. Incluye `while` `do-while` `for`.

#### `continue`

1. continue solo puede usarse en bloques de bucle. Incluye `while` `do-while` `for`.

#### `panic`

La palabra clave panic puede compararse con la palabra clave throw de lenguajes como C++/Java. Pero Zinc no tiene sintaxis try-catch ni ofrece la función `catch_unwind`.

Solo cuando ocurre un error irrecuperable debería usarse panic. Tras la palabra clave panic puede ir una expresión de tipo cadena. Ejemplo:

```rust
fn unwrap<T>(o: Option<T>) -> T {
    if o is Some(v) {
        return v;
    } else {
        panic "unwrap Option failed";
    }
}
```

#### `goto`

`goto` es una palabra clave reservada, aún no implementada. Se prepara para varios propósitos.

1. Combinada con sentencias de etiqueta (label), implementar la función de salto. Sustituye la sentencia break label de rust. Principalmente porque el escenario de uso de break label es limitado: solo puede usarse en bloques de bucle. Pero goto label tiene en C un escenario más habitual: escribir el manejo de errores de forma unificada al final de la función y, en el resto de la función, cuando se encuentra un error, hacer goto hasta ahí para procesarlo de forma unificada. Así se escribe de forma extremadamente cómoda y legible. Usar goto en este escenario es muy razonable. Por supuesto, para evitar que el programador abuse, deberíamos imponer algunas restricciones a la sentencia goto:
    1. goto es una sentencia; no puede usarse dentro de una expresión.
    2. No se puede saltar entre funciones ni lambdas.
    3. Solo se puede saltar hacia fuera de un block, no hacia dentro. El cuerpo de un bucle y las distintas ramas de un match también son blocks distintos.
    4. Solo se puede saltar hacia adelante, no hacia atrás.
    5. Hay que asegurar que, en la ruta del goto, todas las variables se inicializan antes de usarse.

    En resumen, quiero ofrecer una sentencia goto segura y fiable. Que pueda implementar el patrón habitual de C de manejo unificado de errores al final de la función, sin traer otros riesgos.

2. Preparada para la característica de “recursión de cola determinista”. Sustituye la palabra clave `become` de rust.
    1. Se permite usar `goto call_fn();` para implementar recursión de cola determinista.


## Funciones

La definición de una función empieza con la palabra clave `fn`, seguida del nombre de la función y la lista de parámetros, rodeada de paréntesis, luego el tipo de retorno opcional y, por último, el cuerpo de la función rodeado de llaves.

```rust
// definición de función
fn function_name(arg1: Int, arg2: Char) -> Bool {
    println("function body");
    return true;
}

fn main() {
    function_name(0, 'X'); // llamada a función
}
```

Omitir el tipo de retorno significa devolver el tipo `()`.

Los parámetros admiten parámetros con nombre; los parámetros con nombre empiezan por un punto. Al llamar, si se usan parámetros con nombre, también empiezan por un punto. Se usa la sintaxis de punto para ser coherente con la expresión de inicialización de estructuras.
Los parámetros con nombre deben ir después de los parámetros sin nombre.

```
fn run(.from: &Str, .to: &Str) {
    println("run from {} to {}", from, to);
}

fn main() {
    run("home", "bar");  // se puede llamar por el orden de los parámetros
    run(.to = "bar", .from = "home"); // también se puede llamar por nombre; en ese caso el orden no importa.
}
```

Como las funciones ya admiten parámetros con nombre, admitir además coincidencia de patrones en los parámetros se complica un poco. Además, en la posición de parámetros de función la mayoría de patrones no sirven de mucho, así que los parámetros de función no admiten coincidencia de patrones.
En Rust, el patrón más útil que admite la lista de parámetros es el patrón `mut`, que permite modificar el parámetro en sí. En zinc se recomienda evadir eso de esta forma:
```
fn f(v: Vec<Int>) {
    let mut v = move v; // el v de los parámetros no tiene el modificador mut; se puede definir dentro de la función otra variable mut con el mismo nombre y hacerle move del parámetro.
    // ...
    v.push(2); // se necesita un borrow de tipo &mut de v; aquí se exige que la variable v esté modificada con mut
    // ...
}
```

Los parámetros admiten valores por defecto; el valor por defecto debe ser una expresión constante. Los parámetros con valor por defecto deben ir después de los que no tienen valor por defecto.

```
fn increase(value: Int = 1) {}

fn main() {
    increase();  // el valor del argumento es 1
    increase(2); // el valor del argumento es 2
}
```

Se admiten closures trailing. El significado de la sintaxis de llamada con closure trailing es: si al definir la función el tipo del último parámetro es un tipo fn/Fn/FnMut, o un tipo puntero a ellos, al llamar se puede escribir la lambda fuera de la lista de parámetros.
Si además de ese parámetro de tipo función no hay otros parámetros, quien llama puede omitir los paréntesis de la lista de parámetros.
Ejemplo:

```
// el último parámetro de la función lock es un tipo función
fn lock(f: &Fn()) {
    println("lock");
    f();
    println("unlock");
}

fn main() {
    lock( ()=>println("smth") ); // OK, lambda de sintaxis simple como parámetro
    lock( .{ () => println("smth"); } ); // OK, lambda de sintaxis completa como parámetro

    // sintaxis de llamada con closure trailing: se escribe el closure fuera de la lista de parámetros
    lock().{
        () => println("smth");
    };
    // sintaxis de llamada con closure trailing: cuando no hay otros parámetros, se omiten los paréntesis de la lista de parámetros
    lock.{
        () => println("smth");
    };
    // más adelante se puede simplificar aún más la sintaxis de lambda y permitir omitir el símbolo =>
    lock.{
        println("smth");
    };
}
```

Se admite sobrecarga de funciones. Pero se exige que el número de parámetros sea distinto.

```
fn f1(arg1: Int, arg2: Int = 1) {} // admite 1 parámetro o 2 parámetros.
fn f1(arg: String) {} // error, no puede formar sobrecarga con el f1 de arriba. Esta versión admite 1 parámetro, y la de arriba también. Hay conflicto.
```

todo: por ahora no se admiten parámetros de longitud variable; pendiente de mejorar. Se recomienda que, en general, se use sobrecarga de funciones en su lugar; cuando hay demasiados parámetros, usar arrays y slices.

Se admite añadir funciones miembro a un tipo mediante un bloque `impl`.
Cuando el nombre del primer parámetro es la palabra clave `self`, se permite llamar a la función con la forma de punto.

```rust
impl User {
    fn send_email(&self) {}
}
fn main() {
    let u: User = { ... };
    u.send_email();
}
```

El parámetro `self` es especial y tiene varias escrituras simplificadas:
* `self` representa `self: Self`
* `&self` representa `self: &Self`
* `&mut self` representa `self: &mut Self`
* `*self` representa `self: *Self`
* `*weak self` representa `self: *weak Self`

Las funciones pueden usarse como ciudadanos de primera clase. Se admiten tipos de puntero a función.

todo: introducir en la firma de la función una sintaxis nueva que indique que se permite al compilador insertar automáticamente conversiones auto-ref/auto-deref del argumento real al parámetro formal. Sustituye el `AsRef trait / AsMut trait` de Rust
