

## Formato de archivo zno

Cuando el compilador compila un componente de biblioteca, además de generar el archivo binario `.o` correspondiente, también necesita generar un “archivo de cabecera” similar al del lenguaje C, que contiene las firmas de todas las interfaces públicas, para que lo usen las bibliotecas descendentes.
El “archivo de cabecera” que genera el lenguaje zinc usa `.zno` como extensión de archivo, con el sentido de “óxido de zinc”.

El contenido del archivo `.zno` incluye todas las firmas de funciones públicas, definiciones de tipos, declaraciones de variables globales, declaraciones impl, etc. También incluye el cuerpo de todas las funciones que se pueden inlinear entre componentes.

El formato del archivo `.zno` puede ser texto plano o cualquier formato binario personalizado; en principio no hay problema. Pero para acelerar la compilación, al final se eligió implementarlo con un archivo sqlite. A continuación se explican los motivos.

Consideremos este escenario: un proyecto grande contiene muchos componentes muy complejos. El componente a depende del componente b, y b a su vez depende de otros componentes c, d, e, etc.
Si se usa un formato de archivo de texto simple, al compilar el componente a no solo hay que leer por completo el archivo zno del componente b, sino también leer por completo los archivos zno de todos los demás componentes de los que depende b, y luego hacer análisis sintáctico y semántico.
La función `#include` de los compiladores C/C++ se implementa así.
Pero para el sistema de módulos de Zinc no es adecuado copiar de forma directa el diseño de C/C++. Porque C/C++ toma el archivo como unidad de compilación, y Zinc toma el component como unidad de compilación; el volumen de la unidad de compilación es mayor.
Haciendo una analogía, equivale a fusionar todos los archivos de cabecera usados de forma directa o indirecta por un módulo C/C++ relativamente grande en un archivo enorme, y que los demás archivos hagan include de ese archivo de cabecera enorme en cualquier momento, en lugar de dividirlo a mano en varios archivos de cabecera pequeños.
Además hay que considerar la necesidad de inlining entre componentes; el cuerpo de algunas funciones también hay que registrarlo en este archivo de cabecera.
Para un proyecto grande, un archivo de cabecera demasiado grande es un gran desperdicio de tiempo de compilación, porque la mayor parte del contenido de ese montón de zno no se usa al compilar el componente a.
Por tanto, ¿podemos considerar que zno admita lectura bajo demanda, y al compilar a leer solo lo que se necesita? Cortar el archivo zno a la granularidad de cada “declaración”, en lugar de leerlo entero a la granularidad de archivo.
Por ejemplo, si el componente a solo usa de forma simple una función `fn f();` del componente b, entonces sé que las demás declaraciones de funciones, definiciones de tipos, etc. de b no tienen relación con el componente a, y las dependencias de b tampoco tienen relación con a.
Puedo extraer por separado esa firma de función del archivo `b.zno`, hacer análisis sintáctico y semántico, y no ocuparme del resto. Por muy enorme que sea el archivo zno de b, por muy complejo que sea su árbol de dependencias, eso no tiene nada que ver con compilar a.

Para alcanzar el objetivo anterior, necesito cortar todo el archivo de cabecera en varias partes según las distintas declaraciones y construir un índice que facilite las consultas, y eso es precisamente lo que sqlite puede ayudar a hacer bien.

Las ventajas de usar el formato sqlite son:
1. sqlite es el formato que más fácilmente admite la “lectura bajo demanda”; si diseñáramos nuestro propio formato de archivo para alcanzar el mismo efecto, al final probablemente habríamos hecho un archivo de formato sqlite más simplificado, más difícil de usar y con más bugs.
2. La velocidad de lectura y escritura es alta y el tamaño del archivo es adecuado. Hay herramientas auxiliares ya hechas, maduras y estables, que facilitan la depuración. Más adelante también es fácil modificar el schema y ampliar la funcionalidad.
3. En la fase actual, para simplificar la implementación, se guarda de forma directa en sqlite el código fuente serializado; más adelante también se puede cambiar a guardar el IR de código intermedio, y mejorar aún más la velocidad de compilación. Y eso no requiere modificar el diseño general.

