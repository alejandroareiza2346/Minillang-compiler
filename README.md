# Minillang-compiler
**Engineering Lead: Alejandro Areiza Alzate**
**Technical Domain: Compiler Engineering / Programming Language Theory / Systems Programming**

---

## 1. Executive Summary and Architectural Vision

This project implements a **complete, multi-phase compiler** for MiniLang, a statically-scoped, integer-typed programming language designed for educational and systems engineering purposes. The compiler transforms MiniLang source code through eight sequential phases — from lexical analysis to bytecode execution on a custom stack-accumulator virtual machine — producing a fully traceable compilation pipeline with no external runtime dependencies. The architecture enforces strict phase isolation: each compilation stage consumes a well-defined input representation and produces a discrete output artifact, enabling independent testing, instrumentation, and replacement of any individual phase without affecting the rest of the pipeline. A Flask-based web interface and a full CLI provide two independent execution surfaces, both backed by the same core compiler module.

---

## 2. Requirement Analysis and Strategic Alignment

- **Functional:** Full compilation pipeline from MiniLang source to executable bytecode: lexical tokenization, recursive-descent parsing, semantic validation with symbol table and initialization analysis, constant-folding optimization, Three Address Code (TAC) intermediate representation generation, accumulator-based assembly code generation, bytecode encoding with two-pass label resolution, and execution on a register-minimal virtual machine; web interface for step-by-step pipeline visualization; CLI with `--emit` flags for per-phase output inspection; `--trace-vm` flag for instruction-level execution tracing.
- **Non-Functional:** Each compiler phase completes in O(n) or O(n log n) time relative to input token count; full test suite executable in under 5 seconds; web interface operable without installation beyond `pip install -r requirements.txt`; all compiler errors carry source line and column references for accurate diagnostic reporting.
- **Strategic Goal:** Demonstration of end-to-end systems programming competency — from formal language theory (grammar, AST construction, semantic analysis) through low-level code generation and virtual machine design — in a single, self-contained, production-structured Python project.

---

## 3. Technical Stack and Infrastructure

- **Core Language:** Python 3.10+
- **Web Interface:** Flask (`web_app.py`) — single-file server exposing the compiler pipeline via HTTP with step-by-step phase output rendering
- **Testing:** pytest — unit and integration tests covering all eight compiler phases (`tests/`)
- **Packaging:** `pyproject.toml` + `pip install -e .` — compiler installable as a local package, enabling `python -m minilang_compiler.compiler` invocation from any working directory
- **CI/CD:** GitHub Actions workflow (`.github/workflows/`) — automated test execution on push and pull request
- **Execution Environment:** Any POSIX-compatible system or Windows with Python 3.10+; no native compilation toolchain required
- **Design Pattern:** Pipeline architecture — eight discrete, sequentially composed transformation stages with typed interfaces between phases; each stage is a standalone module under `minilang_compiler/`

---

## 4. Engineering Logic and Implementation

The compiler implements the canonical multi-phase translation model. Source text enters the lexer and exits the virtual machine as computed output, passing through eight transformation stages:

**Phase 1 — Lexer (`lexer.py`):** Character-by-character tokenization producing a flat token stream. Handles identifiers, integer literals, operators, keywords, and comments. Reports lexical errors with line and column position. O(n) scan complexity.

**Phase 2 — Parser (`parser.py`):** Recursive-descent parser consuming the token stream and constructing an Abstract Syntax Tree (AST) from node types defined in `ast_nodes.py`. Enforces operator precedence (`*`, `/` before `+`, `-`) and mandatory `else` branches on conditional statements. Reports syntactic errors with token-level position context.

**Phase 3 — Semantic Analyzer (`semantic.py`):** Single-pass AST traversal maintaining a symbol table of declared variables and their initialization state. Performs definite-assignment analysis — variables used before initialization produce compile-time errors; variables conditionally initialized (assigned in one branch of an `if-else` but not both) produce diagnostic warnings. Control-flow analysis tracks initialization state across branching structures.

