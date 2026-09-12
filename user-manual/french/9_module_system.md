# Système de modules

## Composant (component)

L'unité de compilation de Zinc est le composant. Il correspond au crate de Rust.

> In C and C++ programming language terminology, a translation unit (or more casually a compilation unit) is the ultimate input to a C or C++ compiler from which an object file is generated.

L'unité de compilation désigne les fichiers source nécessaires à l'exécution d'un processus de compilateur. Ces fichiers source constituent l'unité d'entrée minimale d'une compilation ; on ne peut pas les découper davantage. Un découpage supplémentaire ne permettrait plus d'exécuter une compilation.

Pour un compilateur C/C++, l'unité de compilation est un fichier source .c ou .cpp, ainsi que tous les fichiers qu'il `#include` directement et indirectement.

Pour le compilateur Rust, l'unité de compilation est un crate. L'utilisateur ne peut pas compiler séparément les différents mod à l'intérieur d'un crate.

Pour le compilateur Zinc, l'unité de compilation est un composant (component). L'entrée de chaque exécution du compilateur est le code source d'un composant complet. L'utilisateur ne peut pas compiler séparément les différents fichiers source d'un composant, puis les assembler en un composant : le compilateur n'en a pas la capacité. Un composant doit être compilé comme un tout.

Les composants ne peuvent pas avoir de dépendances circulaires entre eux.

À l'intérieur d'un composant, il y a des modules (mod) ; les modules à l'intérieur d'un composant peuvent avoir des dépendances circulaires. Car tous les modules d'un même composant sont compilés ensemble dans un même processus de compilateur.

Après compilation, un composant génère un fichier `.o` contenant du code machine, ainsi qu'un fichier `.zno` des interfaces publiques du composant. zno tire son nom de la substance « oxyde de zinc ».

On peut comprendre un fichier `.zno` comme un « fichier d'en-tête » en C, sauf que le fichier d'en-tête est écrit à la main, alors que le fichier `.zno` est généré automatiquement par le compilateur.

Le contenu d'un fichier `.zno` comprend toutes les signatures de fonctions publiques, les définitions de types, les déclarations de variables globales, les déclarations impl, etc. Il comprend aussi le corps de toutes les fonctions pouvant être inlinées entre composants.

## `export`

Si un composant est une bibliothèque, l'utilisateur doit déclarer le nom de ce composant avec `export component_name;`.
Si un composant est un exécutable, l'utilisateur doit définir dans le root mod une fonction `fn main()` comme point d'entrée du programme.
Si un composant n'a ni l'un ni l'autre, le compilateur signale une erreur.

En Rust, le nom d'un crate est spécifié par l'option de compilation `--crate-name NAME` ; je trouve cela déraisonnable : le nom d'un composant devrait être spécifié par son code source.
Après tout, le nom du composant affecte le mangle name des symboles dans le binaire ; si le nom du composant est contrôlé par une option de compilation, il est difficile de garantir la cohérence des résultats de compilation dans différents environnements de compilation.

## `import`

Lorsque nous voulons décrire qu'un composant a dépend d'un autre composant b, il faut écrire `import b;` dans le code source de a. Cela équivaut essentiellement à `extern crate b;` en Rust.

Par analogie avec C/C++, on peut simplement le comprendre comme `#include "b.h"`.

## Présentation de mod

Le module (mod) est l'unité de base d'organisation du code en Zinc. Un module peut contenir des fonctions, des types, des constantes, ainsi que d'autres modules.

### Définition d'un module

On définit un module avec le mot-clé `mod` :

```rust
// définir un module dans un fichier
mod my_module {
    fn helper() -> Int {
        return 42;
    }
    
    pub fn public_function() {
        println(f"helper returned $(helper())");
    }
}
```

### Modules du système de fichiers

Les modules peuvent aussi s'organiser via le système de fichiers. Si un module n'a pas de contenu, le compilateur cherche un fichier ou un répertoire du même nom :

```rust
// dans main.zn
mod network;

// le compilateur cherchera :
// 1. le fichier network.zn
// 2. le fichier network/mod.zn
```

### Visibilité

Les items d'un module sont privés par défaut, et ne sont accessibles qu'à l'intérieur du module qui les définit. Le mot-clé `pub` rend un item visible à l'extérieur :

```rust
mod my_module {
    pub fn public_function() {
        // accessible depuis l'extérieur
    }
    
    fn private_function() {
        // accessible uniquement à l'intérieur de my_module
    }
    
    pub struct PublicStruct {
        pub field: Int,      // membre public
        private_field: Int,  // membre privé
    }
}
```

### Modules imbriqués

Les modules peuvent se définir de façon imbriquée, formant une structure arborescente :

```rust
mod outer {
    pub fn outer_function() {}
    
    mod inner {
        pub fn inner_function() {}
        
        mod deeply_nested {
            pub fn deep_function() {}
        }
    }
}
```

