# zinc-design

Design documents and user manual for **Zinc**: a Rust-inspired systems language that keeps memory safety and thread safety, drops the borrow checker, uses automatic reference counting, and implements generics primarily by dictionary passing.

The compiler and standard library are in [zinc-lang/zinc](https://github.com/zinc-lang/zinc).

## User manual

The Chinese edition in `user-manual/chinese/` is the source of truth.

| Edition | Path |
|---------|------|
| Chinese (source) | [user-manual/chinese/](user-manual/chinese/) |
| English | [user-manual/english/](user-manual/english/) |

Spanish, French, Japanese, and Korean translations are in open pull requests.

## Proposals

Language-design changes go in `proposals/`, using `00-template.zh.md` or `00-template.en.md`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Zinc is MIT-licensed and still a proof of concept; design discussion, translations, and proposals are the most useful contributions here. Implementation work belongs in the compiler repository.
