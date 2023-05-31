# Specifications in ExtAPI.json

## Overview
The specification language of external functions is based on the JSON format. Every external function defined by the Specification Language is an object that represents the specification rules, which reflect the side-effect of this external function. 

## Structure
These Specification Language objects for functions contain four parts:

1. The signature of the function (Mandatory)
   - "return": return value type (Only care about whether the return value is a pointer during analysis)
   - "argument": argument types (Only care about the number of arguments during analysis)

2. The type of the function (Mandatory)
   - Function type represents the properties of the function.
   - For example, "EFT_ALLOC" represents if this external function allocates a new object and assigns it to one of its arguments.
   - For the selection of function type and a more detailed explanation, please refer to the definition of enum *extType* in ExtAPI.h.

3. The switch that controls whether the specification rules defined in the ExtAPI.json overwrite the functions defined in the user code (Mandatory)
   - The switch *overwrite_app_function* controls whether the specification rules defined in the ExtAPI.json overwrite the functions defined in the user code (e.g., CPP files).
   - When the switch *overwrite_app_function* is set to a value of 1, SVF will use the specification rules in ExtAPI.json to conduct the analysis and ignore the user-defined functions in the input CPP/bc files.
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
