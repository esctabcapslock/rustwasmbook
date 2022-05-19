# 왜 Rust와 WebAssembly안가요?

## 저수준의 제어와 고수준의 인간공학(Ergonomics)

JavaScript 웹 어플리케이션은 안정적인 성능을 얻고 유지하기 위해 노력해 왔습니다.
하지만 JavaScript의 dynamic type system과 garbage collection[^1] 중지는 성능에 악영향을 끼칩니다.
게다가 만약 작은 코드 변경이 JIT의 happy path를 벗어나게 된다면 성능이 크게 저하될 수 있습니다.

<!-- JavaScript Web applications struggle to attain and retain reliable performance.
JavaScript's dynamic type system and garbage collection pauses don't help.
Seemingly small code changes can result in drastic performance regressions if
you accidentally wander off the JIT's happy path. -->

Rust는 프로그레머에게 low-level의 제어와 안정적인 성능을 제공합니다.  JavaScript를 괴롭히는 비결정적 garbage collection 중지에서 자유롭습니다. 
프로그래머는 Rust를 통해 간접 참조, 단형성[^2], 그리고 자료구조를 제어할 수 있습니다. 

<!-- Rust gives programmers low-level control and reliable performance. It is free
from the non-deterministic garbage collection pauses that plague JavaScript.
Programmers have control over indirection, monomorphization, and memory layout. -->

## 작은 `.wasm` 파일의 크기

.wasm은 네트워크를 통해 다운로드 받아야 하기 때문에 코드 크기가 매우 중요합니다.
러스트는 런타임 시간이 부족하기 때문에 때문에 작은 ".wasm" 크기를 사용할 수 있습니다. 왜냐하면 가비지 수집기가 없어 코드 비대화가 일어나지 않기 대문니다.
단지 사용하는 기능에 대해서만 (코드 크기로) 용량을 차지합니다.

<!-- Code size is incredibly important since the `.wasm` must be downloaded over the
network. Rust lacks a runtime, enabling small `.wasm` sizes because there is no
extra bloat included like a garbage collector. You only pay (in code size) for
the functions you actually use. -->

## 처음부터 다시 작성하지 않음
<!-- ## Do *Not* Rewrite Everything -->

기존 코드를 버릴 필요가 없습니다. 성능에 가장 민감한 JavaScript 함수만 Rust로 포팅하여도 즉시 성능 개선을 얻을 수 있습니다. 그리고 거기서 멈출 수도 있습니다.
<!-- Existing code bases don't need to be thrown away. You can start by porting your
most performance-sensitive JavaScript functions to Rust to gain immediate
benefits. And you can even stop there if you want to. -->

## 다른 것들과 잘 어울림
<!-- ## Plays Well With Others -->

Rust 및 WebAssembly는 기존 JavaScript 도구에서 통합되어 지원됩니다.
ECMAScript 모듈을 지원하며 npm 및 Webpack과 같은 이미 좋아하는 도구를 계속 사용할 수 있습니다.
<!-- Rust and WebAssembly integrates with existing JavaScript tooling. It supports
ECMAScript modules and you can continue using the tooling you already love, like
npm and Webpack. -->

## The Amenities You Expect

러스트는 개발자들이 기대하는 다음과 같은 현대적인 편의 기능을 갖고 있습니다.

* `cargo`를 통한 강력한 패키지 관리

* 표현적(및 제로 비용) 추상화,

* 그리고 당신을 환영하는 커뮤니티! 😊

<!-- Rust has the modern amenities that developers have come to expect, such as:

* strong package management with `cargo`,

* expressive (and zero-cost) abstractions,

* and a welcoming community! 😊 -->

[^1]: 역자 주) 이미 할당된 메모리에서 사용하지 않는 메모리를 제거하는 행위

[^2]: 역자 주) 프로그램 언어의 각 요소가 한 가지 형태만 가지는 성질. 다형성의 반대