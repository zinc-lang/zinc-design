# Conceptos básicos

## hello world

Un fragmento básico de código fuente de zinc se ve así:

```rust
fn main() {
    println("hello world");
}
```

Los archivos de código fuente terminan con la extensión `.zn` y su contenido debe estar codificado en utf8. El comando de compilación es:

```
zinc main.zn
```

Cuando termina, podemos ver en la carpeta actual un programa ejecutable recién generado. Al ejecutarlo, se imprime la cadena `hello world`.

## Comentarios

zinc admite dos tipos de comentarios: comentarios de línea y comentarios de bloque.

1. Comentarios de línea

    Los comentarios de línea empiezan con `//` y llegan hasta el final de la línea.

2. Comentarios de bloque

    Los comentarios de bloque empiezan con `/*` y terminan en el primer `*/`.

> **TODO**: la función de comentarios de documentación aún no está implementada.

## Funciones

Una definición ordinaria de función en zinc empieza con la palabra clave `fn`, seguida del nombre de la función y un par de paréntesis; dentro de los paréntesis está la lista de parámetros. Después de los paréntesis se puede escribir el tipo de retorno. A continuación va el cuerpo de la función, rodeado de llaves.

Ejemplo:

```rust
fn my_func(arg1: Int32, arg2: String) -> Bool {
    return true;
}
```

Si un proyecto zinc necesita generar un programa ejecutable, debe definir en el módulo raíz una única función `main` como punto de entrada del programa.

## Variables y tipos

Identificadores: empiezan por `_` o por una letra; después se permiten `_`, dígitos o letras.

Un `_` suelto es un identificador especial que significa “ignorar”.

> **TODO**: por ahora no se admiten identificadores unicode ni raw identifier; pendiente de implementar.

### Variables locales

Las variables locales no tienen que marcar el tipo de forma explícita; se admite inferencia de tipos.

Si el nombre de la variable no está modificado con mut, por defecto es inmutable.

```rust
fn main() {
    let x = 5_i;
    x = 6; // error, x es inmutable
}
```

Por legibilidad, en el mismo block las variables locales **no** pueden tener el mismo nombre. En blocks distintos, por supuesto, sí pueden tener el mismo nombre.

### Variables estáticas

La definición de una variable estática se modifica con la palabra clave `static`.

El tipo de una variable estática debe cumplir la restricción `Send` (fíjate en que qué tipos cumplen `Send` es distinto de Rust); consulta el capítulo “Paralelismo y concurrencia”. Si se quiere usar un tipo un-Send como variable static, hay que modificarla con unsafe.

Las variables estáticas deben inicializarse al definirlas, y la expresión de inicialización solo puede ser una “expresión constante”.

Las variables estáticas pueden llevar el modificador mut. Pero una static mut debe modificarse con unsafe. Su lectura y escritura deben hacerse en un contexto unsafe.

```rust
unsafe static mut G: Int32 = 1; // una variable static con mut, o una static cuyo tipo no cumple Send, debe modificarse con unsafe

fn main() {
    println(G.to_string()); // error, G solo puede usarse en un contexto unsafe
}
```

> **Funcionalidad prevista**: más adelante se podrá admitir inferencia de tipos para variables static.

### Constantes

La definición de una constante se modifica con la palabra clave `const`.

De forma similar a las variables estáticas, el tipo de una constante debe cumplir la restricción `Send`. Las constantes deben inicializarse al definirlas, y la expresión de inicialización solo puede ser una “expresión constante”.

Las constantes no pueden llevar el modificador mut.

```
const PI: F32 = 3.14;
```
> **Funcionalidad prevista**: más adelante se podrá admitir inferencia de tipos para constantes const.
