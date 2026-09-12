
# Parallélisme et concurrence

## Thread-safety

Rust a un avantage incomparable : il garantit naturellement la thread-safety d'un programme. Parmi les langages de programmation courants, il est unique. Zinc souhaite aussi conserver cette caractéristique de thread-safety.

### Send/Sync trait

Nous marquons de façon unifiée tous les types qui peuvent être transmis de façon sûre entre threads, et nous appelons cela le `Send trait`.

Rappelons d'abord le sens de la [sémantique de valeur et de la sémantique de référence](https://isocpp.org/wiki/faq/value-vs-ref-semantics#val-vs-ref-semantics).

On peut constater qu'un type à sémantique de valeur satisfait naturellement les exigences du `Send trait`. Exemples :
1. Les types de base Bool, Char, Int, etc., correspondent à une sémantique de valeur, donc satisfont certainement le `Send trait`.
2. Les types composés, comme tuple/struct/enum, etc., si tous leurs membres satisfont `Send`, alors ils ont forcément une sémantique de valeur, donc ces types satisfont aussi le `Send trait`.
3. Un type pointeur ne correspond pas à une sémantique de valeur ; en général `*T` ne satisfait pas le trait `Send`. Mais il faut aussi analyser selon les cas. Un type pointeur, ou un type contenant un membre pointeur, satisfait aussi `Send` dans les deux cas suivants :
    1. Via un mécanisme copy-on-write, une sémantique de valeur a été implémentée ; ceux-ci satisfont aussi le `Send trait`. Par exemple le type `String` : bien qu'il contienne un membre pointeur, son implémentation garantit une « sémantique de valeur » via copy-on-write, donc ce type satisfait aussi le `Send trait`. Pour un type comme `Vec<T>`, bien qu'il implémente aussi le copy-on-write, comme il porte un générique, le fait qu'il satisfasse `Send` est conditionnel. Si et seulement si `T: Send`, alors `Vec<T> : Send`.
    2. Via un mécanisme de synchronisation entre threads, le contenu pointé par le pointeur est thread-safe ; ceux-ci satisfont aussi le `Send trait`. Par exemple le type `*AtomicInt32` : bien que deux pointeurs dans deux threads puissent pointer vers le même `AtomicInt32`, comme `AtomicInt32` a une synchronisation entre threads, le type `*AtomicInt32` satisfait le `Send trait`.

Nous introduisons donc un autre `Sync trait`, qui représente qu'un type possède lui-même une capacité de synchronisation entre threads. On peut alors dire que, pour un type `*T`, si et seulement si `T` satisfait `Sync`, alors `*T` satisfait `Send`.
Pour les pointeurs borrow, c'est analogue. Si et seulement si `T: Sync`, alors `&T : Send` et `&mut T : Send`.

Exemples de types satisfaisant le `Sync trait` :
1. Les types de la série `Atomic`
2. Les types `Mutex<T>` `RwLock<T>`, etc. Attention : comme ces types portent eux-mêmes un générique, le fait qu'ils satisfassent `Sync` est conditionnel. Si et seulement si `T: Send`, alors `Mutex<T> : Sync`.

Une fois ces deux traits en place, il reste deux choses à faire pour garantir la thread-safety :
1. Dans la bibliothèque standard, marquer tous les types de base, ainsi que les types définis par la bibliothèque standard, avec `Send` et `Sync`.
2. Sur les API qui doivent transmettre des données entre threads différents, ajouter une contrainte, limitant le type transmis entre threads à ceux qui satisfont la condition `Send`.

Pour un type défini par l'utilisateur, en général il n'est pas nécessaire de marquer manuellement `Send` `Sync` ; le résultat inféré par le compilateur suffit.
Si l'implémentation de ce type utilise unsafe, le programmeur doit alors analyser lui-même si ce type satisfait `Send` `Sync`, et le spécifier explicitement via unsafe impl.

### Absence de data race

Une fois que le compilateur et la bibliothèque standard ont préparé les types `Send` `Sync` ci-dessus, nous pouvons garantir par vérification à la compilation l'« absence de data race » dans le code métier.
S'il existe une possibilité de data race, le compilateur signale automatiquement une erreur.

Exemple :

```
// on suppose que la fonction send a pour rôle d'envoyer data vers un autre thread pour y être utilisée ; elle doit donc restreindre le type de data à la contrainte Send
fn send<T>(data: T) where T: Send { }

fn main() {
    send(1_i);
    send(String::new());
    send((type Vec<Int>)::new());
    send(box 1_u); // Error. erreur de compilation, *UInt ne satisfait pas la contrainte Send
}
```

### Variables globales

