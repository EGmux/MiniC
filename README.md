# MiniC

MiniC is a small C-like programming language implemented in Rust. It is
designed as a teaching tool: the entire pipeline — parser, type checker, and
interpreter — is intentionally small so you can read and understand every part
of it. A complete MiniC program looks like this:

```c
int factorial(int n)
  if n <= 1 then
    return 1
  else
    return n * factorial(n - 1)

void main() {
  int result = factorial(10);
  print(result)
}
```

---

## Quick Start

```bash
cargo build          # compile the project
cargo test           # run the full test suite (129 tests)
cargo run            # run the sample main program
```

---

## Documentation

Read the documents in order for a complete picture of the project, or jump
directly to the topic you need.

| # | File | What you will learn |
|---|------|---------------------|
| 1 | [Language reference](docs/01-language.md) | What you can write in MiniC: types, statements, operators |
| 2 | [Pipeline overview](docs/02-pipeline.md) | How source code travels from text to execution |
| 3 | [The AST](docs/03-ast.md) | How a MiniC program is represented in memory |
| 4 | [The parser](docs/04-parser.md) | How source text is turned into an AST |
| 5 | [The type checker](docs/05-type-checker.md) | How type errors are caught before running |
| 6 | [The interpreter](docs/06-interpreter.md) | How a type-checked program is executed |
| 7 | [The standard library](docs/07-stdlib.md) | Built-in functions and how to add new ones |
| 8 | [Testing](docs/08-testing.md) | How the test suite is organised and how to add tests |

---

## Project Layout

```
src/
├── ir/           # AST node definitions
├── parser/       # Source text → unchecked AST
├── semantic/     # Type checker: unchecked AST → checked AST
├── environment/  # Shared symbol table used by type checker and interpreter
├── interpreter/  # Tree-walking interpreter
└── stdlib/       # Built-in functions (print, sqrt, pow, …)

tests/            # Integration tests (all tests live here)
docs/             # This documentation
```
