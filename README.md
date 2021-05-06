# Disassembling V8 Bytecode
V8 interpreter generates and caches bytecodes to improve performance. The generated bytecodes can be serialized into files and loaded in later executions of the program. Some tools like `bytenode` use this technique to protect code of published programs.

Sometimes you may want to review the generated bytecodes, or modify the bytecodes for various reasons. You'll need a disassembler to do that. But there is no pulished standard of V8 bytecodes, specification of the bytecodes are different in each version of V8.

It's hard to write a generic disassembler to deal with all versions of V8, creating a disassembler for specific version of V8 would be much more easier.

This document provides a simple guide on how to create a V8 bytecode disassembler for specific V8 version with minimum effort.

![example](https://github.com/noelex/v8dasm/blob/master/example.png)

Code snipets and compilation flags in the following steps are based on V8 version 8.7.220.25. You may need to make slight changes according to your own V8 version. 

## Getting V8 Source Code
First, checkout V8 source code with `gclient` following this [guide](https://v8.dev/docs/source-code)(or [here](https://medium.com/angular-in-depth/how-to-build-v8-on-windows-and-not-go-mad-6347c69aacd4) if you are building on Windows). It' highly recommended to clone V8 source codes to an SSD drive, which would make the building process much faster.

## Determine V8 Version
Next thing to do is to find out which version of V8 to compile. If target program runs in NodeJS, run `node -p process.versions`.

If it's a Electron program, set environment variable `ELECTRON_RUN_AS_NODE` to `1` and execute to program with parameters `-p process.versions`.

## Applying Patches
Checkout source codes for the version determined in last step with version tag (e.g. `git checkout refs/tags/8.7.220.25`). Remember to run `gclient sync` each time after you switch version.

You can apply patches to source codes here if target program uses a patched version of V8. Patches are needed only when they affect interpreter, bytecodes and serializer.

These components usually won't be modified in a downstream code base. If you're not sure, just ignore the patches.

## Modifying V8 Source Code
V8 provides functions to disassemble bytecodes and print object details, but these functions are private, and it does not disassemble bytecodes when deserializing code cache. You'll need to make some changes to the source code to print disassembly after code cache deserialization.

### Print function disassembly after deserialization
In `code-serializer.cc`, insert the following lines after the serialization is completed (after `maybe_result` is successfully converted to `result`) in function `CodeSerializer::Deserialize`:
```
result->GetBytecodeArray().Disassemble(std::cout);
std::cout << std::flush;
```

This enables disassembling for deserialized functions.

### Print function disassembly when SharedFunctionInfoPrint is called
Modify function `SharedFunctionInfo::SharedFunctionInfoPrint` in `objects-printer.cc`, insert the following lines before the last `os << "\n";`:
```
os << "\n; #region SharedFunctionInfoDisassembly\n";
if (this->HasBytecodeArray()) {
    this->GetBytecodeArray().Disassemble(os);
    os << std::flush;
}
os << "; #endregion";
```

This allows nested functions can be disassembled when parent function is disassembled.

### Print object literal details
Modify function `HeapObject::HeapObjectShortPrint` in `objects.cc`, change `OBJECT_BOILERPLATE_DESCRIPTION_TYPE` case branch to:
```
case OBJECT_BOILERPLATE_DESCRIPTION_TYPE:
    len = FixedArray::cast(*this).length();
    os << "<ObjectBoilerplateDescription[" << len << "]>";

    if (len) {
        os << "\n; #region ObjectBoilerplateDescription\n";
        ObjectBoilerplateDescription::cast(*this).
        ObjectBoilerplateDescriptionPrint(os);
        os << "; #endregion";
    }

    break;
```

This allows members defined in object literals to be printed.

### Print fixed array elements
Modify function `HeapObject::HeapObjectShortPrint` in `objects.cc`, change `FIXED_ARRAY_TYPE` case branch to:
```
case FIXED_ARRAY_TYPE:
    len = FixedArray::cast(*this).length();
    os << "<FixedArray[" << len << "]>";

    if (len) {
        os << "\n; #region FixedArray\n";
        FixedArray::cast(*this).FixedArrayPrint(os);
        os << "; #endregion";
    }

    break;
```

This prints elements (functions, strings, numbers, etc.) defined in a fixed array.

## Configuring and Building V8
Follow the guide [here]((https://v8.dev/docs/build-gn#manual)) to build V8, below is an example configuration:
```
is_debug = false
target_cpu = "x64"
v8_enable_backtrace = true
v8_enable_slow_dchecks = true
v8_optimized_debug = false
is_component_build = false
v8_static_library = true
v8_enable_disassembler = true
v8_enable_object_print = true
use_custom_libcxx = false
use_custom_libcxx_for_host = false
v8_use_external_startup_data = false
is_clang = false
enable_iterator_debugging = false
```
The above configuration builds a x64 release version of V8 with disassembler and object print enables, and produce a static library to be used by our custom disassembler. Please note that `target_cpu` and `is_debug` must match target program's V8.

`v8_static_library`, `v8_enable_disassembler` and `v8_enable_object_print` are required to be `true`, you can change other options based on your scenario.

## Compile the Disassembler
Write a program to call `ScriptCompiler::CompileUnboundScript` to load code cache from file. `v8dasm.cpp` is sample program written for Windows, you can modify it to build your own disassembler.

## Disassembling
Run the compiled program to write disassembly into a text file, `v8dasm code.jsc > code.jsc.txt`. Open the file with your favorite editor, if you're using VS Code, you can install `#region folding for VS Code` extension to collapse and expand regions in the disassembly, which will make the code much easier to read.

## Modifying Code Cache
After reviewing the disassembly, you may want to modify the original code cache file. V8 use Adler-32 hash function in `zlib` to compute check sum of the code, and performs integrity check when loading code cache.

To allow the modified code cache to be loaded by target program, you must recalculate Adler-32 hash of the code and rewrite checksum value in the header.

`v8cc.cs` contains a program for validating and rewriting checksum for x64 V8 version 8.7.220.25. You can modify it according to your specific version of V8.

After compiling, run `v8cc validate code.jsc` to verify checksum and `v8cc rehash code.jsc` to overwrite checksum value in the header.