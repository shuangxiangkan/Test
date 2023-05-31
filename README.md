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
       - `CallStmt`,
       - `ConStmt`,
       - `ReturnStmt`,
       - `memset_like`: the function has similar side-effect to function "void *memset(void *str, int c, size_t n)",
       - `memcpy_like`: the function has similar side-effect to function "void *memcpy(void *dest, const void * src, size_t n)",
       - `Rb_tree_ops`: the function has similar side-effect to function "_ZSt29_Rb_tree_insert_and_rebalancebPSt18_Rb_tree_node_baseS0_RS_".

    - For operands of function operation,, there are the following options:
       - `Arg`: represents a parameter,
       - `Obj`: represents a object,
       - `Ret`: represents a return value,
       - `Dummy`: represents a dummy node. Dummy nodes are used in the specification language to represent temporary values or intermediate states during the analysis of side-effects. They act as placeholders for values that are not directly related to function arguments or return values.

 
## Examples

   Below, examples are provided for each operator to help better understand the specification language.
   
   - `AddrStmt`:
      ```json
      "malloc":   {
           "return":  "void *",
           "arguments":  "(size_t)",
           "type": "EFT_ALLOC",
           "overwrite_app_function": 1,
           "AddrStmt": {
            "src": "Obj",
            "dst": "Ret"
         }
       }
      ```
     - `return: void *`: Specifies the return type of the function, which is a pointer to void.
     - `arguments: (size_t)`: Specifies the type of the argument(s) expected by the function. In this case, the "malloc" function expects a single argument of type "size_t".
     - `type: EFT_ALLOC`: Indicates the type of the function. Here, it is specified as "EFT_ALLOC", which means that this external function allocates a new object and assigns it to one of its arguments.
     - `overwrite_app_function: 1`: Sets the switch to 1, indicating that the specification rules defined in ExtAPI.json should overwrite the user-defined function in the code.
     - `AddrStmt: {src: Obj, dst: Ret}`: Specifies the side-effect of the function. In this case, it indicates that after executing the "malloc" function, a new object will be assigned to the return value ("Ret") of the function.

   - `CopyStmt`:
      ```json
      "strstr":   {
           "return":  "char *",
           "arguments":  "(const char *, const char *)",
           "type": "EFT_L_A0",
           "overwrite_app_function": 0,
           "CopyStmt": {
            "src": "Arg0",
            "dst": "Ret"
         }
       }
      ```
        - `overwrite_app_function: 0`: Sets the switch to 0, indicating that the specification rules defined in ExtAPI.json should overwrite the user-defined function in the code.
        - `CopyStmt: {src: Arg0, dst: Ret}`: Specifies the value of first argument ("Arg0") will be copied to the return value ("Ret") of the function.

   - `LoadStmt` and `StoreStmt`:
      ```json
      "_ZNSsC1ERKSs":{
           "return":  "void",
           "arguments":  "(std::basic_string<char, std::char_traits<char>, std::allocator<char> > const&)",
           "type": "CPP_EFT_A0R_A1R",
           "overwrite_app_function": 0,
           "LoadStmt": {
               "src": "Arg1",
               "dst": "Dummy"
           },
           "StoreStmt": {
               "src": "Dummy",
               "dst": "Arg0"
           }
       }
      ```
        - `LoadStmt: {src: Arg1, dst: Dummy}`: It indicates that the value of second argument ("Arg1") will be loaded into a dummy node ("Dummy").
        - `CopyStmt: {src: Dummy, dst: Arg0}`: It indicates that the value in the dummy node ("Dummy")(the same dummy node) will be stored into first argument("Arg0"). The dummy node is used in the "LoadStmt" and "StoreStmt" side-effects. It allows for the temporary storage and manipulation of values during the analysis process.
        
   - `GetStmt`:
      ```json
      "_ZNSt5arrayIPK1ALm2EE4backEv":{
        "return":  "%class.A** ",
        "arguments":  "(%struct.std::array *)",
        "type": "CPP_EFT_DYNAMIC_CAST",
        "overwrite_app_function": 1,
        "GepStmt1": {
            "src": "Arg0",
            "dst": "Dummy",
            "offset": "0"
        },
        "GepStmt2": {
            "src": "Dummy",
            "dst": "Ret",
            "offset": "0"
        }
      ```
        - `GepStmt1: {src: Arg0, dst: Dummy, offset: 0}`: Specifies a getelementptr (GEP) statement side-effect. It indicates that a GEP operation should be performed on the value of the first argument ("Arg0"), with an offset of 0, and the result should be stored in the dummy node ("Dummy").
        - `GepStmt2: {src: Dummy, dst: Ret, offset: 0}`: Specifies another GEP statement side-effect. It indicates that a GEP operation should be performed on the value in the dummy node ("Dummy"), with an offset of 0, and the result should be assigned to the return value ("Ret").

   - `CallStmt` and `ReturnStmt`:
      ```json
      "swapExtCallStmt":{
        "return":  "void",
        "arguments":  "(char **, char **)",
        "type": "EFT_NOOP",
        "overwrite_app_function": 0,
        "CallStmt": {
            "callee_name": "swap",
            "callee_return": "void",
            "callee_arguments": "(char **, char **)",
            "CopyStmt1": {
                "src": "swapExtCallStmt_Arg0",
                "dst": "x"
            },
            "LoadStmt1": {
                "src": "x",
                "dst": "swap_Arg0"
            },
            "CopyStmt2": {
                "src": "swapExtCallStmt_Arg1",
                "dst": "swap_Arg1"
            },
             "ReturnStmt": {
                "src": "swap_Ret",
                "dst": "swapExtCallStmt_Ret"
            }
        }
      }
      ```
        - `CopyStmt1: {src: swapExtCallStmt_Arg0, dst: x}`: Specifies a copy statement side-effect. It indicates that the value of the swapExtCallStmt_Arg0 node (which represents the first argument of the swapExtCallStmt function) should be copied into a node named "x".
        - `LoadStmt1: {src: x, dst: swap_Arg0}`: Specifies a load statement side-effect. It indicates that the value in the "x" node should be loaded into the first argument of the swap function.
        - `CopyStmt2: {src: swapExtCallStmt_Arg1, dst: swap_Arg1}`: Specifies another copy statement side-effect. It indicates that the value of the swapExtCallStmt_Arg1 node (which represents the second argument of the swapExtCallStmt function) should be copied into the second argument of the swap function.
        - `ReturnStmt: {src: swap_Ret, dst: swapExtCallStmt_Ret}`: Specifies a return statement side-effect. It indicates that the value in the "swap_Ret" node (representing the return value of the "swap" function) should be assigned to the return value of the "swapExtCallStmt" function.
        
        In summary, this external function `swapExtCallStmt` performs a call to another function named "swap" with the specified arguments and return type. It includes copy and load statements to handle the argument passing between functions. Additionally, it assigns the return value of the "swap" function to the return value of the "swapExtCallStmt" function.
        
        
        - `CallStmt` and `ReturnStmt`:
      ```json
      "swapExtCallStmt":{
        "return":  "void",
        "arguments":  "(char **, char **)",
        "type": "EFT_NOOP",
        "overwrite_app_function": 0,
        "CallStmt": {
            "callee_name": "swap",
            "callee_return": "void",
            "callee_arguments": "(char **, char **)",
            "CopyStmt1": {
                "src": "swapExtCallStmt_Arg0",
                "dst": "x"
            },
            "LoadStmt1": {
                "src": "x",
                "dst": "swap_Arg0"
            },
            "CopyStmt2": {
                "src": "swapExtCallStmt_Arg1",
                "dst": "swap_Arg1"
            },
             "ReturnStmt": {
                "src": "swap_Ret",
                "dst": "swapExtCallStmt_Ret"
            }
        }
      }
      ```
        - `CopyStmt1: {src: swapExtCallStmt_Arg0, dst: x}`: Specifies a copy statement side-effect. It indicates that the value of the swapExtCallStmt_Arg0 node (which represents the first argument of the swapExtCallStmt function) should be copied into a node named "x".
        - `LoadStmt1: {src: x, dst: swap_Arg0}`: Specifies a load statement side-effect. It indicates that the value in the "x" node should be loaded into the first argument of the swap function.
        - `CopyStmt2: {src: swapExtCallStmt_Arg1, dst: swap_Arg1}`: Specifies another copy statement side-effect. It indicates that the value of the swapExtCallStmt_Arg1 node (which represents the second argument of the swapExtCallStmt function) should be copied into the second argument of the swap function.
        - `ReturnStmt: {src: swap_Ret, dst: swapExtCallStmt_Ret}`: Specifies a return statement side-effect. It indicates that the value in the "swap_Ret" node (representing the return value of the "swap" function) should be assigned to the return value of the "swapExtCallStmt" function.
        
        In summary, this external function `swapExtCallStmt` performs a call to another function named "swap" with the specified arguments and return type. It includes copy and load statements to handle the argument passing between functions. Additionally, it assigns the return value of the "swap" function to the return value of the "swapExtCallStmt" function.
      
