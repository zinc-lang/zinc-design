# Génériques et trait

## Génériques

Les définitions de types et les définitions de fonctions peuvent toutes deux déclarer des paramètres génériques.

### Types génériques

Un type défini par l'utilisateur peut porter des paramètres génériques.
Il y a deux sortes de paramètres génériques : les paramètres génériques de lifetime, et les paramètres génériques de type. Dans la liste des paramètres génériques, l'ordre est : les paramètres de lifetime avant les paramètres de type.

todo: il faudra ultérieurement examiner s'il convient d'ajouter les const generics ; il faut noter les raisons pour lesquelles Swift ne supporte pas cette fonctionnalité

```rust
struct S<'a, T> {
    p: &'a T
}

enum E<T> {
    A(T), B(Int)
}
```

La définition d'un type supporte une clause `where` optionnelle. Une clause `where` peut exprimer : la relation de survie entre lifetimes, ainsi que le fait qu'un type satisfait ou non une contrainte de trait.

L'utilisation d'un type générique nécessite de fournir des arguments génériques.

```rust
fn main() {
    let v = 1;
    let x: S<Int> = S { .p = &v };
}
```

### Fonctions génériques

Les fonctions supportent aussi les génériques. Comme pour les types génériques, les paramètres génériques supportent les paramètres génériques de lifetime et les paramètres génériques de type.
Les fonctions génériques supportent aussi une clause `where` optionnelle.

```
fn swap<'a, T>(lhs: &'a mut T, rhs: &'a mut T) {}
```

Attention : en Zinc, la syntaxe d'appel des fonctions génériques a changé, pour éviter la syntaxe turbo fish de Rust.

<details>

<summary>Qu'est-ce que la syntaxe turbo fish</summary>

<div style="border: 1px solid black; padding: 10px;">

Dans la conception syntaxique de Rust, si l'appel d'une fonction générique utilisait directement la même syntaxe que la définition, il y aurait une ambiguïté syntaxique, comme dans l'exemple suivant :
```rust
fn main() {
    let (the, guardian, stands, resolute) = ("the", "Turbofish", "remains", "undefeated");
    let _: (Bool, Bool) = (the<guardian, stands>(resolute)); // que signifie cette ligne ?
}
```

Le cœur de cette ambiguïté est que l'opérateur inférieur et les chevrons génériques réutilisent le même symbole, ce qui donne deux lectures possibles de la ligne ci-dessus :
1. C'est un appel de fonction générique à l'intérieur de parenthèses, le nom de la fonction est `the`, les arguments génériques sont `guardian` `stands`, l'argument de fonction est `resolute`
2. C'est un tuple de deux éléments. Le premier élément est la comparaison `the<guardian` ; le second est la comparaison `stands>(resolute)`.

Pour résoudre ce conflit de syntaxe, Rust exige que, à l'appel d'une fonction générique, on fasse suivre le nom de la fonction de `::`, donc la syntaxe d'appel correcte est :
```
the::<guardian, stands>(resolute)
```

C'est la syntaxe turbo fish. Par exemple `Vec::<i32>::with_capacity(16)`.

</div>
</details>

<br/>

La syntaxe turbo fish de Rust élimine le problème d'ambiguïté de l'analyse syntaxique, mais elle introduit une incohérence syntaxique, peu esthétique.
Zinc a adopté une autre conception, qui évite à la fois le problème d'ambiguïté syntaxique et préserve la cohérence de la syntaxe.

1. Dans la fully-qualified-call-syntax, si un type générique apparaît, il faut systématiquement commencer par le mot-clé `type`. Exemples :
    * `(type Vec<Int>)::with_capacity(4)`  
      Parce que `Vec` porte explicitement un argument générique, `Vec<Int>::with_capacity(4)` est une erreur de syntaxe.
    * `Vec::with_capacity(4)`  
      On peut ne pas spécifier le paramètre générique de `Vec` ; le paramètre générique peut être inféré depuis le contexte, et l'on peut alors omettre le préfixe type.
    * `(type Vec<String> as Default)::default()`  
      On peut spécifier explicitement le nom du type ainsi que le nom du trait correspondant, pour obtenir une syntaxe pleinement qualifiée d'appel d'une fonction particulière. Cela signifie que des fonctions membres de traits différents peuvent avoir le même nom, et être implémentées pour le même type, sans conflit.

