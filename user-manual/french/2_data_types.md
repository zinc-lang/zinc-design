
# Types de données de base

Convention de nommage : tous les types nommés utilisent le UpperCamelCase. (Sauf le mot-clé `fn`, car ce mot-clé est surtout utilisé dans les définitions de fonctions.)

## Type Bool

Le type `Bool` a deux valeurs possibles, `true` et `false`.

La négation d'un `Bool` s'obtient avec l'opérateur `!`.

## Entiers

|  Taille |  Signé  |  Non signé  |
|-------|---------|----------|
| 8 bit | Int8      |  UInt8      |
| 16 bit | Int16      |  UInt16      |
| 32 bit | Int32      |  UInt32      |
| 64 bit | Int64      |  UInt64      |
| arch-dependent | Int      |  UInt  |
| arch-dependent | Short    | UShort  |

La taille des types Int/UInt dépend de la plateforme cible. Sur une plateforme 64 bit, ils font 64 bit ; sur une plateforme 32 bit, ils font 32 bit. Ils sont équivalents à `intptr_t` / `uintptr_t` en C.
La taille des types Short/UShort dépend de la plateforme cible, et vaut toujours la moitié de Int/UInt. Sur une plateforme 64 bit, ils font 32 bit ; sur une plateforme 32 bit, ils font 16 bit.


Les littéraux entiers autorisent `_` comme séparateur.

Des préfixes distinguent la base : `0o` pour l'octal, `0x` pour l'hexadécimal, `0b` pour le binaire.

Les littéraux entiers autorisent un suffixe de type. Le suffixe est en minuscules, par exemple `123_i8`  `100_i`  `0x64_u`.
En l'absence d'autre information de contexte, le type d'un littéral entier ne peut pas être inféré. Il est recommandé d'ajouter un suffixe de type aux littéraux entiers, pour indiquer explicitement leur type.

Les opérations arithmétiques entières sont toujours en mode wrapping en cas de débordement. Pour d'autres comportements, veuillez appeler la bibliothèque standard. La bibliothèque standard fournit aux entiers des fonctions membres telles que `checked_add` (pas encore implémentées).

Zinc ne fournit pas d'opérateurs de « bit operations », notamment décalage à gauche, décalage à droite, ET bit à bit, OU bit à bit, NON bit à bit, XOR bit à bit.
Ces opérations sont fournies dans la bibliothèque standard sous forme de fonctions membres des types entiers, par exemple `bit_and`, ce qui suffit aux besoins fonctionnels.

Raison : les opérations bit à bit sont peu fréquentes, et ne s'appliquent qu'aux types entiers ; concevoir des opérateurs dédiés serait un peu du gaspillage.
De plus, les opérations bit à bit ne se limitent pas à celles ci-dessus : il y a aussi rotate, reverse, count_ones, etc., une dizaine d'opérations peu courantes. Celles-ci ne peuvent de toute façon être fournies que comme fonctions membres. Il me semble plus cohérent de fournir toutes les opérations bit à bit comme fonctions membres.
Les symboles de la table ASCII ne sont déjà pas très nombreux ; il est plus raisonnable de les réserver à d'autres fonctionnalités plus importantes.


## Nombres flottants

Les nombres flottants se divisent en types `F32` et `F64`.

Les littéraux flottants autorisent `_` comme séparateur.
Les littéraux flottants autorisent un suffixe de type, respectivement `f32` et `f64`.

Les littéraux flottants n'autorisent pas l'omission du 0 devant le point décimal. `.123` ou `-.1` produisent une erreur.

## Type caractère Char

Le type `Char` fait 32 bit et représente un unicode code point.

Un littéral `Char` est entouré de simples quotes. Par exemple `'好'`.

Un littéral `Char` autorise l'échappement. `'\u{3456}'`.

Les littéraux Byte sont supportés, `b'a'`. La valeur d'un littéral Byte ne peut être qu'un caractère ascii, et son type est `UInt8`.

La bibliothèque standard définit un alias de type `type Byte = UInt8;`.

## Tableaux

Les éléments d'un tableau sont tous du même type. La longueur d'un tableau doit être une constante de compilation.

Un tableau d'éléments de type `T` et de longueur `N` s'écrit `[T; N]`. Un tableau est un type valeur : à l'affectation, au passage en paramètre et au retour, tous les éléments sont copiés.

Exemple :

```rust
fn main() {
    let x: [Int; 3] = [1, 2, 3];
}
```

Les littéraux de tableau ont deux formes :

1. Énumérer directement tous les éléments entre crochets, séparés par des virgules.

    `[1, 2, 3]`

2. Séparer l'élément et le nombre par un point-virgule entre crochets, pour représenter plusieurs éléments identiques.

    `[1; 3]`

