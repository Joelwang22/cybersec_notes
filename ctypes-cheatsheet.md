# Python `ctypes`: Native Interface Guide

The standard-library `ctypes` module provides C-compatible data types and lets
Python call functions exported by DLLs and shared libraries. It is useful for
small native wrappers, operating-system APIs, binary structure inspection, and
lab tooling when no suitable Python API already exists.

`ctypes` bypasses Python's normal memory-safety guarantees. A wrong type,
calling convention, structure layout, pointer, buffer size, or ownership rule
can silently corrupt memory or terminate Python. Copy the exact declaration
from the library's headers or authoritative API documentation before calling
anything.

```python
import ctypes as C
import os

# size_t strlen(const char *s);
libc = C.CDLL("ucrtbase" if os.name == "nt" else None)
strlen = libc.strlen
strlen.argtypes = (C.c_char_p,)
strlen.restype = C.c_size_t

print(strlen(b"hello"))  # 5
```

- `ctypes` ships with CPython; no package installation is normally required.
- Prefer a normal Python module, `subprocess`, or a maintained binding when one
  already models the API safely.
- Match the Python process, library, and dependency architecture: 32-bit with
  32-bit or 64-bit with 64-bit.
- Set both `argtypes` and `restype` before the first call.
- Load only trusted libraries, preferably by an expected absolute path.
- The examples perform local, benign operations. Use native security APIs only
  on systems and data covered by explicit authorization.

## Contents