2. L'initialisation de structure et le filtrage par motif adoptent systématiquement une syntaxe de type postfixe. Exemples :
    * Initialisation de structure : `let v = { .x = 1 } : S<Int>;`
    * Filtrage par motif de structure : `expr is { .x = 1 } : S<Int>`

3. À l'appel d'une fonction générique, la liste des arguments génériques doit s'écrire à l'intérieur des parenthèses. Exemples :
    * `std::mem::size_of(<Int>)`

La logique centrale de ces modifications est de faire apparaître tous les noms de types portant des génériques exclusivement dans des contextes préfixés par un identifiant spécifique.
Ainsi, lors de l'analyse syntaxique, lorsque le compilateur rencontre le symbole ou le mot-clé correspondant, il sait que ce qui suit est forcément un type et non une expression, et que les symboles `<` `>` qui y apparaissent doivent être compris comme des chevrons et non comme inférieur et supérieur.

Détaillons maintenant la syntaxe d'appel des fonctions génériques :
```
// la syntaxe de définition de fonction n'a pas changé
fn f<T>(arg: T) {}

// la syntaxe d'appel de fonction a changé
fn main() {
    f(<Int>, 1); // spécifier explicitement le type de l'argument générique
    f(1); // écriture par inférence de type, en omettant l'argument générique
}
```

Les principales raisons de cette conception syntaxique sont, d'une part, d'éviter une syntaxe aussi disgracieuse que le turbo fish, et d'autre part de correspondre à la façon dont Zinc implémente les fonctions génériques.

Sans tenir compte des optimisations du compilateur dans des cas particuliers, une fonction générique Zinc, du point de vue de l'implémentation, est par défaut réalisée par passage de table. Le compilateur traduit la fonction f ci-dessus en :
```
// pseudo-code C correspondant à la définition de la fonction f :
void f(ZnTypeMeta * typeof_T, void * arg) { }
```

Lors d'un appel de fonction comme `f(<Int>, 1)`, l'appelant récupère vraiment le ZnTypeMeta correspondant à `Int` et le transmet comme argument de fonction à f.
Écrire la liste des arguments génériques de la fonction à l'intérieur des parenthèses correspond donc mieux à la sémantique d'exécution des fonctions génériques de Zinc.

Un type pointeur de fonction Zinc peut contenir des génériques : `let pf: fn<T>(T)->() = f;`.
Zinc n'autorise pas le currying des fonctions ; écrire `let pf: fn(Int)->() = f(<Int>);` échoue à la compilation.
Pour ce cas d'instanciation de fonction, veuillez utiliser un lambda ; cette écriture est valide : `let pf: fn(Int)->() = .{ (arg: Int)->() => f(<Int>, arg); };` .

Le principal avantage de cette implémentation des fonctions génériques est : une fonction générique peut aussi être distribuée via une bibliothèque dynamique. C'est un avantage de conception de Swift.

Cela est très utile dans le scénario suivant :
Supposons que nous ayons conçu une bibliothèque dynamique `a.so`, qui expose une interface de fonction générique, utilisée par l'application `b.exe`.
Nous pouvons alors, lors de la mise à jour de l'implémentation interne de `a.so` (sans modification destructive de l'API, bien sûr), ne pas recompiler `b.exe`, et le programme continue de fonctionner normalement.
Si les génériques étaient implémentés par instanciation, ce scénario serait très difficile à supporter. Mettre à jour une bibliothèque amont obligerait l'application en aval à être recompilée et redéployée.

Outre cela, cette conception apporte quelques autres avantages :
1. Support des fonctions `virtual` portant des génériques. Pour les fonctions `virtual`, voir les sections suivantes.
2. Réduction du code size, réduction du temps de compilation.

Le coût, bien sûr, est une baisse de l'efficacité d'exécution. Cela bloque de nombreuses optimisations du compilateur, n'est pas favorable au cache, et nuit au temps d'exécution.
Mais on peut encore, au niveau de l'implémentation, dans certains scénarios particuliers, via une option de compilation supplémentaire, autoriser l'instanciation des génériques comme moyen d'optimisation des performances, pour un meilleur équilibre entre code size et efficacité d'exécution.
1. Lors de la compilation d'un exécutable, et non d'une bibliothèque, on peut tout à fait implémenter les génériques par instanciation
2. Même lors de la compilation d'une bibliothèque, dès lors qu'il n'y a pas besoin de distribuer et déployer séparément une bibliothèque dynamique, on peut implémenter les génériques par instanciation

