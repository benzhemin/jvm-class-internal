# Linking Phase: Visual Flow Diagram

## High-Level Linking Process

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         BYTECODE EXECUTION STARTS                        │
│                                                                         │
│                    JVM needs to execute: getfield #7                   │
└─────────────────────────────────────────────────────────────────────────┘
                                   ↓
                        ┌──────────────────────┐
                        │  Is entry #7 already │
                        │      resolved?       │
                        └──────────────────────┘
                             ↙          ↖
                           NO           YES
                             ↓            ↓
                        [Linking]    [Execute]
                             ↓            ↓
                      (see diagram)  (use cached
                             ↓        pointers)
                        [Execute]      ↓
                             ↓        [Done]
                          [Done]
```

---

## Detailed Linking Flow (when entry NOT YET resolved)

```
═══════════════════════════════════════════════════════════════════════════
                         LINKING PHASE - DETAILED
═══════════════════════════════════════════════════════════════════════════

INPUT: Constant Pool Entry #7 (SYMBOLIC REFERENCE)
┌──────────────────────────────────────┐
│ Fieldref #7                          │
│   tag: 09                            │
│   class_index: 8                     │
│   name_and_type_index: 9             │
│   resolved: false                    │
└──────────────────────────────────────┘
          ↓
═══════════════════════════════════════════════════════════════════════════


PHASE 1: RESOLVE CLASS REFERENCE
─────────────────────────────────────────────────────────────────────────

    Entry #8 (CONSTANT_Class)
    ┌──────────────────────────────┐
    │ Class ref                    │
    │ tag: 07                      │
    │ name_index: 10               │
    └──────────────────────────────┘
              ↓
    Read name: Entry #10 → "SimpleExample"
              ↓
    ✓ Load SimpleExample.class
    ↓
    ┌──────────────────────────────────────────────┐
    │ RESULT: SimpleExample.class loaded in memory│
    │                                              │
    │ Pointer: 0x7f8c92c50000                      │
    │                                              │
    │ Class metadata:                              │
    │   ├─ Super: Object                           │
    │   ├─ Fields:                                 │
    │   │   ├─ myField: type=int, offset=12       │
    │   │   └─ (others)                            │
    │   ├─ Methods:                                │
    │   │   ├─ printField: sig=()V, offset=X      │
    │   │   └─ (others)                            │
    │   ├─ Method Table (vtable)                   │
    │   └─ (other metadata)                        │
    └──────────────────────────────────────────────┘
              ↓
    ✓ Store: resolved_class_ptr = 0x7f8c92c50000

              ↓
═══════════════════════════════════════════════════════════════════════════


PHASE 2: RESOLVE FIELD/METHOD REFERENCE
─────────────────────────────────────────────────────────────────────────

    Entry #9 (CONSTANT_NameAndType)
    ┌──────────────────────────────┐
    │ NameAndType ref              │
    │ tag: 0C                      │
    │ name_index: 11               │
    │ descriptor_index: 12         │
    └──────────────────────────────┘
              ↓
    Read names:
      Entry #11 → "myField"
      Entry #12 → "I"
              ↓
    ✓ Search SimpleExample.fields[] for:
      - field.name == "myField"
      - field.type == int
              ↓
    ✓ FOUND:
    ┌────────────────────────────────────────┐
    │ Field object (from class metadata)     │
    │                                        │
    │ name: "myField"                        │
    │ type: int.class                        │
    │ descriptor: "I"                        │
    │ offset: 12  ← CRUCIAL!                │
    │ modifiers: 0x0002 (private)            │
    │ address: 0x7f8c92c50240                │
    └────────────────────────────────────────┘
              ↓
    ✓ Store: resolved_field_ptr = 0x7f8c92c50240
    ✓ Store: field_offset = 12

              ↓
═══════════════════════════════════════════════════════════════════════════


