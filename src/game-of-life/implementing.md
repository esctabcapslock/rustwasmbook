# 게임 구현하기

## 디자인
<!-- ## Design -->

들어가기 전에 고려해야 할 몇 가지 디자인 선택 사항이 있습니다.
<!-- Before we dive in, we have some design choices to consider. -->

### 무한한 우주
<!-- ### Infinite Universe -->
Game of Life는 무한한 우주에서 진행되지만 메모리와 컴퓨팅 파워는 무한하지 못합니다. 이 다소 성가신 제한을 해결하기 위해 일반적으로
세 가지 방법 중 하나를 통해 제공됩니다.
<!-- The Game of Life is played in an infinite universe, but we do not have infinite
memory and compute power. Working around this rather annoying limitation usually
comes in one of three flavors: -->

<!-- 1. Keep track of which subset of the universe has interesting things happening,
   and expand this region as needed. In the worst case, this expansion is
   unbounded and the implementation will get slower and slower and eventually
   run out of memory. -->
1.  우주의 어느 장소에서 흥미로운 일이 일어나고 있는지 추적하고,
    필요에 따라 이 영역을 확장합니다. 최악의 경우 이 확장은
    제한이 없기 때문에 구현이 점점 느려지고 결국
    메모리 부족을 야기할 것입니다.

2. 가장자리에 있는 셀의 인접한 셀의 수가 중간에 있는 세포보다  적은 고정된 크기의 우주를 만듦니다.
    하지만 무한 글라이더처럼 우주의 끝에 도달하는 패턴이 제거된다는 단점이 있습니다.
  <!-- 2. Create a fixed-size universe, where cells on the edges have fewer neighbors
    than cells in the middle. The downside with this approach is that infinite
    patterns, like gliders, that reach the end of the universe are snuffed out. -->
3. 고정된 크기지만 반복되는 우주를 만듭니다. 여기에서 가장자리의 셀은 우주의 반대쪽 셀과 연결되어 있습니다. 따라서 글라이더는 영원히 계속 달릴 수 있습니다.
<!-- 3. Create a fixed-size, periodic universe, where cells on the edges have
   neighbors that wrap around to the other side of the universe. Because
   neighbors wrap around the edges of the universe, gliders can keep running
   forever. -->

우리는 세 번째 방식을 구현할 것입니다.
<!-- We will implement the third option. -->

### Rust와 JavaScript 연결하기
<!-- ### Interfacing Rust and JavaScript -->

> ⚡ 이것은 이 튜토리얼에서 이해해야 할 가장 중요한 개념 중 하나입니다!
<!-- > ⚡ This is one of the most important concepts to understand and take away from
> this tutorial! -->

JavaScript의 garbage-collected heap(여기에 `Object`, `Array` 및 DOM 트리가 할당되어 있음) 공간은 Rust 값이 있는 WebAssembly의 선형 메모리 공간과 다릅니다. WebAssembly는 현재 garbage-collected heap에 직접 액세스할 수 없습니다(2018년 4월 현재 [인터페이스 유형][interface-types]으로 변경될 예정임). 반면 JavaScript는 WebAssembly 선형 메모리 공간을 읽고 쓸 수 있지만 스칼라 값(`u8`, `i32`, `f64` 등)의 [`ArrayBuffer`][array-buf]로만 가능합니다. ..). WebAssembly 함수는 스칼라 값도 사용하고 반환합니다. 이것들은 모든 WebAssembly 및 JavaScript 통신을 구성하는 빌딩 블록입니다.

<!-- JavaScript's garbage-collected heap — where `Object`s, `Array`s, and DOM nodes are allocated — is distinct from WebAssembly's linear memory space, where our Rust values live. WebAssembly currently has no direct access to the garbage-collected heap (as of April 2018, this is expected to change with the ["Interface Types" proposal][interface-types]). JavaScript, on the other hand, can read and write to the WebAssembly linear memory space, but only as an [`ArrayBuffer`][array-buf] of scalar values (`u8`, `i32`, `f64`, etc...). WebAssembly functions also take and return scalar values. These are the building blocks from which all WebAssembly and JavaScript communication is constituted. -->

[interface-types]: https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md
[array-buf]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer

'wasm_bindgen'은 Rust와 WebAssembly의 경계를 넘어 복합한 구조를 작업하는 방법에 대한 일반적인 이해를 정의합니다. 여기에는 Rust 구조를 랩핑하고, 사용성을 위해 JavaScript 클래스에 포인터를 래핑하거나 Rust에 JavaScript 객체 테이블을 인덱싱하는 작업이 포함됩니다. `wasm_bindgen`은 매우 편리하지만 데이터 표현과 이 경계를 넘어 전달되는 값과 구조를 고려할 필요가 없습니다. 단지 인터페이스 디자인을 구현하기 위한 도구로 생각하십시오.

<!-- `wasm_bindgen` defines a common understanding of how to work with compound structures across this boundary. It involves boxing Rust structures, and wrapping the pointer in a JavaScript class for usability, or indexing into a table of JavaScript objects from Rust. `wasm_bindgen` is very convenient, but it does not remove the need to consider our data representation, and what values and structures are passed across this boundary. Instead, think of it as a tool for implementing the interface design you choose. -->

