# Binary Hex to Constant Pool Mapping

## Understanding the .class File Bytes

The constant pool data starts right after the class file header. Here's understanding how to parse the hex.

```
Offset    Bytes                                   Representation
------    -----                                   ----------------
0x00      CA FE BA BE                             Magic number (0xCAFEBABE)
0x04      00 00                                   Minor version: 0
0x06      00 3D                                   Major version: 61 (Java 17)
0x08      00 1E                                   constant_pool_count: 30
0x0A      >>> CONSTANT POOL STARTS HERE <<<
```

---

## Constant Pool Entries (Starting at Offset 0x0A)

The constant pool is a sequence of entries. Each entry has a 1-byte tag followed by fields specific to that type.

### Entry #1 (Offset 0x0A): CONSTANT_Methodref

```
Hex:      0A 00 02 00 03
          ^  ^^^^^^^^^    ^^^^^^^^^
          |  class_index  name_and_type_index
          tag

Breakdown:
  0A           = Tag 10 = CONSTANT_Methodref
  00 02        = class_index = 2 (points to entry #2)
  00 03        = name_and_type_index = 3 (points to entry #3)

Resolves to:
  Entry #2 (Class) → Entry #4 (Utf8) → "java/lang/Object"
  Entry #3 (NameAndType) → #5, #6 → "<init>", "()V"
  
  Final: java/lang/Object.<init>():V  (Object constructor)
```

### Entry #2 (Offset 0x0F): CONSTANT_Class

```
Hex:      07 00 04
          ^  ^^^^^^^^
          |  name_index
          tag

Breakdown:
  07           = Tag 7 = CONSTANT_Class
  00 04        = name_index = 4 (points to entry #4)

Resolves to:
  Entry #4 (Utf8) → "java/lang/Object"
  
  Final: Class reference to java/lang/Object
```

### Entry #3 (Offset 0x12): CONSTANT_NameAndType

```
Hex:      0C 00 05 00 06
          ^  ^^^^^^^^ ^^^^^^^^
          |  name_idx desc_idx
          tag

Breakdown:
  0C           = Tag 12 = CONSTANT_NameAndType
  00 05        = name_index = 5 (points to entry #5)
  00 06        = descriptor_index = 6 (points to entry #6)

Resolves to:
  Entry #5 (Utf8) → "<init>"
  Entry #6 (Utf8) → "()V"
  
  Final: "<init>":()V  (method signature)
```

### Entry #4 (Offset 0x17): CONSTANT_Utf8

```
Hex:      01 00 10 6A 61 76 61 2F 6C 61 6E 67 2F 4F 62 6A 65 63 74
          ^  ^^^^^^ ^^^^^^.....................^^^^^^^^^^^^^^^^^^^^^^^
          |  length  UTF-8 byte data (16 bytes)
          tag

Breakdown:
  01           = Tag 1 = CONSTANT_Utf8
  00 10        = length = 16 bytes
  6A 61 76 61... = "java/lang/Object" (16 ASCII bytes)

String content: "java/lang/Object"
```

---

## KEY ENTRY: #7 (CONSTANT_Fieldref for myField)

The CONSTANT_Fieldref entry we care about:

### Structure:

```
Hex:      09 00 08 00 09
          ^  ^^^^^^^^    ^^^^^^^^
          |  class_idx   name_and_type_idx
          tag

09           = Tag 9 = CONSTANT_Fieldref  ← This is #7!
00 08        = class_index = 8
00 09        = name_and_type_index = 9
```

### Resolution Process:

**Step 1: Follow class_index (8)**
```
Entry #8 is a CONSTANT_Class:
  07 00 0A   = CONSTANT_Class pointing to entry #10
  
Entry #10 contains:
  01 00 0D 53 69 6D 70 6C 65 45 78 61 6D 70 6C 65
  = CONSTANT_Utf8 with string "SimpleExample"
```

**Step 2: Follow name_and_type_index (9)**
```
Entry #9 is a CONSTANT_NameAndType:
  0C 00 0B 00 0C   = CONSTANT_NameAndType
                     name_index = 11
                     descriptor_index = 12

Entry #11 contains:
  01 00 07 6D 79 46 69 65 6C 64
  = CONSTANT_Utf8 with string "myField"

Entry #12 contains:
  01 00 01 49
  = CONSTANT_Utf8 with string "I"
```

### Final Resolution of Entry #7:

```
#7 (CONSTANT_Fieldref):
  09 00 08 00 09
  ├─ Tag: 09 (CONSTANT_Fieldref)
  ├─ class_index: 00 08 → Entry #8
  │     ├─ Tag: 07 (CONSTANT_Class)
  │     └─ name_index: 00 0A → Entry #10
  │           ├─ Tag: 01 (CONSTANT_Utf8)
  │           └─ String: "SimpleExample"
  │
  └─ name_and_type_index: 00 09 → Entry #9
        ├─ Tag: 0C (CONSTANT_NameAndType)
        ├─ name_index: 00 0B → Entry #11
        │     ├─ Tag: 01 (CONSTANT_Utf8)
        │     └─ String: "myField"
        │
        └─ descriptor_index: 00 0C → Entry #12
              ├─ Tag: 01 (CONSTANT_Utf8)
              └─ String: "I"

COMPLETE INFO: SimpleExample.myField:I
```

---

## KEY ENTRY: #19 (CONSTANT_Methodref for println)

### Structure:

```
Hex:      0A 00 14 00 15
          ^  ^^^^^^^^    ^^^^^^^^
          |  class_idx   name_and_type_idx
          tag

0A           = Tag 10 = CONSTANT_Methodref  ← Entry #19!
00 14        = class_index = 20
00 15        = name_and_type_index = 21
```

### Resolution Process:

**Step 1: Follow class_index (20)**
```
Entry #20 (07 00 16):
  CONSTANT_Class pointing to entry #22

Entry #22:
  CONSTANT_Utf8 = "java/io/PrintStream"
```

**Step 2: Follow name_and_type_index (21)**
```
Entry #21 (0C 00 17 00 18):
  CONSTANT_NameAndType
  name_index = 23
  descriptor_index = 24

Entry #23:
  CONSTANT_Utf8 = "println"

Entry #24:
  CONSTANT_Utf8 = "(I)V"
```

### Final Resolution of Entry #19:

```
#19 (CONSTANT_Methodref):
  0A 00 14 00 15
  ├─ Tag: 0A (CONSTANT_Methodref)
  ├─ class_index: 00 14 → Entry #20
  │     ├─ Tag: 07 (CONSTANT_Class)
  │     └─ name_index: 00 16 → Entry #22
  │           ├─ Tag: 01 (CONSTANT_Utf8)
  │           └─ String: "java/io/PrintStream"
  │
  └─ name_and_type_index: 00 15 → Entry #21
        ├─ Tag: 0C (CONSTANT_NameAndType)
        ├─ name_index: 00 17 → Entry #23
        │     ├─ Tag: 01 (CONSTANT_Utf8)
        │     └─ String: "println"
        │
        └─ descriptor_index: 00 18 → Entry #24
              ├─ Tag: 01 (CONSTANT_Utf8)
              └─ String: "(I)V"

COMPLETE INFO: java/io/PrintStream.println:(I)V
```

---

## Bytecode Instructions Using These Entries

### getfield #7

```
Bytecode Instruction at method offset 0x04:
  B4 00 07
  ^  ^^^^
  |  index into constant pool
  opcode for getfield

B4 = getfield opcode
00 07 = index 7 in constant pool

JVM execution:
  1. Look up constant pool entry #7
  2. Entry #7 is CONSTANT_Fieldref
  3. Follow the chain to get: SimpleExample.myField:I
  4. From the object instance, get the int field value
  5. Push it onto the stack
```

### invokevirtual #19

```
Bytecode Instruction at method offset 0x07:
  B6 00 13
  ^  ^^^^
  |  index into constant pool (0x13 = 19 in decimal)
  opcode for invokevirtual

B6 = invokevirtual opcode
00 13 = index 19 in constant pool

JVM execution:
  1. Look up constant pool entry #19
  2. Entry #19 is CONSTANT_Methodref
  3. Follow the chain to get: java/io/PrintStream.println:(I)V
  4. Get the actual PrintStream object from stack
  5. Do virtual method dispatch to find the real method implementation
  6. Call the method with the int parameter
```

---

## Summary: Binary to High-Level Mapping

```
RAW BINARY (.class file)
  ↓
Hex bytes: 09 00 08 00 09
  ↓
Parse as CONSTANT_Fieldref (tag=09)
  ├─ class_index=08 (u2 value)
  └─ name_and_type_index=09 (u2 value)
  ↓
CONSTANT_Fieldref resolved
  ├─ class_index → #8 → CONSTANT_Class
  │   └─ name_index → #10 → CONSTANT_Utf8
  │       └─ String: "SimpleExample"
  │
  └─ name_and_type_index → #9 → CONSTANT_NameAndType
      ├─ name_index → #11 → CONSTANT_Utf8
      │   └─ String: "myField"
      └─ descriptor_index → #12 → CONSTANT_Utf8
          └─ String: "I"
  ↓
FULLY RESOLVED: SimpleExample.myField:I
  ├─ Class: SimpleExample
  ├─ Field: myField
  └─ Type: int
  ↓
RUNTIME INSTANCE CLASS MAPPING
  JVM loads SimpleExample class metadata
  Finds field "myField" in class object
  Accesses the field value from instance
```

This is how a few bytes in a binary file become meaningful field/method references at runtime!
