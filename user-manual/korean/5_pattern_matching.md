
# 패턴 매칭

패턴 매칭은 주로 `is` 표현식과 `match` 문장 장면에 쓰입니다.

패턴 매칭의 목적은 어떤 표현식의 구조가 기대와 같은지 검사하는 것이며, 내부 값을 꺼내 새 변수에 바인딩하는 것도 허용합니다.

예시는 다음과 같습니다:

```
// 변수 x는 원소가 3개인 tuple이며, 두 번째 원소를 middle 변수에 바인딩한 뒤 middle > 5가 성립하는지 판단
if (x is (_, middle, _)) && middle > 5 {

}

// 변수 y의 타입은 trait을 가리키는 포인터이며, 이 방식으로 y가 실제로 가리키는 구체 타입이 무엇인지 판단할 수 있음. 판단에 성공하면 이 포인터가 p에 바인딩됨.
if y is p: *S {
    // p can be used inside block
}

// 변수 z는 Option<S> 타입이며, 아래에서 Some과 None 두 장면을 각각 처리
match z {
    Some(s) => {

    }
    _ => {}
}
```

표현식과 마찬가지로, 패턴도 중첩할 수 있습니다.

## WildcardPattern

자리 표시자 패턴입니다.

`_` 밑줄은 한 자리를 차지할 수 있습니다.

`...` 생략 부호는 여러 자리를 차지할 수 있습니다.

## LiteralPattern

정수, 부동소수점, Bool, Char 등 타입의 리터럴을 패턴으로 직접 쓰는 것을 허용합니다.

`Str` 리터럴 패턴은 아직 구현되지 않았습니다.

## TypePattern

타입 패턴은 어떤 표현식이 어떤 구체 하위 타입인지 판단하는 데 주로 쓰입니다. 타입 패턴은 `type` 키워드로 시작합니다.

예시:
```
// TR이 trait 이름이고 S가 struct 이름이라고 가정.
// 아래 문장은 trait을 가리키는 포인터가 S를 가리키는 포인터로 성공적으로 다운캐스트될 수 있는지 판단하는 데 쓰임.
fn test(p: &TR) {
    let b = p is type &S;  // type 키워드로 시작하는 이유는, 뒤의 modifier+identifier 패턴과 문법 충돌을 피하기 위함
}
```

## StructPattern

구조체 패턴은 구조체 타입의 값을 매칭하고, 멤버 변수를 구조 분해할 수 있습니다.

예시:
```rust
struct Point { x: Int, y: Int }

fn test(p: Point) {
    // 구조체를 매칭하고, 멤버 변수를 새 변수에 바인딩
    if p is { .x = a, .y = b }: Point {
        println(f"x = $(a), y = $(b)");
    }
    
    // 생략 부호로 다른 멤버를 무시
    if p is { .x = 10, ... }: Point {
        println("x is 10");
    }
    
    // 중첩 패턴 매칭
    match p {
        { .x = 0, .y = 0 } => println("origin");
        { .x = 0, .y = y } => println(f"on y-axis at $(y)");
        { .x = x, .y = 0 } => println(f"on x-axis at $(x)");
        { .x = x, .y = y } => println(f"point ($(x), $(y))");
    }
}
```

## TuplePattern

튜플 패턴은 튜플 타입의 값을 매칭하고, 원소를 구조 분해할 수 있습니다.

예시:
```rust
fn test(t: (Int, String, Bool)) {
    // 튜플을 매칭하고, 원소를 새 변수에 바인딩
    if t is (num, str, flag) {
        println(f"num = $(num), str = $(str), flag = $(flag)");
    }
    
    // 밑줄로 어떤 원소를 무시
    if t is (1, _, true) {
        println("first element is 1 and third is true");
    }
    
    // 중첩 튜플 패턴
    let nested = ((1, 2), (3, 4));
    if nested is ((a, b), (c, d)) {
        println(f"($(a), $(b)), ($(c), $(d))");
    }
}
```

## EnumVariantPattern

열거 변이체 패턴은 열거 타입의 특정 변이체를 매칭하고, 연관 값을 구조 분해할 수 있습니다.

예시:
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

수식어 패턴은 패턴 매칭에서 변수의 바인딩 방식을 제어하는 데 쓰입니다.

- `mut`: 바인딩된 변수를 가변으로 표시
- `&`: 참조를 매칭
- `&mut`: 가변 참조를 매칭
- `ref`: 이동이 아니라 참조로 바인딩
- `ref mut`: 가변 참조로 바인딩

예시:
```rust
fn test(s: String) {
    // mut 수식어: 바인딩된 변수를 가변으로 만듦
    let mut x = s;
    x.push_str(" modified");
    
    // ref 수식어: 참조로 바인딩해 이동을 피함

    // ref mut 수식어: 가변 참조로 바인딩

}
```

## IdentifierPattern

식별자 패턴은 매칭된 값을 변수 이름에 바인딩하는 데 쓰입니다.

예시:
```rust
fn test(x: Int) {
    // 단순 식별자 바인딩
    if x is y {
        println(f"y = $(y)");
    }
    
    // match에서 사용
    match x {
        0 => println("zero");
        n => println(f"other: $(n)");
    }
    
    // 다른 패턴과 결합해 사용
    let tuple = (1, 2, 3);
    if tuple is (first, ...) {
        println(f"first = $(first)");
    }
}
```



## 당분간 지원하지 않음
StrPattern
GroupedPattern
MacroInvocationPattern
RangePattern
SlicePattern
Pattern Guard
Or pattern
