# Overview of compiler stages

Hope this article get you better understand and involved in the development. Have fun!

---

### Basic usage
Before you go on, here is a [blog](http://blogs.adobe.com/flascc/2013/03/18/flascc-and-link-time-optimization/) demonstrating the basic 
usage of gcc(same to clang).
 
> Long story short, gcc/clang is a driver. It accepts the inputs, translates the kinds of options; and given the specific options, it calls frontend driver. Frontend driver calls the real preprocessor(if exits as an executable), real compiler and passes the outputs back and to the backend driver. Backend driver handles linking stage.

Frontend driver and backend driver may be acted by driver itself as clang does. GCC uses its backend driver collect2 if exists.

If the intermediate outputs are assembly files, linker will handle them directly.
If the outputs are IR files, linker will go LTO process. 

You can google out lots of articles. Here I won't repeat in details. 

### LTO (Link Time Optimization)

[LTO](http://gcc.gnu.org/wiki/LinkTimeOptimization) originates from [WHOPR](http://gcc.gnu.org/onlinedocs/gccint/WHOPR.html). 

Usually the optimization is done within the compiling module scope (a single source file). So let's say if a function in a file is never used, it'll still go into the final executable file with traditional linking.  


With **LTO**:  
> 1. All of the inputs are firstly compiled into **IR** (Intermediate Representation) files and passed to linker. Since linker can't handle such files, it would pass them to each linker plugin for furthur processing. 

> 2. If any plugin can handle the specific IR file, it claims the file and extracts all the symbols to regiester them back in linker.  

> 3. Linker now has all symbols it needs to process. Surely it can eliminate the unused symbols, plus other optimizations.

> 4. In the end, linker has the native binary files and the binary files which are handled and generated by linker plugins from the IR files, and finally outputs the final executable file.

To fully support LTO, Crossbridge uses the linker `gold`. You may google out a lot of info about it. And in the following sections, `ld` is an alias name of `gold`.



### How crossbridge compiler does
Now you should be familiar with the compiler stages.

Take clang as an example and open the hood:  

Given the `-v` and `--save-temps`, you will see how compiler driver does.

Here we have non-LTO and LTO. Please note their difference.

##### Non-LTO stages
	sdk/usr/bin/clang -v --save-temps hello.c -emit-swf -swf-size=200x200 -o hello.swf

Outputs (omit the trivials):

	"/Users/yeli/repos/cb/sdk/usr/platform/darwin/bin/clang" -cc1 -triple avm2-unknown-freebsd8 -E … -o hello.i -x c hello.c  

With `-cc1`, clang acts as a frontend driver, with `-E` to do preprocessing. 

	"/Users/yeli/repos/cb/sdk/usr/platform/darwin/bin/clang" -cc1 -triple avm2-unknown-freebsd8 -S … -o hello.s -x cpp-output hello.i  

`-cc1` and `-S` compiling and assembling. See that hello.i turns into hello.s. If you take a look at hello.s, it's purely action script except a hack string `#---SPLIT`. So here involves *source code* to *IR* and *IR* to *Actionscript* procesing.

	"/Users/yeli/repos/cb/sdk/usr/bin/avm2-as" -o hello.o hello.s

The real assembling which turns action script into ABC code. 

	"/Users/yeli/repos/cb/sdk/usr/bin/ld" --plugin /Users/yeli/repos/cb/sdk/usr/bin/../../usr/lib/multiplug.dylib -V --verbose …… -o hello.swf --plugin-opt obj-path=hello.swf.lto.abc --plugin-opt also-emit-llvm=hello.swf.lto.bc --plugin-opt lto-as3-1=hello.swf.lto.1.as --plugin-opt lto-as3-2=hello.swf.lto.2.as --plugin-opt swf-size=200x200 --plugin-opt swf-version=18 --plugin-opt mtriple=avm2-unknown-freebsd8 ……… hello.o /Users/yeli/repos/cb/sdk/usr/bin/../../usr/lib//stdlibs_abc/libm.o /Users/yeli/repos/cb/sdk/usr/bin/../../usr/lib//stdlibs_abc/libc.a

Finally the linking stage. Note that `ld` with `--plugin`. So Crossbridge compiler utilizes the linker plugin mechanism even for non-LTO process.

You may want to look at `lvm-gcc-4.2-2.9/gcc/config/avm2/avm2.h` for gcc and `/llvm-3.2/tools/clang/lib/Driver/Tools.cpp` for clang.

`--plugin-opt` for passing options to linker plugin.  

Remember we have a multiplug.dylib here.


###### Sidenote for `#---SPLIT`
Still remember that hack string `#---SPLIT`? It marks the file with two parts:

1. The first part that contains kinds of common functions from libs.
2. The second part that contains only the code from source file.

These 2 parts will be saved separatedly to process, so we don't have many duplicate functions in final ABC file.

##### LTO

Add `-flto` or `-O4`:  

	sdk/usr/bin/clang" -v --save-temps -flto  hello.c -emit-swf -swf-size=200x200 -o hellolto.swf

Outputs (omit the trivials):

	"/Users/yeli/repos/cb/sdk/usr/platform/darwin/bin/clang" -cc1 -triple avm2-unknown-freebsd8 -E ... -o hello.i -x c hello.c

Same as above.

	 "/Users/yeli/repos/cb/sdk/usr/platform/darwin/bin/clang" -cc1 -triple avm2-unknown-freebsd8 -emit-llvm-bc ... -o hello.o -x cpp-output hello.i

Here hello.o is in binary format of LLVM IR. You can translate it back into IR by `llvm-dis`. 

	"/Users/yeli/repos/cb/sdk/usr/bin/ld" --plugin /Users/yeli/repos/cb/sdk/usr/bin/../../usr/lib/multiplug.dylib -V --verbose …… -o hello.swf --plugin-opt obj-path=hello.swf.lto.abc --plugin-opt also-emit-llvm=hello.swf.lto.bc --plugin-opt lto-as3-1=hello.swf.lto.1.as --plugin-opt lto-as3-2=hello.swf.lto.2.as --plugin-opt swf-size=200x200 --plugin-opt swf-version=18 --plugin-opt mtriple=avm2-unknown-freebsd8 ……… hello.o /Users/yeli/repos/cb/sdk/usr/bin/../../usr/lib//stdlibs_abc/libm.o /Users/yeli/repos/cb/sdk/usr/bin/../../usr/lib//stdlibs_abc/libc.a
	
And yes, LTO process doesn't has the stage of assembling.
It just calls linker ld and expects linker plugin to handle.

##### Sum up

|   |    |
|---|----|
|Non-LTO | Source code -> Actionscript -> Final target |
|LTO | Source code -> LLVM IR -> Final target |


### Linker plugin

We know that outputs (native or IR or actionscript) just go into ld for linking. Now we'll see how Crossbridge handles them.

First, suppose you have looked at `binutils/include/plugin-api.h`. So you have an idea of plugin:

1. onload() will be called once plugin is loaded into memory. And plugin has to save the capabilities of linker it needs and register its handlers.

2. Plugin basically implements ld_plugin_claim_file_handler() for claiming the files it supports, ld_plugin_all_symbols_read_handler() for generating native code and ld_plugin_cleanup_handler() for cleaning up.

3. Linker will call ld_plugin_claim_file_handler() of each plugin in the order for unknown files. If one plugin fails, it calls the next until it's resolved. Otherwise, die.	If a plugin claims a file, it extracts the symbols and add_symbols() to linker.

4. Linker has resolved all symbols, by ld_plugin_all_symbols_read_handler() it notifies each plugin to do the generation of code.

5. All done. Linker calls ld_plugin_cleanup_handler() to clean up and unload each plugins.


In Crossbridge, 3 plugins in total are used to support linking for IR and Actionscript:

|Name| Desc| Loc|
|---|---|---|
|**multiplug**| A wrapper plugins. | /gold-plugins/multiplug.cpp|
|**makeswf**| The real plugin that consumes ABC/AS files and gives out final SWF/EXE file| /gold-plugins/makeswf.cpp |
|**LLVMgold**| It takes LLVM IR files and gives out ABC file|/llvm-3.2/tools/gold/gold-plugin.cpp|

The sequence of register is important here in `multiplug`, because:

1. Linker can't handle ABC/IR/Actionscript.
2. `LLVMgold` can only handle LLVM IR and by default generate ABC.
3. Linker will try each plugin in order.

So we have to make sure:  

1. `LLVMgold` returns ok when it sees Actionscript/ABC or not.
2. Right after LLVMgold, pass AS/ABC file to `makeswf`.
3. Let `makeswf` to generate final SWF file.

### Code Generation

In `LLVMgold`, you will see in ld_plugin_all_symbols_read_handler():

1. It calls lto_codegen_write_merged_modules() to get a single big IR file containing all the IR files it claims.
2. Then passes to `opt` for optimizing.
3. Later to `llc` for generating ABC.

In `makeswf`, you will see it actually calls `AlcTool.jar`(/tools/aet/AlcTool.java) to extract symbols and make swf.

