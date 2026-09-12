## Expressions, instructions et fonctions

## Expressions

Zinc n'est pas un langage d'expressions.
1. Les mots-clés `break` `continue` `return` `goto` `panic`, etc., ne peuvent être utilisés que dans des instructions, pas directement dans des expressions.
2. Seules les expressions peuvent produire une valeur. Les instructions ne le peuvent pas.
3. Une instruction peut contenir une expression, mais une expression ne peut pas contenir directement d'instruction. Les expressions lambda font exception : elles peuvent contenir des instructions, mais les return/break/continue qu'elles contiennent n'affectent que l'intérieur du lambda ; à l'extérieur, le lambda se comporte toujours comme une expression et ne peut pas transférer le flux de contrôle.

Si l'utilisateur a besoin, dans une expression, d'exécuter dans l'ordre des instructions assez complexes, puis de produire finalement une valeur, il doit utiliser un lambda et l'appeler immédiatement.
Les instructions de contrôle de flux à l'intérieur du lambda n'agissent que dans le corps de fonction du lambda, sans effet sur l'expression extérieure.

### Expressions opérateurs

#### Type `Result`

Zinc utilise `Result<R, E>` comme mécanisme principal de gestion des erreurs. C'est un enum défini dans la bibliothèque standard :
```
enum Result<R, E> {
    Ok(R), Err(E)
}
```

Le `E` de `Result<R, E>` représente l'information d'erreur. L'utilisateur peut utiliser n'importe quel type pour décrire l'information d'erreur.

* Si les possibilités d'erreur sont très limitées, concevoir un enum comme information d'erreur est tout à fait raisonnable.

* Si l'on n'a besoin de l'information d'erreur que pour imprimer un log, utiliser simplement une chaîne comme type d'erreur ne pose pas de problème.

* Si l'on a besoin de classer les types d'erreur sur plusieurs niveaux, concevoir un ensemble `trait MyBussinessError` et sa hiérarchie d'héritage, et utiliser `*MyBussinessError` comme type d'erreur retourné, est aussi une très bonne conception.

* Si l'auteur de la fonction n'a vraiment pas besoin que l'appelant se soucie du type d'erreur concret, il suffit de retourner uniformément `Result<R, *Any>`. Tout type d'erreur, une fois boxé, peut se convertir en `*Any`.
L'appelant peut, selon ses besoins, effectuer un downcast et ne traiter que les types d'erreur qui l'intéressent.

* Si l'on souhaite que l'appelant puisse obtenir la pile d'appels au moment où l'erreur s'est produite, on peut envelopper le type `BackTrace` de la bibliothèque standard dans un type personnalisé, et l'utiliser comme type d'erreur.
Le type `BackTrace` peut décrire la pile d'appels courante ; il suffit de le transmettre vers l'extérieur comme partie de l'information d'erreur.

#### Opérateur `?`

L'opérateur `?` est un opérateur postfixe, utilisé ainsi : `expr?`.

L'opérateur `?` est principalement destiné à être utilisé avec le type `Result`. L'opérateur `?` exige que l'expression qui le précède soit de type `Result`.

> **Fonctionnalité prévue** : la surcharge de l'opérateur `?` sera supportée ultérieurement, afin qu'il puisse s'appliquer à des types personnalisés.

La sémantique d'exécution de l'instruction `let x = expr?;` est :

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

Parce qu'il provoque un retour anticipé de toute la fonction, il impose une exigence sur le type de retour de cette fonction.
Si le type de `expr` est `Result<R1, E1>` et que le type de retour de la fonction est `Result<R2, E2>`, alors on exige que `E2` puisse se convertir implicitement de façon naturelle en `E1`, ou que `E2` implémente le trait `From<E1>`.
Ainsi le compilateur est capable de convertir `E1` en `E2` pour le retour.

### Expressions entre parenthèses

`unsafe(<expr>)` est autorisé : l'expression `<expr>` se trouve alors dans un contexte unsafe. Et la valeur de cette expression est égale à la valeur de l'expression `<expr>`.

`const(<expr>)` est autorisé : l'expression `<expr>` se trouve alors dans un contexte const. Et cette expression doit être évaluée à la compilation.

### Expressions d'indexation

### Expressions d'appel de fonction

Appel de fonction

Appel de méthode

