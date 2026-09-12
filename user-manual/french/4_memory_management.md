# Gestion de la mémoire

## Pointeurs ARC

Zinc a choisi le comptage de références comme stratégie centrale de gestion mémoire. ARC signifie automatic reference counting. Le sens de automatic est que, lors des opérations d'incrémentation et de décrémentation du comptage de références, le compilateur choisit automatiquement des instructions atomiques ou non atomiques.

Chaque bloc de mémoire alloué sur le tas porte deux compteurs de références : strong et weak.
Lorsque le compteur strong tombe à 0 :
* Si la valeur du compteur weak n'est pas 0, le destructeur de ce bloc de mémoire est appelé, mais le bloc n'est pas libéré. Ce n'est que lorsque le compteur weak tombe aussi à 0 que le bloc est libéré.
* Si la valeur du compteur weak est 0, le destructeur est appelé et le bloc est libéré directement.

Parce que le pointeur à comptage de références est le type de pointeur le plus courant, Zinc fait correspondre la syntaxe de pointeur la plus concise à la sémantique des pointeurs ARC. Le type `*T` représente un pointeur à comptage de références ARC vers un type `T`.

Le type pointeur weak s'écrit `*weak T`. Un pointeur weak vers un type `T` ne peut pas accéder aux membres ni aux fonctions membres de `T`, et ne peut pas être déréférencé.
Un pointeur weak doit être converti en pointeur strong `*T` via la fonction `std::ptr::upgrade` pour pouvoir accéder aux membres et aux fonctions membres de `T`.

Pour allouer un objet sur le tas, on utilise le mot-clé `box`. Pour une expression `box <expr>`, si le type de `<expr>` est `T`, alors le type de l'expression `box <expr>` est `*T`.
Si une expression `box` échoue à allouer de la mémoire, elle provoque directement un panic.
Si l'utilisateur a vraiment besoin de traiter manuellement une erreur d'allocation, il peut appeler lui-même la fonction `std::alloc::try_box`, et déterminer le succès de l'allocation d'après la valeur de retour.
Les cas où il faut traiter manuellement un échec d'allocation ne sont d'ailleurs pas très adaptés à Zinc : toute la bibliothèque standard provoque directement un panic en cas d'OOM. Se contenter de tester le succès de l'allocation au niveau applicatif ne suffit pas.


Exemple :
```rust
let x: MyObj = MyObj::new();  // si MyObj::new() retourne un type MyObj
let y: *MyObj = box MyObj::new(); // alors box MyObj::new() retourne un type *MyObj

let z = y; // z et y pointent vers le même objet de type MyObj
```

La disposition mémoire d'un pointeur ARC vers un type de taille statique (un type sized ; on ne discute pas pour l'instant des types unsized) ressemble à ceci. À chaque allocation dynamique, on fait d'abord `malloc(sizeof(UInt) + sizeof(MyObj))`, puis on décale le pointeur jusqu'à l'adresse de début de l'objet.
Pour accéder à un membre via un pointeur ARC, le décalage nécessaire est celui du membre par rapport à l'adresse de début de l'objet.
```
                ┌──────────────────────┐
                │ strong_count: UShort │
                │ weak_count: UShort   │
ptr: *MyObj  →  │──────────────────────│
                │ field1               │
                │ ......               │
                └──────────────────────┘
```

En cas de référence circulaire, il suffit qu'un des pointeurs du cycle soit un pointeur weak pour que le cycle puisse être correctement collecté sans fuite. Mais c'est au programmeur de spécifier lui-même quand utiliser un pointeur weak.

Zinc n'a pas le concept d'ownership de Rust, ni de type à sémantique similaire à `Box<T>`, ni de « type d'ownership » move-only.

**Attention** : il n'existe pas de type `*mut T`. Que la variable liée soit elle-même mut ou non, un type `*T` a toujours le droit de modifier ses membres.

Exemple :

```rust
struct S { m: i32, n: i32 }
fn main() {
    let p1: *S = box { .m = 1, .n = 2 }:S ;
    p1.m += 10; // OK, même si la liaison de variable p1 n'est pas décorée par mut, elle a le droit de modifier les membres

    p1 = box { .m = 10, .n = 20 }:S; // erreur, la liaison de variable p1 n'est pas décorée par mut, on ne peut pas lui affecter directement une valeur
}
```

## Sémantique de valeur et sémantique de référence

