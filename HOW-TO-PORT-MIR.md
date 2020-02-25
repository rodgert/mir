# Introduction

  Porting MIR to another target requires implementing a lot of
machine-dependent code. On my estimation this code is about 3K C
lines for target and writing it can take 1 month of work for
experienced person.
  
  This document outlines what should be done for porting MIR.  First I
recommend to port MIR-interpreter and `c2mir` using it.  You need to
write file `mir-<target>.c` for this. Then you can port MIR-generator
by writing file `mir-gen-<target>.c` file.  For more details you can
examine existing files `mir-x86_64.c`, `mir-aarhc64.c`,
`mir-gen-x86_64.c`, and `mir-gen-aarhc64.c`
  
## Common machine dependent functions (file `mir-<target.c>`)

  These function can be used to work with MIR-interpreter and
  MIR-generator.  These functions should be placed in file
  `mir-<target>.c` like `mir-x86_64.c` or `mir-aarch64.c`.  You can
  use MIR internal functions `_MIR_publish_code`,
  `_MIR_get_new_code_addr`, `_MIR_publish_code_by_addr`,
  `_MIR_change_code`, `_MIR_update_code`, and `_MIR_update_code_arr` to
  work with executable code.

  Here is the list of the common function you should provide:
  
  * `void *_MIR_get_thunk (MIR_context_t ctx)` generates small code (thunk) and
    returns address to the code.  The code redirects control flow to
    address stored in it.  The address is not defined by the function
    but can be set by the following function

  * `void _MIR_redirect_thunk (MIR_context_t ctx, void *thunk, void
    *to)` sets up address in the thunk given as the 2nd argument to
    value given by the 3rd argument

  * `void *_MIR_get_wrapper (MIR_context_t ctx, MIR_item_t
    called_func, void *hook_address)` generates a function (`fun1`) code and
    returns its address.  The code calls a function given by
    `hook_address` passing it `ctx` and `called_func` and getting an
    address of another function (`fun2`) which is called at end.  The
    function `fun2` gets arguments which are passed to the generated
    function `fun1`

    * Function `_MIR_get_wrapper` is used for lazy code generation but can be used
      for other purposes in the future.  For lazy code generation,
      `hook_address` is an address of function which generates machine
      code for MIR function `called_func`, redirects MIR function
      thunk to the generated machine code, and returns this code address
    
## Interpreter machine dependent functions (file `mir-target.c`)

  These function are used to implement MIR interpreter.  These
  functions should be also placed in file `mir-<target>.c` and you can
  also use the MIR internal function mentioned above for their
  implementation

  * `void *_MIR_get_bstart_builtin (MIR_context_t ctx)` generates a
    function and returns address to the generated function. The generated
    function has no arguments and returns SP right before the function
    call.  The generated function is used to implement `MIR_BSTART`
    insn in the interpreter
    
  * `void *_MIR_get_bend_builtin (MIR_context_t ctx)` generates a
    function and returns address to the generated function. The
    generated function has one argument and sets up SP from the
    function argument.  The generated function is used to implement
    `MIR_BEND` insn in the interpreter

  * `void *_MIR_get_interp_shim (MIR_context_t ctx, MIR_item_t
    func_item, void *handler)` generates and returns a function behaving as usual
    C function which calls function `void handler (MIR_context_t ctx,
    MIR_item_t func_item, va_list va, MIR_val_t *results)` where `va` can be used to access to
    arguments of the generated function call and `results` will contain
    what the generated function returns.  MIR design assumes that call of external
    C function and MIR function can not be recognized and also that we
    should easily switch from generated code from MIR to the MIR code
    interpretation. It means that we need to have the same interface
    for MIR function interpretation as usual C function.  The
    generated function provides such interface

  * `void *_MIR_get_ff_call (MIR_context_t ctx, size_t nres,
    MIR_type_t *res_types, size_t nargs, MIR_type_t *arg_types)`
    generates function `void fun1 (void *func_addr, MIR_val_t
    *res_arg_addresses)` and returns its address.  The generated
    function `fun1` calls `func_addr` as usual C function which is returning
    `nres` results given as the first elements of `res_arg_addresses`
    and is taking `nargs` arguments as the subsequent elements of
    `res_arg_addresses`.  Result and argument types are described
    correspondingly by vectors `res_types` and `arg_types`
    