Une variable globale peut naturellement être accédée par différents threads, il faut donc aussi la restreindre pour garantir la thread-safety.
1. La définition d'une variable globale peut être décorée par le mot-clé `static` ou `unsafe static`. Une variable globale décorée par `unsafe` ne peut être lue et écrite que dans une zone unsafe. Seule une variable globale en lecture seule définie par `static` est sûre.
2. Le type d'une variable globale non décorée par `unsafe` doit satisfaire les exigences du `Send trait`.

**Attention** : contrairement aux règles de Rust, Zinc exige qu'une variable globale satisfasse la contrainte `Send`, et non la contrainte `Sync`.

Parce que le `Sync trait` de Zinc et le `Sync trait` de Rust n'ont pas la même sémantique. De nombreux types qui satisfont `Sync` en Rust ne satisfont pas `Sync` en Zinc.
Prenons le type de base `Int` : en Rust, il satisfait `Sync` ; mais en Zinc, `Int` ne peut pas satisfaire `Sync`.
Par l'absurde : si nous stipulions que `Int` satisfait `Sync`, cela signifierait qu'un pointeur comme `*Int` satisfait `Send`, donc qu'il peut être transmis entre threads, et l'on verrait alors deux threads obtenir un pointeur vers le même `Int`, sans synchronisation entre threads : c'est une erreur.

* Le `Sync` de Zinc représente un type qui a en interne un mécanisme supplémentaire de synchronisation entre threads. Les types satisfaisant la contrainte `Sync` sont beaucoup moins nombreux qu'en Rust. 
* Le `Send` de Zinc représente un type qui peut être transmis de façon sûre entre threads. Il comprend principalement quatre cas :

    1. Un type classique à sémantique de valeur, sans pointeur en interne ; à la transmission entre threads, on copie un exemplaire, donc c'est certainement sûr
    2. Le type contient un pointeur, et les données pointées sont en lecture seule. Les données partagées sont en lecture seule, donc c'est sûr.
       Par exemple le type `Str` : son API externe ne fournit aucune capacité de modification, ce qui garantit que `Str` satisfait `Send`.
    3. Le type contient un pointeur, et les données pointées satisfont le copy-on-write. Si un thread tente de modifier les données partagées, il en copie un exemplaire avant de modifier ; tant que les données sont en état partagé, elles ne seront certainement pas modifiées ; ce que l'on modifie est toujours sa propre copie, donc c'est sûr.
       Par exemple le type `Vec<Int>` : toutes les fonctions membres ayant une capacité de modification testent le comptage de références ; si ce n'est pas exclusif, les données sont d'abord copiées puis modifiées
    4. Le type contient un pointeur, et les données pointées peuvent être modifiées sous réserve de synchronisation entre threads. Après transmission entre threads, plusieurs threads détiendront un pointeur vers les mêmes données, mais toutes les lectures et écritures des données partagées ont une synchronisation entre threads, donc c'est sûr
       Par exemple les types `*AtomicInt` `*Mutex<String>`


Bien que certains détails de conception de `Send` et `Sync` diffèrent entre Zinc et Rust, les deux garantissent la thread-safety.


<div style="border: 1px solid black; padding: 10px; background: #F0F0F0">

D'après la conception ci-dessus, on voit qu'un type comme `Vec<Int>` satisfait la contrainte `Send`. On peut transmettre de façon sûre ce type entre threads.

1. Si nous transmettons un `Vec<Int>` vers différents threads, et que chaque thread ne fait que le lire, alors seule une copie superficielle se produit, l'efficacité d'exécution est élevée.
2. Si nous transmettons un `Vec<Int>` vers différents threads, et qu'un thread le modifie, alors au moment de la modification dans ce thread, une copie profonde est déclenchée : il y a alors un overhead de performance important, mais pas de problème de thread-safety.
3. Si nous souhaitons transmettre un `Vec<Int>` entre threads, qu'il puisse être modifié, et sans copie profonde. Alors le programmeur doit lui-même garantir qu'il n'existera pas simultanément plusieurs instances de copie superficielle. Cela peut s'obtenir via une expression move.
Transférer l'ownership d'un `Vec<Int>` entre différents threads permet d'atteindre l'objectif d'une modification inter-threads efficace. Mais le compilateur ne suit pas l'ownership au niveau des types, seulement au niveau du flux de contrôle, pour réduire l'impact sur l'utilisateur.
4. Si nous souhaitons obtenir un `Vec<Int>` à sémantique de référence, il suffit de le boxer : le `*Vec<Int>` obtenu est un type à sémantique de référence. Différents pointeurs peuvent modifier le même `Vec<Int>`, en partageant le même bloc de données.
Parce que `Vec<Int>` ne satisfait pas `Sync`, `*Vec<Int>` ne satisfait pas `Send`. Le compilateur peut alors nous aider à vérifier que tous ces pointeurs restent à l'intérieur d'un même thread : un type non-`Send` ne peut pas franchir la frontière d'un thread.
Donc un type `*Vec<Int>` à sémantique de référence n'aura pas non plus de problème de thread-safety. Et les opérations de comptage de références de ce pointeur Arc peuvent être optimisées en opérations d'incrémentation et de décrémentation non atomiques.

