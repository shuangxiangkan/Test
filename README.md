# Specifications in ExtAPI.json

## Overview
The specification language of external functions is based on the JSON format. Every external function defined by the Specification Language is an object that represents the specification rules, which reflect the side-effect of this external function. 

## Structure
These Specification Language objects for functions contain four parts:

1. The signature of the function (`Mandatory`)
   - `return`: return value type. (Only care about whether the return value is a pointer during analysis)
   - `arguments`: argument types. (Only care about the number of arguments during analysis)

2. The type of the function (`Mandatory`)
   - `type`: represents the properties of the function. (For example, "EFT_ALLOC" represents if this external function allocates a new object and assigns it to one of its arguments. For the selection of function type and a more detailed explanation, please refer to the definition of enum *extType* in ExtAPI.h.)


3. The switch that controls whether the specification rules defined in the ExtAPI.json overwrite the functions defined in the user code (`Mandatory`)
   - `overwrite_app_function`: The option controls whether the specification rules defined in the ExtAPI.json overwrite the functions defined in the user code (e.g., CPP files). When the `overwrite_app_function` is set to a value of 1, SVF will use the specification rules in ExtAPI.json to conduct the analysis and ignore the user-defined functions in the input CPP/bc files.
       - `overwrite_app_function = 0`: Analyze the user-defined functions.
       - `overwrite_app_function = 1`: Use specifications in ExtAPI.json to overwrite the user-defined functions.

   For example, the following is the code to be analyzed, which has a `foo()` function:

   ```c
   char* foo(char* arg)
   {
       return arg;
   }

   int main()
   {
       char* ret = foo("abc");
       return 0;
   }
   ```
   function foo() has a definition, but you want SVF not to use this definition when analyzing, and you think foo () should do nothing, Then you can add a new entry in ExtAPI.json, and set `overwrite_app_function: 1`, like the following:
   ```json
   "foo": {
       "return":  "char*",
       "arguments":  "(char*)",
       "type": "EFT_NOOP",
       "overwrite_app_function": 1
   }
   ```
 4. The side-effect of the external function (`Optional`, if there is no side-effect of that function). Function side-effect indicate the relationships between input and output, mainly between function parameters or between parameters and return values after the execution.
   
    -  For operators of function operation, there are the following options:
       - `AddrStmt`,
       - `CopyStmt`,
       - `LoadStmt`
       - `StoreStmt`,
       - `GepStmt`,
       - `Call`,
       - `Return`,
       - `memset_like`: the function has similar side-effect to function "void *memset(void *str, int c, size_t n)",
       - `memcpy_like`: the function has similar side-effect to function "void *memcpy(void *dest, const void * src, size_t n)",
       - `Rb_tree_ops`: the function has similar side-effect to function "_ZSt29_Rb_tree_insert_and_rebalancebPSt18_Rb_tree_node_baseS0_RS_".

    - For operands of function operation,, there are the following options:
       - `Arg`: represents a parameter,
       - `Obj`: represents a object,
       - `Ret`: represents a return value,
       - `Dummy`: represents a dummy node.
