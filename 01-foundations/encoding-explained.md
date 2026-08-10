# Base64 — Structured Learning Notes

> Purpose: Build a strong understanding of Base64 from the fundamentals, including bits, bytes, ASCII, UTF-8, Base64 encoding, Base62, Base64URL, padding, and URL shorteners.

---

## 1. What is Base64?

**Base64 is an encoding scheme that converts binary data into text characters.**

It is important to remember:

- Base64 is **encoding**, not encryption.
- Anyone can decode Base64.
- Base64 is **not compression**.
- Base64 is commonly used when binary data needs to be represented as text.

Basic flow:

```text
Binary data
    ↓
Base64 encode
    ↓
Text
```

And the reverse:

```text
Text
    ↓
Base64 decode
    ↓
Binary data
```

Example:

```text
Hello
  ↓ Base64 encode
SGVsbG8=
  ↓ Base64 decode
Hello
```

---

# 2. What is a Bit?

A **bit** is the smallest unit of binary information.

It can have only two values:

```text
0
1
```

Therefore:

```text
1 bit → 2 possible values
```

With 2 bits:

```text
00
01
10
11
```

There are:

```text
2² = 4 possibilities
```

With 3 bits:

```text
000
001
010
011
100
101
110
111
```

There are:

```text
2³ = 8 possibilities
```

General rule:

```text
n bits → 2ⁿ possible combinations
```

Therefore:

```text
6 bits → 2⁶
      → 64 possible combinations
```

This is the foundation of Base64.

---

# 3. What is a Byte?

A **byte is 8 bits**.

```text
1 byte = 8 bits
```

Since each bit has two possibilities:

```text
8 bits → 2⁸
      → 256 possible values
```

Therefore one byte can represent values from:

```text
0 → 255
```

Example:

```text
01000001
```

is one byte.

Its decimal value is:

```text
64 + 1 = 65
```

So:

```text
01000001 → 65
```

---

# 4. Binary → Decimal

To understand Base64, you should be comfortable converting binary to decimal.

Each bit represents a power of 2:

```text
128 64 32 16 8 4 2 1
 ↓   ↓  ↓  ↓ ↓ ↓ ↓ ↓
 0   1  0  0 0 0 0 1
```

For:

```text
01000001
```

we have:

```text
0×128
+ 1×64
+ 0×32
+ 0×16
+ 0×8
+ 0×4
+ 0×2
+ 1×1

= 65
```

Therefore:

```text
01000001 = 65
```

---

# 5. What is ASCII?

ASCII is a character-to-number mapping.

For example:

```text
A → 65
B → 66
C → 67
...
Z → 90
```

Lowercase:

```text
a → 97
b → 98
c → 99
...
z → 122
```

So:

```text
"A"
 ↓
ASCII
 ↓
65
 ↓
Binary
 ↓
01000001
```

In JavaScript / Node.js:

```javascript
const char = "A";

console.log(char.charCodeAt(0));
// 65
```

The reverse:

```javascript
console.log(String.fromCharCode(65));
// A
```

Important:

**ASCII is not the same thing as a byte.**

A byte is:

```text
8 bits
```

ASCII is a **mapping** that assigns numbers to characters.

You can think of:

```text
Byte value: 01000001
Decimal:    65
ASCII:      A
```

as different representations of the same underlying value in this example.

---

# 6. ASCII vs Unicode vs UTF-8

ASCII is enough for basic English characters, but modern software needs to represent many more characters:

```text
₹
न
你
😊
```

Unicode is the universal character set.

UTF-8 is an encoding used to represent Unicode characters as bytes.

For example:

```text
A
```

in UTF-8 is:

```text
41
```

But characters such as:

```text
😊
```

require multiple bytes in UTF-8.

This distinction matters because computers and programming languages ultimately work with bytes when handling encoded data.

In Node.js:

```javascript
Buffer.from("Hello")
```