El archivo zno no necesita preocuparse de la legibilidad humana, porque este archivo es para el compilador, no para que lo lea una persona.
La función de lectura humana debería estar en la herramienta de generación de documentación; deberíamos ofrecer una herramienta doc generator que genere documentación en formato HTML atractiva, con una composición elegante, resaltado de código, hipervínculos, función de búsqueda, etc., para facilitar la lectura humana.

## Bootstrapping (autoalojamiento)

La primera versión del compilador de zinc se implementó en C++; más tarde se reimplementó el frontend de nuevo en zinc (por supuesto, sin incluir el framework LLVM usado en el backend). Esto se hizo con varios propósitos:

1. En el mundo de la programación hay una frase célebre: “Eating your own dog food”. El diseñador primero debe amar el producto que ha diseñado y usarlo de forma intensiva a largo plazo; así se puede formar de forma efectiva el ciclo Plan-Do-Check-Act y ayuda a descubrir problemas cuanto antes y mejorarlos. Deseo mucho que el lenguaje de programación de mi trabajo diario sea zinc; usarlo de forma directa para escribir su propio compilador permite acumular lo antes posible experiencia de uso de primera mano y descubrir y mejorar problemas con rapidez. Que la primera versión no se implementara en Rust también está relacionado con esto, porque los hábitos de pensamiento y los patrones de diseño de Rust son muy distintos de los de zinc. Por ejemplo, muchas estructuras de datos internas de un compilador implementado en Rust hay que cambiarlas a índices en lugar de punteros; esa forma de pensar es muy distinta de la de otros lenguajes. Como ya estaba decidido que más adelante se reescribiría en zinc, más valía escribir la primera versión en C++, de modo que la segunda versión pudiera migrar de forma fluida sin cambiar la arquitectura principal.
2. Facilitar, tras el open source, aceptar contribuciones de otros desarrolladores. Si el compilador en sí está implementado en C++, ante el PR de un colaborador open source desconocido, personalmente me parece difícil hacer bien la revisión de código y mantener la calidad. Revisar funcionalidad implementada en un lenguaje de programación seguro es mucho más simple.
3. Practicar la idea de compiler as a service. Como es sabido, un lenguaje de programación no es solo implementar un compilador; también necesita todo un ecosistema para ser realmente útil. La parte de la cadena de herramientas de ese ecosistema incluye soporte de IDE, herramienta de formateo automático de código, herramienta de gestión de paquetes, herramienta de generación de documentación, avisos lint personalizados, análisis de calidad de código y otra serie de herramientas. Y muchas de esas herramientas necesitan compartir la misma lógica con el compilador. Así que, si podemos hacer que cada parte del compilador sea una library relativamente fácil de llamar, con una API segura y fácil de usar, construir las herramientas circundantes será mucho más fácil. Por el contrario, si el compilador es una caja negra y sus componentes no se pueden reutilizar con facilidad, construir las herramientas circundantes implica una gran cantidad de trabajo repetitivo. Usar C++ como interfaz API externa de los distintos subcomponentes del compilador no es lo bastante seguro ni fácil de usar.
4. Sentar las bases para las funciones posteriores de metaprogramación (macros, lint personalizados, etc.). La metaprogramación, desde el punto de vista de la implementación, es en realidad una función de plugins del compilador. El compilador necesita cargar de forma dinámica algunos módulos y ejecutar la función de esos plugins durante la compilación. Por supuesto, queremos que las definiciones de macros se implementen en zinc. Y zinc es precisamente un lenguaje bastante adecuado para compilarse como biblioteca de enlace dinámico; la API de la biblioteca compilada también puede admitir funciones avanzadas como genéricos y trait. Si el compilador, como quien llama, también está implementado en zinc, este mecanismo de plugins es muy limpio y conciso. Por el contrario, si se usa C++ para cargar de forma dinámica una biblioteca de enlace dinámico de zinc, aunque se puede hacer, es claramente un problema añadido.

## Introducción a la estructura del compilador

El compilador se divide en estos componentes:
* zinc_cli, programa ejecutable, depende de los componentes siguientes. Internamente contiene principalmente la función de procesamiento de línea de comandos.
    * options
    * session