L'accès à un élément de tableau se fait par une expression d'indexation ; un dépassement de bornes provoque un panic. Exemple :

```rust
fn main() {
    let mut x: [Int; 3] = [1, 2, 3];
    x[1] = 10;
    println(x[5]); // compilation OK, panic à l'exécution
}
```

## Slices

Un slice de tableau est un pointeur borrow vers une partie d'un tableau. Il existe deux types de slices, `Slice<'a, T>` et `SliceMut<'a, T>`, représentant respectivement un accès en lecture seule et un accès en lecture-écriture.

```rust
fn main() {
    let x: [Int; 3] = [1, 2, 3];
    let s: Slice<Int> = &x;
    println(s.len());
    println(s[1]);
}
```

Un slice de tableau est une structure portant un lifetime ; on peut le voir comme un pointeur borrow enrichi de métadonnées. Dans l'exemple ci-dessus, la variable `s` occupe deux pointeurs, qui stockent respectivement l'adresse de début et la longueur.

Note : on n'utilise pas la syntaxe `&[T]` de Rust afin de simplifier le système de types et la difficulté d'implémentation.
En Rust, `[T]` est un type légitime, utilisable de façon autonome, et dyn sized. Cela introduit un cas particulier dans le système de types, gênant pour l'implémentation des génériques. Zinc souhaite que l'utilisateur ne puisse utiliser que les types `&[T]/&mut [T]`, sans type `[T]` dyn sized autonome, ni types `Box<[T]>` `Rc<[T]>`, etc.
On a donc remplacé la syntaxe `&[T]/&mut[T]` et fourni uniquement les types `Slice/SliceMut` dans la bibliothèque standard. Leur sémantique équivaut à exiger, comme en Rust, que `[T]` ne puisse être utilisé qu'avec un pointeur borrow. Le `Slice<'a, T>` de Zinc est équivalent au `&'a [T]` de Rust.

> **TODO** : le type Range n'est pas encore implémenté, mais les opérateurs sont déjà réservés, avec la syntaxe de Swift. Il y a deux types de Range, l'intervalle ouvert `..<` et l'intervalle fermé `..=` ; `RangeFull` est supprimé.
Les opérations de slice ne peuvent pas encore s'exprimer avec l'opérateur d'indexation `[]` combiné au type Range, car la surcharge d'opérateurs n'est pas encore implémentée.
```
let s: Slice<Int> = &x[1..<3]; // supporté ultérieurement
```

## Chaînes de caractères

Il existe deux types de chaînes, `Str` et `String`.

`Str` est une structure définie dans la bibliothèque standard, portant un lifetime : `struct Str<'a>`. Le type `String` est défini dans la bibliothèque standard, implémenté à partir de `Vec<UInt8>`, et est une structure sans lifetime.

Un littéral de chaîne est de type `Str<'static>`.

Note : comme pour les slices de tableaux, on n'utilise pas le mot-clé `str` de Rust afin de simplifier le système de types et la difficulté d'implémentation.
En Rust, `str` est un type légitime, utilisable de façon autonome, et dyn sized. Cela introduit un cas particulier dans le système de types, gênant pour l'implémentation des génériques. Zinc souhaite que l'utilisateur ne puisse utiliser que le type `&str`, sans type `str` dyn sized autonome, ni types `Box<str>` `Rc<str>`, etc.
On a donc remplacé le mot-clé `str` et fourni uniquement le type `Str` dans la bibliothèque standard. Sa sémantique équivaut à exiger, comme en Rust, que `str` ne puisse être utilisé qu'avec un pointeur borrow en lecture seule. Le `Str<'a>` de Zinc est équivalent au `&'a str` de Rust.

L'intérieur d'une chaîne est encodé en utf8.

Pour le comprendre avec l'esprit C++, le `String` de Zinc correspond au `std::string` de C++, sauf que l'implémentation interne utilise un mécanisme copy-on-write. Le `Str` de Zinc correspond au `std::string_view` de C++, avec en plus une vérification de lifetime à la compilation.

Les littéraux de chaîne supportent l'échappement.

todo: supporter les raw string literals. `r##" content "##`.

todo: supporter l'interpolation de chaînes (string interpolation), c'est-à-dire des expressions imbriquées à l'intérieur d'un littéral de chaîne. Exemple :

```
let name: Str = "Amy";
let s: String = f"hello $(name), how are you?";
```

On a choisi les parenthèses plutôt que les accolades, car en Zinc les parenthèses servent généralement aux expressions. Une expression a un type et une valeur. Les accolades servent généralement aux instructions ; une instruction n'a ni type ni valeur.

TODO: pour l'instant, les expressions imbriquées ne supportent que le cas le plus simple d'un seul nom de variable ; le support de toutes les expressions sera ajouté ultérieurement.

## Tuple

