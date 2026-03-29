# 8 — Testing

This document explains how the MiniC test suite is organised and how to add
new tests.

---

## Overview

All tests live in the `tests/` directory as **integration tests**. They use
only the public API of the `mini_c` library crate — there are no
`#[cfg(test)]` blocks inside source files. Run the full suite with:

```bash
cargo test
```

```
tests/
├── parser.rs        # Parser tests (literals, expressions, statements, …)
├── program.rs       # Full-program parsing from fixture files
├── type_checker.rs  # Type-checking tests
├── interpreter.rs   # End-to-end execution tests
├── stdlib.rs        # Standard library function tests
└── fixtures/        # MiniC source files used by program.rs
    ├── empty.minic
    ├── function_single.minic
    ├── function_with_block.minic
    ├── full_program.minic
    ├── invalid_syntax.minic
    └── …
```

---

## The Five Test Files

### `tests/parser.rs` — Parser Tests

Tests individual parser functions (`literal`, `expression`, `statement`,
`fun_decl`, etc.) using **inline strings**. Each test focuses on one small
construct and verifies either the AST node produced or that invalid input
is rejected.

Parser functions return `IResult<&str, T>`: `Ok((remaining_input, value))`
on success or `Err(…)` on failure. Tests use `assert_eq!` for the success
case and `assert!(…is_err())` for the failure case.

Expression and statement parsers return `ExprD<()>` or `StatementD<()>`.
To compare just the expression shape (ignoring the `ty: ()` field), map
over the result:

```rust
#[test]
fn test_primary_literal() {
    assert_eq!(
        expression("42").map(|(r, e)| (r, e.exp)),
        Ok(("", Expr::Literal(Literal::Int(42))))
    );
}
```

When you need to verify that *all* input is consumed (not just the prefix),
wrap the parser in `all_consuming(…)`:

```rust
assert!(all_consuming(expression)("1 + 2)").is_err());
```

---

### `tests/program.rs` — Full-Program Parsing Tests

Tests parsing of **complete MiniC programs** from fixture files in
`tests/fixtures/`. Use this when the program is long enough that an inline
string would be hard to read, or when you want to reuse a program across
multiple tests.

The helper `parse_program_file("name.minic")` reads the fixture, parses it
with `all_consuming(program)`, and returns a `Result<UncheckedProgram, …>`.

```rust
#[test]
fn test_parse_function_single() {
    let prog = parse_program_file("function_single.minic")
        .expect("should parse");
    assert_eq!(prog.functions.len(), 1);
    assert_eq!(prog.functions[0].name, "foo");
}

#[test]
fn test_parse_invalid_syntax_fails() {
    assert!(parse_program_file("invalid_syntax.minic").is_err());
}
```

---

### `tests/type_checker.rs` — Type-Checking Tests

Tests the type checker with short programs as inline strings. Each test
parses a program and then type-checks it, asserting either success or a
specific type error.

A helper `parse_and_type_check(src)` combines both steps:

```rust
#[test]
fn test_type_check_undeclared_var() {
    let result = parse_and_type_check("void main() x = y");
    assert!(result.is_err());
    assert!(result.unwrap_err().message.contains("undeclared"));
}

#[test]
fn test_type_check_int_float_coercion() {
    assert!(parse_and_type_check(
        "void main() { int x = 1; float y = x + 3.14 }"
    ).is_ok());
}
```

---

### `tests/interpreter.rs` — End-to-End Tests

Tests the complete pipeline: parse → type-check → interpret. These are the
highest-level tests, and they verify that programs produce correct output or
correct runtime errors.

A helper `run(src)` runs the full pipeline and returns `Ok(())` or
`Err(RuntimeError)`. Output side effects (e.g., `print`) are not captured
by the test — these tests mainly check return status and runtime errors:

```rust
#[test]
fn test_factorial() {
    assert!(run(
        "int factorial(int n)
           if n <= 1 then return 1 else return n * factorial(n - 1)
         void main() {
           int r = factorial(10);
           print(r)
         }"
    ).is_ok());
}

#[test]
fn test_out_of_bounds() {
    let result = run(
        "void main() { int[] a = [1, 2]; int x = a[5] }"
    );
    assert!(result.is_err());
}
```

---

### `tests/stdlib.rs` — Standard Library Tests

Tests the built-in functions directly, without going through the full
pipeline. These are fast, focused unit-style tests for `print_fn`, `pow_fn`,
`sqrt_fn`, and `NativeRegistry`.

```rust
#[test]
fn test_pow_int_args() {
    let result = pow_fn(vec![Value::Int(2), Value::Int(10)]);
    assert_eq!(result, Ok(Value::Float(1024.0)));
}

#[test]
fn test_default_registry_contains_all_stdlib() {
    let r = NativeRegistry::default();
    assert!(r.lookup("print").is_some());
    assert!(r.lookup("sqrt").is_some());
}
```

---

## Inline Strings vs Fixture Files

| Use | When |
|-----|------|
| Inline string | Short, focused test; program fits in one or two lines |
| Fixture file | Multi-line program; program shared across several tests |

Fixture files go in `tests/fixtures/` with a `.minic` extension.

---

## Worked Example: Adding a New Interpreter Test

Suppose you have added a `min(int, int) → int` function to the interpreter
and want to test it end to end.

**Step 1** — Open `tests/interpreter.rs`.

**Step 2** — Add a test that defines `min` in a MiniC program and checks
the result:

```rust
#[test]
fn test_stdlib_min() {
    assert!(run(
        "int min(int a, int b)
           if a <= b then return a else return b
         void main() {
           int r = min(3, 7);
           print(r)
         }"
    ).is_ok());
}
```

**Step 3** — Run `cargo test test_stdlib_min` to verify your test passes.

If you also want to add a unit test for `min` as a native stdlib function,
open `tests/stdlib.rs` and follow the same pattern as `test_pow_int_args`.

---

## Naming Conventions

- Test functions: `test_<construct>_<scenario>`
  e.g., `test_integer_positive`, `test_if_with_else`, `test_out_of_bounds`
- Fixture files: descriptive, lowercase, with `.minic` extension
  e.g., `function_with_block.minic`, `factorial.minic`

---

**What to read next →** [README.md](../README.md)
