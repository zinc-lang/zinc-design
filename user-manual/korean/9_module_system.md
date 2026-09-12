# 모듈 시스템

## 컴포넌트 (component)

zinc의 컴파일 단위는 컴포넌트입니다. rust의 crate에 해당합니다.

> In C and C++ programming language terminology, a translation unit (or more casually a compilation unit) is the ultimate input to a C or C++ compiler from which an object file is generated.

컴파일 단위란, 컴파일러 프로세스를 한 번 실행하는 데 필요한 소스 파일을 말합니다. 이런 소스 파일은 컴파일러가 한 번 컴파일하는 최소 입력 단위이며, 더 쪼갤 수 없습니다. 더 쪼개면 한 번의 컴파일을 실행할 수 없습니다.

C/C++ 컴파일러에게 컴파일 단위는 하나의 .c 또는 .cpp 소스 파일과, 그것이 직접·간접으로 `#include`한 모든 파일입니다.

Rust 컴파일러에게 컴파일 단위는 하나의 crate입니다. 사용자는 crate 내부의 여러 mod를 각각 컴파일할 수 없습니다.

Zinc 컴파일러에게 컴파일 단위는 하나의 컴포넌트(component)입니다. 컴파일러가 매번 실행할 때 입력은 완전한 컴포넌트의 소스 코드입니다. 사용자는 한 컴포넌트의 여러 소스 파일을 각각 컴파일한 뒤 다시 하나의 컴포넌트로 조립할 수 없으며, 컴파일러에 그런 능력이 없습니다. 한 컴포넌트는 반드시 하나의 전체로 컴파일해야 합니다.

컴포넌트와 컴포넌트 사이에는 순환 의존이 불가합니다.

컴포넌트 내부에는 모듈(mod)이 있으며, 컴포넌트 내부의 모듈 사이에는 순환 의존이 가능합니다. 같은 컴포넌트 안의 모든 모듈이 하나의 컴파일러 프로세스에서 함께 컴파일되기 때문입니다.

컴포넌트는 컴파일 후 기계어를 담은 `.o` 파일과, 해당 컴포넌트의 대외 공개 인터페이스 `.zno` 파일을 생성합니다. zno라는 이름은 “산화아연” 물질에서 왔습니다.

`.zno` 파일은 C 언어의 “헤더 파일”로 이해할 수 있으며, 다만 헤더 파일은 사람이 작성하고 `.zno` 파일은 컴파일러가 자동 생성합니다.

`.zno` 파일 내용은 모든 공개 함수 시그니처, 타입 정의, 전역 변수 선언, impl 선언 등을 포함합니다. 컴포넌트를 넘어 인라인할 수 있는 모든 함수의 함수 본체도 포함합니다.

## `export`

컴포넌트가 라이브러리라면, 사용자가 `export component_name;`으로 이 컴포넌트의 이름을 선언해야 합니다.
컴포넌트가 실행 프로그램이라면, 사용자가 root mod 안에 `fn main()` 함수를 정의해 프로그램 진입점으로 삼아야 합니다.
한 컴포넌트에 이 둘이 모두 없으면 컴파일러가 오류를 냅니다.

Rust에서 crate의 이름은 컴파일 옵션 `--crate-name NAME`으로 지정하는데, 저는 이것이 합리적이지 않다고 생각합니다. 컴포넌트 이름은 그 소스 코드가 지정해야 합니다.
결국 컴포넌트 이름은 바이너리의 심볼 mangle name에 영향을 주며, 컴포넌트 이름을 컴파일 옵션이 제어하면 서로 다른 컴파일 환경에서 컴파일 결과의 일관성을 보장하기 어렵습니다.

## `import`

컴포넌트 a가 다른 컴포넌트 b에 의존함을 기술할 때, a의 소스 코드에 `import b;`를 써야 합니다. 이는 기본적으로 Rust의 `extern crate b;`와 같은 뜻입니다.

C/C++ 언어로 비유하면, `#include "b.h"`의 뜻으로 단순하게 이해할 수 있습니다.

## mod 소개

모듈(mod)은 Zinc에서 코드를 조직하는 기본 단위입니다. 모듈은 함수, 타입, 상수, 그리고 다른 모듈을 포함할 수 있습니다.

### 모듈 정의

`mod` 키워드로 모듈을 정의합니다:

```rust
// 파일에서 모듈 정의
mod my_module {
    fn helper() -> Int {
        return 42;
    }
    
    pub fn public_function() {
        println(f"helper returned $(helper())");
    }
}
```

### 파일 시스템 모듈

모듈은 파일 시스템으로도 조직할 수 있습니다. 모듈에 내용이 없으면, 컴파일러는 같은 이름의 파일 또는 디렉터리를 찾습니다:

```rust
// main.zn에서
mod network;

// 컴파일러가 찾는 것:
// 1. network.zn 파일
// 2. network/mod.zn 파일
```

### 가시성

모듈의 항목은 기본이 비공개이며, 그것을 정의한 모듈 내부에서만 접근할 수 있습니다. `pub` 키워드를 쓰면 항목을 대외에 보이게 할 수 있습니다:

```rust
mod my_module {
    pub fn public_function() {
        // 외부에서 접근할 수 있음
    }
    
    fn private_function() {
        // my_module 내부에서만 접근할 수 있음
    }
    
    pub struct PublicStruct {
        pub field: Int,      // 공개 멤버
        private_field: Int,  // 비공개 멤버
    }
}
```

### 중첩 모듈

모듈은 중첩 정의할 수 있으며, 트리 구조를 이룹니다:

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

### 모듈 경로

`::` 연산자로 모듈의 항목에 접근합니다:

```rust
fn main() {
    outer::outer_function();
    outer::inner::inner_function();
    outer::inner::deeply_nested::deep_function();
}
```

### 가시성 수식어

Zinc는 다음 가시성 수식어를 지원합니다:

1. **기본(비공개)**: 그것을 정의한 모듈 내부에서만 접근할 수 있음
2. **`pub`**: 모든 모듈에 보임
3. **`pub(in path)`**: 지정한 경로의 모듈 안에서만 보임 (> **계획 중인 기능**: 향후 지원)

```rust
mod my_module {
    pub fn public_api() {}
    
    fn internal_helper() {}
    
    // 향후 지원: 부모 모듈에만 보임
    // pub(super) fn parent_visible() {}
}
```

### 재내보내기

`pub use`를 쓰면 다른 모듈의 항목을 재내보낼 수 있습니다:

```rust
mod inner {
    pub fn helper() {}
}

mod outer {
    // inner::helper를 재내보내기
    pub use inner::helper;
}

fn main() {
    // outer::helper로 접근할 수 있음
    outer::helper();
}
```

### 모듈과 컴포넌트의 관계

> **계획 중인 기능**: export path와 import path 기능 지원을 고려합니다. 즉 `export ident;` `import ident;` 같은 표기만 허용하는 것이 아니라, `export ident1::ident2::ident3;`와 `import ident1::ident2::ident3;` 같은 표기도 허용해야 합니다.

주로 고려하는 점은, 컴포넌트와 컴포넌트 사이에 병렬 관계만 있고 포함 관계가 없으면 충분히 유연하지 않다는 것입니다.
한 컴포넌트 내부의 논리 구조는 트리 구조인데, 컴포넌트와 컴포넌트에 포함 관계가 없고 병렬 관계만 있으면, 한 컴포넌트 규모가 너무 커 컴파일 시간이 너무 길어 리팩터가 필요할 때, 리팩터 자체가 반드시 컴포넌트의 논리 구조를 바꾸게 됩니다. 이는 부적절합니다.

컴포넌트 내부의 어떤 하위 트리도 모듈뿐 아니라 컴포넌트가 될 수 있게 허용하는 것을 고려할 수 있으며, 대형 컴포넌트를 서로 다른 컴파일 단위로 나누는 데 도움이 되고, 컴포넌트 내부의 논리 구조에는 영향을 주지 않습니다.

두 가지 원칙은 바꾸지 말아야 합니다:
1. 한 컴포넌트의 논리 구조는 숲이 아니라 트리입니다
2. 컴포넌트와 컴포넌트 사이에는 순환 의존이 불가합니다

예를 들어 아래 그림에서, 컴포넌트 c는 일련의 하위 모듈을 포함하며 규모가 거대한 컴포넌트입니다.

<img src="./mod_tree.png" width="50%" align=center />

컴파일 속도를 높이기 위해, 응집성이 강한 어떤 하위 모듈을 독립 컴포넌트로 나눌 수 있으며, 그림에서 서로 다른 색으로 나타냅니다.
모듈 e와 그 하위 모든 모듈이 하나의 컴포넌트이고, 모듈 g와 그 하위 모든 모듈도 하나의 컴포넌트이며, 모듈 i와 그 하위도 하나의 컴포넌트입니다. 그림에서 같은 색의 모듈은 함께 컴파일되며, 서로 다른 컴포넌트 사이에 순환 의존이 없기만 보장하면 됩니다. 동시에 전체 컴포넌트의 논리 구조는 그대로 유지하며, 그중 컴포넌트 g의 루트 모듈은 `c::d::g` 같은 이름이고, 내부 모든 item의 mangle name도 이런 명명을 유지합니다.


## use 문장

`use` 문장은 다른 모듈의 항목을 현재 스코프에 가져와, 매번 완전한 경로를 쓰지 않게 하는 데 쓰입니다.

### 기본 용법

```rust
use std::collections::HashMap;

fn main() {
    let map = HashMap::new();
    // HashMap을 직접 쓸 수 있으며, 완전한 경로를 쓸 필요가 없음
}
```

### 경로 키워드

- `self`: 현재 mod에 접근할 수 있음
- `super`: 현재 mod의 한 단계 위 mod에 접근할 수 있음
- `::ident`: 전역 범위의 이름, 즉 component 이름에 접근할 수 있음. 합법적인 이름은 export에 정의된 자신의 이름, import로 가져온 의존 항목의 이름, 그리고 std 표준 라이브러리를 포함합니다.
- `::self`: 현재 component의 최상위 mod에 접근할 수 있음. `::ident`로 시작해 본 컴포넌트의 최상위 이름에 접근하는 것과 동등합니다. 주: Rust와 차이가 있습니다. Rust에서는 `crate::` 키워드로 최상위 mod에 접근합니다.

### 예시

```rust
mod outer {
    pub fn outer_fn() {}
    
    mod inner {
        pub fn inner_fn() {}
        
        fn example() {
            // self로 현재 모듈에 접근
            self::inner_fn();
            
            // super로 부모 모듈에 접근
            super::outer_fn();
        }
    }
}

// use로 가져오기
use outer::inner::inner_fn;

fn main() {
    inner_fn();
}
```

### 이름 바꿔 가져오기

`as` 키워드로 가져온 항목의 이름을 바꿀 수 있습니다:

```rust
use std::collections::HashMap as Map;

fn main() {
    let map = Map::new();
}
```

### 여러 항목 가져오기

```rust
use std::collections::{HashMap, HashSet};

fn main() {
    let map = HashMap::new();
    let set = HashSet::new();
}
```
