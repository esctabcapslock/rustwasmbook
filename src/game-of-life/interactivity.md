# 상호작용 추가하기

생명 게임 구현에 몇 가지 대화형 기능을 추가하여 JavaScript 및 WebAssembly 인터페이스를 계속 탐색할 것입니다. 우리는 사용자가 셀을 클릭하여 살아 있는지 여부를 전환할 수 있도록 하고 게임을 일시 중지할 수 있도록 하여 셀 패턴을 훨씬 더 쉽게 그릴 수 있습니다.

<!-- We will continue to explore the JavaScript and WebAssembly interface by adding some interactive features to our Game of Life implementation. We will enable users to toggle whether a cell is alive or dead by clicking on it, and allow pausing the game, which makes drawing cell patterns a lot easier. -->

## Pausing and Resuming the Game

게임이 재생 중인지 일시 중지되었는지 여부를 전환하는 버튼을 추가해 보겠습니다. `wasm-game-of-life/www/index.html`에 `<canvas>` 바로 위에 버튼을 추가하세요.
<!-- Let's add a button to toggle whether the game is playing or paused. To `wasm-game-of-life/www/index.html`, add the button right above the `<canvas>`: -->

```html
<button id="play-pause"></button>
```

`wasm-game-of-life/www/index.js` JavaScript 파일에서 다음과 같은 점을 변경할 것입니다.
<!-- In the `wasm-game-of-life/www/index.js` JavaScript, we will make the following changes: -->

* 최근 `requestAnimationFrame` 호출에서 반환된 식별자를 추적하여 해당 식별자로 `cancelAnimationFrame`을 호출하여 애니메이션을 취소할 수 있습니다.
<!-- * Keep track of the identifier returned by the latest call to `requestAnimationFrame`, so that we can cancel the animation by calling `cancelAnimationFrame` with that identifier. -->

* 재생/일시 정지 버튼을 클릭했을 때 대기 중인 애니메이션 프레임에 대한 식별자가 있는지 확인합니다. 이때 만약 게임이 현재 재생 중이라면 `renderLoop`이 다시 호출되지 않도록 애니메이션 프레임을 취소하여 게임을 효과적으로 일시 중지하려고 합니다. 대기열에 있는 애니메이션 프레임에 대한 식별자가 없으면 현재 일시 중지되어 있다고 판단하여 `requestAnimationFrame`을 호출하여 게임을 다시 시작하려고 합니다.
<!-- * When the play/pause button is clicked, check for whether we have the identifier for a queued animation frame. If we do, then the game is currently playing, and we want to cancel the animation frame so that `renderLoop` isn't called again, effectively pausing the game. If we do not have an identifier for a queued animation frame, then we are currently paused, and we would like to call `requestAnimationFrame` to resume the game. -->

JavaScript가 Rust와 WebAssembly를 구동하기 때문에 이것이 우리가 해야 할 전부이며 Rust 소스를 변경할 필요가 없습니다.
<!-- Because the JavaScript is driving the Rust and WebAssembly, this is all we need to do, and we don't need to change the Rust sources. -->


`requestAnimationFrame`이 반환한 식별자를 추적하기 위해 `animationId` 변수를 도입했습니다. 대기 중인 애니메이션 프레임이 없으면 이 변수를 `null`로 설정합니다.
<!-- We introduce the `animationId` variable to keep track of the identifier returned by `requestAnimationFrame`. When there is no queued animation frame, we set this variable to `null`. -->

```js
let animationId = null;

// 이 함수는 `requestAnimationFrame`의 결과가 
//`animationId`에 할당된다는 점을 제외하고는 이전과 동일합니다
const renderLoop = () => {
  drawGrid();
  drawCells();

  universe.tick();

  animationId = requestAnimationFrame(renderLoop);
};
```

'animationId' 값을 검사한다면 게임이 일시 중지되었는지 여부를 확인할 수 있을 것입니다.
<!-- At any instant in time, we can tell whether the game is paused or not by inspecting the value of `animationId`: -->

```js
const isPaused = () => {
  return animationId === null;
};
```

이제 재생/일시 중지 버튼을 클릭하면 게임이 현재 일시 중지 또는 재생 중인지 확인하고 각각 `renderLoop` 애니메이션을 재개하거나 다음 애니메이션 프레임을 취소합니다. 또한 다음에 클릭할 때 버튼이 수행할 작업을 반영하도록 버튼의 텍스트 아이콘을 업데이트합니다.
<!-- Now, when the play/pause button is clicked, we check whether the game is currently paused or playing, and resume the `renderLoop` animation or cancel the next animation frame respectively. Additionally, we update the button's text icon to reflect the action that the button will take when clicked next. -->

```js
const playPauseButton = document.getElementById("play-pause");

const play = () => {
  playPauseButton.textContent = "⏸";
  renderLoop();
};

const pause = () => {
  playPauseButton.textContent = "▶";
  cancelAnimationFrame(animationId);
  animationId = null;
};

playPauseButton.addEventListener("click", event => {
  if (isPaused()) {
    play();
  } else {
    pause();
  }
});
```

마지막으로, 우리는 이전에 `requestAnimationFrame(renderLoop)`을 직접 호출하여 게임과 애니메이션을 시작했지만,  `play` 매소드의 호출로 게임을 시작하고 싶습니다.