PHASE 3: UPDATE CONSTANT POOL ENTRY
─────────────────────────────────────────────────────────────────────────

    Constant Pool Entry #7 BEFORE resolution:
    ┌──────────────────────────────────┐
    │ Fieldref #7                      │
    │ tag: 09                          │
    │ class_index: 8 (symbolic)        │
    │ name_and_type_index: 9 (symbolic)│
    │ resolved: false                  │
    └──────────────────────────────────┘

                        ↓ (UPDATE WITH RESOLVED DATA)

    Constant Pool Entry #7 AFTER resolution:
    ┌────────────────────────────────────────┐
    │ Fieldref #7 (RESOLVED)                 │
    │                                        │
    │ ORIGINAL SYMBOLIC DATA:                │
    │   tag: 09                              │
    │   class_index: 8 ─────────────────┐   │
    │   name_and_type_index: 9 ────────┐│   │
    │                                   ││   │
    │ RESOLVED DATA (NEW):              ││   │
    │   resolved_class_ptr: ────────────┘│   │
    │     0x7f8c92c50000                 │   │
    │                                    │   │
    │   resolved_field_ptr: ─────────────┘   │
    │     0x7f8c92c50240                     │
    │                                        │
    │   field_offset: 12                     │
    │   (no need to lookup again)            │
    │                                        │
    │   resolved: true                       │
    │   (ready for execution!)               │
    └────────────────────────────────────────┘
              ↓
    ✓ Entry #7 is now FULLY RESOLVED

              ↓
═══════════════════════════════════════════════════════════════════════════

OUTPUT: Constant Pool Entry #7 (RESOLVED REFERENCE)
┌────────────────────────────────────────┐
│ Fieldref #7 (RESOLVED)                 │
│                                        │
│ resolved_class_ptr:   0x7f8c92c50000   │
│ resolved_field_ptr:   0x7f8c92c50240   │
│ field_offset: 12                       │
│ resolved: true                         │
│                                        │
│ ✓ Ready for execution!                 │
└────────────────────────────────────────┘
```

---

## Data Structure Change During Linking

```
BEFORE LINKING (symbolic):

const_pool[7] = {
  tag: 0x09,                    // CONSTANT_Fieldref
  class_index: 8,               // Just an index number
  name_and_type_index: 9        // Just an index number
}

Resolution requires:
  1. Load/lookup constant_pool[8]
  2. Load/lookup constant_pool[9]
  3. Load/lookup constant_pool[10], [11], [12]
  4. Load class from disk
  5. Search fields list
  6. Compare strings


AFTER LINKING (resolved):

const_pool[7] = {
  tag: 0x09,                           // Original tag preserved
  class_index: 8,                      // Original index preserved
  name_and_type_index: 9,              // Original index preserved
  
  // NEW FIELDS ADDED DURING LINKING:
  resolved: true,                      // Resolution status
  resolved_class: 0x7f8c92c50000,     // Direct class pointer
  resolved_field: 0x7f8c92c50240,     // Direct field pointer
  field_offset: 12,                   // Cached offset value
  
  // OPTIONAL: Cache the looked-up values for faster access
  class_name: "SimpleExample",         // Cached
  field_name: "myField",               // Cached
  field_type: "I"                      // Cached
}

Execution requires:
  1. Look up const_pool[7]
  2. Check if resolved == true
  3. Use resolved_class and resolved_field directly
  4. Done!
```

---

## Concrete Memory Layout Example

```
JVM HEAP LAYOUT AFTER LINKING:

┌────────────────────────────────────────────────────────────────┐
│                         JVM HEAP                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  CONSTANT POOL (resolved):                                     │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ [7] Fieldref (resolved)                                │   │
│  │     class_ptr: ─────────────────────┐                  │   │
│  │     field_ptr: ──────────┐          │                  │   │
│  │     offset: 12           │          │                  │   │
│  │                          │          │                  │   │
│  │     is_resolved: true    │          │                  │   │
│  └────────────────────────────────────┼──────────────────┘   │
│                                        │  │                    │
│  ┌────────────────────────────────────┘  │                   │
│  │                                       │                    │
│  │  CLASSES:                             │                    │
│  │  ┌──────────────────────────────┐     │                   │
│  │  │ SimpleExample class @ ────────┼─────┘                   │
│  │  │ 0x7f8c92c50000               │                         │
│  │  │                              │                         │
│  │  │ name: "SimpleExample"         │                         │
│  │  │ fields[]:                     │                         │
│  │  │  [0] → Field @ ──────────────┼──┐                      │
│  │  │       0x7f8c92c50240         │  │                      │
│  │  │       name: "myField"        │  │                      │
│  │  │       type: int              │  │                      │
│  │  │       offset: 12  ← KEY!     │  │                      │
│  │  │                              │  │                      │
│  │  │ methods[]:                    │  │                      │
│  │  │  [0] printField()V @ ...     │  │                      │
│  │  │                              │  │                      │
│  │  │ methodTable[]: (vtable)      │  │                      │
│  │  └──────────────────────────────┘  │                      │
│  │                                     │                      │
│  └─────────────────────────────────────┘                      │
│                                                                │
│  INSTANCES (created later at runtime):                        │
│  ┌────────────────────────────────┐                           │
│  │ SimpleExample instance @ ...   │                           │
│  │                                │                           │
│  │ [Header info...]               │                           │
│  │                                │                           │
│  │ +12 bytes: myField value (42)  │ ← offset points here    │
│  │                                │                           │
│  │ [... other fields ...]         │                           │
│  └────────────────────────────────┘                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘

