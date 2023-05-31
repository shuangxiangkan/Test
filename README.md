# Overview
The specification language of external functions is based on the JSON format. Every external function defined by the Specification Language is an object that represents the specification rules, which reflect the side-effect of this external function. 

# Structure
These Specification Language objects for functions contain four parts:

1. The signature of the function (`Mandatory`)
   - `return`: Return value type. (Only care about whether the return value is a pointer during analysis)
   - `arguments`: Argument types. (Only care about the number of arguments during analysis)

2. The type of the function (`Mandatory`)
   - `type`: Represents the properties of the function. (For example, "EFT_ALLOC" represents if this external function allocates a new object and assigns it to one of its arguments. For the selection of function type and a more detailed explanation, please refer to the definition of enum *extType* in [ExtAPI.h](https://github.com/SVF-tools/SVF/blob/master/svf/include/Util/ExtAPI.h).)


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
       - `AddrStmt`:  Add an Address edge between src node and dst node in SVF IR. 
       - `CopyStmt`:  Add a Copy edge between src node and dst node in SVF IR. 
       - `LoadStmt`:  Add a Load edge between src node and dst node in SVF IR. 
       - `StoreStmt`: Add a Store edge between src node and dst node in SVF IR. 
       - `GepStmt`: Add a Gep edge between src node and dst node with an offset node in SVF IR. 
       - `CallStmt`: In an external call, there may be another callee function being called. Call edges will be added from the actual arguments of the external call to the formal parameters of the callee function in SVF IR.
       - `ReturnStmt`: In an external call, there may be another callee function being called. A return edge will be added from the return value of the callee function to the external call in SVF IR.
       - `CondStmt`: For the true and false branches, corresponding edges will be added accordingly in SVF IR.
       - `memset_like`: The function has similar side-effect to function `void *memset(void *str, int c, size_t n)`. Based on the value of size (e.g., `size_t n`), `n` Store edges will be added from the same src node to different subfields of dst node in SVF IR.
       - `memcpy_like`: The function has similar side-effect to function `void *memcpy(void *dest, const void * src, size_t n)`. Based on the value of size (e.g., `size_t n`), `n` Load and Store edges will be added between `n` pairs of src and dst subfields. A dummy node will be inserted as an intermediate node for each Load and Store edge in SVF IR.

    - For operands of function operation,, there are the following options:
       - `Arg`: Default argument in external function,
       - `Obj`: Represents an object,
       - `Ret`: Default return value in external function,
       - `Dummy`: Represents a dummy node. Dummy nodes are used in the specification language to represent temporary values or intermediate states during the analysis of side-effects. They act as placeholders for values that are not directly related to function arguments or return values.
       - `funName_Arg`: Represents an argument of `funName` function, used in CallStmt to differentiate between different functions
       - `funName_Ret`: Represents a return value of `funName` function, used in CallStmt to differentiate between different functions
       - `user-defined variables`: Such as `x` and `y`, when encountering a user-defined variable, SVFIR creates an SVFInstruction for this variable and performs subsequent operations.

 
# Examples

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
        - `overwrite_app_function: 0`: Sets the switch to 0, indicating that the specification rules defined in ExtAPI.json should not overwrite the user-defined function in the code.
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
        - `CopyStmt: {src: Dummy, dst: Arg0}`: It indicates that the value in the dummy node ("Dummy", the same dummy node) will be stored into first argument("Arg0"). The dummy node is used in the "LoadStmt" and "StoreStmt" side-effects. It allows for the temporary storage and manipulation of values during the analysis process.
        
   - `GepStmt`:
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
        - `CopyStmt1: {src: swapExtCallStmt_Arg0, dst: x}`: Specifies a copy statement side-effect. It indicates that the value of first argument of the swapExtCallStmt function should be copied into a node named "x".
        - `LoadStmt1: {src: x, dst: swap_Arg0}`: Specifies a load statement side-effect. It indicates that the value in the "x" node should be loaded into the first argument of the swap function.
        - `CopyStmt2: {src: swapExtCallStmt_Arg1, dst: swap_Arg1}`: Specifies another copy statement side-effect. It indicates that the value of the second argument of the swapExtCallStmt function should be copied into the second argument of the swap function.
        - `ReturnStmt: {src: swap_Ret, dst: swapExtCallStmt_Ret}`: Specifies a return statement side-effect. It indicates that the value in the return value of the swap function should be assigned to the return value of the swapExtCallStmt function.
        
        In summary, this external function swapExtCallStmt performs a call to another swap function with the specified arguments and return type. It includes copy and load statements to handle the argument passing between functions. Additionally, it assigns the return value of the swap function to the return value of the swapExtCallStmt function.
        
        
   - `CondStmt`:
      ```json
          "foo": {
           "return":  "void",
           "arguments":  "(char **, char **)",
           "type": "EFT_FREE_MULTILEVEL",
           "overwrite_app_function": 1,
           "CondStmt": {
               "TrueBranch": {
                   "Condition": "arg0 == arg1",
                   "CopyStmt": {
                       "src": "Arg0",
                       "dst": "Arg1"
                   }
               },
               "FalseBranch": {
                   "Condition": "arg0 != arg1",
                   "CopyStmt": {
                       "src": "Arg1",
                       "dst": "Ret"
                   }      
              }
          }
      ```
        - `TrueBranch: {Condition: arg0 == arg1}`: Represents the true condition branch.
        - `TrueBranch: {CopyStmt: {src: Arg0, dst: Arg1}"}`: Specifies a copy statement side-effect. It indicates that if the condition is true, the first argument should be copied into the second argument
        - `FalseBranch: {Condition: arg0 != arg1}`: Represents the false condition branch.
        - `FalseBranch: {CopyStmt: {src: Arg1, dst: Ret}}`: Specifies a copy statement side-effect. It indicates that if the condition is false, the second argument should be copied into the return value.

   - `memset_like`:
      ```json
      "llvm.memset":  {
        "return":  "void",
        "arguments":  "(i8*, i8, i32, i32, i1)",
        "type": "EFT_L_A0__A0R_A1",
        "overwrite_app_function": 0,
        "memset_like":  {
            "src": "Arg0",
            "dst": "Arg1",
            "size": "Arg2"
        }
      }
      ```
        - `memset_like`: Specifies a side-effect that is similar to the behavior of the `memset()` function.
        - `src: Arg0`: Specifies the source node from which the value will be copied. In this case, the source node is the first argument of the function (Arg0).
        - `dst: Arg1`: Specifies the destination node to which the value will be copied. In this case, the destination node is the subfields of second argument of the function (Arg1).
        - `size: Arg2`: Specifies the size node that represents the size of subfields of destination node to be handled. In this case, the size node is the third argument of the function (Arg2).

   - `memcpy_like`:
      ```json
      "bcopy":    {
        "return":  "void",
        "arguments":  "(const void *, void *, size_t)",
        "type": "EFT_A1R_A0R",
        "overwrite_app_function": 0,
        "memcpy_like":  {
            "src": "Arg1",
            "dst": "Arg0",
            "size": "Arg2"
        }
      }
      ```
        - `memcpy_like`: Specifies a side-effect that is similar to the behavior of the `memcpy` function.
        - `src: Arg1`: Specifies the source node from which the value will be copied. In this case, the source node is the subfields of second argument of the function (Arg1).
        - `dst: Arg0`: Specifies the destination node to which the value will be copied. In this case, the destination node is the the subfields of first argument of the function (Arg0).
        - `size: Arg2`: Specifies the size node that represents the size of subfields between source and destination to be handled. In this case, the size node is the third argument of the function (Arg2).
      