On voit d'après ce qui précède que l'optimisation copy-on-write est très importante pour un type comme `Vec<T>`.

Supposons que Zinc n'ait pas choisi l'ARC comme mécanisme de base de gestion mémoire, mais un GC, et que l'expression `box` retourne un pointeur GC trace-able.
Alors un type conteneur comme `Vec<T>` ne pourrait, dans son implémentation interne, utiliser que des pointeurs GC, et ne pourrait pas être un type à « sémantique de valeur ». Il ne pourrait qu'être non-Send. Et l'expression move n'aurait plus de sens.
Dans ce cas, en scénario multithread, les types pouvant satisfaire la contrainte `Send` seraient très peu nombreux ; pour garantir la thread-safety, les cas de copie profonde seraient plus nombreux, ce qui apporterait au contraire un énorme coût supplémentaire en performance.
Donc la gestion mémoire ARC et l'objectif de thread-safety de Zinc sont les plus adaptés l'un à l'autre.

Et c'est précisément parce que Zinc a aussi atteint la thread-safety que nous pouvons garantir à la compilation que certains pointeurs ARC ne peuvent certainement pas franchir la frontière d'un thread : eux-mêmes et leurs copies ne peuvent être utilisés qu'à l'intérieur d'un même thread.
Grâce à cette information, le compilateur peut optimiser leurs opérations d'incrémentation et de décrémentation du comptage de références en opérations non atomiques, améliorant encore les performances. Cette optimisation ne dépend que du type, pas du flux de contrôle.

Donc thread-safety et gestion mémoire ARC se complètent parfaitement.

</div>

## Threads

Les API liées aux threads fournies par la bibliothèque standard Zinc sont une encapsulation simple des fonctionnalités de threads du système d'exploitation. Pour créer un thread, on utilise la fonction suivante :

```
// mod std::thread
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: Send + 'static + Fn()->T,
    T: Send + 'static,
```

`JoinHandle` peut appeler la fonction `join()` pour attendre la fin du thread. On peut aussi appeler la fonction `thread()` pour obtenir une variable de type `&Thread`.

Exemple :

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

Le type `ThreadLocal` sert à exprimer une variable « locale au thread ». Chaque thread maintient indépendamment sa propre copie.

Utilisation :

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

## Verrous

Les verrous comprennent `Mutex<T>` `RwLock<T>`.

La conception de l'API des verrous de Rust a un excellent avantage : elle fusionne le verrou et les données protégées en un seul type, évitant ainsi le cas où le verrou et les données protégées ne correspondent pas, et rendant impossible d'accéder aux données en oubliant de prendre le verrou.

Mais elle a quelques petits problèmes :
1. La conception poison
2. Il est possible, dans du code safe, de laisser fuir un MutexGuard, ce qui produit un cas où l'on ne peut plus déverrouiller
3. Dans du code asynchrone, si un await se produit pendant que l'on tient le verrou, un deadlock est facilement déclenché

Zinc a apporté une petite modification à cette API :
1. Suppression de la gestion de l'état poison, car Zinc n'a pas de mécanisme catch unwind
2. Verrouillage et déverrouillage passés à un style callback

```
fn synchronized(p: &Mutex<String>) {
    // ultérieurement, on pourra s'inspirer de Swift pour ajouter davantage de sucre syntaxique, et omettre aussi cette liste de paramètres du lambda
    p.lock.{ (data: &mut String) =>  
        data.push_str("tail");
    };
}
```

Les avantages de cette approche :
1. La portée du verrouillage est claire, entourée d'accolades, assez visible, adaptée à la lecture humaine
2. Il n'y a pas de MutexGuard, donc pas de risque de fuite ; un comportement de verrouillage correspond forcément à un comportement de déverrouillage, ce qui évite le risque de ne pas pouvoir libérer le verrou
3. En cas de verrouillage à plusieurs niveaux, on peut garantir que verrouillage et déverrouillage satisfont forcément le principe dernier entré, premier sorti. Rust, parce qu'il expose MutexGuard et autorise l'utilisateur à drop manuellement un MutexGuard, peut avoir un ordre de déverrouillage réel non structuré
4. Dans la fonction de callback, il est impossible d'utiliser une expression await, car le type de l'API de callback ne correspond pas, ce qui évite qu'un await se produise pendant que l'on tient le verrou