creates a buffer containing the UTF-8 bytes for the string.

For Base64, the important chain is:

```text
Character
   ↓
Unicode character
   ↓
UTF-8 encoding
   ↓
Bytes
   ↓
Bits
```

---

# 7. Why Does Base64 Have 64 Characters?

The standard Base64 alphabet contains 64 characters:

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZ
abcdefghijklmnopqrstuvwxyz
0123456789+/
```

Count:

```text
26 uppercase
+ 26 lowercase
+ 10 digits
+ 2 symbols
----------------
= 64
```

And we already know:

```text
6 bits → 2⁶ → 64 possibilities
```

Therefore, Base64 can assign one character to each possible 6-bit value.

Conceptually:

```text
6-bit value     Base64 character

000000     →    A
000001     →    B
000010     →    C
000011     →    D
...
011001     →    Z
011010     →    a
...
110011     →    z
110100     →    0
...
111101     →    9
111110     →    +
111111     →    /
```

The exact Base64 alphabet is:

```text
Value 0  → A
Value 1  → B
Value 2  → C
...
Value 25 → Z

Value 26 → a
Value 27 → b
...
Value 51 → z

Value 52 → 0
Value 53 → 1
...
Value 61 → 9

Value 62 → +
Value 63 → /
```

---

# 8. What Does "One Base64 Character Represents 6 Bits" Mean?

This statement can sound confusing.

It means:

> One Base64 character can represent one value from 0 through 63, and 6 bits are enough to represent 64 different values.

Because:

```text
2⁶ = 64
```

For example:

```text
000000 → value 0 → A
000001 → value 1 → B
000010 → value 2 → C
...
111111 → value 63 → /
```

So:

```text
1 Base64 character
        ↓
1 value from 0–63
        ↓
6 bits
```

This is why MDN says:

> Each Base64 digit represents 6 bits of data.

It does **not** mean that the character itself is physically stored as only 6 bits.

The Base64 character is still a text character. The statement is about how much binary information that character represents in the Base64 encoding scheme.

---

# 9. Why Does Base64 Use 6 Bits Instead of 8?

A byte has 8 bits.

If we wanted one character to represent all possible byte values, we would need:

```text
2⁸ = 256 characters
```

But Base64 uses only 64 characters:

```text
2⁶ = 64
```

Therefore:

```text
8 bits → 256 possible values
6 bits → 64 possible values
```

Base64 deliberately uses 64 convenient text characters.

---

# 10. The Most Important Base64 Rule: 3 Bytes → 4 Characters

This is the key mechanic.

A byte has 8 bits.

Three bytes:

```text
3 × 8 = 24 bits
```

Base64 works with groups of 6 bits:

```text
24 ÷ 6 = 4
```

Therefore:

```text
3 bytes → 4 Base64 characters
```

This is why Base64 increases the size of data.

---

# 11. Complete Base64 Example: "ABC"

Start with:

```text
ABC
```

ASCII/UTF-8 byte values:

```text
A → 65 → 01000001
B → 66 → 01000010
C → 67 → 01000011
```

Put all bits together:

```text
01000001 01000010 01000011
```

Remove the spaces:

```text
010000010100001001000011
```

There are 24 bits.

Now split them into 6-bit groups:

```text
010000 | 010100 | 001001 | 000011
```

Convert each group to decimal:

```text
010000 → 16
010100 → 20
001001 → 9
000011 → 3
```

Look up those values in the Base64 alphabet:

```text
16 → Q
20 → U
9  → J
3  → D
```

Therefore:

```text
ABC
 ↓
010000010100001001000011
 ↓
010000 | 010100 | 001001 | 000011
 ↓
  Q   |   U    |   J     |   D
 ↓
