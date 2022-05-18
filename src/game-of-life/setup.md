# 개발환경 구축

이 절에서는 Rust 프로그램을 WebAssembly로 컴파일하고 JavaScript에 통합하기 위한 개발환경을 구축하는 방법을 설명합니다.
<!-- This section describes how to set up the toolchain for compiling Rust programs
to WebAssembly and integrate them into JavaScript. -->

## Rust 개발환경

Rust 개발환경을 구축하기 위해서는 `rustup`, `rustc`, 그리고
`cargo`가 필요합니다.
<!-- You will need the standard Rust toolchain, including `rustup`, `rustc`, and
`cargo`. -->

[다음을 통해 Rust 개발환경을 구축합시다.][rust-install]

Rust 및 WebAssembly는 Rust 릴리스 트레인을 탔기 때문에 안정적입니다!
즉, 실험적 기능 플래그가 필요하지 않습니다. 그러나 Rust 1.30 이상이 필요합니다.
<!-- The Rust and WebAssembly experience is riding the Rust release trains to stable!
That means we don't require any experimental feature flags. However, we do
require Rust 1.30 or newer. -->

## `wasm-pack`

'wasm-pack'은 Rust에서 생성된 WebAssembly를 빌드, 테스트 및 게시하기 위한 원스톱 "shop"입니다.
<!-- `wasm-pack` is your one-stop shop for building, testing, and publishing
Rust-generated WebAssembly. -->

[여기서 `wasm-pack`를 설치하세요.][wasm-pack-install]

## `cargo-generate`

[`cargo-generate`는 새로운 Rust 프로젝트를 빠르게 시작하고 실행할 수 있도록 도와줍니다. 기존 git 저장소를 템플릿으로 활용합니다.][cargo-generate]

다음 명령어를 이용해 `cargo-generate`를 설치하십시오.
<!-- Install `cargo-generate` with this command: -->

```
cargo install cargo-generate
```

## `npm`

`npm`은 JavaScript용 패키지 관리자입니다. JavaScript 번들러 및 개발 서버를 설치하고 실행하는 데 사용할 것입니다. 튜토리얼이 끝나면 컴파일된 `.wasm`을 `npm` 에 게시할 것입니다.

<!-- `npm` is a package manager for JavaScript. We will use it to install and run a
JavaScript bundler and development server. At the end of the tutorial, we will
publish our compiled `.wasm` to the `npm` registry. -->

[여기서 `npm`을 설치하세요.][npm-install]


이미 `npm`이 설치되어 있는 경우 다음 명령어로 `npm`이 최신 버전인지 확인하십시오.

```
npm install npm@latest -g
```

[rust-install]: https://www.rust-lang.org/tools/install
[npm-install]: https://www.npmjs.com/get-npm
[wasm-pack]: https://github.com/rustwasm/wasm-pack
[cargo-generate]: https://github.com/ashleygwilliams/cargo-generate
[wasm-pack-install]: https://rustwasm.github.io/wasm-pack/installer/
