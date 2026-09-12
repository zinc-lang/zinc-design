# Prólogo

<img src="../../assets/zinc-logo.png" width="10%" align=center />

Zinc es un lenguaje de programación de sistemas inspirado en Rust y más fácil de usar.

## Características

1. Tipado estático, runtime ligero y sin GC
  * Seguridad de tipos
  * Compilación directa a código máquina, sin máquina virtual y con arranque rápido
  * Sin GC, con poco overhead de memoria
  * Control del layout de memoria
  * Backend basado en el framework LLVM, lo que facilita el multiplataforma
  * Interoperabilidad sencilla

2. Seguridad
  * Memoria segura por defecto: solo con unsafe se puede producir comportamiento indefinido
  * Thread-safe por defecto: solo con unsafe se puede producir data race

3. Facilidad de uso
  * No persigue abstracciones de coste cero (zero-cost abstraction)
  * No hay tipos con semántica move; solo se admiten expresiones con semántica move
  * Hay tipos de borrow, pero no hay borrow checker, lo que evita el enorme impacto de ownership + borrow checker sobre el estilo de programación y mejora mucho la facilidad de uso respecto a Rust

4. Las funciones genéricas se implementan principalmente mediante table passing (paso de tablas)
  * Permite publicar componentes como bibliotecas de enlace dinámico, con interfaces complejas que incluyen genéricos
  * Permite que las funciones virtual lleven parámetros genéricos
  * A largo plazo se espera ABI estable, de modo que se puedan actualizar componentes por separado sin recompilar las dependencias descendentes
  * Ofrece opciones de compilación para que, cuando no importe la estabilidad del ABI, el usuario pueda instanciar las funciones genéricas y optimizar aún más el rendimiento

### Comparación con Rust

1. La sintaxis se parece a Rust y el estilo de la biblioteca estándar también. Igualmente admite memoria segura y thread-safety.
2. La gestión de memoria usa principalmente conteo de referencias, no un mecanismo de ownership.
3. Mejora la facilidad de uso: admite borrow, pero elimina el borrow checker; admite expresiones con semántica move, pero no tipos con semántica move.
4. Mejor soporte del paradigma de programación orientada a objetos
5. Cambia la implementación de genéricos: admite tanto instanciación como table passing (dictionary passing), y se elige con una opción de compilación
6. Parte de la sintaxis se simplifica

### Comparación con Swift

1. Mejor soporte de thread-safety, con thread-safety garantizada en tiempo de compilación
2. Neutral respecto a plataformas: no toma macOS como plataforma principal
3. Renuncia a la interoperabilidad con obj-C y se centra más en la interoperabilidad con C/C++
4. El soporte del paradigma OOP es distinto: por ejemplo, no admite class ni herencia
5. El soporte de move/borrow es distinto

### Comparación con C++

1. zinc es más seguro: sin unsafe no hay comportamiento indefinido
2. Implementación distinta de los genéricos: las funciones genéricas de zinc pueden usarse como funciones virtuales y también compiladas a código binario y publicadas como bibliotecas de enlace dinámico
3. A largo plazo, zinc puede lograr ABI estable
4. El rendimiento de ejecución es bastante inferior al de C/C++, pero debería bastar para el desarrollo de aplicaciones
5. Características de lenguaje más simples, menos lastre histórico, mejor soporte de dinamismo en runtime y menor tamaño de código

### Comparación con Go

1. Sin GC, con menores requisitos de memoria
2. Mejor soporte de características del lenguaje, incluidos genéricos, trait, enum, coincidencia de patrones, etc.
3. Más adecuado para publicar componentes como bibliotecas de enlace dinámico
4. Mejor soporte de thread-safety
5. Mejor capacidad de interoperación entre lenguajes. Go trae GC propio y es difícil interoperar con otros lenguajes que también traen GC.

## Preguntas frecuentes

### 1. ¿Cuál es el posicionamiento del lenguaje Zinc?

La intención original de Zinc es mejorar la facilidad de uso de Rust.

En el diseño de lenguajes de sistemas hay tres dimensiones habituales de evaluación: seguridad, rendimiento y facilidad de uso. Rust lleva la seguridad y el rendimiento al extremo, pero la facilidad de uso sigue siendo un punto débil.
Zinc mantiene la seguridad y, a cambio de no perseguir el rendimiento extremo, mejora la facilidad de uso.

