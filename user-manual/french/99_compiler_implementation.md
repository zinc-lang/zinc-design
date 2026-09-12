

## Format de fichier zno

Lors de la compilation d'un composant bibliothèque, le compilateur, outre le fichier binaire `.o` correspondant, doit aussi générer un « fichier d'en-tête » analogue à celui du C, contenant les signatures de toutes les interfaces publiques, pour que les bibliothèques en aval puissent l'utiliser.
Le « fichier d'en-tête » généré par Zinc a `.zno` comme extension de fichier, dans le sens d'« oxyde de zinc ».

Le contenu d'un fichier `.zno` comprend toutes les signatures de fonctions publiques, les définitions de types, les déclarations de variables globales, les déclarations impl, etc. Il comprend aussi le corps de toutes les fonctions pouvant être inlinées entre composants.

Le format d'un fichier `.zno` peut en principe être du texte pur, ou n'importe quel format binaire personnalisé. Mais pour accélérer la compilation, le choix final a été d'utiliser un fichier sqlite. Voici les raisons.

Considérons ce scénario : un grand projet contient de nombreux composants très complexes. Le composant a dépend du composant b, et b dépend à son tour d'autres composants c, d, e, etc.
Si l'on utilisait un simple format de fichier texte, alors à la compilation du composant a, il faudrait non seulement lire entièrement le fichier zno du composant b, mais aussi lire entièrement les fichiers zno de tous les autres composants dont b dépend, puis faire l'analyse syntaxique et sémantique.
La fonctionnalité `#include` des compilateurs C/C++ est implémentée ainsi.
Mais pour le système de modules de Zinc, copier directement la conception de C/C++ ne convient pas. Car C/C++ a le fichier comme unité de compilation, Zinc a le component comme unité de compilation, et le volume d'une unité de compilation est plus grand.
Par analogie, cela équivaut à fusionner tous les fichiers d'en-tête utilisés directement ou indirectement par un assez grand module C/C++ en un énorme fichier, et que les autres fichiers incluent à tout moment cet énorme fichier d'en-tête, au lieu de le découper manuellement en plusieurs petits fichiers d'en-tête.
Et il faut aussi considérer le besoin d'inlining entre composants : le corps de certaines fonctions doit aussi être enregistré dans ce fichier d'en-tête.
Pour un grand projet, un fichier d'en-tête trop volumineux est un grand gaspillage de temps de compilation, car la majeure partie du contenu de cette masse de zno n'est pas utilisée lors de la compilation du composant a.
Pourrait-on donc envisager de faire supporter à zno une lecture à la demande, et de ne lire, à la compilation de a, que ce qui est nécessaire ? Découper le fichier zno à la granularité de chaque « déclaration », plutôt que de le lire entièrement à la granularité du fichier.
Par exemple, si le composant a n'utilise simplement qu'une fonction `fn f();` du composant b, alors je sais que les autres déclarations de fonctions, définitions de types, etc. de b n'ont aucun rapport avec le composant a, et que les dépendances de b n'ont aucun rapport avec a.
Je peux extraire séparément cette signature de fonction du fichier `b.zno`, faire l'analyse syntaxique et sémantique, et ne pas m'occuper du reste. Quelle que soit la taille du fichier zno de b, quelle que soit la complexité de son arbre de dépendances, cela n'a aucun rapport avec la compilation de a.

Pour atteindre l'objectif ci-dessus, il me faut découper tout le fichier d'en-tête en plusieurs parties selon les différentes déclarations, et établir un index pour faciliter les requêtes : c'est précisément ce que sqlite peut aider à bien faire.

Les avantages d'utiliser le format sqlite :
1. sqlite est le format le plus commode pour supporter une « lecture à la demande » ; si nous concevions notre propre format de fichier pour atteindre le même effet, nous finirions probablement par faire un fichier au format sqlite plus simplifié, plus difficile à utiliser, et avec plus de bugs.
2. Vitesse de lecture-écriture élevée, taille de fichier également adaptée. Des outils d'aide existent déjà, matures et stables, commodes pour le débogage. Étendre les fonctionnalités en modifiant le schema ultérieurement est aussi très commode.
3. À l'étape actuelle, pour simplifier l'implémentation, on sauvegarde directement dans sqlite le code source sérialisé ; ultérieurement, on pourra aussi modifier pour sauvegarder le code intermédiaire IR, afin d'améliorer encore la vitesse de compilation. Et cela ne nécessite pas de modifier la conception d'ensemble.