QUJD
```

So:

```text
ABC → QUJD
```

---

# 12. Why Does Base64 Make Data Larger?

Three bytes contain:

```text
24 bits
```

Base64 represents those 24 bits using:

```text
4 characters
```

Each character is normally stored as at least one byte in common text representations.

Therefore:

```text
3 bytes → 4 Base64 characters
```

The theoretical size ratio is:

```text
4 / 3 ≈ 1.333
```

So Base64 increases the size by approximately:

```text
33%
```

Example:

```text
Original: 3 MB
Base64:   roughly 4 MB
```

Base64 is therefore **not compression**.

---

# 13. What Happens When We Don't Have Exactly 3 Bytes?

Base64 processes data in groups of 3 bytes.

But what if the input is:

```text
A
```

That's only:

```text
8 bits
```

or:

```text
AB
```

which is:

```text
16 bits
```

Base64 still needs to produce a complete group of 4 Base64 characters.

This is where **padding** comes in.

Examples:

```text
A   → QQ==
AB  → QUI=
ABC → QUJD
```

The `=` characters are padding.

They indicate that the final Base64 block did not contain enough original bytes to fill a complete 3-byte group.

Important:

```text
= is not another Base64 data value
```

It is a padding marker.

---

# 14. Base64 Padding

Base64 processes:

```text
3 input bytes → 4 output characters
```

If the final group has:

```text
3 bytes → no padding
2 bytes → one =
1 byte  → two ==
```

Examples:

```text
A   → QQ==
AB  → QUI=
ABC → QUJD
```

Padding allows the decoder to understand how much actual input data existed in the final group.

---

# 15. Base64 Is Encoding, NOT Encryption

This is extremely important.

Base64:

```text
Hello
 ↓
SGVsbG8=
```

Anyone can decode it:

```text
SGVsbG8=
 ↓
Hello
```

There is:

```text
No secret key
No password
No security
```

Encryption is different:

```text
Hello
 ↓ encryption + key
encrypted data
```

Without the key, the original data should not be directly recoverable.

Therefore:

| Property | Base64 | Encryption |
|---|---|---|
| Purpose | Encoding | Security |
| Secret key | No | Yes |
| Reversible | Yes | Yes, with key |
| Secure by itself | No | Designed for security |
| Compresses data | No | Not its purpose |

---

# 16. Base64 Is Not Compression

Base64 normally makes data larger.

Example:

```text
3 bytes
 ↓
4 Base64 characters
```

So:

```text
Base64 ≠ compression
```

Its purpose is compatibility and representation, not reducing size.

---

# 17. Where Is Base64 Used?

Common use cases include:

### APIs

An API may accept an image as:

```json
{
  "image": "/9j/4AAQSkZJRgABAQAAAQ..."
}
```

The string may contain the Base64 representation of the image bytes.

### Images / Data URLs

Example:

```html
<img src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUg...">
```

### Email

Base64 is commonly used in MIME email encoding when binary attachments need to be represented in a text-compatible format.

### JWT

JWT uses Base64URL-style encoding for its header and payload.

Important:

**Base64 encoding inside a JWT does not make the payload secret.**

Anyone can decode the payload.

---

# 18. Base64URL

Standard Base64 uses:

```text
+
/
=
```

These characters can be inconvenient in URLs.

Base64URL modifies the representation:

```text
+ → -
/ → _
```

Padding `=` is normally omitted.

So:

```text
Standard Base64:
abc+/xyz==

Base64URL:
abc-_xyz
```

Base64URL is commonly used in:

- JWTs
- URL-safe tokens
- Other data that needs Base64-like encoding inside URLs

---

# 19. Base64 vs Base64URL

| | Base64 | Base64URL |
|---|---|---|
| Uppercase | Yes | Yes |
| Lowercase | Yes | Yes |
| Digits | Yes | Yes |
| `+` | Yes | No |
| `/` | Yes | No |
| `-` | No | Yes |
| `_` | No | Yes |
| `=` padding | Usually | Usually omitted |
| URL friendly | Not always | Yes |

---

# 20. Base62

Base62 is another encoding alphabet.

It normally uses:

```text
A-Z → 26
a-z → 26
0-9 → 10
--------------
62 characters
```

Alphabet:

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789
```

