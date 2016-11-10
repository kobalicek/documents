Introduction
------------

The following guide explains how to contribute to the following projects:

  * [asmjit](https://github.com/asmjit/asmjit)
  * [asmtk](https://github.com/asmjit/asmtk)
  * [b2d](https://github.com/blend2d/b2d)
  * [mpsl](https://github.com/kobalicek/mpsl)

Since this document is shared between more projects it uses `product` instead of a real project name.

The content of this file is inteded for people that want to contribute to the projects above or people that want some explanations about the conventions used. While some conventions are purely aesthetic to make the code more readable and consistent, others have more pragmatic reasons and were designed to improve compatibility with more (and older) C++ compilers and also other projects. This means that the coding style and conventions don't have to match all C++ developers' taste, but that's life.

Project Structure
-----------------

  * / (root)
    * /src/product - Project source and header files (normally /src should be in include path when including the project from other locations).
    * /test - Tests and sample applications.
    * /tools - Scripts to create project files and miscellaneous scripts for project management (code generation, etc...).

Source File Naming
------------------

  * Source and header file names are all lowercase and use **.cpp** and **.h** extensions.
  * Internal header files are suffixed with **_p.h** and should not be installed.
  * Source files that require certain command line flags to enable cpu-specific instruction sets must use the following suffixes:
    * X86:
      * `*_sse.cpp`    - SSE
      * `*_sse2.cpp`   - SSE2
      * `*_sse3.cpp`   - SSE3
      * `*_ssse3.cpp`  - SSSE3
      * `*_sse4_1.cpp` - SSE4.1
      * `*_sse4_2.cpp` - SSE4.2
      * `*_avx.cpp`    - AVX
      * `*_avx2.cpp`   - AVX2
    * ARM:
      * `*_neon.cpp`   - NEON
  * Common header files:
    * `/src/product/product.h` - For product users, should always be included as `#include <product/product>`.
    * `/src/product/product_apibegin.h` - Included before API is defined (used internally).
    * `/src/product/product_apiend.h` - Included after API is defined (used internally).
    * `/src/product/product_build.h` - Included by all files, contains compiler and target detection (used internally).

Source File Structure
---------------------

Summary:

  * Source and header files contain a header, which specifies product name, brief description, and license information (license and a link to LICENSE.md file).
  * Source and header files that include other header files of the same product use relative paths starting with `./` or `../`, the relative path should first point to the project root and then to the desired file, like `../path/somefile.h`.
  * Source and header files always include `product_apibegin.h` and `product_apiend.h` helpers that may define some macros to make the projec more portable.
  * Header files contain a guard that is a composition of product name, path, and file name - `_PRODUCT_PATH_FILE_H`.
  * Source files always define `PRODUCT_EXPORTS` macro so the project can properly configure API imports and exports.

Explanation:
  * Relative includes guarantee that even if there is the same library already installed and available through global include paths it won't be included / used.
  * Defining `PRODUCT_EXPORTS` in each source file guarantees that the user doesn't need to do anything specific to compile and use the library.
  * Providing `product_build.h` that detects the compiler and target allows to auto-generate most parts of that file by a tool called `cxxtool`.
  * Providing `product_apibegin.h` and `product_apiend.h` allows to use some C++11 features by default when they are available, but make the project compilable by older compilers that don't support these. It also allows to temporarily disable some warnings that make unnecessary noise while compilation. The intention is to only apply these locally and to not break other code (that's why there is always api-end that cleans everything up).

Example of `product/somepath/somefile.h`:

```c++
// [ProductName]
// Brief Description.
//
// [License]
// LicenseName - See LICENSE.md file in the package.

// [Guard]
#ifndef _PRODUCT_SOMEPATH_SOMEFILE_H
#define _PRODUCT_SOMEPATH_SOMEFILE_H

// [Dependencies]
#include "../somepath/file.h"
#include "../somepath/otherfile.h"

// [Api-Begin]
#include "../product_apibegin.h"

namespace product {

//! \addtogroup product_somepath
//! \{

// ============================================================================
// [product::SomeClass]
// ============================================================================

class SomeClass {
public:
  PRODUCT_API SomeClass();
  PRODUCT_API ~SomeClass();
};

//! \}

} // product namespace

// [Api-End]
#include "../product_apiend.h"

// [Guard]
#endif // _PRODUCT_SOMEPATH_SOMEFILE_H
```

Example of `product/path/somefile.cpp`:

```c++
// [ProductName]
// Brief Description.
//
// [License]
// LicenseName - See LICENSE.md file in the package.

// [Export]
#define PRODUCT_EXPORTS

// [Dependencies]
#include "../somepath/somefile.h"

// [Api-Begin]
#include "../product_apibegin.h"

namespace product {

// ============================================================================
// [product::SomeClass]
// ============================================================================

SomeClass::SomeClass() {
  ...
}

SomeClass::~SomeClass() {
  ...
}

} // product namespace

// [Api-End]
#include "../product_apiend.h"
```

Coding Style
------------

  * **2 spaces** indentation, never use tabs.
  * Names of classes and structs are **CamelCased** like **SomeClass**.
  * Variables, functions, and member functions are **camelCased** like **someFunction()**.
  * Use **class** for anything that has a constructor and/or destructor, **struct** otherwise.

  * Namespaces are not indented:
    ```c++
    namespace product {

    CODE;

    } // product namespace
    ```

  * Access specifiers like `public`, `protected`, and `private` are not generally used, use `_` prefix to mark something as internal. Access modifiers have the same indentation as the **class**:
    ```c++
    class SomeClass {
    public:
      ...
    };
    ```

  * No spaces within a condition or function declaration and call:
    ```c++
    // Right:
    if (x)
      someFunction(x);

    /* WRONG:
    if ( x )
      someFunction( x );
    */
    ```

  * If obvious, omit `== nullptr` and `!= nullptr` from conditions, except asserts:
    ```c++
    void* ptr;

    // Right:
    ASMJIT_ASSERT(ptr != nullptr);
    if (ptr) {
      ...
    }

    /* WRONG:
    ASMJIT_ASSERT(ptr);
    if (ptr != nullptr) {
      ...
    }

    // WRONG: Don't use NULL, use nullptr:
    if (ptr != NULL) {
      ...
    }
    */
    ```

  * No line between a statement and opening bracket `{`:
    ```c++
    class SomeClass {
      void someFunction() {
        for (uint32_t i = 0; i < 10; i++) {
          if (i & 0x1) {
            CODE;
          }
        }
      }
    };
    ```

  * If some branch of a condition (if/else) requires a block `{}` then all branches of that condition must be surrounded by a block:
    ```c++
    // Right:
    if (x)
      first();
    else
      second();

    if (x) {
      first();
    }
    else {
      second();
    }

    /* WRONG:
    if (x) {
      first();
    }
    else
      second();
    */
    ```

  * There is a space between a **switch** statement and the condition; **case** statement and branch are indented:
    ```c++
    switch (condition) {
      case 0:
        doSomething();
        break;

      // Case that requires a block.
      case 1: {
        doSomethingElse();
        break;
      }

      // There should be a default branch, even if it's just an ASSERT:
      default:
        ASMJIT_NOT_REACHED();
    }
    ```

    Occassionally **case** and the branch can be merged in a single line:

    ```c++
    switch (condition) {
      case 0: doSomething(); break;
      case 1: doSomethingElse(); break;

      // There should be a default branch, even if it's just an ASSERT:
      default:
        ASMJIT_NOT_REACHED();
    }
    ```

  * Enum names are **CamelCased** and enum values start with lowercase **k**:
    ```c++
    enum AccessType {
      kAccessRO = 0,
      kAccessRW = 1
    };
    ```

    if an enumeration is only used within a single class it should be put inside of it instead of polluting the library namespace. Global enumerations can be put into **product::Globals** namespace:
    ```c++
    namespace product {
    namespace Globals {

    enum AccessType {
      kAccessRO = 0,
      kAccessRW = 1
    };

    } // Globals namespace
    } // product namespace
    ```

  * Preprocessor macros should be only used when necessary. If declared in a public header file they should always be prefixed by the product and contain only uppercase characters:
    ```c++
    #define PRODUCT_MULTIPLY(X, Y) ((X) * (Y))
    ```

    Macros that are only used to generate some code and are thus only used temporarily should be **#undef**ed.

API Design
----------

  * Namespaces:
    * Namespace is in general not deep, but exceptions are allowed and it highly depends on the project complexity.
    * All classes and functions should be placed in the product namespace. Generally, don't pollute the global namespace unless absolutelly necessary.
  * Exception Safety and Error Handling:
    * Exceptions and RTTI are not used. Functions that never throw an exception should be marked `noexcept`, however, functions that call user code that may throw should allow exceptions to be thrown and must not be marked `noexcept`. This means that the product code should generally be safe, and may optionally allow to register an error handler that may throw (this technique is used by asmjit, for example). This allows 3rdparty code to be integrated seamlessly and it also allows to create bindings to other programming languages without surrounding everything in `try-catch` blocks.
    * Use return codes to report errors, and optionally provide error handlers to be registered (see `asmjit::ErrorHandler` and its use, for example).
  * Explicit initialization `init()` and `reset()`:
    * **SomeClass::init()** - Defered initialization. Some classes are initialized by **init()** instead of being initialized by its constructor(s). This design is mostly used by classes that use inheritance and that require to call virtual functions during initialization.
    * **SomeClass::reset(...)** - Resets all members of the class instance into a construction state. Reset generally contains all arguments as its constructor(s) and if the class instance has a reserved memory it should release it. Member functions of **reset(bool release)** signature specify if a possible dynamic memory allocated by the class instance should be released, if `false` the class resets to the construction state, but keeps some memory allocated for later reuse.
  * API exports:
    * API exporting is explicit (must be marked).
    * All functions and constants to be exported must be marked by `PRODUCT_API`. Classes without inheritance are never marked as a whole, instead only static and member functions to be exported are merked (inlines are not marked)
    * Classes that use inheritance must be marked as `PRODUCT_VIRTAPI`. This is required by some compilers to properly export the vtable and rtti.