Un tuple contenant 0 élément s'appelle le type unité (unit type). Le type unité s'écrit `()`. Le type de l'expression tuple vide `()` est le type unité.
Si le type de retour d'une fonction est omis, cela signifie que la fonction retourne le type `()`.

Un tuple contenant 1 élément nécessite une virgule supplémentaire en fin, sinon le compilateur le considère comme une « expression entre parenthèses », et non comme un tuple. `(1,)` est un tuple, `(1)` n'en est pas un.

Un tuple contenant 2 éléments ou plus peut avoir des éléments de types différents.

## Structure struct

On définit une structure avec le mot-clé `struct`. Exemple :

```rust
struct User {
    active: Bool,
    username: String,
    email: String,
}
```

Les structures supportent les génériques.
La structure elle-même et ses membres peuvent être décorés par `pub`.
Une structure peut n'avoir aucun membre entre les accolades.

Les membres d'une structure supportent une valeur par défaut, qui doit être une expression constante.
```rust
struct User {
    active: Bool = true, // valeur par défaut
    username: String = f"",
    email: String = f"",
}
```

L'expression d'initialisation d'une structure est similaire à celle du C :

```rust
let user: User = {  // le type de la liaison de variable indique le type de l'expression d'initialisation de structure
    .active = false,
    .username = "John".to_string(),
    .email = "john@gmail.com".to_string(),
};
// ou
let user = {  // un suffixe de type indique le type de l'expression d'initialisation de structure
    .active = false,
    .username = "John".to_string(),
    .email = "john@gmail.com".to_string(),
} : User;
```

Les structures ne supportent pas l'initialisation anonyme des membres par ordre, à la manière du C :
```
let user: User = { false, "John".to_string(), "john@gmail.com".to_string() };  // erreur
```

Les membres qui ont une valeur par défaut peuvent ne pas être initialisés explicitement.

On peut initialiser une structure à partir d'une autre. Attention : on utilise ici trois points `...`, et non les deux points `..` de Rust.
En Zinc, on utilise uniformément trois points `...` pour représenter les points de suspension. Cette syntaxe est utilisée dans plusieurs contextes. Les points de suspension peuvent être suivis ou non d'une expression.

```rust
// active peut être omis, ce qui signifie qu'il prend sa valeur par défaut
let user1: User = { .username = "smith".to_string(), .email = String::new() };

// initialiser user2 à partir de user1, seul le membre active change de valeur
let user2: User = { .active = true, ...user1 }; 

// initialiser user3 avec les valeurs par défaut de la définition du type User, seul le membre active change de valeur
let user3: User = { .active = true, ... }; 

// équivalent à tout initialiser avec les valeurs par défaut de la définition du type User.
// les points de suspension sont nécessaires, sinon l'analyse lexicale ne peut pas distinguer une initialisation de structure vide d'un bloc d'instructions vide
let user4: User = { ... }; 
```

Les structures ne supportent pas l'héritage. Zinc prévoit de supporter le paradigme orienté objet via l'héritage de trait et le mécanisme `reuse`.

Zinc ne prévoit pas de supporter les tuple structs, c'est-à-dire les structures dont les membres n'ont pas de nom. Pour une structure simple, il est recommandé de fournir séparément une fonction globale correspondante, nommée en snake_case, servant de « constructeur ».
Les règles du langage restent ainsi plus concises et uniformes ; du point de vue de l'usage, les conventions de nommage sont aussi plus uniformes. Il n'apparaît pas d'objet « ressemblant à une fonction » commençant par une majuscule.
Par exemple :

```rust
struct Weight {
    w: UInt32
}

fn weight(w: UInt32) -> Weight {
    return { .w = w };
}

fn main() {
    // un peu plus de travail côté définition ; côté usage, c'est essentiellement comme un tuple struct
    let w = weight(1); 
}
```

Zinc supporte les « structures unité ». Une structure sans membres peut se terminer par un point-virgule, ou avoir des accolades vides :
```rust
// les deux écritures suivantes sont équivalentes
struct Unit{} 
struct Unit;
```
Attention toutefois : l'initialisation d'une structure unité nécessite tout de même des accolades :
```
let x: Unit = { ... };
```

## Énumération enum

Une énumération se définit avec le mot-clé `enum` :

```rust
enum IpAddrKind {
    V4,
    V6,
}

fn main() {
    let k: IpAddrKind = IpAddrKind::V4;
}
```

Les membres d'une énumération supportent des valeurs associées.
```rust
enum IpAddr {
    V4(Byte, Byte, Byte, Byte),
    V6(String),
}
```

Les valeurs associées entre accolades ne sont pas supportées. Si l'utilisateur a besoin de nommer les paramètres, il est recommandé d'emballer tous les paramètres dans une structure, l'effet est le même :