**Phase 4 — Optimizer (`optimizer.py`):** AST-level constant folding pass — evaluates arithmetic expressions over integer literals at compile time, replacing subexpressions with their computed values. Eliminates dead branches in conditionals with statically-known boolean conditions (`if 1 > 0 { ... }` reduces to its true branch unconditionally).

**Phase 5 — IR Generator (`ir.py`):** Lowers the optimized AST to Three Address Code (TAC). Complex expressions are decomposed into sequences of single-operation instructions using compiler-generated temporaries (`t1`, `t2`, ...). Control flow constructs (`if-else`, `while`) are lowered to conditional and unconditional jumps over generated labels.

**Phase 6 — Assembly Generator (`codegen_asm.py`):** Translates TAC to symbolic assembly for a single-accumulator register machine. Each TAC instruction maps to a sequence of `LOAD`, `STORE`, `ADD`, `SUB`, `MUL`, `DIV`, `JMP`, and conditional jump instructions operating on named memory locations.

**Phase 7 — Machine Code Generator (`codegen_machine.py`):** Two-pass assembler converting symbolic assembly to a flat integer bytecode array. Pass one assigns memory addresses to variables, temporaries, and constants; pass two resolves label references to concrete bytecode offsets. Each instruction occupies two array positions: `[opcode, operand]`.

**Phase 8 — Virtual Machine (`runtime_vm.py`):** Fetch-decode-execute interpreter running the bytecode array. State: program counter (PC), accumulator (ACC), and a flat memory array. Supports `IN` (read from input queue) and `OUT` (append to output list) for I/O. Execution terminates on `HALT` opcode.

- **Complexity:** Lexer and each AST pass are O(n) in token/node count. TAC generation is O(n) in AST node count. VM execution is O(k) in bytecode instruction count.
- **Data Structures:** Token list (lexer output), n-ary AST with typed node classes, flat TAC instruction list, symbolic assembly list, integer bytecode array, hash map symbol table, integer memory array (VM).

---

## 5. Quality Assurance and Systematic Testing

The project maintains a structured test suite under `tests/` covering all eight compiler phases independently and end-to-end.

- **Analytical Testing:** Static review of lexer regular expression patterns and parser grammar rules against the MiniLang formal specification; verification of symbol table initialization state transitions across all branching structures; opcode table completeness check against VM dispatch logic.
- **Constructive Testing (Black-Box):** End-to-end program execution tests comparing VM output against expected values for programs covering arithmetic, conditionals, loops, and nested expressions; constant folding validation confirming that `x = 5 + 3 * 2` compiles to `x = 11` with no runtime arithmetic; definite-assignment tests confirming that use-before-initialization produces a compile error, not a runtime fault.
- **Edge Case Handlers:** Division by zero detected at VM execution time with a structured runtime error; unbalanced parentheses reported at parse time with token position; variables conditionally initialized (one branch only) produce a warning rather than an error, allowing programs that guarantee initialization through program logic to proceed; empty input queue on `read` raises a structured `InputExhaustedError`.
- **CI:** GitHub Actions executes `pytest -v` on every push and pull request to `main`.

---

## 6. Security Governance and Compliance

- **Input Handling:** All MiniLang source input is processed exclusively through the lexer's character-level scanner; no `eval()`, `exec()`, or dynamic code execution is used at any compilation phase. The compiler cannot be used to execute arbitrary Python from MiniLang source.
- **Web Interface:** The Flask web application is configured for local development use (`127.0.0.1:5000`). For any public deployment, standard WSGI hardening (Gunicorn, reverse proxy, rate limiting) and input size validation on the source code field are required before exposure.
- **Sandboxed Execution:** The MiniLang virtual machine operates on a bounded integer memory array with no filesystem, network, or OS-level access. Bytecode execution cannot escape the VM's controlled memory space.
- **OWASP Alignment:** The web interface mitigates A03 (Injection) by treating all user input as MiniLang source text processed through a formal lexer, never as executable code or SQL.

