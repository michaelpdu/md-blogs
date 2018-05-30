# WebAssembly

## Table of Contents

 - [Documents](#documents)
 - [Conferences](#conferences)
 - [Papers](#papers)
 - [Blogs](#blogs)
 - [Github Projects](#github)
 - [Other Summary](#summary)

## Documents

[MDN WebAssembly Documents](https://developer.mozilla.org/en-US/docs/WebAssembly)

Compile C/C++ to wasm

![https://mdn.mozillademos.org/files/14647/emscripten-diagram.png](https://mdn.mozillademos.org/files/14647/emscripten-diagram.png)

Install Emscripten from [http://kripken.github.io/emscripten-site/index.html](http://kripken.github.io/emscripten-site/index.html)

Online WASM Assembler
- [WasmFiddle](https://wasdk.github.io/WasmFiddle/)
- [WasmFiddle++](https://anonyco.github.io/WasmFiddlePlusPlus/)
- [WasmExplorer](https://mbebenita.github.io/WasmExplorer/)

[WebAssembly Org](https://webassembly.org/)

1. [Binary Format](https://webassembly.org/docs/binary-encoding/)
2. [Semantics for Text Format](https://webassembly.org/docs/semantics/)

## Conferences

None

## Papers

None

## Blogs

[EOS节点远程执行代码漏洞](https://www.anquanke.com/post/id/146497)

## Github

[WABT: The WebAssembly Binary Toolkit](https://github.com/WebAssembly/wabt)

1. wat2wasm: translate from WebAssembly text format to the WebAssembly binary format
2. wasm2wat: the inverse of wat2wasm, translate from the binary format back to the text format (also known as a .wat)
3. wasm-objdump: print information about a wasm binary. Similiar to objdump.
4. wasm-interp: decode and run a WebAssembly binary file using a stack-based interpreter
5. wat-desugar: parse .wat text form as supported by the spec interpreter (s-expressions, flat syntax, or mixed) and print "canonical" flat format
6. wasm2c: convert a WebAssembly binary file to a C source and header

Online WABT Demos: [https://cdn.rawgit.com/WebAssembly/wabt/aae5a4b7/demo/index.html](https://cdn.rawgit.com/WebAssembly/wabt/aae5a4b7/demo/index.html)

[Compiler infrastructure and toolchain library for WebAssembly, in C++](https://github.com/WebAssembly/binaryen)

1. wasm-shell: A shell that can load and interpret WebAssembly code. It can also run the spec test suite.
2. wasm-as: Assembles WebAssembly in text format (currently S-Expression format) into binary format (going through Binaryen IR).
3. wasm-dis: Un-assembles WebAssembly in binary format into text format (going through Binaryen IR).
4. wasm-opt: Loads WebAssembly and runs Binaryen IR passes on it.
5. asm2wasm: An asm.js-to-WebAssembly compiler, using Emscripten's asm optimizer infrastructure. This is used by Emscripten in Binaryen mode when it uses Emscripten's fastcomp asm.js backend.
6. wasm2asm: A WebAssembly-to-asm.js compiler (still experimental).
7. s2wasm: A compiler from the .s format emitted by the new WebAssembly backend being developed in LLVM. This is used by Emscripten in Binaryen mode when it integrates with the new LLVM backend.
8. wasm-merge: Combines wasm files into a single big wasm file (without sophisticated linking).
9. wasm-ctor-eval: A tool that can execute C++ global constructors ahead of time. Used by Emscripten.
10. wasm.js: wasm.js contains Binaryen components compiled to JavaScript, including the interpreter, asm2wasm, the S-Expression parser, etc., which allow you to use Binaryen with Emscripten and execute code compiled to WASM even if the browser doesn't have native support yet. This can be useful as a (slow) polyfill.
11. binaryen.js: A standalone JavaScript library that exposes Binaryen methods for creating and optimizing WASM modules.

[Binaryen has built-in fuzzing and reducing capabilities](https://github.com/WebAssembly/binaryen/wiki/Fuzzing)

## Summary

- Curated list of awesome things regarding WebAssembly (wasm) ecosystem: [https://github.com/mbasso/awesome-wasm](https://github.com/mbasso/awesome-wasm)