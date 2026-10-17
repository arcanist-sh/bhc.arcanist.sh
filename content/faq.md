+++
title = "FAQ"
description = "Frequently asked questions about the Basel Haskell Compiler"
template = "page.html"
+++

# Frequently Asked Questions

## General

### What is BHC?

BHC (Basel Haskell Compiler) is a new Haskell compiler focused on:
- Native code generation via LLVM
- Predictable performance characteristics
- Modern tooling and developer experience
- Cross-compilation to multiple targets

### How is BHC different from GHC?

| Aspect | BHC | GHC |
|--------|-----|-----|
| Backend | LLVM native | NCG / LLVM |
| Evaluation | Profile-based (strict/lazy) | Lazy by default |
| Target focus | Native, WASM, GPU | General-purpose |
| Maturity | Beta (v0.2.22) | Production-ready |
| Compatibility | Haskell 2010 plus common GHC extensions (compiles all of Pandoc) | Full Haskell + extensions |

BHC is **not** a GHC replacement. It's an alternative compiler for specific use cases where native performance and predictability matter.

### Is BHC production-ready?

**No.** BHC v0.1.0-alpha is a technology preview. It can compile simple programs but lacks many features needed for production use. See "What works?" below.

### Who is building BHC?

BHC is developed by [Raskell](https://arcanist.sh), with the goal of expanding the Haskell ecosystem with specialized tooling.

## Features

### What works in v0.1.0-alpha?

Working features:
- Hello World (`putStrLn "Hello"`)
- Integer literals (`print 42`)
- Arithmetic (`print (1 + 2 * 3)`)
- Let bindings (`let x = 5 in print x`)
- Simple function definitions

### What doesn't work yet?

Not yet implemented:
- `do` notation and monadic operations
- Pattern matching in function definitions
- Type classes
- Imports (except implicit Prelude)
- Lists, tuples, algebraic data types
- Standard library (Data.List, etc.)
- Package management / Hackage

### When will X be supported?

See the [Roadmap](@/roadmap.md) for planned features. We're focusing on:
1. Core language features (pattern matching, type classes)
2. Standard library basics
3. Package ecosystem integration

### Can I use existing Haskell packages?

Not yet. BHC v0.1.0-alpha doesn't support imports or package management. This is planned for future releases.

## Installation

### What platforms are supported?

BHC v0.1.0-alpha supports:
- **Linux x86_64** (Ubuntu, Debian, Fedora, etc.)
- **macOS aarch64** (Apple Silicon M1/M2/M3)

Coming soon:
- macOS x86_64 (Intel)
- Windows x86_64
- Linux aarch64

### How do I install BHC?

```bash
curl -fsSL https://arcanist.sh/bhc/install.sh | sh
```

See [Get Started](@/get-started.md) for detailed instructions.

### How do I uninstall BHC?

```bash
curl -fsSL https://arcanist.sh/bhc/install.sh | sh -s -- --uninstall
```

### I get "command not found" after installing

Add BHC to your PATH:

```bash
echo 'export PATH="$HOME/.bhc/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### I get "library not found" when compiling

Add the RTS library path:

```bash
echo 'export LIBRARY_PATH="$HOME/.bhc/lib:$LIBRARY_PATH"' >> ~/.zshrc
source ~/.zshrc
```

## Compatibility

### Is BHC code compatible with GHC?

**Yes, if you avoid BHC-specific extensions.** Standard Haskell 2010 code that compiles with BHC will also compile with GHC.

BHC-specific extensions (like `BHC2026`, `BHC.StrictDefault`) are namespaced and won't work with GHC.

### Can I mix BHC and GHC in one project?

Not currently. A project must use one compiler. However, you can maintain compatibility by:
1. Writing standard Haskell 2010 code
2. Avoiding BHC-specific extensions
3. Testing with both compilers

### Does BHC support GHC extensions?

BHC supports a subset of GHC extensions for compatibility. See the [Compatibility Charter](@/compatibility.md) for details.

## Contributing

### How can I contribute?

BHC is open source. You can:
- Report bugs on [GitHub](https://github.com/arcanist-sh/bhc/issues)
- Submit pull requests
- Improve documentation
- Share feedback and use cases

### Where can I get help?

- [GitHub Issues](https://github.com/arcanist-sh/bhc/issues) for bugs
- [GitHub Discussions](https://github.com/arcanist-sh/bhc/discussions) for questions
- This documentation site

## License

### What license is BHC under?

BHC is open source. Check the [GitHub repository](https://github.com/arcanist-sh/bhc) for license details.
