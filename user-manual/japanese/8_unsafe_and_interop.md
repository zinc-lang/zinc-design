# unsafe と相互運用

## unsafe キーワード

Zinc 言語は `unsafe` キーワードによって、未定義動作を生じうるコードを印す。`unsafe` を使うと、コンパイラに「このコードは安全でないかもしれないが、正しいことを保証する」と伝えられる。

### unsafe の使用場面

次の操作は unsafe コンテキスト中で行わなければならない：

1. **関数呼び出し、メソッド呼び出し**
   - `extern` で修飾された関数
   - `unsafe` で修飾された関数

2. **生ポインタの参照外し**
   - `*raw T` または `*raw mut T` 型のポインタを参照外しする

3. **型変換**
   - 生ポインタから任意の型への変換
   - 整数とポインタの間の変換

4. **union メンバアクセス**
   - union 型のメンバ変数の読み書き

5. **mut static へのアクセス**
   - `static mut` で定義したグローバル変数の読み書き

### unsafe ブロック

`unsafe` ブロックを使うと unsafe コンテキストを作れる：

```rust
fn main() {
    let x: Int = 42;
    let p: *raw Int = &x as *raw Int;
    
    unsafe {
        let value = *p; // 生ポインタの参照外しには unsafe が必要
        println(f"value = $(value)");
    }
}
```

### unsafe 関数

`unsafe fn` で定義した関数は、呼び出すとき unsafe コンテキスト中でなければならない：

```rust
unsafe fn dangerous_operation() {
    // 未定義動作を生じうるコード
}

fn main() {
    unsafe {
        dangerous_operation(); // unsafe 関数の呼び出し
    }
}
```

### unsafe impl

`unsafe impl` を使うと、`Send` や `Sync` などの marker trait を手動で実装できる：

```rust
struct MyType {
    ptr: *raw mut Int,
}

// MyType が Send であることを手動で宣言する
// プログラマはこの宣言が正しいことを保証する必要がある
unsafe impl Send for MyType {}
```

## FFI（外部関数インタフェース）

### extern 関数

`extern` キーワードを使うと外部関数を宣言でき、通常は C 言語ライブラリの呼び出しに使う：

```rust
// C 標準ライブラリの printf 関数を宣言する
extern fn printf(fmt: *const UInt8, ...) -> Int32;

fn main() {
    unsafe {
        printf("Hello from C!\n" as *const UInt8);
    }
}
```

### extern ブロック

`extern` ブロックを使うと外部関数を一括宣言できる：

```rust
extern "C" {
    fn malloc(size: UInt) -> *raw mut UInt8;
    fn free(ptr: *raw mut UInt8);
    fn strlen(s: *const UInt8) -> UInt;
}

fn main() {
    unsafe {
        let ptr = malloc(100);
        // ptr を使う...
        free(ptr);
    }
}
```

### C 言語との相互運用

Zinc は C 言語と相互運用する複数の方法を提供する：

1. **C 関数の呼び出し**：`extern` で C 関数を宣言し、unsafe ブロック中で呼び出す
2. **C へデータを渡す**：生ポインタ型でデータを渡す
3. **C からデータを受け取る**：生ポインタで C が返すデータを受け取る

例：

```rust
// C 関数を宣言する
extern fn fopen(filename: *const UInt8, mode: *const UInt8) -> *raw mut UInt8;
extern fn fclose(file: *raw mut UInt8) -> Int32;

fn main() {
    unsafe {
        let file = fopen("test.txt\0" as *const UInt8, "r\0" as *const UInt8);
        if file != null {
            // ファイルを使う...
            fclose(file);
        }
    }
}
```

## 安全と不安全の境界