- [Pick a workflow](#pick-a-workflow)
- [The ABI contract](#the-abi-contract)
- [Loading libraries](#loading-libraries)
- [Declaring and calling functions](#declaring-and-calling-functions)
- [C-compatible scalar types](#c-compatible-scalar-types)
- [Strings and mutable buffers](#strings-and-mutable-buffers)
- [Pointers and output parameters](#pointers-and-output-parameters)
- [Arrays](#arrays)
- [Structures, unions, and layout](#structures-unions-and-layout)
- [Callbacks](#callbacks)
- [Error handling](#error-handling)
- [Memory ownership and lifetime](#memory-ownership-and-lifetime)
- [Windows API patterns](#windows-api-patterns)
- [Windows process creation with CreateProcessW](#windows-process-creation)
- [Related Windows process APIs](#related-windows-process-apis)
- [POSIX patterns](#posix-patterns)
- [Binary inspection](#binary-inspection)
- [Writing a reusable wrapper](#writing-a-reusable-wrapper)
- [Testing and debugging](#testing-and-debugging)
- [Troubleshooting](#troubleshooting)
- [Security checklist](#security-checklist)
- [Quick reference](#quick-reference)

<a id="pick-a-workflow"></a>

## [Pick a workflow](#contents "Back to contents")

| Goal                                      | Starting point                                            |
| ----------------------------------------- | --------------------------------------------------------- |
| Load a normal C library                   | `C.CDLL(path)`                                            |
| Load the current process/global symbols   | `C.CDLL(None)`                                            |
| Load a Windows `stdcall` API              | `C.WinDLL(path, use_last_error=True)`                     |
| Start a process with exact Win32 semantics | Fully declare `CreateProcessW`, its structures, wait, and cleanup |
| Locate a library by a short POSIX name    | `ctypes.util.find_library(name)`                          |
| Define parameters and result              | `func.argtypes = (...)`; `func.restype = ...`             |
| Represent a C pointer                     | `C.POINTER(T)` or `C.c_void_p`                            |
| Pass an existing object by address        | `C.byref(obj)`                                            |
| Allocate writable bytes                   | `C.create_string_buffer(size)`                            |
| Allocate writable wide characters         | `C.create_unicode_buffer(size)`                           |
| Represent a fixed C array                 | `T * count`                                               |
| Represent a C `struct`                    | Subclass `C.Structure` and set `_fields_`                 |
| Represent a C `union`                     | Subclass `C.Union` and set `_fields_`                     |
| Create a C-callable Python callback       | `C.CFUNCTYPE(...)`                                        |
| Read captured POSIX `errno`               | Load with `use_errno=True`; call `C.get_errno()`          |
| Read captured Windows last-error          | Load with `use_last_error=True`; call `C.get_last_error()` |
| Convert a failed result into an exception | Assign `func.errcheck`                                    |
| Check size, alignment, or address         | `C.sizeof`, `C.alignment`, `C.addressof`                  |

The shortest safe workflow is:

```text
authoritative C declaration
        -> architecture + calling convention
        -> ctypes type mapping
        -> argtypes + restype
        -> ownership + error contract
        -> one bounded test with known input
```

<a id="the-abi-contract"></a>

## [The ABI contract](#contents "Back to contents")

The application binary interface (ABI) is the agreement between the caller and
native function. `ctypes` cannot infer most of it.

| Contract detail    | Questions to answer                                                       |
| ------------------ | ------------------------------------------------------------------------- |
| Export             | What is the exact symbol name or ordinal?                                 |
| Calling convention | `cdecl`, Windows `stdcall`, COM/HRESULT, or something unsupported?        |
| Arguments          | Exact order, signedness, width, pointer depth, and `const` status?        |
| Return             | Scalar, pointer, structure, error code, or `void`?                        |
| Layout             | Field order, padding, alignment, packing, bit-fields, and byte order?     |
| Strings            | `char*`, `wchar_t*`, encoding, NUL termination, and length units?         |
| Buffers            | Who allocates, how large, and may the callee write to them?               |
| Ownership          | Who frees returned memory, with which allocator, and when?                |
| Errors             | Sentinel return,`errno`, Windows last-error, `HRESULT`, or out parameter? |
| Lifetime           | Does native code retain any pointer or callback after the call?           |
| Threads            | Can callbacks arrive on foreign threads, and is the function thread-safe? |

Never guess a declaration from a function name or an example written for a
different architecture. Start from the exact installed header and library
version. C type widths and exported ABIs vary by operating system and compiler.

### A declaration translated deliberately

Given this C declaration:

```c
size_t strlen(const char *s);
```

translate every part:

```python
strlen.argtypes = (C.c_char_p,)  # const char *
strlen.restype = C.c_size_t      # size_t, not the default C int
```

Without `restype`, `ctypes` assumes the function returns C `int`. That can
truncate a 64-bit pointer or count. Without `argtypes`, incompatible arguments
may reach native code before Python detects the mistake.

<a id="loading-libraries"></a>

## [Loading libraries](#contents "Back to contents")

### Loader classes

| Loader     | Calling behavior                                        | Typical use                            |
| ---------- | ------------------------------------------------------- | -------------------------------------- |
| `C.CDLL`   | Standard C /`cdecl`; releases the GIL during calls      | C libraries on all supported platforms |
| `C.PyDLL`  | Standard C; keeps the GIL and checks Python error state | Python C API only                      |
| `C.WinDLL` | Windows `stdcall`; releases the GIL                     | Win32 APIs declared with `WINAPI`      |
| `C.OleDLL` | Windows `stdcall`; treats failed `HRESULT` as an error  | COM functions returning `HRESULT`      |

`C.CDLL(None)` opens the current process and its globally available symbols.
It is convenient for standard C/POSIX functions but is not a portable way to
reach every library.

```python
import ctypes as C
from ctypes.util import find_library

name = find_library("c")
if name is None:
    raise RuntimeError("C library could not be located")

libc = C.CDLL(name, use_errno=True)
```

`find_library()` is platform-dependent and can return a loader name rather than
an absolute path. For an application-owned dependency, resolve and validate the
expected packaged path instead.

### Load an application-owned library safely

```python
from pathlib import Path
import ctypes as C

base = Path(__file__).resolve().parent
library_path = (base / "native" / "example.dll").resolve()

if library_path.parent != (base / "native").resolve():
    raise RuntimeError("library path escaped its expected directory")
if not library_path.is_file():
    raise FileNotFoundError(library_path)

native = C.CDLL(str(library_path))
```

Do not construct a DLL path from untrusted input. On Windows, passing the full
path is the safest way to select the intended DLL; its dependent DLLs must also
resolve safely. Do not add attacker-writable directories to the DLL search path.

### Accessing symbols

```python
function = library.exported_name
function = getattr(library, "symbol-name-not-valid-in-python")
```

Attribute access caches the function object, which is useful after assigning a
prototype. Index access creates a fresh function object each time. Keep one
configured wrapper rather than repeatedly fetching and reconfiguring a symbol.

<a id="declaring-and-calling-functions"></a>

## [Declaring and calling functions](#contents "Back to contents")

Always configure the function before exposing it to the rest of the program:

```python
import ctypes as C
import os

libc = C.CDLL("ucrtbase" if os.name == "nt" else None)

strlen = libc.strlen
strlen.argtypes = (C.c_char_p,)
strlen.restype = C.c_size_t

length = strlen(b"operator")
```

Use `None` as the `restype` for a C function returning `void`:

```python
cleanup = native.cleanup
cleanup.argtypes = ()
cleanup.restype = None
```

Use `None` as an argument only when the prototype accepts a nullable pointer;
it is passed as C `NULL`.

### Pointer-sized results must be pointer-sized

```python
# void *lookup_handle(const char *name);
lookup_handle.argtypes = (C.c_char_p,)
lookup_handle.restype = C.c_void_p
```

Do not use `c_int` for addresses, handles, `size_t`, or `ssize_t`. On a 64-bit
process, `c_int` is normally only 32 bits.

### Variadic functions

Variadic functions such as `printf` cannot have every argument described by
one `argtypes` tuple. Declare at least the fixed arguments and explicitly wrap
each variadic value in its correct `ctypes` type. ABIs differ, particularly on
some ARM64 platforms, so avoid variadic APIs in wrappers when a fixed-signature
alternative exists.

```python
printf = libc.printf
printf.argtypes = (C.c_char_p,)  # fixed argument only
printf.restype = C.c_int

printf(b"value=%f\n", C.c_double(3.5))
```

Never let untrusted input become a `printf`-style format string.

<a id="c-compatible-scalar-types"></a>

## [C-compatible scalar types](#contents "Back to contents")

| C declaration                   | Common `ctypes` type             | Python value              |
| ------------------------------- | -------------------------------- | ------------------------- |
| `_Bool` / C `bool`              | `C.c_bool`                       | `bool`                    |
| `char`                          | `C.c_char`                       | one-byte `bytes`          |
| `signed char` / `unsigned char` | `C.c_byte` / `C.c_ubyte`         | `int`                     |
| `short` / `unsigned short`      | `C.c_short` / `C.c_ushort`       | `int`                     |
| `int` / `unsigned int`          | `C.c_int` / `C.c_uint`           | `int`                     |
| `long` / `unsigned long`        | `C.c_long` / `C.c_ulong`         | `int`                     |
| `long long` / unsigned          | `C.c_longlong` / `C.c_ulonglong` | `int`                     |
| `int8_t` ... `int64_t`          | `C.c_int8` ... `C.c_int64`       | `int`                     |
| `uint8_t` ... `uint64_t`        | `C.c_uint8` ... `C.c_uint64`     | `int`                     |
| `size_t` / `ssize_t`            | `C.c_size_t` / `C.c_ssize_t`     | `int`                     |
| `float` / `double`              | `C.c_float` / `C.c_double`       | `float`                   |
| `wchar_t`                       | `C.c_wchar`                      | one-character `str`       |
| `char *`                        | `C.c_char_p`                     | `bytes` or `None`         |
| `wchar_t *`                     | `C.c_wchar_p`                    | `str` or `None`           |
| `void *`                        | `C.c_void_p`                     | address as `int` or `None` |
| Python object pointer           | `C.py_object`                    | Python object             |

Use fixed-width types when the native declaration uses fixed-width types. Use
the platform C types when the declaration does. For example, C `long` is 32
bits on 64-bit Windows but commonly 64 bits on 64-bit Unix-like systems.

Inspect assumptions instead of memorizing them:

```python
import ctypes as C
import struct

print("pointer bits:", struct.calcsize("P") * 8)
for ctype in (C.c_int, C.c_long, C.c_void_p, C.c_size_t):
    print(ctype.__name__, C.sizeof(ctype), C.alignment(ctype))
```

Assigning an out-of-range integer to a C integer type can wrap or mask it. Check
application-level ranges before conversion when truncation would be unsafe.

<a id="strings-and-mutable-buffers"></a>

## [Strings and mutable buffers](#contents "Back to contents")

| Native parameter                          | Use                              | Important distinction                    |
| ----------------------------------------- | -------------------------------- | ---------------------------------------- |
| Read-only NUL-terminated `const char *`   | `C.c_char_p`; pass `bytes`       | Encoding comes from the API, not `ctypes` |
| Read-only NUL-terminated `const wchar_t *` | `C.c_wchar_p`; pass `str`       | `wchar_t` width is platform-dependent    |
| Writable `char *` buffer                  | `C.create_string_buffer(size)`   | Capacity is bytes                        |
| Writable `wchar_t *` buffer               | `C.create_unicode_buffer(count)` | Capacity is wide characters              |
| Binary data with explicit length          | `POINTER(c_ubyte)` or an array   | Embedded NUL bytes are valid             |

`c_char_p` and `c_wchar_p` point at string data; they do not create general
writable buffers. Reassigning `.value` changes the pointer target, not the
original string. Use an allocated buffer when native code writes.

```python
import ctypes as C

buf = C.create_string_buffer(64)
buf.value = b"hello"

print(buf.value)  # bytes up to the first NUL
print(buf.raw)    # all 64 bytes
print(C.sizeof(buf))
```

Leave room for a required terminator. When the API reports lengths, determine
whether the unit is bytes, characters, wide characters, or structure elements,
and whether the terminator is included.

```python
text = "lab-host"
wide = C.create_unicode_buffer(text, len(text) + 1)
```

### Bounded reads

```python
data = C.string_at(pointer_value, byte_count)
text = C.wstring_at(wide_pointer, character_count)
```

These functions trust both the address and length. They do not make an invalid
pointer safe. Prefer a known length and a pointer whose lifetime and readable
range are already established; an unbounded NUL search can run past valid
memory.

<a id="pointers-and-output-parameters"></a>

## [Pointers and output parameters](#contents "Back to contents")

| Operation                     | Result                                         |
| ----------------------------- | ---------------------------------------------- |
| `C.POINTER(T)`                | A pointer_type_ for `T *`                      |
| `C.pointer(obj)`              | A pointer object that owns a reference to `obj` |
| `C.byref(obj)`                | A lightweight temporary address for a call     |
| `ptr.contents`                | Object referenced by a typed pointer           |
| `ptr[index]`                  | Pointer indexing with no bounds checking       |
| `C.cast(value, pointer_type)` | Reinterpret an address as another pointer type |
| `C.addressof(obj)`            | Integer address of a `ctypes` object           |

Use `byref()` for an ordinary output parameter:

```python
import ctypes as C

# int read_count(size_t *out_count);
read_count = native.read_count
read_count.argtypes = (C.POINTER(C.c_size_t),)
read_count.restype = C.c_int

count = C.c_size_t()
status = read_count(C.byref(count))
if status != 0:
    raise RuntimeError(f"read_count failed: {status}")

print(count.value)
```

Use `pointer()` when Python must retain or manipulate the pointer object:

```python
number = C.c_int(42)
number_ptr = C.pointer(number)
assert number_ptr.contents.value == 42
```

Check a nullable returned pointer before dereferencing it:

```python
if not result_ptr:
    raise RuntimeError("native function returned NULL")
value = result_ptr.contents
```

Pointer indexing, `.contents`, `cast`, `from_address`, `string_at`, and
`memmove` perform little or no bounds validation. An address being nonzero does
not prove that it is valid, aligned, readable, writable, or still alive.

<a id="arrays"></a>

## [Arrays](#contents "Back to contents")

Multiply a `ctypes` type by a fixed count to create an array type:

```python
import ctypes as C

ByteBlock = C.c_ubyte * 8
block = ByteBlock(0x43, 0x54, 0x59, 0x50, 0x45, 0x53, 0x00, 0x01)

print(len(block))
print(bytes(block))
```

Arrays are fixed-size, mutable, and contiguous. They decay to a compatible
pointer when passed to a declared function parameter.

```python
# void zero_bytes(unsigned char *data, size_t length);
zero_bytes.argtypes = (C.POINTER(C.c_ubyte), C.c_size_t)
zero_bytes.restype = None
zero_bytes(block, len(block))
```

`ctypes` pointer indexing has no knowledge of the allocation's length. Retain
the array and its count together, and validate indices in Python.

<a id="structures-unions-and-layout"></a>

## [Structures, unions, and layout](#contents "Back to contents")

Translate fields in the exact order and type used by the C declaration:

```c
typedef struct {
    uint32_t process_id;
    uint16_t state;
    uint16_t flags;
} ProcessRecord;
```

```python
import ctypes as C

class ProcessRecord(C.Structure):
    _fields_ = [
        ("process_id", C.c_uint32),
        ("state", C.c_uint16),
        ("flags", C.c_uint16),
    ]

assert C.sizeof(ProcessRecord) == 8
assert ProcessRecord.flags.offset == 6
```

### Layout checklist

- Define `_fields_` before first use; changing it after the type is used can
  leave cached pointer types or layouts inconsistent.
- Match the compiler ABI, architecture, field order, anonymous fields, and
  nested types.
- Do not set `_pack_ = 1` merely to make an unexpected size disappear. Use
  packing only when the native declaration or format explicitly requires it.
- Compare `sizeof(struct)` and `offsetof(field)` against a tiny C fixture or
  authoritative ABI documentation.
- Native `Structure` and `Union` use native byte order. Use
  `LittleEndianStructure` or `BigEndianStructure` for a fixed byte order; those
  non-native structures cannot contain pointer fields.
- Compiler-specific bit-field allocation is not portable. Pass unions and
  structures containing bit-fields by pointer, not by value.
- Flexible array members require a deliberately sized backing allocation and a
  custom wrapper; a plain `_fields_` declaration is not enough.

### Nested structures, arrays, and unions

```python
class Coordinates(C.Structure):
    _fields_ = [("x", C.c_int32), ("y", C.c_int32)]

class Value(C.Union):
    _fields_ = [("integer", C.c_uint32), ("raw", C.c_ubyte * 4)]

class Record(C.Structure):
    _fields_ = [
        ("position", Coordinates),
        ("name", C.c_char * 16),
        ("value", Value),
    ]
```

Assigning one structure field from another can create views into the same
backing object rather than independent Python copies. Use `from_buffer_copy()`
or an explicit field-by-field copy when independent storage is required.

<a id="callbacks"></a>

## [Callbacks](#contents "Back to contents")

`CFUNCTYPE` creates a C `cdecl` callback type. On Windows, `WINFUNCTYPE` creates
a `stdcall` callback type.

```python
import ctypes as C

# int compare(const void *left, const void *right);
CompareCallback = C.CFUNCTYPE(C.c_int, C.c_void_p, C.c_void_p)

@CompareCallback
def compare_ints(left, right):
    a = C.cast(left, C.POINTER(C.c_int)).contents.value
    b = C.cast(right, C.POINTER(C.c_int)).contents.value
    return (a > b) - (a < b)

# Keep `compare_ints` referenced for at least as long as native code can call it.
```

Callback rules:

- Match the callback's calling convention, return type, and every argument.
- Keep a strong Python reference. `ctypes` does not keep the callback alive;
  garbage collection followed by a native call can crash the process.
- Also keep alive every Python/`ctypes` object whose address the callback uses.
- Do not let exceptions escape a callback. Catch them, record a safe error
  signal, and return a value permitted by the C contract.
- Keep callback work short and thread-safe. Foreign code may call from a thread
  Python did not create; `ctypes` may create a temporary Python thread state for
  each invocation.
- Do not unload the defining library while a callback or function pointer may
  still be used.

```python
@CompareCallback
def guarded_compare(left, right):
    try:
        a = C.cast(left, C.POINTER(C.c_int)).contents.value
        b = C.cast(right, C.POINTER(C.c_int)).contents.value
        return (a > b) - (a < b)
    except Exception:
        return 0  # only if zero is the documented safe failure behavior
```

Choose the failure return from the native API's callback contract; `0` is not a
universal error value.

<a id="error-handling"></a>

## [Error handling](#contents "Back to contents")

A function call completing does not mean the operation succeeded. Determine
the API's exact failure sentinel and where extended error information lives.

### POSIX `errno`

Load the library with `use_errno=True`, read the captured value immediately
after a documented failure, and format it with `os.strerror()`:

```python
import ctypes as C
import os

libc = C.CDLL(None, use_errno=True)
chdir = libc.chdir
chdir.argtypes = (C.c_char_p,)
chdir.restype = C.c_int

C.set_errno(0)
result = chdir(b"/definitely/not/a/real/path")
if result == -1:
    code = C.get_errno()
    raise OSError(code, os.strerror(code))
```

Only inspect `errno` when the function's documentation says the result denotes
failure. A stale nonzero `errno` after success is not evidence of an error.

### Windows last-error

Load a Windows library with `use_last_error=True`. Read the `ctypes` private
copy using `C.get_last_error()` and construct `C.WinError(code)`.

```python
kernel32 = C.WinDLL("kernel32", use_last_error=True)

open_item = kernel32.SomeDocumentedFunctionW
open_item.argtypes = (...,)
open_item.restype = C.c_void_p

C.set_last_error(0)
result = open_item(...)
if not result:  # only if NULL is the documented failure sentinel
    code = C.get_last_error()
    raise C.WinError(code)
```

Do not use `kernel32.GetLastError()` later through a separate call; intervening
native work can overwrite the thread's real last-error value.

### Reusable `errcheck`

`errcheck` receives `(result, function, arguments)` and can raise or transform
the return value:

```python
def require_nonzero(result, function, arguments):
    if not result:
        code = C.get_last_error()
        raise C.WinError(code)
    return result

some_win32_function.errcheck = require_nonzero
```

Write a separate checker for each error convention. `FALSE`, `NULL`, `-1`, a
negative `HRESULT`, and `INVALID_HANDLE_VALUE` are different contracts.

<a id="memory-ownership-and-lifetime"></a>

## [Memory ownership and lifetime](#contents "Back to contents")

For every pointer, document:

```text
allocated by -> valid until -> mutable by -> freed by -> free function
```

| Pointer source                                | Typical owner                   | Rule                                                                   |
| --------------------------------------------- | ------------------------------- | ---------------------------------------------------------------------- |
| `create_string_buffer`, array, structure      | Python object                   | Keep that object alive while native code uses its address              |
| `c_char_p` from Python `bytes`                | Python                          | Treat as read-only; retain the `bytes` if native code stores it        |
| Borrowed pointer returned by a library        | Library/parent object           | Do not free; obey documented invalidation rules                        |
| Newly allocated pointer returned by a library | Caller, using that library      | Call the matching library's documented release function                |
| Pointer passed only during a synchronous call | Usually caller                  | `byref()` is suitable if native code does not retain it                |
| Callback stored by native code                | Python plus native registration | Retain callback until explicitly unregistered and no call is in flight |

Never free a pointer using a different allocator or C runtime. On Windows in
particular, memory allocated by one runtime DLL may not be safely freed by
another. Configure the library's release function with its own prototype:

```python
# void release_result(void *result);
release_result = native.release_result
release_result.argtypes = (C.c_void_p,)
release_result.restype = None
```

Use `try/finally` or a small owner object so every successful acquisition has
one matching release:

```python
result = acquire_result()
if not result:
    raise RuntimeError("acquire_result returned NULL")

try:
    consume_borrowed_result(result)
finally:
    release_result(result)
```

### Raw memory helpers

| Helper                        | Behavior                          | Main risk                                         |
| ----------------------------- | --------------------------------- | ------------------------------------------------- |
| `C.memmove(dst, src, count)`  | Copies raw bytes; overlap allowed | No allocation bounds checking                     |
| `C.memset(dst, value, count)` | Fills raw bytes                   | Wrong address/count corrupts memory               |
| `C.string_at(ptr, count)`     | Copies bytes from an address      | Invalid or excessive read                         |
| `C.resize(obj, size)`         | Resizes owned memory block        | Python type's logical element count does not grow |
| `T.from_address(addr)`        | Places a live view at an address  | No validation of lifetime or access               |

Prefer indexing an owned `ctypes` array or buffer. Raw-address helpers are for
addresses and bounds already established by the native contract, not for
testing whether an arbitrary address is valid.

<a id="windows-api-patterns"></a>

## [Windows API patterns](#contents "Back to contents")

Use `ctypes.wintypes` aliases to make Win32 declarations readable, but verify
each typedef from the current Microsoft header. Handles and pointers must remain
pointer-sized.

### A no-argument Win32 call

```python
import ctypes as C
from ctypes import wintypes

kernel32 = C.WinDLL("kernel32", use_last_error=True)

GetCurrentProcessId = kernel32.GetCurrentProcessId
GetCurrentProcessId.argtypes = ()
GetCurrentProcessId.restype = wintypes.DWORD

print(GetCurrentProcessId())
```

### A writable wide-character output buffer

```python
import ctypes as C
from ctypes import wintypes

kernel32 = C.WinDLL("kernel32", use_last_error=True)

GetComputerNameW = kernel32.GetComputerNameW
GetComputerNameW.argtypes = (
    wintypes.LPWSTR,
    C.POINTER(wintypes.DWORD),
)
GetComputerNameW.restype = wintypes.BOOL

capacity = wintypes.DWORD(256)
name = C.create_unicode_buffer(capacity.value)

C.set_last_error(0)
if not GetComputerNameW(name, C.byref(capacity)):
    raise C.WinError(C.get_last_error())

print(name.value)
```

Windows API reminders:

- Select the `W` Unicode export explicitly and pass `str`/wide buffers. `ctypes`
  does not expand C preprocessor macros such as `GetComputerName` for you.
- `BOOL` is a 32-bit integer, not C++ `bool`; test its truth value.
- `HANDLE` is pointer-sized. APIs may use either `NULL` or
  `INVALID_HANDLE_VALUE` as failure; check the specific declaration.
- A pseudohandle may not be closed, while an acquired real handle usually must
  be closed with the API's matching function.
- Set `use_last_error=True` on the loader or function prototype when the API
  reports details through last-error.
- Match `WINAPI`/`CALLBACK` (`stdcall`) with `WinDLL`/`WINFUNCTYPE`; ordinary C
  runtime functions normally use `CDLL`/`CFUNCTYPE`.

<a id="windows-process-creation"></a>

## [Windows process creation with `CreateProcessW`](#contents "Back to contents")

The exact spelling is `kernel32.CreateProcessW`: `kernel32`, not `kernal32`,
and exported function names are case-sensitive. This is a useful comprehensive
Windows FFI exercise because it combines a ten-argument function, input and
output pointers, writable strings, nested platform types, last-error capture,
returned resource ownership, waiting, and cleanup.

For ordinary Python programs, prefer `subprocess`. Use `CreateProcessW`
directly when the exercise or application genuinely needs Win32 creation
semantics that the higher-level API does not expose. Creating a process executes
code in the caller's security context and is an operator-visible action, not a
read-only query.

Do not call the bare function before configuring it:

```python
# Convenient for exploration, but not a complete wrapper:
C.windll.kernel32.CreateProcessW

# Better: retain a loader configured for Windows last-error capture.
kernel32 = C.WinDLL("kernel32", use_last_error=True)
CreateProcessW = kernel32.CreateProcessW
```

The shared `C.windll.kernel32` loader uses the correct Windows calling
convention, but it does not enable `use_last_error=True`, and a newly fetched
function still needs exact `argtypes` and `restype`. A named `WinDLL` instance
makes the error and prototype contract explicit.

### Native declaration and all ten parameters

```c
BOOL CreateProcessW(
    LPCWSTR               lpApplicationName,
    LPWSTR                lpCommandLine,
    LPSECURITY_ATTRIBUTES lpProcessAttributes,
    LPSECURITY_ATTRIBUTES lpThreadAttributes,
    BOOL                  bInheritHandles,
    DWORD                 dwCreationFlags,
    LPVOID                lpEnvironment,
    LPCWSTR               lpCurrentDirectory,
    LPSTARTUPINFOW        lpStartupInfo,
    LPPROCESS_INFORMATION lpProcessInformation
);
```

| Parameter              | Direction        | Purpose and safe default                                                                                 |
| ---------------------- | ---------------- | -------------------------------------------------------------------------------------------------------- |
| `lpApplicationName`    | In, optional     | Explicit executable path. Prefer a validated absolute path rather than `NULL` search behavior.           |
| `lpCommandLine`        | In/out, optional | Writable, NUL-terminated command line. `CreateProcessW` may modify it, so use `create_unicode_buffer()`. |
| `lpProcessAttributes`  | In, optional     | Process-handle security/inheritability. Use `None` for default, non-inheritable behavior.                |
| `lpThreadAttributes`   | In, optional     | Primary-thread-handle security/inheritability. Use `None` for default, non-inheritable behavior.         |
| `bInheritHandles`      | In               | Whether inheritable handles are inherited. Use `False` unless inheritance is intentionally configured.   |
| `dwCreationFlags`      | In               | Process creation, priority, console, and environment flags. Use `0` unless each flag is understood.      |
| `lpEnvironment`        | In, optional     | Environment block. `None` inherits the caller's environment. Keep a custom block alive through the call. |
| `lpCurrentDirectory`   | In, optional     | Child working directory. Use an existing absolute directory or `None` to inherit.                        |
| `lpStartupInfo`        | In               | Initialized `STARTUPINFOW`; its `cb` field must equal `sizeof(STARTUPINFOW)`.                            |
| `lpProcessInformation` | Out              | Receives process/thread handles and IDs. Close both handles exactly once.                                |

If `lpApplicationName` is `None`, Windows parses the executable from the first
command-line token. Ambiguous unquoted paths can select a different executable.
Supplying the validated executable path separately avoids that search ambiguity;
still repeat it as argument zero in `lpCommandLine` for conventional `argv[0]`.

### Required structures

These definitions mirror the Windows SDK declarations and retain pointer-sized
`HANDLE` fields:

```python
import ctypes as C
from ctypes import wintypes


class SECURITY_ATTRIBUTES(C.Structure):
    _fields_ = [
        ("nLength", wintypes.DWORD),
        ("lpSecurityDescriptor", wintypes.LPVOID),
        ("bInheritHandle", wintypes.BOOL),
    ]


class STARTUPINFOW(C.Structure):
    _fields_ = [
        ("cb", wintypes.DWORD),
        ("lpReserved", wintypes.LPWSTR),
        ("lpDesktop", wintypes.LPWSTR),
        ("lpTitle", wintypes.LPWSTR),
        ("dwX", wintypes.DWORD),
        ("dwY", wintypes.DWORD),
        ("dwXSize", wintypes.DWORD),
        ("dwYSize", wintypes.DWORD),
        ("dwXCountChars", wintypes.DWORD),
        ("dwYCountChars", wintypes.DWORD),
        ("dwFillAttribute", wintypes.DWORD),
        ("dwFlags", wintypes.DWORD),
        ("wShowWindow", wintypes.WORD),
        ("cbReserved2", wintypes.WORD),
        ("lpReserved2", C.POINTER(wintypes.BYTE)),
        ("hStdInput", wintypes.HANDLE),
        ("hStdOutput", wintypes.HANDLE),
        ("hStdError", wintypes.HANDLE),
    ]


class PROCESS_INFORMATION(C.Structure):
    _fields_ = [
        ("hProcess", wintypes.HANDLE),
        ("hThread", wintypes.HANDLE),
        ("dwProcessId", wintypes.DWORD),
        ("dwThreadId", wintypes.DWORD),
    ]
```

Initialize `STARTUPINFOW` from zeros and set its size. The other reserved
members remain zero/`NULL` unless a documented feature requires them:

```python
startup = STARTUPINFOW()
startup.cb = C.sizeof(STARTUPINFOW)
process_info = PROCESS_INFORMATION()
```

On the common Windows ABIs, verify rather than assume the layouts:

```python
import struct

if struct.calcsize("P") == 8:
    assert C.sizeof(SECURITY_ATTRIBUTES) == 24
    assert C.sizeof(STARTUPINFOW) == 104
    assert C.sizeof(PROCESS_INFORMATION) == 24
else:
    assert C.sizeof(SECURITY_ATTRIBUTES) == 12
    assert C.sizeof(STARTUPINFOW) == 68
    assert C.sizeof(PROCESS_INFORMATION) == 16
```

### Complete function prototypes

```python
LPSECURITY_ATTRIBUTES = C.POINTER(SECURITY_ATTRIBUTES)

kernel32 = C.WinDLL("kernel32", use_last_error=True)

CreateProcessW = kernel32.CreateProcessW
CreateProcessW.argtypes = (
    wintypes.LPCWSTR,  # lpApplicationName
    wintypes.LPWSTR,  # lpCommandLine: writable
    LPSECURITY_ATTRIBUTES,
    LPSECURITY_ATTRIBUTES,
    wintypes.BOOL,
    wintypes.DWORD,
    wintypes.LPVOID,
    wintypes.LPCWSTR,
    C.POINTER(STARTUPINFOW),
    C.POINTER(PROCESS_INFORMATION),
)
CreateProcessW.restype = wintypes.BOOL

CloseHandle = kernel32.CloseHandle
CloseHandle.argtypes = (wintypes.HANDLE,)
CloseHandle.restype = wintypes.BOOL

WaitForSingleObject = kernel32.WaitForSingleObject
WaitForSingleObject.argtypes = (wintypes.HANDLE, wintypes.DWORD)
WaitForSingleObject.restype = wintypes.DWORD

GetExitCodeProcess = kernel32.GetExitCodeProcess
GetExitCodeProcess.argtypes = (
    wintypes.HANDLE,
    C.POINTER(wintypes.DWORD),
)
GetExitCodeProcess.restype = wintypes.BOOL
```

`CreateProcessW` returns nonzero once the process and primary thread objects are
created. It returns before the child finishes initialization, so this alone is
not proof that the child completed its work or even initialized successfully.

### A complete bounded local example

This example starts the current Python interpreter by an explicit path, waits
up to ten seconds, obtains its exit code, and closes both returned handles. It
does not invoke a shell.

```python
from pathlib import Path
import subprocess
import sys

WAIT_OBJECT_0 = 0x00000000
WAIT_TIMEOUT = 0x00000102
WAIT_FAILED = 0xFFFFFFFF


def create_and_wait(
    application: Path,
    arguments: list[str],
    *,
    current_directory: Path | None = None,
    timeout_ms: int = 10_000,
) -> tuple[int, int, int]:
    executable = application.resolve(strict=True)
    if not executable.is_file():
        raise ValueError("application is not a file")
    if not 0 <= timeout_ms <= 0xFFFFFFFE:
        raise ValueError("timeout is outside the DWORD range")
    if any("\0" in argument for argument in arguments):
        raise ValueError("arguments cannot contain NUL characters")

    working_directory: str | None = None
    if current_directory is not None:
        resolved_directory = current_directory.resolve(strict=True)
        if not resolved_directory.is_dir():
            raise ValueError("current_directory is not a directory")
        working_directory = str(resolved_directory)

    # list2cmdline follows the Microsoft C runtime argv quoting rules. A target
    # with a custom command-line parser may require its own documented encoding.
    command_line = subprocess.list2cmdline([str(executable), *arguments])
    if len(command_line) + 1 > 32_767:
        raise ValueError("command line exceeds the CreateProcessW limit")
    command_buffer = C.create_unicode_buffer(command_line)

    startup = STARTUPINFOW()
    startup.cb = C.sizeof(STARTUPINFOW)
    process_info = PROCESS_INFORMATION()

    C.set_last_error(0)
    created = CreateProcessW(
        str(executable),
        command_buffer,
        None,
        None,
        False,
        0,
        None,
        working_directory,
        C.byref(startup),
        C.byref(process_info),
    )
    if not created:
        raise C.WinError(C.get_last_error())

    primary_error: BaseException | None = None
    try:
        wait_result = WaitForSingleObject(process_info.hProcess, timeout_ms)
        if wait_result == WAIT_TIMEOUT:
            raise TimeoutError(
                f"process {process_info.dwProcessId} is still running"
            )
        if wait_result == WAIT_FAILED:
            raise C.WinError(C.get_last_error())
        if wait_result != WAIT_OBJECT_0:
            raise RuntimeError(f"unexpected wait result: 0x{wait_result:08x}")

        exit_code = wintypes.DWORD()
        if not GetExitCodeProcess(process_info.hProcess, C.byref(exit_code)):
            raise C.WinError(C.get_last_error())

        return (
            int(process_info.dwProcessId),
            int(process_info.dwThreadId),
            int(exit_code.value),
        )
    except BaseException as exc:
        primary_error = exc
        raise
    finally:
        cleanup_errors: list[OSError] = []
        for handle in (process_info.hThread, process_info.hProcess):
            if handle and not CloseHandle(handle):
                cleanup_errors.append(C.WinError(C.get_last_error()))
        if cleanup_errors and primary_error is None:
            raise cleanup_errors[0]


python = Path(sys.executable)
pid, thread_id, exit_code = create_and_wait(
    python,
    ["-c", "print('ctypes child completed')"],
    current_directory=Path.cwd(),
)
print(pid, thread_id, exit_code)
```

A wait timeout does **not** terminate the child. The example closes its handles
and reports that the identified process remains running. Do not add automatic
termination merely as cleanup: termination is a separate, disruptive decision
that can corrupt the child's state. Design cancellation explicitly for the
authorized use case.

### Command-line construction

Windows passes one command-line string to the child; the child decides how to
split it into arguments. Keep these rules visible:

- Prefer a validated absolute `lpApplicationName`.
- Put the executable path in `lpCommandLine` as conventional argument zero.
- Pass a writable `create_unicode_buffer`, never `c_wchar_p` backed by a Python
  string literal, because `CreateProcessW` may modify the command line.
- Reject embedded NUL characters and enforce the 32,767-character Unicode limit.
- `subprocess.list2cmdline()` implements Microsoft C runtime parsing rules, not
  a universal quoting language for every Windows program.
- Do not prepend `cmd.exe /c` unless shell syntax or a batch file is explicitly
  required. Shell metacharacters then become a separate injection boundary.
- Never concatenate untrusted executable paths or arguments into a shell
  command. Validate the executable identity and keep arguments structured.

### Custom Unicode environment block

Pass `None` to inherit the caller's environment. A custom Unicode environment
is a sorted sequence of `name=value` strings, each NUL-terminated, followed by
one additional NUL. Keep the buffer referenced until `CreateProcessW` returns
and include `CREATE_UNICODE_ENVIRONMENT`.

```python
from collections.abc import Mapping

CREATE_UNICODE_ENVIRONMENT = 0x00000400


def make_environment_block(values: Mapping[str, str]) -> C.Array[C.c_wchar]:
    rows: list[str] = []
    seen: set[str] = set()

    for name, value in values.items():
        if (
            not name
            or not name.isascii()
            or "=" in name
            or "\0" in name
            or "\0" in value
        ):
            raise ValueError("invalid environment name or value")
        folded = name.casefold()
        if folded in seen:
            raise ValueError("duplicate case-insensitive environment name")
        seen.add(folded)
        rows.append(f"{name}={value}")

    # This compact helper deliberately accepts ASCII names, for which an
    # uppercase key gives a stable case-insensitive ordinal ordering.
    rows.sort(key=lambda row: row.partition("=")[0].upper())
    payload = "\0".join(rows) + "\0"
    return C.create_unicode_buffer(payload, len(payload) + 1)


environment = make_environment_block(
    {
        "LAB_MODE": "1",
        "PATH": r"C:\Windows\System32",
    }
)

created = CreateProcessW(
    str(executable),
    command_buffer,
    None,
    None,
    False,
    CREATE_UNICODE_ENVIRONMENT,
    C.cast(environment, wintypes.LPVOID),
    working_directory,
    C.byref(startup),
    C.byref(process_info),
)
```

An empty custom environment block needs deliberate handling; inheriting with
`None` is normally clearer. If the target depends on ordinary Windows variables,
build from a reviewed copy of `os.environ` rather than accidentally replacing
everything with two entries. Environment values can contain secrets, so do not
log the block.

### Handle inheritance and standard I/O

The safe default is `bInheritHandles=False` with zeroed standard-handle fields.
For intentional redirection:

1. Create the input/output/error handles with the precise access and sharing
   required.
2. Make only the child-facing handles inheritable.
3. Set `startup.dwFlags |= STARTF_USESTDHANDLES` (`0x00000100`).
4. Fill all three of `hStdInput`, `hStdOutput`, and `hStdError` with valid
   handles.
5. Pass `bInheritHandles=True`.
6. Close the parent's unneeded copies promptly and close every remaining handle
   with its documented release function.

Broad inheritance is dangerous in a multithreaded process because unrelated
inheritable handles may leak into the child. For precise modern inheritance,
use `STARTUPINFOEXW` with `PROC_THREAD_ATTRIBUTE_HANDLE_LIST`; that requires a
separate wrapper for attribute-list sizing, initialization, update, lifetime,
and deletion. Do not claim safe redirection from only setting
`STARTF_USESTDHANDLES`.

### Common creation flags

| Flag                           |        Value | Meaning and caution                                                                                                              |
| ------------------------------ | -----------: | -------------------------------------------------------------------------------------------------------------------------------- |
| `CREATE_SUSPENDED`             | `0x00000004` | Primary thread starts suspended. It must later be resumed or the process remains suspended; do not use as an injection shortcut. |
| `DETACHED_PROCESS`             | `0x00000008` | Console process does not inherit the caller's console; incompatible with `CREATE_NEW_CONSOLE`.                                   |
| `CREATE_NEW_CONSOLE`           | `0x00000010` | Gives the child a new console; ignored for GUI applications.                                                                     |
| `CREATE_NEW_PROCESS_GROUP`     | `0x00000200` | Creates a new console process group; affects console control handling.                                                           |
| `CREATE_UNICODE_ENVIRONMENT`   | `0x00000400` | Declares that `lpEnvironment` points to a Unicode block.                                                                         |
| `EXTENDED_STARTUPINFO_PRESENT` | `0x00080000` | `lpStartupInfo` points to `STARTUPINFOEXW`; requires a valid attribute list.                                                     |
| `CREATE_NO_WINDOW`             | `0x08000000` | Console application runs without a console window; ignored with new-console/detached modes.                                      |

Creation flags can be combined only when their contracts are compatible. Do
not copy concealment, suspended-process, debug, priority, or breakaway flags
without understanding their operational and authorization effects.

### Result, wait, exit, and cleanup checklist

```text
CreateProcessW == 0   -> immediately capture ctypes.get_last_error()
CreateProcessW != 0   -> process + primary-thread handles are now owned
WaitForSingleObject   -> WAIT_OBJECT_0, WAIT_TIMEOUT, WAIT_FAILED, or other
GetExitCodeProcess    -> meaningful after a signaled process wait
CloseHandle           -> once for hThread and once for hProcess
```

- Closing a process or thread handle does not terminate the object.
- A process ID is not a handle and can be reused after the process object is
  released; do not treat a PID as durable identity.
- `STILL_ACTIVE` (`259`) means a queried process has not terminated, but a
  program can also choose 259 as its actual exit code. Wait for the process
  handle to become signaled before interpreting the final exit code.
- If standard handles, pipes, tokens, jobs, or attribute lists were created,
  they have separate cleanup requirements beyond the two returned handles.
- Preserve the primary failure if cleanup also fails; do not silently lose
  either event in a production wrapper.

### Related process-start APIs are not interchangeable

| API                        | Use when                                                                   | Important boundary                                                                                      |
| -------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `subprocess.Popen` / `run` | Ordinary Python process creation                                           | Safer structured interface; still validate executable, arguments, environment, handles, and shell use.  |
| `CreateProcessW`           | Exact Win32 creation/handle/flag behavior in the caller's security context | Full prototype, writable command line, handles, and cleanup remain caller-owned.                        |
| `CreateProcessAsUserW`     | Starting with a primary token under its documented privilege/session rules | Token rights, profiles, environment, desktops, and privileges need a dedicated review.                  |
| `CreateProcessWithTokenW`  | Starting through an existing token under that API's rules                  | Not a drop-in substitute; token and caller privileges govern eligibility.                               |
| `CreateProcessWithLogonW`  | Starting from supplied logon credentials                                   | Credentials are sensitive input and must never be embedded in source, command lines, logs, or evidence. |
| `ShellExecuteExW`          | Shell verbs, associations, or document launching                           | Shell resolution and verb behavior differ from direct executable creation.                              |

For authorized lab exercises involving tokens, credentials, another session,
parent-process attributes, suspended starts, or redirected handles, treat each
as its own API contract. Do not infer that a working `CreateProcessW` wrapper
makes the other variants correct or authorized.

<a id="related-windows-process-apis"></a>

## [Related Windows process APIs](#contents "Back to contents")

`CreateProcessW` is only the centre of the process-management family. Real
wrappers commonly also need pipe redirection, a restricted inheritance list,
token/profile/environment handling, shell activation, suspended-start control,
or a job object. Each facility adds handles, buffers, privileges, and cleanup
that must be modelled explicitly.

### Choose the smallest process family

| Need | Primary API family | Additional contracts |
|---|---|---|
| Start an executable as the caller | `subprocess` or `CreateProcessW` | Command line, environment, current directory, process/thread handles |
| Capture standard output/error | `CreatePipe`, `SetHandleInformation`, `ReadFile` | Inheritable child ends, non-inheritable parent ends, draining, EOF, deadlock avoidance |
| Inherit only selected handles | `STARTUPINFOEXW` plus `PROC_THREAD_ATTRIBUTE_HANDLE_LIST` | Opaque list sizing, value lifetime, `EXTENDED_STARTUPINFO_PRESENT`, deletion |
| Start from a primary token | `CreateProcessAsUserW` | Token rights, caller privileges, session/desktop, profile, user environment |
| Start through an existing token | `CreateProcessWithTokenW` | `SeImpersonatePrivilege`, logon flags, caller session behavior |
| Start from explicit credentials | `CreateProcessWithLogonW` | Credential secrecy, logon rights, profile choice; no source/log storage |
| Open a document or invoke a shell verb | `ShellExecuteExW` | Shell associations, COM apartment, optional process handle, possible UI |
| Configure before first instruction | `CREATE_SUSPENDED` then `ResumeThread` | Every failure path must resume or deliberately dispose of the suspended child |
| Control a child tree as a unit | Job-object APIs | Assignment, nested-job rules, limits, and potentially destructive close behavior |

The sections below assume the `SECURITY_ATTRIBUTES`, `STARTUPINFOW`,
`PROCESS_INFORMATION`, `CreateProcessW`, `CloseHandle`,
`WaitForSingleObject`, and `GetExitCodeProcess` definitions from the preceding
chapter.

### Redirected standard I/O with anonymous pipes

The required APIs are:

```python
CreatePipe = kernel32.CreatePipe
CreatePipe.argtypes = (
    C.POINTER(wintypes.HANDLE),
    C.POINTER(wintypes.HANDLE),
    C.POINTER(SECURITY_ATTRIBUTES),
    wintypes.DWORD,
)
CreatePipe.restype = wintypes.BOOL

SetHandleInformation = kernel32.SetHandleInformation
SetHandleInformation.argtypes = (
    wintypes.HANDLE,
    wintypes.DWORD,
    wintypes.DWORD,
)
SetHandleInformation.restype = wintypes.BOOL

ReadFile = kernel32.ReadFile
ReadFile.argtypes = (
    wintypes.HANDLE,
    wintypes.LPVOID,
    wintypes.DWORD,
    C.POINTER(wintypes.DWORD),
    wintypes.LPVOID,  # LPOVERLAPPED; None for this synchronous example
)
ReadFile.restype = wintypes.BOOL

LPCVOID = getattr(wintypes, "LPCVOID", C.c_void_p)

WriteFile = kernel32.WriteFile
WriteFile.argtypes = (
    wintypes.HANDLE,
    LPCVOID,
    wintypes.DWORD,
    C.POINTER(wintypes.DWORD),
    wintypes.LPVOID,
)
WriteFile.restype = wintypes.BOOL
```

Some Python releases do not define `wintypes.LPCVOID`; the fallback preserves
the pointer ABI. The const qualifier remains a caller rule.

```python
HANDLE_FLAG_INHERIT = 0x00000001
STARTF_USESTDHANDLES = 0x00000100
ERROR_BROKEN_PIPE = 109
```

The ownership graph matters more than the pipe names:

```text
parent reads  <- output_read | output_write -> child stdout + stderr
parent writes -> input_write | input_read   -> child stdin

parent keeps: output_read, input_write
child gets:   output_write, input_read
```

Create each pipe with inheritable handles, then clear inheritance on the two
parent ends. After process creation, close the parent's copies of both child
ends immediately. EOF is not observed while any copy of the corresponding
write handle remains open.

```python
def create_pipe_pair() -> tuple[int, int]:
    attributes = SECURITY_ATTRIBUTES()
    attributes.nLength = C.sizeof(SECURITY_ATTRIBUTES)
    attributes.bInheritHandle = True

    read_handle = wintypes.HANDLE()
    write_handle = wintypes.HANDLE()
    if not CreatePipe(
        C.byref(read_handle),
        C.byref(write_handle),
        C.byref(attributes),
        0,
    ):
        raise C.WinError(C.get_last_error())
    return int(read_handle.value), int(write_handle.value)


def make_noninheritable(handle: int) -> None:
    if not SetHandleInformation(handle, HANDLE_FLAG_INHERIT, 0):
        raise C.WinError(C.get_last_error())
```

Configure all three standard handles when `STARTF_USESTDHANDLES` is set:

```python
output_read = output_write = input_read = input_write = 0
try:
    output_read, output_write = create_pipe_pair()
    input_read, input_write = create_pipe_pair()
    make_noninheritable(output_read)
    make_noninheritable(input_write)

    startup = STARTUPINFOW()
    startup.cb = C.sizeof(STARTUPINFOW)
    startup.dwFlags = STARTF_USESTDHANDLES
    startup.hStdInput = input_read
    startup.hStdOutput = output_write
    startup.hStdError = output_write

    # Build the validated executable and writable command_buffer exactly as in
    # the CreateProcessW example.
    process_info = PROCESS_INFORMATION()
    if not CreateProcessW(
        str(executable),
        command_buffer,
        None,
        None,
        True,
        0,
        None,
        working_directory,
        C.byref(startup),
        C.byref(process_info),
    ):
        raise C.WinError(C.get_last_error())
except BaseException:
    # Preserve the primary error while making a best effort to release every
    # pipe handle acquired before creation completed.
    for handle in (input_read, input_write, output_read, output_write):
        if handle:
            CloseHandle(handle)
    raise
```

After a successful create, the parent no longer needs `input_read` or
`output_write`. If it has no input to send, it should also close `input_write`
so the child can observe end-of-input.

```python
for child_end in (input_read, output_write):
    if not CloseHandle(child_end):
        raise C.WinError(C.get_last_error())
input_read = 0
output_write = 0

if not CloseHandle(input_write):
    raise C.WinError(C.get_last_error())
input_write = 0
```

Drain the output while the child runs. Waiting first can deadlock when the
child fills the finite pipe buffer and blocks waiting for the parent to read.

```python
chunks: list[bytes] = []
buffer = C.create_string_buffer(4096)
bytes_read = wintypes.DWORD()

while True:
    C.set_last_error(0)
    ok = ReadFile(
        output_read,
        buffer,
        C.sizeof(buffer),
        C.byref(bytes_read),
        None,
    )
    if ok:
        if bytes_read.value:
            chunks.append(buffer.raw[: bytes_read.value])
        continue

    code = C.get_last_error()
    if code == ERROR_BROKEN_PIPE:
        break
    raise C.WinError(code)

output = b"".join(chunks)
```

Then wait for the process, obtain its exit code, and close `output_read`,
`hThread`, and `hProcess` exactly once. Decode only with the child's documented
encoding. To keep stdout and stderr distinct, create two output pipes and drain
both concurrently. For arbitrary or long-running children, use dedicated reader
threads or correctly modelled overlapped I/O and enforce total output limits;
a synchronous single-pipe loop is only a bounded teaching pattern.

Pipe failure checklist:

| Symptom | Check |
|---|---|
| Parent never sees EOF | A parent or child copy of the pipe's write end is still open |
| Child cannot read stdin | `hStdInput`, inheritance, or the parent write end is wrong/closed early |
| `ReadFile` blocks forever | Child still owns a write handle or is waiting for parent input |
| Child output stalls | Parent waited without concurrently draining the pipe |
| Unrelated handle leaked | `bInheritHandles=True` inherited more than intended; use a handle list |
| Binary output is corrupt | Text decoding was applied before the complete protocol boundary |

### Precise inheritance with `STARTUPINFOEXW`

`bInheritHandles=True` normally exposes every inheritable handle in the parent.
`PROC_THREAD_ATTRIBUTE_HANDLE_LIST` restricts creation to a declared list. The
list is an opaque, size-queried allocation whose values must remain alive until
the list is deleted.

```python
class STARTUPINFOEXW(C.Structure):
    _fields_ = [
        ("StartupInfo", STARTUPINFOW),
        ("lpAttributeList", wintypes.LPVOID),
    ]


InitializeProcThreadAttributeList = (
    kernel32.InitializeProcThreadAttributeList
)
InitializeProcThreadAttributeList.argtypes = (
    wintypes.LPVOID,
    wintypes.DWORD,
    wintypes.DWORD,
    C.POINTER(C.c_size_t),
)
InitializeProcThreadAttributeList.restype = wintypes.BOOL

UpdateProcThreadAttribute = kernel32.UpdateProcThreadAttribute
UpdateProcThreadAttribute.argtypes = (
    wintypes.LPVOID,
    wintypes.DWORD,
    C.c_size_t,  # DWORD_PTR
    wintypes.LPVOID,
    C.c_size_t,
    wintypes.LPVOID,
    C.POINTER(C.c_size_t),
)
UpdateProcThreadAttribute.restype = wintypes.BOOL

DeleteProcThreadAttributeList = kernel32.DeleteProcThreadAttributeList
DeleteProcThreadAttributeList.argtypes = (wintypes.LPVOID,)
DeleteProcThreadAttributeList.restype = None
```

Common constants:

```python
ERROR_INSUFFICIENT_BUFFER = 122
EXTENDED_STARTUPINFO_PRESENT = 0x00080000
PROC_THREAD_ATTRIBUTE_HANDLE_LIST = 0x00020002
```

The first initialization call fails by design and reports the required byte
count. Check that its error is the expected insufficient-buffer result rather
than ignoring every failure.

```python
def build_handle_attribute_list(
    inherited_handles: list[int],
) -> tuple[C.Array, C.Array, int]:
    if not inherited_handles:
        raise ValueError("at least one inherited handle is required")

    size = C.c_size_t()
    C.set_last_error(0)
    first_result = InitializeProcThreadAttributeList(
        None,
        1,
        0,
        C.byref(size),
    )
    if first_result or C.get_last_error() != ERROR_INSUFFICIENT_BUFFER:
        raise C.WinError(C.get_last_error())

    # Pointer-sized elements give the opaque storage pointer alignment while
    # rounding the allocation up to the requested byte count.
    word_count = (size.value + C.sizeof(C.c_void_p) - 1) // C.sizeof(
        C.c_void_p
    )
    storage = (C.c_void_p * word_count)()
    attribute_list = C.cast(storage, wintypes.LPVOID)

    if not InitializeProcThreadAttributeList(
        attribute_list,
        1,
        0,
        C.byref(size),
    ):
        raise C.WinError(C.get_last_error())

    HandleArray = wintypes.HANDLE * len(inherited_handles)
    values = HandleArray(*inherited_handles)

    if not UpdateProcThreadAttribute(
        attribute_list,
        0,
        PROC_THREAD_ATTRIBUTE_HANDLE_LIST,
        C.cast(values, wintypes.LPVOID),
        C.sizeof(values),
        None,
        None,
    ):
        DeleteProcThreadAttributeList(attribute_list)
        raise C.WinError(C.get_last_error())

    # `storage` owns the opaque list and `values` owns the handle array. Keep
    # both alive until after CreateProcessW returns and the list is deleted.
    return storage, values, int(size.value)
```

Use the result with an extended startup structure:

```python
storage, handle_values, _ = build_handle_attribute_list(
    [input_read, output_write]
)
attribute_list = C.cast(storage, wintypes.LPVOID)

startup_ex = STARTUPINFOEXW()
startup_ex.StartupInfo.cb = C.sizeof(STARTUPINFOEXW)
startup_ex.StartupInfo.dwFlags = STARTF_USESTDHANDLES
startup_ex.StartupInfo.hStdInput = input_read
startup_ex.StartupInfo.hStdOutput = output_write
startup_ex.StartupInfo.hStdError = output_write
startup_ex.lpAttributeList = attribute_list

try:
    created = CreateProcessW(
        str(executable),
        command_buffer,
        None,
        None,
        True,  # required for PROC_THREAD_ATTRIBUTE_HANDLE_LIST
        EXTENDED_STARTUPINFO_PRESENT,
        None,
        working_directory,
        C.cast(C.byref(startup_ex), C.POINTER(STARTUPINFOW)),
        C.byref(process_info),
    )
    if not created:
        raise C.WinError(C.get_last_error())
finally:
    DeleteProcThreadAttributeList(attribute_list)
```

Every handle in the list must itself be inheritable and must not be a pseudo
handle such as `GetCurrentProcess()`. The `storage`, `handle_values`, and any
other attribute value must remain alive until deletion. `DeleteProcThreadAttributeList`
releases the list's internal state but does not close handles or free separately
owned values.

Other attribute keys include parent-process selection, mitigation policy, job
lists, and security capabilities. They have different access checks, operating-
system support, value types, and security effects. Do not reuse the handle-list
value shape or constant for another attribute, and do not use parent selection
to misrepresent process provenance.

### Tokens and alternate-security-context creation

An access token is a security authority, not merely an integer handle. Request
only the rights required by the next documented operation, verify the token
type and session, retain its provenance, and close every real token handle.

Core token declarations:

```python
advapi32 = C.WinDLL("advapi32", use_last_error=True)

GetCurrentProcess = kernel32.GetCurrentProcess
GetCurrentProcess.argtypes = ()
GetCurrentProcess.restype = wintypes.HANDLE

OpenProcessToken = advapi32.OpenProcessToken
OpenProcessToken.argtypes = (
    wintypes.HANDLE,
    wintypes.DWORD,
    C.POINTER(wintypes.HANDLE),
)
OpenProcessToken.restype = wintypes.BOOL

DuplicateTokenEx = advapi32.DuplicateTokenEx
DuplicateTokenEx.argtypes = (
    wintypes.HANDLE,
    wintypes.DWORD,
    C.POINTER(SECURITY_ATTRIBUTES),
    C.c_int,  # SECURITY_IMPERSONATION_LEVEL
    C.c_int,  # TOKEN_TYPE
    C.POINTER(wintypes.HANDLE),
)
DuplicateTokenEx.restype = wintypes.BOOL

LogonUserW = advapi32.LogonUserW
LogonUserW.argtypes = (
    wintypes.LPCWSTR,
    wintypes.LPCWSTR,
    wintypes.LPCWSTR,
    wintypes.DWORD,
    wintypes.DWORD,
    C.POINTER(wintypes.HANDLE),
)
LogonUserW.restype = wintypes.BOOL
```

Useful minimum-right constants:

```python
TOKEN_ASSIGN_PRIMARY = 0x0001
TOKEN_DUPLICATE = 0x0002
TOKEN_IMPERSONATE = 0x0004
TOKEN_QUERY = 0x0008
TOKEN_ADJUST_PRIVILEGES = 0x0020

SecurityImpersonation = 2
TokenPrimary = 1
```

Do not default to `TOKEN_ALL_ACCESS`. A primary-token preparation pattern for
an already authorized, already open token is:

```python
desired = TOKEN_QUERY | TOKEN_DUPLICATE | TOKEN_ASSIGN_PRIMARY
primary_token = wintypes.HANDLE()

if not DuplicateTokenEx(
    source_token,
    desired,
    None,
    SecurityImpersonation,
    TokenPrimary,
    C.byref(primary_token),
):
    raise C.WinError(C.get_last_error())

try:
    # Use primary_token only with the separately eligible creation API.
    pass
finally:
    if primary_token and not CloseHandle(primary_token):
        raise C.WinError(C.get_last_error())
```

Replace the `pass` with one bounded operation in real code; it marks the token's
ownership scope, not a runnable action. `GetCurrentProcess()` returns a pseudo
handle and must not be closed. `OpenProcessToken()` and `DuplicateTokenEx()`
return real handles that must be closed.

`LogonUserW` introduces a plaintext credential boundary. Python strings and
temporary Unicode conversions cannot guarantee complete memory erasure. Never
hardcode, log, persist, echo, place on a command line, or put real passwords in
the notes. Prefer a platform credential broker or protected interactive input,
minimize lifetime, and close the resulting token.

### `CreateProcessAsUserW`

This variant takes a primary token and otherwise closely follows
`CreateProcessW`:

```python
CreateProcessAsUserW = advapi32.CreateProcessAsUserW
CreateProcessAsUserW.argtypes = (
    wintypes.HANDLE,
    wintypes.LPCWSTR,
    wintypes.LPWSTR,
    C.POINTER(SECURITY_ATTRIBUTES),
    C.POINTER(SECURITY_ATTRIBUTES),
    wintypes.BOOL,
    wintypes.DWORD,
    wintypes.LPVOID,
    wintypes.LPCWSTR,
    C.POINTER(STARTUPINFOW),
    C.POINTER(PROCESS_INFORMATION),
)
CreateProcessAsUserW.restype = wintypes.BOOL
```

The token must have `TOKEN_QUERY`, `TOKEN_DUPLICATE`, and
`TOKEN_ASSIGN_PRIMARY`. The caller typically needs `SeIncreaseQuotaPrivilege`
and may need `SeAssignPrimaryTokenPrivilege`; Windows may temporarily enable
eligible privileges for the call, but it cannot grant a privilege the caller
does not possess. `ERROR_PRIVILEGE_NOT_HELD` is a policy result, not a signal to
attempt unrelated bypasses.

`CreateProcessAsUserW` uses the session in the token, does not automatically
load that user's registry profile, and does not automatically build that user's
environment. An interactive desktop additionally requires the correct window-
station/desktop access; setting `lpDesktop = "winsta0\\default"` alone does not
grant its DACL permissions.

Call shape after all prerequisites are deliberately prepared:

```python
process_info = PROCESS_INFORMATION()
created = CreateProcessAsUserW(
    primary_token,
    str(executable),
    command_buffer,
    None,
    None,
    False,
    CREATE_UNICODE_ENVIRONMENT,
    environment_pointer,
    working_directory,
    C.byref(startup),
    C.byref(process_info),
)
if not created:
    raise C.WinError(C.get_last_error())
```

On success, apply the same wait, exit-code, and two-handle cleanup contract as
`CreateProcessW`.

### User profile and token-specific environment

`CreateEnvironmentBlock` returns memory owned by `Userenv.dll`, not by Python:

```python
userenv = C.WinDLL("userenv", use_last_error=True)

CreateEnvironmentBlock = userenv.CreateEnvironmentBlock
CreateEnvironmentBlock.argtypes = (
    C.POINTER(wintypes.LPVOID),
    wintypes.HANDLE,
    wintypes.BOOL,
)
CreateEnvironmentBlock.restype = wintypes.BOOL

DestroyEnvironmentBlock = userenv.DestroyEnvironmentBlock
DestroyEnvironmentBlock.argtypes = (wintypes.LPVOID,)
DestroyEnvironmentBlock.restype = wintypes.BOOL
```

```python
environment_pointer = wintypes.LPVOID()
if not CreateEnvironmentBlock(
    C.byref(environment_pointer),
    primary_token,
    False,
):
    raise C.WinError(C.get_last_error())

try:
    # Pass environment_pointer with CREATE_UNICODE_ENVIRONMENT. The child has
    # its own copy after a successful process-creation call returns.
    pass
finally:
    if not DestroyEnvironmentBlock(environment_pointer):
        raise C.WinError(C.get_last_error())
```

User-specific variables such as `USERPROFILE` depend on the profile being
loaded. Profile loading is separately owned:

```python
class PROFILEINFOW(C.Structure):
    _fields_ = [
        ("dwSize", wintypes.DWORD),
        ("dwFlags", wintypes.DWORD),
        ("lpUserName", wintypes.LPWSTR),
        ("lpProfilePath", wintypes.LPWSTR),
        ("lpDefaultPath", wintypes.LPWSTR),
        ("lpServerName", wintypes.LPWSTR),
        ("lpPolicyPath", wintypes.LPWSTR),
        ("hProfile", wintypes.HANDLE),
    ]


LoadUserProfileW = userenv.LoadUserProfileW
LoadUserProfileW.argtypes = (
    wintypes.HANDLE,
    C.POINTER(PROFILEINFOW),
)
LoadUserProfileW.restype = wintypes.BOOL

UnloadUserProfile = userenv.UnloadUserProfile
UnloadUserProfile.argtypes = (wintypes.HANDLE, wintypes.HANDLE)
UnloadUserProfile.restype = wintypes.BOOL
```

Set `PROFILEINFOW.dwSize`, retain the username buffer, and on success retain
both the token and `hProfile` until the child no longer needs the profile.
Unload with the same token/profile pair. Do not call `CloseHandle` on
`hProfile`; `UnloadUserProfile` owns that release contract.

### `CreateProcessWithTokenW`

```python
CreateProcessWithTokenW = advapi32.CreateProcessWithTokenW
CreateProcessWithTokenW.argtypes = (
    wintypes.HANDLE,
    wintypes.DWORD,
    wintypes.LPCWSTR,
    wintypes.LPWSTR,
    wintypes.DWORD,
    wintypes.LPVOID,
    wintypes.LPCWSTR,
    C.POINTER(STARTUPINFOW),
    C.POINTER(PROCESS_INFORMATION),
)
CreateProcessWithTokenW.restype = wintypes.BOOL

LOGON_WITH_PROFILE = 0x00000001
LOGON_NETCREDENTIALS_ONLY = 0x00000002
```

The token needs `TOKEN_QUERY`, `TOKEN_DUPLICATE`, and
`TOKEN_ASSIGN_PRIMARY`; the caller needs `SeImpersonatePrivilege`. Unlike
`CreateProcessAsUserW`, this function starts the child in the caller's session,
not the session stored in the token. Its command-line limit is 1,024 characters,
which is smaller than `CreateProcessW`'s limit.

`LOGON_WITH_PROFILE` loads the profile and can be slow.
`LOGON_NETCREDENTIALS_ONLY` creates a new logon session for outbound network
credentials while retaining the caller's local token, and the credentials are
not validated merely because creation succeeds. Neither flag is proof of local
or remote access.

### `CreateProcessWithLogonW`

```python
CreateProcessWithLogonW = advapi32.CreateProcessWithLogonW
CreateProcessWithLogonW.argtypes = (
    wintypes.LPCWSTR,  # username
    wintypes.LPCWSTR,  # domain or None
    wintypes.LPCWSTR,  # password: sensitive
    wintypes.DWORD,    # logon flags
    wintypes.LPCWSTR,
    wintypes.LPWSTR,
    wintypes.DWORD,
    wintypes.LPVOID,
    wintypes.LPCWSTR,
    C.POINTER(STARTUPINFOW),
    C.POINTER(PROCESS_INFORMATION),
)
CreateProcessWithLogonW.restype = wintypes.BOOL
```

This function does not require the special caller privileges used by the token
variants, but the account must satisfy its logon policy. It cannot be called
from a process running as LocalSystem. `LOGON_WITH_PROFILE` and
`LOGON_NETCREDENTIALS_ONLY` have the documented meanings above; network-only
credentials are not validated at process creation.

Keep the password out of source, arguments, environment, exceptions, console
history, logs, evidence, and project memory. Because Python cannot guarantee
erasure of all Unicode copies, do not describe a `ctypes` password buffer as a
secure vault. This guide supplies the ABI but intentionally contains no sample
credential.

### `ShellExecuteExW`

Shell activation is for registered verbs and file associations, not a more
powerful spelling of `CreateProcessW`. It may invoke COM shell extensions,
display UI, or return without a process handle depending on the verb and mask.

```python
class SHELLEXECUTEINFOW(C.Structure):
    _fields_ = [
        ("cbSize", wintypes.DWORD),
        ("fMask", C.c_ulong),
        ("hwnd", wintypes.HWND),
        ("lpVerb", wintypes.LPCWSTR),
        ("lpFile", wintypes.LPCWSTR),
        ("lpParameters", wintypes.LPCWSTR),
        ("lpDirectory", wintypes.LPCWSTR),
        ("nShow", C.c_int),
        ("hInstApp", wintypes.HINSTANCE),
        ("lpIDList", wintypes.LPVOID),
        ("lpClass", wintypes.LPCWSTR),
        ("hkeyClass", wintypes.HKEY),
        ("dwHotKey", wintypes.DWORD),
        ("hIconOrMonitor", wintypes.HANDLE),
        ("hProcess", wintypes.HANDLE),
    ]


shell32 = C.WinDLL("shell32", use_last_error=True)
ShellExecuteExW = shell32.ShellExecuteExW
ShellExecuteExW.argtypes = (C.POINTER(SHELLEXECUTEINFOW),)
ShellExecuteExW.restype = wintypes.BOOL

SEE_MASK_NOCLOSEPROCESS = 0x00000040
SEE_MASK_NOASYNC = 0x00000100
SEE_MASK_FLAG_NO_UI = 0x00000400
SW_SHOWNORMAL = 1
```

The `hIconOrMonitor` field represents the SDK union between `hIcon` and
`hMonitor`; both are pointer-sized handles, so one field preserves the ABI for
ordinary use. Use a real `Union` if the wrapper needs named access to both
interpretations.

```python
request = SHELLEXECUTEINFOW()
request.cbSize = C.sizeof(SHELLEXECUTEINFOW)
request.fMask = (
    SEE_MASK_NOCLOSEPROCESS | SEE_MASK_NOASYNC | SEE_MASK_FLAG_NO_UI
)
request.lpVerb = "open"
request.lpFile = str(executable)
request.lpParameters = subprocess.list2cmdline(arguments)
request.lpDirectory = working_directory
request.nShow = SW_SHOWNORMAL

if not ShellExecuteExW(C.byref(request)):
    raise C.WinError(C.get_last_error())

if request.hProcess:
    try:
        wait_result = WaitForSingleObject(request.hProcess, 10_000)
        if wait_result == WAIT_FAILED:
            raise C.WinError(C.get_last_error())
    finally:
        if not CloseHandle(request.hProcess):
            raise C.WinError(C.get_last_error())
```

`lpParameters` excludes the executable's argument-zero token because `lpFile`
identifies the item separately. Quoting still depends on the activated target's
parser. `SEE_MASK_NOCLOSEPROCESS` requests `hProcess`, but shell execution can
succeed without a usable process handle for some verbs or activation paths.
Initialize COM appropriately for the calling thread when shell extensions may
be involved. The `runas` verb can show a UAC consent/credential prompt; do not
use it to claim silent elevation or bypass.

### Opening and inspecting process handles

Use the minimum access rights required by the next operation:

```python
OpenProcess = kernel32.OpenProcess
OpenProcess.argtypes = (
    wintypes.DWORD,
    wintypes.BOOL,
    wintypes.DWORD,
)
OpenProcess.restype = wintypes.HANDLE

GetProcessId = kernel32.GetProcessId
GetProcessId.argtypes = (wintypes.HANDLE,)
GetProcessId.restype = wintypes.DWORD

PROCESS_TERMINATE = 0x0001
PROCESS_SET_QUOTA = 0x0100
PROCESS_QUERY_LIMITED_INFORMATION = 0x1000
SYNCHRONIZE = 0x00100000
```

```python
process = OpenProcess(
    PROCESS_QUERY_LIMITED_INFORMATION | SYNCHRONIZE,
    False,
    observed_pid,
)
if not process:
    raise C.WinError(C.get_last_error())

try:
    confirmed_pid = GetProcessId(process)
    if not confirmed_pid:
        raise C.WinError(C.get_last_error())
finally:
    if not CloseHandle(process):
        raise C.WinError(C.get_last_error())
```

A PID can be reused; it is not durable identity. Opening succeeds only if the
security descriptor and requested rights allow it. Do not request
`PROCESS_ALL_ACCESS`, enable `SeDebugPrivilege`, or broaden rights merely to
silence an access denial. `GetCurrentProcess()` returns a pseudo handle that is
valid only in the caller and must not be closed.

### Suspended creation and `ResumeThread`

`CREATE_SUSPENDED` prevents the primary thread from running until it is resumed.
This is appropriate when a documented pre-start operation, such as assigning a
job, must finish first. It is not evidence that memory modification or injection
is required.

```python
ResumeThread = kernel32.ResumeThread
ResumeThread.argtypes = (wintypes.HANDLE,)
ResumeThread.restype = wintypes.DWORD

CREATE_SUSPENDED = 0x00000004
DWORD_FAILURE = 0xFFFFFFFF
```

```python
created = CreateProcessW(
    str(executable),
    command_buffer,
    None,
    None,
    False,
    CREATE_SUSPENDED,
    None,
    working_directory,
    C.byref(startup),
    C.byref(process_info),
)
if not created:
    raise C.WinError(C.get_last_error())

# Perform only the pre-authorized setup that required a suspended start, then
# resume before entering the ordinary wait/exit-code/two-handle lifecycle.
previous_count = ResumeThread(process_info.hThread)
if previous_count == DWORD_FAILURE:
    raise C.WinError(C.get_last_error())
```

`ResumeThread` returns the previous suspend count; a value greater than one
means the thread remains suspended. Every error path after successful creation
must make an explicit choice: finish setup and resume, retain enough state for
operator recovery, or perform a separately authorized disposal. Simply closing
the handles leaves a suspended process behind.

`TerminateProcess` is an abrupt last resort, not normal cleanup:

```python
TerminateProcess = kernel32.TerminateProcess
TerminateProcess.argtypes = (wintypes.HANDLE, wintypes.UINT)
TerminateProcess.restype = wintypes.BOOL
```

It does not give the target process or its DLLs a normal cleanup path and can
leave shared state inconsistent. This reference defines the ABI so failures can
be understood; it deliberately does not add an automatic termination recipe.

### Job objects for child-tree lifetime

A job object groups processes for accounting and limits. Assignment can fail
when the process is already in an incompatible job, is in another session, or
lacks the needed rights. Once assigned, a process cannot simply remove itself
from the job.

```python
class JOBOBJECT_BASIC_LIMIT_INFORMATION(C.Structure):
    _fields_ = [
        ("PerProcessUserTimeLimit", C.c_int64),  # LARGE_INTEGER
        ("PerJobUserTimeLimit", C.c_int64),
        ("LimitFlags", wintypes.DWORD),
        ("MinimumWorkingSetSize", C.c_size_t),
        ("MaximumWorkingSetSize", C.c_size_t),
        ("ActiveProcessLimit", wintypes.DWORD),
        ("Affinity", C.c_size_t),  # ULONG_PTR
        ("PriorityClass", wintypes.DWORD),
        ("SchedulingClass", wintypes.DWORD),
    ]


class IO_COUNTERS(C.Structure):
    _fields_ = [
        ("ReadOperationCount", C.c_uint64),
        ("WriteOperationCount", C.c_uint64),
        ("OtherOperationCount", C.c_uint64),
        ("ReadTransferCount", C.c_uint64),
        ("WriteTransferCount", C.c_uint64),
        ("OtherTransferCount", C.c_uint64),
    ]


class JOBOBJECT_EXTENDED_LIMIT_INFORMATION(C.Structure):
    _fields_ = [
        ("BasicLimitInformation", JOBOBJECT_BASIC_LIMIT_INFORMATION),
        ("IoInfo", IO_COUNTERS),
        ("ProcessMemoryLimit", C.c_size_t),
        ("JobMemoryLimit", C.c_size_t),
        ("PeakProcessMemoryUsed", C.c_size_t),
        ("PeakJobMemoryUsed", C.c_size_t),
    ]
```

```python
CreateJobObjectW = kernel32.CreateJobObjectW
CreateJobObjectW.argtypes = (
    C.POINTER(SECURITY_ATTRIBUTES),
    wintypes.LPCWSTR,
)
CreateJobObjectW.restype = wintypes.HANDLE

SetInformationJobObject = kernel32.SetInformationJobObject
SetInformationJobObject.argtypes = (
    wintypes.HANDLE,
    C.c_int,  # JOBOBJECTINFOCLASS
    wintypes.LPVOID,
    wintypes.DWORD,
)
SetInformationJobObject.restype = wintypes.BOOL

AssignProcessToJobObject = kernel32.AssignProcessToJobObject
AssignProcessToJobObject.argtypes = (
    wintypes.HANDLE,
    wintypes.HANDLE,
)
AssignProcessToJobObject.restype = wintypes.BOOL

TerminateJobObject = kernel32.TerminateJobObject
TerminateJobObject.argtypes = (wintypes.HANDLE, wintypes.UINT)
TerminateJobObject.restype = wintypes.BOOL

JobObjectExtendedLimitInformation = 9
JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE = 0x00002000
```

A common containment pattern is to create the child suspended, configure and
assign the job, then resume the primary thread:

```python
job = CreateJobObjectW(None, None)
if not job:
    raise C.WinError(C.get_last_error())

limits = JOBOBJECT_EXTENDED_LIMIT_INFORMATION()
limits.BasicLimitInformation.LimitFlags = (
    JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE
)

if not SetInformationJobObject(
    job,
    JobObjectExtendedLimitInformation,
    C.byref(limits),
    C.sizeof(limits),
):
    raise C.WinError(C.get_last_error())

# CreateProcessW(..., CREATE_SUSPENDED, ..., &process_info)
if not AssignProcessToJobObject(job, process_info.hProcess):
    raise C.WinError(C.get_last_error())

if ResumeThread(process_info.hThread) == DWORD_FAILURE:
    raise C.WinError(C.get_last_error())
```

`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` makes closing the final job handle
terminate every associated process. That is useful only when this destructive
lifetime policy is explicit and the job contains exactly the intended child
tree. Do not apply it to a named/pre-existing job or close the handle as routine
cleanup without understanding the effect. `TerminateJobObject` is likewise an
explicit disruptive action, not a generic finally-block operation.

The complete ownership order is:

```text
create job -> configure limits -> create child suspended -> assign child
-> resume -> drain/wait -> close thread/process/pipe handles
-> close job only under its chosen lifetime policy
```

If assignment or resume fails, the child can remain suspended and the job's
close policy may terminate it. Preserve both primary and cleanup errors and
make this outcome visible to the operator.

### Related-process ownership matrix

| Resource | Acquired by | Released by | Important rule |
|---|---|---|---|
| Process/thread handle | Any successful process-creation call | `CloseHandle` | Two independent handles; close each once |
| Pipe handle | `CreatePipe` | `CloseHandle` | Close unused duplicates promptly so EOF works |
| Real token handle | `OpenProcessToken`, `DuplicateTokenEx`, `LogonUserW` | `CloseHandle` | Pseudo handles are the exception and are not closed |
| User environment block | `CreateEnvironmentBlock` | `DestroyEnvironmentBlock` | Child copies it during creation; use Unicode flag |
| Loaded user profile | `LoadUserProfileW` | `UnloadUserProfile` | Retain token and returned `hProfile` pairing |
| Attribute-list state | `InitializeProcThreadAttributeList` | `DeleteProcThreadAttributeList` | Backing storage and values remain caller-owned/alive |
| Shell process handle | `ShellExecuteExW` with `SEE_MASK_NOCLOSEPROCESS` | `CloseHandle` | May be absent for some activation paths |
| Job handle | `CreateJobObjectW` | `CloseHandle` | Closing can terminate the job when kill-on-close is set |

### Coverage limits

This process chapter now covers the complete ordinary creation family and the
supporting APIs most exercises require. It does not turn all of Win32 into one
implied workflow. Remote-process memory APIs, remote-thread creation, debugger
attachment, process hollowing, parent-process spoofing, mitigation bypass,
credential dumping, and protected-process manipulation are separate high-risk
subjects with different authorization and evidence requirements; they are not
prerequisites for correctly using `CreateProcessW`.

<a id="posix-patterns"></a>

## [POSIX patterns](#contents "Back to contents")

The process-global C library is sufficient for many standard POSIX symbols:

```python
import ctypes as C

libc = C.CDLL(None, use_errno=True)

getpid = libc.getpid
getpid.argtypes = ()
getpid.restype = C.c_int  # pid_t on the target platform/header

print(getpid())
```

For a separately linked library, locate or configure its loader name:

```python
from ctypes.util import find_library

name = find_library("m")
if name is None:
    raise RuntimeError("math library not found")

libm = C.CDLL(name, use_errno=True)
cos = libm.cos
cos.argtypes = (C.c_double,)
cos.restype = C.c_double

print(cos(0.0))  # 1.0
```

POSIX type aliases such as `pid_t`, `mode_t`, `off_t`, `socklen_t`, and
`time_t` are platform-defined. Confirm them from headers or a compiled fixture;
do not assume they always map to the same `ctypes` scalar.

<a id="binary-inspection"></a>

## [Binary inspection](#contents "Back to contents")

`from_buffer_copy()` can decode a fixed local byte sequence without retaining a
view into the input. Model file or protocol byte order explicitly.

```python
import ctypes as C

class Header(C.LittleEndianStructure):
    _pack_ = 1  # the example format explicitly has no padding
    _fields_ = [
        ("magic", C.c_ubyte * 4),
        ("version", C.c_uint16),
        ("flags", C.c_uint16),
        ("payload_length", C.c_uint32),
    ]

raw = b"LAB!\x01\x00\x02\x00\x10\x00\x00\x00"
if len(raw) < C.sizeof(Header):
    raise ValueError("truncated header")

header = Header.from_buffer_copy(raw)
if bytes(header.magic) != b"LAB!":
    raise ValueError("unexpected magic")
if header.payload_length > 1_048_576:
    raise ValueError("payload is too large")

print(header.version, header.flags, header.payload_length)
```

For stable file and network formats, `struct`, `int.from_bytes()`, and parser
libraries are often clearer and more portable than `ctypes`. Native structures
may contain padding and compiler-specific layout that is not present on disk.

### Inspect an owned structure's bytes

```python
record = ProcessRecord(process_id=1234, state=2, flags=1)
raw = C.string_at(C.addressof(record), C.sizeof(record))
print(raw.hex(" "))
```

This is safe only while `record` is alive and because the read length is its
known allocation size. Padding bytes can exist and should not be treated as
stable serialized data.

<a id="writing-a-reusable-wrapper"></a>

## [Writing a reusable wrapper](#contents "Back to contents")

Keep unsafe declarations private and expose ordinary Python values:

```python
import ctypes as C
from pathlib import Path

class NativeStatusError(RuntimeError):
    pass


class ExampleLibrary:
    def __init__(self, path: Path) -> None:
        resolved = path.resolve(strict=True)
        self._lib = C.CDLL(str(resolved), use_errno=True)

        self._get_version = self._lib.example_get_version
        self._get_version.argtypes = ()
        self._get_version.restype = C.c_uint32

        self._format_name = self._lib.example_format_name
        self._format_name.argtypes = (
            C.c_char_p,
            C.POINTER(C.c_char),
            C.c_size_t,
        )
        self._format_name.restype = C.c_int

    def version(self) -> int:
        return int(self._get_version())

    def format_name(self, value: str) -> str:
        encoded = value.encode("utf-8")
        if len(encoded) > 4096:
            raise ValueError("name is too large")

        output = C.create_string_buffer(4097)
        status = self._format_name(encoded, output, C.sizeof(output))
        if status != 0:
            raise NativeStatusError(f"format_name failed: {status}")
        return output.value.decode("utf-8", errors="strict")
```

A wrapper should centralize:

- trusted loading and supported architecture checks;
- every function prototype and calling convention;
- Python-to-C range and encoding validation;
- output capacities and returned-length checks;
- failure-sentinel and native-error conversion;
- handle, allocation, callback, and shutdown lifetimes;
- a narrow Python API that does not expose raw addresses unnecessarily.

Do not allow callers to reach the unconfigured library object unless they are
also responsible for the full ABI and safety contract.

### Python C API

`C.pythonapi` is a `PyDLL` for the running interpreter's Python C API. It keeps
the GIL during calls and checks the Python error flag. This is an advanced,
CPython-specific boundary: reference counts, borrowed versus owned references,
interpreter state, version compatibility, and C API preconditions still apply.
Prefer normal Python operations and documented extension interfaces.

<a id="testing-and-debugging"></a>

## [Testing and debugging](#contents "Back to contents")

Test the wrapper at the boundary, not only through a happy path.

### ABI fixture checks

A tiny compiled C fixture can report the values that Python must match:

```c
size_t fixture_sizeof_record(void) { return sizeof(ProcessRecord); }
size_t fixture_offsetof_flags(void) { return offsetof(ProcessRecord, flags); }
```

```python
assert C.sizeof(ProcessRecord) == fixture.fixture_sizeof_record()
assert ProcessRecord.flags.offset == fixture.fixture_offsetof_flags()
```

Also test:

- blank, maximum, embedded-NUL, non-ASCII, and invalid-encoding inputs;
- zero-length and one-byte buffers;
- documented `NULL`, `-1`, `FALSE`, and status-code failures;
- returned lengths equal to and larger than capacity;
- 32-bit/64-bit builds and each supported operating system;
- repeated acquisition/release and callback registration/unregistration;
- callbacks after forced garbage collection while the registration is valid;
- concurrent calls if the native library promises thread safety;
- malformed/truncated bytes before `from_buffer_copy()`.

### Crash diagnostics

Enable Python's fault handler before reproducing a native crash:

```powershell
$env:PYTHONFAULTHANDLER = "1"
python wrapper_test.py
```

```bash
PYTHONFAULTHANDLER=1 python wrapper_test.py
```

`faulthandler` can show Python stacks for access violations or segmentation
faults, but it cannot repair memory corruption. Reduce the reproduction to one
call and compare the declaration with the header, symbol, and architecture.
Use a native debugger and memory sanitizer for the library when possible.

### Inspect what Python declared

```python
print(function.argtypes)
print(function.restype)
print(C.sizeof(MyStructure), C.alignment(MyStructure))

for field_name, _ in MyStructure._fields_:
    field = getattr(MyStructure, field_name)
    print(field_name, field.offset, field.size)
```

<a id="troubleshooting"></a>

## [Troubleshooting](#contents "Back to contents")

| Symptom                                      | Likely cause                                            | What to check                                                            |
| -------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------ |
| `OSError: ... not a valid Win32 application` | Architecture mismatch or invalid DLL                    | Python/DLL bitness and file format                                       |
| DLL/shared library cannot be loaded          | Wrong path, missing dependency, loader search rules     | Absolute path, dependency inspection, trusted search directories         |
| Symbol not found                             | Wrong export name, version, decoration, or library      | Export table, exact suffix (`A`/`W`), ordinal, installed header/version  |
| Result looks truncated or negative           | Default or wrong `restype`                              | Pointer width, signedness, `size_t`, handles, `restype` set before call  |
| Arguments appear shifted or stack is damaged | Wrong calling convention/prototype                      | `CDLL` vs `WinDLL`, argument order/count, variadic rules                 |
| Access violation / segmentation fault        | Bad address, type, lifetime, capacity, or layout        | Prototype, pointer depth, writable buffer, bounds, retained owners       |
| Structure fields contain nonsense            | Layout or byte-order mismatch                           | `sizeof`, offsets, alignment, packing, compiler, endian base class       |
| Text is truncated                            | Embedded NUL or undersized output buffer                | API encoding, explicit length, terminator rules,`.raw` vs `.value`       |
| Unicode text is corrupt                      | Wrong `A`/`W` API or encoding                           | Wide API with `str`; narrow API with correctly encoded `bytes`           |
| `errno` is always zero/stale                 | Loader omitted `use_errno=True` or success was inspected | Failure sentinel and immediate `get_errno()`                             |
| Windows error is wrong/stale                 | Loader omitted `use_last_error=True`                    | Failure sentinel and immediate `get_last_error()`                        |
| Callback crashes later                       | Callback or referenced object was collected             | Retain strong references until native unregistration completes           |
| Callback thread-local state disappears       | Callback comes from a foreign thread                    | Explicit synchronization/context; do not rely on `threading.local`       |
| Invalid free / heap corruption               | Allocator mismatch or double release                    | Exact ownership, same library's release API, one acquisition/one release |
| Works once, fails later                      | Native code retained temporary memory                   | Replace temporary/byref input with a retained owner object               |
| Works on one OS only                         | C widths, symbols, ABI, or loader behavior differ       | Platform-specific declarations and CI/fixture checks                     |

### Ten checks before blaming the library

```text
1. Is this the intended library file and version?
2. Do Python and the library have the same architecture?
3. Is the symbol name exact?
4. Is the calling convention exact?
5. Are all argtypes exact, including pointer depth and signedness?
6. Is restype exact and set before the call?
7. Do structure sizes, offsets, packing, and byte order match?
8. Are buffers writable, large enough, and measured in the right unit?
9. Are every pointer, owner, callback, and library still alive?
10. Was the documented failure sentinel checked before errno/last-error?
```

<a id="security-checklist"></a>

## [Security checklist](#contents "Back to contents")

- Prefer a safe, maintained high-level API when it satisfies the requirement.
- Load only an expected library and version from a trusted location. Treat DLL
  search-path control as code-execution control.
- Verify architecture, symbol, calling convention, complete prototype, and
  structure layout before the first real call.
- Validate all Python-side ranges, lengths, encodings, indexes, flags, and enum
  values before conversion to C types.
- Allocate output buffers from documented maximums or a bounded size-query
  pattern; reject contradictory returned lengths.
- Keep owners and callbacks alive for the entire native retention period.
- Pair every acquired resource with its exact documented release function.
- Never dereference, scan, copy, free, or call an arbitrary address merely
  because it is nonzero.
- Do not expose raw `memmove`, `memset`, `cast`, `from_address`, function-address
  calls, or unconfigured library handles to untrusted input.
- Treat imported libraries as executable code, not data. Hashing or signing can
  help identify an expected artifact but does not make unknown code safe.
- Bound native calls where the API permits it. `ctypes` cannot safely interrupt
  every blocked native call; isolate unreliable libraries in a subprocess.
- Do not log raw memory, credentials, tokens, full native buffers, or sensitive
  OS objects. Minimize and redact lab evidence before sharing.
- For penetration tests and labs, confirm scope and operator intent before a
  native action that affects another process, host, credential store, network,
  or security control. A successful return remains an observation to verify.

`ctypes` can call any native code available to the process. This guide focuses
on interface correctness and benign local examples; it deliberately does not
provide process injection, shellcode execution, credential access, evasion, or
security-control bypass recipes.

<a id="quick-reference"></a>

## [Quick reference](#contents "Back to contents")

### Loading and functions

| API                                   | Purpose                                                           |
| ------------------------------------- | ----------------------------------------------------------------- |
| `C.CDLL(path, use_errno=True)`        | Load a standard C calling-convention library                      |
| `C.WinDLL(path, use_last_error=True)` | Load a Windows `stdcall` library                                  |
| `C.PyDLL(path)`                       | Load a library while retaining the GIL and checking Python errors |
| `C.CDLL(None)`                        | Access suitable symbols in the current process/global namespace   |
| `find_library(name)`                  | Attempt platform-specific library-name resolution                 |
| `func.argtypes = (T1, T2)`            | Declare argument conversion and checking                          |
| `func.restype = T`                    | Declare result type; use `None` for C `void`                      |
| `func.errcheck = checker`             | Check or transform a result after the call                        |

### Types and memory

| API                              | Purpose                                          |
| -------------------------------- | ------------------------------------------------ |
| `C.POINTER(T)`                   | Construct the pointer type `T *`                 |
| `C.pointer(obj)`                 | Construct a pointer object retaining `obj`       |
| `C.byref(obj, offset=0)`         | Pass an object's address efficiently             |
| `C.cast(obj, pointer_type)`      | Reinterpret a pointer value                      |
| `C.create_string_buffer(size)`   | Allocate writable `char` storage                 |
| `C.create_unicode_buffer(count)` | Allocate writable `wchar_t` storage              |
| `C.sizeof(obj_or_type)`          | Return native allocation/type size in bytes      |
| `C.alignment(type)`              | Return required native alignment                 |
| `C.addressof(obj)`               | Return a `ctypes` object's address               |
| `C.string_at(ptr, size)`         | Copy bytes from a known readable range           |
| `C.wstring_at(ptr, size)`        | Copy wide characters from a known readable range |
| `C.memmove(dst, src, count)`     | Copy raw memory, permitting overlap              |
| `C.memset(dst, value, count)`    | Fill raw memory                                  |

### Composite types and callbacks

| Form                               | Purpose                                           |
| ---------------------------------- | ------------------------------------------------- |
| `T * count`                        | Fixed-size C array type                           |
| `class X(C.Structure)`             | Native-order C structure                          |
| `class X(C.Union)`                 | Native-order C union                              |
| `class X(C.LittleEndianStructure)` | Explicit little-endian structure without pointers |
| `C.CFUNCTYPE(result, *args)`       | C `cdecl` callback/function prototype             |
| `C.WINFUNCTYPE(result, *args)`     | Windows `stdcall` callback/function prototype     |
| `C.PYFUNCTYPE(result, *args)`      | Python-convention prototype that retains the GIL  |

### Errors and inspection

| API                                         | Purpose                                                    |
| ------------------------------------------- | ---------------------------------------------------------- |
| `C.get_errno()` / `C.set_errno()`           | Read/write the `ctypes` private POSIX error copy           |
| `C.get_last_error()` / `C.set_last_error()` | Read/write the `ctypes` private Windows error copy         |
| `C.WinError(code)`                          | Build an `OSError` from a Windows error code               |
| `obj.value`                                 | Read/write a simple C value or NUL-terminated buffer value |
| `buffer.raw`                                | Read/write the complete allocated byte buffer              |
| `ptr.contents`                              | Access the object referenced by a typed pointer            |
| `Struct.field.offset`                       | Inspect a native structure field offset                    |

## Further reading

- [Python `ctypes` documentation](https://docs.python.org/3/library/ctypes.html)
- [Python `ctypes.util` library-finding reference](https://docs.python.org/3/library/ctypes.html#finding-shared-libraries)
- [Python `faulthandler` documentation](https://docs.python.org/3/library/faulthandler.html)
- [Python `struct` documentation](https://docs.python.org/3/library/struct.html)
- [Microsoft `CreateProcessW` documentation](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw)
- [Microsoft `STARTUPINFOW` documentation](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/ns-processthreadsapi-startupinfow)
- [Microsoft `PROCESS_INFORMATION` documentation](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/ns-processthreadsapi-process_information)
- [Microsoft `CloseHandle` documentation](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle)
- [Microsoft `WaitForSingleObject` documentation](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject)
- [Microsoft `GetExitCodeProcess` documentation](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getexitcodeprocess)
- [Microsoft process creation flags](https://learn.microsoft.com/en-us/windows/win32/procthread/process-creation-flags)
- [Microsoft environment-block guidance](https://learn.microsoft.com/en-us/windows/win32/procthread/changing-environment-variables)
- [Microsoft process and handle inheritance guidance](https://learn.microsoft.com/en-us/windows/win32/procthread/inheritance)
- [Microsoft `CreatePipe` documentation](https://learn.microsoft.com/en-us/windows/win32/api/namedpipeapi/nf-namedpipeapi-createpipe)
- [Microsoft `SetHandleInformation` documentation](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-sethandleinformation)
- [Microsoft `ReadFile` documentation](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile)
- [Microsoft process-thread attribute-list documentation](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-initializeprocthreadattributelist)
- [Microsoft `UpdateProcThreadAttribute` documentation](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-updateprocthreadattribute)
- [Microsoft `CreateProcessAsUserW` documentation](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessasuserw)
- [Microsoft `CreateProcessWithTokenW` documentation](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createprocesswithtokenw)
- [Microsoft `CreateProcessWithLogonW` documentation](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createprocesswithlogonw)
- [Microsoft `CreateEnvironmentBlock` documentation](https://learn.microsoft.com/en-us/windows/win32/api/userenv/nf-userenv-createenvironmentblock)
- [Microsoft `ShellExecuteExW` documentation](https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-shellexecuteexw)
- [Microsoft job-object documentation](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects)
- [Microsoft `SetInformationJobObject` documentation](https://learn.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-setinformationjobobject)