Un fichier zno n'a pas besoin de se soucier de la lisibilité humaine, car ce fichier est destiné au compilateur, pas à la lecture humaine.
La fonctionnalité destinée à la lecture humaine devrait être incluse dans l'outil de génération de documentation ; nous devrions fournir un outil doc generator, générant une documentation au format HTML esthétique, avec une belle mise en page, de la coloration syntaxique, des hyperliens, une fonctionnalité de recherche, etc., commode pour la lecture humaine.

## Bootstrapping

La première version du compilateur Zinc était implémentée en C++, puis le frontend a été réimplémenté en Zinc (sans inclure, bien sûr, le framework LLVM utilisé comme backend). Cela a plusieurs objectifs :

1. Il y a un adage dans le monde de la programmation : « Eating your own dog food ». Le concepteur doit d'abord aimer le produit qu'il a conçu, et l'utiliser intensivement sur le long terme : cela permet de former efficacement une boucle Plan-Do-Check-Act, et aide à découvrir les problèmes plus tôt pour les améliorer. Je souhaite vivement que le langage de programmation de mon travail quotidien soit Zinc, l'utiliser directement pour écrire mon propre compilateur permet d'accumuler le plus vite possible une expérience d'usage de première main, et de découvrir et d'améliorer rapidement les problèmes. Que la première version n'ait pas été implémentée en Rust est aussi lié à cela, car les habitudes de pensée et les design patterns de Rust sont très différents de Zinc. Par exemple, de nombreuses structures de données internes d'un compilateur implémenté en Rust doivent remplacer les pointeurs par des index : cette façon de penser est très différente des autres langages. Puisqu'il était déjà décidé que l'on réécrirait plus tard en Zinc, autant écrire la première version en C++, afin que la deuxième version puisse migrer de façon transparente sans changer l'architecture principale.
2. Faciliter, une fois le projet open source, l'acceptation des contributions d'autres développeurs. Si le compilateur lui-même était implémenté en C++, face aux PR de contributeurs open source que je ne connais pas, j'aurais personnellement du mal à bien faire la revue de code et à maintenir la qualité du code. Alors que relire une fonctionnalité implémentée dans un langage de programmation sûr est beaucoup plus simple.
3. Mettre en pratique l'idée de compiler as a service. Comme chacun sait, un langage de programmation, ce n'est pas seulement implémenter un compilateur : il faut aussi tout un écosystème pour qu'il soit vraiment utile. La partie toolchain de cet écosystème comprend le support IDE, les outils de formatage automatique du code, les outils de gestion de paquets, les outils de génération de documentation, les alertes lint personnalisées, l'analyse de qualité du code, etc. Et nombre de ces outils ont besoin de partager la même logique que le compilateur. Donc, si nous pouvons faire en sorte que chaque partie du compilateur lui-même soit une library assez commode à appeler, avec une API sûre et facile à utiliser, alors construire les outils périphériques sera beaucoup plus efficace. À l'inverse, si le compilateur est une boîte noire, dont les composants ne peuvent pas être facilement réutilisés, alors construire les outils périphériques impliquera beaucoup de travail répétitif, et sera beaucoup moins efficace. Utiliser C++ comme interface API publique des différents sous-composants du compilateur n'est ni assez sûr, ni assez facile à utiliser.
4. Pose les bases des fonctionnalités ultérieures de métaprogrammation (macros, lint personnalisés, etc.). La métaprogrammation, du point de vue de l'implémentation, c'est en réalité une fonctionnalité de plugins du compilateur. Il faut que le compilateur charge dynamiquement certains modules, et exécute les fonctionnalités de ces plugins pendant la compilation. Nous souhaitons bien sûr que les définitions de macros elles-mêmes soient implémentées en Zinc. Et Zinc se prête justement assez bien à être compilé en bibliothèque dynamique ; l'API de la bibliothèque compilée peut aussi supporter des fonctionnalités avancées comme les génériques et les traits. Si le compilateur, en tant qu'appelant, est aussi implémenté en Zinc, alors ce mécanisme de plugins est très propre et concis. À l'inverse, charger dynamiquement une bibliothèque dynamique Zinc depuis C++ est certes possible, mais c'est clairement une complication superflue.

## Brève présentation de la structure du compilateur

