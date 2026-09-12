# unsafe e interoperabilidad

## Palabra clave unsafe

Zinc marca con la palabra clave `unsafe` el código que puede producir comportamiento indefinido. Usar `unsafe` permite decirle al compilador: "sé que este código puede ser inseguro, pero garantizo que es correcto."

### Escenarios de uso de unsafe

Las siguientes operaciones deben realizarse en un contexto unsafe:

1. **Llamadas a funciones y llamadas a métodos**
   - Funciones modificadas con `extern`
   - Funciones modificadas con `unsafe`

2. **Desreferencia de punteros crudos**
   - Desreferenciar punteros de tipo `*raw T` o `*raw mut T`

3. **Conversiones de tipo**
   - Convertir un puntero crudo a cualquier tipo
   - Conversiones entre enteros y punteros

4. **Acceso a miembros de union**
   - Leer y escribir miembros de un tipo union

5. **Acceso a mut static**
   - Leer y escribir variables globales definidas con `static mut`

### Bloques unsafe

Usar un bloque `unsafe` permite crear un contexto unsafe:

```rust
fn main() {
    let x: Int = 42;
    let p: *raw Int = &x as *raw Int;
    
    unsafe {
        let value = *p; // desreferenciar un puntero crudo requiere unsafe
        println(f"value = $(value)");
    }
}
```

### Funciones unsafe

Una función definida con `unsafe fn` debe llamarse en un contexto unsafe:

```rust
unsafe fn dangerous_operation() {
    // código que puede producir comportamiento indefinido
}

fn main() {
    unsafe {
        dangerous_operation(); // llamada a una función unsafe
    }
}
```

### unsafe impl

Usar `unsafe impl` permite implementar a mano ciertos marker trait, como `Send` y `Sync`:

```rust
struct MyType {
    ptr: *raw mut Int,
}

// se declara a mano que MyType es Send
// el programador debe garantizar que esta declaración es correcta
unsafe impl Send for MyType {}
```

## FFI (interfaz de funciones externas)

### Funciones extern

Usar la palabra clave `extern` permite declarar funciones externas, habitualmente para llamar a bibliotecas del lenguaje C:

```rust
// se declara la función printf de la biblioteca estándar de C
extern fn printf(fmt: *const UInt8, ...) -> Int32;

fn main() {
    unsafe {
        printf("Hello from C!\n" as *const UInt8);
    }
}
```

### Bloques extern

Usar un bloque `extern` permite declarar funciones externas en lote:

```rust
extern "C" {
    fn malloc(size: UInt) -> *raw mut UInt8;
    fn free(ptr: *raw mut UInt8);
    fn strlen(s: *const UInt8) -> UInt;
}

fn main() {
    unsafe {
        let ptr = malloc(100);
        // usar ptr...
        free(ptr);
    }
}
```

### Interoperabilidad con el lenguaje C

Zinc ofrece varias formas de interoperar con el lenguaje C:

1. **Llamar a funciones C**: usar `extern` para declarar funciones C y luego llamarlas en un bloque unsafe
2. **Pasar datos a C**: usar tipos de puntero crudo para pasar datos
3. **Recibir datos de C**: usar punteros crudos para recibir los datos que devuelve C

Ejemplo:

```rust
// se declaran funciones C
extern fn fopen(filename: *const UInt8, mode: *const UInt8) -> *raw mut UInt8;
extern fn fclose(file: *raw mut UInt8) -> Int32;

fn main() {
    unsafe {
        let file = fopen("test.txt\0" as *const UInt8, "r\0" as *const UInt8);
        if file != null {
            // usar el archivo...
            fclose(file);
        }
    }
}
```

## Frontera entre lo seguro y lo inseguro