El motivo de este cambio es que el autor observa que, en la inmensa mayoría de escenarios a los que se enfrenta un desarrollador ordinario, el rendimiento de CPU no es el cuello de botella; los casos en los que hay que exprimir el CPU al límite son, al fin y al cabo, una minoría.
En la mayoría de los casos, la gente no elige un lenguaje de sistemas sin recolección de basura por el rendimiento extremo, sino por otras razones. Entre ellas, sin limitarse a:

* Querer un control fuerte del layout de memoria y predictibilidad del rendimiento de ejecución
* Tener necesidades multiplataforma, y en algunas plataformas ser difícil portar el runtime de una máquina virtual
* En algunas plataformas los recursos de memoria son escasos y no se puede usar recolección automática de basura
* Querer escribir bibliotecas de componentes públicos básicos que puedan ser invocadas por distintos lenguajes de alto nivel con mecanismos de GC diferentes. Un lenguaje con GC propio difícilmente puede interoperar de forma eficiente, en el mismo proceso, con lenguajes que usan otro GC

Por tanto, un lenguaje con este posicionamiento debería tener escenarios de uso adecuados:
1. Sin recolección automática de basura, fácil de portar entre plataformas y fácil de interoperar con otros lenguajes de alto nivel. Adecuado para escribir módulos públicos y reutilizables.
2. Buena facilidad de uso manteniendo la seguridad. Menores exigencias al desarrollador y mayor productividad.
3. Rendimiento aceptable, sin necesidad de perseguir el rendimiento extremo ni de exprimir todo el potencial del hardware.

Este diseño no encaja con el posicionamiento de Rust, así que este tipo de cambio solo se puede explorar arrancando un lenguaje de programación nuevo.

El posicionamiento de Zinc es similar al de Swift. Sus rasgos centrales son: sin recolección de basura + seguridad + facilidad de uso.
Además tiene la ventaja de la thread-safety, que Swift no tiene.

### 2. ¿Comparación entre Zinc y Rust?

* Rust insiste en el principio de “zero-cost abstraction”; Zinc abandona ese principio de diseño. Cuando no se pueden conciliar rendimiento de ejecución y facilidad de uso, se prioriza la facilidad de uso.
* Rust usa el mecanismo de “ownership + lifetimes” para garantizar la seguridad de memoria; el borrow checker comprueba en tiempo de compilación aliasing XOR mutation. Zinc usa conteo de referencias y borrow para garantizar la seguridad de memoria, y las reglas de comprobación del compilador son mucho más laxas. Eliminar el borrow checker hace que la facilidad de uso de Zinc mejore enormemente respecto a Rust.
* Rust implementa los genéricos por instanciación; Zinc los implementa por table passing. Eso significa que Rust no es adecuado para compilar código genérico a bibliotecas binarias y publicarlas, mientras que Zinc sí. El soporte de bibliotecas de enlace dinámico y la estabilidad del ABI son objetivos importantes posteriores de Zinc.
* Rust garantiza seguridad de memoria y thread-safety; Zinc también garantiza seguridad de memoria y thread-safety. En seguridad no se ha debilitado nada; solo se paga un cierto coste de rendimiento.

En resumen, en sintaxis y semántica Zinc se parece mucho a Rust; pero en ejecución Zinc se parece mucho a Swift.

### 3. ¿Por qué se llama Zinc?

Rust representa el óxido y la corrosión; Zinc representa el metal zinc. Este metal tiene una característica:

>
> El zinc reacciona fácilmente con el oxígeno en el aire y forma una capa de óxido de zinc, es decir, la capa de pasivación de la superficie del lingote de zinc.
> La capa de pasivación tiene cierto efecto protector y puede inhibir la oxidación y corrosión posteriores.
> La aplicación principal del zinc es el galvanizado del hierro contra la corrosión.
>
> Zinc is most commonly used as an anti-corrosion agent, and galvanization (coating of iron or steel) is the most familiar form.
> (from: https://en.wikipedia.org/wiki/Zinc)
>

Esta palabra refleja la esencia del lenguaje: Zinc se parece a Rust en la superficie, pero bajo la sintaxis superficial su núcleo de ideas es incompatible con Rust.

Por coincidencia, en chino esta palabra también suena bien y es homófona de “lenguaje nuevo”.

### 4. ¿Por qué se eligió el conteo de referencias (reference counting, RC) como mecanismo principal de gestión de memoria?

Ventajas de RC:
* Una ventaja de RC es que permite implementar con facilidad la seguridad de memoria. La base teórica de la seguridad de memoria es "aliasing xor mutation". RC es especialmente adecuado para rastrear el aliasing y, combinado con copy-on-write, permite una muy buena seguridad.
* RC encaja muy bien con el diseño de Zinc en thread-safety. La thread-safety y la gestión de memoria por RC se complementan y se potencian mutuamente. Esa es la razón central por la que Zinc no puede usar GC. Si se adoptara GC, Zinc no podría alcanzar el nivel actual de thread-safety.
* Otra ventaja muy buena de RC es que es simple. Eso significa que impone pocas restricciones al entorno externo, así que interoperar con cualquier otro lenguaje es sencillo. El GC, en cambio, tiene detalles de implementación demasiado complejos, y distintos mecanismos de GC difícilmente pueden coexistir de forma eficiente en el mismo proceso. Si un lenguaje con GC propio interopera con otro que usa un GC distinto, se está buscando problemas: es muy incómodo.

Desventajas de RC:
* Una desventaja de RC es que no puede resolver el problema de fugas por referencias circulares. Pero eso no es un problema de seguridad de memoria; su impacto es relativamente menor. Además, en el ámbito de los lenguajes de sistemas, otros lenguajes como C++/Rust/Swift tampoco lo han resuelto. Basta con que el asignador de memoria y el depurador cooperen bien para detectar y resolver el problema en desarrollo y pruebas. Si solo para resolver esto se añadiera en runtime un mecanismo de mark-and-sweep, el coste sería demasiado alto y no encajaría con el posicionamiento de Zinc. Por supuesto, no me opongo a implementar un escáner de heap solo para la fase de pruebas que, combinado con pruebas automatizadas, ayude a descubrir referencias circulares en los casos de prueba.
* Otra desventaja de RC es que, si se usan con alta frecuencia instrucciones atómicas para incrementar y decrementar el conteo de referencias, el rendimiento de ejecución no es bueno.

Pero podemos usar algunas técnicas de optimización para recuperar un poco de rendimiento:
1. Zinc fomenta de forma natural el uso de tipos con semántica de valor; en muchos casos no hace falta asignar memoria dinámica y también se tiene un control bastante bueno del layout de memoria de los objetos. Esto es amigable con la caché y también reduce los escenarios en los que aparecen punteros de conteo de referencias.
2. Zinc conserva los tipos de puntero borrow; copiar, pasar como parámetro y devolver un puntero borrow no modifica el conteo de referencias. En muchos escenarios un puntero borrow es más razonable que un puntero RC.
3. Zinc conserva la semántica move; usarla de forma razonable puede reducir de forma notable la frecuencia del conteo de referencias.
4. Los punteros RC de Zinc son tipos integrados en el lenguaje, no simulados con una biblioteca. Así el compilador tiene oportunidad, en algunos escenarios, de eliminar por optimización operaciones redundantes de incremento y decremento del conteo.
5. El usuario puede usar asignadores de memoria más avanzados (como mimalloc) en lugar del malloc/free de libc, y mejorar la eficiencia de la asignación y liberación de memoria dinámica.
6. Gracias a las características de thread-safety del lenguaje, el compilador puede analizar que ciertos punteros RC y todas sus copias solo pueden usarse dentro del hilo actual, y por tanto traducir el incremento y decremento del conteo de esos punteros a instrucciones no atómicas. En Zinc, solo cuando el puntero apunta a un tipo que cumple la restricción `Sync` puede usarse de forma segura entre hilos. Si un puntero apunta a un tipo que no es `Sync`, podemos estar seguros de que ese puntero y todas sus copias no pueden usarse entre hilos; ahí hay oportunidad de optimización usando instrucciones no atómicas. **Esa es también la razón por la que se llama puntero de conteo de referencias automático (automatic reference counting): la letra A representa Automatic, no Atomic. El significado principal de “automático” es que el compilador elige automáticamente si usar instrucciones atómicas o no atómicas.**

Estas diferencias muestran que las conclusiones de rendimiento de los mecanismos de conteo de referencias basados en atomic-rc tradicional de otros lenguajes no se aplican a zinc. Comparado con el atomic-rc tradicional, Zinc tiene un mejor potencial de optimización de rendimiento.

En resumen, seguridad, rendimiento y facilidad de uso forman un triángulo imposible.
Cuando Zinc elige seguridad y facilidad de uso, queda destinado a no poder lograr “zero-cost abstraction”; cierta pérdida de rendimiento frente a C/C++ es aceptable.
Aun así, tenemos una serie de técnicas de optimización para garantizar que el rendimiento de ejecución no sea especialmente malo.

Personalmente sospecho que Arc no es el cuello de botella de rendimiento de Zinc, sino el esquema de implementación de los genéricos.

### 5. ¿Cuál es el objetivo de este proyecto?

El diseño de Rust es muy bueno en muchos aspectos; eso está a la vista de todos. Al mismo tiempo, las voces que piden simplificar el umbral de aprendizaje y uso de Rust nunca han cesado.
Siempre he pensado que, en el ámbito de los lenguajes de sistemas seguros, la dirección de diseño de Rust no es la única solución; todavía hay otro espacio de diseño.
El propósito de este proyecto es explorar otras posibilidades de diseño dentro del ámbito de los “lenguajes de sistemas seguros”.

Pero si solo se habla en abstracto de cómo se podría diseñar, nadie escucha. En el círculo de programadores hay una frase muy famosa:
> 
> Talk is cheap, show me your code.
>

Así que necesito un proyecto que implemente de verdad todas las ideas, para que tenga poder de convicción.

La dirección central de diseño de este proyecto es:
**si se mantiene la seguridad de Rust, se sacrifica un poco de rendimiento de ejecución y se gana mejor facilidad de uso, ¿en qué podría convertirse?**

Sobre esa base hay varios objetivos secundarios:
1. Simplificar las características del lenguaje y oponerse a querer abarcarlo todo. Que las distintas características sean lo más ortogonales posible, que no se influyan mutuamente y que se puedan combinar libremente. Evitar parchear las reglas semánticas con reglas especiales. Reducir la carga mental del usuario.
2. Simplificar la implementación del compilador. El compilador de Rust ya es demasiado complejo; casi nadie puede entenderlo por completo. Un proyecto tan hipercomplejo es difícil de hacer avanzar y evolucionar de forma sostenible. Si una característica del lenguaje es demasiado compleja de implementar, suele significar que el usuario del lenguaje tampoco podrá dominarla del todo; entonces deberíamos evitar introducir esa característica desde el principio.
3. Preocuparse por la velocidad de compilación de proyectos grandes y ofrecer buenas sugerencias de IDE. Esto no es solo un problema de implementación del compilador, sino también de diseño de las características del lenguaje.
4. Practicar la idea de compiler as a service. El compilador no debería ser una caja negra; sus componentes deberían publicarse como bibliotecas comunes para la comunidad. Cada componente del compilador debería tener una API pública fácil de usar. Eso impulsará enormemente el desarrollo del ecosistema circundante.
5. Mejor soporte de dinamismo basado en un lenguaje compilado. Incluye: publicar funciones genéricas como artefactos binarios, usar funciones genéricas como funciones virtuales, publicar bibliotecas de enlace dinámico como artefactos binarios, ABI estable, carga dinámica, cierto grado de reflexión en runtime, etc. Todo esto es algo en lo que Rust no destaca especialmente y Swift lo hace mejor.

### 6. Zinc no tiene ninguna innovación especial; solo reordena y combina características de otros lenguajes. ¿Tiene sentido?

Sí tiene sentido. Porque el propósito de este proyecto no es innovar, sino la seguridad y la facilidad de uso. Si la combinación de estas características hace que el usuario sienta seguridad, facilidad de uso y encaje con el escenario de necesidad, basta. Satisfacer las necesidades del usuario es mi objetivo, no innovar por innovar.

En el ámbito de la programación de sistemas, los lenguajes disponibles no son muy abundantes. El autor considera que siguen existiendo ciertos escenarios para los que no tenemos un lenguaje especialmente adecuado.

* C/C++ es muy potente, pero cuando aparece comportamiento indefinido, buscar bugs en código a gran escala es extremadamente doloroso. Para un usuario ordinario, la inmensa mayoría de escenarios no necesita rendimiento extremo; sacrificar un poco de rendimiento de ejecución a cambio de más seguridad es, en esos casos, una decisión rentable.
* Rust es un muy buen lenguaje, pero algunas características están diseñadas de forma demasiado compleja, el paradigma de programación no es amigable para muchos usuarios, la productividad es baja, no es amigable con las bibliotecas de enlace dinámico y no encaja en escenarios que necesitan dinamismo.
* Swift también es muy bueno: como lenguaje nativo tiene buen soporte de dinamismo y también compatibilidad binaria, que son muy buenas propiedades. Pero está controlado por Apple y se centra básicamente en las plataformas de Apple. En muchos escenarios queremos usarlo, pero no podemos. También tiene problemas de thread-safety.
* Zig, como lenguaje de sistemas emergente, aún no es lo bastante maduro. Y también tiene problemas de seguridad de memoria y thread-safety.
* Zinc, en cambio, absorbe en lo posible las ventajas de Rust y Swift. Primero, en seguridad se alinea con Rust y garantiza en tiempo de compilación la seguridad de memoria y la thread-safety; segundo, simplifica las características del lenguaje y reduce la dificultad de uso; tercero, tiene mejor soporte de dinamismo y admite la evolución binaria compatible de bibliotecas de enlace dinámico.

En ingeniería no hay magia, solo compromisos y cesiones según distintos objetivos. Encontrar un escenario de uso relativamente habitual y diseñar características de lenguaje que lo cubran es el principal desafío del diseño de Zinc.
El autor considera que la demanda de "sin GC & seguridad & facilidad de desarrollo" es relativamente habitual, y en el mercado falta un lenguaje lo bastante competitivo que la cubra por completo.
Los escenarios que persiguen "rendimiento extremo" y "zero-cost abstraction" son, en la práctica real, muy pocos; para proyectos que ponen el rendimiento como máxima prioridad, se recomienda usar C/C++/Rust.
El diseño de Zinc no debería tratar esa demanda como prioridad máxima.

Si os parece que este diseño es útil, no dudéis en dejar un like, proponer sugerencias o crear un PR y participar en el diseño e implementación de este lenguaje.

### 7. ¿En qué estado se encuentra Zinc actualmente?

Las características centrales como la gestión de memoria y la thread-safety ya están diseñadas, hay una biblioteca estándar básica y, sobre esas bases, se ha logrado el bootstrapping del compilador.

El estado actual puede considerarse una fase de prueba de concepto (proof of concept): solo se ha empezado, y aún falta mucho para que sea realmente usable.
Muchas características clave del lenguaje, siempre que no impidan el bootstrapping, no se han implementado. Tampoco se han hecho las optimizaciones de rendimiento previstas, ni las herramientas circundantes, ni la mejora de la biblioteca estándar, etc.
El desarrollo y la madurez posteriores siguen exigiendo mucho trabajo, y solo será posible completarlos con la ayuda de la comunidad open source.

### 8. ¿En qué ocasiones es más adecuado usar Zinc? ¿Cómo se piensa promoverlo?

Creo que, para un lenguaje nuevo, ocupar el territorio de un lenguaje viejo con ecosistema maduro es extremadamente difícil. Por el contrario, si se encuentra un sector en fase embrionaria, se profundiza en el escenario nuevo y se construye un ecosistema nuevo, la dificultad de promoción es relativamente mucho menor.
Ampliar el incremento, no disputar el stock, puede reducir de forma notable el coste de promoción.

Algunos escenarios nuevos con potencial:

1. Productos de automóviles inteligentes conectados
2. Escenarios de robótica
3. Dispositivos IoT, dispositivos embebidos y todo tipo de dispositivos inteligentes de pequeño tamaño
4. El ámbito de la inteligencia artificial


### 9. ¿Qué licencia open source eligió Zinc?

Zinc eligió la licencia open source más permisiva, MIT. Porque en esta época, un lenguaje de programación que no sea lo bastante abierto no podrá atraer suficientes participantes de la comunidad; está condenado y no tiene ningún sentido.
Deseo que este proyecto no solo sea open source, sino también abierto. Estoy muy dispuesto a escuchar opiniones y sugerencias, y doy la bienvenida a todos a participar en el diseño e implementación de Zinc.