### Expressions d'accès aux membres

Membres de struct

Membres de tuple

### Expressions de fermeture

La syntaxe des fermetures de Zinc diffère de celle de Rust. Lors de la conception de la syntaxe des fermetures, Zinc a principalement tenu compte des points suivants :
1. La syntaxe des fermetures doit convenir au sucre syntaxique d'« appel trailing ». La syntaxe d'appel trailing permet, dans de nombreux cas, de concevoir des DSL très esthétiques. Des fonctions comme lock / spawn / lazy, par exemple, se prêtent très bien à la syntaxe de trailing closure.
2. Une fermeture doit avoir à la fois une écriture complète et une écriture abrégée
3. La syntaxe complète des fermetures doit être complète fonctionnellement : liste de captures explicite, déclaration de génériques, liste de paramètres complète, type de retour explicite
4. L'écriture abrégée doit être suffisamment concise, en permettant d'omettre tous les éléments syntaxiques inutiles, tout en restant similaire à l'écriture complète

Exemple de syntaxe complète de fermeture :
```
.{
    async [move x, weak y, ref z]<T>(arg: T) -> ReturnTy where T: Constrait
    =>
    statements(x, arg);
    return y;
}
```

La syntaxe complète d'une fermeture commence par `.{` et se termine par `}`, ce qui facilite la combinaison avec la syntaxe d'« appel trailing ». Le contenu interne comprend :
1. Une liste de qualifiers, notamment `async` `const`, etc.
2. Une liste de captures explicite entre crochets ; les modes de capture supportés sont `move` `ref` `ref mut` `weak`
3. Une liste de paramètres génériques entre chevrons
4. Une liste de paramètres de fonction entre parenthèses
5. Un type de retour explicite après `->`
6. Une clause where optionnelle
7. Le corps de la fermeture après `=>`

Exemple de l'écriture la plus abrégée :
```
(x) => x + x
```
Entre les parenthèses se trouve la liste des paramètres ; le type des paramètres peut être omis. Après `=>` se trouve le corps de la fermeture ; en l'absence d'accolades, ce corps ne peut être qu'une expression, et ne peut pas contenir d'instructions.




### Expressions de branchement

if cond then expr else expr

### Expressions yield/await



### Expressions is

Le mot-clé `is` remplace `if-let` `while-let` `matches!`

### Expressions as


### Expressions constantes


## Instructions

Une instruction se termine généralement par un point-virgule. Les instructions vides sont autorisées.

### Blocs d'instructions block

Des accolades contenant plusieurs instructions. Un bloc d'instructions introduit un nouveau scope ; les variables locales du scope intérieur peuvent avoir le même nom que celles du scope extérieur.

Un block vide est autorisé.

### Blocs d'instructions unsafe

Un bloc d'instructions block décoré par `unsafe`.

### Instructions let

```
let <pattern> = <expr> ;
```

> **TODO** : actuellement, la déclaration d'une variable doit être initialisée, pour simplifier l'implémentation et éviter de vérifier via le CFG qu'une variable est initialisée avant usage. Cette exigence pourra être assouplie ultérieurement : il suffira de vérifier que l'initialisation a bien eu lieu avant toute lecture.

### Instructions d'expression

```
<expr> ;
```

### Instructions if

```rust
fn test(cond: Bool) {
    if cond {
        println("then branch");
    } else {
        println("else branch");
    }
}
```

### Instructions match


### Instructions de boucle

#### `while`

1. La condition après while doit être une expression de type Bool
2. Il n'est pas nécessaire d'écrire un point-virgule après les accolades du while

#### `do-while`

1. La condition après while doit être une expression de type Bool
2. Un do-while doit se terminer par un point-virgule

#### `for`

`for  <pattern> in <expr>  { <statements> }`

1. for supporte le filtrage par motif
2. expr doit être un type satisfaisant le trait `Iterable` ou `IterableMut`, ou un pointeur vers un tel type.

Il y a trois façons de désucrer une instruction `for-in`, selon l'écriture de `<pattern>` :

1. pattern est un identifier : `for i in v { <body> }`, alors le type de `i` est le type `Iterable::Item` de `v`. Désucré en :

    ```
    {
        let mut iter = v.iter();
        while (iter.next() is Some(p)) { // p est un borrow en lecture seule vers Item
            let i = *p;                // i est un nouvel Item copié depuis le borrow en lecture seule
            <body>
        }
    }
    ```

