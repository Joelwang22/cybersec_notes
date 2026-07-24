# Python `re`: Quick Coding Guide

<details>
<summary>Show code</summary>

```python
import re
```

</details>

Prefer raw strings for regex patterns: `r"\d+"` instead of `"\\d+"`. Raw
strings prevent Python from processing backslashes first; regex escapes still
keep their usual meaning.

## Pick the right operation

| Goal | Use |
|---|---|
| Validate the entire string | `re.fullmatch(pattern, text)` |
| Find the first match anywhere | `re.search(pattern, text)` |
| Match only at the beginning | `re.match(pattern, text)` |
| Get all matching strings/groups | `re.findall(pattern, text)` |
| Loop over matches with positions | `re.finditer(pattern, text)` |
| Replace matches | `re.sub(pattern, replacement, text)` |
| Split using a pattern | `re.split(pattern, text)` |
| Reuse the same pattern | `re.compile(pattern)` |

Most code uses `fullmatch()`, `search()`, `findall()`, or `sub()`.

<details>
<summary>Show example</summary>

```python
text = "Order 42 costs $19.95"

re.search(r"\d+", text)                  # first match anywhere
re.fullmatch(r"Order .*", text)          # entire string must match
re.findall(r"\d+(?:\.\d+)?", text)      # ['42', '19.95']
re.sub(r"\d", "#", text)                # 'Order ## costs $##.##'
re.split(r"\s+", text)                   # split on whitespace
```

</details>

## Pattern syntax

Use the colour markers to quickly recognise what each part of a regex does:

