# Préface

<img src="../../assets/zinc-logo.png" width="10%" align=center />

Zinc est un langage de programmation système inspiré de Rust, conçu pour être plus facile à utiliser.

## Caractéristiques

1. Typage statique, runtime léger, sans GC
  * Sûreté des types
  * Compilation directe en code machine, sans machine virtuelle, démarrage rapide
  * Pas de GC, faible overhead mémoire
  * Contrôle de la disposition mémoire
  * Backend basé sur le framework LLVM, ce qui facilite le multiplateforme
  * Interopérabilité aisée

2. Sécurité
  * Sécurité mémoire par défaut : seul `unsafe` peut produire un comportement indéfini
  * Thread-safety par défaut : seul `unsafe` peut produire une data race

3. Facilité d'utilisation
  * Ne vise pas le zero-cost abstraction
  * Pas de types à sémantique move : seule l'expression peut avoir une sémantique move
  * Des types borrow existent, mais il n'y a pas de borrow checker, ce qui évite l'impact massif de ownership + borrow checker sur le style de programmation, et améliore considérablement l'ergonomie par rapport à Rust

4. Les fonctions génériques sont principalement implémentées par passage de table (dictionary passing)
  * Possibilité de publier des composants via des bibliothèques dynamiques, avec des interfaces complexes, y compris les génériques
  * Les fonctions `virtual` peuvent porter des paramètres génériques
  * À terme, l'objectif est un ABI stable, permettant de mettre à jour un composant isolément sans recompiler les dépendances en aval
  * Une option de compilation permet, lorsque la stabilité ABI n'est pas requise, d'instancier les fonctions génériques pour gagner en performance

### Comparaison avec Rust

1. Syntaxe proche de Rust, style de bibliothèque standard proche de Rust. Même support de la sécurité mémoire et de la thread-safety.
2. La gestion mémoire repose principalement sur le comptage de références, et non sur le mécanisme d'ownership.
3. Meilleure ergonomie : le borrow est supporté, mais le borrow checker est supprimé ; les expressions à sémantique move sont supportées, mais pas les types à sémantique move.
4. Meilleur support du paradigme orienté objet
5. Implémentation des génériques revue : instanciation et passage de table (dictionary passing) sont tous deux supportés, au choix via une option de compilation
6. Certaines constructions syntaxiques sont simplifiées

### Comparaison avec Swift

1. Meilleur support de la thread-safety, garantie à la compilation
2. Neutre vis-à-vis des plateformes, macOS n'est pas la plateforme principale
3. Abandon de l'interopérabilité avec Objective-C, davantage d'attention portée à l'interopérabilité avec C/C++
4. Support différent du paradigme OOP : par exemple, pas de `class` ni d'héritage de classes
5. Support différent des fonctionnalités move / borrow

### Comparaison avec C++

1. Zinc est plus sûr : sans `unsafe`, il n'y a pas de comportement indéfini
2. Implémentation des génériques différente : une fonction générique Zinc peut servir de fonction virtuelle, et peut aussi être compilée en binaire et publiée via une bibliothèque dynamique
3. À terme, Zinc peut atteindre un ABI stable
4. Les performances d'exécution sont nettement inférieures à celles de C/C++, mais devraient suffire aux besoins du développement d'applications
5. Fonctionnalités de langage plus simples, moins de bagage historique, meilleur support du dynamisme à l'exécution, plus petit volume de code

### Comparaison avec Go

1. Pas de GC, exigences plus faibles en ressources mémoire
2. Meilleur support des fonctionnalités de langage, notamment génériques, trait, enum, filtrage par motif, etc.
3. Mieux adapté à la publication de composants via des bibliothèques dynamiques
4. Meilleur support de la thread-safety
5. Meilleure capacité d'interaction inter-langages. Go embarque un GC, ce qui rend difficile l'interopérabilité avec d'autres langages qui ont aussi un GC.

## Foire aux questions

### 1. Quel est le positionnement du langage Zinc ?

L'intention initiale de Zinc est d'améliorer l'ergonomie de Rust.

Dans la conception des langages système, trois critères d'évaluation sont courants : sécurité, performance, facilité d'utilisation. Rust pousse sécurité et performance à l'extrême, mais l'ergonomie reste un point douloureux.
Zinc conserve la sécurité tout en renonçant à la performance extrême, pour gagner en facilité d'utilisation.