---

## 7. Deployment and Initialization

**Prerequisites:** Python 3.10+

```bash
# Clone the repository
git clone https://github.com/alejandroareiza2346/Minillang-compiler.git

cd Minillang-compiler

# Create and activate virtual environment
python -m venv .venv

# Linux / macOS
source .venv/bin/activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# Install dependencies and compiler package
pip install -r requirements.txt
pip install -e .
```

**Web interface:**

```bash
python web_app.py
# Access at http://127.0.0.1:5000
```

**CLI usage:**

```bash
# Compile and execute a program with inputs
python -m minilang_compiler.compiler examples/program1.minilang --run --inputs 5 10

# Inspect individual pipeline stages
python -m minilang_compiler.compiler file.minilang --emit tokens
python -m minilang_compiler.compiler file.minilang --emit ast
python -m minilang_compiler.compiler file.minilang --emit ir
python -m minilang_compiler.compiler file.minilang --emit asm
python -m minilang_compiler.compiler file.minilang --emit machine

# Trace VM execution instruction by instruction
python -m minilang_compiler.compiler file.minilang --run --trace-vm --inputs 5
```

**Run test suite:**

```bash
pytest -v
```

---

## 8. Repository Structure

```
Minillang-compiler/
├── .github/workflows/          # GitHub Actions CI pipeline
├── minilang_compiler/
│   ├── tokens.py               # Token type definitions
│   ├── ast_nodes.py            # AST node class hierarchy
│   ├── errors.py               # Structured error formatting
│   ├── lexer.py                # Phase 1 — Lexical analysis
│   ├── parser.py               # Phase 2 — Recursive-descent parsing
│   ├── semantic.py             # Phase 3 — Semantic analysis + symbol table
│   ├── optimizer.py            # Phase 4 — Constant folding
│   ├── ir.py                   # Phase 5 — TAC IR generation
│   ├── codegen_asm.py          # Phase 6 — Assembly code generation
│   ├── codegen_machine.py      # Phase 7 — Bytecode encoding (two-pass)
│   ├── runtime_vm.py           # Phase 8 — Virtual machine interpreter
│   └── compiler.py             # Pipeline orchestrator
├── tests/                      # pytest test suite
├── examples/                   # Sample MiniLang programs
├── docs/                       # Technical specification
├── templates/                  # Flask HTML templates
├── web_app.py                  # Flask web interface
├── pyproject.toml              # Package metadata
├── requirements.txt            # Runtime dependencies
└── GUIA_EXPLICACION_PROYECTO.md  # Phase-by-phase explanation guide
```

---

## 9. MiniLang Language Reference

MiniLang is a statically-scoped, integer-typed imperative language. All values are integers; no floating-point, string, or boolean types exist.

```
// Read two integers and print their product
read a;
read b;
result = a * b;
print result;
end

// Factorial (iterative)
read n;
fact = 1;
i = 1;
while i <= n {
    fact = fact * i;
    i = i + 1;
}
print fact;
end
```

**Supported constructs:** variable assignment, `read`, `print`, `if-else` (both branches required), `while`, arithmetic operators (`+`, `-`, `*`, `/`), relational operators (`<`, `>`, `<=`, `>=`, `==`, `!=`), unary negation.

---

## 10. Professional Background

Project designed and developed by **Alejandro Areiza Alzate**, Computer Engineering student at Universidad Autónoma Latinoamericana (UNAULA), Medellín, and GitHub Developer Program member.

- **LinkedIn:** [Alejandro A. Alzate](https://www.linkedin.com/in/alejandroareizaa/)
- **Research (ORCID):** [0009-0002-2116-6918](https://orcid.org/0009-0002-2116-6918)
- **Certifications:** Microsoft Learn Level 6 — 26,950 XP (Azure Identity, Network Security & SQL Security); Cisco; Google; IBM; OWASP Top 10

---

## 11. License

Distributed under the **MIT License**. See `LICENSE` for full terms.