Unlike Base64, Base62 doesn't use:

```text
+
/
=
```

This makes it naturally convenient for URLs and IDs.

---

# 21. Base62 vs Base64

| | Base62 | Base64 |
|---|---|---|
| Characters | 62 | 64 |
| A-Z | Yes | Yes |
| a-z | Yes | Yes |
| 0-9 | Yes | Yes |
| `+` `/` | No | Yes |
| Padding `=` | Usually no | Usually yes |
| URL friendly | Yes | Not always |
| Common purpose | Short IDs | Binary → text |
| Binary data | Possible | Very common |

The important distinction is:

```text
Base64:
Binary data → text representation

Base62:
Often used as a number → short alphanumeric representation
```

However, Base62 **can** also be used to encode arbitrary binary data. It is just less natural/efficient for that job than Base64.

---

# 22. Why Base62 Can Encode Binary Data Too

Base64 is not the only possible binary-to-text encoding.

You could take:

```text
Binary bytes
    ↓
interpret as a large integer
    ↓
convert integer to Base62
    ↓
Base62 string
```

And reverse it:

```text
Base62 string
    ↓
decode to integer
    ↓
convert back to bytes
```

However, there is an important issue:

### Leading zero bytes

Suppose the binary data is:

```text
00 48 65 6c 6c 6f
```

If you simply interpret this as an integer, the leading `00` doesn't affect the numerical value.

Therefore, an implementation needs additional rules to preserve leading zero bytes.

Base64 avoids this issue because it operates directly on groups of bytes.

---

# 23. Why Base64 Is More Natural for Binary Data

Base64 has:

```text
64 = 2⁶
```

Therefore:

```text
1 Base64 character = 6 bits
```

This creates a clean relationship:

```text
3 bytes
= 24 bits
= 4 × 6-bit groups
= 4 Base64 characters
```

Base62 has:

```text
62 ≠ 2ⁿ
```

More precisely:

```text
log₂(62) ≈ 5.95 bits per character
```

So Base62 is slightly less efficient at representing arbitrary binary data.

---

# 24. Base64 vs Base62 for URL Shorteners

You can use Base64/Base64URL for short URLs.

For example:

```text
Database ID
123456789
    ↓
Base64/Base64URL
    ↓
short code
```

But Base62 is very common for URL shorteners because its alphabet is naturally:

```text
A-Z
a-z
0-9
```

Example:

```text
https://short.com/aZ91kP
```

There are no special URL characters.

---

# 25. Important: URL Shorteners Usually Don't Encode the Entire URL

A common misconception is:

```text
Long URL
   ↓
Base62
   ↓
Short URL
```

Usually, that is not what a URL shortener does.

Instead:

```text
Long URL
https://example.com/products/iphone/...
             ↓
       Database record
             ↓
        ID = 123456
             ↓
         Base62
             ↓
          w7E9
```

Database:

```text
┌────────┬──────────────────────────────────────┐
│ ID     │ Original URL                         │
├────────┼──────────────────────────────────────┤
│ 123456 │ https://example.com/products/...     │
└────────┴──────────────────────────────────────┘
```

Short URL:

```text
https://short.com/w7E9
```

When a user visits it:

```text
w7E9
 ↓
Base62 decode
 ↓
123456
 ↓
Database lookup
 ↓
Original URL
 ↓
HTTP redirect
```

So Base62 is being used to create a compact representation of an ID, not necessarily to compress the entire original URL.

---

# 26. Character → ASCII → Binary → Base64

This is the complete chain worth remembering.

Take:

```text
A
```

### Step 1: Character

```text
A
```

### Step 2: ASCII value

```text
A → 65
```

### Step 3: Binary