WebAssembly와 JavaScript 간의 인터페이스를 설계할 때 다음 속성을 고려하고자 합니다
<!-- When designing an interface between WebAssembly and JavaScript, we want to optimize for the following properties: -->
<!-- 
1. **Minimizing copying into and out of the WebAssembly linear memory.**
   Unnecessary copies impose unnecessary overhead. -->

1. **WebAssembly 선형 메모리의 복사를 최소화합니다.**
  불필요한 복사는 불필요한 오버헤드를 야기합니다.

2. **직렬화[^1] 및 역직렬화 최소화.** 
  복사와 마찬가지로 직렬화 및 역직렬화도 오버헤드를 부과하고 종종 복사도 부과합니다. 불투명 핸들을 데이터 구조에 전달할 수 있다면(한 쪽에서 직렬화한 뒤 WebAssembly 선형 메모리의 알려진 위치에 복사하고, 다른 쪽에서 역직렬화하는 대신) 많은 오버헤드를 줄일 수 있습니다. `wasm_bindgen`은 JavaScript `Object` 또는 boxed Rust 구조에 대한 불투명 핸들을 정의하기 때문에 작업에 도움이 됩니다

<!-- 2. **Minimizing serializing and deserializing.** Similar to copies, serializing and deserializing also imposes overhead, and often imposes copying as well. If we can pass opaque handles to a data structure — instead of serializing it on one side, copying it into some known location in the WebAssembly linear memory, and deserializing on the other side — we can often reduce a lot of overhead. `wasm_bindgen` helps us define and work with opaque handles to JavaScript `Object`s or boxed Rust structures. -->

일반적으로 좋은 JavaScript↔WebAssembly 인터페이스 디자인은 길고 수명이 긴 데이터 구조가 WebAssembly 선형 메모리에 있는 Rust 유형으로 구현되고 JavaScript에 불투명 핸들로 노출되는 경우가 많습니다. JavaScript는 이러한 불투명 핸들을 취하고, 데이터를 변환하고, 많은 계산을 수행하고, 데이터를 쿼리하고, 궁극적으로 복사 가능한 작은 결과를 반환하는 내보낸 WebAssembly 함수를 호출합니다. 계산의 작은 결과만 반환함으로써 JavaScript의 garbage-collected heap과 WebAssembly 선형 메모리 사이에서 양쪽으로 많은 것들을 복사 및/또는 직렬화하는 것을 피할 수 있습니다.

<!-- As a general rule of thumb, a good JavaScript↔WebAssembly interface design is often one where large, long-lived data structures are implemented as Rust types that live in the WebAssembly linear memory, and are exposed to JavaScript as opaque handles. JavaScript calls exported WebAssembly functions that take these opaque handles, transform their data, perform heavy computations, query the data, and ultimately return a small, copy-able result. By only returning the small result of the computation, we avoid copying and/or serializing everything back and forth between the JavaScript garbage-collected heap and the WebAssembly linear memory. -->
<!-- 
### Interfacing Rust and JavaScript in our Game of Life -->
### 생명게임에서 Life에서 Rust와 JavaScript 간의 인터페이스

피해야 할 몇 가지 위험을 열거하는 것으로 시작하겠습니다. 우리는 매 틱마다 WebAssembly 선형 메모리 안팎으로 전체 우주를 복사하고 싶지 않습니다. 우리는 우주의 모든 셀에 개체를 할당하고 싶지 않으며 각 셀을 읽고 쓰기 위해 경계를 넘는 호출을 부과하고 싶지도 않습니다.

<!-- Let's start by enumerating some hazards to avoid. We don't want to copy the whole universe into and out of the WebAssembly linear memory on every tick. We do not want to allocate objects for every cell in the universe, nor do we want to impose a cross-boundary call to read and write each cell. -->

이것이 우리를 다음과 같이 구현하도록 할 것입니다. WebAssembly 선형 메모리에 있고 각 셀에 대한 바이트가 있는 평면 배열로 우주를 나타낼 수 있습니다. '0'은 죽은 세포이고 '1'은 살아있는 세포입니다.
 
메모리에서 4 x 4 우주는 다음과 같습니다.

<!-- Where does this leave us? We can represent the universe as a flat array that lives in the WebAssembly linear memory, and has a byte for each cell. `0` is a dead cell and `1` is a live cell.
 
Here is what a 4 by 4 universe looks like in memory: -->

![Screenshot of a 4 by 4 universe](../images/game-of-life/universe.png)

우주의 주어진 행과 열에서 셀의 실제 주소를 찾기 위해, 우리는 다음 공식을 사용할 수 있습니다.
<!-- To find the array index of the cell at a given row and column in the universe,
we can use this formula: -->

```text
index(row, column, universe) = row * width(universe) + column
```
<!-- index(row, column, universe) = row * width(universe) + column -->

우리는 우주의 세포를 자바스크립트에 노출시키는 몇 가지 방법이 있습니다. 먼저 `우주`를 구현하기 위하여 ['std::fmt::Display']를 구현합니다. 이는 텍스트 문자로 렌더링된 셀의 Rust `String`을 생성하는 데 사용할 수 있습니다. 이 Rust String은 WebAssembly 선형 메모리에서 JavaScript의 garbage-collected heap에 있는 JavaScript 문자열로 복사된 다음 HTML `textContent`를 설정하여 표시됩니다. 챕터의 후반부에서, 우리는 이 구현을 진화시켜 우주의 세포들을 겹겹이 사이에 복사하는 것을 피하고 `<canvas>`로 렌더링할 것이다.

