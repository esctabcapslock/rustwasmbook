<!-- # Testing Conway's Game of Life -->
# 생명 게임 테스트하기

이제 생명 게임의 Rust 구현이 브라우저에서 JavaScript를 이용해 완성되었습니다. 이제 Rust로 생성된 WebAssembly 함수를 테스트하는 것에 대해 이야기해 보겠습니다.

<!-- Now that we have our Rust implementation of the Game of Life rendering in the browser with JavaScript, let's talk about testing our Rust-generated WebAssembly functions. -->

우리는 우리가 기대하는 출력을 제공하는지 확인하기 위해 `tick` 기능을 테스트할 것입니다.

<!-- We are going to test our `tick` function to make sure that it gives us the output that we expect. -->

다음으로 `wasm_game_of_life/src/lib.rs` 파일의 기존 `impl Universe` 블록 내부에 일부 setter 및 getter 함수를 만들고 싶습니다. 우리는 `set_width`와 `set_height` 함수를 생성하여 다양한 크기의 `Universe`를 생성할 것입니다.

<!-- Next, we'll want to create some setter and getter  functions inside our existing `impl Universe` block in the `wasm_game_of_life/src/lib.rs` file. We are going to create a `set_width` and a `set_height` function so we can create `Universe`s of different sizes. -->

```rust
#[wasm_bindgen]
impl Universe { 
    // ...

    /// 우주의 width 설정
    ///
    /// 세포들을 죽은 상태로 설정
    pub fn set_width(&mut self, width: u32) {
        self.width = width;
        self.cells = (0..width * self.height).map(|_i| Cell::Dead).collect();
    }

    /// 우주의 height 설정
    ///
    /// 세포들을 죽은 상태로 설정
    pub fn set_height(&mut self, height: u32) {
        self.height = height;
        self.cells = (0..self.width * height).map(|_i| Cell::Dead).collect();
    }

}
```

우리는 `#[wasm_bindgen]` 속성 없이 `wasm_game_of_life/src/lib.rs` 파일 안에 또 다른 `impl Universe` 블록을 생성할 것입니다. 여기에는 JavaScript에 노출하고 싶지 않은 테스트에 필요한 몇 가지 기능이 있습니다. Rust로 생성된 WebAssembly 함수는 빌린 참조자를 반환할 수 없습니다. attribute를 사용하여 Rust에서 생성된 WebAssembly를 컴파일하고 발생하는 오류를 살펴보십시오.
<!-- We are going to create another `impl Universe` block inside our `wasm_game_of_life/src/lib.rs` file without the `#[wasm_bindgen]` attribute. There are a few functions we need for testing that we don't want to expose to our JavaScript. Rust-generated WebAssembly functions cannot return borrowed references. Try compiling the Rust-generated WebAssembly with the attribute and take a look at the errors you get. -->

우리는 `Universe`의 `cells`의 내용을 가져오기 위해 `get_cells`를 구현할 것입니다. 우리는 또한 `Universe`의 특정 행과 열에 있는 `cells`를 `Alive`로 설정할 수 있도록 `set_cells` 함수를 작성할 것입니다.
<!-- We are going to write the implementation of `get_cells` to get the contents of the `cells` of a `Universe`. We'll also write a `set_cells` function so we can set `cells` in a specific row and column of a `Universe` to be `Alive`. -->

```rust
impl Universe {
    /// 셀의 상태를 얻습니다.
    pub fn get_cells(&self) -> &[Cell] {
        &self.cells
    }

    
    /// 각 셀의 행과 열을 배열로 전달하여
    /// 해당하는 셀이 살아있도록 설정합니다.
    pub fn set_cells(&mut self, cells: &[(u32, u32)]) {
        for (row, col) in cells.iter().cloned() {
            let idx = self.get_index(row, col);
            self.cells[idx] = Cell::Alive;
        }
    }

}
```

이제 `wasm_game_of_life/tests/web.rs` 파일에서 테스트를 생성할 것입니다.
<!-- Now we're going to create our test in the `wasm_game_of_life/tests/web.rs` file. -->

그렇게 하기 전에 파일에 이미 하나의 작업을 테스트해야 합니다. Rust가 생성한 WebAssembly 테스트는 `wasm-game-of-life` 디렉토리에서 `wasm-pack test --chrome --headless`를 실행하여 작동하는지 확인할 수 있습니다. `--firefox`, `--safari` 및 `--node` 옵션을 사용하여 해당 브라우저에서 코드를 테스트할 수도 있습니다.
<!-- Before we do that, there is already one working test in the file. You can confirm that the Rust-generated WebAssembly test is working by running `wasm-pack test --chrome --headless` in the `wasm-game-of-life` directory. You can also use the `--firefox`, `--safari`, and `--node` options to test your code in those browsers. -->