> **TODO** : actuellement, un type UnSized n'est pas autorisé comme argument générique, et un trait non plus.

> **Fonctionnalité prévue** : on pourra ultérieurement supporter une condition « ou » comme condition de clause where, comme sucre syntaxique de surcharge du type des paramètres de fonction :
```
fn f<T>(arg: T) where T: Int32 | UInt32 {}
// équivalent à :
trait AnonymousTR {}
impl AnonymousTR for Int32 {}
impl AnonymousTR for UInt32 {}
fn f<T>(arg: T) where T: AnonymousTR {}
```

## trait

Exemple de syntaxe de définition d'un trait :

```
trait TrName<T> : SuperTrait1 + SuperTrait2 where T: Condition {
    type AssocType;

    virtual fn method(&self);

    fn static_method() -> Self;
}
```

La définition d'un trait supporte :
1. Des paramètres génériques optionnels
2. Des traits parents optionnels, plusieurs traits parents sont supportés, séparés par `+`
3. Une clause de condition where optionnelle

Le corps d'une définition de trait peut comprendre :
1. Des types associés
2. Des constantes associées
3. Des fonctions associées

### impl trait

Un trait peut servir à unifier l'abstraction de différents types. La syntaxe pour spécifier qu'un type concret `SomeType` implémente un trait `TrName` est la suivante :

```
impl TrName for SomeType {

}
```

Bloc impl
1. Les fonctions internes ne peuvent pas spécifier pub séparément
2. Ni spécifier virtual

`impl Trait for Trait` n'est pas autorisé.

**Règle de l'orphelin**

Le principe de base est :

### Pointeurs vers un trait

Les deux cas d'usage d'un trait sont :
1. Comme borne supérieure d'une contrainte générique
2. Comme type pointeur vers un trait

Les variables globales, les variables locales et les membres ne peuvent pas utiliser directement un trait comme type. Mais on peut utiliser un pointeur vers un trait comme type.

Si le type `S` implémente `trait R`, alors un pointeur vers `S` peut être upcasté en pointeur vers `R`. Exemple :

```rust
struct S {}
trait R {
    virtual fn f(&self);
}
impl R for S {
    fn f(&self) {}
}

fn test(p: *S) {
    let p1: *R = p; // ok, upcast
    p1.f();
}
```

### Fonctions virtual

Un pointeur vers un trait peut appeler les fonctions membres du trait. Mais il y a une restriction : seules les fonctions membres marquées virtual peuvent être appelées via un pointeur vers un trait.

> Note :
> Rust autorise l'utilisation de la syntaxe `where Self: Sized` pour marquer une fonction membre d'un trait comme « non-virtual ». Cette syntaxe est très étrange et difficile à comprendre. Zinc introduit directement un mot-clé pour ce marquage, ce qui est plus lisible.
>

Exemple :

```rust
trait TR {
    virtual fn f1(&self);
    fn f2(&self);
}

fn test(p: &TR) {
    p.f1(); // OK
    p.f2(); // erreur de compilation, f2 n'est pas une fonction virtual, elle ne peut pas être appelée via un pointeur vers un trait
}

fn test2<T>(arg: T) where T: TR {
    arg.f2(); // OK. Une fonction non virtual peut toujours être appelée sous forme générique.
}
```

Par ailleurs, attention : ce n'est pas toutes les fonctions écrites dans un trait qui peuvent être décorées par virtual. Une fonction pouvant être décorée par virtual doit satisfaire les exigences suivantes :
1. Le nom du premier paramètre doit être `self`, et son type un type pointeur vers `Self`. Y compris `&self` `&mut self` `*self` `*weak self`.
2. Hormis le premier paramètre, les autres paramètres ne peuvent pas utiliser le type `Self`, ni un type associé
3. Le type de retour de la fonction ne peut pas utiliser le type `Self`, ni un type associé

Exemple :
```
trait TR {
    virtual fn clone()->Self; // erreur de compilation, ne peut pas être décorée par virtual. Pas de paramètre self, le type de retour utilise Self.
    virtual fn g(arg: Int); // erreur de compilation, ne peut pas être décorée par virtual. Pas de paramètre self.
    virtual fn h(self);  // erreur de compilation, ne peut pas être décorée par virtual. Le paramètre self n'est pas un type pointeur.
}
```