<!-- We have several ways of exposing the universe's cells to JavaScript. To begin, we will implement [`std::fmt::Display`][`Display`] for `Universe`, which we can use to generate a Rust `String` of the cells rendered as text characters. This Rust String is then copied from the WebAssembly linear memory into a JavaScript String in the JavaScript's garbage-collected heap, and is then displayed by setting HTML `textContent`. Later in the chapter, we'll evolve this implementation to avoid copying the universe's cells between heaps and to render to `<canvas>`. -->

*또 다른 실행 가능한 디자인 대안은 전체 우주를 JavaScript에 노출하는 대신 Rust가 각 틱 후에 상태를 변경한 모든 셀의 목록을 반환하는 것입니다. 이렇게 하면 JavaScript는 렌더링할 때 전체 유니버스를 반복할 필요가 없으며 관련 하위 집합만 반복할 수 있습니다. 이 변화-기반 설계는 구현하기가 약간 더 어렵다는 단점이 있습니다.*

<!-- *Another viable design alternative would be for Rust to return a list of every cell that changed states after each tick, instead of exposing the whole universe to JavaScript. This way, JavaScript wouldn't need to iterate over the whole universe when rendering, only the relevant subset. The trade off is that this delta-based design is slightly more difficult to implement.* -->

<!-- ## Rust Implementation -->
## Rust 구현

이전 장에서 초기 프로젝트 템플릿을 복제했습니다. 이제 해당 프로젝트 템플릿을 사용할 것입니다.
<!-- In the last chapter, we cloned an initial project template. We will modify that
project template now. -->

<!-- Let's begin by removing the `alert` import and `greet` function from
`wasm-game-of-life/src/lib.rs`, and replacing them with a type definition for
cells: -->

`wasm-game-of-life/src/lib.rs`에서 `alert` 가져오기 및 `greet` 기능을 제거하고 셀과 관련된 자료형들을 정의하는 것으로 시작하겠습니다.


```rust
#[wasm_bindgen]
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Cell {
    Dead = 0,
    Alive = 1,
}
```


각 셀이 단일 바이트로 표시되도록 `#[repr(u8)]`이 있는 것이 중요합니다. `Dead`는 `0`이고 `Alive`는 `1`에 대응된다는 것도 중요하므로 덧셈을 통해 셀의 살아있는 이웃을 쉽게 계산할 수 있습니다.

다음으로, 우주를 정의합시다. 우주에는 너비와 높이가 있고 길이가 `width * height`인 셀의 벡터가 있습니다.

<!-- 
It is important that we have `#[repr(u8)]`, so that each cell is represented as a single byte. It is also important that the `Dead` variant is `0` and that the `Alive` variant is `1`, so that we can easily count a cell's live neighbors with addition.

Next, let's define the universe. The universe has a width and a height, and a vector of cells of length `width * height`. -->

```rust
#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<Cell>,
}
```

주어진 행과 열에 있는 셀에 액세스하려면 앞에서 설명한 대로 행과 열을 인덱스로 변환하여 셀 벡터로 변환합니다.
<!-- To access the cell at a given row and column, we translate the row and column into an index into the cells vector, as described earlier: -->

```rust
impl Universe {
    fn get_index(&self, row: u32, column: u32) -> usize {
        (row * self.width + column) as usize
    }

    // ...
}
```

세포의 다음 상태를 계산하기 위해 우리는 얼마나 많은 이웃이 살아 있는지 세어야 합니다. 그렇게 하기 위해 `live_neighbor_count` 메소드를 작성해 봅시다!
<!-- In order to calculate the next state of a cell, we need to get a count of how many of its neighbors are alive. Let's write a `live_neighbor_count` method to do just that! -->

```rust
impl Universe {
    // ...

    fn live_neighbor_count(&self, row: u32, column: u32) -> u8 {
        let mut count = 0;
        for delta_row in [self.height - 1, 0, 1].iter().cloned() {
            for delta_col in [self.width - 1, 0, 1].iter().cloned() {
                if delta_row == 0 && delta_col == 0 {
                    continue;
                }

                let neighbor_row = (row + delta_row) % self.height;
                let neighbor_col = (column + delta_col) % self.width;
                let idx = self.get_index(neighbor_row, neighbor_col);
                count += self.cells[idx] as u8;
            }
        }
        count
    }
}
```

`live_neighbor_count` 매소드는 변화와 나머지 연산을 사용하여 `if`문을 통해 우주의 가장자리를 특수 케이스로 사용하는 것을 방지합니다. 변화량 `-1`을 적용할 때 `self.height - 1`을 *더합니다.* `1`을 빼는 것을 나머지 연산이 대체하도록 합니다. `row`와 `column`은 `0`이 될 수 있는데, 여기서 `1`을 빼려고 하면 unsigned integer underflow가 발생합니다.

