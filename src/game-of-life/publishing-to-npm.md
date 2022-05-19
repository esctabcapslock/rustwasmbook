# Publishing to npm
# npm에 퍼블리싱하기

이제 작동하는 빠르고 작은 `wasm-game-of-life` 패키지가 있습니다. 다른 JavaScript 개발자가 생명 게임이 필요할 경우 이를 재사용할 수 있도록 npm에 게시할 수 있습니다.
<!-- Now that we have a working, fast, *and* small `wasm-game-of-life` package, we can publish it to npm so other JavaScript developers can reuse it, if they ever need an off-the-shelf Game of Life implementation. -->

## Prerequisites

먼저 [npm 계정이 있는지 확인](https://www.npmjs.com/signup)합니다.
<!-- First, [make sure you have an npm account](https://www.npmjs.com/signup). -->

둘째, 다음 명령어을 실행하여 로컬로 계정에 로그인했는지 확인하십시오.
<!-- Second, make sure you are logged into your account locally, by running this command: -->

```
wasm-pack login
```

## 퍼블리싱
<!-- ## Publishing -->

`wasm-game-of-life` 디렉토리에서 `wasm-pack`을 실행하여 `wasm-game-of-life/pkg` 빌드가 최신 버전인지 확인하십시오:
<!-- Make sure that the `wasm-game-of-life/pkg` build is up to date by running `wasm-pack` inside the `wasm-game-of-life` directory: -->

```
wasm-pack build
```

잠시 시간을 내어 `wasm-game-of-life/pkg`의 내용을 확인하십시오. 이것이 다음 단계에서 npm에 게시할 내용입니다!
<!-- Take a moment to check out the contents of `wasm-game-of-life/pkg` now, this is what we are publishing to npm in the next step! -->

준비가 되면 `wasm-pack publish`를 실행하여 npm에 패키지를 업로드합니다.
<!-- When you're ready, run `wasm-pack publish` to upload the package to npm: -->

```
wasm-pack publish
```

이것이 npm에 게시하는 데 필요한 전부입니다!
<!-- That's all it takes to publish to npm!-->

...다른 사람들도 이 튜토리얼을 수행한 것을 제외하고는 `wasm-game-of-life` 이름이 npm에서 사용되며 마지막 명령은 아마도 작동하지 않았을 것입니다.
<!-- ...except other folks have also done this tutorial, and therefore the `wasm-game-of-life` name is taken on npm, and that last command probably didn't work. -->

`wasm-game-of-life/Cargo.toml`을 열고 `name` 끝에 사용자 이름을 추가하여 패키지를 고유하고 명확하게 만듧니다.
<!--Open up `wasm-game-of-life/Cargo.toml` and add your username to the end of the `name` to disambiguate the package in a unique way: -->

```toml
[package]
name = "wasm-game-of-life-my-username"
```

그런 다음 다시 빌드하고 게시합니다.
<!-- Then, rebuild and publish again: -->

```
wasm-pack build
wasm-pack publish
```

이번에는 작동해야합니다!
<!-- This time it should work! -->
