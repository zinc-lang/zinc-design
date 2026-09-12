# Notions de base

## hello world

Un programme Zinc de base ressemble à ceci :

```rust
fn main() {
    println("hello world");
}
```

Les fichiers source se terminent par l'extension `.zn`, et leur contenu doit être encodé en utf8. La commande de compilation est :

```
zinc main.zn
```

Une fois l'exécution terminée, on voit dans le dossier courant un nouvel exécutable généré. En l'exécutant, on peut afficher la chaîne `hello world`.

## Commentaires

Zinc prend en charge deux types de commentaires : les commentaires de ligne et les commentaires de bloc.

1. Commentaires de ligne

    Un commentaire de ligne commence par `//` et se poursuit jusqu'à la fin de la ligne.

2. Commentaires de bloc

    Un commentaire de bloc commence par `/*` et se termine au premier `*/`.

> **TODO** : les commentaires de documentation ne sont pas encore implémentés.

## Fonctions

Une définition de fonction Zinc ordinaire commence par le mot-clé `fn`, suivi du nom de la fonction et d'une paire de parenthèses ; à l'intérieur des parenthèses se trouve la liste des paramètres. Après les parenthèses, on peut écrire le type de retour. Vient ensuite le corps de la fonction, entouré d'accolades.

Exemple :

```rust
fn my_func(arg1: Int32, arg2: String) -> Bool {
    return true;
}
```

Si un projet Zinc doit produire un exécutable, il doit définir dans le module racine une unique fonction `main`, qui sert de point d'entrée du programme.

## Variables et types

Identifiants : commencent par `_` ou une lettre, puis peuvent contenir `_`, des chiffres ou des lettres.

Un `_` isolé est un identifiant spécial, qui signifie « ignorer ».

> **TODO** : les identifiants unicode et les raw identifiers ne sont pas encore supportés, à implémenter.

### Variables locales

Les variables locales n'ont pas besoin d'un type explicite ; l'inférence de type est autorisée.

Si le nom de variable n'est pas décoré par `mut`, elle est immuable par défaut.

```rust
fn main() {
    let x = 5_i;
    x = 6; // erreur, x est immuable
}
```

Pour la lisibilité, les variables locales d'un même block **ne** peuvent **pas** avoir le même nom. Les variables locales de blocks différents peuvent bien sûr avoir le même nom.

### Variables statiques

La définition d'une variable statique est décorée par le mot-clé `static`.

Le type d'une variable statique doit satisfaire la contrainte `Send` (noter que les types qui satisfont `Send` diffèrent de Rust) ; voir le chapitre « Parallélisme et concurrence ». Pour utiliser un type non-Send comme variable static, il faut la décorer par `unsafe`.

Une variable statique doit être initialisée à la définition, et l'expression d'initialisation ne peut être qu'une « expression constante ».

Une variable statique peut porter le modificateur `mut`. Mais une variable static `mut` doit être décorée par `unsafe`. Sa lecture et son écriture doivent se faire dans un contexte unsafe.

```rust
unsafe static mut G: Int32 = 1; // une variable static décorée par mut, ou une variable static dont le type ne satisfait pas Send, doit être décorée par unsafe

fn main() {
    println(G.to_string()); // erreur, G ne peut être utilisé que dans un contexte unsafe
}
```

> **Fonctionnalité prévue** : l'inférence de type pour les variables static pourra être supportée ultérieurement.

### Constantes

La définition d'une constante est décorée par le mot-clé `const`.

Comme pour les variables statiques, le type d'une constante doit satisfaire la contrainte `Send`. Une constante doit être initialisée à la définition, et l'expression d'initialisation ne peut être qu'une « expression constante ».

Une constante ne peut pas porter le modificateur `mut`.

```
const PI: F32 = 3.14;
```
> **Fonctionnalité prévue** : l'inférence de type pour les constantes const pourra être supportée ultérieurement.