<!-- The `live_neighbor_count` method uses deltas and modulo to avoid special casing the edges of the universe with `if`s. When applying a delta of `-1`, we *add* `self.height - 1` and let the modulo do its thing, rather than attempting to subtract `1`. `row` and `column` can be `0`, and if we attempted to subtract `1` from them, there would be an unsigned integer underflow. -->

이제 현재 세대에서 다음 세대를 계산하는 데 필요한 모든 것이 있습니다! 게임의 각 규칙은 `match` 표현식에 대한 조건으로의 직접적인 번역을 따릅니다. 또한 JavaScript가 틱이 발생하는 시점을 제어하기를 원하기 때문에 이 메소드를 `#[wasm_bindgen]` 블록 안에 넣어 JavaScript에 노출되도록 할 것입니다.

<!-- Now we have everything we need to compute the next generation from the current one! Each of the Game's rules follows a straightforward translation into a condition on a `match` expression. Additionally, because we want JavaScript to control when ticks happen, we will put this method inside a `#[wasm_bindgen]` block, so that it gets exposed to JavaScript. -->

```rust
/// JavaScript로 내보낼 Public methods.
#[wasm_bindgen]
impl Universe {
    pub fn tick(&mut self) {
        let mut next = self.cells.clone();

        for row in 0..self.height {
            for col in 0..self.width {
                let idx = self.get_index(row, col);
                let cell = self.cells[idx];
                let live_neighbors = self.live_neighbor_count(row, col);

                let next_cell = match (cell, live_neighbors) {
                    // 규칙 1: 살아있는 이웃이 2개 미만인 살아있는 세포는 
                    // 마치 인구 부족으로 인한 것처럼 죽습니다.
                    (Cell::Alive, x) if x < 2 => Cell::Dead,
                    // 규칙 2: 살아있는 이웃이 2~3개 있는 살아있는 세포는 
                    // 다음 세대에 계속 살아갑니다.
                    (Cell::Alive, 2) | (Cell::Alive, 3) => Cell::Alive,
                    // 규칙 3: 살아있는 이웃이 세 개 이상 있는 살아있는 
                    // 세포는 마치 인구 과잉으로 인해 죽는 것처럼 보입니다.
                    (Cell::Alive, x) if x > 3 => Cell::Dead,
                    // 규칙 4: 정확히 3개의 살아있는 이웃이 있는 죽은 세포는 
                    // 마치 번식에 의한 것처럼 살아있는 세포가 됩니다.
                    (Cell::Dead, 3) => Cell::Alive,
                    // 다른 모든 셀은 동일한 상태로 유지됩니다.
                    (otherwise, _) => otherwise,
                };

                next[idx] = next_cell;
            }
        }

        self.cells = next;
    }

    // ...
}
```

지금까지 우주의 상태는 세포의 벡터로 표현되었습니다. 사람이 읽을 수 있도록 기본 텍스트 렌더러를 구현해 보겠습니다. 아이디어는 우주를 한 줄씩 텍스트로 작성하고 살아 있는 각 셀에 대해 유니코드 문자 `◼`("검정색 중간 정사각형")를 인쇄하는 것입니다. 죽은 세포의 경우 `◻`("흰색 중간 정사각형")를 인쇄합니다.
<!-- So far, the state of the universe is represented as a vector of cells. To make this human readable, let's implement a basic text renderer. The idea is to write the universe line by line as text, and for each cell that is alive, print the Unicode character `◼` ("black medium square"). For dead cells, we'll print `◻` (a "white medium square"). -->

Rust의 표준 라이브러리에서 [`Display`] 트레잇을 구현함으로써 우리는 사용자가 보는 방식으로 구조를 설정하는 방법을 추가할 수 있습니다. 이것은 또한 자동으로 [`to_string`] 메서드를 제공합니다.
<!-- By implementing the [`Display`] trait from Rust's standard library, we can add a
way to format a structure in a user-facing manner. This will also automatically
give us a [`to_string`] method. -->

[`Display`]: https://doc.rust-lang.org/1.25.0/std/fmt/trait.Display.html
[`to_string`]: https://doc.rust-lang.org/1.25.0/std/string/trait.ToString.html

```rust
use std::fmt;

impl fmt::Display for Universe {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        for line in self.cells.as_slice().chunks(self.width as usize) {
            for &cell in line {
                let symbol = if cell == Cell::Dead { '◻' } else { '◼' };
                write!(f, "{}", symbol)?;
            }
            write!(f, "\n")?;
        }

        Ok(())
    }
}
```

마지막으로 살아있는 세포와 죽은 세포의 흥미로운 패턴과 'render' 메서드로 우주를 초기화하는 생성자를 정의합니다.
<!-- Finally, we define a constructor that initializes the universe with an interesting pattern of live and dead cells, as well as a `render` method: -->

```rust
/// JavaScript로 내보낼 Public methods.
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn new() -> Universe {
        let width = 64;
        let height = 64;

        let cells = (0..width * height)
            .map(|i| {
                if i % 2 == 0 || i % 7 == 0 {
                    Cell::Alive
                } else {
                    Cell::Dead
                }
            })
            .collect();

        Universe {
            width,
            height,
            cells,
        }
    }

    pub fn render(&self) -> String {
        self.to_string()
    }
}
```

이것으로 Game of Life 구현의 Rust 부분의 절반이 완료되었습니다!
<!-- With that, the Rust half of our Game of Life implementation is complete! -->