```text
65 → 01000001
```

### Step 4: Collect bytes

For:

```text
ABC
```

we get:

```text
A → 01000001
B → 01000010
C → 01000011
```

### Step 5: Join bits

```text
010000010100001001000011
```

### Step 6: Split into 6-bit groups

```text
010000 | 010100 | 001001 | 000011
```

### Step 7: Convert groups to numbers

```text
16 | 20 | 9 | 3
```

### Step 8: Base64 lookup

```text
16 → Q
20 → U
9  → J
3  → D
```

### Final result

```text
ABC → QUJD
```

---

# 27. The Complete Mental Model

This is the hierarchy to remember:

```text
                    Character
                       │
                       ▼
               Unicode / ASCII
                       │
                       ▼
                    UTF-8
                       │
                       ▼
                     Bytes
                       │
                       ▼
                 Binary bits
                       │
                       ▼
              6-bit Base64 groups
                       │
                       ▼
               Base64 characters
```

For example:

```text
"ABC"
  │
  ▼
A = 65
B = 66
C = 67
  │
  ▼
01000001 01000010 01000011
  │
  ▼
010000 | 010100 | 001001 | 000011
  │
  ▼
 Q     |   U    |   J     |   D
  │
  ▼
"QUJD"
```

---

# 28. Node.js Examples

## Character → ASCII

```javascript
const char = "A";

console.log(char.charCodeAt(0));
// 65
```

## ASCII → Character

```javascript
console.log(String.fromCharCode(65));
// A
```

## String → Base64

```javascript
const original = "Hello";

const encoded = Buffer
  .from(original)
  .toString("base64");

console.log(encoded);
// SGVsbG8=
```

## Base64 → String

```javascript
const decoded = Buffer
  .from("SGVsbG8=", "base64")
  .toString("utf8");

console.log(decoded);
// Hello
```

## File → Base64

```javascript
const fs = require("fs");

const base64 = fs
  .readFileSync("image.jpg")
  .toString("base64");

console.log(base64);
```

## Base64 → File

```javascript
const buffer = Buffer.from(base64, "base64");

fs.writeFileSync("image-copy.jpg", buffer);
```

---

# 29. Things to Learn Next

To become completely comfortable with Base64 and binary data, learn these concepts in this order:

```text
1. Bits
   ↓
2. Bytes
   ↓
3. Binary ↔ Decimal
   ↓
4. ASCII
   ↓
5. Unicode
   ↓
6. UTF-8
   ↓
7. Character encoding
   ↓
8. Base64 alphabet
   ↓
9. 6-bit grouping
   ↓
10. Base64 padding
   ↓
11. Base64URL
   ↓
12. Base62
   ↓
13. Node.js Buffer
   ↓
14. Binary files
```

The most important distinction to keep in your head is:

```text
Character
    ≠
ASCII
    ≠
Byte
    ≠
Bit
    ≠
Base64 character
```

They are related, but they are different concepts.

---

# 30. Quick Cheat Sheet

```text
Bit
→ 0 or 1

Byte
→ 8 bits
→ 0–255

ASCII
→ Character ↔ number mapping
→ A = 65
→ a = 97

UTF-8
→ Encoding that represents Unicode characters as bytes

Base64
→ Binary data → text
→ 64-character alphabet
→ 6 bits per Base64 value
→ 3 bytes → 4 Base64 characters
→ ~33% size increase
→ Not encryption
→ Not compression

Base64URL
→ URL-safe Base64
→ + becomes -
→ / becomes _
→ = padding usually removed

Base62
→ 62 alphanumeric characters
→ Common for short IDs / URL shorteners
→ Can encode binary, but Base64 is more natural for arbitrary binary data
```

---

# 31. One-Sentence Summary

> **Base64 takes bytes, treats the bits in groups of 6, maps each 6-bit value to one of 64 text characters, and uses padding when the final input group is incomplete.**
