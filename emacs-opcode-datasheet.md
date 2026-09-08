# Emacs 31 Bytecode Architecture & Opcode Datasheet

This datasheet provides a comprehensive reference for GNU Emacs bytecode execution, instruction encoding, stack manipulation, and byte-code object formats (as implemented in `src/bytecode.c` and `lisp/emacs-lisp/bytecomp.el`).

---

## 1. Virtual Machine Architecture

Emacs bytecode executes on an evaluation-stack-based virtual machine (`exec_byte_code`).

### 1.1 VM State
* **`pc` (Program Counter)**: Pointer to the current byte in the unibyte bytecode string.
* **`top` (Stack Pointer)**: Pointer to the top element of the Lisp value stack.
* **`vectorp` (Constants Vector)**: Pointer to array of `Lisp_Object` constants and symbol literals.
* **`bytestr_data`**: Base address of the bytecode string. All jump offsets (`goto`) are **absolute byte offsets** relative to `bytestr_data`.

### 1.2 Byte-Code Closure Object Structure
A compiled function object is a pseudovector (`PVEC_CLOSURE` / `#[...]`) with the following elements:

| Index | Field | Description |
|---|---|---|
| `0` | `ARGLIST` / `args_template` | Integer encoding arity (see below) or parameter symbol list |
| `1` | `CODE` | Unibyte string containing raw bytecode opcodes and operands |
| `2` | `CONSTANTS` | Vector of `Lisp_Object` constants, symbol names, and sub-closures |
| `3` | `STACK_DEPTH` | Integer fixnum representing maximum stack slots required |
| `4` | `DOC_STRING` | Optional docstring or integer reference into doc file |
| `5` | `INTERACTIVE` | Optional interactive specification string or expression |

#### `args_template` Integer Bit-Packing
When lexical binding is active, arity is encoded as a single integer:
* **Bits 0–6**: Mandatory parameter count (`mandatory = args_template & 127`).
* **Bit 7**: `&rest` parameter present flag (`rest = (args_template & 128) != 0`).
* **Bits 8–14**: Maximum non-rest positional argument count (`nonrest = args_template >> 8`).

---

## 2. Instruction Encoding Schemes

Bytecode instructions consist of a 1-byte opcode, optionally followed by immediate operand bytes.

### 2.1 Three-Bit Immediate Family (Opcodes 0–47)
Six opcode families pack small operand values (0–5) directly into the opcode's lower 3 bits:
* `stack-ref` (base `0`)
* `varref` (base `8`)
* `varset` (base `16`)
* `varbind` (base `24`)
* `call` (base `32`)
* `unbind` (base `40`)

| Pattern | Operand Range | Encoded Bytes |
|---|---|---|
| `base + N` (`0 <= N <= 5`) | `0 <= N <= 5` | `[opcode]` (1 byte total) |
| `base + 6` | `0 <= N <= 255` | `[opcode, byte0]` (2 bytes total) |
| `base + 7` | `N >= 256` | `[opcode, byte0, byte1]` (3 bytes total, 16-bit little-endian) |

*(Note: `stack-ref 0` is unused/invalid; `dup` is used instead).*

### 2.2 Constant Reference Encoding
* **`0xC0 .. 0xFF` (`192 .. 255`)**: Immediate constant index `0 .. 63` (`idx = opcode - 192`). 1 byte.
* **`0x81` (`129`, `constant2`)**: Constant index `>= 64`. Followed by 2 bytes (16-bit little-endian index).

### 2.3 Jump Target Encoding
All `goto` instructions take a **2-byte little-endian unsigned offset** indicating the absolute destination byte index from the start of the bytecode string:
```
target_pc = byte0 | (byte1 << 8)
```

### 2.4 Stack Modification Encoding
* **`stack-set` (`178`)**: 1-byte operand specifying stack offset `0 .. 255`.
* **`stack-set2` (`179`)**: 2-byte operand (16-bit little-endian) for stack offset `> 255`.
* **`discardN` (`182`)**: 1-byte operand:
  * If bit 7 (`0x80`) is **0**: pop and discard `N` items from the top of the stack.
  * If bit 7 (`0x80`) is **1**: preserve TOS, discard `N & 0x7F` items *immediately underneath* TOS.