`wasm-game-of-life` 디렉토리 내에서 `wasm-pack build`를 실행하여 WebAssembly로 다시 컴파일하십시오.
<!-- Recompile it to WebAssembly by running `wasm-pack build` within the `wasm-game-of-life` directory. -->

<!-- ## Rendering with JavaScript -->
## 자바스크립트로 랜더링하기

먼저 `<pre>` 요소를 `wasm-game-of-life/www/index.html`에 추가하여 `<script>` 태그 바로 위의 우주를 렌더링해 보겠습니다.
<!-- First, let's add a `<pre>` element to `wasm-game-of-life/www/index.html` to render the universe into, just above the `<script>` tag: -->

```html
<body>
  <pre id="game-of-life-canvas"></pre>
  <script src="./bootstrap.js"></script>
</body>
```

추가적으로, 우리는 웹 페이지의 중앙에 `<pre>`가 표시되길 원합니다. CSS flex box를 사용하여 이 작업을 수행할 수 있습니다. `wasm-game-of-life/www/index.html`의 `<head>` 안에 다음 `<style>` 태그를 추가합니다.
<!-- Additionally, we want the `<pre>` centered in the middle of the Web page. We can use CSS flex boxes to accomplish this task. Add the following `<style>` tag inside `wasm-game-of-life/www/index.html`'s `<head>`: -->

```html
<style>
  body {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }
</style>
```

`wasm-game-of-life/www/index.js` 파일 상단에서 이전의 `greet` 기능 대신 `Universe`를 가져오도록 import 구문을 수정하겠습니다.
<!-- At the top of `wasm-game-of-life/www/index.js`, let's fix our import to bring in the `Universe` rather than the old `greet` function: -->

```js
import { Universe } from "wasm-game-of-life";
```

<!-- Also, let's get that `<pre>` element we just added and instantiate a new
universe: -->
또한 방금 추가한 `<pre>` 요소를 가져와서 새 우주의 인스턴스를 생성하겠습니다.

```js
const pre = document.getElementById("game-of-life-canvas");
const universe = Universe.new();
```

JavaScript는 [`requestAnimationFrame` 루프][requestAnimationFrame]에서 실행됩니다. 각 반복에서 현재 우주를 `<pre>`로 그린 다음 `Universe::tick`을 호출합니다.
<!-- The JavaScript runs in [a `requestAnimationFrame`
loop][requestAnimationFrame]. On each iteration, it draws the current universe
to the `<pre>`, and then calls `Universe::tick`. -->

[requestAnimationFrame]: https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame

```js
const renderLoop = () => {
  pre.textContent = universe.render();
  universe.tick();

  requestAnimationFrame(renderLoop);
};
```

렌더링 프로세스를 시작하려면 렌더링 루프의 첫 번째 반복에 대한 초기 호출을 수행하기만 하면 됩니다.
<!-- To start the rendering process, all we have to do is make the initial call for
the first iteration of the rendering loop: -->

```js
requestAnimationFrame(renderLoop);
```