**Attention** : une fonction virtual peut introduire de nouveaux paramètres génériques. Exemple :
```
trait TR {
    virtual fn f<T>(&self, arg: T) where T: Hash + Ord + Default; // OK
}
```

> **Fonctionnalité prévue** : on pourra ultérieurement supporter l'écriture `* Trait1 + Trait2 + 'a` :
1. Avec `+`, on ne peut pas avoir de type concret, seulement un trait ajouté à un trait
2. Il ne peut y avoir qu'un seul type non-marker trait. S'ajouter soi-même n'est pas autorisé non plus.
3. `* Trait1 + Trait2` peut se convertir implicitement en `*Trait1` ou `*Trait2`

Cette fonctionnalité peut être indispensable dans certains scénarios, par exemple si l'on veut `* MyTrait + Sync` pour exprimer que le type pointé satisfait non seulement la contrainte MyTrait, mais aussi la contrainte Sync.
On ne peut pas l'exprimer autrement.

### Types associés

Un type associé est un placeholder de type défini dans un trait ; à l'implémentation du trait, il faut spécifier le type concret.

Exemple :
```rust
trait Container {
    type Item;
    
    fn add(&mut self, item: Self::Item);
    fn get(&self) -> Option<Self::Item>;
}

struct IntContainer {
    items: Vec<Int>,
}

impl Container for IntContainer {
    type Item = Int;
    
    fn add(&mut self, item: Int) {
        self.items.push(item);
    }
    
    fn get(&self) -> Option<Int> {
        self.items.last().copied()
    }
}
```

Le type `Self` représente le type concret qui implémente le trait.

> **TODO** : les Higher-Kinded Types ne sont actuellement pas supportés.

### Héritage de trait

En Zinc, les traits supportent l'héritage. On peut, à la définition d'un trait, spécifier zéro ou plusieurs traits parents.

Pour des traits en relation d'héritage, un pointeur vers le trait enfant peut se convertir implicitement en pointeur vers le trait parent (pointeurs ARC / BORROW tous deux). Exemple :
```
trait TR: TR1 + TR2 + TR3 {}

fn test(p: *TR) {
    let p1: *TR1 = p; // OK
    let p2: *TR2 = p; // OK
    let p3: *TR3 = p; // OK
}
```

Un pointeur vers un trait parent peut aussi, via le filtrage par motif, être downcasté en pointeur vers un trait enfant. Par exemple `is` et `match` peuvent le faire.

```
trait TR: TR1 + TR2 + TR3 {}

fn test(p: *TR1) {
    if (p is p1: *TR) { // tenter un downcast, qui peut échouer
        // utiliser la variable p1
    }

    match (p) {
        p2: *TR => {}
        _ => {}
    }
}
```

Autres fonctionnalités :
1. Dans un trait enfant, on peut spécifier la valeur d'un associated type / const
2. Dans un trait enfant, on peut override le corps de fonction d'un trait parent
3. L'héritage multiple est supporté ; il faut considérer le problème de conflit de noms de fonctions en héritage multiple
    1. En héritage multiple, les conflits de noms sont interdits
    2. En héritage multiple, ajouter un type parent ou ajuster l'ordre d'héritage affecte l'ABI
    3. `trait TR1 : TR2 {}`  et `trait TR1 where Self: TR2 {}` n'ont pas la même signification.

### reuse

Le but du mécanisme reuse est de réutiliser des membres et des fonctions membres. reuse est un sucre syntaxique, qui permet à l'utilisateur de déléguer une implémentation à un autre type. Exemple :

```
struct S1 { }
impl TR1 for S1 {
    fn f1(&self, arg: Int) {}
}

struct S2 {
    base: S1  // le nom du membre n'a pas de restriction, n'importe lequel convient
}
impl TR1 for S2 {
    reuse self.base;  // cela signifie que les fonctions membres de ce bloc impl réutilisent toutes les fonctions membres du même trait implémentées par self.base
}

// le code ci-dessus est équivalent à :
impl TR1 for S2 {
    // implémenter toutes les fonctions membres, et chaque corps de fonction appelle la fonction membre correspondante de `self.base`.
    fn f1(&self, arg: Int) {
        self.base.f1(arg);
    }
}

