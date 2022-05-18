<div align="center">

  <h1>The Rust and WebAssembly Book</h1>

  <strong>이 작은 책에는  Rust와 WebAssembly를 사용하는 방법이 담겨 있습니다. 훌륭한 예제와 함께 말이죠..</strong>

  <strong>이 글은 다음 [레포](https://github.com/rustwasm/book)를 번역한 것입니다. </strong>

  <h3>
    <a href="https://rustwasm.github.io/docs/book/">책 읽기</a>
    <span> | </span>
    <a href="https://github.com/rustwasm/book/blob/master/CONTRIBUTING.md">기여</a>
    <span> | </span>
    <a href="https://discordapp.com/channels/442252698964721669/443151097398296587">Chat</a>
  </h3>

  <sub>Built with 🦀🕸 by <a href="https://rustwasm.github.io/">The Rust and WebAssembly Working Group</a></sub>
</div>

## About

이 레포지토리에는 WebAssembly와 Rust를 사용하는 법, 일반적인 워크플로우, 그리리고 어떻게 시작하는지 담겨 있습니다. 이것은 Rust를 통해 멋진 일을 할 수 있도록 당신을 도울 것입니다.


만약 당장 WebAssembly와 Rust를 배우고 싶다면, 이 책을 [온라인][book]에서 읽을 수 있습니다.

[Open issues for improving the Rust and WebAssembly book.][book-issues]

[book-issues]: https://github.com/rustwasm/book/issues

## Building the Book

The book is made using [`mdbook`][mdbook]. To install it you'll need `cargo`
installed. If you don't have any Rust tooling installed, you'll need to install
[`rustup`][rustup] first. Follow the instructions on the site in order to get
setup.

Once you have that done then just do the following:

```bash
$ cargo install mdbook
```

Make sure the `cargo install` directory is in your `$PATH` so that you can run
the binary.

Now just run this command from this directory:

```bash
$ mdbook build
```

This will build the book and output files into a directory called `book`. From
there you can navigate to the `index.html` file to view it in your browser. You
could also run the following command to automatically generate changes if you
want to look at changes you might be making to it:

```bash
$ mdbook serve
```

This will automatically generate the files as you make changes and serves them
locally so you can view them easily without having to call `build` every time.

The files are all written in Markdown so if you don't want to generate the book
to read them then you can read them from the `src` directory.

[mdbook]: https://github.com/rust-lang-nursery/mdBook
[rustup]: https://github.com/rust-lang-nursery/rustup.rs/
[book]: https://esctabcapslock.github.io/rustwasmbook/game-of-life/introduction.html