개발 서버가 여전히 실행 중인지 확인하십시오. (`wasm-game-of-life/www` 디렉토리 내부에서 `npm run start` 명령어 실행). 이곳
[http://localhost:8080/](http://localhost:8080/) 에서 볼 수 있습니다.
<!-- Make sure your development server is still running (run `npm run start` inside
`wasm-game-of-life/www`) and this is what
[http://localhost:8080/](http://localhost:8080/) should look like: -->

[![텍스트 렌더링을 사용한 Game of Life 구현 스크린샷](../images/game-of-life/initial-game-of-life-pre.png)](../images/game-of-life/initial-game-of-life-pre.png)

<!-- ## Rendering to Canvas Directly from Memory -->
## 메모리에서 직접 캔버스로 렌더링

Rust에서 `String`을 생성(및 할당)한 다음 `wasm-bindgen`이 이를 유효한 JavaScript 문자열로 변환하도록 하면 우주 셀의 불필요한 복사본이 만들어집니다. JavaScript 코드는 이미 우주의 너비와 높이를 알고 있고 셀을 직접 구성하는 WebAssembly의 선형 메모리를 읽을 수 있으므로 `render` 메서드를 수정하여 셀 배열의 시작 부분에 대한 포인터를 반환토록 만들 것입니다..

<!-- Generating (and allocating) a `String` in Rust and then having `wasm-bindgen` convert it to a valid JavaScript string makes unnecessary copies of the universe's cells. As the JavaScript code already knows the width and height of the universe, and can read WebAssembly's linear memory that make up the cells directly, we'll modify the `render` method to return a pointer to the start of the cells array. -->

또한 유니코드 텍스트를 렌더링하는 대신 [Canvas API]를 사용하도록 바꿀 것입니다. 튜토리얼의 나머지 부분에서 이 방식을 사용할 것입니다.

<!-- Also, instead of rendering Unicode text, we'll switch to using the [Canvas API]. We will use this design in the rest of the tutorial. -->

[Canvas API]: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API

`wasm-game-of-life/www/index.html` 내부에서 이전에 추가한 `<pre>`를 새롭게 렌더링할 `<canvas>`로 교체해 보겠습니다. (`<body>` 테그 내부 및 JavaScript를 로드하는 `<script>` 테그 앞에 있어야 합니다.):
<!-- Inside `wasm-game-of-life/www/index.html`, let's replace the `<pre>` we added earlier with a `<canvas>` we will render into (it too should be within the `<body>`, before the `<script>` that loads our JavaScript): -->

```html
<body>
  <canvas id="game-of-life-canvas"></canvas>
  <script src='./bootstrap.js'></script>
</body>
```

Rust 구현에서 필요한 정보를 얻으려면 우주의 너비, 높이 및 셀 배열에 대한 포인터에 대한 getter 함수를 더 추가해야 합니다. 이 모든 것은 JavaScript에도 노출됩니다. `wasm-game-of-life/src/lib.rs`에 다음을 추가합시다.

<!-- To get the necessary information from the Rust implementation, we'll need to add some more getter functions for a universe's width, height, and pointer to its cells array. All of these are exposed to JavaScript as well. Make these additions to `wasm-game-of-life/src/lib.rs`: -->

```rust
/// Public methods, exported to JavaScript.
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn width(&self) -> u32 {
        self.width
    }

    pub fn height(&self) -> u32 {
        self.height
    }

    pub fn cells(&self) -> *const Cell {
        self.cells.as_ptr()
    }
}
```

다음으로, `wasm-game-of-life/www/index.js`에서 `wasm-game-of-life`에서 `Cell`을 가져와서 캔버스에 렌더링할 때 사용할 상수를 정의해 보겠습니다.

<!-- Next, in `wasm-game-of-life/www/index.js`, let's also import `Cell` from `wasm-game-of-life`, and define some constants that we will use when rendering to the canvas: -->

```js
import { Universe, Cell } from "wasm-game-of-life";

const CELL_SIZE = 5; // px
const GRID_COLOR = "#CCCCCC";
const DEAD_COLOR = "#FFFFFF";
const ALIVE_COLOR = "#000000";
```

이제 이 JavaScript 코드의 나머지 부분을 다시 작성하여 더 이상 `<pre>`의 `textContent`에 쓰지 않고 대신 `<canvas>`에 그릴 것입니다.
<!-- Now, let's rewrite the rest of this JavaScript code to no longer write to the `<pre>`'s `textContent` but instead draw to the `<canvas>`: -->

```js
// 우주를 생성하고 너비와 높이를 구하십시오.
const universe = Universe.new();
const width = universe.width();
const height = universe.height();

// 모든 셀이 1px 테두리를 갖게끔 캔버스 크기를 설정합니다.
const canvas = document.getElementById("game-of-life-canvas");
canvas.height = (CELL_SIZE + 1) * height + 1;
canvas.width = (CELL_SIZE + 1) * width + 1;

const ctx = canvas.getContext('2d');

const renderLoop = () => {
  universe.tick();

  drawGrid();
  drawCells();

  requestAnimationFrame(renderLoop);
};
```

셀 사이의 격자를 그리기 위해 동일한 간격의 가로선과 세로선들을 그립니다. 이 선은 십자형으로 교차하여 격자 형태를 형성합니다.

<!-- To draw the grid between cells, we draw a set of equally-spaced horizontal lines, and a set of equally-spaced vertical lines. These lines criss-cross to form the grid. -->

```js
const drawGrid = () => {
  ctx.beginPath();
  ctx.strokeStyle = GRID_COLOR;

  // Vertical lines.
  for (let i = 0; i <= width; i++) {
    ctx.moveTo(i * (CELL_SIZE + 1) + 1, 0);
    ctx.lineTo(i * (CELL_SIZE + 1) + 1, (CELL_SIZE + 1) * height + 1);
  }

  // Horizontal lines.
  for (let j = 0; j <= height; j++) {
    ctx.moveTo(0,                           j * (CELL_SIZE + 1) + 1);
    ctx.lineTo((CELL_SIZE + 1) * width + 1, j * (CELL_SIZE + 1) + 1);
  }

  ctx.stroke();
};
```
 raw wasm 모듈 `wasm_game_of_life_bg`에 정의된 `memory`를 통해 WebAssembly의 선형 메모리에 직접 액세스할 수 있습니다. 셀을 그리려면 우주의 셀에 대한 포인터를 얻고, 셀 버퍼를 오버레이하는 'Uint8Array'를 구성하고, 각 셀을 반복하고, 셀이 죽었는지 살았는지에 따라 흰색 또는 검은색 직사각형을 각각 그려야 합니다. 포인터와 오버레이로 작업함으로써 모든 틱에서 경계를 가로질러 셀을 복사하는 것을 방지합니다.

<!-- We can directly access WebAssembly's linear memory via `memory`, which is defined in the raw wasm module `wasm_game_of_life_bg`. To draw the cells, we get a pointer to the universe's cells, construct a `Uint8Array` overlaying the cells buffer, iterate over each cell, and draw a white or black rectangle depending on whether the cell is dead or alive, respectively. By working with pointers and overlays, we avoid copying the cells across the boundary on every tick. -->

```js
// 파일 맨 윗부분에서  WebAssembly 메모리를 가져옵니다.
import { memory } from "wasm-game-of-life/wasm_game_of_life_bg";

// ...

const getIndex = (row, column) => {
  return row * width + column;
};

const drawCells = () => {
  const cellsPtr = universe.cells();
  const cells = new Uint8Array(memory.buffer, cellsPtr, width * height);

  ctx.beginPath();

  for (let row = 0; row < height; row++) {
    for (let col = 0; col < width; col++) {
      const idx = getIndex(row, col);

      ctx.fillStyle = cells[idx] === Cell.Dead
        ? DEAD_COLOR
        : ALIVE_COLOR;

      ctx.fillRect(
        col * (CELL_SIZE + 1) + 1,
        row * (CELL_SIZE + 1) + 1,
        CELL_SIZE,
        CELL_SIZE
      );
    }
  }

  ctx.stroke();
};
```

렌더링 프로세스를 시작하려면 위와 동일한 코드를 사용하여 렌더링 루프의 첫 번째 반복을 시작합니다.
<!-- To start the rendering process, we'll use the same code as above to start the first iteration of the rendering loop: -->

```js
drawGrid();
drawCells();
requestAnimationFrame(renderLoop);
```

`requestAnimationFrame()`을 호출하기 *전에* 여기서 `drawGrid()` 및 `drawCells()`를 호출합니다. 이렇게 하는 이유는 수정하기 전에 우주의 *초기* 상태가 그려지기 때문입니다. 대신 단순히 `requestAnimationFrame(renderLoop)`을 호출하면 그려지는 첫 번째 프레임이 실제로는 `universe.tick()`에 대한 첫 번째 호출이 *나중*이 되는 상황이 됩니다. 이는 이 세포의 수명에 대한 두 번째 "틱"입니다. 

<!-- Note that we call `drawGrid()` and `drawCells()` here _before_ we call `requestAnimationFrame()`. The reason we do this is so that the _initial_ state of the universe is drawn before we make modifications. If we instead simply called `requestAnimationFrame(renderLoop)`, we'd end up with a situation where the first frame that was drawn would actually be _after_ the first call to `universe.tick()`, which is the second "tick" of the life of these cells. -->

<!-- ## It Works! -->
## 작동 확인!

`wasm-game-of-life` 디렉토리에서 다음 명령을 실행하여 WebAssembly 및 바인딩 glue를 다시 빌드합니다.

<!-- Rebuild the WebAssembly and bindings glue by running this command from within the root `wasm-game-of-life` directory: -->

```
wasm-pack build
```

개발 서버가 여전히 실행 중인지 확인하십시오. 그렇지 않은 경우 `wasm-game-of-life/www` 디렉토리에서 다음 명령어를 다시 시작하십시오.

<!-- Make sure your development server is still running. If it isn't, start it again from within the `wasm-game-of-life/www` directory: -->

```
npm run start
```

[http://localhost:8080/](http://localhost:8080/)을 새로고침하면 흥미진진한 생명 게임을 맞이할 수 있을 것입니다!
<!-- If you refresh [http://localhost:8080/](http://localhost:8080/), you should be greeted with an exciting display of life! -->

[![Screenshot of the Game of Life implementation](../images/game-of-life/initial-game-of-life.png)](../images/game-of-life/initial-game-of-life.png)

그 외에 [hashlife](https://en.wikipedia.org/wiki/Hashlife)라는 Game of Life를 구현하기 위한 정말 깔끔한 알고리즘도 있습니다. 공격적인 메모이제이션을 사용하며 실제로 *기하급수적으로 빠르게* 미래 세대를 더 오래 실행할 수 있습니다! 이를 감안할 때 이 튜토리얼에서 해시라이프를 구현하지 않은 이유가 궁금할 것입니다. 우리가 Rust와 WebAssembly 통합에 초점을 맞추고 있는 이 텍스트의 범위를 벗어났지만 hashlife에 대해 직접 배우기를 강력히 권장합니다!

<!-- As an aside, there is also a really neat algorithm for implementing the Game of Life called [hashlife](https://en.wikipedia.org/wiki/Hashlife). It uses aggressive memoizing and can actually get *exponentially faster* to compute future generations the longer it runs! Given that, you might be wondering why we didn't implement hashlife in this tutorial. It is out of scope for this text, where we are focusing on Rust and WebAssembly integration, but we highly encourage you to go learn about hashlife on your own! -->

## 연습문제


* 하나의 우주선으로 우주를 초기화하십시오.

* 초기 우주를 하드 코딩하는 대신, 각 셀이 살아 있거나 죽을 확률이 50:50인 임의의 우주를 생성하십시오.

* Initialize the universe with a single space ship.

* Instead of hard-coding the initial universe, generate a random one, where each  cell has a fifty-fifty chance of being alive or dead.

  *힌트: [`js-sys` 크레이트](https://crates.io/crates/js-sys) 를 사용합시다. 이는
  [`Math.random` JavaScript
  함수](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random)와 비슷합니다.*

  <details>
    <summary>정답</summary>
    *먼저,  `js-sys`종속성을 `wasm-game-of-life/Cargo.toml`에 추가합니다:*

    ```toml
    # ...
    [dependencies]
    js-sys = "0.3"
    # ...
    ```

    *그러면 `js_sys::Math::random` 를 통해 동전 던지를 기능을 사용할 수 있습니다.:*

    ```rust
    extern crate js_sys;

    // ...

    if js_sys::Math::random() < 0.5 {
        // Alive...
    } else {
        // Dead...
    }
    ```
  </details>

* 각 셀을 바이트로 나타내면 셀을 쉽게 반복할 수 있지만 메모리 낭비가 발생합니다. 각 바이트는 8비트이지만 각 셀이 살아 있는지 여부를 나타내기 위해 단 하나의 비트만 필요합니다. 각 셀이 단일 비트 공간만 사용하도록 데이터 표현을 리팩터링합니다.
  <!-- * Representing each cell with a byte makes iterating over cells easy, but it comes at the cost of wasting memory. Each byte is eight bits, but we only require a single bit to represent whether each cell is alive or dead. Refactor the data representation so that each cell uses only a single bit of space. -->

  <details>
    <summary>정답</summary>

    Rust에서는 ['fixedbitset' 크레이트 및 'FixedBitSet' 타입](https://crates.io/crates/fixedbitset)을 사용하여 `Vec<Cell>` 대신 셀을 나타낼 수 있습니다.
    <!-- In Rust, you can use [the `fixedbitset` crate and its `FixedBitSet` type](https://crates.io/crates/fixedbitset) to represent cells instead of `Vec<Cell>`: -->

    ```rust
    // Cargo.toml에 종속성을 추가했는지 확인하십시오!
    extern crate fixedbitset;
    use fixedbitset::FixedBitSet;

    // ...

    #[wasm_bindgen]
    pub struct Universe {
        width: u32,
        height: u32,
        cells: FixedBitSet,
    }
    ```
    우주 생성자는 다음과 같이 조정할 수 있습니다.
    <!-- The Universe constructor can be adjusted the following way: -->

    ```rust
    pub fn new() -> Universe {
        let width = 64;
        let height = 64;

        let size = (width * height) as usize;
        let mut cells = FixedBitSet::with_capacity(size);

        for i in 0..size {
            cells.set(i, i % 2 == 0 || i % 7 == 0);
        }

        Universe {
            width,
            height,
            cells,
        }
    }
    ```
    유니버스의 다음 틱에서 셀을 업데이트하려면 `FixedBitSet`의 `set` 메서드를 사용합니다.
    <!-- To update a cell in the next tick of the universe, we use the `set` method of `FixedBitSet`: -->

    ```rust
    next.set(idx, match (cell, live_neighbors) {
        (true, x) if x < 2 => false,
        (true, 2) | (true, 3) => true,
        (true, x) if x > 3 => false,
        (false, 3) => true,
        (otherwise, _) => otherwise
    });
    ```
    비트 시작에 대한 포인터를 JavaScript에 전달하려면 `FixedBitSet`을 슬라이스로 변환한 다음 슬라이스를 포인터로 변환할 수 있습니다.
    <!-- To pass a pointer to the start of the bits to JavaScript, you can convert the `FixedBitSet` to a slice and then convert the slice to a pointer: -->

    ```rust
    #[wasm_bindgen]
    impl Universe {
        // ...

        pub fn cells(&self) -> *const u32 {
            self.cells.as_slice().as_ptr()
        }
    }
    ```
    JavaScript에서 Wasm 메모리에서 `Uint8Array`를 구성할 때 배열의 길이가 더 이상 바이트당 `width * height`가 아니라 `width * height / 8`이라는 주의하십시오:
    <!-- In JavaScript, constructing a `Uint8Array` from Wasm memory is the same as before, except that the length of the array is not `width * height` anymore, but `width * height / 8` since we have a cell per bit rather than per byte: -->

    ```js
    const cells = new Uint8Array(memory.buffer, cellsPtr, width * height / 8);
    ```
    인덱스와 'Uint8Array'가 주어지면 *n*번째 비트가 다음 함수로 설정되었는지 확인할 수 있습니다.
    <!-- Given an index and `Uint8Array`, you can determine whether the
    *n<sup>th</sup>* bit is set with the following function: -->

    ```js
    const bitIsSet = (n, arr) => {
      const byte = Math.floor(n / 8);
      const mask = 1 << (n % 8);
      return (arr[byte] & mask) === mask;
    };
    ```
    이 모든 것을 감안할 때 `drawCells`의 새 버전은 다음과 같습니다.
    <!-- Given all that, the new version of `drawCells` looks like this: -->

    ```js
    const drawCells = () => {
      const cellsPtr = universe.cells();

      // This is updated!
      const cells = new Uint8Array(memory.buffer, cellsPtr, width * height / 8);

      ctx.beginPath();

      for (let row = 0; row < height; row++) {
        for (let col = 0; col < width; col++) {
          const idx = getIndex(row, col);

          // This is updated!
          ctx.fillStyle = bitIsSet(idx, cells)
            ? ALIVE_COLOR
            : DEAD_COLOR;

          ctx.fillRect(
            col * (CELL_SIZE + 1) + 1,
            row * (CELL_SIZE + 1) + 1,
            CELL_SIZE,
            CELL_SIZE
          );
        }
      }

      ctx.stroke();
    };
    ```

  </details>


[^1]: 역자 주) 직렬화는 메모리를 디스크에 저장하거나 네트워크 통신에 사용하기 위한 형식으로 변환하는 것을 말한다. 참조 형식 데이터를 전송할 수는 없지 않는가?