```rust
enum E {
    A{ x:Byte, y: String } // erreur
}
// écriture de remplacement :
struct EA {
    x: Byte,
    y: String,
}
enum E {
    A(EA),
}
```

La surcharge des membres d'énumération est supportée, avec les mêmes règles que la surcharge de fonctions : seule la surcharge avec un nombre de paramètres différent est supportée.

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

Une énumération est essentiellement une union étiquetée (tagged union). L'utilisateur peut spécifier, pour un enum personnalisé, le type entier du tag, ainsi que la valeur de tag de chaque membre.
La valeur de tag doit être une expression constante, doit pouvoir s'exprimer dans l'entier correspondant, et les valeurs de tag des membres ne doivent pas se répéter.

Seuls les enum sans valeur associée peuvent spécifier la valeur du tag.

```rust
enum IpAddr : UInt8 {
    V4 = 1,  //
    V6 = 2,
}
```

Un enum sans valeur associée peut être converti directement en entier via l'opérateur `as`.

La définition d'un type énumération supporte les génériques. Les membres de l'énumération ne peuvent pas introduire de nouveaux génériques.

> **Fonctionnalité prévue** : supporter la définition d'énumérations non exhaustives (non-exhaustive enum), avec comme syntaxe des points de suspension après le dernier membre :
```rust
enum Puctuation {
    Comma,
    Dot,
    Semi,
    ...  // points de suspension, équivalent à un enum décoré par #[non_exhaustive] en Rust
}
```

### Types pointeurs

Expliqués au chapitre 4.

### Quelques types énumération spéciaux définis dans la bibliothèque standard

1. `Option`

    ```rust
    enum Option<T> { None, Some(T) }
    ```

    Le type Option a une optimisation spéciale de disposition mémoire.

2. `Result`

    ```rust
    enum Result<T, E> { Ok(T), Err(E) }
    ```

    Result supporte l'opérateur `?`, ce qui facilite la gestion des erreurs.

## Union

Une union se définit avec le mot-clé `union`. La taille globale d'une union dépend de la taille du plus grand membre.

```
union U {
    data1: Byte,
    data2: UInt,
}
```

Le type union est unsafe : la lecture et l'écriture des membres d'un type union doivent se faire dans un contexte unsafe.

La syntaxe d'initialisation d'une union est similaire à celle d'une struct. Mais on ne peut choisir qu'un seul membre pour l'initialisation.

Un type non-trivial ne peut pas servir de membre d'un type union.
Les types non-triviaux comprennent :
1. Les types qui ont un destructeur, par exemple Arc/Weak, etc.
2. Les types qui portent des paramètres de lifetime, par exemple &/&mut, etc.
3. Si un membre d'une struct est un type non-trivial, alors cette struct est non-trivial
4. Si une valeur associée d'un enum est un type non-trivial, alors cet enum est non-trivial
5. Si l'élément d'un type tableau est un type non-trivial, alors ce tableau est non-trivial
6. Si un élément d'un type tuple contient un type non-trivial, alors ce tuple est non-trivial

La définition d'un type union ne peut pas porter de paramètres génériques.

Une union a principalement deux cas d'usage :
1. L'interaction avec C
2. La conversion de type bit à bit (similaire à `std::mem::transmute` en Rust)

Par exemple, pour convertir bit à bit un type `F32` en type `UInt32`, on peut s'aider d'une union :
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

## Taille des types

Les types se divisent en Sized et UnSized. Un type Sized peut servir de variable locale, de variable globale ou de membre ; un type UnSized ne le peut pas : on ne peut y accéder qu'indirectement via un pointeur.

Les types UnSized comprennent :
1. Les types déclarés par `extern type`
    * `extern type` sert surtout aux scénarios d'interopérabilité. Cela signifie que la définition du type se trouve dans un autre langage, et que Zinc n'y a pas accès. On ne connaît donc pas sa taille, et on ne peut pas accéder directement à ses membres.

2. Tous les traits
    * Si l'on définit `trait R`, on ne peut pas déclarer une variable de type `R`, mais on peut utiliser `&R / &mut R / *R`, etc.

3. Un type personnalisé `S` avec un `impl UnSized for S {}` explicite
    * `UnSized` est un trait intégré, qui permet à l'utilisateur de spécifier qu'un type personnalisé est aussi Unsized, via `impl UnSized for MyType {}`.
        Le but est d'interdire l'usage de ce type comme « type à sémantique de valeur », et de n'autoriser que l'usage via un pointeur : ce type n'a besoin que d'une « sémantique de référence ».
        Un cas typique est le type `File` de la bibliothèque standard. Ce type n'autorise pas l'utilisateur à l'utiliser comme « type à sémantique de valeur » : l'utilisateur ne peut obtenir qu'un type pointeur vers `File`.

Dans les scénarios génériques, un type UnSized n'est actuellement pas autorisé comme argument générique.
