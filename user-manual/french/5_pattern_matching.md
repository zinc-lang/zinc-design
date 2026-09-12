
# Filtrage par motif

Le filtrage par motif s'utilise principalement dans les expressions `is` et les instructions `match`.

Le but du filtrage par motif est de vérifier si la structure d'une expression correspond à ce qui est attendu, et d'autoriser aussi d'en extraire les valeurs internes pour les lier à de nouvelles variables.

Exemple :

```
// la variable x est un tuple de 3 éléments ; on lie le deuxième élément à la variable middle, puis on teste si middle > 5
if (x is (_, middle, _)) && middle > 5 {

}

// le type de la variable y est un pointeur vers un trait ; on peut ainsi déterminer le type concret réellement pointé par y. Si le test réussit, ce pointeur est lié à p.
if y is p: *S {
    // p can be used inside block
}

// la variable z est de type Option<S> ; on traite ci-dessous les deux cas Some et None
match z {
    Some(s) => {

    }
    _ => {}
}
```

Comme les expressions, les motifs peuvent être imbriqués.

## WildcardPattern

Motif joker.

`_` le souligné peut occuper une position.

`...` les points de suspension peuvent occuper plusieurs positions.

## LiteralPattern

Les littéraux des types entier, flottant, Bool, Char, etc. peuvent être utilisés directement comme motifs.

Les motifs littéraux `Str` ne sont pas encore implémentés.

## TypePattern

Le motif de type sert principalement à déterminer si une expression est un sous-type concret. Un motif de type commence par le mot-clé `type`.

Exemple :
```
// on suppose que TR est un nom de trait, S un nom de struct.
// la phrase suivante sert à déterminer si un pointeur vers un trait peut être downcasté avec succès en pointeur vers S.
fn test(p: &TR) {
    let b = p is type &S;  // on commence par le mot-clé type pour éviter un conflit de syntaxe avec le motif modifier+identifier qui suit
}
```

## StructPattern

Le motif de structure sert à correspondre à une valeur de type structure, et peut déstructurer ses membres.

Exemple :
```rust
struct Point { x: Int, y: Int }

fn test(p: Point) {
    // correspondre à une structure, lier les membres à de nouvelles variables
    if p is { .x = a, .y = b }: Point {
        println(f"x = $(a), y = $(b)");
    }
    
    // utiliser les points de suspension pour ignorer les autres membres
    if p is { .x = 10, ... }: Point {
        println("x is 10");
    }
    
    // filtrage par motif imbriqué
    match p {
        { .x = 0, .y = 0 } => println("origin");
        { .x = 0, .y = y } => println(f"on y-axis at $(y)");
        { .x = x, .y = 0 } => println(f"on x-axis at $(x)");
        { .x = x, .y = y } => println(f"point ($(x), $(y))");
    }
}
```

## TuplePattern

Le motif de tuple sert à correspondre à une valeur de type tuple, et peut déstructurer ses éléments.

Exemple :
```rust
fn test(t: (Int, String, Bool)) {
    // correspondre à un tuple, lier les éléments à de nouvelles variables
    if t is (num, str, flag) {
        println(f"num = $(num), str = $(str), flag = $(flag)");
    }
    
    // utiliser le souligné pour ignorer certains éléments
    if t is (1, _, true) {
        println("first element is 1 and third is true");
    }
    
    // motif de tuple imbriqué
    let nested = ((1, 2), (3, 4));
    if nested is ((a, b), (c, d)) {
        println(f"($(a), $(b)), ($(c), $(d))");
    }
}
```

## EnumVariantPattern

Le motif de variante d'énumération sert à correspondre à une variante particulière d'un type énumération, et peut déstructurer ses valeurs associées.

Exemple :
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

Le motif modificateur sert à contrôler le mode de liaison des variables dans le filtrage par motif.

- `mut` : marque la variable liée comme mutable
- `&` : correspond à une référence
- `&mut` : correspond à une référence mutable
- `ref` : lie par référence, plutôt que par déplacement
- `ref mut` : lie par référence mutable

Exemple :
```rust
fn test(s: String) {
    // modificateur mut : rend la variable liée mutable
    let mut x = s;
    x.push_str(" modified");
    
    // modificateur ref : lie par référence, évite le déplacement

    // modificateur ref mut : lie par référence mutable

}
```

## IdentifierPattern

Le motif identifiant sert à lier la valeur correspondante à un nom de variable.

Exemple :
```rust
fn test(x: Int) {
    // liaison simple par identifiant
    if x is y {
        println(f"y = $(y)");
    }
    
    // utilisation dans un match
    match x {
        0 => println("zero");
        n => println(f"other: $(n)");
    }
    
    // combinaison avec d'autres motifs
    let tuple = (1, 2, 3);
    if tuple is (first, ...) {
        println(f"first = $(first)");
    }
}
```



## Pas encore supportés
StrPattern
GroupedPattern
MacroInvocationPattern
RangePattern
SlicePattern
Pattern Guard
Or pattern