Le compilateur se divise en ces composants :
* zinc_cli, programme exécutable, dépend des composants suivants. Il contient principalement les fonctionnalités de traitement de la ligne de commande.
    * options
    * session
* zinc_frontend, frontend du compilateur, responsable des vérifications syntaxiques et sémantiques ; il contient plusieurs sous-modules, dont les deux principaux sont :
    * zinc_syntax  l'entrée principale est le code source, la sortie est la structure de données SyntaxTree. Comprend les fonctionnalités d'erreur de syntaxe. Le lexer est aussi inclus dans ce composant.
    * zinc_semantic  l'entrée principale est la structure de données SyntaxTree, la sortie est la structure de données SemanticModel. Comprend les fonctionnalités d'erreur sémantique.
        * hir
        * mir
* zinc_backend, backend du compilateur, responsable de l'optimisation des performances et de la génération de code ; il contient plusieurs sous-modules, dont les deux principaux sont :
    * optimization  optimisation des performances, l'entrée est la structure de données SemanticModel, qu'il modifie et transforme
    * llvm_ir_gen  l'entrée principale est la structure de données SemanticModel, la sortie est LLVM IR. Il contient aussi quelques optimisations de performances.


Ils dépendent tous de zinc_std, qui est la bibliothèque standard de Zinc.

## Compilation incrémentale

TODO:

Les structures de données Copy-On-Write se prêtent particulièrement bien à l'implémentation de la compilation incrémentale. Combinées à une évaluation paresseuse Query-Based, elles permettent de tirer pleinement parti des résultats mis en cache de la compilation précédente.


## Pas de surcharge de fonctions basée sur le type des paramètres

L'objectif principal est de mieux supporter la publication de composants via des bibliothèques dynamiques.

Le scénario central est : lorsqu'un composant est publié à l'extérieur via une bibliothèque dynamique, nous souhaitons pouvoir, lors des mises à jour ultérieures de la version de la bibliothèque, que les utilisateurs en aval n'aient pas besoin de recompiler leur code, et puissent directement lier et utiliser.

Supposons maintenant que nous supportons la surcharge de fonctions basée sur le type des paramètres, et considérons ce cas :
lib_v1 a une fonction, de signature `fn f(arg: *Base);` ; dans la version lib_v2, nous voulons ajouter une version surchargée `fn f(arg: *Sub);`, où `Sub` est un trait héritant de `Base`.
Si l'utilisateur en aval ne recompile pas son code, et l'utilise avec lib_v2, le code peut toujours s'exécuter, mais il appellera toujours la version `Base`. Cette surcharge nouvellement ajoutée n'aura aucun effet.

Lorsque nous disons « pas de surcharge de fonctions basée sur le type des paramètres », le sens réel est « pas de résolution de version de fonction à la compilation basée sur le type des paramètres » : nous exigeons que l'utilisateur fasse la résolution de version de fonction à l'exécution.

Toujours dans le scénario ci-dessus, si l'utilisateur a besoin, dans lib_v2, de faire un traitement supplémentaire pour un paramètre de type `*Sub`, l'utilisateur doit modifier le corps de la fonction d'origine, et non ajouter une signature de fonction d'une nouvelle version surchargée :
```
fn f(arg: *Base) {
    if (arg is s:*Sub) {
        // tester par filtrage par motif : si arg est de type *Sub, exécuter la logique spéciale
    } else {

    }
}
```
Après publication de la bibliothèque dynamique de la nouvelle version lib_v2, le programme exécutable n'a pas besoin d'être recompilé, et le comportement de cette application se met automatiquement à jour.

Cela sacrifie bien sûr l'efficacité d'exécution. Mais pour réaliser une mise à jour transparente des bibliothèques dynamiques, ce coût à l'exécution est indispensable. Une résolution de surcharge à la compilation ne peut pas obtenir ce dynamisme.

Cette conception n'a pas de problème de pouvoir d'expression, seulement un problème de performance d'exécution. Et la performance d'exécution a une chance d'être en partie compensée par diverses mesures d'optimisation. En particulier, lorsque l'utilisateur n'a pas besoin de « publier une bibliothèque dynamique à ABI stable », on peut indiquer au compilateur via une option de compilation d'activer des optimisations plus agressives.

## Pas d'exceptions (exception), pas de déroulement de pile (unwind)

Zinc fournit le mot-clé `panic`, que l'on peut comparer au mot-clé `throw` d'autres langages. Mais ce n'est pas la même chose que le `throw` d'autres langages ou le `panic!` de Rust.