| Marker | Category | What it controls |
|:---:|---|---|
| 🟦 | [**Characters**](#characters) | What text can match |
| 🟩 | [**Position**](#position) | Where a match can occur |
| 🟨 | [**Repetition**](#repetition) | How many times something can occur |
| 🟪 | [**Groups and choices**](#groups-and-choices) | Structure, capture, and alternatives |
| 🟧 | [**Lookarounds**](#lookarounds) | Required surrounding context |

<a id="characters"></a>

### 🟦 [Characters](#pattern-syntax)

| Pattern | Meaning |
|---|---|
| **<samp class="syntax-characters">abc</samp>** | Literal text |
| **<samp class="syntax-characters">.</samp>** | Any character except newline |
| **<samp class="syntax-characters">\\.</samp>** | Literal dot |
| **<samp class="syntax-characters">[abc]</samp>** | One of `a`, `b`, or `c` |
| **<samp class="syntax-characters">[^abc]</samp>** | Anything except `a`, `b`, or `c` |
| **<samp class="syntax-characters">[a-z]</samp>** | Character from `a` through `z` |
| **<samp class="syntax-characters">\d</samp>** / **<samp class="syntax-characters">\D</samp>** | Digit / not a digit |
| **<samp class="syntax-characters">\s</samp>** / **<samp class="syntax-characters">\S</samp>** | Whitespace / not whitespace |
| **<samp class="syntax-characters">\w</samp>** / **<samp class="syntax-characters">\W</samp>** | Word character / not a word character |

`\d`, `\s`, and `\w` are Unicode-aware. Use `[0-9]`, explicit ranges, or
`re.ASCII` when you specifically need ASCII behavior.

<a id="position"></a>

### 🟩 [Position](#pattern-syntax)

| Pattern | Meaning |
|---|---|
| **<samp class="syntax-position">^</samp>** | Start of string; also start of line with `re.MULTILINE` |
| **<samp class="syntax-position">$</samp>** | End of string or before its final newline; also line end with `re.MULTILINE` |
| **<samp class="syntax-position">\A</samp>** | Absolute start of string |
| **<samp class="syntax-position">\Z</samp>** | Absolute end of string |
| **<samp class="syntax-position">\b</samp>** | Word boundary |
| **<samp class="syntax-position">\B</samp>** | Not a word boundary |

For validation, use `fullmatch()` instead of manually adding `^...$`.

<details>
<summary>Show example</summary>

```python
valid = re.fullmatch(r"[A-Za-z_][A-Za-z0-9_]*", name) is not None
```

</details>

<a id="repetition"></a>

### 🟨 [Repetition](#pattern-syntax)

| Pattern | Meaning |
|---|---|
| **<samp class="syntax-repetition">x\*</samp>** | Zero or more |
| **<samp class="syntax-repetition">x+</samp>** | One or more |
| **<samp class="syntax-repetition">x?</samp>** | Zero or one |
| **<samp class="syntax-repetition">x{3}</samp>** | Exactly 3 |
| **<samp class="syntax-repetition">x{2,5}</samp>** | Between 2 and 5 |
| **<samp class="syntax-repetition">x{2,}</samp>** | At least 2 |
| **<samp class="syntax-repetition">x{,5}</samp>** | At most 5 |

Repetition is greedy by default. Add `?` to take as little as possible:
`*?`, `+?`, `??`, or `{m,n}?`.

<details>
<summary>Show example</summary>

```python
re.findall(r"<.*?>", "<b>one</b><i>two</i>")
# ['<b>', '</b>', '<i>', '</i>']
```

</details>

When the delimiter is known, a negated class is often clearer than `.*?`:

<details>
<summary>Show example</summary>

```python
r'"[^"\r\n]*"'   # quoted text without embedded quotes or newlines
```

</details>

<a id="groups-and-choices"></a>

### 🟪 [Groups and choices](#pattern-syntax)

| Pattern | Meaning |
|---|---|
| **<samp class="syntax-groups">(abc)</samp>** | Capture a group |
| **<samp class="syntax-groups">(?:abc)</samp>** | Group without capturing |
| **<samp class="syntax-groups">(?P&lt;name&gt;abc)</samp>** | Named captured group |
| **<samp class="syntax-groups">a&#124;b</samp>** | Match `a` or `b` |
| **<samp class="syntax-groups">\1</samp>** | Match the same text as group 1 |
| **<samp class="syntax-groups">(?P=name)</samp>** | Match the same text as a named group |

Use `(?:...)` when you need grouping but do not need the captured value.

<details>
<summary>Show example</summary>

```python
price_re = re.compile(r"\$(?P<amount>\d+(?:\.\d{2})?)")

if match := price_re.search("Total: $19.95"):
    print(match.group("amount"))            # 19.95
```

</details>

<a id="lookarounds"></a>

### 🟧 [Lookarounds](#pattern-syntax)

Lookarounds check nearby text without including it in the match.

| Pattern | Meaning |
|---|---|
| **<samp class="syntax-lookarounds">(?=...)</samp>** | Must be followed by |
| **<samp class="syntax-lookarounds">(?!...)</samp>** | Must not be followed by |
| **<samp class="syntax-lookarounds">(?&lt;=...)</samp>** | Must be preceded by |
| **<samp class="syntax-lookarounds">(?&lt;!...)</samp>** | Must not be preceded by |

<details>
<summary>Show examples</summary>

```python
r"\d+(?= USD)"       # digits followed by ' USD'
r"foo(?!bar)"        # 'foo' not followed by 'bar'
r"(?<=\$)\d+"       # digits preceded by '$'
r"(?<!\w)cat(?!\w)" # standalone 'cat'
```

</details>

Lookbehinds must match a fixed length; `(?<=ab|cd)` works, but `(?<=a+)` and
`(?<=a|bc)` do not.

## Working with matches

`search()`, `match()`, and `fullmatch()` return a `Match` or `None`.

<details>
<summary>Show example</summary>

```python
match = re.search(r"(?P<user>\w+)@(?P<host>[\w.]+)", text)

if match:
    match.group()          # entire match
    match.group(1)         # first captured group
    match.group("user")    # named captured group
    match.groups()         # tuple of captured groups
    match.groupdict()      # dict of named groups
    match.start()          # starting index
    match.end()            # ending index
    match.span()           # (start, end)
```

</details>

Use the walrus operator when you only need the match inside the condition:

<details>
<summary>Show example</summary>

```python
if match := re.search(r"\d+", text):
    print(match.group())
```

</details>

## Finding multiple matches

Use `findall()` when you only need the values:

<details>
<summary>Show example</summary>

```python
re.findall(r"\d+", "a1 b22")             # ['1', '22']
re.findall(r"(\w+)=(\d+)", "x=1 y=2")    # [('x', '1'), ('y', '2')]
```

</details>

Capturing groups change what `findall()` returns. Use `(?:...)` for groups you
do not want returned.

Use `finditer()` when you need `Match` objects, groups, or positions:

<details>
<summary>Show example</summary>

```python
for match in re.finditer(r"\w+", "one two"):
    print(match.group(), match.span())
# one (0, 3)
# two (4, 7)
```

</details>

Matches do not overlap. A lookahead can find overlapping matches:

<details>
<summary>Show example</summary>

```python
[m.group(1) for m in re.finditer(r"(?=(ana))", "banana")]
# ['ana', 'ana']
```

</details>

## Replacing matches

<details>
<summary>Show examples</summary>

```python
re.sub(r"\s+", " ", text)                         # collapse whitespace
re.sub(r"(\w+), (\w+)", r"\2 \1", "Lovelace, Ada")
re.sub(r"(?P<key>\w+)=(?P<value>\w+)",
       r"\g<key>: \g<value>", "x=42")
```

</details>

Prefer `\g<1>` over `\1` when digits immediately follow the reference. In a
replacement function, return the replacement text directly; backreferences are
only expanded in replacement strings.

Use a function when replacement logic is needed:

<details>
<summary>Show callable replacement</summary>

```python
def double(match: re.Match[str]) -> str:
    return str(int(match.group()) * 2)

re.sub(r"\d+", double, "3 cats and 4 dogs")
# '6 cats and 8 dogs'
```

</details>

Use `subn()` when you also need the number of replacements:

<details>
<summary>Show example</summary>

```python
cleaned, count = re.subn(r"\s+", " ", text)
```

</details>

## Splitting

<details>
<summary>Show examples</summary>

```python
re.split(r"[,;]\s*", "red, green;blue")   # ['red', 'green', 'blue']
re.split(r"([,;])\s*", "a,b;c")           # delimiters are included
```

</details>

Capturing the separator includes it in the result. Pass `maxsplit=N` to limit
the number of splits.

## Reusing patterns and flags

Compile a regex when it is reused or deserves a descriptive name:

<details>
<summary>Show compiled pattern</summary>

```python
EMAIL_RE = re.compile(
    r"[\w.+-]+@[\w.-]+\.[A-Za-z]{2,}",
    re.IGNORECASE,
)

if EMAIL_RE.fullmatch(value):
    print("email-shaped input")
```

</details>

Combine flags with `|`:

| Flag | Effect |
|---|---|
| `re.IGNORECASE` / `re.I` | Ignore case |
| `re.MULTILINE` / `re.M` | Make `^` and `$` work per line |
| `re.DOTALL` / `re.S` | Make `.` include newlines |
| `re.VERBOSE` / `re.X` | Allow spacing and comments in patterns |
| `re.ASCII` / `re.A` | Give `\w`, `\d`, `\s`, and `\b` ASCII behavior |

<details>
<summary>Show example</summary>

```python
pattern = re.compile(r"^hello.*world$", re.I | re.M)
```

</details>

Use `VERBOSE` for patterns that are difficult to read on one line:

<details>
<summary>Show verbose pattern</summary>

```python
DATE_RE = re.compile(
    r"""
    (?P<year>\d{4})
    - (?P<month>0[1-9]|1[0-2])
    - (?P<day>0[1-9]|[12]\d|3[01])
    """,
    re.VERBOSE,
)
```

</details>

## Common recipes

These match useful shapes; they do not guarantee semantic validity.

<details>
<summary>Show recipes</summary>

```python
# Signed integer or decimal
r"[+-]?(?:\d+(?:\.\d*)?|\.\d+)"

# ASCII identifier
r"[A-Za-z_][A-Za-z0-9_]*"

# Hex colour
r"#(?:[0-9A-Fa-f]{3}|[0-9A-Fa-f]{6})\b"

# ISO date shape (use datetime.date to validate the actual date)
r"\d{4}-\d{2}-\d{2}"

# IPv4 shape (validate that each part is 0-255 afterward)
r"(?:\d{1,3}\.){3}\d{1,3}"

# Quoted string with backslash escapes
r'"(?:\\.|[^"\\])*"'

# Collapse whitespace
clean = re.sub(r"\s+", " ", text).strip()

# Remove repeated adjacent words
clean = re.sub(r"\b(\w+)(?:\s+\1)+\b", r"\1", text, flags=re.I)

# camelCase to snake_case
snake = re.sub(r"(?<=[a-z0-9])(?=[A-Z])", "_", name).lower()

# Extract key=value pairs
pairs = dict(re.findall(r"(\w+)=([^\s]+)", text))

# Keep only digits
digits = re.sub(r"\D", "", text)
```

</details>

Escape arbitrary text before inserting it into a pattern:

<details>
<summary>Show example</summary>

```python
needle = "a+b?"
match = re.search(re.escape(needle), text)
```

</details>

`re.escape()` is for pattern fragments, not replacement strings.

## Common mistakes

- Use `fullmatch()`—not `match()`—to validate an entire string.
- Prefer raw-string patterns so `"\b"` does not become a backspace before the
  regex engine sees it.
- Remember that captured groups change the output of `findall()`.
- Use `re.escape(value)` when inserting literal or user-provided text.
- Remember that `\w` and `\d` are Unicode-aware unless ASCII is requested.
- Avoid using regex as a full parser for HTML, JSON, CSV, or programming code.
- Treat dates, email addresses, and IP patterns as shapes; validate their meaning
  with the relevant parser afterward.
- Avoid ambiguous nested repetition such as `r"(a+)+$"`; it can become extremely
  slow on a non-match.

When a regex misbehaves, inspect the real input and the match:

<details>
<summary>Show debugging code</summary>

```python
print(repr(text))
print(match.group(), match.span(), match.groups())
re.compile(pattern, re.DEBUG)
```

</details>

## Compact lookup

<details>
<summary>Show compact lookup</summary>

```text
🟦 CHARACTERS   .  [abc]  [^abc]  [a-z]  \d \D  \s \S  \w \W
🟩 POSITION     ^  $  \A  \Z  \b  \B
🟨 REPEAT       *  +  ?  {m}  {m,n}  {m,}  {,n}
🟨 LAZY         *?  +?  ??  {m,n}?
🟪 GROUPS       (...)  (?:...)  (?P<n>...)  \1  (?P=n)
🟪 CHOICE       a|b
🟧 LOOKAROUND   (?=...)  (?!...)  (?<=...)  (?<!...)

CORE API        fullmatch search match findall finditer sub split compile
```

</details>

Python documentation: <https://docs.python.org/3/library/re.html>