### Chemins de modules

On accède aux items d'un module avec l'opérateur `::` :

```rust
fn main() {
    outer::outer_function();
    outer::inner::inner_function();
    outer::inner::deeply_nested::deep_function();
}
```

### Modificateurs de visibilité

Zinc supporte les modificateurs de visibilité suivants :

1. **Par défaut (privé)** : accessible uniquement à l'intérieur du module qui le définit
2. **`pub`** : visible pour tous les modules
3. **`pub(in path)`** : visible uniquement dans le module du chemin spécifié (> **Fonctionnalité prévue** : support futur)

```rust
mod my_module {
    pub fn public_api() {}
    
    fn internal_helper() {}
    
    // support futur : visible uniquement dans le module parent
    // pub(super) fn parent_visible() {}
}
```

### Réexportation

`pub use` permet de réexporter les items d'autres modules :

```rust
mod inner {
    pub fn helper() {}
}

mod outer {
    // réexporter inner::helper
    pub use inner::helper;
}

fn main() {
    // accessible via outer::helper
    outer::helper();
}
```

### Relation entre modules et composants

> **Fonctionnalité prévue** : envisager de supporter les fonctionnalités export path et import path. C'est-à-dire non seulement autoriser les écritures `export ident;` `import ident;`, mais aussi autoriser les écritures `export ident1::ident2::ident3;` et `import ident1::ident2::ident3;`.

La principale considération est que, si les composants n'ont entre eux qu'une relation de juxtaposition, sans relation d'inclusion, ce n'est pas assez flexible.
La structure logique à l'intérieur d'un composant est arborescente ; si les composants n'ont pas de relation d'inclusion, seulement une relation de juxtaposition, alors lorsqu'un composant devient trop grand et que le temps de compilation est trop long, et qu'il faut le refactoriser, la refactorisation elle-même changerait forcément la structure logique du composant. Ce n'est pas approprié.

On peut envisager d'autoriser qu'un certain sous-arbre à l'intérieur d'un composant soit aussi un composant, et non seulement un module, ce qui aiderait à découper un grand composant en différentes unités de compilation, sans affecter la structure logique interne du composant.

Nous devons maintenir deux principes inchangés :
1. La structure logique d'un composant est un arbre, et non une forêt
2. Les composants ne peuvent pas avoir de dépendances circulaires entre eux

Par exemple, dans la figure ci-dessous, le composant c contient une série de sous-modules, c'est un composant de très grande taille.

<img src="./mod_tree.png" width="50%" align=center />

Pour accélérer la compilation, nous pouvons extraire certains de ses sous-modules à forte cohésion en composants indépendants, représentés par différentes couleurs dans la figure.
Le module e et tous ses sous-modules forment un composant ; le module g et tous ses sous-modules forment aussi un composant ; le module i et ses sous-modules forment aussi un composant. Les modules de même couleur dans la figure sont compilés ensemble ; il suffit de garantir qu'il n'y a pas de dépendance circulaire entre les différents composants. En même temps, la structure logique de l'ensemble du composant reste inchangée : le module racine du composant g a un nom comme `c::d::g`, et le mangle name de tous les items à l'intérieur conserve aussi ce nommage.


## Instruction use

L'instruction `use` sert à introduire dans la portée courante les items d'autres modules, pour éviter d'écrire à chaque fois le chemin complet.

### Usage de base

```rust
use std::collections::HashMap;

fn main() {
    let map = HashMap::new();
    // on peut utiliser HashMap directement, sans écrire le chemin complet
}
```

### Mots-clés de chemin

- `self` : accède au mod courant
- `super` : accède au mod de niveau supérieur du mod courant
- `::ident` : accède à un nom dans la portée globale, c'est-à-dire un nom de component. Les noms légitimes comprennent le nom défini dans export, les noms des dépendances importées, ainsi que la bibliothèque standard std.
- `::self` : accède au mod de premier niveau du component courant. Équivalent à accéder au nom de premier niveau de ce composant en commençant par `::ident`. Note : cela diffère de Rust. En Rust, on accède au mod de premier niveau via le mot-clé `crate::`.

### Exemple

```rust
mod outer {
    pub fn outer_fn() {}
    
    mod inner {
        pub fn inner_fn() {}
        
        fn example() {
            // utiliser self pour accéder au module courant
            self::inner_fn();
            
            // utiliser super pour accéder au module parent
            super::outer_fn();
        }
    }
}

// importer avec use
use outer::inner::inner_fn;

fn main() {
    inner_fn();
}
```

### Import avec renommage

Le mot-clé `as` permet de renommer un item importé :

```rust
use std::collections::HashMap as Map;

fn main() {
    let map = Map::new();
}
```

### Importer plusieurs items

```rust
use std::collections::{HashMap, HashSet};

fn main() {
    let map = HashMap::new();
    let set = HashSet::new();
}
```