Bien sûr, cela apporte un inconvénient : on ne peut plus, comme auparavant, utiliser naturellement pendant le verrouillage les instructions de flux de contrôle `break` `continue` `return`, le compilateur déverrouillant automatiquement. Car les instructions de flux de contrôle dans la fonction de callback n'affectent que l'intérieur du lambda ; s'il faut traiter le flux de contrôle extérieur, il faut, via différentes valeurs de retour de ce lambda, faire des tests supplémentaires à l'extérieur du lambda. Cela dit, cette écriture est au moins, du point de vue de la lisibilité, très claire. Ne pas pouvoir utiliser directement les instructions de flux de contrôle n'est pas forcément une mauvaise chose.

### Conception anti-copie

Le type de verrou de Rust est conçu avec une sémantique move, mais Zinc n'a pas de type à sémantique move. Cela apporte un nouveau risque : l'utilisateur peut involontairement copier une variable `Mutex<T>`, ce qui produit facilement un bug.

Exemple :
```
// exemple de pseudo-code, ce code ne peut en réalité pas compiler
fn test(s: &Mutex<String>) {
    let str_copy = *s; // dans certains cas, cette copie peut être très dissimulée, ce qui peut produire un bug
    str_copy.lock.{ (s: &mut String) => println(s) };
}
```

En Zinc, pour éviter cette situation, on a introduit le `UnSized` trait. Ce trait est un marker trait, sans fonction membre. Et l'on stipule :
1. On peut, à la définition d'un type, faire `impl UnSized for MyType {}` pour un type personnalisé
2. Tous les types `UnSized` ne peuvent pas servir directement de variable globale, de constante, de variable locale ou de membre. Ce type ne peut être accédé qu'indirectement via un pointeur.
3. On ne peut pas déréférencer un type pointeur vers `UnSized`.

En stipulant dans la bibliothèque standard qu'un type comme `Mutex` est `UnSized`, on peut éviter le bug ci-dessus.

```
struct S {
    value: Mutex<MyType>, // Error, Mutex ne peut pas servir directement de membre, il est recommandé d'utiliser *Mutex<MyType> à la place
}

fn f() { 
    let local = Mutex::new(MyType::new()); // Error, Mutex ne peut pas servir directement de variable locale, il est recommandé d'utiliser *Mutex<MyType> à la place
}

static GLOBAL_DATA: Mutex<MyType> = Mutex::new(MyType::new()); // Error, Mutex ne peut pas servir directement de variable globale, il est recommandé d'utiliser &'static Mutex<MyType> à la place

static GLOBAL_DATA: &'static Mutex<MyType> = &Mutex::new(MyType::new()); // OK
```

Des fonctions comme `swap` dans la bibliothèque standard ne peuvent pas non plus opérer sur un type `UnSized`. Cette conception peut donc éviter l'erreur ci-dessus « avoir copié tout le Mutex par inadvertance ».

Des types comme `File` dans la bibliothèque standard ont une conception similaire. L'utilisateur ne peut jamais utiliser que « un type pointeur vers File », et ne peut pas utiliser directement le type `File` comme type valeur.

> **TODO** : la conception des UnSized types doit encore être perfectionnée. Il faut autoriser un UnSized type comme paramètre de fonction et comme retour de fonction. Certaines fonctions génériques doivent aussi autoriser un argument générique UnSized.
> Mais il faut restreindre tous les UnSized types à n'être utilisables que comme variables temporaires ; avant d'être liés à une variable de type sized, ils ne peuvent qu'être move, pas copiés.

```
impl<T> UnSized for Mutex<T> {}
impl<T> Mutex<T> where T: ?Sized { // il faut autoriser que T puisse être unsized
    fn new(v: T) -> Mutex<T> { // il faut autoriser un unsized type comme paramètre et retour de fonction
        return {
            .val = move v, // un paramètre unsized ne peut qu'être move
            ...
        };
    }
}
```

## I/O asynchrone

> **TODO** : la fonctionnalité d'I/O asynchrone n'est pas encore implémentée.

Idée de base : l'ergonomie prime sur les considérations de performance. Rejeter résolument la conception Move/Pin de Rust, et aligner l'ergonomie sur C#. La conception Move/Pin, dans la pratique, introduit une complexité bien supérieure aux bénéfices, le jeu n'en vaut pas la chandelle.
Hormis le problème des références circulaires Arc, l'ergonomie de Zinc devrait être comparable à celle de C#.
