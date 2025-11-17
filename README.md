# ORT (Object Record Table) Format Specification

Version: 1.0.1
Last Updated: 2025

## Table of Contents

- [1. Introduction](#1-introduction)
- [2. Design Philosophy](#2-design-philosophy)
- [3. Lexical Structure](#3-lexical-structure)
- [4. Data Types](#4-data-types)
- [5. Syntax Rules](#5-syntax-rules)
- [6. Header Syntax](#6-header-syntax)
- [7. Data Line Syntax](#7-data-line-syntax)
- [8. Nested Structures](#8-nested-structures)
- [9. Escape Sequences](#9-escape-sequences)
- [10. Comments](#10-comments)
- [11. Complete Examples](#11-complete-examples)
- [12. Parsing Rules](#12-parsing-rules)
- [13. Best Practices](#13-best-practices)

---

## 1. Introduction

### 1.1 Overview

Object Record Table (ORT) is a CSV-like structured data format designed specifically for token optimization in Large Language Model (LLM) contexts. Unlike traditional human-readable formats like JSON and YAML, ORT prioritizes computational efficiency while maintaining readability.

### 1.2 Goals

- **Token Efficiency**: Minimize the number of tokens required to represent structured data
- **Structural Clarity**: Maintain clear relationships between data elements
- **Native Support**: Support objects and arrays as first-class data structures
- **Simplicity**: Keep the syntax simple and predictable

### 1.3 Use Cases

ORT is ideal for:
- Data interchange with LLMs
- Uniform data structures with multiple records
- Configuration files for AI applications
- Structured data logging

ORT is NOT ideal for:
- Heterogeneous data structures
- Direct application I/O operations
- Human-editable configuration files requiring comments throughout

---

## 2. Design Philosophy

### 2.1 Token Optimization

ORT achieves token efficiency through:
1. **Header-based field definitions**: Field names declared once in header
2. **Positional value mapping**: Data lines contain only values
3. **Minimal delimiters**: Using only essential punctuation
4. **No redundant whitespace**: Compact representation

### 2.2 Comparison with Other Formats

```ort
# ORT Format (110 characters, 35 tokens)
users:id,profile(name,age,address(city,country)):
1,(John Doe,30,(New York,USA))
2,(Jane Smith,25,(London,UK))
```

```json
// JSON Format (398 characters, 118 tokens)
{
  "users": [
    {
      "id": 1,
      "profile": {
        "name": "John Doe",
        "age": 30,
        "address": {
          "city": "New York",
          "country": "USA"
        }
      }
    },
    {
      "id": 2,
      "profile": {
        "name": "Jane Smith",
        "age": 25,
        "address": {
          "city": "London",
          "country": "UK"
        }
      }
    }
  ]
}
```

---

## 3. Lexical Structure

### 3.1 Character Set

ORT uses UTF-8 encoding and supports the full Unicode character set.

### 3.2 Line Terminators

- Unix style: `\n` (LF)
- Windows style: `\r\n` (CRLF)
- Both are supported and normalized during parsing

### 3.3 Whitespace

- Spaces and tabs are trimmed from line beginnings and endings
- Whitespace within values is preserved
- Empty lines are ignored

### 3.4 Reserved Characters

The following characters have special meaning in ORT:

| Character | Purpose | Escape Required |
|-----------|---------|-----------------|
| `:` | Header delimiter | No (only in headers) |
| `,` | Value separator | Yes (in values) |
| `(` | Object/nested field start | Yes (in string values) |
| `)` | Object/nested field end | Yes (in string values) |
| `[` | Array start | Yes (in string values) |
| `]` | Array end | Yes (in string values) |
| `\` | Escape character | Yes (always `\\`) |
| `#` | Comment marker | No (only at line start) |

---

## 4. Data Types

ORT supports six primitive and composite data types:

### 4.1 Null

Represents absence of a value.

**Syntax**: Empty string or no value between delimiters

**Examples**:
```ort
users:id,name,email:
1,John,
2,Jane,jane@example.com
```

**JSON Equivalent**:
```json
{
  "users": [
    {"id": 1, "name": "John", "email": null},
    {"id": 2, "name": "Jane", "email": "jane@example.com"}
  ]
}
```

### 4.2 Boolean

Boolean values representing true or false.

**Syntax**:
- `true` - Boolean true
- `false` - Boolean false

**Case Sensitivity**: Lowercase only

**Examples**:
```ort
settings:enabled,verified:
true,false
false,true
```

**JSON Equivalent**:
```json
{
  "settings": [
    {"enabled": true, "verified": false},
    {"enabled": false, "verified": true}
  ]
}
```

### 4.3 Number

Numeric values including integers and floating-point numbers.

**Syntax**:
- Integer: `42`, `-17`, `0`
- Float: `3.14`, `-0.5`, `999.99`
- Scientific notation: NOT supported in current version

**Range**:
- Integers: 64-bit signed (-2^63 to 2^63-1)
- Floats: 64-bit IEEE 754 double precision

**Examples**:
```ort
products:id,price:
101,999.99
102,29.99
103,79.99
```

**Special Cases**:
- Leading zeros are preserved for strings: `007` → `"007"`
- Pure numbers are parsed as numbers: `007` → `7`
- To force string interpretation, use escape: `\007` → `"007"` (if not parseable as number)

### 4.4 String

UTF-8 encoded text values.

**Syntax**: Raw text without quotes

**Characteristics**:
- No surrounding quotes required
- Whitespace is trimmed from beginning and end
- Internal whitespace is preserved
- Special characters must be escaped

**Examples**:
```ort
users:id,name:
1,John Doe
2,Jane Smith
```

**Trimming Behavior**:
```ort
data:value:
  hello world
```
Parsed as: `"hello world"` (leading/trailing spaces removed)

### 4.5 Array

Ordered collection of values.

**Syntax**: `[value1,value2,value3]`

**Characteristics**:
- Square brackets `[]` delimit arrays
- Values separated by commas
- Can contain any data type
- Can be nested
- Empty arrays: `[]`

**Examples**:

**Simple Array**:
```ort
colors:
[red,green,blue,yellow]
```

**Nested Array**:
```ort
matrix:
[[1,2,3],[4,5,6],[7,8,9]]
```

**Mixed Type Array**:
```ort
data:
[42,hello world,true,(id:100,active:false),[1,2,3]]
```

**Array Field**:
```ort
users:id,tags:
1,[admin,user]
2,[]
3,[guest]
```

### 4.6 Object

Unordered collection of key-value pairs.

**Syntax**:
- Inline: `(key1:value1,key2:value2)`
- Header-based: Defined in header, values in data lines

**Characteristics**:
- Parentheses `()` delimit inline objects
- Key-value pairs separated by colons `:`
- Pairs separated by commas
- Can be nested
- Empty objects: `()`

**Examples**:

**Inline Object**:
```ort
data:
[(id:1,name:Alice),(id:2,name:Bob)]
```

**Header-based Object** (Preferred):
```ort
users:id,name:
1,Alice
2,Bob
```

**Nested Object**:
```ort
users:id,profile(name,age):
1,(Alice,30)
2,(Bob,25)
```

---

## 5. Syntax Rules

### 5.1 Document Structure

An ORT document consists of one or more sections. Each section has:

1. **Header Line**: Defines structure and field names
2. **Data Lines**: Contains actual values

### 5.2 Two Main Formats

#### 5.2.1 Named Section Format

**Syntax**: `keyName:field1,field2,...:`

**Usage**: Creating named arrays in the root object

**Example**:
```ort
users:id,name:
1,Alice
2,Bob
```

**Result**:
```json
{
  "users": [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"}
  ]
}
```

#### 5.2.2 Top-Level Format

**Syntax**: `:field1,field2,...:`

**Usage**: Creating root-level objects or arrays

**Single Object**:
```ort
:id,name,email:
1001,Alice Williams,alice@example.com
```

**Result**:
```json
{
  "id": 1001,
  "name": "Alice Williams",
  "email": "alice@example.com"
}
```

**Multiple Objects (Array)**:
```ort
:id,name:
1,Alice
2,Bob
```

**Result**:
```json
[
  {"id": 1, "name": "Alice"},
  {"id": 2, "name": "Bob"}
]
```

---

## 6. Header Syntax

### 6.1 Header Format

**General Form**: `[keyName]:field1,field2,...:`

**Components**:
1. **Optional Key Name**: Identifier for the data section
2. **Colon**: Separates key name from fields
3. **Field List**: Comma-separated field names
4. **Trailing Colon**: Marks end of header

### 6.2 Field Names

**Rules**:
- Must be valid identifiers
- Case-sensitive
- Can contain letters, numbers, underscores
- Cannot start with a number (by convention)
- No spaces allowed

**Valid Examples**:
- `id`
- `firstName`
- `user_name`
- `item2`
- `_private`

**Invalid Examples**:
- `first name` (contains space)
- `2ndItem` (starts with number - technically allowed but not recommended)

### 6.3 Nested Fields

Nested fields represent object structures.

**Syntax**: `fieldName(nestedField1,nestedField2,...)`

**Example**:
```ort
users:id,profile(name,age,email):
1,(John Doe,30,john@example.com)
2,(Jane Smith,25,jane@example.com)
```

### 6.4 Deeply Nested Fields

Nesting can be arbitrarily deep.

**Example**:
```ort
users:id,profile(name,age,address(city,country)):
1,(John Doe,30,(New York,USA))
2,(Jane Smith,25,(London,UK))
```

**JSON Equivalent**:
```json
{
  "users": [
    {
      "id": 1,
      "profile": {
        "name": "John Doe",
        "age": 30,
        "address": {
          "city": "New York",
          "country": "USA"
        }
      }
    },
    {
      "id": 2,
      "profile": {
        "name": "Jane Smith",
        "age": 25,
        "address": {
          "city": "London",
          "country": "UK"
        }
      }
    }
  ]
}
```

---

## 7. Data Line Syntax

### 7.1 Basic Data Lines

Data lines contain comma-separated values corresponding to fields in the header.

**Rules**:
1. Values must match header field order
2. Number of values must equal number of fields
3. Values are separated by commas
4. Leading/trailing whitespace is trimmed

**Example**:
```ort
users:id,name,age:
1,Alice,30
2,Bob,25
```

### 7.2 Value Parsing

Values are parsed in the following order:

1. **Empty string** → `null`
2. **`[]`** → Empty array
3. **`()`** → Empty object
4. **`[...]`** → Array
5. **`(...)`** → Object (if contains `:`) or Nested object (if in nested field)
6. **Numeric string** → Number (if parseable)
7. **`true`/`false`** → Boolean
8. **Everything else** → String

### 7.3 Nested Object Values

For fields defined with nested structure in header, values must be wrapped in parentheses.

**Example**:
```ort
users:id,profile(name,age):
1,(Alice,30)
2,(Bob,25)
```

**Invalid**:
```ort
users:id,profile(name,age):
1,Alice,30  # WRONG: Not wrapped in parentheses
```

### 7.4 Multiple Data Lines

Multiple data lines create an array of objects.

**Example**:
```ort
users:id,name:
1,Alice
2,Bob
3,Charlie
```

**Result**:
```json
{
  "users": [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"},
    {"id": 3, "name": "Charlie"}
  ]
}
```

---

## 8. Nested Structures

### 8.1 Objects within Objects

**Header Definition**:
```ort
users:id,profile(name,contact(email,phone)):
```

**Data**:
```ort
1,(John,(john@example.com,555-1234))
```

**Result**:
```json
{
  "users": [{
    "id": 1,
    "profile": {
      "name": "John",
      "contact": {
        "email": "john@example.com",
        "phone": "555-1234"
      }
    }
  }]
}
```

### 8.2 Arrays within Objects

**Example**:
```ort
users:id,name,tags:
1,Alice,[admin,user]
2,Bob,[guest]
3,Charlie,[]
```

### 8.3 Objects within Arrays

**Inline Objects**:
```ort
data:
[(id:1,name:Alice),(id:2,name:Bob)]
```

**Result**:
```json
{
  "data": [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"}
  ]
}
```

### 8.4 Complex Nesting

Combining all nesting types:

```ort
records:id,data(values,metadata(tags,settings(options))):
1,([1,2,3],([dev,test],((verbose:true,debug:false))))
```

**Result**:
```json
{
  "records": [{
    "id": 1,
    "data": {
      "values": [1, 2, 3],
      "metadata": {
        "tags": ["dev", "test"],
        "settings": {
          "options": {
            "verbose": true,
            "debug": false
          }
        }
      }
    }
  }]
}
```

---

## 9. Escape Sequences

### 9.1 Purpose

Escape sequences allow special characters to be included in string values.

### 9.2 Supported Escape Sequences

| Escape Sequence | Character | Description |
|-----------------|-----------|-------------|
| `\\` | `\` | Backslash |
| `\,` | `,` | Comma |
| `\(` | `(` | Left parenthesis |
| `\)` | `)` | Right parenthesis |
| `\[` | `[` | Left square bracket |
| `\]` | `]` | Right square bracket |
| `\n` | Line Feed | Newline |
| `\t` | Tab | Horizontal tab |
| `\r` | Carriage Return | Carriage return |

### 9.3 Examples

#### Escaping Delimiters

```ort
messages:id,text:
1,\(Hello\, World!\)
2,Price: $99\,99
3,Use backslash: \\
4,Array syntax: \[1\,2\,3\]
```

**Result**:
```json
{
  "messages": [
    {"id": 1, "text": "(Hello, World!)"},
    {"id": 2, "text": "Price: $99,99"},
    {"id": 3, "text": "Use backslash: \\"},
    {"id": 4, "text": "Array syntax: [1,2,3]"}
  ]
}
```

#### Newlines and Tabs

```ort
texts:id,content:
1,First line\nSecond line\nThird line
2,Name:\tJohn\nAge:\t30
```

**Result**:
```json
{
  "texts": [
    {"id": 1, "content": "First line\nSecond line\nThird line"},
    {"id": 2, "content": "Name:\tJohn\nAge:\t30"}
  ]
}
```

### 9.4 Escape Processing

**Processing Rules**:
1. Backslash followed by recognized character → Replace with escaped character
2. Backslash followed by unrecognized character → Keep the character, remove backslash
3. Backslash at end of string → Keep backslash

---

## 10. Comments

### 10.1 Syntax

Comments start with `#` at the beginning of a line.

**Characteristics**:
- Line comments only (no inline comments)
- `#` must be the first non-whitespace character
- Everything after `#` to end of line is ignored
- Comments can appear anywhere in the document

### 10.2 Examples

```ort
# This is a comment
users:id,name:  # This is NOT a comment (not at line start)
1,Alice
# Another comment
2,Bob
```

### 10.3 Documentation Style

```ort
# User Database
# Format: ID, Name, Email, Active Status
# Last Updated: 2025-01-15

users:id,name,email,active:
1001,Alice Williams,alice@example.com,true
1002,Bob Johnson,bob@example.com,false
```

---

## 11. Complete Examples

### 11.1 Basic Object Array

```ort
users:age,id,name:
30,1,John Doe
25,2,Jane Smith
35,3,Bob Johnson
```

### 11.2 Simple Array

```ort
colors:
[red,green,blue,yellow]
```

### 11.3 Top-Level Single Object

```ort
:id,name,email,active:
1001,Alice Williams,alice@example.com,true
```

### 11.4 Nested Objects

```ort
users:id,profile(name,age,address(city,country)):
1,(John Doe,30,(New York,USA))
2,(Jane Smith,25,(London,UK))
```

### 11.5 Nested Array

```ort
matrix:
[[1,2,3],[4,5,6],[7,8,9]]
```

### 11.6 Mixed Array

```ort
data:
[42,hello world,true,(id:100,active:false),[1,2,3]]
```

### 11.7 Multiple Sections

```ort
products:id,name,price:
101,Laptop,999.99
102,Mouse,29.99
103,Keyboard,79.99

categories:id,name:
1,Electronics
2,Accessories
```

### 11.8 Null and Empty Values

```ort
records:id,name,email,tags:
1,John Doe,,[]
2,Jane Smith,jane@example.com,()
3,Bob,bob@example.com,[admin,user]
```

### 11.9 Escape Characters

```ort
messages:id,text:
1,\(Hello\, World!\)
2,Price: $99\,99
3,Use backslash: \\
4,Array syntax: \[1\,2\,3\]
```

### 11.10 Newline and Tab

```ort
texts:id,content:
1,First line\nSecond line\nThird line
2,Name:\tJohn\nAge:\t30
3,Multi\nLine\nText
```

### 11.11 Boolean Values

```ort
settings:id,feature,enabled,verified:
1,notifications,true,false
2,dark_mode,false,true
3,auto_save,true,true
```

---

## 12. Parsing Rules

### 12.1 Value Type Detection

Values are parsed using the following algorithm:

```
function parse_value(string):
  trimmed = trim(string)

  if trimmed is empty:
    return null

  if trimmed == "[]":
    return empty_array

  if trimmed == "()":
    return empty_object

  if trimmed starts with '[' and ends with ']':
    return parse_array(trimmed)

  if trimmed starts with '(' and ends with ')':
    if contains ':' at depth 0:
      return parse_inline_object(trimmed)
    else:
      return parse_nested_object(trimmed)

  unescaped = unescape(trimmed)

  if unescaped is valid number:
    return number

  if unescaped == "true":
    return true

  if unescaped == "false":
    return false

  return string
```

### 12.2 Delimiter Depth Tracking

When parsing values, track nesting depth to correctly identify delimiters:

```
depth = 0
bracket_depth = 0

for each character:
  if character == '(':
    depth++
  else if character == ')':
    depth--
  else if character == '[':
    bracket_depth++
  else if character == ']':
    bracket_depth--
  else if character == ',' and depth == 0 and bracket_depth == 0:
    # This comma is a value separator
```

### 12.3 Field Count Validation

**Rule**: Number of values in data line must exactly match number of fields in header.

**Example Error**:
```ort
users:id,name,age:
1,Alice  # ERROR: Expected 3 values, got 2
```

### 12.4 Nested Object Validation

**Rule**: For nested fields, values must be wrapped in parentheses and contain correct number of nested values.

**Example Error**:
```ort
users:id,profile(name,age):
1,(Alice)  # ERROR: Expected 2 nested values, got 1
```

---

## 13. Best Practices

### 13.1 When to Use ORT

**Good Use Cases**:
- Uniform data structures (same fields across records)
- Large datasets for LLM consumption
- Token-optimized data transfer
- Structured logging

**Poor Use Cases**:
- Heterogeneous data (different fields per record)
- Direct application configuration
- Human-primary editing scenarios
- Data with frequent schema changes

### 13.2 Naming Conventions

**Field Names**:
- Use camelCase or snake_case consistently
- Keep names concise but descriptive
- Avoid abbreviations unless widely understood

**Examples**:
```ort
# Good
users:id,firstName,lastName,emailAddress:

# Acceptable
users:id,first_name,last_name,email_address:

# Avoid
users:i,fn,ln,ea:  # Too cryptic
```

### 13.3 Structure Design

**Prefer Flat Over Nested** (when reasonable):
```ort
# Better for token efficiency
users:id,name,city,country:
1,John,New York,USA

# More nested, but more tokens
users:id,name,address(city,country):
1,John,(New York,USA)
```

**Use Nesting for Logical Grouping**:
```ort
# Good use of nesting
users:id,profile(name,age,email),settings(theme,language):
```

### 13.4 Data Organization

**Group Related Sections**:
```ort
# Good: Related data together
users:id,name:
1,Alice
2,Bob

user_roles:user_id,role:
1,admin
2,user
```

**Use Comments for Clarity**:
```ort
# User Master Data
users:id,name,email:
1,Alice,alice@example.com

# User Permissions
permissions:user_id,resource,access:
1,/admin,read-write
```

### 13.5 Error Handling

**Always Validate**:
1. Field count matches value count
2. Nested structures are properly formed
3. Escape sequences are valid
4. Data types are appropriate

**Provide Clear Error Messages**:
```
Line 5: Expected 3 values but got 2
  1,Alice
```

### 13.6 Performance Considerations

**For Large Datasets**:
- Stream parsing when possible
- Validate headers before processing data
- Use appropriate buffer sizes
- Consider memory constraints for nested structures

**Token Optimization**:
- Minimize field name lengths (while maintaining clarity)
- Use flat structures when appropriate
- Avoid redundant nesting

---

## Appendix A: Type Conversion Table

| ORT Value | Parsed Type | Notes |
|-----------|-------------|-------|
| (empty) | null | Empty string |
| `true` | boolean | Lowercase only |
| `false` | boolean | Lowercase only |
| `42` | number | Integer |
| `3.14` | number | Float |
| `-17` | number | Negative number |
| `hello` | string | Raw text |
| `[]` | array | Empty array |
| `[1,2,3]` | array | Array of numbers |
| `()` | object | Empty object |
| `(a:1,b:2)` | object | Inline object |
| `(val1,val2)` | object | Nested object (context-dependent) |

---

## Appendix B: Character Encoding

ORT documents must be encoded in UTF-8. Parsers should:
1. Accept UTF-8 with or without BOM
2. Reject invalid UTF-8 sequences
3. Preserve Unicode characters in string values
4. Handle surrogate pairs correctly

---

## Appendix C: Implementation Notes

### C.1 Parser Requirements

A compliant ORT parser must:
1. Support all data types defined in Section 4
2. Implement escape sequence processing (Section 9.2)
3. Validate field/value count matching (Section 13.3)
4. Handle arbitrary nesting depth (Section 8)
5. Ignore comments and empty lines (Section 10)

### C.2 Generator Requirements

A compliant ORT generator must:
1. Escape special characters in string values
2. Generate valid headers for object arrays
3. Maintain field order consistency
4. Output minimal whitespace
5. Use UTF-8 encoding

### C.3 Error Handling

Implementations should provide clear error messages including:
- Line number
- Problematic content
- Description of the error
- Suggested fix (when applicable)

---

## Appendix D: References

- [ORT GitHub Repository](https://github.com/ort-format/ORT)
- [ORT Playground](https://ort-format.github.io/ORT-Playground/)
- [TOON Format](https://github.com/toon-format/toon) - Inspiration for ORT

---

**End of Specification**