<!-- Finally, we were previously kick-starting the game and its animation by calling `requestAnimationFrame(renderLoop)` directly, but we want to replace that with a call to `play` so that the button gets the correct initial text icon. -->

```diff
// 이것은  `requestAnimationFrame(renderLoop)`이었습니다.
play();
```

[http://localhost:8080/](http://localhost:8080/)을 새로고침하면 이제 버튼을 클릭하여 게임을 일시 중지하고 다시 시작할 수 있습니다!
<!-- Refresh [http://localhost:8080/](http://localhost:8080/) and we should now be able to pause and resume the game by clicking on the button! -->

## `"click"` 이벤트에서 셀 상태 토글
<!-- ## Toggling a Cell's State on `"click"` Events -->

이제 게임을 일시 중지할 수 있으므로 셀을 클릭하여 변경하는 기능을 추가할 차례입니다.
<!-- Now that we can pause the game, it's time to add the ability to mutate the cells by clicking on them. -->

세포를 토글한다는 것은 그 상태를 살아있는 상태에서 죽은 상태로 또는 죽은 상태에서 살아있는 상태로 바꾸는 것입니다. `wasm-game-of-life/src/lib.rs`의 `Cell`에 `toggle` 메소드를 추가합니다.
<!-- To toggle a cell is to flip its state from alive to dead or from dead to alive. Add a `toggle` method to `Cell` in `wasm-game-of-life/src/lib.rs`: -->

```rust
impl Cell {
    fn toggle(&mut self) {
        *self = match *self {
            Cell::Dead => Cell::Alive,
            Cell::Alive => Cell::Dead,
        };
    }
}
```

주어진 행과 열에서 셀의 상태를 토글하려면 행과 열 쌍을 주소로 변환하여 셀 벡터에 넣고 해당 인덱스에 있는 셀에 대해 토글 메서드를 호출합니다.
<!-- To toggle the state of a cell at given row and column, we translate the row and column pair into an index into the cells vector and call the toggle method on the cell at that index: -->

```rust
/// JavaScript로 내보낼 Public methods.
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn toggle_cell(&mut self, row: u32, column: u32) {
        let idx = self.get_index(row, column);
        self.cells[idx].toggle();
    }
}
```


이 메소드는 JavaScript에서 호출할 수 있도록 `#[wasm_bindgen]` 주석이 달린 `impl` 블록 내에서 정의됩니다.
<!-- This method is defined within the `impl` block that is annotated with `#[wasm_bindgen]` so that it can be called by JavaScript. -->

`wasm-game-of-life/www/index.js`에서 `<canvas>` 요소의 클릭 이벤트를 수신하고 클릭 이벤트의 페이지 상대 좌표를 캔버스 상대 좌표로 변환한 다음 행으로 변환합니다. 및 열, `toggle_cell` 메서드를 호출하고 마지막으로 장면을 다시 그립니다.
<!-- In `wasm-game-of-life/www/index.js`, we listen to click events on the `<canvas>` element, translate the click event's page-relative coordinates into canvas-relative coordinates, and then into a row and column, invoke the `toggle_cell` method, and finally redraw the scene. -->

```js
canvas.addEventListener("click", event => {
  const boundingRect = canvas.getBoundingClientRect();

  const scaleX = canvas.width / boundingRect.width;
  const scaleY = canvas.height / boundingRect.height;

  const canvasLeft = (event.clientX - boundingRect.left) * scaleX;
  const canvasTop = (event.clientY - boundingRect.top) * scaleY;

  const row = Math.min(Math.floor(canvasTop / (CELL_SIZE + 1)), height - 1);
  const col = Math.min(Math.floor(canvasLeft / (CELL_SIZE + 1)), width - 1);

  universe.toggle_cell(row, col);

  drawGrid();
  drawCells();
});
```

`wasm-game-of-life`에서 `wasm-pack build`로 다시 빌드한 다음 [http://localhost:8080/](http://localhost:8080/) 다시 새로 고침하면 이제 우리 고유의 패턴을 그릴 수 있습니다. 셀을 클릭하여 상태를 전환합니다.

## 연습문제

* [`<input type="range">`][input-range] 위젯을 도입하여 애니메이션 프레임당 발생하는 틱 수를 제어합니다.

* 클릭 시 우주를 임의의 초기 상태로 재설정하는 버튼을 추가합니다. 우주를 모든 죽은 세포로 재설정하는 또 다른 버튼입니다.

* 'Ctrl + 클릭' 시 타겟 셀 중앙에 [글라이더](https://en.wikipedia.org/wiki/Glider_(Conway%27s_Life))를 삽입합니다. 'Shift + 클릭'에서 펄서를 삽입합니다

<!-- Rebuild with `wasm-pack build` in `wasm-game-of-life`, then refresh [http://localhost:8080/](http://localhost:8080/) again and we can now draw our own patterns by clicking on the cells and toggling their state.

## Exercises

* Introduce an [`<input type="range">`][input-range] widget to control how many ticks occur per animation frame.

* Add a button that resets the universe to a random initial state when clicked. Another button that resets the universe to all dead cells.

* On `Ctrl + Click`, insert a [glider](https://en.wikipedia.org/wiki/Glider_(Conway%27s_Life)) centered on the target cell. On `Shift + Click`, insert a pulsar. -->

[input-range]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/range