## C2MIR machine dependent code

  After writing `mir-<target>.c` file and making MIR interpreter work
(use make `interp-test` to check this), you can make `c2m` working
with the interpreter.  You need to create a lot files but they are
mostly copies of already existing ones.

  * First create directory `c2mir/<target>` and files `c<target>.h`,
    `c<target>-code.c`, and `mirc-<target>-linux.h` in this directory.
    The simplest way to do this is to copy an existing directory
    `x86_64` or `aarch64`, rename files in the new directory, and
    modify them

    * file `c<target>.h` defines types and macros used by C2MIR compiler.
      In most cases of 64-bit target, you don't need to change
      anything

    * file `c<target>-code.c` defines platform depended constants
      (e.g. standard include directories) and functions (concerning
      data alignments) used by C2MIR compiler. You just need to rename
      definitions containing `<target>` in their names

    * file `mirc-target-linux.h` contains predefined macros of C2MIR
      compiler.  You should rename some of them.  To find what macros
      to rename, you can use `cpp -dM < /dev/null` on platform to
      which you port MIR

  * Second include files `c<target>.h` and `c<target>-code.c` into
    file `c2mir.c` and add target standard include and library
    directories in files `c2mir.c` and `c2mir-driver.c`

C programs compiled by C2MIR compiler need some compiler specific
files.  The easiest way to do this is to copy existing target
dependent directory (e.g. aarch64) in directory `mirc/include` and
modify files in the new directory.  Again in most cases of 64-bit
target, you don't need to change anything probably except macros
related long double and `va_list` definitions.

To run C tests for C2MIR with MIR intepreter you can use `make
c2mir-interp-test`.
  
## Machine-dependent code for MIR-generator (file `mir-gen-<target>.c`)

  The last step for porting MIR is to make MIR generator generates