2. pattern est ref identifier : `for ref i in v { <body> }`, alors le type de `i` est le type `&Iterable::Item` de `v`. Désucré en :

    ```
    {
        let mut iter = v.iter();
        while (iter.next() is Some(i)) { // i est un borrow en lecture seule vers Item
            <body>
        }
    }
    ```

3. pattern est ref mut identifier : `for ref mut i in v { <body> }`, alors le type de `i` est le type `&mut Iterable::Item` de `v`. Désucré en :

    ```
    {
        let mut iter = v.iter_mut();
        while (iter.next() is Some(i)) { // i est un borrow en lecture-écriture vers Item
            <body>
        }
    }
    ```

### Instructions de saut

#### `return`

1. return peut être suivi d'une expression ou non.
2. Le type de l'expression après return doit être compatible avec le type de retour de la fonction.

#### `break`

1. break ne peut être utilisé que dans un bloc de boucle. Y compris `while` `do-while` `for`.

#### `continue`

1. continue ne peut être utilisé que dans un bloc de boucle. Y compris `while` `do-while` `for`.

#### `panic`

Le mot-clé panic peut être comparé au mot-clé throw de langages comme C++/Java. Mais Zinc n'a pas de syntaxe try-catch, et ne fournit pas non plus de fonction `catch_unwind`.

On ne devrait utiliser panic que lorsqu'une erreur irrécupérable se produit. Le mot-clé panic peut être suivi d'une expression de type chaîne. Exemple :

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

`goto` est un mot-clé réservé, pas encore implémenté. Il est prévu pour plusieurs objectifs.

1. En combinaison avec une instruction label, réaliser une fonctionnalité de saut. Remplacer l'instruction break label de Rust. Principalement parce que les cas d'usage de break label sont limités : il ne peut être utilisé que dans un bloc de boucle. Mais goto label a, en C, un cas d'usage plus courant : regrouper le traitement des erreurs à la fin de la fonction, et, ailleurs dans la fonction, goto vers cet endroit pour un traitement unifié. C'est extrêmement commode et lisible. Utiliser goto dans ce cas est tout à fait raisonnable. Bien sûr, pour éviter les abus, nous devrions imposer quelques restrictions aux instructions goto :
    1. goto est une instruction, il ne peut pas être utilisé à l'intérieur d'une expression.
    2. On ne peut pas sauter d'une fonction ou d'un lambda à un autre.
    3. On ne peut que sortir d'un block, pas y entrer. Le corps d'une boucle et les différentes branches d'un match sont aussi des blocks distincts.
    4. On ne peut sauter que vers l'avant, pas vers l'arrière.
    5. Il faut garantir que, sur le chemin du goto, toutes les variables sont initialisées avant d'être utilisées.

    En résumé, je souhaite fournir une instruction goto sûre et fiable. Pouvoir réaliser le style de programmation courant en C — traitement unifié des erreurs en fin de fonction — sans introduire d'autres risques.

2. Préparer la fonctionnalité de « récursion terminale déterministe ». Remplacer le mot-clé `become` de Rust.
    1. Autoriser `goto call_fn();` pour réaliser une récursion terminale déterministe.


## Fonctions

Une définition de fonction commence par le mot-clé `fn`, suivi du nom de la fonction et de la liste des paramètres, entourée de parenthèses, puis d'un type de retour optionnel, enfin du corps de la fonction entouré d'accolades.

```rust
// définition de fonction
fn function_name(arg1: Int, arg2: Char) -> Bool {
    println("function body");
    return true;
}

fn main() {
    function_name(0, 'X'); // appel de fonction
}
```

Omettre le type de retour signifie retourner le type `()`.

Les paramètres supportent les paramètres nommés ; un paramètre nommé commence par un point. À l'appel, si l'on utilise un paramètre nommé, on commence aussi par un point. La syntaxe du point vise à rester cohérente avec celle de l'expression d'initialisation de structure.
Les paramètres nommés doivent se trouver après les paramètres non nommés.

```
fn run(.from: &Str, .to: &Str) {
    println("run from {} to {}", from, to);
}

fn main() {
    run("home", "bar");  // on peut appeler dans l'ordre des paramètres
    run(.to = "bar", .from = "home"); // on peut aussi appeler par nom, l'ordre n'importe alors pas.
}
```

