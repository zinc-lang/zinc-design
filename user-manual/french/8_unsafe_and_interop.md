# unsafe et interopérabilité

## Mot-clé unsafe

Zinc utilise le mot-clé `unsafe` pour marquer le code susceptible de produire un comportement indéfini. Utiliser `unsafe` permet de dire au compilateur : « je sais que ce code peut être unsafe, mais je garantis qu'il est correct. »

### Cas d'usage de unsafe

Les opérations suivantes doivent se faire dans un contexte unsafe :

1. **Appels de fonctions, appels de méthodes**
   - Fonctions décorées par `extern`
   - Fonctions décorées par `unsafe`

2. **Déréférencement de pointeurs nus**
   - Déréférencer un pointeur de type `*raw T` ou `*raw mut T`

3. **Conversions de type**
   - Conversion d'un pointeur nu vers n'importe quel type
   - Conversion entre entiers et pointeurs

4. **Accès aux membres d'une union**
   - Lire et écrire les membres d'un type union

5. **Accès à mut static**
   - Lire et écrire une variable globale définie par `static mut`

### Blocs unsafe

Utiliser un bloc `unsafe` permet de créer un contexte unsafe :

```rust
fn main() {
    let x: Int = 42;
    let p: *raw Int = &x as *raw Int;
    
    unsafe {
        let value = *p; // déréférencer un pointeur nu, nécessite unsafe
        println(f"value = $(value)");
    }
}
```

### Fonctions unsafe

Une fonction définie par `unsafe fn` doit être appelée dans un contexte unsafe :

```rust
unsafe fn dangerous_operation() {
    // code susceptible de produire un comportement indéfini
}

fn main() {
    unsafe {
        dangerous_operation(); // appel d'une fonction unsafe
    }
}
```

### unsafe impl

Utiliser `unsafe impl` permet d'implémenter manuellement certains marker traits, comme `Send` et `Sync` :

```rust
struct MyType {
    ptr: *raw mut Int,
}

// déclarer manuellement que MyType est Send
// le programmeur doit garantir que cette déclaration est correcte
unsafe impl Send for MyType {}
```

## FFI (interface de fonctions externes)

### Fonctions extern

Le mot-clé `extern` permet de déclarer une fonction externe, généralement pour appeler une bibliothèque C :

```rust
// déclarer la fonction printf de la bibliothèque standard C
extern fn printf(fmt: *const UInt8, ...) -> Int32;

fn main() {
    unsafe {
        printf("Hello from C!\n" as *const UInt8);
    }
}
```

### Blocs extern

Un bloc `extern` permet de déclarer des fonctions externes en lot :

```rust
extern "C" {
    fn malloc(size: UInt) -> *raw mut UInt8;
    fn free(ptr: *raw mut UInt8);
    fn strlen(s: *const UInt8) -> UInt;
}

fn main() {
    unsafe {
        let ptr = malloc(100);
        // utiliser ptr...
        free(ptr);
    }
}
```

### Interopérabilité avec le langage C

Zinc fournit plusieurs moyens d'interopérer avec le langage C :

1. **Appeler une fonction C** : déclarer une fonction C avec `extern`, puis l'appeler dans un bloc unsafe
2. **Transmettre des données à C** : utiliser un type pointeur nu pour transmettre des données
3. **Recevoir des données depuis C** : utiliser un pointeur nu pour recevoir les données retournées par C

Exemple :

```rust
// déclarer des fonctions C
extern fn fopen(filename: *const UInt8, mode: *const UInt8) -> *raw mut UInt8;
extern fn fclose(file: *raw mut UInt8) -> Int32;

fn main() {
    unsafe {
        let file = fopen("test.txt\0" as *const UInt8, "r\0" as *const UInt8);
        if file != null {
            // utiliser le fichier...
            fclose(file);
        }
    }
}
```

## Frontière entre le sûr et l'unsafe