* zinc_frontend, frontend del compilador, responsable de la comprobación sintáctica y semántica; contiene varios submódulos, de los cuales los dos más importantes son:
    * zinc_syntax  la entrada principal es el código fuente; la salida es la estructura de datos SyntaxTree. Incluye la función de error de sintaxis. El lexer también está incluido en este componente.
    * zinc_semantic  la entrada principal es la estructura de datos SyntaxTree; la salida es la estructura de datos SemanticModel. Incluye la función de error semántico.
        * hir
        * mir
* zinc_backend, backend del compilador, responsable de la optimización de rendimiento y la generación de código; contiene varios submódulos, de los cuales los dos más importantes son:
    * optimization  optimización de rendimiento; la entrada es la estructura de datos SemanticModel, que se modifica y transforma
    * llvm_ir_gen  la entrada principal es la estructura de datos SemanticModel; la salida es LLVM IR. También incluye algunas optimizaciones de rendimiento.


Todos dependen de zinc_std, que es la biblioteca estándar de zinc.

## Compilación incremental

TODO:

Las estructuras de datos Copy-On-Write son especialmente adecuadas para implementar la compilación incremental. Combinadas con la evaluación perezosa Query-Based, se pueden aprovechar al máximo los resultados en caché de la última compilación.


## No se admite sobrecarga de funciones basada en el tipo de los parámetros

El propósito principal es admitir mejor la publicación de componentes mediante bibliotecas de enlace dinámico.

El escenario central es: cuando un componente se publica hacia fuera como biblioteca de enlace dinámico, esperamos poder, al actualizar más adelante la versión de la biblioteca, que el usuario downstream no necesite recompilar el código y pueda enlazarlo y usarlo de forma directa.

Ahora supongamos que admitimos sobrecarga de funciones basada en el tipo de los parámetros; consideremos este caso:
en lib_v1 hay una función con firma `fn f(arg: *Base);`; en la versión lib_v2 queremos añadir una versión sobrecargada `fn f(arg: *Sub);`, donde `Sub` es un trait que hereda de `Base`.
Si el usuario downstream no recompila el código, al usarlo junto con lib_v2 el código sigue pudiendo ejecutarse, pero siempre llamará a la versión `Base`. Esta sobrecarga nueva no tendrá efecto en absoluto.

Cuando decimos “no se admite sobrecarga de funciones basada en el tipo de los parámetros”, el significado real es “no se admite resolver la versión de la función en tiempo de compilación según el tipo de los parámetros”; exigimos de forma forzada que el usuario resuelva la versión de la función en runtime.

Siguiendo el escenario anterior, si el usuario necesita en lib_v2 un tratamiento extra para un parámetro de tipo `*Sub`, el usuario debe modificar el cuerpo de la función original, no añadir una firma de función de una versión sobrecargada:
```
fn f(arg: *Base) {
    if (arg is s:*Sub) {
        // se juzga mediante coincidencia de patrones; si arg es de tipo *Sub, se ejecuta la lógica especial
    } else {

    }
}
```
Tras publicar la biblioteca de enlace dinámico de la nueva versión lib_v2, el programa ejecutable no necesita recompilarse y el comportamiento de esta aplicación se actualiza automáticamente.

Hacerlo así, por supuesto, sacrifica eficiencia de ejecución. Pero para lograr una actualización transparente de la biblioteca de enlace dinámico, este coste de runtime es necesario. Resolver la sobrecarga en tiempo de compilación no puede obtener este dinamismo.

Este diseño no tiene problema de capacidad de expresión, solo de rendimiento de ejecución. Y el rendimiento de ejecución tiene oportunidad de recuperarse en parte mediante diversas medidas de optimización. Sobre todo cuando el usuario no necesita “publicar una biblioteca de enlace dinámico con ABI estable”, se puede indicar al compilador mediante una opción de compilación que active optimizaciones más agresivas.

## Sin excepciones (exception) ni unwind de pila

El lenguaje zinc ofrece la palabra clave `panic`, que se puede comparar con la palabra clave `throw` de otros lenguajes. Pero no es lo mismo que el `throw` de otros lenguajes ni que el `panic!` de rust.