---

## 3. Complete Master Opcode Table

| Opcode | Hex | Octal | Mnemonic | Operands | Stack Effect | Description |
|---|---|---|---|---|---|---|
| 0 | 0x00 | 000 | `byte-stack-ref` | - | - | Invalid opcode (reserved to detect corruption) |
| 1–5 | 0x01–0x05 | 001–005 | `byte-stack-ref1..5` | None | `[] -> [val]` | Push copy of stack slot `top - N` |
| 6 | 0x06 | 006 | `byte-stack-ref6` | 1 byte (`N`) | `[] -> [val]` | Push copy of stack slot `top - N` |
| 7 | 0x07 | 007 | `byte-stack-ref7` | 2 bytes (`N`) | `[] -> [val]` | Push copy of stack slot `top - N` |
| 8–13 | 0x08–0x0D | 010–015 | `byte-varref0..5` | None | `[] -> [val]` | Push dynamic symbol value from `constants[N]` |
| 14 | 0x0E | 016 | `byte-varref6` | 1 byte (`idx`) | `[] -> [val]` | Push dynamic symbol value from `constants[idx]` |
| 15 | 0x0F | 017 | `byte-varref7` | 2 bytes (`idx`)| `[] -> [val]` | Push dynamic symbol value from `constants[idx]` |
| 16–21 | 0x10–0x15 | 020–025 | `byte-varset0..5` | None | `[val] -> []` | Set dynamic symbol in `constants[N]` to `val` |
| 22 | 0x16 | 026 | `byte-varset6` | 1 byte (`idx`) | `[val] -> []` | Set dynamic symbol in `constants[idx]` to `val` |
| 23 | 0x17 | 027 | `byte-varset7` | 2 bytes (`idx`)| `[val] -> []` | Set dynamic symbol in `constants[idx]` to `val` |
| 24–29 | 0x18–0x1D | 030–035 | `byte-varbind0..5`| None | `[val] -> []` | Dynamically bind `constants[N]` to `val` |
| 30 | 0x1E | 036 | `byte-varbind6` | 1 byte (`idx`) | `[val] -> []` | Dynamically bind `constants[idx]` to `val` |
| 31 | 0x1F | 037 | `byte-varbind7` | 2 bytes (`idx`)| `[val] -> []` | Dynamically bind `constants[idx]` to `val` |
| 32–37 | 0x20–0x25 | 040–045 | `byte-call0..5` | None | `[fn, a1..aN] -> [res]` | Call function with `N` arguments |
| 38 | 0x26 | 046 | `byte-call6` | 1 byte (`N`) | `[fn, a1..aN] -> [res]` | Call function with `N` arguments |
| 39 | 0x27 | 047 | `byte-call7` | 2 bytes (`N`)| `[fn, a1..aN] -> [res]` | Call function with `N` arguments |
| 40–45 | 0x28–0x2D | 050–055 | `byte-unbind0..5`| None | `[] -> []` | Unbind `N` dynamic bindings |
| 46 | 0x2E | 056 | `byte-unbind6` | 1 byte (`N`) | `[] -> []` | Unbind `N` dynamic bindings |
| 47 | 0x2F | 057 | `byte-unbind7` | 2 bytes (`N`)| `[] -> []` | Unbind `N` dynamic bindings |
| 48 | 0x30 | 060 | `byte-pophandler` | None | `[] -> []` | Pop current condition-case / catch handler |
| 49 | 0x31 | 061 | `byte-pushconditioncase` | 2 bytes (`pc`) | `[handlers] -> []` | Install condition-case handler pointing to `pc` |
| 50 | 0x32 | 062 | `byte-pushcatch` | 2 bytes (`pc`) | `[tag] -> []` | Install catch handler pointing to `pc` |
| 56 | 0x38 | 070 | `byte-nth` | None | `[n, list] -> [elem]` | `(nth n list)` |
| 57 | 0x39 | 071 | `byte-symbolp` | None | `[x] -> [bool]` | `(symbolp x)` |
| 58 | 0x3A | 072 | `byte-consp` | None | `[x] -> [bool]` | `(consp x)` |
| 59 | 0x3B | 073 | `byte-stringp` | None | `[x] -> [bool]` | `(stringp x)` |
| 60 | 0x3C | 074 | `byte-listp` | None | `[x] -> [bool]` | `(listp x)` |
| 61 | 0x3D | 075 | `byte-eq` | None | `[a, b] -> [bool]` | `(eq a b)` |
| 62 | 0x3E | 076 | `byte-memq` | None | `[elt, list] -> [sub]` | `(memq elt list)` |
| 63 | 0x3F | 077 | `byte-not` | None | `[x] -> [bool]` | `(not x)` / `(null x)` |
| 64 | 0x40 | 0100 | `byte-car` | None | `[list] -> [car]` | `(car list)` |
| 65 | 0x41 | 0101 | `byte-cdr` | None | `[list] -> [cdr]` | `(cdr list)` |
| 66 | 0x42 | 0102 | `byte-cons` | None | `[car, cdr] -> [cons]` | `(cons car cdr)` |
| 67 | 0x43 | 0103 | `byte-list1` | None | `[a] -> [list]` | `(list a)` |
| 68 | 0x44 | 0104 | `byte-list2` | None | `[a, b] -> [list]` | `(list a b)` |
| 69 | 0x45 | 0105 | `byte-list3` | None | `[a, b, c] -> [list]` | `(list a b c)` |
| 70 | 0x46 | 0106 | `byte-list4` | None | `[a, b, c, d] -> [list]`| `(list a b c d)` |
| 71 | 0x47 | 0107 | `byte-length` | None | `[seq] -> [len]` | `(length seq)` |
| 72 | 0x48 | 0110 | `byte-aref` | None | `[array, idx] -> [elt]` | `(aref array idx)` |
| 73 | 0x49 | 0111 | `byte-aset` | None | `[array, idx, v] -> [v]` | `(aset array idx v)` |
| 74 | 0x4A | 0112 | `byte-symbol-value` | None | `[sym] -> [val]` | `(symbol-value sym)` |
| 75 | 0x4B | 0113 | `byte-symbol-function` | None | `[sym] -> [fun]` | `(symbol-function sym)` |
| 76 | 0x4C | 0114 | `byte-set` | None | `[sym, val] -> [val]` | `(set sym val)` |
| 77 | 0x4D | 0115 | `byte-fset` | None | `[sym, fun] -> [fun]` | `(fset sym fun)` |
| 78 | 0x4E | 0116 | `byte-get` | None | `[sym, prop] -> [val]` | `(get sym prop)` |
| 79 | 0x4F | 0117 | `byte-substring` | None | `[str, from, to] -> [sub]` | `(substring str from to)` |
| 80 | 0x50 | 0120 | `byte-concat2` | None | `[s1, s2] -> [str]` | `(concat s1 s2)` |
| 81 | 0x51 | 0121 | `byte-concat3` | None | `[s1, s2, s3] -> [str]` | `(concat s1 s2 s3)` |
| 82 | 0x52 | 0122 | `byte-concat4` | None | `[s1..s4] -> [str]` | `(concat s1 s2 s3 s4)` |
| 83 | 0x53 | 0123 | `byte-sub1` | None | `[x] -> [x - 1]` | `(1- x)` |
| 84 | 0x54 | 0124 | `byte-add1` | None | `[x] -> [x + 1]` | `(1+ x)` |
| 85 | 0x55 | 0125 | `byte-eqlsign` | None | `[a, b] -> [bool]` | `(= a b)` |
| 86 | 0x56 | 0126 | `byte-gtr` | None | `[a, b] -> [bool]` | `(> a b)` |
| 87 | 0x57 | 0127 | `byte-lss` | None | `[a, b] -> [bool]` | `(< a b)` |
| 88 | 0x58 | 0130 | `byte-leq` | None | `[a, b] -> [bool]` | `(<= a b)` |
| 89 | 0x59 | 0131 | `byte-geq` | None | `[a, b] -> [bool]` | `(>= a b)` |
| 90 | 0x5A | 0132 | `byte-diff` | None | `[a, b] -> [a - b]` | `(- a b)` |
| 91 | 0x5B | 0133 | `byte-negate` | None | `[x] -> [-x]` | `(- x)` |
| 92 | 0x5C | 0134 | `byte-plus` | None | `[a, b] -> [a + b]` | `(+ a b)` |
| 93 | 0x5D | 0135 | `byte-max` | None | `[a, b] -> [max]` | `(max a b)` |
| 94 | 0x5E | 0136 | `byte-min` | None | `[a, b] -> [min]` | `(min a b)` |
| 95 | 0x5F | 0137 | `byte-mult` | None | `[a, b] -> [a * b]` | `(* a b)` |
| 96 | 0x60 | 0140 | `byte-point` | None | `[] -> [pt]` | `(point)` |
| 98 | 0x62 | 0142 | `byte-goto-char` | None | `[pt] -> [pt]` | `(goto-char pt)` |
| 99 | 0x63 | 0143 | `byte-insert` | None | `[str] -> [nil]` | `(insert str)` |
| 100 | 0x64 | 0144 | `byte-point-max` | None | `[] -> [max]` | `(point-max)` |
| 101 | 0x65 | 0145 | `byte-point-min` | None | `[] -> [min]` | `(point-min)` |
| 102 | 0x66 | 0146 | `byte-char-after` | None | `[pt] -> [char]` | `(char-after pt)` |
| 103 | 0x67 | 0147 | `byte-following-char` | None | `[] -> [char]` | `(following-char)` |
| 104 | 0x68 | 0150 | `byte-preceding-char` | None | `[] -> [char]` | `(preceding-char)` |
| 105 | 0x69 | 0151 | `byte-current-column` | None | `[] -> [col]` | `(current-column)` |
| 106 | 0x6A | 0152 | `byte-indent-to` | None | `[col] -> [col]` | `(indent-to col)` |
| 108 | 0x6C | 0154 | `byte-eolp` | None | `[] -> [bool]` | `(eolp)` |
| 109 | 0x6D | 0155 | `byte-eobp` | None | `[] -> [bool]` | `(eobp)` |
| 110 | 0x6E | 0156 | `byte-bolp` | None | `[] -> [bool]` | `(bolp)` |
| 111 | 0x6F | 0157 | `byte-bobp` | None | `[] -> [bool]` | `(bobp)` |
| 112 | 0x70 | 0160 | `byte-current-buffer` | None | `[] -> [buf]` | `(current-buffer)` |
| 113 | 0x71 | 0161 | `byte-set-buffer` | None | `[buf] -> [buf]` | `(set-buffer buf)` |
| 114 | 0x72 | 0162 | `byte-save-current-buffer`| None | `[] -> []` | Records unwind protect for buffer |
| 117 | 0x75 | 0165 | `byte-forward-char` | None | `[n] -> [nil]` | `(forward-char n)` |
| 118 | 0x76 | 0166 | `byte-forward-word` | None | `[n] -> [nil]` | `(forward-word n)` |
| 119 | 0x77 | 0167 | `byte-skip-chars-forward` | None | `[str, lim] -> [num]` | `(skip-chars-forward str lim)` |
| 120 | 0x78 | 0170 | `byte-skip-chars-backward`| None | `[str, lim] -> [num]` | `(skip-chars-backward str lim)` |
| 121 | 0x79 | 0171 | `byte-forward-line` | None | `[n] -> [num]` | `(forward-line n)` |
| 122 | 0x7A | 0172 | `byte-char-syntax` | None | `[ch] -> [syntax]` | `(char-syntax ch)` |
| 123 | 0x7B | 0173 | `byte-buffer-substring` | None | `[beg, end] -> [str]` | `(buffer-substring beg end)` |
| 124 | 0x7C | 0174 | `byte-delete-region` | None | `[beg, end] -> [nil]` | `(delete-region beg end)` |
| 125 | 0x7D | 0175 | `byte-narrow-to-region` | None | `[beg, end] -> [nil]` | `(narrow-to-region beg end)` |
| 126 | 0x7E | 0176 | `byte-widen` | None | `[] -> [nil]` | `(widen)` |
| 127 | 0x7F | 0177 | `byte-end-of-line` | None | `[n] -> [nil]` | `(end-of-line n)` |
| 129 | 0x81 | 0201 | `byte-constant2` | 2 bytes (`idx`) | `[] -> [val]` | Push `constants[idx]` (`idx >= 64`) |
| 130 | 0x82 | 0202 | `byte-goto` | 2 bytes (`pc`) | `[] -> []` | Jump unconditionally to `pc` |
| 131 | 0x83 | 0203 | `byte-goto-if-nil` | 2 bytes (`pc`) | `[val] -> []` | Pop `val`; if `nil`, jump to `pc` |
| 132 | 0x84 | 0204 | `byte-goto-if-not-nil` | 2 bytes (`pc`) | `[val] -> []` | Pop `val`; if non-nil, jump to `pc` |
| 133 | 0x85 | 0205 | `byte-goto-if-nil-else-pop` | 2 bytes (`pc`) | `[val] -> [val]` (or pop) | Peek `val`: if `nil`, jump; else pop |
| 134 | 0x86 | 0206 | `byte-goto-if-not-nil-else-pop` | 2 bytes (`pc`) | `[val] -> [val]` (or pop) | Peek `val`: if non-nil, jump; else pop |
| 135 | 0x87 | 0207 | `byte-return` | None | `[val] -> (return)` | Return `val` from current frame |
| 136 | 0x88 | 0210 | `byte-discard` | None | `[val] -> []` | Drop top element of stack |
| 137 | 0x89 | 0211 | `byte-dup` | None | `[val] -> [val, val]` | Duplicate top element of stack |
| 138 | 0x8A | 0212 | `byte-save-excursion` | None | `[] -> []` | Record unwind protect for buffer/point/mark |
| 140 | 0x8C | 0214 | `byte-save-restriction` | None | `[] -> []` | Record unwind protect for clipping limits |
| 142 | 0x8E | 0216 | `byte-unwind-protect` | None | `[handler] -> []` | Record unwind handler (lambda or form) |
| 147 | 0x93 | 0223 | `byte-set-marker` | None | `[m, pos, buf] -> [m]` | `(set-marker m pos buf)` |
| 148 | 0x94 | 0224 | `byte-match-beginning` | None | `[n] -> [pos]` | `(match-beginning n)` |
| 149 | 0x95 | 0225 | `byte-match-end` | None | `[n] -> [pos]` | `(match-end n)` |
| 150 | 0x96 | 0226 | `byte-upcase` | None | `[obj] -> [obj]` | `(upcase obj)` |
| 151 | 0x97 | 0227 | `byte-downcase` | None | `[obj] -> [obj]` | `(downcase obj)` |
| 152 | 0x98 | 0230 | `byte-string=` | None | `[s1, s2] -> [bool]` | `(string= s1 s2)` |
| 153 | 0x99 | 0231 | `byte-string<` | None | `[s1, s2] -> [bool]` | `(string< s1 s2)` |
| 154 | 0x9A | 0232 | `byte-equal` | None | `[a, b] -> [bool]` | `(equal a b)` |
| 155 | 0x9B | 0233 | `byte-nthcdr` | None | `[n, list] -> [sub]` | `(nthcdr n list)` |
| 156 | 0x9C | 0234 | `byte-elt` | None | `[seq, n] -> [elt]` | `(elt seq n)` |
| 157 | 0x9D | 0235 | `byte-member` | None | `[elt, list] -> [sub]` | `(member elt list)` |
| 158 | 0x9E | 0236 | `byte-assq` | None | `[key, alist] -> [pair]`| `(assq key alist)` |
| 159 | 0x9F | 0237 | `byte-nreverse` | None | `[list] -> [rev]` | `(nreverse list)` |
| 160 | 0xA0 | 0240 | `byte-setcar` | None | `[cell, val] -> [val]` | `(setcar cell val)` |
| 161 | 0xA1 | 0241 | `byte-setcdr` | None | `[cell, val] -> [val]` | `(setcdr cell val)` |
| 162 | 0xA2 | 0242 | `byte-car-safe` | None | `[obj] -> [car]` | `(car-safe obj)` |
| 163 | 0xA3 | 0243 | `byte-cdr-safe` | None | `[obj] -> [cdr]` | `(cdr-safe obj)` |
| 164 | 0xA4 | 0244 | `byte-nconc` | None | `[l1, l2] -> [res]` | `(nconc l1 l2)` |
| 165 | 0xA5 | 0245 | `byte-quo` | None | `[a, b] -> [a / b]` | `(/ a b)` |
| 166 | 0xA6 | 0246 | `byte-rem` | None | `[a, b] -> [a % b]` | `(% a b)` |
| 167 | 0xA7 | 0247 | `byte-numberp` | None | `[x] -> [bool]` | `(numberp x)` |
| 168 | 0xA8 | 0250 | `byte-integerp` | None | `[x] -> [bool]` | `(integerp x)` |
| 175 | 0xAF | 0257 | `byte-listN` | 1 byte (`N`) | `[a1..aN] -> [list]` | Make list of `N` stack items |
| 176 | 0xB0 | 0260 | `byte-concatN` | 1 byte (`N`) | `[s1..sN] -> [str]` | Concatenate `N` stack items |
| 177 | 0xB1 | 0261 | `byte-insertN` | 1 byte (`N`) | `[s1..sN] -> [nil]` | Insert `N` items into current buffer |
| 178 | 0xB2 | 0262 | `byte-stack-set` | 1 byte (`off`) | `[val] -> []` | Set stack slot `top - off` to `val`, pop TOS |
| 179 | 0xB3 | 0263 | `byte-stack-set2`| 2 bytes (`off`)| `[val] -> []` | Set stack slot `top - off` to `val`, pop TOS |
| 182 | 0xB6 | 0266 | `byte-discardN` | 1 byte (`flag\|N`) | Variable | Pop `N` items (or `N` items under TOS if `0x80`) |
| 183 | 0xB7 | 0267 | `byte-switch` | None | `[val, table] -> []` | Pop hash table and value, jump to table address |
| 192–255 | 0xC0–0xFF | 0300–0377 | `byte-constant0..63` | None | `[] -> [val]` | Push constant `constants[opcode - 192]` |

