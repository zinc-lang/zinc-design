# unsafe와 상호 운용

## unsafe 키워드

Zinc 언어는 `unsafe` 키워드로 미정의 동작을 일으킬 수 있는 코드를 표시합니다. `unsafe`를 쓰면 컴파일러에게 "이 코드가 안전하지 않을 수 있음을 알지만, 그것이 올바름을 보장한다"고 알릴 수 있습니다.

### unsafe의 사용 장면

다음 연산은 반드시 unsafe 컨텍스트에서 해야 합니다:

1. **함수 호출, 메서드 호출**
   - `extern`으로 수식된 함수
   - `unsafe`로 수식된 함수

2. **날포인터 역참조**
   - `*raw T` 또는 `*raw mut T` 타입 포인터를 역참조

3. **타입 변환**
   - 날포인터를 어떤 타입으로든 변환
   - 정수와 포인터 사이의 변환

4. **union 멤버 접근**
   - union 타입의 멤버 변수를 읽고 씀

5. **mut static 접근**
   - `static mut`로 정의된 전역 변수를 읽고 씀

### unsafe 블록

`unsafe` 블록을 쓰면 unsafe 컨텍스트를 만들 수 있습니다:

```rust
fn main() {
    let x: Int = 42;
    let p: *raw Int = &x as *raw Int;
    
    unsafe {
        let value = *p; // 날포인터를 역참조하려면 unsafe가 필요
        println(f"value = $(value)");
    }
}
```

### unsafe 함수

`unsafe fn`으로 정의한 함수는 호출할 때 반드시 unsafe 컨텍스트에 있어야 합니다:

```rust
unsafe fn dangerous_operation() {
    // 미정의 동작을 일으킬 수 있는 코드
}

fn main() {
    unsafe {
        dangerous_operation(); // unsafe 함수 호출
    }
}
```

### unsafe impl

`unsafe impl`을 쓰면 `Send`와 `Sync` 같은 어떤 marker trait을 수동으로 구현할 수 있습니다:

```rust
struct MyType {
    ptr: *raw mut Int,
}

// MyType이 Send임을 수동으로 선언
// 프로그래머는 이 선언이 올바름을 보장해야 함
unsafe impl Send for MyType {}
```

## FFI（외부 함수 인터페이스）

### extern 함수

`extern` 키워드로 외부 함수를 선언할 수 있으며, 보통 C 언어 라이브러리를 호출하는 데 쓰입니다:

```rust
// C 표준 라이브러리의 printf 함수를 선언
extern fn printf(fmt: *const UInt8, ...) -> Int32;

fn main() {
    unsafe {
        printf("Hello from C!\n" as *const UInt8);
    }
}
```

### extern 블록

`extern` 블록으로 외부 함수를 일괄 선언할 수 있습니다:

```rust
extern "C" {
    fn malloc(size: UInt) -> *raw mut UInt8;
    fn free(ptr: *raw mut UInt8);
    fn strlen(s: *const UInt8) -> UInt;
}

fn main() {
    unsafe {
        let ptr = malloc(100);
        // ptr 사용...
        free(ptr);
    }
}
```

### C 언어와의 상호 운용

Zinc는 C 언어와 상호 운용하는 여러 방식을 제공합니다:

1. **C 함수 호출**: `extern`으로 C 함수를 선언한 뒤 unsafe 블록에서 호출
2. **C에 데이터 전달**: 날포인터 타입으로 데이터를 전달
3. **C로부터 데이터 수신**: 날포인터로 C가 반환한 데이터를 수신

예시:

```rust
// C 함수 선언
extern fn fopen(filename: *const UInt8, mode: *const UInt8) -> *raw mut UInt8;
extern fn fclose(file: *raw mut UInt8) -> Int32;

fn main() {
    unsafe {
        let file = fopen("test.txt\0" as *const UInt8, "r\0" as *const UInt8);
        if file != null {
            // 파일 사용...
            fclose(file);
        }
    }
}
```

## 안전과 불안전의 경계
