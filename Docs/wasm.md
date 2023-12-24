


- [WebAssembly support in W3CE](#webassembly-support-in-w3ce)
  - [What is WASM?](#what-is-wasm)
    - [Pros](#pros)
    - [Cons](#cons)
  - [WASM and W3CE](#wasm-and-w3ce)
    - [Loading and initialization](#loading-and-initialization)
    - [Native calling convention and type mapping](#native-calling-convention-and-type-mapping)
    - [Callback handling tip](#callback-handling-tip)


# WebAssembly support in W3CE

W3CE now supports [WASM](https://webassembly.org/) as a scripting backend for custom maps, meant to supplement the more traditional JASS and Lua VMs.  

## What is WASM?

Feel free to skip this section if you already know about WASM. This is meant to serve as a primer for newcomers to WASM.

WASM is a simple, portable compilation target like CIL or JVM. Various languages support compiling to WASM:
* C
* C++
* Rust
* AssemblyScript
* Kotlin
* ... among [others](https://github.com/mbasso/awesome-wasm#languages)

By compiling your program to WASM, it can then be run in a number of compliant WASM VMs at _near-native_ or interpreter speeds, like [V8](https://v8.dev/) (Chromium), [SpiderMonkey](https://spidermonkey.dev/) (Firefox), [Wasmer](https://wasmer.io/), [Wasmtime](https://wasmtime.dev/) or **[Wasm3](https://github.com/wasm3/wasm3)**, which is what W3CE currently uses, among others.

Here is a quick summary outlining the pros and cons of WASM (as implemented by Wasm3) compared to Lua and JASS.

### Pros
* Great execution speed, [outperforming PUC-Rio Lua](https://github.com/wasm3/wasm3/blob/main/docs/Performance.md#wasm3-vs-other-languages) by a factor of ~4x
* * JASS is obviously left in the dust
* Supports [many different languages](https://github.com/mbasso/awesome-wasm#languages) - pick your poison!
* Environment agnostic - you can easily reuse most of your code outside of W3CE and vice-versa.
* Safety-oriented - WASM was built with explicit sandboxing in mind for use on the Web, and a compliant VM will never allow untrusted code to break the sandbox.
* Tooling maturity - WASM has an established ecosystem of tooling, including compilers, optimizers, parsers, code generators, and more.

### Cons
* Some languages map poorly onto the current revision of WASM, mostly due to a lack of *opaque types* and *garbage collection*, somewhat limiting their viability for use inside WASM, or crippling interoperation with the host environment. [^1]
* Lack of standards around interacting with the host environment (i.e. Web, or WC3 in our case). [^2]
* WASM by itself is a relatively low-level target, making it unsuitable for writing code by hand - you need an existing language that compiles to WASM.

[^1]: This is mostly only an issue on Web targets, where it is crucial for code inside WASM to be able to seamlessly and performantly interop with JavaScript. For our purposes in W3CE it is almost a non-issue.

[^2]: This is relatively easily solved by having supporting libraries on the WASM side, as well as having clearly-defined native/wasm interop semantics.

## WASM and W3CE

Here you will find the various technical details on the WASM integration in W3CE and how to actually make use of this feature. Note that for now, there is no established tooling to make WASM-enabled WC3 maps easily, so for the moment this is aimed at those who want to take the plunge and investigate.

Rust support for W3CE will come in the future to make this process more straightforward. If you would like to make an integration targeting another language - feel free to do so and reach out if you need help!

### Loading and initialization

W3CE will look for a `war3map.wasm` file in the root of the map MPQ, and if one is found - it will attempt to load it. No additional flags are needed - the presence of the `.wasm` binary is enough to trigger this feature.

These exports are supported in a `war3map.wasm`
| export | kind | purpose |
| --- | --- | --- |
| `main()`   | **mandatory** | analogue of `main` in Lua and JASS; called first during map load |
| `config()` | **mandatory** | like `main` but runs in the lobby to initialize map metadata |
| `native_callback(i32 callback_id)` | **mandatory** | called by the runtime to dispatch `code` callbacks registered through `TriggerAddAction` etc. |
| `i32 malloc(i32 size, i32 align)`<br>`free(i32 ptr, i32 size, i32 align)` | **mandatory** | called by the runtime to allocate and free memory within the WASM linear memory [^3] [^4] [^5] |
| `init()` | *optional* | runs before `main` and `config` in both map and lobby contexts |

[^3]: At the moment this is only used for backtrace allocation, however, there may be more use-cases in the future.  
[^4]: The `align` argument in both `malloc` and `free` can be freely ignored by the underlying implementation, and the `size` argument can be ignored only in `free`.  
The runtime guarantees to properly supply these values regardless in case the implementation wants to make use of them. As an example: a `malloc(32, 4)` call will be always matched with a `free(ptr, 32, 4)` call, if it is called. 
[^5]: The runtime doesn't necessarily call `free` for each `malloc` - the docs will specify which side is responsible for deallocating memory in each case.

### Native calling convention and type mapping

All regular WC3 natives are exposed to the WASM environment, but because WASM only supports primitive types (i32, i64, f32, f64) in function signatures, certain steps are taken to remap types.

The following table describes types are handled by the environment, in *argument* and *return* positions.

| type | position | mapping |
| --- | --- | --- |
| `integer` | argument/return | converted to an `i32` |
| `boolean` | argument/return | converted to an `i32` |
| `real` | argument/return | converted to an `f32` |
| `handle` | argument/return | converted to an `i32` in both directions |
| `string` | argument | converted to an `i32` interpreted as a `char*` pointer to a nul-terminated C string in linear memory |
| `string` | return | converted to an `char* buf, i32* len` **pointer pair** in **argument** position, appended to the end of the function signature [^6] [^7] |
| `code` | argument | converted to an `i32`, which is then used in `native_callback` [^8]|
| `code` | return | `code` returns are unsupported in JASS |

[^6]: `buf` is a pointer to a buffer inside linear memory, `len` is a pointer to an `i32` indicating the maximum length of the buffer. The runtime will write the string up to `len`, truncating if necessary, and write the actual length to `len`. Pointers are always passed as `i32`-s.
[^7]: It's possible that an option to use `malloc` to allocate the string and return a pointer instead will be added. Please let us know if you'd prefer this variant instead!
[^8]: When it is time for the engine to invoke the callback, the `native_callback` export will be called with the previously-passed `i32` value, allowing to associate the call with a function or a closure.

### Callback handling tip

Because WASM does not support creating dynamic exports, all callbacks are handled via the unified `native_callback` interface. When you register a callback using e.g. `TriggerAddAction`, the environment remembers the `i32` value you passed to it. When that trigger is invoked, the engine will call `native_callback` with that value.

You're supposed to use this single `i32` to then invoke the correct callback. A simple way of doing this is to have a plain array with function pointers or closures, and then simply calling `callbacks[idx]()` inside `native_callback`. When you're registering a `code` action, make sure to remember to add your callback to your internal array!

Examples:
| original | remapped |
| --- | --- |
| `string GetPlayerName(player p)` | `void GetPlayerName(i32 p, char* buf, i32 len)` |
| `string GetStoredString(gamecache gc, string mk, string k)` | `void GetStoredString(i32 p, char* mk, char* k, char* buf, i32* len)` |
| `real GetUnitX(unit u)` | `f32 GetUnitX(i32 u)` |
| `nothing SetUnitPositionLoc(unit u, location loc)` | `void SetUnitPositionLoc(i32 u, i32 loc)` |
