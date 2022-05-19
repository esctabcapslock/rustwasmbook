<!-- # Shrinking `.wasm` Size -->
# `.wasm` 크기 줄이기

생명게임 웹 애플리케이션과 같이 네트워크를 통해 클라이언트에 제공하는 `.wasm` 바이너리의 경우 코드 크기를 주시해야 합니다. `.wasm`이 작을수록 페이지 로드가 빨라져 사용자가 더 행복해집니다.
<!-- For `.wasm` binaries that we ship to clients over the network, such as our Game of Life Web application, we want to keep an eye on code size. The smaller our `.wasm` is, the faster our page loads get, and the happier our users are. -->

## 빌드 구성을 통해 Game of Life `.wasm` 바이너리를 얼마나 작게 얻을 수 있나요?
<!-- ## How small can we get our Game of Life `.wasm` binary via build configuration? -->


[잠시 시간을 내어 더 작은 `.wasm` 코드 크기를 얻기 위해 조정할 수 있는 빌드 구성 옵션을 검토하세요.](../reference/code-size.html#optimizing-builds-for-code-size)
<!-- [Take a moment to review the build configuration options we can tweak to get smaller `.wasm` code sizes.](../reference/code-size.html#optimizing-builds-for-code-size) -->

기본 릴리스 빌드 구성(디버그 기호 없음)에서 WebAssembly 바이너리는 29,410바이트입니다.
<!-- With the default release build configuration (without debug symbols), our WebAssembly binary is 29,410 bytes: -->

```
$ wc -c pkg/wasm_game_of_life_bg.wasm
29410 pkg/wasm_game_of_life_bg.wasm
```

LTO를 활성화하고 `opt-level = "z"`를 설정하고 `wasm-opt -Oz`를 실행하면 결과 `.wasm` 바이너리가 17,317바이트로 축소됩니다.
<!-- After enabling LTO, setting `opt-level = "z"`, and running `wasm-opt -Oz`, the resulting `.wasm` binary shrinks to only 17,317 bytes: -->

```
$ wc -c pkg/wasm_game_of_life_bg.wasm
17317 pkg/wasm_game_of_life_bg.wasm
```


거의 모든 HTTP 서버가 하는 `gzip`으로 압축하면 겨우 9,045바이트가 됩니다!
<!-- And if we compress it with `gzip` (which nearly every HTTP server does) we get down to a measly 9,045 bytes! -->

```
$ gzip -9 < pkg/wasm_game_of_life_bg.wasm | wc -c
9045
```

## Exercises

* [`wasm-snip` 도구](../reference/code-size.html#use-the-wasm-snip-tool)를 사용하여 Game of Life의 `.wasm` 바이너리에서 패닉 인프라 기능을 제거하십시오. 얼마나 많은 바이트를 저장합니까?

<!-- * Use [the `wasm-snip` tool](../reference/code-size.html#use-the-wasm-snip-tool) to remove the panicking infrastructure functions from our Game of Life's `.wasm` binary. How many bytes does it save? -->

* [`wee_alloc`을 전역 할당자](https://github.com/rustwasm/wee_alloc)를 사용하거나 사용하지 않고 생병게임 프로젝를 빌드합니다. 이 프로젝트를 시작하기 위해 복제한 `rustwasm/wasm-pack-template` 템플릿에는 `wasm-game-of-life/Cargo.toml`의 `[feature]` 섹션에 있는 `default` 키에 추가하여 활성화할 수 있는 "wee_alloc" cargo 기능이 있습니다.:
<!-- * Build our Game of Life crate with and without [`wee_alloc` as its global allocator](https://github.com/rustwasm/wee_alloc). The `rustwasm/wasm-pack-template` template that we cloned to start this project has a "wee_alloc" cargo feature that you can enable by adding it to the `default` key in the `[features]` section of `wasm-game-of-life/Cargo.toml`: -->

  ```toml
  [features]
  default = ["wee_alloc"]
  ```

  `wee_alloc`을 사용하면 `.wasm` 바이너리를 얼마나 줄일 수 있습니까?
  <!-- How much size does using `wee_alloc` shave off of the `.wasm` binary? -->

* 우리는 하나의 'Universe'만 인스턴스화하므로 생성자를 제공하는 대신 단일 `static mut` 전역 인스턴스를 조작하는 작업을 내보낼 수 있습니다. 이 전역 인스턴스가 이전 장에서 논의한 이중 버퍼링 기술을 사용하는 경우 해당 버퍼도 `정적 mut` 전역으로 만들 수 있습니다. 이것은 생명 게임 구현에서 모든 동적 할당을 제거하고 할당자를 포함하지 않는 `#![no_std]` 크레이트로 만들 수 있습니다. 할당자 종속성을 완전히 제거하여 `.wasm`에서 얼마나 많은 크기가 제거되었습니까?
<!-- * We only ever instantiate a single `Universe`, so rather than providing a constructor, we can export operations that manipulate a single `static mut` global instance. If this global instance also uses the double buffering technique discussed in earlier chapters, we can make those buffers also be `static mut` globals. This removes all dynamic allocation from our Game of Life implementation, and we can make it a `#![no_std]` crate that doesn't include an allocator. How much size was removed from the `.wasm` by completely removing the allocator dependency? -->
