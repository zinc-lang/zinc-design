
# Paralelismo y concurrencia

## Thread-safety

Rust tiene una ventaja inigualable: garantiza de forma natural la thread-safety del programa. Entre los lenguajes de programación habituales, es único. Zinc también espera conservar esta característica de thread-safety.

### Trait Send/Sync

Unificamos con una marca todos los tipos que se pueden pasar de forma segura entre hilos: se llama `Send trait`.

Primero recordemos el significado de [semántica de valor y semántica de referencia](https://isocpp.org/wiki/faq/value-vs-ref-semantics#val-vs-ref-semantics).

Se puede ver que los tipos de semántica de valor cumplen de forma natural el requisito del `Send trait`. Ejemplos:
1. Los tipos básicos Bool, Char, Int, etc. cumplen semántica de valor, por tanto cumplen con seguridad el `Send trait`.
2. Los tipos compuestos, como tuple/struct/enum, etc., si todos los miembros cumplen `Send`, entonces también son necesariamente de semántica de valor, así que esos tipos también cumplen el `Send trait`.
3. Los tipos puntero no cumplen semántica de valor; en general `*T` no cumple el trait `Send`. Pero también hay que analizar según el caso concreto. Un tipo puntero, o un tipo que contiene internamente un miembro puntero, también cumple `Send` en estos dos casos:
    1. Mediante el mecanismo de copy-on-write implementan semántica de valor; esos también cumplen el `Send trait`. Por ejemplo el tipo `String`: aunque internamente contiene un miembro puntero, su implementación garantiza “semántica de valor” mediante copy-on-write, así que ese tipo también cumple el `Send trait`. Para un tipo como `Vec<T>`, aunque también implementa copy-on-write, como lleva genéricos, si cumple `Send` tiene una condición previa. Si y solo si `T: Send`, entonces `Vec<T> : Send`.
    2. Mediante un mecanismo de sincronización de hilos se garantiza que el contenido al que apunta el puntero es thread-safe; esos también cumplen el `Send trait`. Por ejemplo el tipo `*AtomicInt32`: aunque puede ocurrir que punteros distintos en dos hilos apunten al mismo `AtomicInt32`, como `AtomicInt32` tiene sincronización de hilos, el tipo `*AtomicInt32` cumple el `Send trait`.

Por tanto, introducimos además otro `Sync trait`, que representa que un tipo tiene en sí capacidad de sincronización de hilos. Entonces podemos decir que, para el tipo `*T`, si y solo si `T` cumple `Sync`, `*T` cumple `Send`.
Para los punteros borrow, igual. Si y solo si `T: Sync`, entonces `&T : Send` y `&mut T : Send`.

Ejemplos de tipos que cumplen el `Sync trait`:
1. Los tipos de la serie `Atomic`
2. Tipos como `Mutex<T>` `RwLock<T>`. Nota: como estos tipos llevan genéricos, si cumplen `Sync` tiene una condición previa. Si y solo si `T: Send`, entonces `Mutex<T> : Sync`.

Con estos dos trait, aún hay que hacer dos cosas para garantizar la thread-safety:
1. En la biblioteca estándar, marcar todos los tipos básicos y los tipos definidos en la biblioteca estándar con `Send` y `Sync`.
2. En las API que necesitan pasar datos entre hilos distintos, añadir una restricción: el tipo que se pasa entre hilos debe cumplir la condición `Send`.

Los tipos definidos por el usuario, en general, no necesitan marcarse a mano con `Send` `Sync`; basta con el resultado que infiere el compilador.
Si la implementación de ese tipo usa unsafe, entonces el programador debe analizar él mismo si el tipo cumple `Send` `Sync` e indicarlo de forma explícita con unsafe impl.

### Sin data race

Cuando el compilador y la biblioteca estándar tienen preparados los tipos `Send` `Sync` anteriores, podemos garantizar mediante comprobación de compilación que en el código de negocio no hay “data race”.
Si aparece la posibilidad de un data race, el compilador dará error automáticamente.

Ejemplo:

```
// supongamos que lo que hace la función send es enviar data a otro hilo para usarlo; entonces necesita restringir que el tipo de data cumpla la restricción Send
fn send<T>(data: T) where T: Send { }

fn main() {
    send(1_i);
    send(String::new());
    send((type Vec<Int>)::new());
    send(box 1_u); // Error. Error de compilación: *UInt no cumple la restricción Send
}
```

### Variables globales

Las variables globales, de forma natural, pueden ser accedidas por hilos distintos, así que también hay que restringirlas para garantizar la thread-safety.
1. La definición de una variable global puede modificarse con las palabras clave `static` o `unsafe static`. Una variable global modificada con `unsafe` solo puede leerse y escribirse en una zona unsafe. Solo las variables globales de solo lectura definidas con `static` son seguras.
2. El tipo de una variable global no modificada con `unsafe` debe cumplir el requisito del `Send trait`.

**Nota**: a diferencia de las reglas de Rust, Zinc exige que las variables globales cumplan la restricción `Send`, no la restricción `Sync`.

Porque el `Sync trait` de Zinc y el `Sync trait` de Rust tienen semántica distinta. Muchos tipos que en Rust cumplen `Sync` en Zinc no cumplen `Sync`.
Tomemos como ejemplo el tipo básico `Int`: en Rust cumple `Sync`; pero en Zinc, `Int` no puede cumplir `Sync`.
Por reducción al absurdo: si estipuláramos que `Int` cumple `Sync`, eso significaría que un puntero como `*Int` cumple `Send`, entonces se podría pasar entre hilos, y veríamos que dos hilos obtienen un puntero al mismo `Int` sin sincronización de hilos, lo cual es incorrecto.

* El `Sync` de Zinc representa un tipo que internamente tiene un mecanismo extra de sincronización de hilos. Los tipos que cumplen la restricción `Sync` son muchos menos que en Rust. 
* El `Send` de Zinc representa un tipo que se puede pasar de forma segura entre hilos. Incluye principalmente cuatro casos:

    1. El tipo clásico de semántica de valor, que internamente no contiene punteros; al pasarse entre hilos se copia un duplicado, por tanto es con seguridad seguro
    2. El tipo contiene punteros y los datos a los que apuntan son de solo lectura. Los datos compartidos son de solo lectura, por tanto es seguro.
       Por ejemplo el tipo `Str`: su API externa no ofrece ninguna capacidad de modificación, lo que garantiza que `Str` cumple `Send`.
    3. El tipo contiene punteros y los datos a los que apuntan cumplen copy-on-write. Si un hilo intenta modificar los datos compartidos, copia esos datos y luego los modifica; mientras los datos estén en estado compartido, con seguridad no se modificarán; lo que se modifica es siempre el duplicado que se copió, por tanto es seguro.
       Por ejemplo el tipo `Vec<Int>`: todas las funciones miembro con capacidad de modificación juzgan el conteo de referencias; si no es exclusivo, primero copian los datos y luego modifican
    4. El tipo contiene punteros y los datos a los que apuntan se pueden modificar bajo la premisa de sincronización de hilos. Tras pasarse entre hilos, varios hilos tendrán punteros a los mismos datos, pero todas las lecturas y escrituras de los datos compartidos tienen sincronización de hilos, por tanto es seguro
       Por ejemplo los tipos `*AtomicInt` `*Mutex<String>`


Aunque algunos detalles del `Send` y `Sync` diseñados por Zinc y Rust son distintos, ambos logran garantizar la thread-safety.


<div style="border: 1px solid black; padding: 10px; background: #F0F0F0">

Del diseño anterior se puede ver que tipos como `Vec<Int>` cumplen la restricción `Send`. Podemos pasar de forma segura ese tipo entre hilos.

1. Si pasamos `Vec<Int>` a hilos distintos y cada hilo solo lo lee, solo ocurrirá una copia superficial y la eficiencia de ejecución será muy alta.
2. Si pasamos `Vec<Int>` a hilos distintos y algún hilo lo modifica, al modificarse en ese hilo se dispara una copia profunda; entonces hay un overhead de rendimiento mayor, pero no hay problema de thread-safety.
3. Si queremos pasar `Vec<Int>` entre hilos, poder modificarlo y que no ocurra una copia profunda, el programador debe garantizar él mismo que no existan a la vez varias instancias de copia superficial. Esto se puede lograr con una expresión move.
Transferir el ownership de un `Vec<Int>` entre hilos distintos permite modificarlo de forma eficiente entre hilos. Pero el compilador no rastrea el ownership a nivel de tipos, solo a nivel de flujo de control, para reducir el impacto en el usuario.
4. Si queremos obtener un `Vec<Int>` de semántica de referencia, basta con hacerle box; el `*Vec<Int>` obtenido es un tipo de semántica de referencia. Distintos punteros pueden modificar el mismo `Vec<Int>` y compartir el mismo bloque de datos.
Como `Vec<Int>` no cumple `Sync`, `*Vec<Int>` no cumple `Send`. Entonces el compilador puede ayudarnos a comprobar que todos esos punteros están dentro del mismo hilo; un tipo non-`Send` no puede cruzar el límite de un hilo.
Así que el tipo `*Vec<Int>` de semántica de referencia tampoco tiene problema de thread-safety. Además, las operaciones de conteo de referencias de este puntero Arc se pueden optimizar a operaciones de incremento y decremento non-atomic.

De lo anterior se ve que la optimización de copy-on-write es muy importante para un tipo como `Vec<T>`.

Supongamos que el lenguaje Zinc no eligiera ARC como mecanismo base de gestión de memoria, sino GC, y que la expresión `box` devolviera un puntero GC trace-able.
Entonces un tipo contenedor como `Vec<T>`, cuya implementación interna solo podría usar punteros GC, no podría ser un tipo de “semántica de valor”. Solo podría ser non-Send. Y la expresión move tampoco tendría sentido.
Así, en un escenario multithread, los tipos que podrían cumplir la restricción `Send` serían poquísimos; para garantizar la thread-safety, los escenarios en los que habría que hacer copia profunda serían más, y eso traería un enorme coste extra de rendimiento.
Por tanto, la gestión de memoria ARC y el objetivo de thread-safety de Zinc son los que mejor encajan.

Y precisamente porque Zinc también logra thread-safety, podemos garantizar en tiempo de compilación que ciertos punteros ARC con seguridad no pueden cruzar el límite de un hilo; ellos y sus copias solo pueden usarse dentro del mismo hilo.
Usando esa información, el compilador puede optimizar sus operaciones de incremento y decremento del conteo de referencias a operaciones no atómicas, y mejorar aún más el rendimiento. Esta optimización solo depende del tipo, no del flujo de control.

Por tanto, la thread-safety y la gestión de memoria ARC se complementan a la perfección.

</div>

## Hilos

La API de hilos que ofrece la biblioteca estándar de Zinc es un encapsulado simple de la funcionalidad de hilos del sistema operativo. Para crear un hilo se usa esta función:

```
// mod std::thread
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: Send + 'static + Fn()->T,
    T: Send + 'static,
```

`JoinHandle` puede llamar a la función `join()` para esperar a que termine el hilo. También puede llamar a la función `thread()` para obtener una variable de tipo `&Thread`.

Ejemplo:

```Rust
use std::thread::spawn;
fn main() {
    let h = spawn.{
        println("thread 2");
    };
    println("thread 1");
    h.join().get_or_panic();
}
```

### ThreadLocal

El tipo `ThreadLocal` se usa para expresar una variable “local al hilo”. Cada hilo mantiene de forma independiente su propio duplicado.

La forma de uso es la siguiente:

```
use std::thread::spawn;

fn main() {
    static G: ThreadLocal<Int64> = ThreadLocal::new(0);

    let h = spawn.{
        *G += 1;
    };
    h.join().get_or_panic();
    println(*G);
}
```

## Locks

Los locks incluyen `Mutex<T>` `RwLock<T>`.

El diseño de API de locks de Rust tiene una muy buena ventaja: une el lock y los datos protegidos en un solo tipo, evitando así que el lock y los datos protegidos no coincidan, y también que se acceda a los datos olvidándose de tomar el lock.

Pero tiene algunos problemas menores:
1. El diseño de poison
2. Es posible que en código safe se permita la fuga de MutexGuard, apareciendo un escenario en el que no se puede desbloquear
3. En código asíncrono, si ocurre un await al estar bloqueado, es fácil disparar un deadlock

Zinc hizo una pequeña modificación a esta API:
1. Se cancela la gestión del estado poison, porque Zinc no tiene mecanismo de catch unwind
2. Bloquear y desbloquear pasan a ser de estilo callback

```
fn synchronized(p: &Mutex<String>) {
    // más adelante se puede aprender de swift y añadir más azúcar sintáctico, omitiendo también esta lista de parámetros de la lambda
    p.lock.{ (data: &mut String) =>  
        data.push_str("tail");
    };
}
```

Las ventajas de hacerlo así son:
1. El alcance del lock es claro, rodeado de llaves, bastante visible, adecuado para la lectura humana
2. No hay MutexGuard, así que no hay riesgo de fuga; una acción de bloquear corresponde necesariamente a una de desbloquear, evitando el riesgo de que el lock no se pueda liberar
3. Al bloquear en varias capas, se puede asegurar que bloquear y desbloquear cumplen necesariamente el principio de último en entrar, primero en salir. Rust, al exponer MutexGuard, permite al usuario hacer drop de MutexGuard a mano, así que el orden real de desbloqueo puede no ser estructurado
4. En la función callback no se puede usar una expresión await, porque el tipo de API de la función callback no coincide, evitando así que ocurra un await mientras se tiene el lock

Por supuesto, trae una desventaja: ya no se pueden usar de forma natural, durante el lock, las sentencias de flujo de control `break` `continue` `return`, con el compilador desbloqueando automáticamente. Porque las sentencias de flujo de control dentro de la función callback solo afectan al interior de la lambda; si hay que tratar el flujo de control externo, hay que hacer juicios extra fuera de la lambda según los distintos valores de retorno de esa lambda. Dicho esto, al menos desde el punto de vista de la legibilidad, se escribe de forma muy clara. No poder usar de forma directa sentencias de flujo de control no tiene por qué ser algo malo.

### Diseño anti-copia

El tipo de lock de Rust está diseñado con semántica move, pero Zinc no tiene tipos con semántica move. Eso trae un riesgo nuevo: el usuario puede copiar por descuido una variable `Mutex<T>`, y eso puede producir bugs con facilidad.

Ejemplo:
```
// ejemplo de pseudocódigo; este código en realidad no puede compilar
fn test(s: &Mutex<String>) {
    let str_copy = *s; // en algunos casos esta copia puede ser muy oculta, y eso puede producir un bug
    str_copy.lock.{ (s: &mut String) => println(s) };
}
```

En Zinc, para evitar que ocurra esto, se introduce el `UnSized` trait. Este trait es un marker trait, sin funciones miembro. Al mismo tiempo se estipula:
1. Al definir un tipo, se permite `impl UnSized for MyType {}` para un tipo personalizado
2. Ningún tipo `UnSized` puede usarse de forma directa como variable global, constante, variable local o miembro. Ese tipo solo puede accederse de forma indirecta mediante un puntero.
3. No se permite desreferenciar un tipo puntero que apunta a `UnSized`.

Si en la biblioteca estándar estipulamos que un tipo como `Mutex` es `UnSized`, se puede evitar el bug anterior.

```
struct S {
    value: Mutex<MyType>, // Error, Mutex no puede usarse de forma directa como miembro; se recomienda usar *Mutex<MyType> en su lugar
}

fn f() { 
    let local = Mutex::new(MyType::new()); // Error, Mutex no puede usarse de forma directa como variable local; se recomienda usar *Mutex<MyType> en su lugar
}

static GLOBAL_DATA: Mutex<MyType> = Mutex::new(MyType::new()); // Error, Mutex no puede usarse de forma directa como variable global; se recomienda usar &'static Mutex<MyType> en su lugar

static GLOBAL_DATA: &'static Mutex<MyType> = &Mutex::new(MyType::new()); // OK
```

Funciones de la biblioteca estándar como `swap` tampoco pueden operar sobre tipos `UnSized`. Por tanto, este diseño puede evitar el error anterior de "copiar por descuido todo el Mutex".

Tipos de la biblioteca estándar como `File` tienen un diseño similar. El usuario solo puede usar siempre un “tipo puntero que apunta a File”, y no puede usar de forma directa el tipo `File` como tipo valor.

> **TODO**: el diseño de UnSized type aún necesita perfeccionarse. Hay que permitir que UnSized type se use como parámetro de función y como retorno de función. Ciertas funciones genéricas también deben permitir argumentos genéricos UnSized.
> Pero hay que restringir que todos los UnSized type solo puedan usarse como variables temporales; antes de vincularse a una variable de tipo sized, solo pueden hacerse move, no copiarse.

```
impl<T> UnSized for Mutex<T> {}
impl<T> Mutex<T> where T: ?Sized { // hay que permitir que T pueda ser unsized
    fn new(v: T) -> Mutex<T> { // hay que permitir unsized type como parámetro y retorno de función
        return {
            .val = move v, // un parámetro unsized solo puede hacerse move
            ...
        };
    }
}
```

## I/O asíncrono

> **TODO**: la función de I/O asíncrono aún no está implementada.

Idea básica: la facilidad de uso por encima de las consideraciones de rendimiento. Se rechaza de forma decidida el diseño Move/Pin de Rust y se alinea la facilidad de uso con C#. El diseño Move/Pin, en la práctica, introduce una complejidad que supera con creces el beneficio; no merece la pena.
Salvo el problema de referencias circulares de Arc, la facilidad de uso de Zinc debería ser similar a la de C#.
