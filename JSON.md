# JSON: A Deep Systems-Level Analysis
## From Lexical Tokens to Runtime Representation

---

## Table of Contents
1. [Foundational Concepts & Philosophy](#foundational-concepts)
2. [Lexical Structure & Grammar](#lexical-structure)
3. [Parser Architecture & Implementation](#parser-architecture)
4. [Memory Representation & Runtime Models](#memory-representation)
5. [Serialization & Deserialization Mechanics](#serialization-mechanics)
6. [Network Transport & Wire Format](#network-transport)
7. [Performance Characteristics & Optimization](#performance-characteristics)
8. [Security Considerations & Attack Vectors](#security-considerations)
9. [Comparative Analysis: JSON vs Alternatives](#comparative-analysis)
10. [Implementation Deep-Dive: Building a JSON Parser](#implementation-deep-dive)

---

<a name="foundational-concepts"></a>
## 1. Foundational Concepts & Philosophy

### 1.1 What JSON Actually Is

JSON (JavaScript Object Notation) is not merely a "data format" — it's a **context-free grammar** that defines a serialization protocol for transforming in-memory data structures into a linear byte stream and vice versa. At its core, JSON solves the **impedance mismatch** problem between:

- **Runtime heap structures** (objects, arrays, primitives in memory)
- **Sequential byte streams** (files, network packets, message queues)

When we say "JSON is language-independent," we mean its grammar can be mapped to the type systems of virtually any programming language through **structural typing** rather than nominal typing.

### 1.2 Historical Context & Design Philosophy

JSON emerged from Douglas Crockford's work in the early 2000s, distilled from JavaScript's object literal notation. The design philosophy:

1. **Minimalism**: Only 6 data types (object, array, string, number, boolean, null)
2. **Human readability**: Text-based, not binary
3. **Language agnosticism**: No language-specific constructs
4. **Self-describing**: Schema embedded in structure
5. **UTF-8 native**: Unicode support built-in

This is fundamentally different from binary protocols like Protocol Buffers or MessagePack, which prioritize compactness over human readability.

### 1.3 The Serialization Problem Space

Consider what happens when you want to send an object over a network:

```
HEAP MEMORY (Process A)          WIRE (Network)          HEAP MEMORY (Process B)
┌─────────────────┐                                     ┌─────────────────┐
│ Object@0x7f3a   │         ┌──────────────┐            │ Object@0x9b2c   │
│ ├─ name: "foo"  │ ──────> │ {            │ ────────>  │ ├─ name: "foo"  │
│ ├─ age: 25      │         │  "name":...  │            │ ├─ age: 25      │
│ └─ tags: [...]  │         │  "age":...   │            │ └─ tags: [...]  │
└─────────────────┘         └──────────────┘            └─────────────────┘
   Pointer graph              Byte stream                 Reconstructed graph
```

JSON provides the **codec** (encoder/decoder pair) that performs this transformation while preserving semantic meaning.

---

<a name="lexical-structure"></a>
## 2. Lexical Structure & Grammar

### 2.1 The Formal Grammar (EBNF Notation)

JSON's grammar is defined in RFC 8259. Here's the complete formal specification:

```ebnf
JSON-text = ws value ws

value = false / null / true / object / array / number / string

object = begin-object [ member *( value-separator member ) ] end-object
member = string name-separator value

array = begin-array [ value *( value-separator value ) ] end-array

number = [ minus ] int [ frac ] [ exp ]
int = zero / ( digit1-9 *DIGIT )
frac = decimal-point 1*DIGIT
exp = e [ minus / plus ] 1*DIGIT

string = quotation-mark *char quotation-mark
char = unescaped /
       escape (
           %x22 /          ; "    quotation mark
           %x5C /          ; \    reverse solidus
           %x2F /          ; /    solidus
           %x62 /          ; b    backspace
           %x66 /          ; f    form feed
           %x6E /          ; n    line feed
           %x72 /          ; r    carriage return
           %x74 /          ; t    tab
           %x75 4HEXDIG )  ; uXXXX Unicode escape

ws = *( %x20 / %x09 / %x0A / %x0D )  ; Space, Tab, LF, CR
```

### 2.2 Lexical Tokens & Token Classes

A JSON parser performs **lexical analysis** (tokenization) followed by **syntactic analysis** (parsing). The token classes:

| Token Type | Examples | Byte Representation |
|------------|----------|---------------------|
| `T_LBRACE` | `{` | `0x7B` |
| `T_RBRACE` | `}` | `0x7D` |
| `T_LBRACKET` | `[` | `0x5B` |
| `T_RBRACKET` | `]` | `0x5D` |
| `T_COLON` | `:` | `0x3A` |
| `T_COMMA` | `,` | `0x2C` |
| `T_STRING` | `"hello"` | `0x22 ... 0x22` |
| `T_NUMBER` | `123.45e-2` | `0x31 0x32 ...` |
| `T_TRUE` | `true` | `0x74 0x72 0x75 0x65` |
| `T_FALSE` | `false` | `0x66 0x61 ...` |
| `T_NULL` | `null` | `0x6E 0x75 0x6C 0x6C` |
| `T_WHITESPACE` | ` \t\n\r` | `0x20 0x09 0x0A 0x0D` |

### 2.3 Character Encoding: UTF-8 Mechanics

JSON **MUST** be encoded in UTF-8 (RFC 8259 §8.1). This has profound implications:

```
Character: 你
Unicode Code Point: U+4F60
UTF-8 Encoding: 0xE4 0xBD 0xA0

Byte Stream:
┌────┬────┬────┐
│ E4 │ BD │ A0 │  ← 3 bytes for one character
└────┴────┴────┘

In JSON string: "\u4f60" or direct UTF-8 bytes
```

The parser must handle:
- **BOM detection**: Skip UTF-8 BOM (`0xEF 0xBB 0xBF`) if present
- **Invalid sequences**: Reject malformed UTF-8
- **Surrogate pairs**: Handle `\uD800-\uDFFF` correctly

### 2.4 String Escape Sequences: The Encoding Problem

JSON strings require escaping for certain characters. This creates a **two-level encoding**:

```
Source String (in memory): "He said \"Hi\"\n"
                            ↓ (escape)
JSON Wire Format:          "He said \"Hi\"\\n"
                            ↓ (UTF-8 encode)
Byte Stream:               0x22 0x48 0x65 ... 0x5C 0x6E 0x22
```

**Escape sequence processing:**

```c
// Pseudo-code from typical JSON parser
char unescape(char c) {
    switch(c) {
        case '"':  return 0x22;  // Quotation mark
        case '\\': return 0x5C;  // Backslash
        case '/':  return 0x2F;  // Forward slash (optional escape)
        case 'b':  return 0x08;  // Backspace
        case 'f':  return 0x0C;  // Form feed
        case 'n':  return 0x0A;  // Line feed (newline)
        case 'r':  return 0x0D;  // Carriage return
        case 't':  return 0x09;  // Tab
        case 'u':  return parse_unicode_escape();  // \uXXXX
    }
}
```

**Unicode escape handling** (`\uXXXX`):

```python
# Parsing \u4f60
hex_digits = "4f60"
code_point = int(hex_digits, 16)  # 0x4F60 = 20320

# Check for surrogate pairs (U+D800 to U+DFFF)
if 0xD800 <= code_point <= 0xDBFF:  # High surrogate
    # Must be followed by low surrogate \uDC00-\uDFFF
    low_surrogate = parse_next_escape()
    actual_code_point = (
        ((code_point - 0xD800) << 10) +
        (low_surrogate - 0xDC00) +
        0x10000
    )
    return encode_utf8(actual_code_point)
else:
    return encode_utf8(code_point)
```

### 2.5 Number Format: IEEE 754 Implications

JSON numbers follow this grammar:
```
number = [ minus ] int [ frac ] [ exp ]
```

**Critical insight**: JSON does **not** specify precision or range. The spec says:

> "This specification allows implementations to set limits on the range and precision of numbers accepted."

This creates interoperability issues:

```json
{
  "precise": 9007199254740993,
  "problematic": 1.7976931348623157e+308,
  "tiny": 2.2250738585072014e-308
}
```

**Why this matters:**

1. **JavaScript's Number type**: IEEE 754 double-precision (64-bit)
   - Max safe integer: `2^53 - 1` (9,007,199,254,740,991)
   - Beyond this, precision loss occurs

2. **Parsing issues:**
   ```javascript
   JSON.parse('{"id": 9007199254740993}').id
   // Returns: 9007199254740992 (precision lost!)
   ```

3. **Cross-language concerns:**
   - Java: `BigDecimal` for arbitrary precision
   - Python: `int` has unlimited precision
   - C: `long` may be 32-bit or 64-bit
   - Rust: Explicit `i64`, `u64`, `f64` choices

**Parser implementation detail:**

```c
// Simplified number parser (à la RapidJSON)
double parse_number(const char* str) {
    bool negative = false;
    if (*str == '-') {
        negative = true;
        str++;
    }

    // Parse integer part
    uint64_t i64 = 0;
    while (isdigit(*str)) {
        i64 = i64 * 10 + (*str - '0');
        if (i64 > MAX_SAFE_INTEGER) {
            // Switch to floating-point parsing
            return parse_as_double(str);
        }
        str++;
    }

    // Check for fractional part
    if (*str == '.') {
        return parse_as_double(str);
    }

    // Check for exponent
    if (*str == 'e' || *str == 'E') {
        return parse_as_double(str);
    }

    return negative ? -(double)i64 : (double)i64;
}
```

---

<a name="parser-architecture"></a>
## 3. Parser Architecture & Implementation

### 3.1 Parser Design Philosophies

There are three primary approaches to JSON parsing:

#### A. Recursive Descent Parser (DOM-style)

**Architecture:**
```
Input Stream → Lexer → Token Stream → Recursive Parser → DOM Tree
```

**Characteristics:**
- Builds complete in-memory representation
- Easy to implement and reason about
- High memory usage for large documents
- Single-pass construction

**Example structure** (conceptual):

```c
typedef enum {
    JSON_OBJECT,
    JSON_ARRAY,
    JSON_STRING,
    JSON_NUMBER,
    JSON_TRUE,
    JSON_FALSE,
    JSON_NULL
} JsonType;

typedef struct JsonValue {
    JsonType type;
    union {
        struct {
            size_t size;
            char** keys;
            struct JsonValue** values;
        } object;
        struct {
            size_t size;
            struct JsonValue** elements;
        } array;
        char* string;
        double number;
    } data;
} JsonValue;

JsonValue* parse_value(Lexer* lexer) {
    Token tok = lexer_peek(lexer);

    switch (tok.type) {
        case T_LBRACE:
            return parse_object(lexer);
        case T_LBRACKET:
            return parse_array(lexer);
        case T_STRING:
            return parse_string(lexer);
        case T_NUMBER:
            return parse_number(lexer);
        // ... etc
    }
}

JsonValue* parse_object(Lexer* lexer) {
    expect(lexer, T_LBRACE);
    JsonValue* obj = create_object();

    if (peek(lexer) == T_RBRACE) {
        expect(lexer, T_RBRACE);
        return obj;
    }

    while (true) {
        Token key = expect(lexer, T_STRING);
        expect(lexer, T_COLON);
        JsonValue* value = parse_value(lexer);

        object_insert(obj, key.string, value);

        if (peek(lexer) == T_COMMA) {
            expect(lexer, T_COMMA);
        } else {
            break;
        }
    }

    expect(lexer, T_RBRACE);
    return obj;
}
```

#### B. SAX-style (Event-driven) Parser

**Architecture:**
```
Input Stream → Parser → Events → Callback Handlers → Application Logic
```

**Characteristics:**
- No intermediate DOM
- Constant memory usage
- Requires callback handling
- Fast for selective parsing

**Event model:**

```c
typedef enum {
    EVENT_OBJECT_START,
    EVENT_OBJECT_KEY,
    EVENT_OBJECT_END,
    EVENT_ARRAY_START,
    EVENT_ARRAY_END,
    EVENT_STRING,
    EVENT_NUMBER,
    EVENT_BOOLEAN,
    EVENT_NULL
} EventType;

typedef struct {
    bool (*on_object_start)(void* ctx);
    bool (*on_object_key)(void* ctx, const char* key, size_t len);
    bool (*on_object_end)(void* ctx);
    bool (*on_array_start)(void* ctx);
    bool (*on_array_end)(void* ctx);
    bool (*on_string)(void* ctx, const char* str, size_t len);
    bool (*on_number)(void* ctx, double num);
    // ... etc
} SaxHandler;

void parse_sax(const char* json, SaxHandler* handler, void* ctx) {
    Lexer lexer = {json, 0};
    parse_value_sax(&lexer, handler, ctx);
}
```

**Use case example** (extracting single field):

```c
typedef struct {
    const char* target_key;
    double result;
    bool found;
} ExtractContext;

bool on_key(void* ctx, const char* key, size_t len) {
    ExtractContext* ectx = (ExtractContext*)ctx;
    if (strncmp(key, ectx->target_key, len) == 0) {
        ectx->found = true;
    }
    return true;
}

bool on_number(void* ctx, double num) {
    ExtractContext* ectx = (ExtractContext*)ctx;
    if (ectx->found) {
        ectx->result = num;
        return false;  // Stop parsing
    }
    return true;
}
```

#### C. SIMD-Accelerated Parser (Modern Approach)

**Architecture:**
```
Input Stream → SIMD Scanner → Structural Indices → Lazy Tree Builder → DOM
```

**Key innovation** (from simdjson):

1. **Stage 1**: Identify structural characters using SIMD
2. **Stage 2**: Build tape/index of structures
3. **Stage 3**: Navigate using index (lazy evaluation)

**SIMD structural character detection:**

```c
// Simplified from simdjson
// Process 64 bytes at once using AVX2

#include <immintrin.h>

uint64_t find_structural_bits(const char* data) {
    __m256i input = _mm256_loadu_si256((const __m256i*)data);

    // Create masks for structural characters
    __m256i lbrace = _mm256_set1_epi8('{');
    __m256i rbrace = _mm256_set1_epi8('}');
    __m256i lbracket = _mm256_set1_epi8('[');
    __m256i rbracket = _mm256_set1_epi8(']');
    __m256i colon = _mm256_set1_epi8(':');
    __m256i comma = _mm256_set1_epi8(',');

    // Compare all at once
    __m256i cmp1 = _mm256_cmpeq_epi8(input, lbrace);
    __m256i cmp2 = _mm256_cmpeq_epi8(input, rbrace);
    __m256i cmp3 = _mm256_cmpeq_epi8(input, lbracket);
    // ... etc

    // Combine results
    __m256i structural = _mm256_or_si256(cmp1, cmp2);
    structural = _mm256_or_si256(structural, cmp3);
    // ... etc

    // Extract bitmask
    return _mm256_movemask_epi8(structural);
}
```

This approach achieves **gigabytes per second** parsing speeds by:
- Processing 32-64 bytes per instruction
- Avoiding branch mispredictions
- Building structural index for O(1) navigation

### 3.2 Memory Management Strategies

#### Arena Allocation (Fast Path)

Most JSON parsers use arena/region allocators for performance:

```c
typedef struct {
    char* buffer;
    size_t size;
    size_t offset;
} Arena;

void* arena_alloc(Arena* arena, size_t size) {
    size = ALIGN(size, 8);  // 8-byte alignment
    if (arena->offset + size > arena->size) {
        return NULL;  // Out of memory
    }
    void* ptr = arena->buffer + arena->offset;
    arena->offset += size;
    return ptr;
}

// All JSON nodes allocated from arena
JsonValue* create_value(Arena* arena, JsonType type) {
    JsonValue* v = arena_alloc(arena, sizeof(JsonValue));
    v->type = type;
    return v;
}

// Free entire document at once
void free_json(JsonValue* root, Arena* arena) {
    free(arena->buffer);  // Single free() call
}
```

**Benefits:**
- **No fragmentation**: Sequential allocation
- **Cache-friendly**: Adjacent data = better locality
- **Fast cleanup**: Single free operation
- **No per-object overhead**: No metadata per allocation

**Trade-off**: Cannot free individual nodes, only entire document.

### 3.3 Error Handling & Recovery

Professional parsers provide detailed error information:

```c
typedef struct {
    size_t line;
    size_t column;
    size_t offset;
    const char* message;
    const char* context;  // Surrounding text
} ParseError;

ParseError create_error(Lexer* lexer, const char* message) {
    return (ParseError){
        .line = lexer->line,
        .column = lexer->column,
        .offset = lexer->offset,
        .message = message,
        .context = get_context(lexer, 20)  // 20 chars context
    };
}

// Usage
if (peek(lexer) != T_COLON) {
    return create_error(lexer,
        "Expected ':' after object key");
}
```

**Error recovery strategies:**

1. **Fail-fast**: Stop at first error (most common)
2. **Error accumulation**: Continue parsing, collect all errors
3. **Synchronization**: Skip to next structural element
4. **Fuzzy matching**: Suggest corrections (advanced)

---

<a name="memory-representation"></a>
## 4. Memory Representation & Runtime Models

### 4.1 In-Memory Layout Across Languages

#### JavaScript (V8 Engine)

V8 represents JSON objects as **hidden classes** with inline properties:

```
Object: { "name": "Alice", "age": 25 }

Memory Layout:
┌──────────────────────────┐
│ Hidden Class Pointer     │ ──→ [HiddenClass: {name, age}]
├──────────────────────────┤
│ Properties Pointer       │ ──→ [Properties Array]
├──────────────────────────┤
│ Elements Pointer         │     (null for plain objects)
├──────────────────────────┤
│ Inline Property: name    │ ──→ [String: "Alice"]
├──────────────────────────┤
│ Inline Property: age     │     [SMI: 25] (Small Integer)
└──────────────────────────┘
```

**Key optimizations:**
- **SMI** (Small Integer): Values fitting in 31 bits stored inline, tagged
- **Hidden classes**: Share structure across similar objects
- **Inline caching**: Property access compiled to direct offset

Arrays use separate **elements backing store**:

```
Array: [1, 2, 3]

┌──────────────────────────┐
│ Hidden Class (Array)     │
├──────────────────────────┤
│ Elements Pointer         │ ──→ [1, 2, 3, <hole>, <hole>, ...]
├──────────────────────────┤
│ Length                   │     3
└──────────────────────────┘
```

#### Python (CPython)

Python dictionaries use **hash tables** with open addressing:

```python
# {"key": "value"}

PyDictObject:
┌──────────────────────────┐
│ ob_refcnt                │  Reference count
├──────────────────────────┤
│ ob_type                  │  → &PyDict_Type
├──────────────────────────┤
│ ma_used                  │  Number of active entries
├──────────────────────────┤
│ ma_version_tag           │  Version for inline caching
├──────────────────────────┤
│ ma_keys                  │  → DictKeys structure
├──────────────────────────┤
│ ma_values                │  → Values array
└──────────────────────────┘

DictKeys (hash table):
┌─────┬─────┬─────┬─────┐
│hash │hash │hash │hash │  Hashes
├─────┼─────┼─────┼─────┤
│ idx │ idx │ idx │ idx │  Indices into entries
└─────┴─────┴─────┴─────┘

Entries:
┌──────────────────┐
│ hash("key")      │
│ &PyStr("key")    │
│ &PyStr("value")  │
└──────────────────┘
```

**Performance characteristics:**
- Average lookup: O(1)
- Worst case (collisions): O(n)
- Load factor threshold: 2/3 before resize
- Resize: O(n) with table doubling

#### Java (HotSpot JVM)

Java's `HashMap` uses **separate chaining** with tree conversion:

```java
// HashMap internal structure

HashMap {
    Node<K,V>[] table;
    int size;
    int threshold;
    float loadFactor;
}

// Before Java 8: linked list
Node<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;  // Chain for collisions
}

// After Java 8: converts to tree if bucket has 8+ elements
TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;
    boolean red;  // Red-black tree color
}
```

**Memory overhead calculation:**

```
JSON object with N key-value pairs:
- Array overhead: 16 bytes (header) + 8N bytes (pointers)
- Node overhead: 32N bytes (hash=4, key=8, value=8, next=8, header=4)
- Key strings: ~40 bytes each (average)
- Value objects: variable

Total ≈ 16 + 40N + (value sizes)
For N=10: ~416 bytes + values
```

#### Rust (serde_json)

Rust uses **enum-based tagged union**:

```rust
// From serde_json's source
pub enum Value {
    Null,
    Bool(bool),
    Number(Number),
    String(String),
    Array(Vec<Value>),
    Object(Map<String, Value>),
}

// Map is typically BTreeMap or HashMap
pub struct Map<K, V> {
    inner: MapImpl<K, V>,  // BTreeMap by default
}
```

**Memory layout (simplified):**

```
Value::Object(map):
┌─────────────────────────┐
│ Discriminant (tag): 5   │  1 byte (or padding to 8)
├─────────────────────────┤
│ Map pointer             │  8 bytes
└─────────────────────────┘  Total: 16 bytes

Map (BTreeMap):
┌─────────────────────────┐
│ Root node pointer       │
├─────────────────────────┤
│ Length                  │
└─────────────────────────┘

BTreeNode:
┌──────┬──────┬──────┐
│ Key₁ │ Val₁ │ ...  │  Internal nodes
└──────┴──────┴──────┘
```

**Benefits:**
- **Zero-cost abstraction**: No runtime overhead vs C
- **Memory safety**: Ownership prevents leaks/use-after-free
- **Tight layout**: Enums size = largest variant + discriminant

### 4.2 Cache Line Optimization

Modern parsers consider CPU cache architecture:

```
L1 Cache Line: 64 bytes
L2 Cache Line: 64 bytes
L3 Cache Line: 64 bytes (shared)

Optimal struct packing:
```

```c
// Bad: 96 bytes (crosses 2 cache lines)
struct JsonValue_Bad {
    JsonType type;        // 4 bytes
    char padding1[4];     // Alignment
    union {
        // ... 64 bytes
    } data;
    JsonValue* next;      // 8 bytes
    char* source_ref;     // 8 bytes
    uint32_t line;        // 4 bytes
    uint32_t column;      // 4 bytes
    // Total: 96 bytes
};

// Good: 64 bytes (fits in one cache line)
struct JsonValue_Good {
    JsonType type;        // 4 bytes
    uint32_t flags;       // 4 bytes (line/column encoded)
    union {
        // ... 48 bytes
    } data;
    JsonValue* next;      // 8 bytes
    // Total: 64 bytes
};
```

**Cache-conscious parsing:**

```c
// Prefetch next node during parse
void parse_array_optimized(Lexer* lexer, JsonValue* arr) {
    JsonValue* prev = NULL;

    while (true) {
        JsonValue* elem = alloc_value();

        // Prefetch next allocation
        __builtin_prefetch(arena->buffer + arena->offset + 64);

        parse_value(lexer, elem);

        if (prev) {
            prev->next = elem;
        } else {
            arr->data.array.head = elem;
        }

        prev = elem;

        if (peek(lexer) != T_COMMA) break;
        advance(lexer);
    }
}
```

### 4.3 String Interning & Deduplication

For JSON with repeated keys, interning saves memory:

```c
typedef struct {
    char** strings;
    size_t capacity;
    size_t count;
} StringPool;

const char* intern_string(StringPool* pool, const char* str, size_t len) {
    // Check if string already exists
    uint32_t hash = hash_string(str, len);
    size_t idx = hash % pool->capacity;

    for (size_t i = idx; i < pool->capacity; i++) {
        if (pool->strings[i] == NULL) {
            // Not found, insert
            char* interned = malloc(len + 1);
            memcpy(interned, str, len);
            interned[len] = '\0';
            pool->strings[i] = interned;
            return interned;
        }
        if (strncmp(pool->strings[i], str, len) == 0) {
            // Found, return existing
            return pool->strings[i];
        }
    }

    // Pool full, resize...
}
```

**Memory savings example:**

```json
// 100 objects with same keys
[
  {"id": 1, "name": "...", "email": "..."},
  {"id": 2, "name": "...", "email": "..."},
  // ... 98 more
]

Without interning:
  "id" × 100 = 300 bytes
  "name" × 100 = 500 bytes
  "email" × 100 = 600 bytes
  Total: 1400 bytes

With interning:
  "id" × 1 = 3 bytes
  "name" × 1 = 5 bytes
  "email" × 1 = 6 bytes
  Total: 14 bytes (99% reduction)
```

---

<a name="serialization-mechanics"></a>
## 5. Serialization & Deserialization Mechanics

### 5.1 Serialization Pipeline

The transformation from runtime object to JSON bytes:

```
Object Graph → Traversal → Encoding → Buffering → Output
```

**Depth-First Traversal:**

```c
void serialize_value(JsonValue* v, Buffer* buf) {
    switch (v->type) {
        case JSON_OBJECT:
            buffer_append(buf, "{", 1);
            for (size_t i = 0; i < v->data.object.size; i++) {
                if (i > 0) buffer_append(buf, ",", 1);

                // Serialize key
                buffer_append(buf, "\"", 1);
                write_escaped_string(buf, v->data.object.keys[i]);
                buffer_append(buf, "\":", 2);

                // Serialize value (recursive)
                serialize_value(v->data.object.values[i], buf);
            }
            buffer_append(buf, "}", 1);
            break;

        case JSON_ARRAY:
            buffer_append(buf, "[", 1);
            for (size_t i = 0; i < v->data.array.size; i++) {
                if (i > 0) buffer_append(buf, ",", 1);
                serialize_value(v->data.array.elements[i], buf);
            }
            buffer_append(buf, "]", 1);
            break;

        case JSON _STRING:
            buffer_append(buf, "\"", 1);
            write_escaped_string(buf, v->data.string);
            buffer_append(buf, "\"", 1);
            break;

        case JSON_NUMBER:
            write_number(buf, v->data.number);
            break;

        case JSON_TRUE:
            buffer_append(buf, "true", 4);
            break;

        case JSON_FALSE:
            buffer_append(buf, "false", 5);
            break;

        case JSON_NULL:
            buffer_append(buf, "null", 4);
            break;
    }
}
```

### 5.2 String Escaping: Character-by-Character Analysis

**Fast path for ASCII strings:**

```c
void write_escaped_string(Buffer* buf, const char* str) {
    const char* ptr = str;
    const char* start = str;

    while (*ptr) {
        // Fast path: no escaping needed
        if (*ptr > 0x1F && *ptr != '"' && *ptr != '\\') {
            ptr++;
            continue;
        }

        // Flush accumulated chars
        if (ptr > start) {
            buffer_append(buf, start, ptr - start);
        }

        // Escape special char
        switch (*ptr) {
            case '"':  buffer_append(buf, "\\\"", 2); break;
            case '\\': buffer_append(buf, "\\\\", 2); break;
            case '\b': buffer_append(buf, "\\b", 2); break;
            case '\f': buffer_append(buf, "\\f", 2); break;
            case '\n': buffer_append(buf, "\\n", 2); break;
            case '\r': buffer_append(buf, "\\r", 2); break;
            case '\t': buffer_append(buf, "\\t", 2); break;
            default:
                // Control character (0x00-0x1F)
                char escaped[7];
                snprintf(escaped, 7, "\\u%04x", (unsigned char)*ptr);
                buffer_append(buf, escaped, 6);
                break;
        }

        ptr++;
        start = ptr;
    }

    // Flush remaining
    if (ptr > start) {
        buffer_append(buf, start, ptr - start);
    }
}
```

**SIMD-accelerated escaping** (modern approach):

```c
// Check 32 bytes at once for characters needing escape
#include <immintrin.h>

bool needs_escaping_simd(const char* str, size_t len) {
    size_t i = 0;

    // Process 32 bytes at a time
    for (; i + 32 <= len; i += 32) {
        __m256i chunk = _mm256_loadu_si256((const __m256i*)(str + i));

        // Check for control chars (< 0x20)
        __m256i ctrl = _mm256_cmpgt_epi8(_mm256_set1_epi8(0x20), chunk);

        // Check for quote (0x22)
        __m256i quote = _mm256_cmpeq_epi8(chunk, _mm256_set1_epi8(0x22));

        // Check for backslash (0x5C)
        __m256i backslash = _mm256_cmpeq_epi8(chunk, _mm256_set1_epi8(0x5C));

        // Combine
        __m256i special = _mm256_or_si256(_mm256_or_si256(ctrl, quote), backslash);

        if (_mm256_movemask_epi8(special) != 0) {
            return true;  // Found character needing escape
        }
    }

    // Handle remaining bytes
    for (; i < len; i++) {
        if (str[i] < 0x20 || str[i] == '"' || str[i] == '\\') {
            return true;
        }
    }

    return false;
}
```

### 5.3 Number Serialization: Precision Considerations

Converting floating-point to string while preserving precision:

```c
// Using Grisu2 algorithm (fast but not always shortest)
void write_number_grisu2(Buffer* buf, double value) {
    if (value == 0.0) {
        buffer_append(buf, "0", 1);
        return;
    }

    if (value < 0.0) {
        buffer_append(buf, "-", 1);
        value = -value;
    }

    // Extract IEEE 754 components
    uint64_t bits = *(uint64_t*)&value;
    int exponent = ((bits >> 52) & 0x7FF) - 1023;
    uint64_t mantissa = (bits & 0xFFFFFFFFFFFFF) | 0x10000000000000;

    // Generate decimal representation
    char digits[32];
    int decimal_exponent;
    grisu2_algorithm(mantissa, exponent, digits, &decimal_exponent);

    // Format with appropriate notation
    format_decimal(buf, digits, decimal_exponent);
}

// Using Ryu algorithm (fastest shortest representation)
void write_number_ryu(Buffer* buf, double value) {
    char result[25];  // Max length for double
    int len = d2s_buffered_n(value, result);
    buffer_append(buf, result, len);
}
```

**Precision loss example:**

```c
double original = 0.1 + 0.2;  // 0.30000000000000004 (IEEE 754)

// Naive serialization
sprintf(buffer, "%f", original);  // "0.300000"

// Correct serialization (shortest roundtrip)
sprintf(buffer, "%.17g", original);  // "0.30000000000000004"

// Round-trip test
double roundtrip = strtod("0.30000000000000004", NULL);
assert(roundtrip == original);  // Passes
```

### 5.4 Buffered Writing

Minimize syscalls by batching writes:

```c
typedef struct {
    char* data;
    size_t size;
    size_t capacity;
    int fd;  // File descriptor or -1 for in-memory
} Buffer;

void buffer_append(Buffer* buf, const char* data, size_t len) {
    // Ensure capacity
    if (buf->size + len > buf->capacity) {
        if (buf->fd >= 0) {
            // Flush to file
            write(buf->fd, buf->data, buf->size);
            buf->size = 0;

            if (len > buf->capacity) {
                // Direct write for large chunks
                write(buf->fd, data, len);
                return;
            }
        } else {
            // Grow buffer (in-memory)
            size_t new_cap = buf->capacity * 2;
            while (new_cap < buf->size + len) {
                new_cap *= 2;
            }
            buf->data = realloc(buf->data, new_cap);
            buf->capacity = new_cap;
        }
    }

    memcpy(buf->data + buf->size, data, len);
    buf->size += len;
}

void buffer_flush(Buffer* buf) {
    if (buf->fd >= 0 && buf->size > 0) {
        write(buf->fd, buf->data, buf->size);
        buf->size = 0;
    }
}
```

**Performance impact:**

```
Without buffering:
  1000 object serializations = 5000+ write() syscalls
  Syscall overhead: ~300ns each = 1.5ms total

With buffering (16KB buffer):
  1000 object serializations = ~10 write() syscalls
  Syscall overhead: ~300ns × 10 = 3μs total

Speedup: 500×
```

### 5.5 Pretty-Printing & Formatting

Adding whitespace for human readability:

```c
void serialize_pretty(JsonValue* v, Buffer* buf, int indent) {
    switch (v->type) {
        case JSON_OBJECT:
            buffer_append(buf, "{\n", 2);
            for (size_t i = 0; i < v->data.object.size; i++) {
                if (i > 0) buffer_append(buf, ",\n", 2);

                // Indentation
                for (int j = 0; j < indent + 2; j++) {
                    buffer_append(buf, " ", 1);
                }

                // Key
                buffer_append(buf, "\"", 1);
                write_escaped_string(buf, v->data.object.keys[i]);
                buffer_append(buf, "\": ", 3);

                // Value (recursive with increased indent)
                serialize_pretty(v->data.object.values[i], buf, indent + 2);
            }
            buffer_append(buf, "\n", 1);
            for (int j = 0; j < indent; j++) {
                buffer_append(buf, " ", 1);
            }
            buffer_append(buf, "}", 1);
            break;

        // ... similar for arrays
    }
}
```

**Size comparison:**

```json
// Compact: 45 bytes
{"name":"Alice","age":25,"city":"NYC"}

// Pretty-printed: 78 bytes (+73%)
{
  "name": "Alice",
  "age": 25,
  "city": "NYC"
}
```

---

<a name="network-transport"></a>
## 6. Network Transport & Wire Format

### 6.1 HTTP/REST Transport

Typical HTTP exchange with JSON payload:

```http
POST /api/users HTTP/1.1
Host: example.com
Content-Type: application/json
Content-Length: 57
Accept: application/json
Connection: keep-alive

{"name":"Alice","email":"alice@example.com","age":25}
```

**TCP/IP stack traversal:**

```
Application Layer:  JSON string in memory
       ↓ (write() syscall)
Transport Layer:    TCP segment (MSS ~1460 bytes)
       ↓
Network Layer:      IP packet (MTU ~1500 bytes)
       ↓
Data Link Layer:    Ethernet frame (1518 bytes)
       ↓
Physical Layer:     Electrical signals on wire
```

**Fragmentation consideration:**

```
JSON payload: 10KB
TCP MSS: 1460 bytes

Fragmentation:
  Segment 1: 1460 bytes
  Segment 2: 1460 bytes
  Segment 3: 1460 bytes
  Segment 4: 1460 bytes
  Segment 5: 1460 bytes
  Segment 6: 1460 bytes
  Segment 7: 740 bytes

Total: 7 TCP segments (7 IP packets)
```

### 6.2 WebSocket Streaming

Sending JSON over WebSocket:

```
WebSocket Frame Structure:
┌─────────┬─────────┬──────────┬─────────────┬─────────┐
│ FIN=1   │ RSV=000 │ Opcode=1 │ Mask=1      │ Len=45  │
│ 1 bit   │ 3 bits  │ 4 bits   │ 1 bit       │ 7 bits  │
├─────────┴─────────┴──────────┴─────────────┴─────────┤
│ Masking Key (32 bits)                                │
├──────────────────────────────────────────────────────┤
│ Payload (JSON, masked)                               │
│ {"name":"Alice","age":25}                            │
└──────────────────────────────────────────────────────┘
```

**Masking operation:**

```c
void mask_payload(uint8_t* payload, size_t len, uint32_t mask_key) {
    uint8_t* key_bytes = (uint8_t*)&mask_key;

    for (size_t i = 0; i < len; i++) {
        payload[i] ^= key_bytes[i % 4];
    }
}
```

**Streaming large JSON:**

```c
// Send array of objects incrementally
void stream_json_array(WebSocket* ws, Iterator* it) {
    ws_send_text(ws, "[", 1);

    bool first = true;
    while (has_next(it)) {
        if (!first) {
            ws_send_text(ws, ",", 1);
        }
        first = false;

        JsonValue* obj = next(it);
        Buffer buf = {0};
        serialize_value(obj, &buf);
        ws_send_text(ws, buf.data, buf.size);
        buffer_free(&buf);
    }

    ws_send_text(ws, "]", 1);
}
```

### 6.3 Compression

JSON is highly compressible due to repetition:

```
Original JSON (100KB):
[
  {"id":1,"name":"Alice","email":"alice@example.com"},
  {"id":2,"name":"Bob","email":"bob@example.com"},
  // ... 1000 similar objects
]

Gzip compressed (15KB):
- Compression ratio: 85%
- Repeated strings ("id", "name", "email") compressed via dictionary
- Numbers compressed via Huffman coding
```

**HTTP compression negotiation:**

```http
GET /api/data HTTP/1.1
Accept-Encoding: gzip, deflate, br

HTTP/1.1 200 OK
Content-Type: application/json
Content-Encoding: gzip
Content-Length: 15360
```

**Compression performance trade-offs:**

| Algorithm | Ratio | Compress Speed | Decompress Speed |
|-----------|-------|----------------|------------------|
| None      | 1.0   | N/A            | N/A              |
| Gzip (6)  | 6.5   | 20 MB/s        | 200 MB/s         |
| Brotli (6)| 7.2   | 10 MB/s        | 150 MB/s         |
| Zstd (3)  | 6.8   | 400 MB/s       | 800 MB/s         |

### 6.4 Chunked Transfer Encoding

Streaming JSON without knowing size in advance:

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: application/json

5\r\n
[{"id\r\n
7\r\n
":1,"na\r\n
8\r\n
me":"Ali\r\n
4\r\n
ce"}\r\n
0\r\n
\r\n
```

**Chunked encoding pattern:**

```c
void send_chunked_json(int socket, JsonValue* root) {
    // Start chunked response
    write(socket, "HTTP/1.1 200 OK\r\n", 17);
    write(socket, "Transfer-Encoding: chunked\r\n", 28);
    write(socket, "Content-Type: application/json\r\n\r\n", 34);

    // Serialize in chunks
    Buffer chunk = {.capacity = 4096};
    chunk.data = malloc(chunk.capacity);

    serialize_value_chunked(root, &chunk, socket);

    // Send final chunk
    write(socket, "0\r\n\r\n", 5);

    free(chunk.data);
}

void flush_chunk(Buffer* chunk, int socket) {
    // Send chunk size in hex
    char size_hex[16];
    snprintf(size_hex, 16, "%zx\r\n", chunk->size);
    write(socket, size_hex, strlen(size_hex));

    // Send chunk data
    write(socket, chunk->data, chunk->size);
    write(socket, "\r\n", 2);

    chunk->size = 0;
}
```

---

<a name="performance-characteristics"></a>
## 7. Performance Characteristics & Optimization

### 7.1 Parsing Benchmarks

**Typical parsing speeds** (on modern CPU, 3GHz):

| Library      | Language | Speed (MB/s) | Memory Overhead |
|--------------|----------|--------------|-----------------|
| simdjson     | C++      | 2,500        | 1.2×            |
| RapidJSON    | C++      | 800          | 1.5×            |
| ujson        | Python/C | 500          | 2.0×            |
| json (std)   | Python   | 50           | 3.0×            |
| serde_json   | Rust     | 600          | 1.3×            |
| JSON.parse() | JS (V8)  | 400          | 2.5×            |
| Jackson      | Java     | 200          | 4.0×            |
| Gson         | Java     | 80           | 5.0×            |

**Bottleneck analysis** (profiling typical parser):

```
Total time: 100ms parsing 10MB JSON

Breakdown:
├─ 40ms (40%): String validation & unescaping
├─ 25ms (25%): Memory allocation
├─ 20ms (20%): Hash table insertions (for objects)
├─ 10ms (10%): Number parsing
└─ 5ms  (5%):  Structural parsing (brackets, braces)
```

### 7.2 Optimization Techniques

#### Technique 1: Avoid Allocations

**Before optimization:**

```c
// Allocates new string for each unescape operation
char* unescape_string(const char* str, size_t len) {
    char* result = malloc(len + 1);  // Worst case: no escapes
    // ... unescape logic
    return result;
}
```

**After optimization:**

```c
// In-place unescaping (input buffer modified)
size_t unescape_inplace(char* str, size_t len) {
    char* read = str;
    char* write = str;

    while (read < str + len) {
        if (*read == '\\') {
            read++;
            *write++ = unescape_char(*read++);
        } else {
            *write++ = *read++;
        }
    }

    return write - str;  // New length
}
```

#### Technique 2: SIMD String Validation

**Scalar UTF-8 validation:**

```c
bool validate_utf8_scalar(const char* str, size_t len) {
    for (size_t i = 0; i < len; ) {
        uint8_t byte = str[i];

        if (byte < 0x80) {
            i++;  // ASCII
        } else if ((byte & 0xE0) == 0xC0) {
            // 2-byte sequence
            if (i + 1 >= len) return false;
            if ((str[i+1] & 0xC0) != 0x80) return false;
            i += 2;
        } else if ((byte & 0xF0) == 0xE0) {
            // 3-byte sequence
            if (i + 2 >= len) return false;
            if ((str[i+1] & 0xC0) != 0x80) return false;
            if ((str[i+2] & 0xC0) != 0x80) return false;
            i += 3;
        } else if ((byte & 0xF8) == 0xF0) {
            // 4-byte sequence
            if (i + 3 >= len) return false;
            // ... validation
            i += 4;
        } else {
            return false;
        }
    }
    return true;
}
```

**SIMD UTF-8 validation** (processing 64 bytes/iteration):

```c
// From simdjson's UTF-8 validation
bool validate_utf8_simd(const uint8_t* data, size_t len) {
    // State machine using SIMD lookup tables
    __m256i prev_input = _mm256_set1_epi8(0);
    __m256i prev_first_len = _mm256_set1_epi8(0);

    for (size_t i = 0; i < len; i += 32) {
        __m256i input = _mm256_loadu_si256((const __m256i*)(data + i));

        // Classify bytes (ASCII, 2-byte start, 3-byte start, etc.)
        __m256i first_len = classify_bytes(input);

        // Check continuation bytes
        __m256i range = continuation_lengths(prev_first_len);

        // Validate using lookup table
        if (!check_valid(input, range)) {
            return false;
        }

        prev_input = input;
        prev_first_len = first_len;
    }

    return true;
}
```

**Speedup: 10-15× faster than scalar validation.**

#### Technique 3: Perfect Hash for Keywords

**Standard keyword matching:**

```c
JsonType match_keyword(const char* str, size_t len) {
    if (len == 4) {
        if (memcmp(str, "true", 4) == 0) return JSON_TRUE;
        if (memcmp(str, "null", 4) == 0) return JSON_NULL;
    } else if (len == 5) {
        if (memcmp(str, "false", 5) == 0) return JSON_FALSE;
    }
    return JSON_ERROR;
}
```

**Perfect hash (no collisions):**

```c
// Perfect hash function for {true, false, null}
// Hash = (first_char * 7 + length) % 11

JsonType match_keyword_perfect(const char* str, size_t len) {
    static const JsonType table[11] = {
        [2] = JSON_NULL,   // hash("null") = 2
        [4] = JSON_TRUE,   // hash("true") = 4
        [7] = JSON_FALSE,  // hash("false") = 7
    };

    size_t hash = (str[0] * 7 + len) % 11;
    JsonType type = table[hash];

    // Verify (to handle hash collisions in larger sets)
    if (verify_keyword(str, len, type)) {
        return type;
    }

    return JSON_ERROR;
}
```

#### Technique 4: Branch Prediction Hints

```c
// Likely/unlikely macros
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

JsonValue* parse_value(Lexer* lexer) {
    Token tok = peek(lexer);

    // Most JSON is objects/arrays, hint to CPU
    if (likely(tok == T_LBRACE || tok == T_LBRACKET)) {
        return parse_container(lexer);
    }

    // Error path is unlikely
    if (unlikely(tok == T_ERROR)) {
        return NULL;
    }

    return parse_primitive(lexer);
}
```

**CPU impact:**

- Without hints: ~15% branch mispredictions
- With hints: ~3% branch mispredictions
- Performance gain: 5-10% overall

### 7.3 Memory Pool Allocation

**Custom allocator for JSON nodes:**

```c
typedef struct Pool {
    void* blocks[256];
    size_t block_count;
    size_t current_offset;
    size_t block_size;
} Pool;

Pool* pool_create(size_t block_size) {
    Pool* p = malloc(sizeof(Pool));
    p->block_size = block_size;
    p->block_count = 1;
    p->blocks[0] = malloc(block_size);
    p->current_offset = 0;
    return p;
}

void* pool_alloc(Pool* p, size_t size) {
    // Align to 8 bytes
    size = (size + 7) & ~7;

    // Check current block
    if (p->current_offset + size > p->block_size) {
        // Allocate new block
        if (p->block_count >= 256) {
            return NULL;  // Pool exhausted
        }
        p->blocks[p->block_count++] = malloc(p->block_size);
        p->current_offset = 0;
    }

    void* ptr = (char*)p->blocks[p->block_count - 1] + p->current_offset;
    p->current_offset += size;
    return ptr;
}

void pool_destroy(Pool* p) {
    for (size_t i = 0; i < p->block_count; i++) {
        free(p->blocks[i]);
    }
    free(p);
}
```

**Performance comparison:**

```c
// Standard malloc/free
for (int i = 0; i < 10000; i++) {
    JsonValue* v = malloc(sizeof(JsonValue));
    // ... use v
    free(v);
}
// Time: 850μs

// Pool allocation
Pool* pool = pool_create(64 * 1024);
for (int i = 0; i < 10000; i++) {
    JsonValue* v = pool_alloc(pool, sizeof(JsonValue));
    // ... use v
}
pool_destroy(pool);
// Time: 45μs (19× faster)
```

### 7.4 Lazy Parsing

**Concept**: Don't parse entire document, only accessed paths.

```c
// Represents a JSON value that hasn't been parsed yet
typedef struct {
    const char* start;  // Pointer to raw JSON
    const char* end;
    JsonType type;      // Determined from first char
    bool parsed;
    union {
        JsonValue* value;  // Parsed representation
        void* metadata;    // Indices for fast access
    };
} LazyValue;

LazyValue* create_lazy(const char* json, size_t len) {
    LazyValue* lv = malloc(sizeof(LazyValue));
    lv->start = json;
    lv->end = json + len;
    lv->type = detect_type(json[0]);
    lv->parsed = false;
    return lv;
}

// Only parse when accessed
JsonValue* lazy_get(LazyValue* lv, const char* key) {
    if (!lv->parsed) {
        // Scan for key without full parse
        const char* value_start = find_key_value(lv->start, lv->end, key);
        if (value_start) {
            // Parse only this value
            return parse_value_at(value_start);
        }
        return NULL;
    }

    return object_get(lv->value, key);
}
```

**Use case** (extracting single field from large JSON):

```c
// Full parse: 10ms for 1MB JSON
JsonValue* doc = parse_json(json_1mb);
JsonValue* name = object_get(doc, "user.name");

// Lazy parse: 0.1ms (100× faster)
LazyValue* doc = create_lazy(json_1mb, 1024*1024);
JsonValue* name = lazy_get(doc, "user.name");
```

---

<a name="security-considerations"></a>
## 8. Security Considerations & Attack Vectors

### 8.1 Denial of Service Attacks

#### Attack 1: Hash Collision DoS

**Vulnerability**: Predictable hash function allows attacker to craft keys that collide.

```json
{
  "key_0": 1,
  "key_collision_0": 2,
  "key_collision_collision_0": 3,
  // ... 100,000 colliding keys
}
```

**Impact:**
- Hash table degrades to O(n) lookup
- Single request can consume seconds of CPU time
- Memory exhaustion from long collision chains

**Mitigation:**

```c
// Use randomized hash seed (per-process or per-request)
uint32_t hash_string_secure(const char* str, size_t len, uint32_t seed) {
    uint32_t hash = seed;
    for (size_t i = 0; i < len; i++) {
        hash = ((hash << 5) + hash) ^ str[i];
    }
    return hash;
}

// Generate random seed on startup
uint32_t hash_seed = generate_random_seed();
```

**Alternative**: Use cryptographic hash (slower but collision-resistant):

```c
uint32_t hash_siphash(const char* str, size_t len, uint64_t key[2]) {
    // SipHash-2-4 implementation
    // ~2× slower but cryptographically secure
    return siphash24(str, len, key);
}
```

#### Attack 2: Deeply Nested Objects

**Attack payload:**

```json
{
  "a": {
    "b": {
      "c": {
        // ... 10,000 levels deep
```

**Impact:**
- Stack overflow (recursive parser)
- Excessive memory allocation
- CPU exhaustion

**Mitigation:**

```c
#define MAX_NESTING_DEPTH 1000

JsonValue* parse_value_safe(Lexer* lexer, int depth) {
    if (depth > MAX_NESTING_DEPTH) {
        return error("Nesting depth exceeded");
    }

    Token tok = peek(lexer);

    if (tok == T_LBRACE) {
        return parse_object_safe(lexer, depth + 1);
    }
    // ... etc
}
```

**Iterative parsing** (avoid stack overflow):

```c
// Use explicit stack instead of call stack
typedef struct {
    JsonValue* values[MAX_NESTING_DEPTH];
    int top;
} ParseStack;

JsonValue* parse_iterative(Lexer* lexer) {
    ParseStack stack = {.top = -1};

    while (true) {
        Token tok = next(lexer);

        switch (tok.type) {
            case T_LBRACE:
                if (++stack.top >= MAX_NESTING_DEPTH) {
                    return error("Too deep");
                }
                stack.values[stack.top] = create_object();
                break;

            case T_RBRACE:
                if (stack.top == 0) {
                    return stack.values[0];  // Done
                }
                JsonValue* obj = stack.values[stack.top--];
                object_add(stack.values[stack.top], obj);
                break;

            // ... handle other tokens
        }
    }
}
```

#### Attack 3: Large String Allocation

**Attack payload:**

```json
{
  "data": "AAAAAAAAAA... [10MB of 'A'] ..."
}
```

**Impact:**
- Memory exhaustion
- Allocation failure DoS
- Slowloris-style resource holding

**Mitigation:**

```c
#define MAX_STRING_LENGTH (1024 * 1024)  // 1MB

char* parse_string_limited(Lexer* lexer) {
    size_t len = 0;
    const char* start = lexer->pos;

    // Scan for end quote
    while (*lexer->pos != '"') {
        if (++len > MAX_STRING_LENGTH) {
            return error("String too large");
        }
        if (*lexer->pos == '\\') {
            lexer->pos++;  // Skip escaped char
        }
        lexer->pos++;
    }

    // Allocate and copy
    char* str = malloc(len + 1);
    memcpy(str, start, len);
    str[len] = '\0';
    return str; }
```

### 8.2 Injection Attacks

#### SQL Injection via JSON

**Vulnerable code:**

```c
// NEVER DO THIS
void insert_user(JsonValue* user) {
    const char* name = object_get_string(user, "name");
    const char* email = object_get_string(user, "email");

    char query[1024];
    sprintf(query, "INSERT INTO users (name, email) VALUES ('%s', '%s')",
            name, email);

    sql_execute(query);
}
```

**Attack:**

```json
{
  "name": "Alice'; DROP TABLE users; --",
  "email": "alice@example.com"
}
```

**Mitigation** (parameterized queries):

```c
void insert_user_safe(JsonValue* user) {
    const char* name = object_get_string(user, "name");
    const char* email = object_get_string(user, "email");

    PreparedStatement* stmt = sql_prepare(
        "INSERT INTO users (name, email) VALUES (?, ?)"
    );

    sql_bind_text(stmt, 1, name);
    sql_bind_text(stmt, 2, email);
    sql_execute(stmt);
    sql_finalize(stmt);
}
```

#### JavaScript Injection

**Vulnerable code:**

```html
<script>
var userData = <%= json_data %>;  // Server-side template
</script>
```

**Attack:**

```json
{
  "name": "</script><script>alert('XSS')</script><script>"
}
```

**Mitigation:**

```html
<script>
var userData = JSON.parse('<%= escape_json(json_data) %>');
</script>
```

```c
char* escape_json_for_html(const char* json) {
    // Escape: <, >, &, ', "
    // Also escape </script> patterns

    size_t len = strlen(json);
    char* escaped = malloc(len * 6 + 1);  // Worst case
    char* out = escaped;

    for (size_t i = 0; i < len; i++) {
        switch (json[i]) {
            case '<':  strcpy(out, "\\u003c"); out += 6; break;
            case '>':  strcpy(out, "\\u003e"); out += 6; break;
            case '&':  strcpy(out, "\\u0026"); out += 6; break;
            case '\'': strcpy(out, "\\u0027"); out += 6; break;
            case '"':  strcpy(out, "\\u0022"); out += 6; break;
            default:   *out++ = json[i];
        }
    }

    *out = '\0';
    return escaped;
}
```

### 8.3 Information Disclosure

#### Timing Attacks

**Vulnerable comparison:**

```c
bool verify_api_key(const char* provided, const char* expected) {
    return strcmp(provided, expected) == 0;  // Early exit on mismatch
}
```

**Attack**: Measure response time to guess key character-by-character.

**Mitigation** (constant-time comparison):

```c
bool verify_api_key_safe(const char* provided, const char* expected) {
    size_t provided_len = strlen(provided);
    size_t expected_len = strlen(expected);

    // Always compare full length
    size_t max_len = provided_len > expected_len ? provided_len : expected_len;

    volatile uint8_t diff = provided_len ^ expected_len;

    for (size_t i = 0; i < max_len; i++) {
        uint8_t a = i < provided_len ? provided[i] : 0;
        uint8_t b = i < expected_len ? expected[i] : 0;
        diff |= a ^ b;
    }

    return diff == 0;
}
```

#### Error Message Information Leakage

**Bad:**

```json
{
  "error": "Database connection failed: user 'admin'@'192.168.1.100' access denied for database 'production_db'"
}
```

**Good:**

```json
{
  "error": "Internal server error",
  "code": "ERR_DATABASE"
}
```

**Implementation:**

```c
JsonValue* create_error_response(ErrorCode code, const char* internal_msg) {
    JsonValue* err = create_object();

    // User-facing generic message
    object_set_string(err, "error", get_public_error_message(code));
    object_set_string(err, "code", get_error_code_string(code));

    // Log detailed internal error
    log_error("[%s] %s", get_error_code_string(code), internal_msg);

    return err;
}
```

---

<a name="comparative-analysis"></a>
## 9. Comparative Analysis: JSON vs Alternatives

### 9.1 JSON vs XML

**Structural comparison:**

```json
// JSON: 58 bytes
{"name":"Alice","age":25,"active":true}
```

```xml
<!-- XML: 86 bytes -->
<user>
  <name>Alice</name>
  <age>25</age>
  <active>true</active>
</user>
```

**Key differences:**

| Feature | JSON | XML |
|---------|------|-----|
| **Verbosity** | Compact | Verbose (40-60% larger) |
| **Types** | Native types | Everything is text |
| **Arrays** | Built-in `[]` | Must use repeated elements |
| **Attributes** | N/A | Attributes vs elements choice |
| **Namespaces** | No | Yes (`xmlns`) |
| **Schema** | Optional (JSON Schema) | XSD, DTD, RelaxNG |
| **Comments** | No | Yes `<!-- -->` |
| **Parse Speed** | Faster (2-5×) | Slower |

**When to use XML:**
- Document-centric data (mixed content)
- Strict validation requirements
- Legacy system integration
- Need for comments/processing instructions

### 9.2 JSON vs Protocol Buffers

**Protocol Buffers definition:**

```protobuf
message User {
  required string name = 1;
  required int32 age = 2;
  required bool active = 3;
}
```

**Binary encoding:**

```
JSON (58 bytes):
{"name":"Alice","age":25,"active":true}

Protobuf (14 bytes):
0A 05 41 6C 69 63 65 10 19 18 01

Breakdown:
0A       = field 1 (name), wire type 2 (length-delimited)
05       = length 5
41...65  = "Alice" in ASCII
10       = field 2 (age), wire type 0 (varint)
19       = 25 as varint
18       = field 3 (active), wire type 0
01       = true
```

**Performance comparison:**

| Metric | JSON | Protocol Buffers |
|--------|------|------------------|
| **Size** | 58B | 14B (76% smaller) |
| **Parse time** | 100ns | 20ns (5× faster) |
| **Schema** | Optional | Required |
| **Evolution** | Easy (additive) | Field numbers |
| **Human readable** | Yes | No |
| **Language support** | Universal | Good (code generation) |

### 9.3 JSON vs MessagePack

**MessagePack encoding:**

```
JSON (58 bytes):
{"name":"Alice","age":25,"active":true}

MessagePack (32 bytes):
83            = fixmap (3 elements)
A4 6E 61 6D 65  = fixstr "name" (length 4)
A5 41 6C 69 63 65 = fixstr "Alice" (length 5)
A3 61 67 65    = fixstr "age" (length 3)
19            = positive fixint 25
A6 61 63 74 69 76 65 = fixstr "active" (length 6)
C3            = true
```

**Characteristics:**

- **Size**: 30-50% smaller than JSON
- **Speed**: 2-3× faster parsing
- **Schema**: Not required (self-describing)
- **Compatibility**: Can round-trip with JSON
- **Trade-off**: Binary format (not human-readable)

### 9.4 JSON vs BSON

**BSON (MongoDB's format):**

```
BSON (61 bytes - larger than JSON!):
\x3D\x00\x00\x00              Document size (61 bytes)
\x02name\x00                   String type + field name
\x06\x00\x00\x00Alice\x00      Length + value + null
\x10age\x00                    Int32 type + field name
\x19\x00\x00\x00              Value (25)
\x08active\x00                 Boolean type + field name
\x01                          true
\x00                          End of document
```

**Why larger?**
- Explicit type tags (1 byte per field)
- 4-byte length prefixes
- Null terminators
- Padding for alignment

**Advantages:**
- **Fast traversal**: Don't need to parse to skip
- **In-place modification**: Can update values without rewriting
- **Rich types**: Date, Binary, ObjectId, etc.
- **Ordered**: Preserves field order

---

<a name="implementation-deep-dive"></a>
## 10. Implementation Deep-Dive: Building a JSON Parser

### 10.1 Complete Lexer Implementation

```c
// lexer.h
#ifndef LEXER_H
#define LEXER_H

#include <stddef.h>
#include <stdbool.h>

typedef enum {
    T_LBRACE,    // {
    T_RBRACE,    // }
    T_LBRACKET,  // [
    T_RBRACKET,  // ]
    T_COLON,     // :
    T_COMMA,     // ,
    T_STRING,    // "..."
    T_NUMBER,    // 123.45
    T_TRUE,      // true
    T_FALSE,     // false
    T_NULL,      // null
    T_EOF,       // End of input
    T_ERROR      // Parse error
} TokenType;

typedef struct {
    TokenType type;
    union {
        char* string;
        double number;
    } value;
    size_t line;
    size_t column;
} Token;

typedef struct {
    const char* input;
    size_t length;
    size_t position;
    size_t line;
    size_t column;
    char error_msg[256];
} Lexer;

void lexer_init(Lexer* lex, const char* input, size_t length);
Token lexer_next(Lexer* lex);
Token lexer_peek(Lexer* lex);
void token_free(Token* tok);

#endif
```

```c
// lexer.c
#include "lexer.h"
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <math.h>

void lexer_init(Lexer* lex, const char* input, size_t length) {
    lex->input = input;
    lex->length = length;
    lex->position = 0;
    lex->line = 1;
    lex->column = 1;
    lex->error_msg[0] = '\0';
}

static char peek_char(Lexer* lex, size_t offset) {
    size_t pos = lex->position + offset;
    return pos < lex->length ? lex->input[pos] : '\0';
}

static char advance(Lexer* lex) {
    if (lex->position >= lex->length) {
        return '\0';
    }

    char c = lex->input[lex->position++];

    if (c == '\n') {
        lex->line++;
        lex->column = 1;
    } else {
        lex->column++;
    }

    return c;
}

static void skip_whitespace(Lexer* lex) {
    while (lex->position < lex->length) {
        char c = peek_char(lex, 0);
        if (c == ' ' || c == '\t' || c == '\n' || c == '\r') {
            advance(lex);
        } else {
            break;
        }
    }
}

static Token make_token(TokenType type) {
    Token tok = {.type = type, .line = 0, .column = 0};
    return tok;
}

static Token make_error(Lexer* lex, const char* message) {
    Token tok = {.type = T_ERROR, .line = lex->line, .column = lex->column};
    snprintf(lex->error_msg, sizeof(lex->error_msg), "%s", message);
    return tok;
}

static int hex_to_int(char c) {
    if (c >= '0' && c <= '9') return c - '0';
    if (c >= 'a' && c <= 'f') return 10 + (c - 'a');
    if (c >= 'A' && c <= 'F') return 10 + (c - 'A');
    return -1;
}

static Token parse_string(Lexer* lex) {
    advance(lex);  // Skip opening quote

    char buffer[4096];
    size_t len = 0;

    while (true) {
        char c = peek_char(lex, 0);

        if (c == '\0') {
            return make_error(lex, "Unterminated string");
        }

        if (c == '"') {
            advance(lex);
            break;
        }

        if (c == '\\') {
            advance(lex);
            char escaped = peek_char(lex, 0);
            advance(lex);

            switch (escaped) {
                case '"':  buffer[len++] = '"'; break;
                case '\\': buffer[len++] = '\\'; break;
                case '/':  buffer[len++] = '/'; break;
                case 'b':  buffer[len++] = '\b'; break;
                case 'f':  buffer[len++] = '\f'; break;
                case 'n':  buffer[len++] = '\n'; break;
                case 'r':  buffer[len++] = '\r'; break;
                case 't':  buffer[len++] = '\t'; break;
                case 'u': {
                    // Unicode escape \uXXXX
                    int codepoint = 0;
                    for (int i = 0; i < 4; i++) {
                        int hex = hex_to_int(peek_char(lex, 0));
                        if (hex == -1) {
                            return make_error(lex, "Invalid unicode escape");
                        }
                        codepoint = (codepoint << 4) | hex;
                        advance(lex);
                    }

                    // Convert codepoint to UTF-8
                    if (codepoint < 0x80) {
                        buffer[len++] = (char)codepoint;
                    } else if (codepoint < 0x800) {
                        buffer[len++] = 0xC0 | (codepoint >> 6);
                        buffer[len++] = 0x80 | (codepoint & 0x3F);
                    } else {
                        buffer[len++] = 0xE0 | (codepoint >> 12);
                        buffer[len++] = 0x80 | ((codepoint >> 6) & 0x3F);
                        buffer[len++] = 0x80 | (codepoint & 0x3F);
                    }
                    break;
                }
                default:
                    return make_error(lex, "Invalid escape sequence");
            }
        } else {
            buffer[len++] = c;
            advance(lex);
        }

        if (len >= sizeof(buffer) - 1) {
            return make_error(lex, "String too long");
        }
    }

    buffer[len] = '\0';

    Token tok = make_token(T_STRING);
    tok.value.string = strdup(buffer);
    return tok;
}

static Token parse_number(Lexer* lex) {
    const char* start = lex->input + lex->position;
    const char* p = start;

    // Optional minus
    if (*p == '-') p++;

    // Integer part
    if (*p == '0') {
        p++;
    } else if (*p >= '1' && *p <= '9') {
        while (*p >= '0' && *p <= '9') p++;
    } else {
        return make_error(lex, "Invalid number");
    }

    // Fractional part
    if (*p == '.') {
        p++;
        if (*p < '0' || *p > '9') {
            return make_error(lex, "Invalid number: expected digit after '.'");
        }
        while (*p >= '0' && *p <= '9') p++;
    }

    // Exponent
    if (*p == 'e' || *p == 'E') {
        p++;
        if (*p == '+' || *p == '-') p++;
        if (*p < '0' || *p > '9') {
            return make_error(lex, "Invalid number: expected digit in exponent");
        }
        while (*p >= '0' && *p <= '9') p++;
    }

    // Parse the number
    char* end;
    double value = strtod(start, &end);

    if (end != p) {
        return make_error(lex, "Invalid number format");
    }

    // Advance lexer position
    lex->position += (p - start);
    lex->column += (p - start);

    Token tok = make_token(T_NUMBER);
    tok.value.number = value;
    return tok;
}

static Token parse_keyword(Lexer* lex, const char* keyword, TokenType type) {
    size_t len = strlen(keyword);

    for (size_t i = 0; i < len; i++) {
        if (peek_char(lex, i) != keyword[i]) {
            return make_error(lex, "Invalid keyword");
        }
    }

    // Ensure keyword is not followed by alphanumeric
    char next = peek_char(lex, len);
    if (isalnum(next) || next == '_') {
        return make_error(lex, "Invalid keyword");
    }

    for (size_t i = 0; i < len; i++) {
        advance(lex);
    }

    return make_token(type);
}

Token lexer_next(Lexer* lex) {
    skip_whitespace(lex);

    if (lex->position >= lex->length) {
        return make_token(T_EOF);
    }

    char c = peek_char(lex, 0);

    switch (c) {
        case '{': advance(lex); return make_token(T_LBRACE);
        case '}': advance(lex); return make_token(T_RBRACE);
        case '[': advance(lex); return make_token(T_LBRACKET);
        case ']': advance(lex); return make_token(T_RBRACKET);
        case ':': advance(lex); return make_token(T_COLON);
        case ',': advance(lex); return make_token(T_COMMA);
        case '"': return parse_string(lex);
        case 't': return parse_keyword(lex, "true", T_TRUE);
        case 'f': return parse_keyword(lex, "false", T_FALSE);
        case 'n': return parse_keyword(lex, "null", T_NULL);
        case '-':
        case '0': case '1': case '2': case '3': case '4':
        case '5': case '6': case '7': case '8': case '9':
            return parse_number(lex);
        default:
            return make_error(lex, "Unexpected character");
    }
}

void token_free(Token* tok) {
    if (tok->type == T_STRING && tok->value.string) {
        free(tok->value.string);
        tok->value.string = NULL;
    }
}
```

### 10.2 Complete Parser Implementation

```c
// parser.h
#ifndef PARSER_H
#define PARSER_H

#include "lexer.h"

typedef enum {
    JSON_NULL,
    JSON_BOOLEAN,
    JSON_NUMBER,
    JSON_STRING,
    JSON_ARRAY,
    JSON_OBJECT
} JsonType;

typedef struct JsonValue JsonValue;

typedef struct {
    char* key;
    JsonValue* value;
} JsonPair;

struct JsonValue {
    JsonType type;
    union {
        bool boolean;
        double number;
        char* string;
        struct {
            JsonValue** elements;
            size_t count;
            size_t capacity;
        } array;
        struct {
            JsonPair* pairs;
            size_t count;
            size_t capacity;
        } object;
    } as;
};

JsonValue* json_parse(const char* input, size_t length);
void json_free(JsonValue* value);
void json_print(JsonValue* value, int indent);

#endif
```

```c
// parser.c
#include "parser.h"
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

typedef struct {
    Lexer lexer;
    Token current;
    Token peek;
    bool has_error;
} Parser;

static JsonValue* parse_value(Parser* p);

static void parser_init(Parser* p, const char* input, size_t length) {
    lexer_init(&p->lexer, input, length);
    p->current = lexer_next(&p->lexer);
    p->peek = lexer_next(&p->lexer);
    p->has_error = false;
}

static Token advance(Parser* p) {
    Token prev = p->current;
    p->current = p->peek;
    p->peek = lexer_next(&p->lexer);
    return prev;
}

static bool check(Parser* p, TokenType type) {
    return p->current.type == type;
}

static bool match(Parser* p, TokenType type) {
    if (check(p, type)) {
        advance(p);
        return true;
    }
    return false;
}

static Token expect(Parser* p, TokenType type, const char* message) {
    if (check(p, type)) {
        return advance(p);
    }

    fprintf(stderr, "Parse error at line %zu, column %zu: %s\n",
            p->current.line, p->current.column, message);
    p->has_error = true;

    Token error = {.type = T_ERROR};
    return error;
}

static JsonValue* json_create(JsonType type) {
    JsonValue* value = calloc(1, sizeof(JsonValue));
    value->type = type;
    return value;
}

static JsonValue* parse_null(Parser* p) {
    expect(p, T_NULL, "Expected 'null'");
    return json_create(JSON_NULL);
}

static JsonValue* parse_boolean(Parser* p) {
    JsonValue* value = json_create(JSON_BOOLEAN);

    if (match(p, T_TRUE)) {
        value->as.boolean = true;
    } else if (match(p, T_FALSE)) {
        value->as.boolean = false;
    }

    return value;
}

static JsonValue* parse_number(Parser* p) {
    Token tok = expect(p, T_NUMBER, "Expected number");

    JsonValue* value = json_create(JSON_NUMBER);
    value->as.number = tok.value.number;

    return value;
}

static JsonValue* parse_string(Parser* p) {
    Token tok = expect(p, T_STRING, "Expected string");

    JsonValue* value = json_create(JSON_STRING);
    value->as.string = tok.value.string;  // Transfer ownership

    return value;
}

static JsonValue* parse_array(Parser* p) {
    expect(p, T_LBRACKET, "Expected '['");

    JsonValue* array = json_create(JSON_ARRAY);
    array->as.array.capacity = 8;
    array->as.array.elements = malloc(sizeof(JsonValue*) * array->as.array.capacity);
    array->as.array.count = 0;

    if (match(p, T_RBRACKET)) {
        return array;  // Empty array
    }

    while (true) {
        JsonValue* element = parse_value(p);

        if (p->has_error) {
            json_free(array);
            return NULL;
        }

        // Grow array if needed
        if (array->as.array.count >= array->as.array.capacity) {
            array->as.array.capacity *= 2;
            array->as.array.elements = realloc(
                array->as.array.elements,
                sizeof(JsonValue*) * array->as.array.capacity
            );
        }

        array->as.array.elements[array->as.array.count++] = element;

        if (match(p, T_RBRACKET)) {
            break;
        }

        expect(p, T_COMMA, "Expected ',' or ']'");
    }

    return array;
}

static JsonValue* parse_object(Parser* p) {
    expect(p, T_LBRACE, "Expected '{'");

    JsonValue* object = json_create(JSON_OBJECT);
    object->as.object.capacity = 8;
    object->as.object.pairs = malloc(sizeof(JsonPair) * object->as.object.capacity);
    object->as.object.count = 0;

    if (match(p, T_RBRACE)) {
        return object;  // Empty object
    }

    while (true) {
        Token key_tok = expect(p, T_STRING, "Expected string key");

        if (p->has_error) {
            json_free(object);
            return NULL;
        }

        expect(p, T_COLON, "Expected ':'");

        JsonValue* value = parse_value(p);

        if (p->has_error) {
            free(key_tok.value.string);
            json_free(object);
            return NULL;
        }

        // Grow object if needed
        if (object->as.object.count >= object->as.object.capacity) {
            object->as.object.capacity *= 2;
            object->as.object.pairs = realloc(
                object->as.object.pairs,
                sizeof(JsonPair) * object->as.object.capacity
            );
        }

        object->as.object.pairs[object->as.object.count].key = key_tok.value.string;
        object->as.object.pairs[object->as.object.count].value = value;
        object->as.object.count++;

        if (match(p, T_RBRACE)) {
            break;
        }

        expect(p, T_COMMA, "Expected ',' or '}'");
    }

    return object;
}

static JsonValue* parse_value(Parser* p) {
    switch (p->current.type) {
        case T_NULL:     return parse_null(p);
        case T_TRUE:
        case T_FALSE:    return parse_boolean(p);
        case T_NUMBER:   return parse_number(p);
        case T_STRING:   return parse_string(p);
        case T_LBRACKET: return parse_array(p);
        case T_LBRACE:   return parse_object(p);
        default:
            fprintf(stderr, "Unexpected token type %d\n", p->current.type);
            p->has_error = true;
            return NULL;
    }
}

JsonValue* json_parse(const char* input, size_t length) {
    Parser p;
    parser_init(&p, input, length);

    JsonValue* root = parse_value(&p);

    if (p.has_error) {
        json_free(root);
        return NULL;
    }

    if (!check(&p, T_EOF)) {
        fprintf(stderr, "Unexpected data after JSON value\n");
        json_free(root);
        return NULL;
    }

    return root;
}

void json_free(JsonValue* value) {
    if (!value) return;

    switch (value->type) {
        case JSON_STRING:
            free(value->as.string);
            break;

        case JSON_ARRAY:
            for (size_t i = 0; i < value->as.array.count; i++) {
                json_free(value->as.array.elements[i]);
            }
            free(value->as.array.elements);
            break;

        case JSON_OBJECT:
            for (size_t i = 0; i < value->as.object.count; i++) {
                free(value->as.object.pairs[i].key);
                json_free(value->as.object.pairs[i].value);
            }
            free(value->as.object.pairs);
            break;

        default:
            break;
    }

    free(value);
}

void json_print(JsonValue* value, int indent) {
    if (!value) return;

    switch (value->type) {
        case JSON_NULL:
            printf("null");
            break;

        case JSON_BOOLEAN:
            printf("%s", value->as.boolean ? "true" : "false");
            break;

        case JSON_NUMBER:
            printf("%g", value->as.number);
            break;

        case JSON_STRING:
            printf("\"%s\"", value->as.string);
            break;

        case JSON_ARRAY:
            printf("[\n");
            for (size_t i = 0; i < value->as.array.count; i++) {
                for (int j = 0; j < indent + 2; j++) printf(" ");
                json_print(value->as.array.elements[i], indent + 2);
                if (i < value->as.array.count - 1) printf(",");
                printf("\n");
            }
            for (int j = 0; j < indent; j++) printf(" ");
            printf("]");
            break;

        case JSON_OBJECT:
            printf("{\n");
            for (size_t i = 0; i < value->as.object.count; i++) {
                for (int j = 0; j < indent + 2; j++) printf(" ");
                printf("\"%s\": ", value->as.object.pairs[i].key);
                json_print(value->as.object.pairs[i].value, indent + 2);
                if (i < value->as.object.count - 1) printf(",");
                printf("\n");
            }
            for (int j = 0; j < indent; j++) printf(" ");
            printf("}");
            break;
    }
}
```

### 10.3 Test Program

```c
// main.c
#include "parser.h"
#include <stdio.h>
#include <string.h>

int main() {
    const char* json_text =
        "{\n"
        "  \"name\": \"Alice\",\n"
        "  \"age\": 25,\n"
        "  \"active\": true,\n"
        "  \"scores\": [98, 87, 92],\n"
        "  \"metadata\": {\n"
        "    \"created\": \"2025-01-17\",\n"
        "    \"tags\": [\"user\", \"premium\"]\n"
        "  }\n"
        "}";

    printf("Parsing JSON:\n%s\n\n", json_text);

    JsonValue* root = json_parse(json_text, strlen(json_text));

    if (root) {
        printf("Parsed successfully!\n\n");
        printf("Pretty print:\n");
        json_print(root, 0);
        printf("\n");

        json_free(root);
    } else {
        printf("Parse failed!\n");
        return 1;
    }

    return 0;
}
```

**Compilation:**

```bash
gcc -O2 -Wall -Wextra -std=c99 \
    lexer.c parser.c main.c \
    -o json_parser

./json_parser
```

---

## Conclusion

JSON's apparent simplicity belies a complex ecosystem of trade-offs, optimizations, and security considerations. From the lexical analysis of UTF-8 byte streams through SIMD-accelerated parsing to memory-efficient runtime representations, every layer of the stack involves careful engineering decisions.

**Key takeaways:**

1. **Grammar simplicity enables performance**: The context-free grammar allows for efficient single-pass parsing
2. **Memory representation dominates performance**: Choice of data structures (hash tables, arrays, arenas) has 10-100× impact
3. **Security is paramount**: Input validation, depth limits, and DoS prevention are non-negotiable
4. **Format choice matters**: JSON trades size/speed for universality and human readability
5. **Modern parsers use SIMD**: Processing 32-64 bytes per instruction achieves gigabytes/second throughput

The JSON parser we've built here represents fundamental patterns applicable across all text-processing domains: tokenization, recursive descent parsing, arena allocation, and defensive programming. Understanding these mechanics from first principles transforms JSON from a "simple data format" into a window into the deeper patterns of systems programming.