---

## 4. Key Mechanisms for Custom Compilers

### 4.1 Implementing Lexical Variables
1. **Binding / Function Entry**: Function arguments and local `let` bindings reside in stack slots.
2. **Reading Variable**: Emit `byte-stack-ref` with the relative stack index (`top - slot`).
3. **Writing / Mutating Variable**: Compute expression on stack, emit `byte-stack-set` with slot offset.
4. **Scope Exit**: Emit `byte-discardN` (or `byte-discardN-preserve-tos`) to pop inner local variables while preserving return values.

### 4.2 Implementing Control Flow (`if` / `while` / `and` / `or`)
* **`if condition then else`**:
  ```
  [eval condition]
  goto-if-nil <ELSE_LABEL>
  [eval then branch]
  goto <END_LABEL>
  ELSE_LABEL:
  [eval else branch]
  END_LABEL:
  ```
* **Short-Circuit `and` (`goto-if-nil-else-pop`)**:
  ```
  [eval expr1]
  goto-if-nil-else-pop <FALSE_LABEL>
  [eval expr2]
  FALSE_LABEL:
  ```
* **Short-Circuit `or` (`goto-if-not-nil-else-pop`)**:
  ```
  [eval expr1]
  goto-if-not-nil-else-pop <TRUE_LABEL>
  [eval expr2]
  TRUE_LABEL:
  ```

### 4.3 Function Invocation Protocol
To call `foo(x, y)`:
1. Push `foo` (via `constant`, `varref`, or `stack-ref`).
2. Push arg 1 (`x`).
3. Push arg 2 (`y`).
4. Emit `call 2` (`0x22`).
Result replaces the function on the stack (`TOP = res`).

---

## 5. Instantiating Byte-Code in Emacs
In Elisp, byte-code is instantiated using `make-byte-code`:
```elisp
(make-byte-code
  args_template       ; e.g. 2 for 2 mandatory args, or (#x0202)
  byte_string         ; unibyte string of raw bytes, e.g. (unibyte-string #xc0 #x20 ...)
  constants_vector    ; vector of symbols/objects: [print "hello"]
  max_stack_depth     ; e.g. 4
  docstring           ; optional
  interactive_spec)   ; optional
```
This byte-code object is immediately callable as a first-class function.
