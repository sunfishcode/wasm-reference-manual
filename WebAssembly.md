WebAssembly Reference Manual
================================================================================

0. [Introduction](#introduction)
0. [Basics](#basics)
0. [Module](#module)
0. [Instruction Descriptions](#instruction-descriptions)
0. [Instructions](#instructions)
0. [Instantiation](#instantiation)
0. [Execution](#execution)
0. [Text Format](#text-format)

![WebAssembly Logo][logo]

[logo]: https://github.com/WebAssembly/web-assembly-logo/raw/master/dist/logo/web-assembly-logo.png "The WebAssembly Logo"


Introduction
--------------------------------------------------------------------------------

WebAssembly, or "wasm", is a general-purpose virtual [ISA] designed to be a
compilation target for a wide variety of programming languages. Much of its
distinct personality derives from its security, code compression, and decoding
optimization features.

The unit of WebAssembly code is the [*module*](#module). Modules consist of a
header followed by a sequence of sections. There are sections describing a
WebAssembly's interactions with other modules ([imports] and [exports]),
sections declaring [data](#data-section) and other implements used by the
module, and sections defining [*functions*].

WebAssembly modules are encoded in binary form for size and decoding efficiency.
They may be losslessly translated to [text form] for readability.

WebAssembly code must be validated before it can be instantiated and executed.
WebAssembly is designed to allow decoding and validation to be performed in a
single linear pass through a WebAssembly module, and to enable many parts of
decoding and validation to be performed concurrently. For example,
[loops are explicitly identified](#loop), and decoders can be sure that program
state is consistent at all control flow merge points within a function without
having to see the entire function body first.

A WebAssembly module can be [*instantiated*] to produce a WebAssembly instance,
which contains all the data structures required by the module's code for
execution. Instances can include [linear memories], which can serve the purpose
of an address space for program data. For security and determinism, linear
memory is *sandboxed*, and the other data structures in an instance, including
the call stack, are allocated outside of linear memory so that they cannot be
corrupted by errant linear-memory accesses. Instances can also include [tables],
which can serve the purpose of an address space for indirect function calls,
among other things. An instance can then be executed, either by execution of its
[start function](#start-section) or by calls to its exported functions, and its
exported linear memories and global variables can be accessed.

Along with the other contents, each function contains a sequence of
[*instructions*], which are [described](#instruction-descriptions) here with some
simple conventions. There are instructions for performing integer and
floating-point arithmetic, directing control flow, loading and storing to linear
memory (as a [load-store architecture]), calling functions, and more. During
[*execution*], instructions conceptually communicate with each other primarily
via pushing and popping values on a virtual stack, which allows them to have a
very compact encoding.

Implementations of WebAssembly need not perform all the steps literally as
described here; they need only behave ["as if"] they did so in all observable
respects.

> Except where specified otherwise, WebAssembly instructions are not required to
execute in [constant time].

[ISA]: https://en.wikipedia.org/wiki/Instruction_set
[imports]: #import-section
[exports]: #export-section
[*execution*]: #execution
[*functions*]: https://en.wikipedia.org/wiki/Subroutine
[*instructions*]: #instructions
[*instantiated*]: #instantiation
[load-store architecture]: https://en.wikipedia.org/wiki/Load/store_architecture
["as if"]: https://en.wikipedia.org/wiki/As-if_rule
[constant time]: https://www.bearssl.org/constanttime.html


Basics
--------------------------------------------------------------------------------

0. [Bytes](#bytes)
0. [Pages](#pages)
0. [Nondeterminism](#nondeterminism)
0. [Linear Memories](#linear-memories)
0. [Tables](#tables)
0. [Encoding Types](#encoding-types)
0. [Language Types](#language-types)
0. [Embedding Environment](#embedding-environment)

### Bytes

[*Bytes*] in WebAssembly are 8-[bit], and are the addressing unit of
[linear-memory] accesses.

[*Bytes*]: https://en.wikipedia.org/wiki/Byte

### Pages

[*Pages*] in WebAssembly are 64 [KiB], and are the units used in [linear-memory]
size declarations and size operations.

[*Pages*]: https://en.wikipedia.org/wiki/Page_(computer_memory)

### Nondeterminism

When semantics are specified as *nondeterministic*, a WebAssembly implementation
may perform any one of the discrete set of specified alternatives.

There is no requirement that a given implementation make the same choice every
time, even for successive executions of the same instruction within the same
instance of a module.

There is no ["undefined behavior"] in WebAssembly where the semantics become
completely unspecified. Thus, WebAssembly has only
*Limited Local Nondeterminism*.

> All instances of nondeterminism in WebAssembly are explicitly described as
such with a link to here.

["undefined behavior"]: https://en.wikipedia.org/wiki/Undefined_behavior

### Linear Memories

A *linear memory* is a contiguous, [byte]-addressable, readable and writable
range of memory spanning from offset `0` and extending up to a *linear-memory
size*, allocated as part of a WebAssembly instance. The size of a linear memory
is always a multiple of the [page] size and may be increased dynamically (with
the [`grow_memory`](#grow-memory) instruction) up to an optional declared
*maximum length*. Linear memories are sandboxed, so they don't overlap with each
other or with other parts of a WebAssembly instance, including the call stack,
globals, and tables, and their bounds are enforced.

Linear memories can either be [defined by a module](#linear-memory-section)
or [imported](#import-section).

### Tables

A *table* is similar to a [linear memory] whose elements, instead of being
bytes, are opaque values. Each table has a [table element type] specifying what
kind of data they hold. A table of `anyfunc` is used as the index space for
[indirect calls](#indirect-call).

Tables can be [defined by a module](#table-section) or
[imported](#import-section).

> In the future, tables are expected to be generalized to hold a wide variety of
opaque values and serve a wide variety of purposes.

### Encoding Types

0. [Primitive Encoding Types](#primitive-encoding-types)
0. [varuPTR Immediate Type](#varuptr-immediate-type)
0. [memflags Immediate Type](#memflags-immediate-type)
0. [Array](#array)
0. [Byte Array](#byte-array)
0. [Identifier](#identifier)
0. [Type Encoding Type](#language-types)

#### Primitive Encoding Types

*Primitive encoding types* are the basic types used to represent fields within a
Module.

| Name               | Size (in bytes)  | Description                                  |
| ------------------ | ---------------- | -------------------------------------------- |
| `uint32`           | 4                | unsigned; value limited to 32 bits           |
|                    |                  |                                              |
| `varuint1`         | 1                | unsigned [LEB128]; value limited to 1 bit    |
| `varuint7`         | 1                | unsigned [LEB128]; value limited to 7 bits   |
| `varuint32`        | 1-5              | unsigned [LEB128]; value limited to 32 bits  |
| `varuint64`        | 1-10             | unsigned [LEB128]; value limited to 64 bits  |
|                    |                  |                                              |
| `varsint7`         | 1                | [signed LEB128]; value limited to 7 bits     |
| `varsint32`        | 1-5              | [signed LEB128]; value limited to 32 bits    |
| `varsint64`        | 1-10             | [signed LEB128]; value limited to 64 bits    |
|                    |                  |                                              |
| `float32`          | 4                | IEEE 754-2008 [binary32]                     |
| `float64`          | 8                | IEEE 754-2008 [binary64]                     |

[LEB128] encodings may contain padding `0x80` bytes, and [signed LEB128]
encodings may contain padding `0x80` and `0xff` bytes.

Except when specified otherwise, all values are encoded in
[little-endian byte order].

**Validation:**
 - For the types that have value limits, the encoded value is required to be
   within the limit.
 - The size of the encoded value is required to be within the type's size range.

> These types aren't used to describe values at execution time.

[LEB128]: https://en.wikipedia.org/wiki/LEB128
[signed LEB128]: https://en.wikipedia.org/wiki/LEB128#Signed_LEB128

#### varuPTR Immediate Type

A *varuPTR immediate* is either [varuint32] or [varuint64] depending on whether
the linear memory associated with the instruction using it is 32-bit or 64-bit.

#### memflags Immediate Type

A *memflags immediate* is a [varuint32] containing the following bit fields:

| Name     | Description                                            |
| -------- | ------------------------------------------------------ |
| `$align` | alignment in bytes, encoded as the [log2] of the value |

> As implied by the log2 encoding, `$align` can only be a [power of 2].

> `$flags` may hold additional fields in the future.

[power of 2]: https://en.wikipedia.org/wiki/Power_of_two
[log2]: https://en.wikipedia.org/wiki/Binary_logarithm

#### Array

An *array* of a given type is a [varuint32] indicating a number of elements,
followed by a sequence of that many elements of that type.

> Array elements needn't all be the same size in some representations.

#### Byte Array

A *byte array* is an [array] of [bytes].

> Byte arrays may contain arbitrary bytes and aren't required to be
[valid UTF-8] or any other format.

#### Identifier

An *identifier* is a [byte array] which is valid [UTF-8].

**Validation:**
 - Decoding the bytes according to the [UTF-8 decode without BOM or fail]
   algorithm is required to succeed.

[UTF-8 decode without BOM or fail]: https://encoding.spec.whatwg.org/#utf-8-decode-without-bom-or-fail

> Identifiers may contain [NUL] characters, aren't required to be
[NUL-terminated], aren't required to be [normalized][unicode normalization], and
aren't required to be marked with a [BOM] (though they aren't prohibited from
containing a BOM).

[NUL]: https://en.wikipedia.org/wiki/Null_character
[NUL-terminated]: https://en.wikipedia.org/wiki/Null-terminated_string
[BOM]: https://en.wikipedia.org/wiki/Byte_order_mark
[unicode normalization]: http://www.unicode.org/reports/tr15/

> Normalization is not performed when considering whether two identifiers are
the same.

#### Type Encoding Type

A *type encoding* is a value indicating a particular [language type].

| Name      | Binary Encoding |
| --------- | --------------- |
| `i32`     | `-0x01`         |
| `i64`     | `-0x02`         |
| `f32`     | `-0x03`         |
| `f64`     | `-0x04`         |
| `anyfunc` | `-0x10`         |
| `func`    | `-0x20`         |
| `void`    | `-0x40`         |

Type encodings are encoded as their Binary Encoding value in a [varsint7].

**Validation:**
 - A type encoding is required to be one of the values defined here.

### Language Types

*Language types* describe runtime values and language constructs. Each language
type has a [type encoding](#type-encoding-type).

0. [Value Types](#value-types)
0. [Table Element Types](#table-element-types)
0. [Signature Types](#signature-types)
0. [Block Signature Types](#block-signature-types)

#### Value Types

*Value types* are the types of individual input and output values of
instructions at execution time.

**Validation:**
 - A value type is required to be one of the integer or floating-point value
   types.

0. [Integer Value Types](#integer-value-types)
0. [Booleans](#booleans)
0. [Floating-Point Value Types](#floating-point-value-types)

##### Integer Value Types

*Integer value types* describe fixed-width integer values.

| Name  | Bits | Description                                                      |
| ----- | ---- | ---------------------------------------------------------------- |
| `i32` | 32   | [32-bit integer](https://en.wikipedia.org/wiki/32-bit)           |
| `i64` | 64   | [64-bit integer](https://en.wikipedia.org/wiki/64-bit_computing) |

Integer value types in WebAssembly aren't inherently signed or unsigned. They
may be interpreted as signed or unsigned by individual operations. When
interpreted as signed, a [two's complement] interpretation is used.

**Validation:**
 - An integer value type is required to be one of the values defined here.

> The [minimum signed integer value] is supported; consequently, two's
complement signed integers aren't symmetric around zero.

> When used as linear-memory indices or function table indices, integer types
may play the role of "pointers".

> Integer value types are sometimes described as fixed-point types with no
fractional digits in other languages.

##### Booleans

[Boolean][actual boolean] values in WebAssembly are represented as values of
type `i32`. In a boolean context, such as a `br_if` condition, any non-zero
value is interpreted as true and `0` is interpreted as false.

Any operation that produces a boolean value, such as a comparison, produces the
values `0` and `1` for false and true.

[actual boolean]: https://en.wikipedia.org/wiki/Boolean_data_type

> WebAssembly often uses alternate encodings for integers and boolean values,
rather than using the literal encodings described here.

##### Floating-Point Value Types

*Floating-point value types* describe IEEE 754-2008 floating-point values.

| Name  | Bits | Description                                                    |
| ----- | ---- | -------------------------------------------------------------- |
| `f32` | 32   | IEEE 754-2008 [binary32], commonly known as "single precision" |
| `f64` | 64   | IEEE 754-2008 [binary64], commonly known as "double precision" |

**Validation:**
 - A floating-point value type is required to be one of the values defined here.

> Unlike with [Numbers in ECMAScript], [NaN] values in WebAssembly have sign
bits and significand fields which may be observed and manipulated (though they
are usually unimportant).

[Numbers in ECMAScript]: https://tc39.github.io/ecma262/#sec-ecmascript-language-types-number-type
[NaN]: https://en.wikipedia.org/wiki/NaN

#### Table Element Types

*Table element types* are the types that may be used in a [table].

| Name       | Description                                  |
| ---------- | -------------------------------------------- |
| `anyfunc`  | a reference to a function with any signature |

**Validation:**
 - Table element types are required to be one of the values defined here.

#### Signature Types

*Signature types* are the types that may be defined in the [Type Section].

| Name       | Description                                  |
| ---------- | -------------------------------------------- |
| `func`     | a function signature                         |

**Validation:**
 - Signature types are required to be one of the values defined here.

#### Block Signature Types

*Block signature types* are the types that may be used as a `block` or other
control-flow construct signature.

Block signature types include the [value types], which indicate single-element
type sequences containing the type, and the following:

| Name       | Description                                  |
| ---------- | -------------------------------------------- |
| `void`     | an empty type sequence                       |

**Validation:**
 - Block signature types are required to be either a [value type] or one of the
   values defined here.

### Embedding Environment

A WebAssembly runtime environment will typically provide APIs for
interacting with the outside world, as well as mechanisms for loading and
linking wasm modules. This is the *embedding environment*.


Module
--------------------------------------------------------------------------------

0. [Module Contents](#module-contents)
0. [Known Sections](#known-sections)
0. [Custom Sections](#custom-sections)
0. [Module Index Spaces](#module-index-spaces)
0. [Module Types](#module-types)

### Module Contents

A module starts with header:

| Field Name     | Type     | Description                                                             |
| -------------- | -------- | ----------------------------------------------------------------------- |
| `magic_cookie` | [uint32] | magic cookie identifying the contents of a file as a WebAssembly module |
| `version`      | [uint32] | WebAssembly version number.                                             |

The header is then followed by a sequence of sections. Each section consists of
a [varuint7] *opcode* followed by a [byte array] *payload*. The opcode is
required to either indicate a *known section*, or be `0x00`, indicating a
[*custom section*](#custom-sections).

**Validation:**
 - `magic_cookie` is required to be `0x6d736100` (the string "\0asm").
 - `version` is required to be `0x1`.
 - For each present [known section](#known-sections), the requirements of its
   **Validation** clause are required, if one is specified for the section kind.
 - The requirements of the **Validation** clauses of every
   [index space](#module-index-spaces) are required, if one is specified for the
   index space.
 - Known sections are required to appear at most once, and those present are
   required to be ordered according to the order in the
   [enumeration of the Known Sections](#known-sections).
 - Custom sections are required to start their payload with an [identifier]
   *name*.
 - The encoding for the module is required to consist exclusively of the header
   and the sections.
 - The requirements of every component [encoding type] of the module are
   required.

> The `magic_cookie` bytes begin with a [NUL] character, indicating to generic
tools that the ensuing contents are not generally "text", followed by the
[UTF-8] encoding of the string "asm".

> The version is expected to change infrequently if ever; forward-compatible
extension is intended to be achieved by adding sections, types, instructions and
others without bumping the version.

> The section byte array length field is usually redundant, as most section
encodings end up specifying their size through other means as well, however it
is still useful to allow streaming decoders to quickly skip over whole sections.

### Known Sections

There are several *known sections*:

1. [Type Section]
0. [Import Section]
0. [Function Section]
0. [Table Section]
0. [Linear-Memory Section]
0. [Global Section]
0. [Export Section]
0. [Start Section]
0. [Element Section]
0. [Code Section]
0. [Data Section]

#### Type Section

**Opcode:** `0x01`.

The Type Section consists of an [array] of function signatures.

Each *function signature* consists of:

| Field Name      | Type               | Description                             |
| --------------- | ------------------ | --------------------------------------- |
| `form`          | [signature type]   | the type of signature                   |

If `form` is `func`, the following fields are appended.

| Field Name      | Type                    | Description                             |
| --------------- | ----------------------- | --------------------------------------- |
| `params`        | [array] of [value type] | the parameters to the function          |
| `returns`       | [array] of [value type] | the return types of the function        |

**Validation:**
 - `form` is required to be `func`.
 - Each `returns` array is required to contain at most one element.

> In the future, this section may contain other forms of type entries as well,
and support for function signatures with multiple return types.

#### Import Section

**Opcode:** `0x02`.

The Import Section consists of an [array] of imports.

An *import* consists of:

| Field Name      | Type                 | Description                              |
| --------------- | -------------------- | ---------------------------------------- |
| `module_name`   | [identifier]         | the name of the module to import from    |
| `export_name`   | [identifier]         | the name of the export in that module    |
| `kind`          | [external kind]      | the kind of import                       |

If `kind` is `Function`, the following fields are appended.

| Field Name      | Type                 | Description                              |
| --------------- | -------------------- | ---------------------------------------- |
| `sig_index`     | [varuint32]          | signature index into the [Type Section]  |

If `kind` is `Table`, the following fields are appended.

| Field Name      | Type                 | Description                              |
| --------------- | -------------------- | ---------------------------------------- |
| `desc`          | [table description]  | a description of the table               |

If `kind` is `Memory`, the following fields are appended.

| Field Name      | Type                        | Description                        |
| --------------- | --------------------------- | ---------------------------------- |
| `desc`          | [linear-memory description] | a description of the linear memory |

If `kind` is `Global`, the following fields are appended.

| Field Name      | Type                 | Description                              |
| --------------- | -------------------- | ---------------------------------------- |
| `desc`          | [global description] | a description of the global variable     |

The meaning of an import's `module_name` and `export_name` are determined by
the [embedding environment].

Imports provide access to constructs, defined and allocated by external entities
outside the scope of this reference manual (though they may be exports provided
by other WebAssembly modules), but which have behavior consistent with their
corresponding concepts defined in this reference manual. They can be accessed
through their respective [module index spaces](#module-index-spaces).

**Validation:**
 - All global imports are required to be immutable.
 - Each import is required to be resolved by the [embedding environment].
 - A linear-memory import's `minimum` length is required to be at most the
   imported linear memory's `minimum` length.
 - A linear-memory import is required to have a `maximum` length if the imported
   linear memory has a `maximum` length.
 - If present, a linear-memory import's `maximum` length is required to be at
   least the imported linear memory's `maximum` length.
 - A table import's `minimum` length is required to be at most the imported
   table's `minimum` length.
 - A table import is required to have a `maximum` length if the imported table
   has a `maximum` length.
 - If present, a table import's `maximum` length is required to be at least the
   imported table's `maximum` length.
 - Embedding-environment-specific validation requirements may be required of
   each import.

> Global imports may be permitted to be mutable in the future.

> `module_name` will often identify a module to import from, and `export_name`
an export in that module to import, but [embedding environments] may provide
other mechanisms for resolving imports as well.

#### Function Section

**Opcode:** `0x03`.

The Function Section consists of an [array] of function declarations. Its
elements directly correspond to elements in the [Code Section] array.

A *function declaration* consists of:
 - an index in the [Type Section] of the signature of the function.

**Validation:**
 - The array is required to be the same length as the [Code Section] array.

#### Table Section

**Opcode:** `0x04`.

The Table Section consists of an [array] of [table descriptions].

#### Linear-Memory Section

**Opcode:** `0x05`.

The Memory Section consists of an [array] of [linear-memory descriptions].

> Implementations are encouraged to attempt to reserve enough resources for
allocating up to the `maximum` length up front, if a `maximum` length is
present. Otherwise, implementations are encouraged to allocate only enough for
the `minimum` length up front.

#### Global Section

**Opcode:** `0x06`.

The Global Section consists of an [array] of global declarations.

A *global declaration* consists of:

| Field Name      | Type                             | Description                              |
| --------------- | -------------------------------- | ---------------------------------------- |
| `desc`          | [global description]             | a description of the global variable     |
| `init`          | [instantiation-time initializer] | the initial value of the global variable |

**Validation:**
 - The type of the value returned by `init` must be the same as `desc`'s
   `type`.

> Exporting of mutable global variables may be permitted in the future.

#### Export Section

**Opcode:** `0x07`.

The Export Section consists of an [array] of exports.

An *export* consists of:

| Field Name      | Type               | Description                             |
| --------------- | ------------------ | --------------------------------------- |
| `name`          | [identifier]       | field name                              |
| `kind`          | [external kind]    | the kind of export                      |
| `index`         | [varuint32]        | an index into an [index space]          |

If `kind` is `Function`, `index` identifies an element in the
[function index space].

If `kind` is `Table`, `index` identifies an element in the [table index space].

If `kind` is `Memory`, `index` identifies an element in the
[linear-memory index space].

If `kind` is `Global`, `index` identifies an element in the
[global index space].

The meaning of `name` is determined by the [embedding environment].

Exports provide access to an instance's constructs to external entities outside
the scope of this reference manual (though they may be other WebAssembly
modules), but which have behavior consistent with their corresponding concepts
defined in this reference manual.

**Validation:**
 - Each export's name is required to be unique among all the exports' names.
 - Each export's index is required to be within the bounds of its associated
   index space.
 - All global exports are required to be immutable.

> Because exports reference index spaces which include imports, modules can
re-export their imports.
> The immutability restriction might be lifted in a future version, as part of the [threads proprosal](https://github.com/WebAssembly/threads/blob/master/proposals/threads/Globals.md).

#### Start Section

**Opcode:** `0x08`.

The Start Section consists of a [varuint32] index into the
[function index space]. This is used by
[Instance Execution](#instance-execution).

**Validation:**
 - The index is required to be within the bounds of the [Code Section] array.
 - The function signature indexed in the [Type Section] is required to have an
   empty parameter list and an empty return list.

#### Element Section

**Opcode:** `0x09`.

The Element Section consists of an [array] of table initializers.

A *table initializer* consists of:

| Field Name      | Type                             | Description                                       |
| --------------- | -------------------------------- | ------------------------------------------------- |
| `index`         | [varuint32]                      | identifies a table in the [table index space]     |
| `offset`        | [instantiation-time initializer] | the index of the element in the table to start at |

If the [table]'s `element_type` is `anyfunc`, the following fields are appended.

| Field Name      | Type                             | Description                                       |
| --------------- | -------------------------------- | ------------------------------------------------- |
| `elems`         | [array] of [varuint32]           | indices into the [function index space]           |

**Validation:**
 - The type of the value returned by `offset` must be `i32`.
 - For each table initializer in the array:
    - `index` is required to be within the bounds of the [table index space].
    - A table is identified by `index` in the [table index space] and:
       - The sum of the value of `offset` and the number of elements in `elems`
         is required to be at most the `minimum` length declared for the table.
       - For each element of `elems`:
          - The element is required to be an index within the bounds of the
            associated [index space].

> The Element Sections is to the [Table Section] as the [Data Section] is to the
[Linear-Memory Section].

> Table initializers are sometimes called "segments".

#### Code Section

**Opcode:** `0x0a`.

The Code Section consists of an [array] of function bodies.

A *function body* consists of:

| Field Name      | Type                       | Description                       |
| --------------- | -------------------------- | --------------------------------- |
| `body_size`     | [varuint32]                | the size of `body` in bytes       |
| `locals`        | [array] of local entry     | local variable declarations       |
| `body`          | sequence of [instructions] | the instructions                  |

A *local entry* consists of:

| Field Name      | Type                       | Description                                     |
| --------------- | -------------------------- | ----------------------------------------------- |
| `count`         | [varuint32]                | number of local variables of the following type |
| `type`          | [value type]               | type of the variables                           |

**Validation:**
 - For each function body:
   - Control-flow constructs are required to form properly nested *regions*.
     Each [`loop`](#loop), [`block`](#block), and the function entry begin a
     region required to be terminated with an [`end`](#end). Each [`if`](#if)
     begins a region terminated with either an [`end`](#end) or an
     [`else`](#else). Each [`else`](#else) begins a region terminated with an
     [`end`](#end). Each `end` and each `else` terminates exactly one region.
   - The last instruction in the function body must be an `end`.
   - For each instruction:
      - The requirements of the **Validation** clause in the associated
        instruction description are required.
   - For each instruction reachable from at least one control-flow path:
      - The value stack is required to have at least as many elements as the
        number of operands in the instruction's signature, on every path.
      - The [types] of the operands passed to the instruction are required to
        conform to the instruction's signature's operands, on every path.
      - The [types] of the values on the value stack are required to be the
        same for all paths that reach the instruction.
      - All values that will be popped from the value stack at the instruction
        are required to have been pushed within the same region (or within a
        region nested inside it).
   - For each instruction not reachable from any control-flow path:
      - It is required that if fallthrough paths were added to every
        [barrier instruction][Q] in the function, that there exist a possible
        fall-through return type sequences for each barrier instruction, such
        that the otherwise unreachable instruction would satisfy the
        requirements for reachable instructions.

> These validation requirements are sufficient to ensure that WebAssembly has
*reducible control flow*, which essentially means that all loops have exactly
one entry point.

> There are no implicit type conversions, subtyping, or function overloading in
WebAssembly.

> The constraint on unreachable instructions is sometimes called
"polymorphic type checking", or "stack-polymorphism", however it does not
require any kind of dynamic typing behavior.

##### Positions Within A Function Body

A *position* within a function refers to an element of the instruction sequence.

#### Data Section

**Opcode:** `0x0b`.

The Data Section consists of an [array] of data initializers.

A *data initializer* consists of:

| Field Name      | Type                             | Description                                          |
| --------------- | -------------------------------- | ---------------------------------------------------- |
| `index`         | [varuint32]                      | a [linear-memory index](#linear-memory-index-space)  |
| `offset`        | [instantiation-time initializer] | the index of the byte in memory to start at          |
| `data`          | [byte array]                     | data to initialize the contents of the linear memory |

It describes data to be loaded into the linear memory identified by the index in
the [linear-memory index space] during
[linear-memory instantiation](#linear-memory-instantiation).

**Validation:**
 - The type of the value returned by `offset` must be `i32`.
 - For each data initializer in the array:
    - The linear-memory index is required to be within the bounds of the
      [linear-memory index space].
    - A linear memory is identified by the linear-memory index in the
      linear-memory index space and:
       - The sum of `offset` and the length of `data` is required to be at most
         the `minimum` length declared for the linear memory.

> Data initializers are sometimes called "segments".

### Custom Sections

Custom sections may be used for debugging information or non-standard language
extensions. The contents of the payload of a custom section after its name are
not subject to validation. Some custom sections are described here to promote
interoperability, though they aren't required to be used.

0. [Name Section]

#### Name Section

**Name:** `name`

TODO: This section currently describes a now-obsolete format. This needs to
be updated to the [new extensible name section format].

[new extensible name section format]: https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md#name-section

The Name Section consists of an [array] of function name descriptors, which
each describe names for the function with the corresponding index in the
[function index space] and which consist of:
 - the function name, an [identifier].
 - the names of the locals in the function, an [array] of [identifiers].

The Name Section should be sequenced after any known sections.

The Name Section doesn't change execution semantics and malformed constructs,
such as out-of-bounds indices, or the section not being after any known
sections, in this section cause the section to be ignored, and don't trigger
validation failures.

> Name data is represented as an explicit section in WebAssembly, however in
[text form] it may be represented as an integrated part of the syntax for
functions rather than as a discrete section.

> The expectation is that, when a binary WebAssembly module is presented in a
human-readable format in a browser or other development environment, the names
in this section are to be used as the names of functions and locals in
[text form].

### Module Index Spaces

*Module Index Spaces* are abstract mappings from indices, starting from zero, to
various types of elements.

0. [Function Index Space]
0. [Global Index Space]
0. [Linear-Memory Index Space]
0. [Table Index Space]

#### Function Index Space

The *function index space* begins with an index for each imported function, in
the order the imports appear in the [Import Section], if present, followed by an
index for each function in the [Function Section], if present, in the order of
that section.

**Validation:**
 - For each element in the index space, the type index is required to be within
   the bounds of the [Type Section] array.

> The function index space is used by [`call`](#call) instructions to identify
the callee of a direct call.

#### Global Index Space

The *global index space* begins with an index for each imported global, in the
order the imports appear in the [Import Section], if present, followed by an
index for each global in the [Global Section], if present, in the order of that
section.

> The global index space is used by:
 - the [`get_global`](#get-global) and [`set_global`](#set-global) instructions.
 - the [Data Section], to define the offset of a data initializer (in a linear
   memory) as the value of a global variable.

#### Linear-Memory Index Space

The *linear-memory index space* begins with an index for each imported linear
memory, in the order the imports appear in the [Import Section], if present,
followed by an index for each linear memory in the [Linear-Memory Section], if
present, in the order of that section.

**Validation:**
 - The index space is required to have at most one element.
 - For each linear-memory declaration in the index space:
    - If a `maximum` length is present, it is required to be at least the
      `minimum` length.
    - If a `maximum` length is present, the index of every byte in a linear
      memory with the `maximum` length is required to be representable in an
      [varuPTR].

> The validation rules here specifically avoid requiring the size in bytes of
any linear memory to be representable as a [varuPTR]. For example a 32-bit
linear-memory address space could theoretically be resized to 4 GiB if the
implementation has sufficient resources; the index of every byte would be
addressable, even though the total number of bytes would not be.

> Multiple linear memories may be permitted in the future.

> 64-bit linear memories may be permitted in the future.

##### Default Linear Memory

The linear memory with index `0`, if there is one, is called the
*default linear memory*, which is used by several instructions.

#### Table Index Space

The *table index space* begins with an index for each imported table, in the
order the imports appear in the [Import Section], if present, followed by an
index for each table in the [Table Section], if present, in the order of that
section.

**Validation:**
 - The index space is required to contain at most one table.
 - For each table in the index space:
    - If a `maximum` length is present, it is required to be at least
      the table's `minimum` length.
    - If a `maximum` length is present, the index of every element in a table
      with the `maximum` length is required to be representable in a [varuPTR].

> The table index space is currently only used by the [Element Section].

### Module Types

These types describe various data structures present in WebAssembly modules:

0. [Resizable Limits](#resizable-limits)
0. [Linear-Memory Description](#linear-memory-description)
0. [Table Description](#table-description)
0. [Global Description](#global-description)
0. [External Kinds](#external-kinds)
0. [Instantiation-Time Initializers](#instantiation-time-initializers)

> These types aren't used to describe values at execution time.

#### Resizable Limits

| Field Name | Type         | Description                                            |
| ---------- | ------------ | ------------------------------------------------------ |
| `flags`    | [varuint32]  | bit-packed flags; see below.                           |
| `minimum`  | [varuint32]  | minimum length (in units of table elements or [pages]) |

If bit `0x1` is set in `flags`, the following fields are appended.

| Field Name | Type         | Description                                            |
| ---------- | ------------ | ------------------------------------------------------ |
| `maximum`  | [varuint32]  | maximum length (in same units as `minimum`)            |

#### Linear-Memory Description

| Field Name | Type               | Description                                       |
| ---------- | ------------------ | ------------------------------------------------- |
| `limits`   | [resizable limits] | linear-memory flags and sizes in units of [pages] |

#### Table Description

| Field Name      | Type                 | Description                                |
| --------------- | -------------------- | ------------------------------------------ |
| `element_type`  | [table element type] | the element type of the [table]            |
| `resizable`     | [resizable limits]   | table flags and sizes in units of elements |

**Validation:**
 - The `element_type` is required to be `anyfunc`.

> The words "size" and "length" are used interchangeably when describing linear
memory, since the elements are byte-sized.

> In the future, other `element_type` values may be permitted.

#### Global Description

| Field Name      | Type                 | Description                               |
| --------------- | -------------------- | ----------------------------------------- |
| `type`          | [value type]         | the type of the global variable           |
| `mutability`    | [varuint1]           | `0` if immutable, `1` if mutable          |

#### External Kinds

Externals are entities which can either be defined within a module and
[exported], or [imported] from another module. They are encoded as a [varuint7]
and can be any one of the following values:

| Name       | Binary Encoding |
| ---------- | --------------- |
| `Function` | `0x00`          |
| `Table`    | `0x01`          |
| `Memory`   | `0x02`          |
| `Global`   | `0x03`          |

**Validation:**
 - The value is required to be one of the above values.

#### Instantiation-Time Initializers

An *instantiation-time initializer* is a single [instruction], which is one of
the following:
 - [`const`](#constant) (of any type).
 - [`get_global`](#get-global).

The value produced by a module initializer is the value that such an instruction
would produce if it were executed within a function body.

**Validation:**
 - The requirements of the instructions are required.
 - For `get_global` instructions, the indexed global is required to be
   an immutable import.

> In the future, more instructions may be permitted as instantiation-time
initializers.

> Instantiation-time initializers are sometimes called "constant expressions".


Instruction Descriptions
--------------------------------------------------------------------------------

Instructions in the [Instructions](#instructions) section are introduced with
tables giving a concise description of several of their attributes, followed by
additional content.

[Instructions](#instructions) are encoded as their Opcode value followed by
their immediate operand values.

0. [Instruction Mnemonic Field](#instruction-mnemonic-field)
0. [Instruction Opcode Field](#instruction-opcode-field)
0. [Instruction Immediates Field](#instruction-immediates-field)
0. [Instruction Signature Field](#instruction-signature-field)
0. [Instruction Families Field](#instruction-families-field)
0. [Instruction Description](#instruction-description)

### Instruction Mnemonic Field

Instruction [mnemonics] are short names identifying specific instructions.

Many instructions have type-specific behavior, in which case there is a unique
mnemonic for each type, formed by prepending a *type prefix*, such as `i32.` or
`f64.`, to the base instruction mnemonic.

Conversion instructions have additional type-specific behavior; their mnemonic
additionally has a *type suffix* appended, such as `/i32` or `/f64`, indicating
the input type.

The base mnemonics for [signed][S] and [unsigned][U] instructions have the
convention of ending in "_s" and "_u" respectively.

[mnemonics]: https://en.wikipedia.org/wiki/Assembly_language#Opcode_mnemonics_and_extended_mnemonics

### Instruction Opcode Field

These values are used in WebAssembly the to encode instruction [opcodes].

The opcodes for [signed][S] and [unsigned][U] instructions have the convention
that the unsigned opcode is always one greater than the signed opcode.

[opcodes]: https://en.wikipedia.org/wiki/Opcode

### Instruction Immediates Field

Immediates, if present, is a list of value names with associated
[encoding types], representing values provided by the module itself as input to
an instruction.

### Instruction Signature Field

Instruction signatures describe the explicit inputs and outputs to an
instruction. They are described in the following form:

`(` *operands* `)` `:` `(` *returns* `)`

*Operands* describes a list of [types] for values provided by program execution
as input to an instruction. *Returns* describes a list of [types] for values
computed by the instruction that are provided back to the program execution.

Within the signature for a [linear-memory access instruction][M], `iPTR` refers
an [integer  type](#integer-value-types) with the index bitwidth of the accessed
linear memory.

Besides literal [types], descriptions of [types] can be from the following
mechanisms:
 - A [typed] value name of the form

   *name*`:` *type*

   where *name* just provides an identifier for use in
   [instruction descriptions](#instruction-description). It is replaced by
   *type*.
 - A type parameter list of the form

   *name*`[` *length* `]`

   where *name* identifies the list, and *length* gives the length of the list.
   The length may be a literal value, an immediate operand value, or one of the
   named values defined below. Each type parameter in the list may be *bound* to
   a type as described in the instruction's description, or it may be inferred
   from the type of a corresponding operand value. The parameter list is
   replaced by the types bound to its parameters. If the list appears multiple
   times in a signature, it is replaced by the same types at each appearance.

#### Named Values

The following named values are defined:
 - `$args` is defined in [call instructions][L] and indicates the length of the
   callee signature parameter list.
 - `$returns` is also defined in [call instructions][L] and indicates the length
   of callee signature return list.
 - `$any` indicates the number of values on the value stack pushed within the
   enclosing region.
 - `$block_arity` is defined in [branch instructions][B] and indicates the
   number of values types in the target control-flow stack entry's signature.

### Instruction Families Field

WebAssembly instructions may belong to several families, indicated in the tables
by their family letter:

0. [B: Branch Instruction Family][B]
0. [Q: Control-Flow Barrier Instruction Family][Q]
0. [L: Call Instruction Family][L]
0. [G: Generic Integer Instruction Family][G]
0. [S: Signed Integer Instruction Family][S]
0. [U: Unsigned Integer Instruction Family][U]
0. [T: Shift Instruction Family][T]
0. [R: Remainder Instruction Family][R]
0. [F: Floating-Point Instruction Family][F]
0. [E: Floating-Point Bitwise Instruction Family][E]
0. [C: Comparison Instruction Family][C]
0. [M: Linear-Memory Access Instruction Family][M]
0. [Z: Linear-Memory Size Instruction Family][Z]

#### B: Branch Instruction Family

##### Branching

In a branch according to a given control-flow stack entry, first the value stack
is resized down to the entry's limit value.

Then, if the entry's [label] is bound, the current position is set to the bound
position. Otherwise, the position to bind the label to is found by scanning
forward through the instructions, as if executing just [`block`](#block),
[`loop`](#loop), and [`end`](#end) instructions, until the label is bound. Then
the current position is set to that position.

Then, control-flow stack entries are popped until the given control-flow stack
entry is popped.

> In practice, implementations may precompute the destinations of branches so
that they don't literally need to scan in this manner.

> Branching is sometimes called "jumping" in other languages.

> Branch instructions can only target [labels] within the same function.

##### Branch Index Validation

A depth index is a valid branch index if it is less than the length of the
control-flow stack at the branch instruction.

#### Q: Control-Flow Barrier Instruction Family

These instructions either trap or reassign the current position, such that
execution doesn't proceed (or "fall through") to the instruction that lexically
follows them.

#### L: Call Instruction Family

##### Calling

If the called function&mdash;the *callee*&mdash;is a function in the module, it
is [executed](#function-execution). Otherwise the callee is an imported function
which is executed according to its own semantics. The [`$args`] operands,
excluding `$callee` when present, are passed as the incoming arguments. The
return value of the call is defined by the execution.

At least one unit of [call-stack resources] is consumed during the execution of
the callee, and released when it completes.

**Trap:** Call Stack Exhausted, if the instance has insufficient
[call-stack resources].

**Trap:** Callee Trap, if a trap occurred during the execution of the callee.

> This means that implementations aren't permitted to perform implicit
opportunistic tail-call optimization (TCO).

> The execution state of the function currently being executed remains live
during the call, and the execution of the called function is performed
independently. In this way, calls form a stack-like data structure called the
*call stack*.

> Data associated with the call stack is stored outside any linear-memory
address space and isn't directly accessible to applications.

##### Call Validation

 - The members of `$T[`[`$args`]`]` are bound to the operand types of the callee
   signature, and the members of `$T[`[`$returns`]`]` are bound to the return
   types of the callee signature.

#### G: Generic Integer Instruction Family

Except where otherwise specified, these instructions don't specifically
interpret their operands as explicitly signed or unsigned, and therefore don't
have an inherent concept of overflow.

#### S: Signed Integer Instruction Family

Except where otherwise specified, these instructions interpret their operand
values as signed, return result values interpreted as signed, and [trap] when
the result value can't be represented as such.

#### U: Unsigned Integer Instruction Family

Except where otherwise specified, these instructions interpret their operand
values as unsigned, return result values interpreted as unsigned, and [trap]
when the result value can't be represented as such.

#### T: Shift Instruction Family

In the shift and rotate instructions, *left* means in the direction of greater
significance, and *right* means in the direction of lesser significance.

##### Shift Count

The second operand in shift and rotate instructions specifies a *shift count*,
which is interpreted as an unsigned quantity modulo the number of bits in the
first operand.

> As a result of the modulo, in `i32.` instructions, only the least-significant
5 bits of the second operand affect the result, and in `i64.` instructions only
the least-significant 6 bits of the second operand affect the result.

> The shift count is interpreted as unsigned even in otherwise signed
instructions such as [`shr_s`](#integer-shift-right-signed).

#### R: Remainder Instruction Family

> The remainder instructions (`%`) are related to their corresponding division
instructions (`/`) by the identity `x == (x/y)*y + (x%y)`.

#### F: Floating-Point Instruction Family

Instructions in this family follow the [IEEE 754-2008] standard, except that:

 - They support only "non-stop" mode, and floating-point exceptions aren't
   otherwise observable. In particular, neither alternate floating-point
   exception handling attributes nor the non-computational operations on status
   flags are supported.

 - They use the IEEE 754-2008 `roundTiesToEven` rounding attribute, except where
   otherwise specified. Non-default directed rounding attributes aren't
   supported.

 - Extended and extendable precision formats aren't supported. All computations
   must be strictly and correctly rounded after each instruction.

> The exception and rounding behavior specified here are the default behavior on
most contemporary software environments.

> All computations are correctly rounded, subnormal values are fully supported,
and negative zero, NaNs, and infinities are all produced as result values to
indicate overflow, invalid, and divide-by-zero exceptional conditions, and
interpreted appropriately when they appear as operands. Compiler optimizations
that introduce changes to the effective precision, rounding, or range of any
computation are not permitted. Implementations are not permitted to contract or
fuse operations to elide intermediate rounding steps. All numeric results are
deterministic, as are the rules for how NaNs are handled as operands and for
when NaNs are to be generated as results. The only floating-point nondeterminism
is in the specific bit-patterns of NaN result values.

> In IEEE 754-1985, ["subnormal numbers"] are called "denormal numbers";
WebAssembly follows IEEE 754-2008, which calls them "subnormal numbers".

When the result of any instruction in this family (which excludes `neg`, `abs`,
`copysign`, `load`, `store`, and `const`) is a NaN, the sign bit and the
significand field (which doesn't include the implicit leading digit of the
significand) of the NaN are computed by one of the following rules,
selected [nondeterministically]:

 - If the instruction has any NaN non-immediate operand values with significand
   fields that have any bits set to `1` other than the most significant bit of
   the significand field, the result is a NaN with a nondeterministic sign bit,
   `1` in the most significant digit of the significand, and nondeterministic
   values in the remaining bits of the significand field.

 - Otherwise, the result is a NaN with a nondeterministic sign bit, `1` in the
   most significant digit of the significand, and `0` in the remaining bits of
   the significand field.

Implementations are permitted to further implement the IEEE 754-2008 section
"Operations with NaNs" recommendation that operations propagate NaN bits from
their operands, however it isn't required.

> The NaN propagation rules are intended to support NaN-boxing.

> At present, there is no observable difference between quiet and signaling NaN
other than the difference in the bit pattern.

> IEEE 754-2008 is the current revision of IEEE 754; a new revision is expected
to be released some time in 2018, and it's expected to be a minor and
backwards-compatible revision, so WebAssembly is expected to update to it.

[IEEE 754-2008]: https://en.wikipedia.org/wiki/IEEE_floating_point
["subnormal numbers"]: https://en.wikipedia.org/wiki/Subnormal_number

#### E: Floating-Point Bitwise Instruction Family

These instructions operate on floating-point values, but do so in purely bitwise
ways, including in how they operate on NaN and zero values.

They correspond to the "Sign bit operations" in IEEE 754-2008.

#### C: Comparison Instruction Family

WebAssembly comparison instructions compare two values and return a [boolean]
result value.

> In accordance with IEEE 754-2008, for the comparison instructions, negative
zero is considered equal to zero, and NaN values aren't less than, greater than,
or equal to any other values, including themselves.

#### M: Linear-Memory Access Instruction Family

These instructions load from and store to a linear memory.

##### Effective Address

The *effective address* of a linear-memory access is computed by adding `$base`
and `$offset`, both interpreted as unsigned, at infinite range and precision, so
that there is no overflow.

##### Alignment

**Slow:** If the effective address isn't a multiple of `$align`, the access is
*misaligned*, and the instruction may execute very slowly.

> When `$align` is at least the size of the access, the access is
*naturally aligned*. When it's less, the access is *unaligned*. Naturally
aligned accesses may be faster than unaligned accesses, though both may be much
faster than misaligned accesses.

> There is no other semantic effect associated with `$align`; misaligned and
unaligned loads and stores still behave normally.

##### Accessed Bytes

The *accessed bytes* consist of a contiguous sequence of [bytes] starting at the
[effective address], with a size implied by the accessing instruction.

**Trap:** Out Of Bounds, if any of the accessed bytes are beyond the end of the
accessed linear memory. This trap is triggered before any of the bytes are
actually accessed.

> Linear-memory accesses trap on an out-of-bound access, which differs from
[TypedArrays in ECMAScript] where storing out of bounds silently does nothing
and loading out of bounds silently returns `undefined`.

[TypedArrays in ECMAScript]: https://tc39.github.io/ecma262/#sec-typedarray-objects

##### Loading

For a load access, a value is read from the [accessed bytes], in
[little-endian byte order], and returned.

##### Storing

For a store access, the value to store is written to the [accessed bytes], in
[little-endian byte order].

> If any of the bytes are out of bounds, the Out Of Bounds trap is triggered
before any of the bytes are written to.

##### Linear-Memory Access Validation

 - `$align` is required to be at most the number of [accessed bytes] (the
   [*natural alignment*](#alignment)).
 - The module is required to contain a default linear memory.

#### Z: Linear-Memory Size Instruction Family

##### Linear-Memory Size Validation

 - The module is required to contain a default linear memory.

### Instruction Description

Instruction semantics are described for use in the context of
[function-body execution](#function-body-execution). Some instructions also have
a special validation clause, introduced by "**Validation:**", which defines
instruction-specific validation requirements.


Instructions
--------------------------------------------------------------------------------

0. [Control Flow Instructions](#control-flow-instructions)
0. [Basic Instructions](#basic-instructions)
0. [Integer Arithmetic Instructions](#integer-arithmetic-instructions)
0. [Floating-Point Arithmetic Instructions](#floating-point-arithmetic-instructions)
0. [Integer Comparison Instructions](#integer-comparison-instructions)
0. [Floating-Point Comparison Instructions](#floating-point-comparison-instructions)
0. [Conversion Instructions](#conversion-instructions)
0. [Load And Store Instructions](#load-and-store-instructions)
0. [Additional Memory-Related Instructions](#additional-memory-related-instructions)

### Control Flow Instructions

0. [Block](#block)
0. [Loop](#loop)
0. [Unconditional Branch](#unconditional-branch)
0. [Conditional Branch](#conditional-branch)
0. [Table Branch](#table-branch)
0. [If](#if)
0. [Else](#else)
0. [End](#end)
0. [Return](#return)
0. [Unreachable](#unreachable)

#### Block

| Mnemonic    | Opcode | Immediates                           | Signature | Families |
| ----------- | ------ | ------------------------------------ | --------- | -------- |
| `block`     | 0x02   | `$signature`: [block signature type] | `() : ()` |          |

The `block` instruction pushes an entry onto the control-flow stack. The entry
contains an unbound [label], the current length of the value stack, and
`$signature`.

> Each `block` needs a corresponding [`end`](#end) to bind its label and pop
its control-flow stack entry.

#### Loop

| Mnemonic    | Opcode | Immediates                           | Signature | Families |
| ----------- | ------ | ------------------------------------ | --------- | -------- |
| `loop`      | 0x03   | `$signature`: [block signature type] | `() : ()` |          |

The `loop` instruction binds a [label] to the current position, and pushes an
entry onto the control-flow stack. The entry contains that label, the current
length of the value stack, and `$signature`.

> The `loop` instruction doesn't perform a loop by itself. It merely introduces
a label that may be used by a branch to form an actual loop.

> Since `loop`'s control-flow stack entry starts with an empty type sequence,
branches to the top of the loop must not have any result values.

> Each `loop` needs a corresponding [`end`](#end) to pop its control-flow stack
entry.

> There is no requirement that loops eventually terminate or contain observable
side effects.

#### Unconditional Branch

| Mnemonic    | Opcode | Immediates            | Signature                                             | Families |
| ----------- | ------ | --------------------- | ----------------------------------------------------- | -------- |
| `br`        | 0x0c   | `$depth`: [varuint32] | `($T[`[`$block_arity`]`]) : ($T[`[`$block_arity`]`])` | [B] [Q]  |

The `br` instruction [branches](#branching) according to the control-flow stack
entry `$depth` from the top. It returns the values of its operands.

**Validation:**
 - $depth is required to be a [valid branch index](#branch-index-validation).

TODO: Explicitly describe the binding of $T.

#### Conditional Branch

| Mnemonic    | Opcode | Immediates            | Signature                                                              | Families |
| ----------- | ------ | --------------------- | ---------------------------------------------------------------------- | -------- |
| `br_if`     | 0x0d   | `$depth`: [varuint32] | `($T[`[`$block_arity`]`], $condition: i32) : ($T[`[`$block_arity`]`])` | [B]      |

If `$condition` is [true], the `br_if` instruction [branches](#branching)
according to the control-flow stack entry `$depth` from the top. Otherwise, it
does nothing (and control "falls through"). It returns the values of its
operands, except `$condition`.

**Validation:**
 - $depth is required to be a [valid branch index](#branch-index-validation).

TODO: Explicitly describe the binding of $T.

#### Table Branch

| Mnemonic    | Opcode | Immediates                                                | Signature                                                          | Families |
| ----------- | ------ | --------------------------------------------------------- | ------------------------------------------------------------------ | -------- |
| `br_table`  | 0x0e   | `$table`: [array] of [varuint32], `$default`: [varuint32] | `($T[`[`$block_arity`]`], $index: i32) : ($T[`[`$block_arity`]`])` | [B] [Q]  |

First, the `br_table` instruction selects a depth to use. If `$index` is within
the bounds of `$table`, the depth is the value of the indexed `$table` element.
Otherwise, the depth is `$default`.

Then, it [branches](#branching) according to the control-flow stack entry that
depth from the top. It returns the values of its operands, except `$index`.

**Validation:**
 - All entries of the table, and `$default`, are required to be
   [valid branch indices](#branch-index-validation).

> This instruction serves the role of what is sometimes called a ["jump table"]
in other languages. "Branch" is used here instead to emphasize the commonality
with the other branch instructions.

> The `$default` label isn't considered to be part of the branch table.

["jump table"]: https://en.wikipedia.org/w/index.php?title=Jump_table

TODO: Explicitly describe the binding of $T.

#### If

| Mnemonic    | Opcode | Immediates                           | Signature                   | Families |
| ----------- | ------ | ------------------------------------ | --------------------------- | -------- |
| `if`        | 0x04   | `$signature`: [block signature type] | `($condition: i32) : ()`    | [B]      |

The `if` instruction pushes an entry onto the control-flow stack. The entry
contains an unbound [label], the current length of the value stack, and
`$signature`. If `$condition` is [false], it then [branches](#branching)
according to this entry.

> Each `if` needs either a corresponding [`else`](#else) or [`end`](#end) to
bind its label and pop its control-flow stack entry.

#### Else

| Mnemonic    | Opcode | Signature                             | Families |
| ----------- | ------ | ------------------------------------- | -------- |
| `else`      | 0x05   | `($T[`[`$any`]`]) : ($T[`[`$any`]`])` | [B]      |

The `else` instruction binds the control-flow stack top's [label] to the current
position, pops an entry from the control-flow stack, pushes a new entry onto the
control-flow stack containing an unbound [label], the length of the current
value stack, and the signature of the control-flow stack entry that was just
popped, and then [branches](#branching) according to this entry. It returns the
values of its operands.

**Validation:**
 - `$T[`[`$any`]`]` is required to be the type sequence described by the
   signature of the popped control-flow stack entry.

> Each `else` needs a corresponding [`end`](#end) to bind its label and pop its
control-flow stack entry.

> Unlike in the branch instructions, `else` and `end` do not ignore surplus
values on the stack, as [`$any`] is bound to the number of values pushed within
the current block.

TODO: Explicitly describe the binding of $T.

#### End

| Mnemonic    | Opcode | Signature                             | Families |
| ----------- | ------ | ------------------------------------- | -------- |
| `end`       | 0x0b   | `($T[`[`$any`]`]) : ($T[`[`$any`]`])` |          |

The `end` instruction pops an entry from the control-flow stack. If the entry's
[label] is unbound, the label is bound to the current position. It returns the
values of its operands.

**Validation:**
 - `$T[`[`$any`]`]` is required to be the type sequence described by the
   signature of the popped control-flow stack entry.
 - If the control-flow stack entry was pushed by an `if` (and there was no
   `else`), the signature is required to be `void`.

> Each `end` ends a region begun by a corresponding `block`, `loop`, `if`,
`else`, or the function entry.

> Unlike in the branch instructions, `else` and `end` do not ignore surplus
values on the stack, as [`$any`] is bound to the number of values pushed within
the current block.

TODO: Explicitly describe the binding of $T.

#### Return

| Mnemonic    | Opcode | Signature                                             | Families |
| ----------- | ------ | ----------------------------------------------------- | -------- |
| `return`    | 0x0f   | `($T[`[`$block_arity`]`]) : ($T[`[`$block_arity`]`])` | [B] [Q]  |

The `return` instruction [branches](#branching) according to the control-flow
stack bottom. It returns the values of its operands.

> `return` is semantically equivalent to a `br` to the outermost control region.

> Implementations needn't literally perform a branch before performing the
actual function return.

TODO: Explicitly describe the binding of $T.

#### Unreachable

| Mnemonic      | Opcode | Signature                 | Families |
| ------------- | ------ | ------------------------- | -------- |
| `unreachable` | 0x00   | `() : ()`                 | [Q]      |

**Trap:** Unreachable reached, always.

> The `unreachable` instruction is meant to represent code that isn't meant to
be executed except in the case of a bug in the application.

### Basic Instructions

0. [No-Op](#nop)
0. [Drop](#drop)
0. [Constant](#constant)
0. [Get Local](#get-local)
0. [Set Local](#set-local)
0. [Tee Local](#tee-local)
0. [Get Global](#get-global)
0. [Set Global](#set-global)
0. [Select](#select)
0. [Call](#call)
0. [Indirect Call](#indirect-call)

#### No-Op

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `nop`       | 0x01   | `() : ()`                   |          |

The `nop` instruction does nothing.

#### Drop

| Mnemonic    | Opcode | Signature                    | Families |
| ----------- | ------ | ---------------------------- | -------- |
| `drop`      | 0x1a   | `($T[1]) : ()`               |          |

The `drop` instruction does nothing.

> This differs from `nop` in that it has an operand, so it can be used to
discard unneeded values from the value stack.

> This instruction is sometimes called "value-polymorphic" because it can
accept values of any type.

TODO: Explicitly describe the binding of $T.

#### Constant

| Mnemonic    | Opcode | Immediates            | Signature    | Families |
| ----------- | ------ | --------------------- | ------------ | -------- |
| `i32.const` | 0x41   | `$value`: [varsint32] | `() : (i32)` |          |
| `i64.const` | 0x42   | `$value`: [varsint64] | `() : (i64)` |          |
| `f32.const` | 0x43   | `$value`: [float32]   | `() : (f32)` |          |
| `f64.const` | 0x44   | `$value`: [float64]   | `() : (f64)` |          |

The `const` instruction returns the value of `$value`.

> Floating-point constants can be created with arbitrary bit-patterns.

#### Get Local

| Mnemonic    | Opcode | Immediates         | Signature      | Families |
| ----------- | ------ | ------------------ | -------------- | -------- |
| `get_local` | 0x20   | `$id`: [varuint32] | `() : ($T[1])` |          |

The `get_local` instruction returns the value of the local at index `$id` in the
locals vector of the current [function execution]. The type parameter is bound
to the type of the local.

**Validation:**
 - `$id` is required to be within the bounds of the locals vector.

#### Set Local

| Mnemonic    | Opcode | Immediates         | Signature      | Families |
| ----------- | ------ | ------------------ | -------------- | -------- |
| `set_local` | 0x21   | `$id`: [varuint32] | `($T[1]) : ()` |          |

The `set_local` instruction sets the value of the local at index `$id` in the
locals vector of the current [function execution] to the value given in the
operand. The type parameter is bound to the type of the local.

**Validation:**
 - `$id` is required to be within the bounds of the locals vector.

> `set_local` is semantically equivalent to a similar `tee_local` followed by a
`drop`.

#### Tee Local

| Mnemonic    | Opcode | Immediates         | Signature           | Families |
| ----------- | ------ | ------------------ | ------------------- | -------- |
| `tee_local` | 0x22   | `$id`: [varuint32] | `($T[1]) : ($T[1])` |          |

The `tee_local` instruction sets the value of the locals at index `$id` in the
locals vector of the current [function execution] to the value given in the
operand. Its return value is the value of its operand. The type parameter is
bound to the type of the local.

**Validation:**
 - `$id` is required to be within the bounds of the locals vector.

> This instruction's name is inspired by the ["tee" command] in other languages,
since it forwards the value of its operand to two places, the local and the
return value.

["tee" command]: https://en.wikipedia.org/wiki/Tee_(command)

#### Get Global

| Mnemonic     | Opcode | Immediates         | Signature      | Families |
| ------------ | ------ | ------------------ | -------------- | -------- |
| `get_global` | 0x23   | `$id`: [varuint32] | `() : ($T[1])` |          |

The `get_global` instruction returns the value of the global identified by index
`$id` in the [global index space]. The type parameter is bound to the type of
the global.

**Validation:**
 - `$id` is required to be within the bounds of the global index space.

#### Set Global

| Mnemonic     | Opcode | Immediates         | Signature      | Families |
| ------------ | ------ | ------------------ | -------------- | -------- |
| `set_global` | 0x24   | `$id`: [varuint32] | `($T[1]) : ()` |          |

The `set_global` instruction sets the value of the global identified by index
`$id` in the [global index space] to the value given in the operand. The type
parameter is bound to the type of the global.

**Validation:**
 - `$id` is required to be within the bounds of the global index space.
 - The indexed global is required to be declared not immutable.

#### Select

| Mnemonic    | Opcode | Signature                                   | Families |
| ----------- | ------ | ------------------------------------------- | -------- |
| `select`    | 0x1b   | `($T[1], $T[1], $condition: i32) : ($T[1])` |          |

The `select` instruction returns its first operand if `$condition` is [true], or
its second operand otherwise.

> This instruction differs from the conditional or ternary operator, eg.
`x?y:z`, in some languages, in that it's not short-circuiting.

> This instruction is similar to a "conditional move" in other languages and is
meant to have similar performance properties.

> This instruction is sometimes called "value-polymorphic" because it can
operate on values of any type.

TODO: Explicitly describe the binding of $T.

#### Call

| Mnemonic    | Opcode | Immediates             | Signature                                  | Families |
| ----------- | ------ | ---------------------- | ------------------------------------------ | -------- |
| `call`      | 0x10   | `$callee`: [varuint32] | `($T[`[`$args`]`]) : ($T[`[`$returns`]`])` | [L]      |

The `call` instruction performs a [call](#calling) to the function with index
`$callee` in the [function index space].

**Validation:**
 - `$callee` is required to be within the bounds of the function index space.
 - [Call validation](#call-validation) is required; the callee signature is the
   signature of the indexed function.

#### Indirect Call

| Mnemonic        | Opcode | Immediates                                         | Signature                                                | Families |
| --------------- | ------ | -------------------------------------------------- | -------------------------------------------------------- | -------- |
| `call_indirect` | 0x11   | `$signature`: [varuint32], `$reserved`: [varuint1] | `($T[`[`$args`]`], $callee: i32) : ($T[`[`$returns`]`])` | [L]      |

The `call_indirect` instruction performs a [call](#calling) to the function in
the default table with index `$callee`.

**Trap:** Indirect Callee Absent, if the indexed table element is the special
"null" value.

**Trap:** Indirect Call Type Mismatch, if the signature of the function with
index `$callee` differs from the signature in the [Type Section] with index
`$signature`.

**Validation:**
 - [Call validation](#call-validation) is required; the callee signature is the
   signature with index `$signature` in the [Type Section].

> The dynamic caller/callee signature match is structural rather than nominal.

> Indices in the default table can provide applications with the functionality
of function pointers.

> In future versions of WebAssembly, the reserved immediate may be used to index additional tables.

### Integer Arithmetic Instructions

0. [Integer Add](#integer-add)
0. [Integer Subtract](#integer-subtract)
0. [Integer Multiply](#integer-multiply)
0. [Integer Divide, Signed](#integer-divide-signed)
0. [Integer Divide, Unsigned](#integer-divide-unsigned)
0. [Integer Remainder, Signed](#integer-remainder-signed)
0. [Integer Remainder, Unsigned](#integer-remainder-unsigned)
0. [Integer Bitwise And](#integer-bitwise-and)
0. [Integer Bitwise Or](#integer-bitwise-or)
0. [Integer Bitwise Exclusive-Or](#integer-exclusive-bitwise-or)
0. [Integer Shift Left](#integer-shift-left)
0. [Integer Shift Right, Signed](#integer-shift-right-signed)
0. [Integer Shift Right, Unsigned](#integer-shift-right-unsigned)
0. [Integer Rotate Left](#integer-rotate-left)
0. [Integer Rotate Right](#integer-rotate-right)
0. [Integer Count Leading Zeros](#integer-count-leading-zeros)
0. [Integer Count Trailing Zeros](#integer-count-trailing-zeros)
0. [Integer Population Count](#integer-population-count)
0. [Integer Equal To Zero](#integer-equal-to-zero)

#### Integer Add

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.add`   | 0x6a   | `(i32, i32) : (i32)`        | [G]      |
| `i64.add`   | 0x7c   | `(i64, i64) : (i64)`        | [G]      |

The integer `add` instruction returns the [two's complement sum] of its
operands. The carry bit is silently discarded.

> Due to WebAssembly's use of [two's complement] to represent signed values,
this instruction can be used to add either signed or unsigned values.

#### Integer Subtract

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.sub`   | 0x6b   | `(i32, i32) : (i32)`        | [G]      |
| `i64.sub`   | 0x7d   | `(i64, i64) : (i64)`        | [G]      |

The integer `sub` instruction returns the [two's complement difference] of its
operands. The borrow bit is silently discarded.

> Due to WebAssembly's use of [two's complement] to represent signed values,
this instruction can be used to subtract either signed or unsigned values.

> An integer negate operation can be performed by a `sub` instruction with zero
as the first operand.

#### Integer Multiply

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.mul`   | 0x6c   | `(i32, i32) : (i32)`        | [G]      |
| `i64.mul`   | 0x7e   | `(i64, i64) : (i64)`        | [G]      |

The integer `mul` instruction returns the low half of the
[two's complement product] its operands.

> Due to WebAssembly's use of [two's complement] to represent signed values,
this instruction can be used to multiply either signed or unsigned values.

#### Integer Divide, Signed

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.div_s` | 0x6d   | `(i32, i32) : (i32)`        | [S]      |
| `i64.div_s` | 0x7f   | `(i64, i64) : (i64)`        | [S]      |

The `div_s` instruction returns the signed quotient of its operands, interpreted
as signed. The quotient is silently rounded to the nearest integer toward zero.

**Trap:** Signed Integer Overflow, when the [minimum signed integer value] is
divided by `-1`.

**Trap:** Integer Division By Zero, when the second operand (the divisor) is
zero.

#### Integer Divide, Unsigned

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.div_u` | 0x6e   | `(i32, i32) : (i32)`        | [U]      |
| `i64.div_u` | 0x80   | `(i64, i64) : (i64)`        | [U]      |

The `div_u` instruction returns the unsigned quotient of its operands,
interpreted as unsigned. The quotient is silently rounded to the nearest integer
toward zero.

**Trap:** Integer Division By Zero, when the second operand (the divisor) is
zero.

#### Integer Remainder, Signed

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.rem_s` | 0x6f   | `(i32, i32) : (i32)`        | [S] [R]  |
| `i64.rem_s` | 0x81   | `(i64, i64) : (i64)`        | [S] [R]  |

The `rem_s` instruction returns the signed remainder from a division of its
operand values interpreted as signed, with the result having the same sign as
the first operand (the dividend).

**Trap:** Integer Division By Zero, when the second operand (the divisor) is
zero.

> This instruction doesn't trap when the [minimum signed integer value] is
divided by `-1`; it returns `0` which is the correct remainder (even though the
same operands to `div_s` do cause a trap).

> This instruction differs from what is often called a ["modulo" operation] in
its handling of negative numbers.

> This instruction has some [common pitfalls] to avoid.

["modulo" operation]: https://en.wikipedia.org/wiki/Modulo_operation
[common pitfalls]: https://en.wikipedia.org/wiki/Modulo_operation#Common_pitfalls

#### Integer Remainder, Unsigned

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.rem_u` | 0x70   | `(i32, i32) : (i32)`        | [U] [R]  |
| `i64.rem_u` | 0x82   | `(i64, i64) : (i64)`        | [U] [R]  |

The `rem_u` instruction returns the unsigned remainder from a division of its
operand values interpreted as unsigned.

**Trap:** Integer Division By Zero, when the second operand (the divisor) is
zero.

> This instruction corresponds to what is sometimes called "modulo" in other
languages.

#### Integer Bitwise And

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.and`   | 0x71   | `(i32, i32) : (i32)`        | [G]      |
| `i64.and`   | 0x83   | `(i64, i64) : (i64)`        | [G]      |

The `and` instruction returns the [bitwise and] of its operands.

[bitwise and]: https://en.wikipedia.org/wiki/Bitwise_operation#AND

#### Integer Bitwise Or

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.or`    | 0x72   | `(i32, i32) : (i32)`        | [G]      |
| `i64.or`    | 0x84   | `(i64, i64) : (i64)`        | [G]      |

The `or` instruction returns the [bitwise inclusive-or] of its operands.

[bitwise inclusive-or]: https://en.wikipedia.org/wiki/Bitwise_operation#OR

#### Integer Bitwise Exclusive-Or

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.xor`   | 0x73   | `(i32, i32) : (i32)`        | [G]      |
| `i64.xor`   | 0x85   | `(i64, i64) : (i64)`        | [G]      |

The `xor` instruction returns the [bitwise exclusive-or] of its operands.

> A [bitwise negate] operation can be performed by a `xor` instruction with
negative one as the first operand, an operation sometimes called
"one's complement" in other languages.

[bitwise exclusive-or]: https://en.wikipedia.org/wiki/Bitwise_operation#XOR
[bitwise negate]: https://en.wikipedia.org/wiki/Bitwise_operation#NOT

#### Integer Shift Left

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.shl`   | 0x74   | `(i32, i32) : (i32)`        | [T], [G] |
| `i64.shl`   | 0x86   | `(i64, i64) : (i64)`        | [T], [G] |

The `shl` instruction returns the value of the first operand [shifted] to the
left by the [shift count].

> This instruction effectively performs a multiplication by two to the power of
the shift count.

#### Integer Shift Right, Signed

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.shr_s` | 0x75   | `(i32, i32) : (i32)`        | [T], [S] |
| `i64.shr_s` | 0x87   | `(i64, i64) : (i64)`        | [T], [S] |

The `shr_s` instruction returns the value of the first operand
[shifted](https://en.wikipedia.org/wiki/Arithmetic_shift) to the right by the
[shift count].

> This instruction corresponds to what is sometimes called
"arithmetic right shift" in other languages.

> `shr_s` is similar to `div_s` when the divisor is a power of two, however the
rounding of negative values is different. `shr_s` effectively rounds down, while
`div_s` rounds toward zero.

#### Integer Shift Right, Unsigned

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.shr_u` | 0x76   | `(i32, i32) : (i32)`        | [T], [U] |
| `i64.shr_u` | 0x88   | `(i64, i64) : (i64)`        | [T], [U] |

The `shr_u` instruction returns the value of the first operand [shifted] to the
right by the [shift count].

> This instruction corresponds to what is sometimes called
"logical right shift" in other languages.

> This instruction effectively performs an unsigned division by two to the power
of the shift count.

#### Integer Rotate Left

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.rotl`  | 0x77   | `(i32, i32) : (i32)`        | [T], [G] |
| `i64.rotl`  | 0x89   | `(i64, i64) : (i64)`        | [T], [G] |

The `rotl` instruction returns the value of the first operand [rotated] to the
left by the [shift count].

> Rotating left is similar to shifting left, however vacated bits are set to the
values of the bits which would otherwise be discarded by the shift, so the bits
conceptually "rotate back around".

#### Integer Rotate Right

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.rotr`  | 0x78   | `(i32, i32) : (i32)`        | [T], [G] |
| `i64.rotr`  | 0x8a   | `(i64, i64) : (i64)`        | [T], [G] |

The `rotr` instruction returns the value of the first operand [rotated] to the
right by the [shift count].

> Rotating right is similar to shifting right, however vacated bits are set to
the values of the bits which would otherwise be discarded by the shift, so the
bits conceptually "rotate back around".

#### Integer Count Leading Zeros

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.clz`   | 0x67   | `(i32) : (i32)`             | [G]      |
| `i64.clz`   | 0x79   | `(i64) : (i64)`             | [G]      |

The `clz` instruction returns the number of leading zeros in its operand. The
*leading zeros* are the longest contiguous sequence of zero-bits starting at the
most significant bit and extending downward.

> This instruction is fully defined when all bits are zero; it returns the
number of bits in the operand type.

#### Integer Count Trailing Zeros

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.ctz`   | 0x68   | `(i32) : (i32)`             | [G]      |
| `i64.ctz`   | 0x7a   | `(i64) : (i64)`             | [G]      |

The `ctz` instruction returns the number of trailing zeros in its operand. The
*trailing zeros* are the longest contiguous sequence of zero-bits starting at
the least significant bit and extending upward.

> This instruction is fully defined when all bits are zero; it returns the
number of bits in the operand type.

#### Integer Population Count

| Mnemonic     | Opcode | Signature                  | Families |
| ------------ | ------ | -------------------------- | -------- |
| `i32.popcnt` | 0x69   | `(i32) : (i32)`            | [G]      |
| `i64.popcnt` | 0x7b   | `(i64) : (i64)`            | [G]      |

The `popcnt` instruction returns the number of 1-bits in its operand.

> This instruction is fully defined when all bits are zero; it returns `0`.

> This instruction corresponds to what is sometimes called a ["hamming weight"]
in other languages.

["hamming weight"]: https://en.wikipedia.org/wiki/Hamming_weight

#### Integer Equal To Zero

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.eqz`   | 0x45   | `(i32) : (i32)`             | [G]      |
| `i64.eqz`   | 0x50   | `(i64) : (i32)`             | [G]      |

The `eqz` instruction returns [true] if the operand is equal to zero, or [false]
otherwise.

> This serves as a form of "logical not" operation which can be used to invert
[boolean] values.

### Floating-Point Arithmetic Instructions

0. [Floating-Point Add](#floating-point-add)
0. [Floating-Point Subtract](#floating-point-subtract)
0. [Floating-Point Multiply](#floating-point-multiply)
0. [Floating-Point Divide](#floating-point-divide)
0. [Floating-Point Square Root](#floating-point-square-root)
0. [Floating-Point Minimum](#floating-point-minimum)
0. [Floating-Point Maximum](#floating-point-maximum)
0. [Floating-Point Ceiling](#floating-point-ceiling)
0. [Floating-Point Floor](#floating-point-floor)
0. [Floating-Point Truncate](#floating-point-truncate)
0. [Floating-Point Round To Nearest Integer](#floating-point-round-to-nearest-integer)
0. [Floating-Point Absolute Value](#floating-point-absolute-value)
0. [Floating-Point Negate](#floating-point-negate)
0. [Floating-Point CopySign](#floating-point-copysign)

#### Floating-Point Add

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.add`   | 0x92   | `(f32, f32) : (f32)`        | [F]      |
| `f64.add`   | 0xa0   | `(f64, f64) : (f64)`        | [F]      |

The floating-point `add` instruction performs the IEEE 754-2008 `addition`
operation according to the [general floating-point rules][F].

#### Floating-Point Subtract

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.sub`   | 0x93   | `(f32, f32) : (f32)`        | [F]      |
| `f64.sub`   | 0xa1   | `(f64, f64) : (f64)`        | [F]      |

The floating-point `sub` instruction performs the IEEE 754-2008 `subtraction`
operation according to the [general floating-point rules][F].

#### Floating-Point Multiply

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.mul`   | 0x94   | `(f32, f32) : (f32)`        | [F]      |
| `f64.mul`   | 0xa2   | `(f64, f64) : (f64)`        | [F]      |

The floating-point `mul` instruction performs the IEEE 754-2008 `multiplication`
operation according to the [general floating-point rules][F].

#### Floating-Point Divide

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.div`   | 0x95   | `(f32, f32) : (f32)`        | [F]      |
| `f64.div`   | 0xa3   | `(f64, f64) : (f64)`        | [F]      |

The `div` instruction performs the IEEE 754-2008 `division` operation according
to the [general floating-point rules][F].

#### Floating-Point Square Root

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.sqrt`  | 0x91   | `(f32) : (f32)`             | [F]      |
| `f64.sqrt`  | 0x9f   | `(f64) : (f64)`             | [F]      |

The `sqrt` instruction performs the IEEE 754-2008 `squareRoot` operation
according to the [general floating-point rules][F].

#### Floating-Point Minimum

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.min`   | 0x96   | `(f32, f32) : (f32)`        | [F]      |
| `f64.min`   | 0xa4   | `(f64, f64) : (f64)`        | [F]      |

The `min` instruction returns the minimum value among its operands. For this
instruction, negative zero is considered less than zero. If either operand is a
NaN, the result is a NaN determined by the [general floating-point rules][F].

> This instruction corresponds to what is sometimes called "minNaN" in other
languages.

> This differs from the IEEE 754-2008 `minNum` operation in that it returns a
NaN if either operand is a NaN, and in that the behavior when the operands are
zeros of differing signs is fully specified.

> This differs from the common `x<y?x:y` expansion in its handling of
negative zero and NaN values.

#### Floating-Point Maximum

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.max`   | 0x97   | `(f32, f32) : (f32)`        | [F]      |
| `f64.max`   | 0xa5   | `(f64, f64) : (f64)`        | [F]      |

The `max` instruction returns the maximum value among its operands. For this
instruction, negative zero is considered less than zero. If either operand is a
NaN, the result is a NaN determined by the [general floating-point rules][F].

> This instruction corresponds to what is sometimes called "maxNaN" in other
languages.

> This differs from the IEEE 754-2008 `maxNum` operation in that it returns a
NaN if either operand is a NaN, and in that the behavior when the operands are
zeros of differing signs is fully specified.

> This differs from the common `x>y?x:y` expansion in its handling of negative
zero and NaN values.

#### Floating-Point Ceiling

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.ceil`  | 0x8d   | `(f32) : (f32)`             | [F]      |
| `f64.ceil`  | 0x9b   | `(f64) : (f64)`             | [F]      |

The `ceil` instruction performs the IEEE 754-2008
`roundToIntegralTowardPositive` operation according to the
[general floating-point rules][F].

> ["Ceiling"][Floor and Ceiling Functions] describes the rounding method used
here; the value is rounded up to the nearest integer.

#### Floating-Point Floor

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.floor` | 0x8e   | `(f32) : (f32)`             | [F]      |
| `f64.floor` | 0x9c   | `(f64) : (f64)`             | [F]      |

The `floor` instruction performs the IEEE 754-2008
`roundToIntegralTowardNegative` operation according to the
[general floating-point rules][F].

> ["Floor"][Floor and Ceiling Functions] describes the rounding method used
here; the value is rounded down to the nearest integer.

#### Floating-Point Truncate

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.trunc` | 0x8f   | `(f32) : (f32)`             | [F]      |
| `f64.trunc` | 0x9d   | `(f64) : (f64)`             | [F]      |

The `trunc` instruction performs the IEEE 754-2008
`roundToIntegralTowardZero` operation according to the
[general floating-point rules][F].

> ["Truncate"] describes the rounding method used here; the fractional part of
the value is discarded, effectively rounding to the nearest integer toward zero.

> This form of rounding is called a `chop` in other languages.

["Truncate"]: https://en.wikipedia.org/wiki/Truncation

#### Floating-Point Round To Nearest Integer

| Mnemonic      | Opcode | Signature                 | Families |
| ------------- | ------ | ------------------------- | -------- |
| `f32.nearest` | 0x90   | `(f32) : (f32)`           | [F]      |
| `f64.nearest` | 0x9e   | `(f64) : (f64)`           | [F]      |

The `nearest` instruction performs the IEEE 754-2008
`roundToIntegralTiesToEven` operation according to the
[general floating-point rules][F].

> "Nearest" describes the rounding method used here; the value is
[rounded to the nearest integer], with
[ties rounded toward the value with an even least-significant digit].

> This instruction differs from [`Math.round` in ECMAScript] which rounds ties
up, and it differs from [`round` in C] which rounds ties away from zero.

> This instruction corresponds to what is called `roundeven` in other languages.

[rounded to the nearest integer]: https://en.wikipedia.org/wiki/Nearest_integer_function
[ties rounded toward the value with an even least-significant digit]: https://en.wikipedia.org/wiki/Rounding#Round_half_to_even
[`Math.round` in ECMAScript]: https://tc39.github.io/ecma262/#sec-math.round
[`round` in C]: http://en.cppreference.com/w/c/numeric/math/round

#### Floating-Point Absolute Value

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.abs`   | 0x8b   | `(f32) : (f32)`             | [E]      |
| `f64.abs`   | 0x99   | `(f64) : (f64)`             | [E]      |

The `abs` instruction performs the IEEE 754-2008 `abs` operation.

> This is a bitwise instruction; it sets the sign bit to zero and preserves all
other bits, even when the operand is a NaN or a zero.

> This differs from comparing whether the operand value is less than zero and
negating it, because comparisons treat negative zero as equal to zero, and NaN
values as not less than zero.

#### Floating-Point Negate

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.neg`   | 0x8c   | `(f32) : (f32)`             | [E]      |
| `f64.neg`   | 0x9a   | `(f64) : (f64)`             | [E]      |

The `neg` instruction performs the IEEE 754-2008 `negate` operation.

> This is a bitwise instruction; it inverts the sign bit and preserves all other
bits, even when the operand is a NaN or a zero.

> This differs from subtracting the operand value from negative zero or
multiplying it by negative one, because subtraction and multiplication follow
the [general floating-point rules][F] and may not preserve the bits of NaN
values.

#### Floating-Point CopySign

| Mnemonic       | Opcode | Signature                | Families |
| -------------- | ------ | ------------------------ | -------- |
| `f32.copysign` | 0x98   | `(f32, f32) : (f32)`     | [E]      |
| `f64.copysign` | 0xa6   | `(f64, f64) : (f64)`     | [E]      |

The `copysign` instruction performs the IEEE 754-2008 `copySign` operation.

> This is a bitwise instruction; it combines the sign bit from the second
operand with all bits other than the sign bit from the first operand, even if
either operand is a NaN or a zero.

### Integer Comparison Instructions

0. [Integer Equality](#integer-equality)
0. [Integer Inequality](#integer-inequality)
0. [Integer Less Than, Signed](#integer-less-than-signed)
0. [Integer Less Than, Unsigned](#integer-less-than-unsigned)
0. [Integer Less Than Or Equal To, Signed](#integer-less-than-or-equal-to-signed)
0. [Integer Less Than Or Equal To, Unsigned](#integer-less-than-or-equal-to-unsigned)
0. [Integer Greater Than, Signed](#integer-greater-than-signed)
0. [Integer Greater Than, Unsigned](#integer-greater-than-unsigned)
0. [Integer Greater Than Or Equal To, Signed](#integer-greater-than-or-equal-to-signed)
0. [Integer Greater Than Or Equal To, Unsigned](#integer-greater-than-or-equal-to-unsigned)

#### Integer Equality

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.eq`    | 0x46   | `(i32, i32) : (i32)`        | [C], [G] |
| `i64.eq`    | 0x51   | `(i64, i64) : (i32)`        | [C], [G] |

The integer `eq` instruction tests whether the operands are equal.

#### Integer Inequality

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.ne`    | 0x47   | `(i32, i32) : (i32)`        | [C], [G] |
| `i64.ne`    | 0x52   | `(i64, i64) : (i32)`        | [C], [G] |

The integer `ne` instruction tests whether the operands are not equal.

> This instruction corresponds to what is sometimes called "differs" in other
languages.

#### Integer Less Than, Signed

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.lt_s`  | 0x48   | `(i32, i32) : (i32)`        | [C], [S] |
| `i64.lt_s`  | 0x53   | `(i64, i64) : (i32)`        | [C], [S] |

The `lt_s` instruction tests whether the first operand is less than the second
operand, interpreting the operands as signed.

#### Integer Less Than, Unsigned

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.lt_u`  | 0x49   | `(i32, i32) : (i32)`        | [C], [U] |
| `i64.lt_u`  | 0x54   | `(i64, i64) : (i32)`        | [C], [U] |

The `lt_u` instruction tests whether the first operand is less than the second
operand, interpreting the operands as unsigned.

#### Integer Less Than Or Equal To, Signed

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.le_s`  | 0x4c   | `(i32, i32) : (i32)`        | [C], [S] |
| `i64.le_s`  | 0x57   | `(i64, i64) : (i32)`        | [C], [S] |

The `le_s` instruction tests whether the first operand is less than or equal to
the second operand, interpreting the operands as signed.

> This instruction corresponds to what is sometimes called "at most" in other
languages.

#### Integer Less Than Or Equal To, Unsigned

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.le_u`  | 0x4d   | `(i32, i32) : (i32)`        | [C], [U] |
| `i64.le_u`  | 0x58   | `(i64, i64) : (i32)`        | [C], [U] |

The `le_u` instruction tests whether the first operand is less than or equal to
the second operand, interpreting the operands as unsigned.

> This instruction corresponds to what is sometimes called "at most" in other
languages.

#### Integer Greater Than, Signed

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.gt_s`  | 0x4a   | `(i32, i32) : (i32)`        | [C], [S] |
| `i64.gt_s`  | 0x55   | `(i64, i64) : (i32)`        | [C], [S] |

The `gt_s` instruction tests whether the first operand is greater than the
second operand, interpreting the operands as signed.

#### Integer Greater Than, Unsigned

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.gt_u`  | 0x4b   | `(i32, i32) : (i32)`        | [C], [U] |
| `i64.gt_u`  | 0x56   | `(i64, i64) : (i32)`        | [C], [U] |

The `gt_u` instruction tests whether the first operand is greater than the
second operand, interpreting the operands as unsigned.

#### Integer Greater Than Or Equal To, Signed

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.ge_s`  | 0x4e   | `(i32, i32) : (i32)`        | [C], [S] |
| `i64.ge_s`  | 0x59   | `(i64, i64) : (i32)`        | [C], [S] |

The `ge_s` instruction tests whether the first operand is greater than or equal
to the second operand, interpreting the operands as signed.

> This instruction corresponds to what is sometimes called "at least" in other
languages.

#### Integer Greater Than Or Equal To, Unsigned

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `i32.ge_u`  | 0x4f   | `(i32, i32) : (i32)`        | [C], [U] |
| `i64.ge_u`  | 0x5a   | `(i64, i64) : (i32)`        | [C], [U] |

The `ge_u` instruction tests whether the first operand is greater than or equal
to the second operand, interpreting the operands as unsigned.

> This instruction corresponds to what is sometimes called "at least" in other
languages.

### Floating-Point Comparison Instructions

0. [Floating-Point Equality](#floating-point-equality)
0. [Floating-Point Inequality](#floating-point-inequality)
0. [Floating-Point Less Than](#floating-point-less-than)
0. [Floating-Point Less Than Or Equal To](#floating-point-less-than-or-equal-to)
0. [Floating-Point Greater Than](#floating-point-greater-than)
0. [Floating-Point Greater Than Or Equal To](#floating-point-greater-than-or-equal-to)

#### Floating-Point Equality

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.eq`    | 0x5b   | `(f32, f32) : (i32)`        | [C], [F] |
| `f64.eq`    | 0x61   | `(f64, f64) : (i32)`        | [C], [F] |

The floating-point `eq` instruction performs the IEEE 754-2008
`compareQuietEqual` operation according to the
[general floating-point rules][F].

> This instruction corresponds to what is sometimes called "ordered and equal",
or "oeq", in other languages.

#### Floating-Point Inequality

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.ne`    | 0x5c   | `(f32, f32) : (i32)`        | [C], [F] |
| `f64.ne`    | 0x62   | `(f64, f64) : (i32)`        | [C], [F] |

The floating-point `ne` instruction performs the IEEE 754-2008
`compareQuietNotEqual` operation according to the
[general floating-point rules][F].

> Unlike the other floating-point comparison instructions, this instruction
returns [true] if either operand is a NaN. It is the logical inverse of the `eq`
instruction.

> This instruction corresponds to what is sometimes called
"unordered or not equal", or "une", in other languages.

#### Floating-Point Less Than

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.lt`    | 0x5d   | `(f32, f32) : (i32)`        | [C], [F] |
| `f64.lt`    | 0x63   | `(f64, f64) : (i32)`        | [C], [F] |

The `lt` instruction performs the IEEE 754-2008 `compareQuietLess` operation
according to the [general floating-point rules][F].

> This instruction corresponds to what is sometimes called "ordered and less
than", or "olt", in other languages.

#### Floating-Point Less Than Or Equal To

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.le`    | 0x5f   | `(f32, f32) : (i32)`        | [C], [F] |
| `f64.le`    | 0x65   | `(f64, f64) : (i32)`        | [C], [F] |

The `le` instruction performs the IEEE 754-2008 `compareQuietLessEqual`
operation according to the [general floating-point rules][F].

> This instruction corresponds to what is sometimes called "ordered and less
than or equal", or "ole", in other languages.

#### Floating-Point Greater Than

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.gt`    | 0x5e   | `(f32, f32) : (i32)`        | [C], [F] |
| `f64.gt`    | 0x64   | `(f64, f64) : (i32)`        | [C], [F] |

The `gt` instruction performs the IEEE 754-2008 `compareQuietGreater` operation
according to the [general floating-point rules][F].

> This instruction corresponds to what is sometimes called "ordered and greater
than", or "ogt", in other languages.

#### Floating-Point Greater Than Or Equal To

| Mnemonic    | Opcode | Signature                   | Families |
| ----------- | ------ | --------------------------- | -------- |
| `f32.ge`    | 0x60   | `(f32, f32) : (i32)`        | [C], [F] |
| `f64.ge`    | 0x66   | `(f64, f64) : (i32)`        | [C], [F] |

The `ge` instruction performs the IEEE 754-2008 `compareQuietGreaterEqual`
operation according to the [general floating-point rules][F].

> This instruction corresponds to what is sometimes called "ordered and greater
than or equal", or "oge", in other languages.

### Conversion Instructions

0. [Integer Wrap](#integer-wrap)
0. [Integer Extend, Signed](#integer-extend-signed)
0. [Integer Extend, Unsigned](#integer-extend-unsigned)
0. [Truncate Floating-Point to Integer, Signed](#truncate-floating-point-to-integer-signed)
0. [Truncate Floating-Point to Integer, Unsigned](#truncate-floating-point-to-integer-unsigned)
0. [Floating-Point Demote](#floating-point-demote)
0. [Floating-Point Promote](#floating-point-promote)
0. [Convert Integer To Floating-Point, Signed](#convert-integer-to-floating-point-signed)
0. [Convert Integer To Floating-Point, Unsigned](#convert-integer-to-floating-point-unsigned)
0. [Reinterpret](#reinterpret)

#### Integer Wrap

| Mnemonic       | Opcode | Signature                | Families |
| -------------- | ------ | ------------------------ | -------- |
| `i32.wrap/i64` | 0xa7   | `(i64) : (i32)`          | [G]      |

The `wrap` instruction returns the value of its operand silently wrapped to its
result type. Wrapping means reducing the value modulo the number of unique
values in the result type.

> This instruction corresponds to what is sometimes called an integer "truncate"
in other languages, however WebAssembly uses the word "truncate" to mean
effectively discarding the least significant digits, and the word "wrap" to mean
effectively discarding the most significant digits.

#### Integer Extend, Signed

| Mnemonic           | Opcode | Signature            | Families |
| ------------------ | ------ | -------------------- | -------- |
| `i64.extend_s/i32` | 0xac   | `(i32) : (i64)`      | [S]      |

The `extend_s` instruction returns the value of its operand [sign-extended] to
its result type.

#### Integer Extend, Unsigned

| Mnemonic           | Opcode | Signature            | Families |
| ------------------ | ------ | -------------------- | -------- |
| `i64.extend_u/i32` | 0xad   | `(i32) : (i64)`      | [U]      |

The `extend_u` instruction returns the value of its operand zero-extended to its
result type.

#### Truncate Floating-Point to Integer, Signed

| Mnemonic          | Opcode | Signature             | Families |
| ----------------- | ------ | --------------------- | -------- |
| `i32.trunc_s/f32` | 0xa8   | `(f32) : (i32)`       | [F], [S] |
| `i32.trunc_s/f64` | 0xaa   | `(f64) : (i32)`       | [F], [S] |
| `i64.trunc_s/f32` | 0xae   | `(f32) : (i64)`       | [F], [S] |
| `i64.trunc_s/f64` | 0xb0   | `(f64) : (i64)`       | [F], [S] |

The `trunc_s` instruction performs the IEEE 754-2008
`convertToIntegerTowardZero` operation, with the result value interpreted as
signed, according to the [general floating-point rules][F].

**Trap:** Invalid Conversion To Integer, when a floating-point Invalid condition
occurs, due to the operand being outside the range that can be converted
(including NaN values and infinities).

> This form of rounding is called a `chop` in other languages.

#### Truncate Floating-Point to Integer, Unsigned

| Mnemonic          | Opcode | Signature             | Families |
| ----------------- | ------ | --------------------- | -------- |
| `i32.trunc_u/f32` | 0xa9   | `(f32) : (i32)`       | [F], [U] |
| `i32.trunc_u/f64` | 0xab   | `(f64) : (i32)`       | [F], [U] |
| `i64.trunc_u/f32` | 0xaf   | `(f32) : (i64)`       | [F], [U] |
| `i64.trunc_u/f64` | 0xb1   | `(f64) : (i64)`       | [F], [U] |

The `trunc_u` instruction performs the IEEE 754-2008
`convertToIntegerTowardZero` operation, with the result value interpreted as
unsigned, according to the [general floating-point rules][F].

**Trap:** Invalid Conversion To Integer, when an Invalid condition occurs, due
to the operand being outside the range that can be converted (including NaN
values and infinities).

> This instruction's result is unsigned, so it almost always rounds down,
however it does round up in one place: negative values greater than negative one
truncate up to zero.

#### Floating-Point Demote

| Mnemonic         | Opcode | Signature              | Families |
| ---------------- | ------ | ---------------------- | -------- |
| `f32.demote/f64` | 0xb6   | `(f64) : (f32)`        | [F]      |

The `demote` instruction performs the IEEE 754-2008 `convertFormat` operation,
converting from its operand type to its result type, according to the
[general floating-point rules][F].

> This is a narrowing conversion which may round or overflow to infinity.

#### Floating-Point Promote

| Mnemonic          | Opcode | Signature             | Families |
| ----------------- | ------ | --------------------- | -------- |
| `f64.promote/f32` | 0xbb   | `(f32) : (f64)`       | [F]      |

The `promote` instruction performs the IEEE 754-2008 `convertFormat` operation,
converting from its operand type to its result type, according to the
[general floating-point rules][F].

> This is a widening conversion and is always exact.

#### Convert Integer To Floating-Point, Signed

| Mnemonic            | Opcode | Signature           | Families |
| ------------------- | ------ | ------------------- | -------- |
| `f32.convert_s/i32` | 0xb2   | `(i32) : (f32)`     | [F], [S] |
| `f32.convert_s/i64` | 0xb4   | `(i64) : (f32)`     | [F], [S] |
| `f64.convert_s/i32` | 0xb7   | `(i32) : (f64)`     | [F], [S] |
| `f64.convert_s/i64` | 0xb9   | `(i64) : (f64)`     | [F], [S] |

The `convert_s` instruction performs the IEEE 754-2008 `convertFromInt`
operation, with its operand value interpreted as signed, according to the
[general floating-point rules][F].

> `f64.convert_s/i32` is always exact; the other instructions here may round.

#### Convert Integer To Floating-Point, Unsigned

| Mnemonic            | Opcode | Signature           | Families |
| ------------------- | ------ | ------------------- | -------- |
| `f32.convert_u/i32` | 0xb3   | `(i32) : (f32)`     | [F], [U] |
| `f32.convert_u/i64` | 0xb5   | `(i64) : (f32)`     | [F], [U] |
| `f64.convert_u/i32` | 0xb8   | `(i32) : (f64)`     | [F], [U] |
| `f64.convert_u/i64` | 0xba   | `(i64) : (f64)`     | [F], [U] |

The `convert_u` instruction performs the IEEE 754-2008 `convertFromInt`
operation, with its operand value interpreted as unsigned, according to the
[general floating-point rules][F].

> `f64.convert_u/i32` is always exact; the other instructions here may round.

#### Reinterpret

| Mnemonic              | Opcode | Signature         | Families |
| --------------------- | ------ | ----------------- | -------- |
| `i32.reinterpret/f32` | 0xbc   | `(f32) : (i32)`   |          |
| `i64.reinterpret/f64` | 0xbd   | `(f64) : (i64)`   |          |
| `f32.reinterpret/i32` | 0xbe   | `(i32) : (f32)`   |          |
| `f64.reinterpret/i64` | 0xbf   | `(i64) : (f64)`   |          |

The `reinterpret` instruction returns a value which has the same bit-pattern as
its operand value, in its result type.

> The operand type is always the same width as the result type, so this
instruction is always exact.

### Load And Store Instructions

0. [Load](#load)
0. [Store](#store)
0. [Extending Load, Signed](#extending-load-signed)
0. [Extending Load, Unsigned](#extending-load-unsigned)
0. [Wrapping Store](#wrapping-store)

#### Load

| Mnemonic    | Opcode | Immediates                                 | Signature               | Families |
| ----------- | ------ | ------------------------------------------ | ----------------------- | -------- |
| `i32.load`  | 0x28   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i32)` | [M], [G] |
| `i64.load`  | 0x29   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i64)` | [M], [G] |
| `f32.load`  | 0x2a   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (f32)` | [M], [E] |
| `f64.load`  | 0x2b   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (f64)` | [M], [E] |

The `load` instruction performs a [load](#loading) of the same size as its type
from the [default linear memory].

Floating-point loads preserve all the bits of the value, performing an
IEEE 754-2008 `copy` operation.

**Validation:**
 - [Linear-memory access validation] is required.

#### Store

| Mnemonic    | Opcode | Immediates                                 | Signature                         | Families |
| ----------- | ------ | ------------------------------------------ | --------------------------------- | -------- |
| `i32.store` | 0x36   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR, $value: i32) : ()` | [M], [G] |
| `i64.store` | 0x37   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR, $value: i64) : ()` | [M], [G] |
| `f32.store` | 0x38   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR, $value: f32) : ()` | [M], [F] |
| `f64.store` | 0x39   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR, $value: f64) : ()` | [M], [F] |

The `store` instruction performs a [store](#storing) of `$value` of the same
size as its type to the [default linear memory].

Floating-point stores preserve all the bits of the value, performing an
IEEE 754-2008 `copy` operation.

**Validation:**
 - [Linear-memory access validation] is required.

#### Extending Load, Signed

| Mnemonic       | Opcode | Immediates                                 | Signature               | Families |
| -------------- | ------ | ------------------------------------------ | ----------------------- | -------- |
| `i32.load8_s`  | 0x2c   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i32)` | [M], [S] |
| `i32.load16_s` | 0x2e   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i32)` | [M], [S] |
|                |        |                                            |                         |          |
| `i64.load8_s`  | 0x30   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i64)` | [M], [S] |
| `i64.load16_s` | 0x32   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i64)` | [M], [S] |
| `i64.load32_s` | 0x34   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i64)` | [M], [S] |

The signed extending load instructions perform a [load](#loading) of narrower
width than their type from the [default linear memory], and return the value
[sign-extended] to their type.
 - `load8_s` loads an 8-bit value.
 - `load16_s` loads a 16-bit value.
 - `load32_s` loads a 32-bit value.

**Validation:**
 - [Linear-memory access validation] is required.

#### Extending Load, Unsigned

| Mnemonic       | Opcode | Immediates                                 | Signature               | Families |
| -------------- | ------ | ------------------------------------------ | ----------------------- | -------- |
| `i32.load8_u`  | 0x2d   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i32)` | [M], [U] |
| `i32.load16_u` | 0x2f   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i32)` | [M], [U] |
|                |        |                                            |                         |          |
| `i64.load8_u`  | 0x31   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i64)` | [M], [U] |
| `i64.load16_u` | 0x33   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i64)` | [M], [U] |
| `i64.load32_u` | 0x35   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR) : (i64)` | [M], [U] |

The unsigned extending load instructions perform a [load](#loading) of narrower
width than their type from the [default linear memory], and return the value
zero-extended to their type.
 - `load8_u` loads an 8-bit value.
 - `load16_u` loads a 16-bit value.
 - `load32_u` loads a 32-bit value.

**Validation:**
 - [Linear-memory access validation] is required.

#### Wrapping Store

| Mnemonic      | Opcode | Immediates                                 | Signature                         | Families |
| ------------- | ------ | ------------------------------------------ | --------------------------------- | -------- |
| `i32.store8`  | 0x3a   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR, $value: i32) : ()` | [M], [G] |
| `i32.store16` | 0x3b   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR, $value: i32) : ()` | [M], [G] |
|               |        |                                            |                                   |          |
| `i64.store8`  | 0x3c   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR, $value: i64) : ()` | [M], [G] |
| `i64.store16` | 0x3d   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR, $value: i64) : ()` | [M], [G] |
| `i64.store32` | 0x3e   | `$flags`: [memflags], `$offset`: [varuPTR] | `($base: iPTR, $value: i64) : ()` | [M], [G] |

The wrapping store instructions performs a [store](#storing) of `$value` to the
[default linear memory], silently wrapped to a narrower width.
 - `store8` stores an 8-bit value.
 - `store16` stores a 16-bit value.
 - `store32` stores a 32-bit value.

**Validation:**
 - [Linear-memory access validation] is required.

> See the comment in the [wrap instruction](#integer-wrap) about the meaning of
the name "wrap".

### Additional Memory-Related Instructions

0. [Grow Linear-Memory Size](#grow-linear-memory-size)
0. [Current Linear-Memory Size](#current-linear-memory-size)

#### Grow Linear-Memory Size

| Mnemonic      | Opcode | Immediates              | Signature                 | Families |
| ------------- | ------ | ----------------------- | ------------------------- | -------- |
| `grow_memory` | 0x40   | `$reserved`: [varuint1] | `($delta: iPTR) : (iPTR)` | [Z]      |

The `grow_memory` instruction increases the size of the [default linear memory]
by `$delta`, in units of unsigned [pages]. If the index of any byte of the
referenced linear memory would be unrepresentable as unsigned in an `iPTR`, if
allocation fails due to insufficient dynamic resources, or if the linear memory
has a `maximum` length and the actual size would exceed the `maximum` length, it
returns `-1` and the linear-memory size is not increased; otherwise the
linear-memory size is increased, and `grow_memory` returns the previous
linear-memory size, also as an unsigned value in units of [pages]. Newly
allocated bytes are initialized to all zeros.

**Validation**:
 - [Linear-memory size validation](#linear-memory-size-validation) is required.
 - `$reserved` is required to be `0`.

> This instruction can fail even when the `maximum` length isn't yet reached,
due to resource exhaustion.

> Since the return value is in units of pages, `-1` isn't otherwise a valid
linear-memory size. Also, note that `-1` isn't the only "negative" value (when
interpreted as signed) that can be returned; other such values can indicate
valid returns.

> `$reserved` is intended for future use.

#### Current Linear-Memory Size

| Mnemonic         | Opcode | Immediates              | Signature              | Families |
| ---------------- | ------ | ----------------------- | ---------------------- | -------- |
| `current_memory` | 0x3f   | `$reserved`: [varuint1] | `() : (iPTR)`          | [Z]      |

The `current_memory` instruction returns the size of the [default linear
memory], as an unsigned value in units of [pages].

**Validation**:
 - [Linear-memory size validation](#linear-memory-size-validation) is required.
 - `$reserved` is required to be `0`.

> `$reserved` is intended for future use.


Instantiation
--------------------------------------------------------------------------------

WebAssembly code execution requires an *instance* of a module, which contains a
reference to the module plus additional information added during instantiation,
which consists of the following steps:
 - The entire module is first validated according to the requirements of the
   **Validation** clause of the [top-level module description](#module-contents)
   and all clauses it transitively requires. If there are any failures,
   instantiation aborts and doesn't produce an instance.
 - If a [Linear-Memory Section] is present, each linear memory is
   [instantiated](#linear-memory-instantiation).
 - If a [Table Section] is present, each table is
   [instantiated](#table-instantiation).
 - A finite quantity of [call-stack resources] is allocated.
 - A *globals vector* is allocated, which is a heterogeneous vector of globals
   with an element for each entry in the module's [Global Section], if present.
   The initial value of each global is the value of its
   [instantiation-time initializer](#instantiation-time-initializers), if it has
   one, or an all-zeros bit-pattern otherwise.

**Trap:** Dynamic Resource Exhaustion, if dynamic resources are insufficient to
support creation of the module instance or any of its components.

> The contents of an instance, including functions and their bodies, are outside
any linear-memory address space and not any accessible to applications.
WebAssembly is therefore conceptually a [Harvard Architecture].

[Harvard Architecture]: https://en.wikipedia.org/wiki/Harvard_architecture

#### Linear-Memory Instantiation

A linear memory is instantiated as follows:

For a linear-memory definition in the [Linear-Memory Section], as opposed to a
[linear-memory import](#import-section), a vector of [bytes] with the length
being the value of the linear memory's `minimum` length field times the
[page size] is created, added to the instance, and initialized to all zeros. For
a linear-memory import, storage for the vector is already allocated.

For each [Data Section] entry with and `index` value equal to the index of the
linear memory, the contents of its `data` field are copied into the linear
memory starting at its `offset` field.

**Trap:** Dynamic Resource Exhaustion, if dynamic resources are insufficient to
support creation of any of the vectors.

#### Table Instantiation

A table is instantiated as follows:

For a table definition in the [Table Section], as opposed to a
[table import](#import-section), a vector of elements is created with the
table's `minimum` length, with elements of the table's element type, and
initialized to all special "null" values specific to the element type. For a
table import, storage for the table is already allocated.

For each table initializer in the [Element Section], for the table identified by
the table index in the [table index space]:
 - A contiguous of elements in the table starting at the table initializer's
   start offset is initialized according to the elements of the table element
   initializers array, which specify an indexed element in their selected index
   space.

**Trap:** Dynamic Resource Exhaustion, if dynamic resources are insufficient to
support creation of any of the tables.

#### Call-Stack Resources

Call-stack resources are an abstract quantity, with discrete units, of which a
[nondeterministic] amount is allocated during instantiation, belonging to an
instance.

> These resources is used by [call instructions][L].

> The specific resource limit serves as an upper bound only; implementations may
[nondeterministically] perform a trap sooner if they exhaust other dynamic
resources.


Execution
--------------------------------------------------------------------------------

0. [Instance Execution](#instance-execution)
0. [Function Execution](#function-execution)

### Instance Execution

If the module contains a [Start Section], the referenced function is
[executed](#function-execution).

### Function Execution

TODO: This section should be improved to be more approachable.

Function execution can be prompted by a [call-family instruction][L] from within
the same module, by [instance execution](#instance-execution), or by a call to
an [exported](#export-section) function from another module or from the
[embedding environment].

The input to execution of a function consists of:
 - the function to be executed.
 - the incoming argument values, one for each parameter [type] of the function.
 - a module instance

For the duration of the execution of a function body, several data structures
are created:
 - A *control-flow stack*, with each entry containing
    - a [label] for reference from branch instructions.
    - a *limit* integer value, which is an index into the value stack indicating
      where to reset it to on a branch to that label.
    - a *signature*, which is a [block signature type] indicating the number and
      types of result values of the region.
 - A *value stack*, which carries values between instructions.
 - A *locals* vector, a heterogeneous vector of values containing an element for
   each type in the function's parameter list, followed by an element for each
   local declaration in the function.
 - A *current position*.

> Implementations needn't create a literal vector to store the locals, or
literal stacks to manage values at execution time.

> These data structures are all allocated outside any linear-memory address
space and are not any accessible to applications.

#### Function Execution Initialization

The current position starts at the first instruction in the function body. The
value stack begins empty. The control-flow stack begins with an entry holding a
[label] bound to the last instruction in the instruction sequence, a limit value
of zero, and a signature corresponding to the function's return types:
 - If the function's return type sequence is empty, its signature is `void`.
 - If the function's return type sequence has exactly one element, the signature
   is that element.

The value of each incoming argument is copied to the local with the
corresponding index, and the rest of the locals are initialized to all-zeros
bit-pattern values.

#### Function-Body Execution

The instruction at the current position is remembered, and the current position
is incremented to point to the position following it. Then the remembered
instruction is executed as follows:

For each operand [type] in the instruction's signature in reverse order, a value
is popped from the value stack and provided as the corresponding operand value.
The instruction is then executed as described in the
[Instruction](#instructions) description for it. Each of the
instruction's return values are then pushed onto the value stack.

If the current position is now past the end of the sequence,
[function return execution](#function-return-execution) is initiated and
execution of the function is thereafter complete.

Otherwise, [execution](#function-body-execution) is restarted with the new
current position.

**Trap:** Dynamic Resource Exhaustion, if any dynamic resource used by the
implementation is exhausted, at any point during function-body execution.

#### Labels

A label is a value which is either *unbound*, or *bound* to a specific position.

#### Function Return Execution

One value for each return [type] in the function signature in reverse order is
popped from the value stack. If the function execution was prompted by a
[call instruction][L], these values are provided as the call's return values.
Otherwise, they are provided to the [embedding environment].

#### Instruction Traps

Instructions may *trap*, in which case the function execution which encountered
the trap is immediately terminated. If the function execution was prompted by a
[call instruction][L], it traps too. Otherwise, abnormal termination is reported
to the [embedding environment].

> Except for the call stack and the state of executing functions, the contents
of an instance, including any linear memories, are left intact after a trap.
This allows inspection by debugging tools and crash reporters. It is also valid
to call exported functions in an instance that has seen a trap.


Text Format
--------------------------------------------------------------------------------

TODO: Describe the text format.


[B]: #b-branch-instruction-family
[Q]: #q-control-flow-barrier-instruction-family
[L]: #l-call-instruction-family
[G]: #g-generic-integer-instruction-family
[S]: #s-signed-integer-instruction-family
[U]: #u-unsigned-integer-instruction-family
[T]: #t-shift-instruction-family
[R]: #r-remainder-instruction-family
[F]: #f-floating-point-instruction-family
[E]: #e-floating-point-bitwise-instruction-family
[C]: #c-comparison-instruction-family
[M]: #m-linear-memory-access-instruction-family
[Z]: #z-linear-memory-size-instruction-family
[Type Section]: #type-section
[Import Section]: #import-section
[Function Section]: #function-section
[Table Section]: #table-section
[Linear-Memory Section]: #linear-memory-section
[Export Section]: #export-section
[Start Section]: #start-section
[Code Section]: #code-section
[Data Section]: #data-section
[Global Section]: #global-section
[Element Section]: #element-section
[Name Section]: #name-section
[function index space]: #function-index-space
[global index space]: #global-index-space
[linear-memory index space]: #linear-memory-index-space
[table index space]: #table-index-space
[accessed bytes]: #accessed-bytes
[array]: #array
[binary32]: https://en.wikipedia.org/wiki/Single-precision_floating-point_format
[binary64]: https://en.wikipedia.org/wiki/Double-precision_floating-point_format
[bit]: https://en.wikipedia.org/wiki/Bit
[block signature type]: #block-signature-types
[boolean]: #booleans
[byte]: #bytes
[bytes]: #bytes
[byte array]: #byte-array
[call-stack resources]: #call-stack-resources
[default linear memory]: #default-linear-memory
[effective address]: #effective-address
[embedding environment]: #embedding-environment
[embedding environments]: #embedding-environment
[encoding type]: #encoding-types
[encoding types]: #encoding-types
[exported]: #export-section
[external kind]: #external-kinds
[false]: #booleans
[function execution]: #function-execution
[global description]: #global-description
[Floor and Ceiling Functions]: https://en.wikipedia.org/wiki/Floor_and_ceiling_functions
[identifier]: #identifier
[identifiers]: #identifier
[imported]: #import-section
[index space]: #module-index-spaces
[instantiation-time initializer]: #instantiation-time-initializers
[instruction]: #instructions
[instructions]: #instructions
[varuPTR]: #varuptr-immediate-type
[KiB]: https://en.wikipedia.org/wiki/Kibibyte
[known section]: #known-sections
[label]: #labels
[labels]: #labels
[language type]: #language-types
[linear memory]: #linear-memories
[linear memories]: #linear-memories
[linear-memory]: #linear-memories
[linear-memory access validation]: #linear-memory-access-validation
[linear-memory description]: #linear-memory-description
[linear-memory descriptions]: #linear-memory-description
[little-endian byte order]: https://en.wikipedia.org/wiki/Endianness#Little-endian
[memflags]: #memflags-immediate-type
[minimum signed integer value]: https://en.wikipedia.org/wiki/Two%27s_complement#Most_negative_number
[nondeterministic]: #nondeterminism
[nondeterministically]: #nondeterminism
[page]: #pages
[pages]: #pages
[page size]: #pages
[resizable limits]: #resizable-limits
[rotated]: https://en.wikipedia.org/wiki/Bitwise_operation#Rotate_no_carry
[shift count]: #shift-count
[shifted]: https://en.wikipedia.org/wiki/Logical_shift
[sign-extended]: https://en.wikipedia.org/wiki/Sign_extension
[signature type]: #signature-types
[table]: #tables
[tables]: #tables
[table element type]: #table-element-types
[table description]: #table-description
[table descriptions]: #table-description
[text form]: #text-format
[true]: #booleans
[type]: #value-types
[types]: #value-types
[typed]: #value-types
[trap]: #instruction-traps
[two's complement]: https://en.wikipedia.org/wiki/Two%27s_complement
[two's complement difference]: https://en.wikipedia.org/wiki/Two%27s_complement#Subtraction
[two's complement product]: https://en.wikipedia.org/wiki/Two%27s_complement#Multiplication
[two's complement sum]: https://en.wikipedia.org/wiki/Two%27s_complement#Addition
[value type]: #value-types
[value types]: #value-types
[uint32]: #primitive-type-encodings
[UTF-8]: https://en.wikipedia.org/wiki/UTF-8
[valid UTF-8]: https://encoding.spec.whatwg.org/#utf-8-decode-without-bom-or-fail
[varuint1]: #primitive-type-encodings
[varuint7]: #primitive-type-encodings
[varuint32]: #primitive-type-encodings
[varuint64]: #primitive-type-encodings
[varsint7]: #primitive-type-encodings
[varsint32]: #primitive-type-encodings
[varsint64]: #primitive-type-encodings
[float32]: #primitive-type-encodings
[float64]: #primitive-type-encodings
[`$args`]: #named-values
[`$returns`]: #named-values
[`$any`]: #named-values
[`$block_arity`]: #named-values
