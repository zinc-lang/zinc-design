# Sistema de módulos

## Componentes (component)

La unidad de compilación de zinc es el componente. Corresponde al crate de rust.

> In C and C++ programming language terminology, a translation unit (or more casually a compilation unit) is the ultimate input to a C or C++ compiler from which an object file is generated.

Unidad de compilación se refiere a los archivos de código fuente que necesita una ejecución del proceso del compilador. Esos archivos de código fuente son la unidad de entrada mínima de una ejecución del compilador y no se pueden seguir dividiendo. Si se siguen dividiendo, ya no se puede ejecutar una compilación.

Para un compilador C/C++, la unidad de compilación es un archivo fuente .c o .cpp, más todos los archivos que incluye de forma directa e indirecta con `#include`.

Para el compilador de Rust, la unidad de compilación es un crate. El usuario no puede compilar por separado varios mod internos de un crate.

Para el compilador de Zinc, la unidad de compilación es un componente (component). La entrada de cada ejecución del compilador es el código fuente de un componente completo. El usuario no puede compilar por separado varios archivos fuente de un componente y luego ensamblarlos en un componente; el compilador no tiene esa capacidad. Un componente debe compilarse como un todo.

Entre componentes no puede haber dependencia circular.

Dentro de un componente hay módulos (mod); entre los módulos de un componente sí puede haber dependencia circular. Porque todos los módulos del mismo componente se compilan juntos en un solo proceso del compilador.

Tras la compilación, un componente genera un archivo `.o` que contiene código máquina y un archivo `.zno` con la interfaz pública del componente. El nombre zno proviene de la sustancia “óxido de zinc”.

El archivo `.zno` puede entenderse como el “archivo de cabecera” del lenguaje C, solo que el archivo de cabecera lo escribe una persona y el archivo `.zno` lo genera automáticamente el compilador.

El contenido del archivo `.zno` incluye todas las firmas de funciones públicas, definiciones de tipos, declaraciones de variables globales, declaraciones impl, etc. También incluye el cuerpo de todas las funciones que se pueden inlinear entre componentes.

## `export`

Si un componente es una biblioteca, el usuario debe usar `export component_name;` para declarar el nombre de ese componente.
Si un componente es un programa ejecutable, el usuario debe definir en el root mod una función `fn main()` como punto de entrada del programa.
Si un componente no tiene ninguna de las dos, el compilador dará error.

En Rust el nombre del crate lo indica la opción de compilación `--crate-name NAME`; me parece que eso no es razonable: el nombre del componente debería indicarlo su código fuente.
Al fin y al cabo, el nombre del componente afecta al mangle name de los símbolos en el binario; si el nombre del componente lo controla una opción de compilación, es difícil garantizar la consistencia del resultado de compilación en distintos entornos de compilación.

## `import`

Cuando queremos describir que un componente a depende de otro componente b, hay que escribir en el código fuente de a `import b;`. Equivale básicamente a `extern crate b;` de Rust.

Si se hace una analogía con el lenguaje C/C++, se puede entender de forma simple como `#include "b.h"`.

## Introducción a mod

El módulo (mod) es la unidad básica de organización de código en Zinc. Un módulo puede contener funciones, tipos, constantes y otros módulos.

### Definición de módulos

Se usa la palabra clave `mod` para definir un módulo:

```rust
// definir un módulo en un archivo
mod my_module {
    fn helper() -> Int {
        return 42;
    }
    
    pub fn public_function() {
        println(f"helper returned $(helper())");
    }
}
```

### Módulos del sistema de archivos

Los módulos también se pueden organizar mediante el sistema de archivos. Si un módulo no tiene contenido, el compilador buscará un archivo o directorio del mismo nombre:

```rust
// en main.zn
mod network;

// el compilador buscará:
// 1. el archivo network.zn
// 2. el archivo network/mod.zn
```

### Visibilidad

Los ítems de un módulo son privados por defecto y solo se pueden acceder dentro del módulo que los define. Usar la palabra clave `pub` hace que un ítem sea visible hacia fuera:

```rust
mod my_module {
    pub fn public_function() {
        // se puede acceder desde fuera
    }
    
    fn private_function() {
        // solo se puede acceder dentro de my_module
    }
    
    pub struct PublicStruct {
        pub field: Int,      // miembro público
        private_field: Int,  // miembro privado
    }
}
```

### Módulos anidados

Los módulos se pueden definir de forma anidada, formando una estructura de árbol:

```rust
mod outer {
    pub fn outer_function() {}
    
    mod inner {
        pub fn inner_function() {}
        
        mod deeply_nested {
            pub fn deep_function() {}
        }
    }
}
```

### Rutas de módulos