`wasm_game_of_life/tests/web.rs` 파일에서 `wasm_game_of_life` 크레이트와 `Universe` 유형을 내보내야 합니다.
<!-- In the `wasm_game_of_life/tests/web.rs` file, we need to export our
`wasm_game_of_life` crate and the `Universe` type. -->

```rust
extern crate wasm_game_of_life;
use wasm_game_of_life::Universe;
```

`wasm_game_of_life/tests/web.rs` 파일에서 우리는 우주선을 생성하는 기능을 만들 것입니다.

우리는 `tick` 함수을 호출할 우주선 입력을 원할 것이고 한 번의 틱 후에 얻을 것으로 예상되는 우주선을 원할 것입니다. `input_spaceship` 함수에서 우주선을 생성하기 위해 `Alive`로 초기화하려는 셀을 선택했습니다. `input_spaceship`의 한 tick 후의 위치가 `expected_spaceship` 함수에서 수동으로 계산되었습니다. 한 틱 후 입력된 우주선의 셀이 예상된 우주선과 동일한 것을 직접 확인할 수 있습니다.

<!-- In the `wasm_game_of_life/tests/web.rs` file we'll want to create some spaceship builder functions.

We'll want one for our input spaceship that we'll call the `tick` function on and we'll want the expected spaceship we will get after one tick. We picked the cells that we want to initialize as `Alive` to create our spaceship in the `input_spaceship` function. The position of the spaceship in the `expected_spaceship` function after the tick of the `input_spaceship` was calculated manually. You can confirm for yourself that the cells of the input spaceship after one tick is the same as the expected spaceship. -->

```rust
#[cfg(test)]
pub fn input_spaceship() -> Universe {
    let mut universe = Universe::new();
    universe.set_width(6);
    universe.set_height(6);
    universe.set_cells(&[(1,2), (2,3), (3,1), (3,2), (3,3)]);
    universe
}

#[cfg(test)]
pub fn expected_spaceship() -> Universe {
    let mut universe = Universe::new();
    universe.set_width(6);
    universe.set_height(6);
    universe.set_cells(&[(2,1), (2,3), (3,2), (3,3), (4,2)]);
    universe
}
```

이제 `test_tick` 함수에 대한 구현을 작성합니다. 먼저 `input_spaceship()`과 `expected_spaceship()`의 인스턴스를 만듭니다. 그런 다음 `input_universe`에서 `tick`을 호출합니다. 마지막으로 `assert_eq!` 매크로를 사용하여 `get_cells()`를 호출하여 `input_universe`와 `expected_universe`가 동일한 `Cell` 배열 값을 갖는지 확인합니다. 우리는 코드 블록에 `#[wasm_bindgen_test]` 속성을 추가하여 Rust에서 생성한 WebAssembly 코드를 테스트하고 `wasm-pack test`를 사용하여 WebAssembly 코드를 테스트할 수 있습니다.

<!-- Now we will write the implementation for our `test_tick` function. First, we create an instance of our `input_spaceship()` and our `expected_spaceship()`. Then, we call `tick` on the `input_universe`. Finally, we use the `assert_eq!` macro to call `get_cells()` to ensure that `input_universe` and `expected_universe` have the same `Cell` array values. We add the `#[wasm_bindgen_test]` attribute to our code block so we can test our Rust-generated WebAssembly code and use `wasm-pack test` to test the WebAssembly code. -->

```rust
#[wasm_bindgen_test]
pub fn test_tick() {
    // 우주선을 테스트하기 위해 작은 우주를 만듧시다
    let mut input_universe = input_spaceship();

    // 이것은 우리 우주에서 한 tick 후 우주선의 모습이어야 합니다.
    let expected_universe = expected_spaceship();

    // `tick`을 호출한 다음 `우주`의 셀들이 동일한지 확인합니다.
    input_universe.tick();
    assert_eq!(&input_universe.get_cells(), &expected_universe.get_cells());
}
```

`wasm-pack test --firefox --headless`를 실행하여 `wasm-game-of-life` 디렉토리 내에서 테스트를 실행합니다.
<!-- Run the tests within the `wasm-game-of-life` directory by running
`wasm-pack test --firefox --headless`. -->