El panic de Rust tiene dos modos de comportamiento, `abort` y `unwind`, que se pueden indicar con esta opción de compilación de `rustc`.
```
-C    panic=val -- panic strategy to compile crate with
```
* El `panic` de rust, en la estrategia `abort`, termina el proceso de forma directa; la función `catch_unwind` no tiene efecto. Equivale a llamar a la función `abort` de libc.
* El `panic` de rust, en la estrategia `unwind`, ejecuta el unwind de pila y llama al destructor de las variables locales de cada capa de función, incluido el decremento automático del conteo de referencias de los punteros ARC, etc., y además puede ser capturado por `catch_unwind`, deteniendo el proceso de unwind de pila. Este proceso es similar al exception de otros lenguajes.

El `panic` de zinc es simplemente imprimir la pila de llamadas de función y luego abort. No tiene proceso de unwind, no se puede capturar y solo equivale al comportamiento del `panic` de rust en la estrategia `abort`. zinc no admite el comportamiento `unwind`.

Este diseño se eligió inspirado en este [blog](https://smallcultfollowing.com/babysteps/blog/2024/05/02/unwind-considered-harmful/) de nikomatsakis. Los motivos son los siguientes:

1. unwind aumenta la dificultad de implementación del compilador y de la biblioteca estándar
    1. unwind aumenta el volumen de código. Porque cada llamada a función puede producir unwind de pila; el compilador necesita registrar, en cada lugar donde puede ocurrir unwind de pila, qué destructores de variables locales hay que llamar.
    2. unwind reduce las oportunidades de optimización. El grafo de flujo de control del código se vuelve más complejo; en muchos sitios aparece una rama extra de flujo de control en la que puede ocurrir unwind, lo que afecta al análisis estático del compilador.
    3. unwind exige que la biblioteca estándar, al implementarse, considere el problema de exception safety.
2. unwind aumenta el coste de aprendizaje y uso para el usuario
    1. zinc, como lenguaje tardío, difícilmente puede construir un ecosistema independiente. En la mayoría de las aplicaciones, es muy probable que solo se use zinc para desarrollar uno de los módulos, lo que significa que la interoperabilidad es importante. Si zinc introduce un mecanismo de unwind, ¿qué relación tiene con el exception de C++ y el panic de Rust ya existentes? ¿Se pueden capturar mutuamente? Si esperamos que un módulo de bajo nivel escrito en zinc pueda ser llamado por lenguajes de alto nivel como Go/C#/Java/Javascript/Python, ¿pueden esos lenguajes de alto nivel capturar el panic que lanza zinc? ¿Este mecanismo, para el usuario, es una comodidad o un lío añadido?
    2. Introducir un mecanismo de unwind en el lenguaje exige que todas las bibliotecas de terceros, al diseñarse, consideren el problema de exception safety. Sobre todo cuando se involucra unsafe e interoperabilidad, esto es difícil. El posicionamiento de zinc enfatiza la simplicidad y la facilidad de uso; ¿no es un poco alta la exigencia de que un desarrollador ordinario también considere el diseño de exception safety?
    3. ¿Qué escenarios de manejo de errores son incómodos de expresar con `Option` `Result` y necesariamente hay que hacerlos con un unwind capturable? Si implementamos esta característica, ¿qué escenarios se beneficiarían, y son esos escenarios lo bastante convincentes?

Personalmente, creo que el ecosistema de zinc debería construirse así: todos los “errores recuperables” devuelven de forma unificada el tipo `Option/Result`; ese tipo de error significa que este proceso, al encontrarlo, tiene medios de tratamiento. Todos los “errores irrecuperables” usan de forma unificada `panic`; el significado de ese tipo de error es que la aplicación actual, en la fase de ejecución, no tiene ningún medio de tratamiento para ese error; es un bug del código, y la única solución es volver a modificar el código, recompilar el programa y volver a ejecutarlo.

Si alguien no está de acuerdo con esta decisión, debe escribir una propuesta de diseño completa para abrir el debate; el contenido debería responder de forma suficiente a las dudas anteriores.

## Genéricos