When getfield #7 executes:
  1. Get instance pointer from stack: 0x7f8e...
  2. Get offset from resolved constant_pool[7]: 12
  3. Calculate: 0x7f8e... + 12
  4. Read int value at that address
  5. Push to stack
```

---

## Comparison: Before vs After Linking

```
┌──────────────────────────────────────────────────────────────────┐
│                    BEFORE LINKING                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Constant Pool Entry #7:                                        │
│  ┌────────────────────────────────┐                             │
│  │ tag: 09                        │                             │
│  │ class_index: 8                 │ ← Just indices              │
│  │ name_and_type_index: 9         │                             │
│  │ resolved: false                │                             │
│  └────────────────────────────────┘                             │
│                     ↓                                             │
│  Lookup #8 → "SimpleExample" (string)                          │
│                     ↓                                             │
│  Load SimpleExample.class from disk                             │
│                     ↓                                             │
│  Parse class structure                                          │
│                     ↓                                             │
│  Search fields: name=="myField", type=="I"                     │
│                     ↓                                             │
│  Compare strings, find match                                    │
│                     ↓                                             │
│  Get field location                                             │
│                                                                  │
│  TIME: ~milliseconds (I/O, parsing, searching)                 │
│  REPEATED: Every time entry is accessed before caching        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                     AFTER LINKING                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Constant Pool Entry #7:                                        │
│  ┌────────────────────────────────┐                             │
│  │ tag: 09                        │                             │
│  │ class_index: 8                 │                             │
│  │ name_and_type_index: 9         │                             │
│  │ resolved: true                 │                             │
│  │ resolved_class: 0x7f8c92c50000 │ ← Direct pointers!         │
│  │ resolved_field: 0x7f8c92c50240 │                             │
│  │ field_offset: 12               │                             │
│  └────────────────────────────────┘                             │
│                     ↓                                             │
│  Use pointers directly                                          │
│                     ↓                                             │
│  Access field at offset 12                                     │
│                                                                  │
│  TIME: ~nanoseconds (just pointer arithmetic)                  │
│  REPEATED: Instant access, no re-resolution                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Resolution Errors During Linking

```
                       LINKING STARTS
                            ↓
                   Try to load "SimpleExample"
                            ↓
                      ┌─────────────┐
                      │   Success? │
                      └─────────────┘
                       ↙          ↖
                     YES          NO
                      ↓            ↓
                  Continue     ClassNotFoundException
                      ↓        (throw & stop)
           Try to find "myField"
                      ↓
                ┌─────────────┐
                │   Found?   │
                └─────────────┘
                 ↙          ↖
               YES          NO
                ↓            ↓
            Continue   NoSuchFieldException
                ↓        (throw & stop)
      Check type == "I"
                ↓
          ┌─────────────┐
          │   Match?   │
          └─────────────┘
           ↙          ↖
         YES          NO
          ↓            ↓
      SUCCESS     LinkageError
                (throw & stop)
          ↓
    Mark as resolved
          ↓
     Return control
```

---

## Summary: What Gets Output from Linking

```
INPUT TO LINKING:
  ├─ Symbolic reference in constant pool (indices to strings)
  └─ Classloader (to find classes)

OUTPUT FROM LINKING:
  ├─ Resolved class pointer
  │   └─ Points to loaded class object in memory
  │       └─ Contains all class metadata
  │
  ├─ Resolved field/method pointer
  │   └─ Points to specific field/method object
  │       └─ Contains offset, type, signature, etc.
  │
  ├─ Cached metadata
  │   └─ Offset (for field access)
  │   └─ Virtual method table index (for method calls)
  │
  ├─ Resolution status
  │   └─ Boolean flag: is_resolved = true
  │
  └─ Exception (if resolution fails)
      └─ ClassNotFoundException, NoSuchFieldException, LinkageError, etc.

STORED IN:
  Constant pool entry (updated in place)
  Classloader's cache (for class reuse)
  Method's compiled code (cached pointers for fast access)
```
