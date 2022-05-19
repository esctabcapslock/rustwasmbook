# WebAssembly란 무엇인가요?

웹어셈블리(Wasm)는 간단한 기계 모델이며 [광범위한 사양]을 가진 실행 파일 형식입니다. 배포하기 좋고, 크기가 작으며, 네이티브 속도 또는 그에 가까운 속도로 실행되도록 설계되었습니다.

<!-- WebAssembly (wasm) is a simple machine model and executable format with an
[extensive specification]. It is designed to be portable, compact, and execute
at or near native speeds. -->

프로그래밍 언어로서 웹어셈블리는 같은 구조를 나타내는 두 가지 포맷으로 구성되어 있다.

<!-- As a programming language, WebAssembly is comprised of two formats that
represent the same structures, albeit in different ways: -->


1.  `.wat` text 형식은 (또한  "**W**eb**A**ssembly **T**ext"이므로 `wat`라고도 불림)
   [S-expressions]를 사용하며, Scheme 및 Clojure와 같은 Lisp 언어 계열과 유사합니다.

<!-- 
1. The `.wat` text format (called `wat` for "**W**eb**A**ssembly **T**ext") uses
   [S-expressions], and bears some resemblance to the Lisp family of languages
   like Scheme and Clojure. -->

2.  `.wasm` binary 포멧은 lower-level이며 wasm 가상 머신에서 직접 사용하기 위한 용도입니다. ELF 및 Mach-O와 개념적으로 유사합니다.
<!-- 2. The `.wasm` binary format is lower-level and intended for consumption
   directly by wasm virtual machines. It is conceptually similar to ELF and
   Mach-O. -->

참고로 다음은 `wat`로 작성한 펙토리얼 함수입니다:
<!-- For reference, here is a factorial function in `wat`: -->

```
(module
  (func $fac (param f64) (result f64)
    local.get 0
    f64.const 1
    f64.lt
    if (result f64)
      f64.const 1
    else
      local.get 0
      local.get 0
      f64.const 1
      f64.sub
      call $fac
      f64.mul
    end)
  (export "fac" (func $fac)))
```

만약 'wasm' 파일이 어떻게 생겼는지 궁금하다면 위의 코드로 [wat2wasm 데모]를 사용할 수 있다.
<!-- If you're curious about what a `wasm` file looks like you can use the [wat2wasm
demo] with the above code. -->

## 선형 Memory
<!-- ## Linear Memory -->

WebAssembly는 매우 간단한 [메모리 모델]을 갖는다. wasm 모듈은 기본적으로  바이트로 된 flat array인 단일 "선형 메모리"에 액세스할 수 있습니다.
이 [메모리는 증가할 수 있으며] 페이지 크기(64K)의 배수이며 축소될 수 없습니다.

<!-- WebAssembly has a very simple [memory model]. A wasm module has access to a
single "linear memory", which is essentially a flat array of bytes. This
[memory can be grown] by a multiple of the page size (64K). It cannot be shrunk. -->

## WebAssembly는 단지 웹을 위한 것입니까?
<!-- ## Is WebAssembly Just for the Web? -->

현재 일반적으로 JavaScript 및 웹 커뮤니티에서 주목을 받고 있지만 wasm은 실행 환경에 구예받지 않습니다. 따라서 wasm이 "portable executable"이 될 것이라고 추측하는 것이 합리적입니다.
이는 향후 다양한 컨텍스트(context)[^1]에서 사용될 "실행 파일" 형식입니다.
그러나, *아직은*  wasm은 대부분 JavaScript와 관련되어 있습니다. (웹 및 [Node.js] 모두 포함).

<!-- Although it has currently gathered attention in the JavaScript and Web
communities in general, wasm makes no assumptions about its host
environment. Thus, it makes sense to speculate that wasm will become a "portable
executable" format that is used in a variety of contexts in the future. As of
*today*, however, wasm is mostly related to JavaScript (JS), which comes in many
flavors (including both on the Web and [Node.js]). -->

[메모리 모델]: https://webassembly.github.io/spec/core/syntax/modules.html#syntax-mem
[메모리는 증가할 수 있으며]: https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-instr-memory
[extensive specification]: https://webassembly.github.io/spec/
[value types]: https://webassembly.github.io/spec/core/syntax/types.html#value-types
[Node.js]: https://nodejs.org
[S-expressions]: https://en.wikipedia.org/wiki/S-expression
[wat2wasm 데모]: https://webassembly.github.io/wabt/demo/wat2wasm/

[^1]: 역자 주) 나중에 같은 지점에서 계속될 수 있도록 저장해야 하는 작업에서 사용하는 최소 데이터 집합.