Se usa el operador `::` para acceder a los ítems de un módulo:

```rust
fn main() {
    outer::outer_function();
    outer::inner::inner_function();
    outer::inner::deeply_nested::deep_function();
}
```

### Modificadores de visibilidad

Zinc admite los siguientes modificadores de visibilidad:

1. **Por defecto (privado)**: solo se puede acceder dentro del módulo que lo define
2. **`pub`**: visible para todos los módulos
3. **`pub(in path)`**: solo visible dentro del módulo de la ruta indicada (> **Funcionalidad prevista**: se admitirá en el futuro)

```rust
mod my_module {
    pub fn public_api() {}
    
    fn internal_helper() {}
    
    // se admitirá en el futuro: solo visible en el módulo padre
    // pub(super) fn parent_visible() {}
}
```

### Reexportación

Usar `pub use` permite reexportar ítems de otros módulos:

```rust
mod inner {
    pub fn helper() {}
}

mod outer {
    // reexportar inner::helper
    pub use inner::helper;
}

fn main() {
    // se puede acceder mediante outer::helper
    outer::helper();
}
```

### Relación entre módulos y componentes

> **Funcionalidad prevista**: se considera admitir las funciones export path e import path. Es decir, no solo permitir las escrituras `export ident;` `import ident;`, sino también `export ident1::ident2::ident3;` e `import ident1::ident2::ident3;`.

Principalmente porque, si entre componentes solo hay una relación de yuxtaposición y no de contención, no es lo bastante flexible.
La estructura lógica interna de un componente es un árbol; si entre componentes no hay relación de contención, solo de yuxtaposición, entonces cuando un componente es demasiado grande y el tiempo de compilación demasiado largo y hay que refactorizar, la propia refactorización necesariamente cambia la estructura lógica del componente. Eso no es adecuado.

Se puede considerar permitir que un subárbol interno de un componente también sea un componente, y no solo un módulo, lo que ayuda a dividir un componente grande en distintas unidades de compilación sin afectar a la estructura lógica interna del componente.

Hay que mantener invariables dos principios:
1. La estructura lógica de un componente es un árbol, no un bosque
2. Entre componentes no puede haber dependencia circular

Por ejemplo, en la figura siguiente, el componente c contiene una serie de submódulos y es un componente de gran escala.

<img src="./mod_tree.png" width="50%" align=center />

Para acelerar la compilación, podemos extraer algunos de sus submódulos de fuerte cohesión como componentes independientes; en la figura se representan con distintos colores.
El módulo e y todos sus módulos subordinados son un componente; el módulo g y todos sus módulos subordinados también son un componente; el módulo i y sus subordinados también son un componente. Los módulos del mismo color en la figura se compilan juntos; basta con garantizar que entre componentes distintos no hay dependencia circular. Al mismo tiempo, se mantiene invariable la estructura lógica de todo el componente; el módulo raíz del componente g tiene un nombre como `c::d::g`, y el mangle name de todos sus ítems internos también mantiene esa nomenclatura.


## Sentencia use

La sentencia `use` se usa para introducir en el ámbito actual ítems de otros módulos, evitando escribir cada vez la ruta completa.

### Uso básico

```rust
use std::collections::HashMap;

fn main() {
    let map = HashMap::new();
    // se puede usar HashMap de forma directa, sin escribir la ruta completa
}
```

### Palabras clave de ruta

- `self`: se puede acceder al mod actual
- `super`: se puede acceder al mod de nivel superior del mod actual
- `::ident`: se puede acceder a un nombre en el ámbito global, es decir, un nombre de component. Los nombres legales incluyen el nombre propio definido en export, los nombres de dependencias importadas y la biblioteca estándar std.
- `::self`: se puede acceder al mod de nivel superior del component actual. Equivale a acceder a los nombres de nivel superior de este componente empezando por `::ident`. Nota: es distinto de Rust. En Rust se accede al mod de nivel superior mediante la palabra clave `crate::`.

### Ejemplo

```rust
mod outer {
    pub fn outer_fn() {}
    
    mod inner {
        pub fn inner_fn() {}
        
        fn example() {
            // usar self para acceder al módulo actual
            self::inner_fn();
            
            // usar super para acceder al módulo padre
            super::outer_fn();
        }
    }
}

// importar con use
use outer::inner::inner_fn;

fn main() {
    inner_fn();
}
```

### Importación con cambio de nombre

Usar la palabra clave `as` permite cambiar el nombre de un ítem importado:

```rust
use std::collections::HashMap as Map;

fn main() {
    let map = Map::new();
}
```

### Importar varios ítems

```rust
use std::collections::{HashMap, HashSet};

fn main() {
    let map = HashMap::new();
    let set = HashSet::new();
}
```
