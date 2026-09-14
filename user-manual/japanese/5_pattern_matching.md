
# パターンマッチ

パターンマッチは主に `is` 式と `match` 文の場面で使う。

パターンマッチの目的は、ある式の構造が期待どおりかどうかを検査することであり、内部の値を取り出して新しい変数に束縛することも許す。

例は次のとおり：

```
// 変数 x は 3 要素の tuple であり、2 番目の要素を middle 変数に束縛し、さらに middle > 5 が成立するかを判定する
if (x is (_, middle, _)) && middle > 5 {

}

// 変数 y の型は trait を指すポインタであり、この仕方で y が実際に指す具体型が何かを判定できる。判定が成功すれば、このポインタは p に束縛される。
if y is p: *S {
    // p can be used inside block
}

// 変数 z は Option<S> 型であり、以下で Some と None の二通りの場面をそれぞれ処理する
match z {
    Some(s) => {

    }
    _ => {}
}
```

式と同様に、パターンも入れ子にできる。

## WildcardPattern

プレースホルダパターン。

`_` 下線は一つの位置を占められる。

`...` 省略記号は複数の位置を占められる。

## LiteralPattern

整数、浮動小数点数、Bool、Char などの型のリテラルを、直接パターンとして使うことを許す。

`Str` リテラルパターンは当面未実装である。

## TypePattern

型パターンは主に、ある式がある具体的な部分型かどうかを判定するために使う。型パターンは `type` キーワードで始まる。

例：
```
// TR が trait 名、S が struct 名であると仮定する。
// 次の文は、trait を指すポインタが、S を指すポインタへダウンキャストできるかどうかを判定するために使う。
fn test(p: &TR) {
    let b = p is type &S;  // type キーワードで始めるのは、後ろの modifier+identifier パターンとの構文衝突を避けるためである
}
```

## StructPattern

構造体パターンは構造体型の値をマッチし、そのメンバ変数を分解できる。

例：
```rust
struct Point { x: Int, y: Int }

fn test(p: Point) {
    // 構造体をマッチし、メンバ変数を新しい変数に束縛する
    if p is { .x = a, .y = b }: Point {
        println(f"x = $(a), y = $(b)");
    }
    
    // 省略記号で他のメンバを無視する
    if p is { .x = 10, ... }: Point {
        println("x is 10");
    }
    
    // 入れ子のパターンマッチ
    match p {
        { .x = 0, .y = 0 } => println("origin");
        { .x = 0, .y = y } => println(f"on y-axis at $(y)");
        { .x = x, .y = 0 } => println(f"on x-axis at $(x)");
        { .x = x, .y = y } => println(f"point ($(x), $(y))");
    }
}
```

## TuplePattern

タプルパターンはタプル型の値をマッチし、その要素を分解できる。

例：
```rust
fn test(t: (Int, String, Bool)) {
    // タプルをマッチし、要素を新しい変数に束縛する
    if t is (num, str, flag) {
        println(f"num = $(num), str = $(str), flag = $(flag)");
    }
    
    // 下線で一部の要素を無視する
    if t is (1, _, true) {
        println("first element is 1 and third is true");
    }
    
    // 入れ子のタプルパターン
    let nested = ((1, 2), (3, 4));
    if nested is ((a, b), (c, d)) {
        println(f"($(a), $(b)), ($(c), $(d))");
    }
}
```

## EnumVariantPattern

列挙変体パターンは列挙型の特定の変体をマッチし、その関連値を分解できる。

例：
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

修飾子パターンは、パターンマッチ中で変数の束縛方式を制御するために使う。

- `mut`：束縛した変数を可変と印す
- `&`：参照をマッチする
- `&mut`：可変参照をマッチする
- `ref`：移動ではなく参照によって束縛する
- `ref mut`：可変参照によって束縛する

例：
```rust
fn test(s: String) {
    // mut 修飾子：束縛した変数を可変にする
    let mut x = s;
    x.push_str(" modified");
    
    // ref 修飾子：参照によって束縛し、移動を避ける

    // ref mut 修飾子：可変参照によって束縛する

}
```

## IdentifierPattern

識別子パターンは、マッチした値を一つの変数名に束縛するために使う。

例：
```rust
fn test(x: Int) {
    // 単純な識別子束縛
    if x is y {
        println(f"y = $(y)");
    }
    
    // match 中で使う
    match x {
        0 => println("zero");
        n => println(f"other: $(n)");
    }
    
    // 他のパターンと組み合わせて使う
    let tuple = (1, 2, 3);
    if tuple is (first, ...) {
        println(f"first = $(first)");
    }
}
```



## 当面サポートしないもの
StrPattern
GroupedPattern
MacroInvocationPattern
RangePattern
SlicePattern
Pattern Guard
Or pattern