Présentation de la [sémantique de valeur et de la sémantique de référence](https://isocpp.org/wiki/faq/value-vs-ref-semantics#val-vs-ref-semantics) :
1. La sémantique de valeur signifie que, à la copie, la valeur est dupliquée : la nouvelle variable et l'ancienne n'ont aucun rapport. Une modification de l'une ne se reflète pas sur l'autre.
2. La sémantique de référence signifie que, à la copie, c'est le pointeur qui est copié : la nouvelle variable et l'ancienne partagent des données, et une modification de ces données partagées affecte les deux variables.

Le critère de jugement suit la logique de ce pseudo-code :
```rust
let mut y: T = x;
modify(&mut y); // modifier y
print(x);
print(y);
// Si une modification de y n'affecte en aucun cas x, on peut dire que le type T est un type à sémantique de valeur. Sinon, le type T est un type à sémantique de référence.
```

Zinc encourage l'utilisateur, lors de la définition d'un type, à le concevoir comme un type à « sémantique de valeur ».

Car si un type `S` est défini avec une sémantique de valeur, alors le type `*S` correspondant a une sémantique de référence, et l'utilisateur peut très facilement choisir d'utiliser `S` ou `*S` selon le cas.
Alors que si l'on conçoit directement le type avec une « sémantique de référence » à la définition, il est alors difficile d'obtenir le type à « sémantique de valeur » correspondant, ce qui est peu commode pour l'utilisateur.

## Borrow

Pourquoi avons-nous besoin de pointeurs borrow ?

Parce qu'avec seulement des pointeurs ARC/WEAK, le pouvoir d'expression est insuffisant.
Un type `*T` ne peut toujours pointer que vers la tête d'un « objet alloué dynamiquement ». Un objet alloué dynamiquement porte forcément un compteur de références. Car il doit opérer sur ce compteur, et garantir que le compteur se trouve à un décalage fixe par rapport à l'adresse qu'il pointe.
Si nous avons besoin d'un pointeur vers le milieu d'un objet, nous ne pouvons pas utiliser un pointeur de type `*T`.

Si tous les pointeurs sûrs du langage ne pouvaient pointer que vers la tête d'un objet, la disposition mémoire des objets ne pourrait pas être compacte. Cette conception ne favoriserait pas l'exploitation des types valeur.

Si nous autorisons la prise d'adresse d'une partie intermédiaire d'un objet, obtenant un nouveau pointeur, comme ci-dessous :
```
                ┌───────────────────┐
                │ strong_count: u32 │
                │ weak_count: u32   │
ptr: *MyObj  →  │───────────────────│
                │ field1: i32       │
                │ field2: MyStruct  │ ← borrow_ptr: &MyStruct
                └───────────────────┘
```

Alors ce pointeur borrow_ptr a forcément les caractéristiques suivantes :
1. À l'exécution, il n'a pas assez d'informations pour trouver le compteur de références porté par l'objet qu'il pointe
2. À l'exécution, il n'a pas la capacité de contrôler activement le lifetime de l'objet qu'il pointe

Par conséquent, le lifetime d'un pointeur borrow doit forcément être vérifié statiquement à la compilation, afin de garantir que le pointeur borrow vit forcément moins longtemps que l'objet emprunté, sinon on aurait un dangling pointer.

Zinc conserve donc aussi les pointeurs borrow de Rust, ainsi que les paramètres de lifetime. Un pointeur borrow ne peut s'obtenir que par une opération de « prise d'adresse » ; le compilateur suit à la compilation la plage de survie du pointeur borrow.
Pour distinguer les droits de lecture-écriture, les pointeurs borrow se divisent en deux types : le type `&mut T` en lecture-écriture et le type `&T` en lecture seule. Les expressions opérateurs correspondantes sont `&mut <expr>` et `&<expr>`.

Les pointeurs borrow sont un très bon complément au pouvoir d'expression des pointeurs ARC :
1. Un pointeur ARC ne peut pointer que vers un objet alloué dynamiquement sur le tas ; un pointeur borrow peut pointer aussi bien vers un objet sur le tas que vers un objet sur la pile.
2. Un pointeur ARC ne peut pointer que vers la tête d'un objet alloué dynamiquement sur le tas, et ne supporte pas l'arithmétique de pointeurs ; un pointeur borrow peut pointer vers la tête ou le milieu d'un objet, et réaliser un décalage de pointeur en empruntant un membre.
3. Un type `*T`, à la copie et au passage en paramètre, doit toujours incrémenter ou décrémenter le comptage de références, ce qui a un overhead de performance. L'affectation, le passage en paramètre et le retour d'un pointeur borrow n'ont pas besoin d'opérer sur le compteur de références. Utiliser correctement les pointeurs borrow aide à réduire les opérations redondantes d'incrémentation et de décrémentation du comptage de références.

Points communs entre les pointeurs borrow de Zinc et de Rust :
* Tous deux ont une vérification de lifetime. C'est-à-dire que le compilateur doit vérifier à la compilation que le pointeur borrow lui-même vit moins longtemps que l'objet emprunté.
* Tous deux ont une vérification des droits de lecture-écriture. Un borrow de type `&T` n'a que le droit de lecture, pas d'écriture ; un borrow de type `&mut T` a les droits de lecture-écriture.

Différences entre les pointeurs borrow de Zinc et de Rust :
* Les règles de mutabilité sont différentes. En Zinc, pour un type pointeur ARC `*T`, que la liaison de variable elle-même soit mutable ou non, on peut toujours obtenir via cette variable les deux types de borrow `&T` et `&mut T`.
* Zinc n'a pas de « type d'ownership » ni de « règles de borrow checking ». Un borrow en lecture-écriture n'est pas exclusif. Les règles sémantiques sont beaucoup plus souples.


### Exemple 1 : un borrow pointe vers une adresse sur la pile
```rust
struct S { m: Int, n: Int }
fn main() {
  // on peut pointer directement vers une variable temporaire ou un littéral. Le compilateur génère alors automatiquement une variable locale anonyme, puis fait pointer le pointeur borrow vers elle.
  let mut p2: &Int = &1;

  {
    let x1 = { .m = 1, .n = 2 }: S;
    let p1 = &mut x1; // erreur de compilation, on ne peut pas obtenir un borrow en lecture-écriture sur la variable en lecture seule x1
    p2 = &x1.m; 
  }
  print(p2); // erreur de compilation, x1 vit moins longtemps que p2

  let mut x2 = { .m = 3, .n = 3 }: S;
  let p3 = &mut x2; // correct
  let p4 = &x2.n; // correct
  p3.m += 1;    // correct, p3 a le droit d'écriture
  println(p4); // correct, p3 et p4 peuvent coexister sans problème, p3 n'est pas exclusif
}
```

### Exemple 2 : un borrow pointe vers une adresse sur le tas

Exemple de code Zinc :

```rust
fn main() {
  let mut b: *Int = box 1_i;

  let p1: &Int = &*b; // autorisé, pointe vers une adresse sur le tas
  let p2: &mut Int = &mut *b; // autorisé, pointe vers une adresse sur le tas

  // p1/p2 peuvent coexister, un borrow mut n'est pas exclusif
  println(f"$(p1)");
  *p2 += 1;
  println(f"$(p2)"); 

  b = box 10_i; // réaffectation de b

  // lire et écrire p1/p2/b ne pose aucun problème, p1/p2 n'ont pas été invalidés
  println(f"$(p1)");
  *p2 += 1;
  println(f"$(p2)"); 
  println(f"$(b)");
}
```
L'exemple de code ci-dessus, s'il était écrit en Rust en remplaçant `*Int` par `Box<i32>`, provoquerait une erreur de compilation. Car en Rust, un borrow mutable est exclusif et ne peut pas coexister avec d'autres borrows. Mais Zinc compile, et il n'y a pas de problème de sécurité mémoire.
La raison est que, en Zinc, lorsqu'on prend un borrow sur un membre via un pointeur ARC, le compilateur génère systématiquement une variable temporaire supplémentaire qui copie b une fois, assurant que le compteur de références est incrémenté de +1, puis prend le borrow sur le membre. Cette variable temporaire est libérée à la fin du block courant, et le compteur de références est alors automatiquement décrémenté de 1.
Par conséquent, l'affectation ultérieure de b ne provoque pas la libération immédiate de la mémoire pointée à l'origine : p1/p2 restent valides jusqu'à la fin de la fonction, moment où cette mémoire d'origine est enfin libérée.

Voici le cas des pointeurs ARC à plusieurs niveaux :
```rust
struct Obj { p: *Int } // Zinc n'a pas de type pointeur à sémantique move, on ne peut utiliser que des pointeurs *.

fn main() {
  let mut o: *Obj = box { .p = box 1_i }:Obj;
  let q: &mut Int = &mut *o.p; // on copie temporairement le pointeur p, et non o
  *q = 2; // q a le droit de modification

  // créer un pointeur borrow via un pointeur ARC
  let r: &mut Obj = &mut *o;
  r.p = box 10_i; // autorisé, pas d'erreur de compilation, pas de dangling pointer.

  o = box { .p = box 100_i }:Obj; // autorisé, pas d'erreur de compilation, pas de dangling pointer.

  println(*q); // q pointe toujours vers l'ancienne valeur, le résultat affiché est 2
  println(*r.p); // r.p pointe vers la nouvelle valeur 100
}
```

En résumé, lorsqu'on crée un nouveau pointeur borrow via un pointeur Arc, le moyen qu'utilise Zinc pour éviter qu'il devienne un dangling pointer est d'augmenter de façon protectrice le comptage de références, et de retarder la libération de la mémoire.
Le compilateur n'a besoin de faire qu'une analyse statique locale à l'intérieur d'une fonction pour garantir la sécurité mémoire.

Du point de vue des performances d'exécution : cette conception sacrifie forcément une partie des performances, mais l'impact ne devrait pas être trop important.

1. Ce qui précède n'explique le comportement attendu du code que du point de vue sémantique, et ne décrit pas les instructions concrètes du point de vue de l'optimisation.

   Du point de vue de l'optimisation, il n'est pas nécessaire d'incrémenter systématiquement le comptage de références à chaque création d'un nouveau pointeur borrow. Ce n'est que lorsque le compilateur ne peut pas garantir à la compilation que l'objet pointé par le pointeur borrow survivra forcément dans cette région que, par prudence, le comptage de références est automatiquement incrémenté pour éviter une libération trop précoce de la mémoire.
   Dans de nombreux cas, le compilateur a assez d'informations pour optimiser directement ces opérations supplémentaires d'incrémentation et de décrémentation du comptage de références.

2. Par rapport à Rust, le moment de libération de certains objets alloués dynamiquement sur le tas est retardé.

   Du point de vue de la garantie de sécurité mémoire, un GC mark-and-sweep repose d'ailleurs sur une idée similaire : tant qu'un pointeur pointe encore vers ce bloc de mémoire, le runtime garantit qu'il ne sera pas libéré.
   La différence est seulement qu'un GC mark-and-sweep peut traiter les références circulaires, alors que l'ARC ne le peut pas.
   On notera qu'un pointeur borrow créé à l'intérieur d'une fonction ne peut certainement pas vivre plus longtemps que cette fonction ; donc si un bloc de mémoire est libéré de façon différée à cause d'un pointeur borrow, une fois le bloc de code contenant ce pointeur borrow terminé, ce bloc de mémoire peut aussi être libéré, sans trop de retard.

3. Pour peu que l'on utilise raisonnablement les pointeurs borrow, par rapport à un design qui utiliserait l'ARC partout, on peut souvent réduire la fréquence des opérations de comptage de références.
   1. Un pointeur borrow, à l'affectation, au passage en paramètre et au retour, n'a pas besoin de modifier le comptage de références
   2. Via un pointeur borrow, obtenir un pointeur borrow vers un de ses membres, tant que le membre n'est pas un type pointeur ARC, n'a pas besoin de modifier le comptage de références
   3. Même pour un borrow sur un membre de type pointeur ARC, on peut éliminer par optimisation une partie des modifications redondantes du comptage de références. Cette optimisation dépend de ce que le compilateur a assez d'informations pour savoir qu'un objet pointé par un certain pointeur ARC n'a, pendant un certain lifetime, aucun risque déterministe d'être libéré.


Du point de vue de l'ergonomie : cette conception est une simplification considérable de Rust. **Le borrow checker est supprimé, et la restriction selon laquelle un borrow de type &mut est exclusif disparaît.** Au niveau des types, les types move only sont éliminés.


Du point de vue de la disposition mémoire : la collaboration entre ARC et pointeurs borrow garantit le contrôle du programmeur sur la disposition mémoire.
1. Pour tous les types, la disposition mémoire est déterministe ; le compilateur n'insère pas implicitement de membres cachés.
2. Ce n'est que lorsque nous utilisons explicitement le mot-clé `box` pour allouer de la mémoire sur le tas que le comptage de références est enregistré en tête de la mémoire allouée dynamiquement, avec alors un overhead mémoire de comptage de références. Les variables locales utilisées sur la pile n'ont pas d'overhead mémoire supplémentaire de comptage de références.
3. Tout type utilisé comme membre est directement aplati, sans overhead mémoire supplémentaire de comptage de références. Sauf si le programmeur spécifie que le membre est un type ARC, le compilateur n'insère pas implicitement d'opération box ni de compteur de références.

## Opérations de borrow un peu plus complexes

Les types conteneurs de Zinc, comme `Vec<T>`, supportent aussi l'opération de borrow sur un élément. Exemple :

```rust
fn main() {
    let mut vec = (type Vec<Int>)::from(&[1i,2i,3i,4i] as Slice<Int>);
    let p: &Int = vec.index(1); // ultérieurement, avec la surcharge d'opérateurs, on pourra écrire &vec[1]
    vec.push(5); // ceci peut provoquer un agrandissement de vec
    println(*p); // comment garantir que p est légal ?
}
```

Ici, notre stratégie reste la suivante : à la création du borrow `p: &Int`, on incrémente le comptage de références du tableau sous-jacent, et à la mort de `p` on le décrémente.

La question est : si `p` est un borrow, comment le compilateur sait-il qu'à la mort de cette variable borrow il faut faire une opération de comptage de références, et à quelle adresse la faire ?

Réponse : pour que le code ci-dessus puisse compiler, il faut la collaboration du compilateur et de la bibliothèque standard. L'opération de borrow `vec.index(1)` ne retourne en réalité pas un type borrow natif `&Int`, mais une structure portant des données supplémentaires : c'est un « type pointeur intelligent analogue à un type borrow ».
Puis, via cette variable temporaire, on fait une opération `deref` pour obtenir finalement le pointeur borrow `p: &Int`.

```rust
// ce type implémente le Deref trait, et peut obtenir un type &T via le déréférencement automatique inséré par le compilateur ; puis on accède à l'élément via le type &T.
// ce type implémente le Drop trait, et peut décrémenter le comptage de références du tableau correspondant lors de la destruction
struct ItemRef<'a, T> {
    ptr: *Shared<T>, // pointe vers la tête du tableau
    item: &'a T, // le véritable borrow
}
```

Après désucrage par le compilateur, le code ci-dessus signifie en réalité ceci :

```rust
fn main() {
    let mut vec = (type Vec<Int>)::from(&[1i,2i,3i,4i] as Slice<Int>);
    let _temp: ItemRef<Int> = vec.index(1); // variable locale ajoutée implicitement par le compilateur, c'est un « fat borrow pointer »
    let p: &Int = _temp.deref(); // opération de « déréférencement automatique » ajoutée implicitement par le compilateur
    // attention : si nous ne lions pas la valeur de retour de index à une variable locale, ce _temp devrait être détruit dès la fin de l'instruction index, et non à la fin du bloc d'instructions.

    vec.push(5); // à l'agrandissement, la condition copy-on-write est déclenchée, un nouvel espace est alloué, mais l'ancien espace a un compteur de références supérieur à 0, il n'est pas libéré, p reste légal
    println(*p);
    // le compilateur détruit implicitement la variable _temp, ce qui fait alors tomber le compteur de références de l'ancien espace à 0, et le contenu du tableau d'origine est libéré
}
```

Note : la règle du « déréférencement automatique » est que, dans les scénarios de filtrage par motif, de passage en paramètre et de retour, s'il existe `T: Deref<U>`, alors `T` peut se convertir en type `U` en appelant automatiquement la fonction membre `deref`.

Pour atteindre l'objectif ci-dessus, le trait Index de la bibliothèque standard Zinc est défini ainsi :
```
pub trait Index<Idx>
{
    type Output<'a>;

    fn index<'s>(&'s self, index: Idx) -> Self::Output<'s>;
}
```

Attention : le type de retour de la fonction index n'est plus un type borrow natif figé `&Item`, mais un type structure `struct ItemRef<'a, Item>` qui encapsule `&Item` avec d'autres données.
On peut le voir comme un type pointeur intelligent personnalisé : un type qui « fait porter des métadonnées supplémentaires à un type borrow natif », une sorte de « fat borrow pointer ».
Par conséquent, ce type associé `Output` doit aussi être défini comme un constructeur de type, c'est-à-dire un generic associated type en Rust.
Ce pointeur contient non seulement un borrow vers l'élément concret, mais aussi un pointeur vers la tête du tableau interne, et ce fat pointer a un destructeur : il a assez d'informations pour maintenir correctement à l'exécution la valeur du comptage de références.

Du seul point de vue de l'analyse sémantique, cela semble avoir un gros impact sur les performances. Mais si l'on considère l'inlining des quelques fonctions clés `index` `deref` `drop` utilisées ici, on peut tout à fait, en phase d'optimisation, supprimer les opérations supplémentaires de comptage de références :
1. Si, dans la fonction main, on n'a jamais appelé de fonction de modification comme push, et qu'on n'a jamais pris de borrow mutable sur vec, alors le compilateur a amplement assez d'informations pour savoir que ce bloc de mémoire n'a aucun risque d'être libéré, et peut complètement supprimer toutes les opérations supplémentaires d'incrémentation et de décrémentation du comptage de références, ainsi que les variables locales superflues ;
2. Si, dans la fonction main, on prend un borrow mutable sur vec et qu'on le transmet à d'autres fonctions, le compilateur n'est pas certain que vec puisse être modifié : une opération protectrice de comptage de références est alors indispensable, ce n'est pas un gaspillage de performance.

## Comment Zinc résout le problème classique d'invalidation d'itérateur

Lorsqu'on explique les fonctionnalités de sécurité mémoire de Rust, voici un exemple classique :

```rust
///// Rust code
fn main() {
    let mut vec: Vec<i32> = Vec::from(&[1i,2i,3i]);
    for ref _i in &vec {
        vec.push(4);
    }
}
```

Rust produira une erreur de compilation sur cet exemple, car `vec.iter()` a créé un borrow en lecture seule sur `vec`, stocké dans l'itérateur ; tandis que `vec.push` a besoin de créer un borrow en lecture-écriture sur `vec`.
Selon les règles de borrow checking de Rust, les deux entrent en conflit, donc le compilateur signale une erreur, ce qui empêche efficacement le problème d'invalidation d'itérateur, et évite un comportement indéfini courant en C++. C'est un mérite de Rust qu'il faut saluer.

Zinc garantit la sécurité d'une autre manière. Voici le code Zinc correspondant ; au niveau du code source, il se ressemble beaucoup, mais l'effet d'exécution réel est très différent de Rust :

```rust
fn main() {
    let mut vec: Vec<Int> = (type Vec<Int>)::from(&[1i,2i,3i,4i] as Slice<Int>);
    for ref _i in vec {
        vec.push(4);
    }
}
```

1. Parce que Zinc n'a pas de « type d'ownership », `Vec<T>` n'a pas une sémantique move, mais une sémantique copy. On peut librement faire `let vec2 = vec;`, et continuer à utiliser la variable `vec`.
2. Bien que `Vec<T>` ait une sémantique copy, cela ne signifie pas que, à la copie, tous les membres sont immédiatement copiés : une optimisation copy-on-write est implémentée.
Lors de la copie d'un `Vec<T>`, il n'y a qu'une copie superficielle : on incrémente seulement de 1 le comptage de références du tableau alloué sur le tas interne, le contenu des éléments n'est pas vraiment copié.
3. La copie profonde se produit lors de la modification d'une variable `Vec<T>` : si, au moment de la modification, on constate que le comptage de références du tableau interne est supérieur à 1, alors tous les éléments sont copiés, puis la modification est effectuée.
4. Dans l'exemple ci-dessus, à la création de l'itérateur, le comptage de références du tableau interne est incrémenté de 1. Si, pendant l'itération, il n'y a pas d'écriture sur `vec`, alors la copie profonde ne se produira jamais ;
Si, pendant l'itération, un `vec.push` se produit, on constate alors qu'il faut déclencher une copie profonde : `vec` crée en interne un nouveau tableau, et y ajoute un 4 à la fin.
5. Mais le tableau pointé par l'itérateur n'a pas changé, n'a pas été libéré ni modifié, et l'itération continue. L'itérateur et `vec` pointent désormais vers deux tableaux complètement différents, il n'y a donc aucun comportement indéfini.
6. Une fois la boucle for terminée, le lifetime de l'itérateur se termine ; à sa destruction, il décrémente de 1 le comptage de références du tableau d'origine, ce qui déclenche alors seulement la destruction et la libération du tableau d'origine.

Ainsi, Zinc évite également le « comportement indéfini », n'introduit pas d'« insécurité mémoire », mais paie un certain coût en performance : lors de l'opération push, le type `Vec<T>` d'origine a implicitement effectué une copie profonde.
Ce coût est acceptable : nous ne voulons ni de comportement indéfini comme en C++, ni de vérifications à la compilation trop strictes comme en Rust ; en garantissant « sécurité » et « facilité d'utilisation », la perte de performance d'exécution est déjà le plus petit coût à payer.
Si l'utilisateur souhaite économiser cette copie profonde, il doit lui-même veiller à ne pas modifier le vec d'origine pendant la boucle.

Pour Rust, la sécurité mémoire et la thread-safety sont un **objectif de conception**. Le « principe d'exclusion mutuelle entre mutabilité et partage » (alias XOR mutation principle) est le **fondement théorique** menant à cet objectif.
L'ownership et le borrow checker sont le **moyen d'implémentation**. Zinc a choisi d'utiliser la technique copy-on-write pour atteindre le même objectif : des chemins différents, la même destination.
Ou, pour le dire autrement, cette conception de Zinc consiste à transformer la vérification statique à la compilation de « mutabilité et partage mutuellement exclusifs » (alias XOR mutation) en une garantie à l'exécution.
Le copy-on-write garantit essentiellement qu'au moment de modifier la mémoire, ce bloc n'a qu'un seul alias, ce qui revient au même que ownership + borrow checker. Comparaison détaillée :

* La vérification ownership + borrow checker à la compilation de Rust garantit à la compilation qu'un objet n'a qu'un unique mutable alias. Mais plusieurs immutable alias peuvent coexister.

  Si le principe alias XOR mutation est violé, c'est une erreur de compilation.

* Le `RefCell<T>` de Rust s'utilise via les fonctions membres `borrow` `borrow_mut`. Il garantit à l'exécution qu'un objet n'a qu'un unique mutable alias, mais plusieurs immutable alias peuvent coexister.

  Si le principe alias XOR mutation est violé, c'est un panic à l'exécution, l'exécution est refusée.

* Le `RwLock<T>` de Rust s'utilise via les fonctions membres `read` `write`. Il garantit à l'exécution qu'un objet n'a qu'un unique mutable alias, mais plusieurs immutable alias peuvent coexister.

  Si le principe alias XOR mutation est violé, le thread courant est bloqué en attente à l'exécution, jusqu'à ce que la condition soit satisfaite pour continuer. `Mutex<T>` est similaire : il garantit qu'un bloc de données n'a qu'un seul accesseur à la fois, en lecture comme en écriture.

* Le copy-on-write de Zinc s'implémente ainsi : dans une immutable method, on ne fait pas de vérification de ref-count ; dans une mutable method, on fait une vérification de ref-count, pour garantir qu'au moment de la modification l'objet n'a qu'un unique alias.

  Si le principe alias XOR mutation est violé, l'objet alloué sur le tas est copié. On garantit qu'au moment de modifier / libérer la mémoire, il n'y a qu'un seul alias.

Toutes les techniques d'implémentation ci-dessus satisfont le même principe « alias XOR mutation », et peuvent toutes atteindre l'objectif de « sécurité mémoire ». La conception de Rust n'est qu'une des options pour implémenter la « sécurité mémoire », absolument pas la seule conception viable.

## Lifetimes

On appelle lifetime l'intervalle pendant lequel une variable est vivante au cours de l'exécution du programme. Le lifetime d'une variable commence à sa création et se termine à sa destruction.

### Lifetime des variables globales

Une variable globale survit pendant toute l'exécution du programme, donc son lifetime existe du début à la fin du programme.

> **TODO** : dans l'implémentation actuelle, aucune variable globale n'appelle de destructeur. Il faudra discuter ultérieurement si cette conception est raisonnable, et si nous devons détruire les variables globales avant de quitter le processus.

### Lifetime des variables temporaires

Une variable temporaire est une variable produite par une expression, mais non liée à un nom concret.

Le moment où le lifetime d'une variable temporaire se termine a deux cas : l'un à la fin de l'instruction courante ; l'autre à la fin du bloc d'instructions courant. Cela dépend de ce qu'elle a été empruntée ou non par une variable de lifetime plus long.

Exemple :
```rust
fn f() -> S { ... }
// on suppose que S a une méthode membre method
impl S {
  fn method(&self) -> &S { return self; }
}

fn test() {
  f(); // la variable retournée par f() est une variable temporaire, elle n'est liée à aucun nom de variable. Le lifetime de cette variable temporaire se termine à la fin de l'instruction.

  let p1: &S = &f(); // la variable temporaire retournée par f() a été empruntée, et le lifetime du borrow dépasse cette instruction. Le lifetime de cette variable temporaire est alors étendu jusqu'à la fin du bloc d'instructions ; la variable temporaire n'est détruite qu'à la fin de la fonction.

  let p2: &S = f().method(); // la variable temporaire retournée par f() a été empruntée, et le lifetime du borrow dépasse cette instruction. Le lifetime de cette variable temporaire est alors étendu jusqu'à la fin du bloc d'instructions ; la variable temporaire n'est détruite qu'à la fin de la fonction.
  // utilisation de p1 p2
}
```

### Lifetime des variables locales

En général, le lifetime d'une variable locale, y compris d'un paramètre de fonction, commence à sa création et se termine à la fin du bloc de code courant.

Il n'y a pas de chose telle que non-lexical-lifetime, et ce n'est pas nécessaire. Rust a introduit cela parce que l'exclusivité des types mut borrow, si l'analyse n'est pas assez précise, provoque de nombreuses erreurs de compilation inutiles et limite le pouvoir d'expression de l'utilisateur. Si nous n'avons pas de règle de vérification à la compilation de l'exclusivité des mut borrow, alors nous n'avons pas non plus besoin de chercher à assouplir les règles de vérification à la compilation.

* Pour un type personnalisé, on peut lui implémenter le `Drop trait` ; une variable de ce type appellera automatiquement le destructeur correspondant à la fin de son lifetime.
* Pour un type ARC, à la fin du lifetime, le comptage de références de l'espace mémoire du tas pointé est automatiquement décrémenté de 1.
* Pour un type borrow, son destructeur ne fait rien.

Exemple :
```rust
struct S { m: i32 }
impl Drop for S {
  fn drop(&mut self) {
    println("drop S");
  }
}

fn test() {
  let s1 = { .m = 1 }:S;
  let s2 = s1; // une copie se produit ici

  // s1 et s2 sont détruits à la fin de test
}
```

Si l'on a dit plus haut « en général, le lifetime d'une variable locale se termine à la fin du bloc de code », c'est qu'il existe aussi des « cas particuliers ». Le cas particulier est l'expression `move`.

En Zinc, le mot-clé move peut être suivi d'une expression, appelée expression move. Une sémantique de déplacement se produit alors, et non une sémantique de copie.

```rust
fn test() {
  let x = { .m = 1 }:S;
  let y = move x;
  println(x.m); // erreur de compilation, le lifetime de x est déjà terminé, on ne peut plus utiliser x ensuite
}
```

Si ce que l'on move est un pointeur ARC, on peut utiliser cette fonctionnalité pour réduire, dans certains cas, les opérations d'incrémentation et de décrémentation du comptage de références :
```rust
fn f(arg: *S) { }

fn test() {
  let s: *S = box { .m = 1 }:S;
  f(s); // si on appelle ainsi, une copie se produit au passage en paramètre, le comptage de références est incrémenté de 1, et décrémenté de 1 à la fin du corps de f. En sortant de test, le comptage de références est encore décrémenté de 1,
  f(move s); // si on appelle ainsi, on garantit que le comptage de références n'est pas automatiquement incrémenté de 1 au passage en paramètre, et n'est pas non plus décrémenté de 1 à la fin du corps de test. Le lifetime de s est transféré à la maintenance du corps de f.
  // après que s a été move, réutiliser s déclenche une erreur de compilation

  // ...
}
```

Si l'on souhaite que le lifetime de la variable `x` se termine plus tôt, on peut utiliser l'instruction `move x;`, afin que le résultat de l'expression move ne soit lié à aucune variable : ce x est alors détruit sur-le-champ, et non à la fin du bloc d'instructions.

En résumé, pour la sémantique move de Zinc, le choix n'est pas du côté de la « définition de type » : tous les types sont naturellement copiables ou déplaçables.
Utiliser ou non la sémantique move, le choix est du côté de l'« expression ». Zinc fournit une « expression » à sémantique move, et non un « type » à sémantique move.

L'instruction `return` a par défaut une sémantique move ; `return expr;` est équivalent à `return move expr;`, il n'est pas nécessaire d'écrire explicitement le mot-clé `move`.

## Annotations de lifetime

Une annotation de lifetime peut être vue comme une sorte de paramètre générique. Les types borrow ont besoin de ce genre de paramètre générique.
Les annotations de lifetime explicites s'utilisent généralement dans les signatures de fonctions, pour exprimer la relation de lifetime entre les paramètres et le type de retour.

```rust
fn find<'a, 'b>(strings: Slice<'a, Str<'b>>) -> Str<'b> {

}
```

> **TODO** : les règles d'omission des annotations de lifetime dans les signatures de fonctions ne sont pas encore entièrement implémentées.

## Destructeurs et `Drop`

Un destructeur est une fonction membre d'un objet. À la fin du lifetime de l'objet, le compilateur insère automatiquement des instructions pour appeler le destructeur.

Le destructeur fait les choses suivantes :
1. Appeler la fonction `Drop::drop` implémentée par ce type, s'il n'y en a pas, ne pas l'appeler
2. Appeler la fonction `Drop::drop` des membres
3. Décrémenter de 1 le comptage de références correspondant à tous les membres de type pointeur Arc et Weak contenus dans ce type. Si un objet pointé par un pointeur ARC a un compteur de références tombé à 0, le destructeur de cet objet est automatiquement appelé.

Le destructeur est toujours généré automatiquement par le compilateur ; l'utilisateur ne peut contrôler que le comportement de la fonction `Drop::drop`, qui n'est qu'une partie du processus de destruction.
Si un type et tous ses membres n'ont ni fonction `Drop::drop`, ni pointeur de type comptage de références, alors le destructeur de ce type est vide.

En général, l'utilisateur ne devrait pas appeler activement le destructeur d'un objet. Mais en écrivant du code unsafe, l'utilisateur est autorisé à forcer l'appel du destructeur d'un objet via la fonction `unsafe fn destruct_in_place`.

La fonction `Drop::drop` ne peut jamais être appelée explicitement par l'utilisateur.

Exemple d'`impl std::mem::Drop` trait pour un type personnalisé :

```rust
struct S {
  p: *i32,
  m: String,
}

impl Drop for S {
  fn drop(&mut self) {
    println("S is dropped.")
  }
}
```

Attention : une fonction `Drop::drop` personnalisée n'a pas à s'occuper de la destruction et de la libération mémoire des membres ; les destructeurs des membres sont appelés automatiquement.
