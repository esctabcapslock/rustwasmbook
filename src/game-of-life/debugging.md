<!-- # Debugging -->
# 디버깅

더 많은 코드를 작성하기 전에 문제가 발생했을 때를 대비해 몇 가지 디버깅 도구를 사용하고 싶을 것입니다. 잠시 시간을 내어 [Rust 생성 WebAssembly를 디버깅하는 데 사용할 수 있는 도구 및 접근 방식을 서술한 참조 페이지][reference-debugging]을 검토하세요.

<!-- Before we write much more code, we will want to have some debugging tools in our belt for when things go wrong. Take a moment to review the [reference page listing tools and approaches available for debugging Rust-generated WebAssembly][reference-debugging]. -->

[reference-debugging]: ../reference/debugging.html

<!-- ## Enable Logging for Panics -->
## Panic에 대한 로깅 활성화

[코드 패닉이 발생하면 개발자 콘솔에 유익한 오류 메시지가 표시되기를 원합니다.](../reference/debugging.html#logging-panics)

우리의 `wasm-pack-template`은 `wasm-game-of-life/src/utils.rs`에 구성된 [`console_error_panic_hook` 크레이트][panic-hook]에 대한 기본적으로 활성화된 선택적 종속성과 함께 제공됩니다. 단지 초기화 함수나 공통 코드 경로에 후크를 설치하기만 하면 됩니다. 이는 `wasm-game-of-life/src/lib.rs`의 `Universe::new` 생성자 내부에서 호출할 수 있습니다.:

<!-- Our `wasm-pack-template` comes with an optional, enabled-by-default dependency on [the `console_error_panic_hook` crate][panic-hook] that is configured in `wasm-game-of-life/src/utils.rs`. All we need to do is install the hook in an initialization function or common code path. We can call it inside the
`Universe::new` constructor in `wasm-game-of-life/src/lib.rs`: -->

```rust
pub fn new() -> Universe {
    utils::set_panic_hook();

    // ...
}
```

[panic-hook]: https://github.com/rustwasm/console_error_panic_hook

<!-- ## Add Logging to our Game of Life -->
## 생명 게임에 로깅 추기

[`web-sys` 크레이트를 통해 `console.log` 기능을 사용하여 일부 로깅을 추가][logging]하여 `Universe::tick` 기능의 각 셀에 대해 알아보겠습니다.

먼저 `web-sys`를 종속 항목으로 추가하고 `wasm-game-of-life/Cargo.toml`에서 `"console"` 기능을 활성화합니다.

<!-- Let's [use the `console.log` function via the `web-sys` crate to add some
logging][logging] about each cell in our `Universe::tick` function.

First, add `web-sys` as a dependency and enable its `"console"` feature in
`wasm-game-of-life/Cargo.toml`: -->

```toml
[dependencies]

# ...

[dependencies.web-sys]
version = "0.3"
features = [
  "console",
]
```
인간공학을 위해 `console.log` 기능을 `println!`스타일 매크로로 래핑합니다.

<!-- For ergonomics, we'll wrap the `console.log` function up in a `println!`-style macro: -->

[logging]: ../reference/debugging.html#logging-with-the-console-apis

```rust
extern crate web_sys;

// `console.log` 로깅을 위한 `println!(..)` 스타일의 구문을 제공하는 매크로입니다.
macro_rules! log {
    ( $( $t:tt )* ) => {
        web_sys::console::log_1(&format!( $( $t )* ).into());
    }
}
```

이제 Rust 코드에 `log` 호출을 삽입하여 콘솔에 메시지 로깅을 시작할 수 있습니다. 예를 들어, 각 셀의 상태, 라이브 이웃 수 및 다음 상태를 기록하려면 `wasm-game-of-life/src/lib.rs`를 다음과 같이 수정할 수 있습니다.

<!-- Now, we can start logging messages to the console by inserting calls to `log` in Rust code. For example, to log each cell's state, live neighbors count, and next state, we could modify `wasm-game-of-life/src/lib.rs` like this: -->

```diff
diff --git a/src/lib.rs b/src/lib.rs
index f757641..a30e107 100755
--- a/src/lib.rs
+++ b/src/lib.rs
@@ -123,6 +122,14 @@ impl Universe {
                 let cell = self.cells[idx];
                 let live_neighbors = self.live_neighbor_count(row, col);

+                log!(
+                    "cell[{}, {}] is initially {:?} and has {} live neighbors",
+                    row,
+                    col,
+                    cell,
+                    live_neighbors
+                );
+
                 let next_cell = match (cell, live_neighbors) {
                     // 규칙 1: 살아있는 이웃이 2개 미만인 살아있는 세포는 
                     // 마치 인구 부족으로 인한 것처럼 죽습니다.
@@ -140,6 +147,8 @@ impl Universe {
                     (otherwise, _) => otherwise,
                 };

+                log!("    it becomes {:?}", next_cell);
+
                 next[idx] = next_cell;
             }
         }
```

## 디버거를 사용하여 각 tick 사이에 일시 중지
<!-- ## Using a Debugger to Pause Between Each Tick -->

[브라우저의 스테핑 디버거는 Rust가 생성한 WebAssembly가 상호 작용하는 JavaScript를 검사하는 데 유용합니다.](../reference/debugging.html#using-a-debugger)

예를 들어, `universe.tick()` 호출 위에 JavaScript[ `debugger;`][dbg-stmt]문을 배치하여 `renderLoop` 함수의 각 반복에서 디버거를 사용하여 일시 중지할 수 있습니다.

<!-- 
[Browser's stepping debuggers are useful for inspecting the JavaScript that our Rust-generated WebAssembly interacts with.](../reference/debugging.html#using-a-debugger)

For example, we can use the debugger to pause on each iteration of our `renderLoop` function by placing [a JavaScript `debugger;` statement][dbg-stmt] above our call to `universe.tick()`. -->

```js
const renderLoop = () => {
  debugger;
  universe.tick();

  drawGrid();
  drawCells();

  requestAnimationFrame(renderLoop);
};
```
이것은 기록된 메시지를 검사하고 현재 렌더링된 프레임을 이전 프레임과 비교하기 위한 편리한 체크포인트를 제공합니다.
<!-- This provides us with a convenient checkpoint for inspecting logged messages, and comparing the currently rendered frame to the previous one. -->

[dbg-stmt]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/debugger

[![Screenshot of debugging the Game of Life](../images/game-of-life/debugging.png)](../images/game-of-life/debugging.png)

<!-- ## Exercises

* Add logging to the `tick` function that records the row and column of each cell that transitioned states from live to dead or vice versa.

* Introduce a `panic!()` in the `Universe::new` method. Inspect the panic's backtrace in your Web browser's JavaScript debugger. Disable debug symbols, rebuild without the `console_error_panic_hook` optional dependency, and inspect the stack trace again. Not as useful is it  -->

## 연습문제

* 라이브에서 데드로 또는 그 반대로 상태를 전환한 각 셀의 행과 열을 기록하는 `tick` 기능에 로깅을 추가합니다.

* `Universe::new` 메소드에 `panic!()`을 도입합니다. 웹 브라우저의 JavaScript 디버거에서 패닉의 역추적을 검사하십시오. 디버그 기호를 비활성화하고 `console_error_panic_hook` 선택적 종속성 없이 다시 빌드하고 스택 추적을 다시 검사합니다. 그다지 유용하지 않습니다
