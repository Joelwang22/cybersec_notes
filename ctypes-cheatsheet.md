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
- [POSIX patterns](#posix-patterns)
- [Binary inspection](#binary-inspection)
- [Writing a reusable wrapper](#writing-a-reusable-wrapper)
- [Testing and debugging](#testing-and-debugging)
- [Troubleshooting](#troubleshooting)
- [Security checklist](#security-checklist)
- [Quick reference](#quick-reference)

<a id="pick-a-workflow"></a>

## [Pick a workflow](#contents "Back to contents")

| Goal                                      | Starting point                                             |
| ----------------------------------------- | ---------------------------------------------------------- |
| Load a normal C library                   | `C.CDLL(path)`                                             |
| Load the current process/global symbols   | `C.CDLL(None)`                                             |
| Load a Windows `stdcall` API              | `C.WinDLL(path, use_last_error=True)`                      |
| Locate a library by a short POSIX name    | `ctypes.util.find_library(name)`                           |
| Define parameters and result              | `func.argtypes = (...)`; `func.restype = ...`              |
| Represent a C pointer                     | `C.POINTER(T)` or `C.c_void_p`                             |
| Pass an existing object by address        | `C.byref(obj)`                                             |
| Allocate writable bytes                   | `C.create_string_buffer(size)`                             |
| Allocate writable wide characters         | `C.create_unicode_buffer(size)`                            |
| Represent a fixed C array                 | `T * count`                                                |
| Represent a C `struct`                    | Subclass `C.Structure` and set `_fields_`                  |
| Represent a C `union`                     | Subclass `C.Union` and set `_fields_`                      |
| Create a C-callable Python callback       | `C.CFUNCTYPE(...)`                                         |
| Read captured POSIX `errno`               | Load with `use_errno=True`; call `C.get_errno()`           |
| Read captured Windows last-error          | Load with `use_last_error=True`; call `C.get_last_error()` |
| Convert a failed result into an exception | Assign `func.errcheck`                                     |
| Check size, alignment, or address         | `C.sizeof`, `C.alignment`, `C.addressof`                   |

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

| Contract detail    | Questions to answer                                                        |
| ------------------ | -------------------------------------------------------------------------- |
| Export             | What is the exact symbol name or ordinal?                                  |
| Calling convention | `cdecl`, Windows `stdcall`, COM/HRESULT, or something unsupported?         |
| Arguments          | Exact order, signedness, width, pointer depth, and `const` status?         |
| Return             | Scalar, pointer, structure, error code, or `void`?                         |
| Layout             | Field order, padding, alignment, packing, bit-fields, and byte order?      |
| Strings            | `char*`, `wchar_t*`, encoding, NUL termination, and length units?          |
| Buffers            | Who allocates, how large, and may the callee write to them?                |
| Ownership          | Who frees returned memory, with which allocator, and when?                 |
| Errors             | Sentinel return, `errno`, Windows last-error, `HRESULT`, or out parameter? |
| Lifetime           | Does native code retain any pointer or callback after the call?            |
| Threads            | Can callbacks arrive on foreign threads, and is the function thread-safe?  |

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
| `C.CDLL`   | Standard C / `cdecl`; releases the GIL during calls     | C libraries on all supported platforms |
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

| C declaration                   | Common `ctypes` type             | Python value               |
| ------------------------------- | -------------------------------- | -------------------------- |
| `_Bool` / C `bool`              | `C.c_bool`                       | `bool`                     |
| `char`                          | `C.c_char`                       | one-byte `bytes`           |
| `signed char` / `unsigned char` | `C.c_byte` / `C.c_ubyte`         | `int`                      |
| `short` / `unsigned short`      | `C.c_short` / `C.c_ushort`       | `int`                      |
| `int` / `unsigned int`          | `C.c_int` / `C.c_uint`           | `int`                      |
| `long` / `unsigned long`        | `C.c_long` / `C.c_ulong`         | `int`                      |
| `long long` / unsigned          | `C.c_longlong` / `C.c_ulonglong` | `int`                      |
| `int8_t` ... `int64_t`          | `C.c_int8` ... `C.c_int64`       | `int`                      |
| `uint8_t` ... `uint64_t`        | `C.c_uint8` ... `C.c_uint64`     | `int`                      |
| `size_t` / `ssize_t`            | `C.c_size_t` / `C.c_ssize_t`     | `int`                      |
| `float` / `double`              | `C.c_float` / `C.c_double`       | `float`                    |
| `wchar_t`                       | `C.c_wchar`                      | one-character `str`        |
| `char *`                        | `C.c_char_p`                     | `bytes` or `None`          |
| `wchar_t *`                     | `C.c_wchar_p`                    | `str` or `None`            |
| `void *`                        | `C.c_void_p`                     | address as `int` or `None` |
| Python object pointer           | `C.py_object`                    | Python object              |

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

| Native parameter                           | Use                              | Important distinction                     |
| ------------------------------------------ | -------------------------------- | ----------------------------------------- |
| Read-only NUL-terminated `const char *`    | `C.c_char_p`; pass `bytes`       | Encoding comes from the API, not `ctypes` |
| Read-only NUL-terminated `const wchar_t *` | `C.c_wchar_p`; pass `str`        | `wchar_t` width is platform-dependent     |
| Writable `char *` buffer                   | `C.create_string_buffer(size)`   | Capacity is bytes                         |
| Writable `wchar_t *` buffer                | `C.create_unicode_buffer(count)` | Capacity is wide characters               |
| Binary data with explicit length           | `POINTER(c_ubyte)` or an array   | Embedded NUL bytes are valid              |

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

| Operation                     | Result                                          |
| ----------------------------- | ----------------------------------------------- |
| `C.POINTER(T)`                | A pointer _type_ for `T *`                      |
| `C.pointer(obj)`              | A pointer object that owns a reference to `obj` |
| `C.byref(obj)`                | A lightweight temporary address for a call      |
| `ptr.contents`                | Object referenced by a typed pointer            |
| `ptr[index]`                  | Pointer indexing with no bounds checking        |
| `C.cast(value, pointer_type)` | Reinterpret an address as another pointer type  |
| `C.addressof(obj)`            | Integer address of a `ctypes` object            |

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

| Symptom                                      | Likely cause                                             | What to check                                                            |
| -------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------ |
| `OSError: ... not a valid Win32 application` | Architecture mismatch or invalid DLL                     | Python/DLL bitness and file format                                       |
| DLL/shared library cannot be loaded          | Wrong path, missing dependency, loader search rules      | Absolute path, dependency inspection, trusted search directories         |
| Symbol not found                             | Wrong export name, version, decoration, or library       | Export table, exact suffix (`A`/`W`), ordinal, installed header/version  |
| Result looks truncated or negative           | Default or wrong `restype`                               | Pointer width, signedness, `size_t`, handles, `restype` set before call  |
| Arguments appear shifted or stack is damaged | Wrong calling convention/prototype                       | `CDLL` vs `WinDLL`, argument order/count, variadic rules                 |
| Access violation / segmentation fault        | Bad address, type, lifetime, capacity, or layout         | Prototype, pointer depth, writable buffer, bounds, retained owners       |
| Structure fields contain nonsense            | Layout or byte-order mismatch                            | `sizeof`, offsets, alignment, packing, compiler, endian base class       |
| Text is truncated                            | Embedded NUL or undersized output buffer                 | API encoding, explicit length, terminator rules, `.raw` vs `.value`      |
| Unicode text is corrupt                      | Wrong `A`/`W` API or encoding                            | Wide API with `str`; narrow API with correctly encoded `bytes`           |
| `errno` is always zero/stale                 | Loader omitted `use_errno=True` or success was inspected | Failure sentinel and immediate `get_errno()`                             |
| Windows error is wrong/stale                 | Loader omitted `use_last_error=True`                     | Failure sentinel and immediate `get_last_error()`                        |
| Callback crashes later                       | Callback or referenced object was collected              | Retain strong references until native unregistration completes           |
| Callback thread-local state disappears       | Callback comes from a foreign thread                     | Explicit synchronization/context; do not rely on `threading.local`       |
| Invalid free / heap corruption               | Allocator mismatch or double release                     | Exact ownership, same library's release API, one acquisition/one release |
| Works once, fails later                      | Native code retained temporary memory                    | Replace temporary/byref input with a retained owner object               |
| Works on one OS only                         | C widths, symbols, ABI, or loader behavior differ        | Platform-specific declarations and CI/fixture checks                     |

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