```

### Traits intégrés courants

`Any` trait

`Fn` trait

`Send` trait

`Sync` trait

### Paradigme de programmation orientée objet

Par la combinaison des fonctionnalités de langage ci-dessus, Zinc peut aussi supporter le paradigme de programmation orientée objet, mais d'un style différent de celui habituel dans l'industrie (comme C++/Java/C#/Swift).
La caractéristique principale de Zinc est : pas de support de `class` ni d'héritage de class. Seuls les traits peuvent hériter, et seuls les pointeurs vers un trait peuvent faire de la répartition dynamique.

Voici, sous plusieurs angles, pourquoi cette conception.

1. Les types `class` à sémantique de référence ont une disposition mémoire trop peu flexible

    Du point de vue de la disposition mémoire, on peut voir une class à sémantique de référence d'un autre langage comme la combinaison d'un `pointeur ARC + structure` en Zinc. On n'autorise que l'usage de la sémantique de référence, pas de la structure à sémantique de valeur correspondante.
    Cela signifie que, lorsqu'une variable de type class est utilisée comme variable locale ou comme membre, il y a toujours une allocation dynamique, et toujours une couche supplémentaire d'indirection de pointeur. C'est en réalité une régression du pouvoir d'expression, et non un renforcement.
    Alors que si l'on fournit séparément à l'utilisateur le `pointeur ARC` et la `structure`, par exemple `Vec<S>` et `Vec<*S>` sont des types différents, utilisables à la demande selon le scénario, c'est plus flexible, l'utilisateur a davantage de liberté.

    Une class à sémantique de référence, dans tous les scénarios, porte à l'intérieur de chaque objet un pointeur « en-tête d'objet ». Cette conception de Zinc, elle, n'affecte pas la disposition mémoire de l'objet lui-même.

2. Ajouter un type `class` à sémantique de référence ferait apparaître des incohérences dans le système de types de Zinc, de nombreux scénarios nécessiteraient des règles spéciales, l'implémentation serait complexe

    Une class à sémantique de référence apporterait au système de types de Zinc toute une série de règles supplémentaires, ce n'est pas rentable. Surtout lorsque divers pointeurs sont utilisés avec une class.
    En Zinc, il existe déjà plusieurs types de pointeurs, et une « class à sémantique de référence » peut elle-même se comprendre comme une combinaison « pointeur implicite + structure ».
    Cette fonctionnalité n'est pas orthogonale aux autres fonctionnalités du langage, car ce « pointeur implicite » devrait-il être ARC, weak ou borrow : l'utilisateur n'a pas le choix.

    Considérons ce scénario : lorsque `T` est une class, que signifie `&T`. Ce borrow pointe-t-il vers la variable pointeur elle-même, ou vers l'adresse de début de l'objet ?
    Parce qu'une class lie de force le `pointeur ARC` et la `structure`, quel que soit le schéma de conception choisi, cela apportera des difficultés.
    Séparer le pointeur et la structure rend l'expression sémantique plus claire. Le type `&S` est un borrow de `S` ; le type `&*S` est un borrow de `*S`. Sémantique claire, implémentation simple aussi.

    Considérons l'exemple générique suivant ; on constate qu'unifier dans les génériques une struct à sémantique de valeur et une class à sémantique de référence est très pénible :
    ```
    fn test<T>(arg: T) {
        // si nous introduisons une class à sémantique de référence, alors l'expression de prise de borrow et l'expression de déréférencement se comportent différemment pour un type à sémantique de valeur et un type à sémantique de référence.
        // dans un scénario générique, il faudrait alors faire à l'exécution des tests supplémentaires pour unifier ces deux cas par les génériques.
        let p: &T = &arg;
        let c: T = *p;
    }
    ```

3. Et si, comme en C++, nous introduisions class et héritage, mais que class n'avait pas de sémantique de référence, est-ce que cela irait ?

    En C++, la différence sémantique entre class et struct n'est pas grande. Les types valeur sont supportés, sans liaison avec un pointeur.
    Mais l'héritage de class signifie que le sous-type doit accepter toutes les interfaces implémentées par le type parent, ce qui pose problème.

    Dans la conception traditionnelle de l'héritage, une class enfant implémente forcément toutes les interfaces (interface/protocol) de la class parent.

    Considérons un trait comme `Send` en Zinc, qui exige que tous les membres satisfassent `Send`. Si nous introduisions l'héritage, on aurait ce cas : le type parent implémente `Send`,
    le sous-type hérite du type parent, mais ajoute un membre non-`Send` ; nous souhaitons que le sous-type réutilise tous les membres et toutes les méthodes du type parent, sans implémenter `Send` : avec la conception traditionnelle de l'héritage, c'est impossible.

    Dans une conception qui utilise l'héritage, un sous-type ne peut pas ne choisir qu'une partie des interfaces du type parent. Pour atteindre l'objectif ci-dessus, il faudrait aussi abandonner l'héritage et passer à la composition. Donc l'héritage ne peut faciliter la réutilisation de code que dans certains scénarios, et ne s'adapte pas à tous les scénarios.

4. L'héritage de class nécessite de concevoir des règles spéciales pour des scénarios particuliers

    1. Un scénario typique est l'héritage multiple.
    Selon le schéma traditionnel qui supporte l'héritage, si nous supportons qu'une class hérite de plusieurs class, il faut alors considérer le scénario de l'« héritage en diamant ».
    C++ a conçu pour ce scénario des règles syntaxiques et sémantiques spéciales, et de nombreuses normes de programmation y imposent aussi de nombreuses contraintes.
    Java/C#/Swift, etc., ne supportent tout simplement pas l'héritage multiple, ce qui limite la capacité de réutilisation du code et simplifie les règles sémantiques.

    2. Un autre scénario typique est celui des constructeurs. Les règles des différents langages sont toutes différentes. En C++, appeler une fonction virtuelle dans un constructeur n'a pas l'effet d'une fonction virtuelle.
    Les constructeurs de Java/C# n'accèdent pas à des variables non initialisées, grâce à l'« initialisation à 0 » des champs.
    Swift ne peut pas supporter l'« initialisation à 0 », et souhaite que les constructeurs n'accèdent jamais à un membre non initialisé, donc ses règles sont les plus complexes.
    Un constructeur restreint le nom de la fonction, ce qui signifie que supporter les constructeurs oblige à supporter la surcharge de fonctions basée sur le type ; un constructeur restreint le type de retour, ce qui signifie qu'on ne peut pas faire de gestion d'erreurs via le type de retour.
    Ces idées de conception ne correspondent pas à Zinc.

    En résumé, les langages traditionnels ont supporté l'héritage de class, mais cette fonctionnalité s'accompagne de règles sémantiques spéciales supplémentaires, différentes d'un langage à l'autre, peu esthétiques.

Zinc a suivi l'approche de conception de Rust, en adoptant une conception plus concise : seul l'héritage de trait est autorisé.

La conception de Zinc est plus facile à comprendre et à utiliser pour l'utilisateur :
* Toute réutilisation de membres se fait systématiquement selon le modèle de la « composition » ;
* Toute réutilisation de méthodes membres se fait systématiquement via le « mécanisme reuse » ;
* Tous les cas où il faut unifier une interface publique pour plusieurs types se font systématiquement via « trait+impl » ;
* Tous les cas où il faut de la répartition dynamique se font systématiquement via un « pointeur vers un trait ».

Les règles ci-dessus s'appliquent à tous les types (types intégrés, enum, struct, tuple, chaînes, tableaux, etc.), sans cas particulier. Les règles ci-dessus suffisent pleinement à supporter le paradigme OOP. Le pouvoir d'expression n'est pas en cause : toutes les fonctionnalités supportées par les langages traditionnels peuvent s'exprimer, seulement dans certains cas la syntaxe est un peu plus verbeuse.

Les règles sémantiques sont concises, cohérentes et orthogonales.

### Disposition mémoire

Un pointeur vers un trait est un fat pointer. Les types pointeurs mentionnés dans cette section comprennent les six suivants : `*T`, `*weak T`, `&T`, `&mut T`, `*raw T`, `*raw mut T`.

```rust
trait TRBase { virtual fn f1(&self); }
trait TRSub : TRBase { virtual fn f2(&self); }

