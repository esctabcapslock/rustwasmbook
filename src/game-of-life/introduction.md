# 튜토리얼: 콘웨이의 생명 게임
<!-- # Tutorial: Conway's Game of Life -->

이것은 Rust와 WebAssembly를 통해  [콘웨이의 생명 게임][gol] 을 구현하는 튜토리얼입니다.
<!-- This is a tutorial that implements [Conway's Game of Life][gol] in Rust and
WebAssembly. -->

[gol]: https://namu.wiki/w/%EC%BD%98%EC%9B%A8%EC%9D%B4%EC%9D%98%20%EC%83%9D%EB%AA%85%20%EA%B2%8C%EC%9E%84

## 이 튜토리얼은 누구를 위한 것입니까?

이 튜토리얼은 이미 기본적인 Rust와 JavaScript의 기초를 갖고 있는 사람을 위한 것입니다.
앞으로 Rust, WebAssembly 및 JavaScript를 사용하는 방법을 배울 것입니다.
<!-- This tutorial is for anyone who already has basic Rust and JavaScript
experience, and wants to learn how to use Rust, WebAssembly, and JavaScript
together. -->

기본적인 Rust, JavaScript 및 HTML을 읽고 작성할 줄 알아야 합나다. 물론 반드시 전문가 수준이 될 필요는 없습니다.

<!-- You should be comfortable reading and writing basic Rust, JavaScript, and
HTML. You definitely do not need to be an expert. -->

## 무엇을 배울까요?

* WebAssembly로 컴파일하기 위해 Rust 툴체인을 설정하는 방법.
<!-- * How to set up a Rust toolchain for compiling to WebAssembly. -->

* Rust, WebAssembly, JavaScript, HTML 및 CSS를 이용해 여러 언어로 된 프로그램을 개발하기 위한 워크플로.
<!-- * A workflow for developing polyglot programs made from Rust, WebAssembly,
  JavaScript, HTML, and CSS. -->

* Rust와 WebAssembly의 장점과 JavaScript의 장점을 최대한 활용하도록 API를 설계하는 방법.
<!-- * How to design APIs to take maximum advantage of both Rust and WebAssembly's
  strengths and also JavaScript's strengths. -->

<!-- * How to debug WebAssembly modules compiled from Rust.

* How to time profile Rust and WebAssembly programs to make them faster.

* How to size profile Rust and WebAssembly programs to make `.wasm` binaries
  smaller and faster to download over the network. -->
  * Rust에서 컴파일된 WebAssembly 모듈을 디버그하는 방법.

* Rust 및 WebAssembly 프로그램을 더 빠르게 만드는 방법.

* 네트워크를 통해 더 작고 빠르게 다운로드하기 위해서, `.wasm` 바이너리의 크기를줄이는 방법
   