Puisque les fonctions supportent les paramètres nommés, supporter en plus le filtrage par motif sur les paramètres serait un peu complexe. De plus, à la position des paramètres de fonction, la plupart des motifs ne servent pas à grand-chose ; les paramètres de fonction ne supportent donc pas le filtrage par motif.
En Rust, le motif le plus utile supporté dans la liste des paramètres est le motif `mut`, qui permet de modifier le paramètre lui-même. En Zinc, il est recommandé de contourner cela ainsi :
```
fn f(v: Vec<Int>) {
    let mut v = move v; // le v des paramètres n'est pas décoré par mut ; on peut redéfinir à l'intérieur de la fonction une variable mut du même nom, et y move le paramètre.
    // ...
    v.push(2); // il faut obtenir un borrow de type &mut de v, ce qui exige que la variable v soit décorée par mut
    // ...
}
```

Les paramètres supportent une valeur par défaut, qui doit être une expression constante. Un paramètre avec valeur par défaut doit se trouver après les paramètres sans valeur par défaut.

```
fn increase(value: Int = 1) {}

fn main() {
    increase();  // la valeur de l'argument est 1
    increase(2); // la valeur de l'argument est 2
}
```

Les trailing closures sont supportées. La signification de la syntaxe d'appel trailing closure est : si, à la définition de la fonction, le type du dernier paramètre est un type fn/Fn/FnMut, ou un pointeur vers l'un de ces types, alors à l'appel on peut écrire le lambda à l'extérieur de la liste des paramètres.
S'il n'y a aucun autre paramètre en dehors de ce paramètre de type fonction, l'appelant peut omettre les parenthèses requises pour la liste des paramètres.
Exemple :

```
// le dernier paramètre de la fonction lock est un type fonction
fn lock(f: &Fn()) {
    println("lock");
    f();
    println("unlock");
}

fn main() {
    lock( ()=>println("smth") ); // OK, lambda en syntaxe simple comme argument
    lock( .{ () => println("smth"); } ); // OK, lambda en syntaxe complète comme argument

    // syntaxe d'appel trailing closure : écrire la fermeture à l'extérieur de la liste des paramètres
    lock().{
        () => println("smth");
    };
    // syntaxe d'appel trailing closure : s'il n'y a pas d'autre paramètre, omettre les parenthèses de la liste des paramètres
    lock.{
        () => println("smth");
    };
    // ultérieurement, on pourra encore simplifier la syntaxe du lambda, en autorisant l'omission du symbole =>
    lock.{
        println("smth");
    };
}
```

La surcharge de fonctions est supportée. Mais le nombre de paramètres doit être différent.

```
fn f1(arg1: Int, arg2: Int = 1) {} // peut accepter 1 ou 2 arguments.
fn f1(arg: String) {} // erreur, ne peut pas former une surcharge avec le f1 ci-dessus. Cette version peut accepter 1 argument, la version ci-dessus aussi. Conflit.
```

todo: les paramètres variadiques ne sont pas encore supportés, à améliorer ultérieurement. En général, il est recommandé d'utiliser la surcharge de fonctions à la place ; lorsque le nombre de paramètres est trop élevé, utiliser des tableaux et des slices.

On peut ajouter des fonctions membres à un type via un bloc `impl`.
Lorsque le nom du premier paramètre est le mot-clé `self`, on peut appeler la fonction avec la syntaxe du point.

```rust
impl User {
    fn send_email(&self) {}
}
fn main() {
    let u: User = { ... };
    u.send_email();
}
```

Le paramètre `self` est spécial ; il a plusieurs écritures simplifiées :
* `self` représente `self: Self`
* `&self` représente `self: &Self`
* `&mut self` représente `self: &mut Self`
* `*self` représente `self: *Self`
* `*weak self` représente `self: *weak Self`

Les fonctions peuvent être utilisées comme citoyens de première classe. Les types pointeurs de fonction sont supportés.

todo: introduire une nouvelle syntaxe dans la signature de fonction, indiquant que le compilateur est autorisé à insérer automatiquement une conversion auto-ref/auto-deref de l'argument réel vers le paramètre formel. Remplacer le `AsRef trait / AsMut trait` de Rust