Ce choix vient de l'observation suivante : dans l'immense majorité des situations rencontrées par un développeur ordinaire, les performances CPU ne sont pas le goulot d'étranglement ; les cas où l'on doit extraire le maximum du CPU restent rares.
Le plus souvent, on choisit un langage système sans garbage collection non pour la performance extrême, mais pour d'autres raisons, notamment :

* Un fort contrôle de la disposition mémoire, et une prévisibilité des performances d'exécution
* Un besoin de multiplateforme, le portage d'un runtime de machine virtuelle étant difficile sur certaines plateformes
* Des ressources mémoire trop limitées sur certaines plateformes pour un garbage collection automatique
* Le souhait d'écrire des bibliothèques de composants communs, appelables depuis divers langages de haut niveau qui utilisent des mécanismes de GC différents. Un langage avec son propre GC a du mal à interopérer efficacement, dans un même processus, avec un langage dont le GC est différent

Un langage ainsi positionné a donc des cas d'usage pertinents :
1. Pas de garbage collection automatique, ce qui facilite le multiplateforme et l'interopérabilité avec divers langages de haut niveau. Convient à l'écriture de modules communs et génériques.
2. Une très bonne ergonomie, tout en garantissant la sécurité. Exigences plus faibles pour les développeurs, productivité plus élevée.
3. Un niveau de performance correct, sans chercher la performance extrême ni extraire tout le potentiel du matériel.

Un tel design ne correspond pas au positionnement de Rust ; ces modifications ne peuvent donc se faire qu'en lançant un nouveau langage.

Le positionnement de Zinc est proche de celui de Swift : les points centraux sont sans garbage collection + sécurité + facilité d'utilisation.
Il possède en plus l'avantage de la thread-safety, que Swift n'a pas.

### 2. Comparaison entre Zinc et Rust ?

* Rust tient au principe de « zero-cost abstraction » ; Zinc l'abandonne. Lorsque performance d'exécution et ergonomie sont incompatibles, l'ergonomie l'emporte.
* Rust garantit la sécurité mémoire par le mécanisme « ownership + lifetimes » ; le borrow checker vérifie à la compilation aliasing XOR mutation. Zinc garantit la sécurité mémoire par le comptage de références et le borrow ; les règles de vérification à la compilation sont beaucoup plus souples. Supprimer le borrow checker améliore considérablement l'ergonomie de Zinc par rapport à Rust.
* Rust implémente les génériques par instanciation ; Zinc les implémente par passage de table. Cela signifie que Rust n'est pas adapté à la publication de code générique sous forme de bibliothèque binaire, alors que Zinc l'est. Le support des bibliothèques dynamiques et la stabilité ABI sont des objectifs importants pour la suite de Zinc.
* Rust garantit la sécurité mémoire et la thread-safety ; Zinc aussi. Sur le plan de la sécurité, il n'y a pas d'affaiblissement, seulement un certain coût en performance.

En résumé, au niveau syntaxe et sémantique, Zinc est très proche de Rust ; au niveau de l'exécution, Zinc est très proche de Swift.

### 3. Pourquoi le nom Zinc ?

Rust évoque la rouille et la corrosion ; Zinc évoque le métal zinc. Ce métal a une particularité :