code for your target. You need to provide the following
machine-dependent functions and definitions used by MIR-generator:
 
  * `void target_init (MIR_context_t ctx)` and `void target_finish (MIR_context_t ctx)`
    can be used to initialize/finalize data
    internally for this file which can be used during all MIR generator
    work in given context

  * Register Description
    * it is a good practice to describe all hard regs you would like
      to use in MIR-code during MIR-generator as C enumerator.
      The enumerator should contains hard regs used for register allocations, stack
      access, parameter passing etc.  All hard registers added in
      machinize should be here.  MIR generator does not use the enumerator costants but you
      can use them in `mir-gen-target.c` file for you convinence.  MIR generator only refers for
      the following target hard registers descriptions:

      * `MAX_HARD_REG` is maximal number of the described hard registers
      * `SP_HARD_REG` and `HARD_REG_FRAME_POINTER` are hard register numbers used
        as stack and frame pointer according target ABI
      * `TEMP_INT_HARD_REG1`, `TEMP_INT_HARD_REG2`, `TEMP_FLOAT_HARD_REG1`, `TEMP_FLOAT_HARD_REG2`,
        `TEMP_DOUBLE_HARD_REG1`, `TEMP_DOUBLE_HARD_REG2`, `TEMP_LDOUBLE_HARD_REG1`, and `TEMP_DOUBLE_HARD_REG2`
	are numbers of hard regs not used in machinized code and for register allocation.
	It is better use callee-cloberred hard regs.  These regs occurs in MIR code only after RA.
	The corresponding `_REG1` and `_REG2` with the same prefix should be distinctive but hard regs
	with different prefixes can be the same
    * `int target_hard_reg_type_ok_p (MIR_reg_t hard_reg, MIR_type_t type)` should return true
      only if HARD_REG can contain value of `type`
    * `int target_fixed_hard_reg_p (MIR_reg_t hard_reg)` should return true you want that `hard_reg` 
      is not used for register allocation.  For stack and frame pointers, the temporary hard registers
      (ones with prefix `TEMP_` above) the function should always return true
    * `int target_call_used_hard_reg_p (MIR_reg_t hard_reg)` returns true if `hard_reg` can be cloberred
      by callee according target ABI
    
  * `int target_locs_num (MIR_reg_t loc, MIR_type_t type)` returns number of standard 64-bit stack
    slots should be used for value of `type`.  Usually it is just `1` but it can be `2` for
    long double type

  * Array `const MIR_insn_code_t target_io_dup_op_insn_codes[]` contains codes of insns
    requiring to have output and one input operand to be
    the same on this target.  `MIR_INSN_BOUND` is used as the end
    marker

  * `MIR_disp_t target_get_stack_slot_offset (MIR_context_t ctx, MIR_type_t type, MIR_reg_t slot)`
    returns offset of the slot relative the stack frame
    pointer. MIR registers which did not get a hard register will keep their
    values in 64-bit stack slots.  The slot numbers start with 0.  The transformation of slot number
    to the offset depends on your own or ABI stack frame layout.  You should be not aware of stack
    slot alignment.  It is MIR generator responsibility

  * `void target_machinize (MIR_context_t ctx)` transforms MIR code
    for the target.  Usually it includes call transformations
    according to ABI by adding MIR insns which put call arguments
    into hard registers or on the stack and setting the called
    function result from the return register, setting up return
    register(s) for the current functions.  The function can also

    * change some MIR insns into another sequence of MIR insns which are better for the target.
      It may also include peephole optimization
    * change some MIR insns into built-in function calls if a MIR insn requires a lot of
      target insns to implement it
    * change MIR insn operands on some hard register if the corresponding target insn works
      only with this hard register.  In this case you need generate move insn to copy the original
      operand into the hard register

    Adding and deleting MIR insns should be done with MIR-generator
    functions `gen_add_insn_before`, `gen_add_insn_after`, and
    `gen_delete_insn`.

  * `void target_make_prolog_epilog (MIR_context_t ctx, bitmap_t
    used_hard_regs, size_t stack_slots_num)` adds MIR insns for prologue and epilogue of the
    generated function.  This usually includes
    saving/restoring used callee-saved hard registers, stack space
    allocation, setting up set frame pointer register

  * `int target_insn_ok_p (MIR_context_t ctx, MIR_insn_t insn)`. MIR
    generator works mostly on simplified code.  For any simplified (or
    machinized) insn the function should return TRUE.  In brief simplified insns are

    * all insns with registers
    * register moves with immediate
    * moves which are loads or stores with indirect reg addressing

    MIR generator tries to combine several data dependent insns into one.  Return
    also TRUE if you provide translation of the combined insn.  The
    more combined insns you can translate, the better generated code
    will be
    
  * `uint8_t *target_translate (MIR_context_t ctx, size_t *len)`
    generates and returns address of machine code for the current MIR
    function. The function should generate machine insn(s) for any MIR
    insn for which `target_insn_ok_p` return TRUE.  The function
    returns code length in bytes through parameter `len`.

  * `void target_rebase (MIR_context_t ctx, uint8_t *base)` is called
    by MIR generator after `target_translate` call.  Sometimes after
    `target_translate` call we want to modify the output machine code
    for start code address `base` known only at this point.  This function
    can be used for this.  Usually it works on data collected by
    during `target_translate` execution

It is better to start with small generator tests by using `make
gen-test`.  After the successful test code generation you can
continue with bigger test set generated by `c2m`.  Please use `make
c2mir-gen-test` to run this test set.  And finally you can run `make
c2mir-bootstrap-test` which uses MIR-generator on a pretty big test
(C-to-MIR compiler).