Le panic de Rust a deux modes de comportement, respectivement `abort` et `unwind`, que l'on peut spécifier via l'option de compilation suivante de `rustc`.
```
-C    panic=val -- panic strategy to compile crate with
```
* Le `panic` de Rust, dans la stratégie `abort`, termine directement le processus, la fonction `catch_unwind` n'a aucun effet. Équivalent à appeler la fonction `abort` de la libc.
* Le `panic` de Rust, dans la stratégie `unwind`, exécute un déroulement de pile (unwind), et appelle le destructeur des variables locales de chaque couche de fonction, y compris la décrémentation automatique du comptage de références des pointeurs ARC, etc., et il peut être capturé par `catch_unwind`, ce qui interrompt le processus de déroulement de pile. Ce processus est similaire aux exceptions d'autres langages.

Le `panic` de Zinc consiste simplement à imprimer la pile d'appels, puis à abort. Il n'a pas de processus unwind, il ne peut pas être capturé, et équivaut seulement au comportement du `panic` de Rust dans la stratégie `abort`. Zinc ne supporte pas le comportement `unwind`.

Cette conception s'inspire de ce [billet de blog](https://smallcultfollowing.com/babysteps/blog/2024/05/02/unwind-considered-harmful/) de nikomatsakis. Les raisons sont les suivantes :

1. unwind augmente la difficulté d'implémentation du compilateur et de la bibliothèque standard
    1. unwind augmente le volume de code. Car chaque appel de fonction peut provoquer un déroulement de pile ; le compilateur doit enregistrer, à chaque endroit où un déroulement de pile peut se produire, quels destructeurs de variables locales appeler.
    2. unwind réduit les opportunités d'optimisation. Le graphe de flot de contrôle du code devient plus complexe ; de nombreux endroits ont une arête de flot de contrôle supplémentaire où un unwind peut se produire, ce qui affecte l'analyse statique du compilateur.
    3. unwind exige que la bibliothèque standard, lors de l'implémentation, prenne en compte le problème de l'exception safety.
2. unwind augmente le coût d'apprentissage et d'usage pour l'utilisateur
    1. Zinc, en tant que langage tardif, a du mal à établir un écosystème indépendant. Dans la plupart des applications, on n'utilisera probablement Zinc que pour développer l'un des modules, ce qui signifie que l'interopérabilité est importante. Si Zinc introduisait un mécanisme unwind, quelle serait sa relation avec les exceptions de C++ et le panic de Rust, pourraient-ils se capturer mutuellement ? Si nous souhaitons que des modules de bas niveau écrits en Zinc puissent être appelés par des langages de haut niveau comme Go/C#/Java/Javascript/Python, ces langages de haut niveau peuvent-ils capturer le panic lancé par Zinc ? Ce mécanisme est-il, pour l'utilisateur, une commodité, ou une source de confusion ?
    2. Introduire un mécanisme unwind dans le langage exige que toutes les bibliothèques tierces, lors de la conception, prennent en compte le problème de l'exception safety. Surtout lorsqu'il s'agit d'unsafe et d'interopérabilité, c'est difficile. Zinc se positionne sur la simplicité et la facilité d'utilisation ; exiger d'un développeur ordinaire qu'il considère aussi la conception exception safety, n'est-ce pas un peu trop ?
    3. Quels scénarios de gestion d'erreurs sont peu commodes à exprimer avec `Option` `Result`, et exigent forcément un unwind capturable ? Si nous implémentons cette fonctionnalité, quels scénarios en bénéficieraient, et ces scénarios sont-ils suffisamment convaincants ?

Je pense que l'écosystème de Zinc devrait se construire ainsi : toutes les « erreurs récupérables » retournent systématiquement un type `Option/Result` ; ce type d'erreur signifie que le processus courant, confronté à cette erreur, a des moyens de la traiter. Toutes les « erreurs irrécupérables » utilisent systématiquement `panic` ; ce type d'erreur signifie que l'application courante, à l'exécution, n'a aucun moyen de traiter cette erreur : c'est un bug dans le code, la seule solution est de revenir modifier le code, recompiler le programme, et le réexécuter.

Si quelqu'un d'autre n'est pas d'accord avec cette décision, il faut rédiger une proposition de conception complète pour ouvrir la discussion, dont le contenu devrait répondre de façon suffisante aux questions ci-dessus.

## Génériques