>
> Le zinc réagit facilement avec l'oxygène de l'air et forme une couche d'oxyde de zinc, c'est-à-dire une couche de passivation à la surface du lingot.
> Cette couche de passivation a un rôle protecteur et inhibe la poursuite de l'oxydation et de la corrosion.
> L'application principale du zinc est la protection anticorrosion du fer par galvanisation.
>
> Zinc is most commonly used as an anti-corrosion agent, and galvanization (coating of iron or steel) is the most familiar form.
> (from: https://en.wikipedia.org/wiki/Zinc)
>

Ce mot reflète l'essence du langage : Zinc ressemble superficiellement à Rust, mais sous la syntaxe de surface, son noyau conceptuel est incompatible avec Rust.

Par coïncidence, ce mot se prononce assez naturellement en chinois, et est homophone de « nouveau langage ».

### 4. Pourquoi le comptage de références (reference counting, RC) comme principal mécanisme de gestion mémoire ?

Avantages du RC :
* Un avantage du RC est qu'il permet d'implémenter facilement la sécurité mémoire. Le fondement théorique de la sécurité mémoire est « aliasing xor mutation ». Le RC est particulièrement adapté au suivi de l'aliasing ; combiné au copy-on-write, on obtient une très bonne sécurité.
* Le RC s'accorde particulièrement bien avec la conception de Zinc en matière de thread-safety. Thread-safety et gestion mémoire par RC se renforcent mutuellement. C'est la raison principale pour laquelle Zinc ne peut pas utiliser un GC. Avec un GC, Zinc ne pourrait pas atteindre le niveau actuel de thread-safety.
* Un autre très bon avantage du RC est sa simplicité. Cela implique peu de contraintes sur l'environnement extérieur, donc l'interopérabilité avec n'importe quel autre langage est simple. Le GC a des détails d'implémentation trop complexes ; différents mécanismes de GC coexistent difficilement efficacement dans un même processus. Faire interopérer un langage avec GC avec un autre langage dont le GC est différent, c'est se compliquer la vie.

Inconvénients du RC :
* Un inconvénient du RC est qu'il ne résout pas les fuites dues aux références circulaires. Mais ce n'est pas un problème de sécurité mémoire ; son impact est relativement limité. Et dans le périmètre des langages système, les autres langages, y compris C++/Rust/Swift, n'ont pas non plus résolu ce problème. Il suffit que l'allocateur mémoire et le débogueur collaborent bien pour que le problème soit assez facile à détecter et à résoudre pendant le développement et les tests. Ajouter un mécanisme de mark-and-sweep au runtime uniquement pour résoudre ce problème coûterait trop cher, et ne correspond pas au positionnement de Zinc. Je ne m'oppose pas, en revanche, à implémenter un scanner de tas utilisable uniquement en phase de test, combiné aux tests automatisés, pour aider à détecter les références circulaires dans les cas de test.
* Un autre inconvénient du RC est que, si l'on utilise trop fréquemment des instructions atomiques pour incrémenter et décrémenter le comptage de références, les performances d'exécution ne sont pas bonnes.

Mais on peut utiliser quelques optimisations pour récupérer un peu de performance :
1. Zinc encourage naturellement l'usage des types à sémantique de valeur ; dans de nombreux cas, aucune allocation dynamique n'est nécessaire, et le contrôle de la disposition mémoire des objets est assez bon. Cela est favorable au cache, et réduit aussi les situations où un pointeur à comptage de références apparaît.
2. Zinc conserve les types pointeurs borrow. La copie, le passage en paramètre et le retour d'un pointeur borrow ne modifient pas le comptage de références. Dans de nombreux cas, un pointeur borrow est plus raisonnable qu'un pointeur RC.
3. Zinc conserve la sémantique move ; l'utiliser correctement peut fortement réduire la fréquence des opérations de comptage de références.
4. Les pointeurs RC de Zinc sont des types intégrés au langage, et non simulés par une bibliothèque. Le compilateur a donc l'occasion, dans certains cas, d'éliminer par optimisation les incrémentations et décrémentations redondantes du comptage de références.
5. L'utilisateur peut remplacer le malloc/free de la libc par un allocateur plus avancé (comme mimalloc), pour améliorer l'efficacité de l'allocation et de la libération dynamiques.
6. Grâce aux fonctionnalités de thread-safety du langage, le compilateur peut analyser que certains pointeurs RC et toutes leurs copies ne peuvent être utilisés qu'à l'intérieur du thread courant, et donc traduire les incrémentations et décrémentations correspondantes en instructions non atomiques. En Zinc, un pointeur ne peut être utilisé de façon sûre entre threads que s'il pointe vers un type satisfaisant la contrainte `Sync`. Si un pointeur pointe vers un type non-`Sync`, on peut alors être certain que ce pointeur et toutes ses copies ne peuvent pas être utilisés entre threads : il y a là une opportunité d'optimisation, en utilisant des instructions non atomiques. **C'est aussi pourquoi on l'appelle pointeur à comptage de références automatique (automatic reference counting) : la lettre A représente Automatic et non Atomic. Le sens principal de « automatique » est que le compilateur choisit automatiquement entre instructions atomiques et non atomiques.**

Ces différences montrent que les conclusions de performance des mécanismes de comptage de références basés sur le atomic-rc traditionnel, dans d'autres langages, ne s'appliquent pas à Zinc. Comparé au atomic-rc traditionnel, Zinc a un meilleur potentiel d'optimisation.

En résumé, sécurité, performance et facilité d'utilisation forment un triangle impossible.
Lorsque Zinc choisit sécurité et facilité d'utilisation, il est inévitable de ne pas atteindre le « zero-cost abstraction » ; une certaine perte de performance par rapport à C/C++ est acceptable.
Nous disposons néanmoins d'une série d'optimisations pour que les performances d'exécution ne soient pas trop mauvaises.

Personnellement, je pense que Arc n'est pas le goulot de performance de Zinc : c'est le schéma d'implémentation des génériques qui l'est.

### 5. Quel est l'objectif de ce projet ?

Les qualités de conception de Rust dans de nombreux domaines sont évidentes. En même temps, les appels à simplifier l'apprentissage et l'usage de Rust n'ont jamais cessé.
J'ai toujours pensé que, dans le périmètre des langages système sûrs, la direction de conception de Rust n'est pas la seule solution, et qu'il reste d'autres espaces de conception.
Le but de ce projet est d'explorer d'autres possibilités de conception dans le périmètre des « langages de programmation système sûrs ».

Mais si l'on se contente de parler en général de ce que l'on pourrait concevoir autrement, personne n'écoute. Dans le milieu des programmeurs, une phrase est très célèbre :
> 
> Talk is cheap, show me your code.
>

Il me faut donc un projet qui réalise vraiment toutes ces idées, pour que ce soit convaincant.

La direction de conception centrale du projet est :
**À quoi cela pourrait ressembler si l'on conserve la sécurité de Rust, que l'on sacrifie un peu de performance d'exécution, et que l'on gagne une meilleure ergonomie.**

Sur cette base, il y a aussi quelques objectifs secondaires :
1. Simplifier les fonctionnalités du langage, s'opposer à l'empilement de fonctionnalités. Rendre les fonctionnalités aussi orthogonales que possible, sans qu'elles s'influencent mutuellement, afin qu'elles puissent se combiner librement. Éviter autant que possible de patcher les règles sémantiques et d'introduire des règles spéciales. Alléger la charge cognitive de l'utilisateur.
2. Simplifier l'implémentation du compilateur. Le compilateur Rust est aujourd'hui trop complexe : très peu de personnes le comprennent entièrement. Un projet aussi complexe est difficile à faire progresser et à faire évoluer durablement. Si une fonctionnalité de langage est trop complexe à implémenter, cela signifie souvent que les utilisateurs du langage ne pourront pas non plus la maîtriser complètement : il faut alors éviter d'introduire une telle fonctionnalité dès le départ.
3. S'intéresser à la vitesse de compilation des grands projets, et fournir de bonnes fonctionnalités d'aide IDE. Ce n'est pas seulement un problème d'implémentation du compilateur, c'est aussi un problème de conception des fonctionnalités du langage.
4. Mettre en pratique l'idée de compiler-as-a-service. Le compilateur ne doit pas être une boîte noire : ses différents composants doivent être exposés à la communauté comme des bibliothèques publiques. Chaque composant du compilateur doit avoir une API publique claire et facile à utiliser. Cela stimulera grandement le développement de l'écosystème périphérique.
5. Un meilleur support du dynamisme, fondé sur un langage compilé. Cela inclut : publier des fonctions génériques comme artefacts binaires, utiliser des fonctions génériques comme fonctions virtuelles, publier des bibliothèques dynamiques comme artefacts binaires, stabilité ABI, chargement dynamique, un certain degré de réflexion à l'exécution, etc. Ce sont des domaines où Rust n'excelle pas particulièrement, mais où Swift se comporte mieux.

### 6. Zinc n'a rien de particulièrement innovant : ce n'est qu'un réarrangement de fonctionnalités déjà présentes dans d'autres langages. Est-ce utile ?

Oui, c'est utile. Car le but de ce projet n'est pas l'innovation, c'est la sécurité et la facilité d'utilisation. Si le résultat de cette combinaison de fonctionnalités est que l'utilisateur ressent sécurité, facilité d'utilisation et adéquation à ses cas d'usage, cela suffit. Satisfaire les besoins des utilisateurs est mon objectif, pas innover pour innover.

Dans le domaine de la programmation système, les langages disponibles ne sont pas très nombreux. L'auteur estime qu'il existe encore certains cas où nous n'avons pas de langage vraiment adapté.

* C/C++ est très puissant, mais lorsqu'un comportement indéfini apparaît dans le code, trouver le bug dans une base de code à grande échelle est extrêmement douloureux. Pour un utilisateur ordinaire, la grande majorité des cas n'exigent pas une performance extrême ; sacrifier un peu de performance d'exécution pour gagner en sécurité est alors un choix rentable.
* Rust est un très bon langage, mais certaines fonctionnalités sont trop complexes, le paradigme de programmation n'est pas assez amical pour beaucoup d'utilisateurs, la productivité de développement est faible, les bibliothèques dynamiques sont mal supportées, et il n'est pas adapté aux scénarios qui nécessitent du dynamisme.
* Swift est également très bon : en tant que langage natif, il a un bon support du dynamisme, et il supporte aussi la compatibilité binaire — ce sont d'excellentes propriétés. Mais il est contrôlé par Apple, et se concentre essentiellement sur les plateformes Apple. Dans de nombreux cas, on voudrait l'utiliser, mais on ne le peut pas. Il a également des problèmes de thread-safety.
* Zig, en tant que langage système émergent, n'est pas encore assez mature. Il a également des problèmes de sécurité mémoire et de thread-safety.
* Zinc, lui, a été créé en absorbant autant que possible les avantages de Rust et de Swift. Premièrement, il s'aligne sur Rust en matière de sécurité, en garantissant à la compilation la sécurité mémoire et la thread-safety ; deuxièmement, il simplifie les fonctionnalités du langage et abaisse la difficulté d'utilisation ; troisièmement, il offre un meilleur support du dynamisme, et une évolution binaire compatible des bibliothèques dynamiques.

En ingénierie, il n'y a pas de magie, seulement des compromis selon les objectifs. Trouver un cas d'usage assez courant et concevoir des fonctionnalités de langage qui y répondent : tel est le principal défi de conception de Zinc.
L'auteur estime que la demande « sans GC & sécurité & facilité de développement » est assez courante, et que le marché manque d'un langage suffisamment compétitif pour y répondre pleinement.
Les cas où l'on vise la « performance extrême » et le « zero-cost abstraction » sont très rares en pratique ; pour les projets qui font de la performance la priorité absolue, il est recommandé d'utiliser C/C++/Rust.
La conception de Zinc ne devrait pas traiter ce type de besoin comme une priorité de premier rang.

Si vous trouvez cette conception utile, n'hésitez pas à laisser une étoile, à proposer des suggestions, ou à créer une PR pour participer à la conception et à l'implémentation de ce langage.

### 7. Quel est l'état actuel de Zinc ?

Les fonctionnalités centrales telles que la gestion mémoire et la thread-safety sont déjà conçues ; il existe une bibliothèque standard de base. Et, sur ces fondations, le compilateur a été bootstrapé.

L'état actuel peut être considéré comme une preuve de concept (proof of concept) : ce n'est qu'un début, encore loin d'être réellement utilisable.
De nombreuses fonctionnalités de langage essentielles, dès lors qu'elles n'empêchaient pas le bootstrapping, n'ont pas été implémentées. Les optimisations de performance prévues, ainsi que les outils périphériques et le parachèvement de la bibliothèque standard, n'ont pas encore été faits.
La poursuite du développement et de la maturation représente encore une charge de travail importante, et ne sera possible qu'avec l'aide de la communauté open source.

### 8. Dans quels cas Zinc est-il assez adapté ? Comment le promouvoir ?

Je pense qu'un nouveau langage a énormément de mal à occuper le terrain de langages anciens dont l'écosystème est déjà mature. En revanche, si l'on trouve un secteur encore à l'état naissant, que l'on y approfondit de nouveaux scénarios et que l'on construit un nouvel écosystème, alors la difficulté de promotion est nettement plus faible.
Élargir l'incrément, plutôt que de se disputer l'existant, peut fortement réduire le coût de promotion.

Quelques nouveaux scénarios prometteurs :

1. Produits automobiles intelligents et connectés
2. Scénarios robotiques
3. Appareils IoT, systèmes embarqués, divers petits appareils intelligents
4. Domaine de l'intelligence artificielle


### 9. Quelle licence open source Zinc a-t-il choisie ?

Zinc a choisi la licence open source la plus permissive, MIT. Car à notre époque, un langage de programmation insuffisamment ouvert ne peut pas attirer assez de participants communautaires : il est voué à l'échec, et n'a aucun sens.
Je souhaite que ce projet soit non seulement open source, mais aussi ouvert. Je suis très disposé à écouter vos avis et suggestions, et j'invite chacun à participer à la conception et à l'implémentation de Zinc.