struct S { i: Int }
impl TRBase for S { fn f1(&self) {} }
impl TRSub for S { fn f2(&self) {} }

fn main() {
    let p1: *S = box S{.i = 1}; // p1 est un thin pointer
    let p2: *TRSub = p1; // p2 est un fat pointer
    let p3: *TRBase = p2; // upcast

    if p3 is psub: *TRSub {
        // downcast
    }
    if p2 is ps: *S {
        // downcast
    }
}
```

La disposition des fat pointers de Zinc diffère de celle de Rust. Un pointeur vers un trait contient 3 membres :
```
      p2
┌───────────────────┐
│ object_ptr        │→ pointe vers la tête de l'objet. L'objet peut être un type intégré, une struct définie par l'utilisateur, un enum, etc.
│───────────────────│
│ typemeta_ptr      │→ pointe vers le TypeMeta du type concret de l'objet, qui contient name, size, etc. du type, ainsi que les pointeurs de destructeur, de fonction de copie, etc.
│───────────────────│
│ vtable_ptr        │→ pointe vers la table de fonctions virtuelles de ce type concret pour ce trait, c'est-à-dire un tableau de pointeurs de fonction
└───────────────────┘
```

* Un pointeur vers un trait peut appeler les fonctions membres virtual du trait. À l'appel d'une fonction membre, on extrait le pointeur de fonction à la position correspondante de la vtable puis on l'appelle ; le décalage est déterminé à la compilation.

* Un pointeur vers un trait peut, via le filtrage par motif, se convertir en pointeur d'un autre type, car le fat pointer conserve l'information de type à l'exécution de l'objet. On peut déterminer à l'exécution vers quel type concret ce pointeur de trait pointe ; si la correspondance réussit, on extrait l'object_ptr qu'il contient et on l'utilise comme pointeur du type concret. Le downcast est aussi supporté : `*TRBase` peut déterminer à l'exécution s'il est de type `*TRSub`.

* Un pointeur vers un trait supporte l'upcast. Un type `*TRSub` peut se convertir en type `*TRBase`.

> Note :
> En Rust, un pointeur vers un trait s'appelle trait object ; c'est aussi un fat pointer, mais il ne supporte pas le downcast. Seul le trait spécial `Any` peut supporter le downcast, les autres traits ne le peuvent pas.
> Parce qu'il n'a que la taille de deux pointeurs, un peu gras, mais pas assez gras, insuffisant pour supporter tout le paradigme de programmation orientée objet que Zinc souhaite supporter.
>
> En s'inspirant de la disposition mémoire des Protocol de Swift, on peut concevoir qu'il suffit d'ajouter encore un pointeur. Les deux pointeurs vers des métadonnées conçus par Zinc pour les fat pointers peuvent s'apparenter à la value witness table et à la protocol witness table de Swift.
>

## Questions en suspens :

1. Interdire d'implémenter un trait pour un type pointeur poserait-il un problème de pouvoir d'expression ? (on prévoit ultérieurement d'introduire une syntaxe particulière dans les paramètres de fonction, équivalente à un AsRef/AsMut trait intégré au langage)

2. Interdire d'implémenter un trait pour un type quelconque poserait-il un problème de pouvoir d'expression ?
    ```
    impl<T> ToString for T where T: Display {} // cette écriture n'est actuellement pas supportée. Il est recommandé de la remplacer par l'héritage de trait.
    ```

3. Comment autoriser un UnSized type comme argument générique ? On ne peut actuellement pas implémenter un type comme `Mutex<File>` ; un `MutexFile` figé a une composabilité trop faible.

4. Faut-il supporter impl TR comme type de retour ?

5. Faut-il supporter les const generics ? Noter que Swift ne les supporte pas ; cela poserait probablement un défi important pour le runtime et l'ABI.

## Covariance et contravariance

|           |   'a   |   T   |   U   |
|-----------|--------|-------|-------|
| &'a T     |  covariant  |  covariant  |       |
| &'a mut T |  covariant  |  invariant  |       |
| *T        |        |  covariant  |       |
| Mutable<T>|        |  invariant  |       |
| Vec<T>    |        |  covariant  |       |
| fn(T)->U  |        |  contravariant  |  covariant  |
| *raw T    |        |  covariant  |       |
| *raw mut T|        |  invariant  |       |

On pourra ultérieurement continuer à supporter les fonctionnalités de covariance et de contravariance des types.
Les mots-clés `in` `out` sont déjà réservés.

Exemples de cas d'usage :
1. Conversion de `Option<*Sub>` en `Option<*Base>`
2. Conversion de `Result<*SubR, *SubE>` en `Result<*BaseR, *BaseE>`
3. Conversion de `(*Sub, *Sub)` en `(*Base, *Base)`
4. Conversion de `fn(*Base)->*Sub` en `fn(*Sub)->*Base`
