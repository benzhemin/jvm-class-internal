# Field and Method Offset Calculation - Quick Reference

## Key Concept: Why Offsets?

```
PROBLEM:
  If we stored field name as a string and had to look it up every time
  we accessed a field, it would be SLOW (hashtable lookup).

SOLUTION:
  Calculate field offsets ONCE during class loading.
  Then use simple memory arithmetic at runtime: address = base + offset

RESULT:
  Field access is O(1) - just addition!
```

---

## Field Offset Calculation - In 4 Steps

### Step 1: Determine Object Header Size

```
Most JVMs:  header_size = 12 or 16 bytes

Breakdown (typical 64-bit JVM):
  ├─ Class pointer: 8 bytes
  ├─ Monitor/hash: 4-8 bytes
  └─ Total: 12-16 bytes

This is where field offsets START from!
```

### Step 2: Parse and Sort Fields

```
From .class file FIELDS section:
  Field 1: boolean myFlag   (1 byte)
  Field 2: long myLong      (8 bytes)
  Field 3: int myInt        (4 bytes)

After sorting by size (largest first):
  Field 1: long myLong      (8 bytes)    ← Largest
  Field 2: int myInt        (4 bytes)
  Field 3: boolean myFlag   (1 byte)     ← Smallest

WHY? Minimize memory padding/fragmentation
```

### Step 3: Assign Offsets Sequentially

```
Algorithm:
  current_offset = header_size  (e.g., 12)
  
  for each field (in sorted order):
    field.offset = current_offset
    current_offset += field.size

EXAMPLE:
  header = 12
  offset = 12
  
  long myLong (8 bytes):
    offset = 12
    next = 12 + 8 = 20
  
  int myInt (4 bytes):
    offset = 20
    next = 20 + 4 = 24
  
  boolean myFlag (1 byte):
    offset = 24
    next = 24 + 1 = 25
  
  instance_size = 25 bytes
```

### Step 4: Store Calculated Values

```
In Class Object:
  MyClass.fields[0].offset = 12    ← CACHED
  MyClass.fields[1].offset = 20    ← CACHED
  MyClass.fields[2].offset = 24    ← CACHED
  MyClass.instance_size = 25       ← CACHED

These values are REUSED forever!
```

---

## Field Offset Usage at Runtime

### Instance Creation

```
new MyClass();

1. Allocate: memory = malloc(instance_size)  [25 bytes]
2. Set class pointer at offset 0
3. Set monitor at offset 8
4. Initialize each field:
   
   For myLong (offset 12):
     address = base + 12
     *(long*)(address) = 0L  [default]
   
   For myInt (offset 20):
     address = base + 20
     *(int*)(address) = 0    [default]
   
   For myFlag (offset 24):
     address = base + 24
     *(byte*)(address) = 0   [default]
```

### Field Read (getfield #N)

```
Bytecode:  getfield #7

1. Resolve constant_pool[7]:
   ├─ field: MyClass.myInt
   └─ offset: 20  ← PRECALCULATED
2. Get object from stack: obj = 0x7f8e00001000
3. Calculate: address = 0x7f8e00001000 + 20 = 0x7f8e00001014
4. Read: value = *(int*)0x7f8e00001014
5. Push: stack.push(value)

PERFORMANCE: ~5 nanoseconds (just arithmetic + memory read)
```

### Field Write (putfield #N)

```
Bytecode:  putfield #7

1. Pop: value = stack.pop()  [99]
2. Pop: obj = stack.pop()    [0x7f8e00001000]
3. Get offset from constant_pool[7]: 20
4. Calculate: address = 0x7f8e00001000 + 20
5. Write: *(int*)address = 99
6. Done!

PERFORMANCE: ~5 nanoseconds (same as read)
```

---

## Method Offset Calculation

### Bytecode Offset (in method bytecode)

```
Method bytecode (9 bytes):
┌─────┬─────┬───────┐
│ Off │ Hex │ Instr │
├─────┼─────┼───────┤
│ 0   │ AA  │ aload_0        │
│ 1   │ B4  │ getfield       │  (3 bytes total: opcode + 2 arg)
│ 2   │ 00  │   index high   │
│ 3   │ 07  │   index low    │
│ 4   │ B6  │ invokevirtual  │  (3 bytes total)
│ 5   │ 00  │   index high   │
│ 6   │ 13  │   index low    │
│ 7   │ B1  │ return         │
└─────┴─────┴───────┘

Offsets calculated during bytecode parsing:
  Instruction at offset 0: aload_0
  Instruction at offset 1: getfield  (next: 1+3=4)
  Instruction at offset 4: invokevirtual (next: 4+3=7)
  Instruction at offset 7: return (next: 7+1=8)
```

### Why Bytecode Offsets Matter

```
1. Debugging: Map bytecode position ↔ source line
   LineNumberTable:
     bytecode 0 → source line 4
     bytecode 4 → source line 5
     
2. Exception handling: Which bytecode instruction failed?
3. Stack traces: Show exact location in method

NOT used for regular method calls (unlike field offsets).
```

---

## VTable Index (Method Dispatch)

### What is VTable?

```
Virtual method table - array of method pointers for polymorphism

Example:
  VTable[0] = Object.equals
  VTable[1] = Object.hashCode
  ...
  VTable[10] = MyClass.printField
  ...
```

### How VTable Indices are Assigned

```
During class loading:

1. Inherit parent's vtable
   Object has 11 methods → copy to MyClass.vtable[0..10]

2. For each new method:
   If overrides parent:
     Replace vtable entry (same index)
   If new method:
     Append to vtable (new index)

EXAMPLE:
  Object methods: [0..10]
  MyClass.printField():void
    → Not in Object
    → Assign index 11
    → MyClass.vtable[11] = MyClass.printField
```

### VTable Usage at Method Invocation

```
Bytecode: invokevirtual #19

1. Resolve constant_pool[19]: method PrintStream.println
   - vtable_index = 42
   
2. Get object from stack: obj = System.out
3. Get actual class: actual_class = obj.class
4. Lookup in vtable: method = actual_class.vtable[42]
5. Call: method.invoke(obj, args)

WHY VTABLE?
  ✓ Polymorphism: subclass can override
  ✓ Fast dispatch: O(1) array lookup
  ✓ No type checking: vtable handles correct method
```

---

## Memory Layout Example (Concrete)

```
SimpleExample class with field: myField (int)

AFTER CLASS LOADING:
  SimpleExample.class.instance_size = 16 bytes
  SimpleExample.fields[0].name = "myField"
  SimpleExample.fields[0].offset = 12
  SimpleExample.fields[0].type = int

INSTANCE CREATION:
  obj = new SimpleExample();
  
  Memory allocation: 16 bytes
  obj = 0x7f8e12345678
  
  Layout:
  ┌─────────────┬────────────────────────────┐
  │ Bytes 0-7   │ Class pointer              │
  │ Bytes 8-11  │ Monitor                    │
  │ Bytes 12-15 │ myField value (0 default)  │ ← offset 12!
  └─────────────┴────────────────────────────┘

FIELD ACCESS:
  getfield myField
  → Calculate: address = 0x7f8e12345678 + 12
  → Read: value = *(int*)address
  → Done!

TIMING: 
  Without offset: ~25 nanoseconds (hashtable lookup)
  With offset: ~5 nanoseconds (arithmetic + read)
  → 5x FASTER!
```

---

## Comparison: Static vs Dynamic Offsets

```
┌─────────────────────┬─────────────────────┐
│ Static Offsets      │ Dynamic Offsets     │
│ (JVM approach) ✓    │ (Slow approach) ✗   │
├─────────────────────┼─────────────────────┤
│ Calculated at:      │ Calculated at:      │
│ class load time     │ runtime             │
│ (once)              │ (every access)      │
│                     │                     │
│ Storage:            │ Storage:            │
│ In class metadata   │ Lookup table        │
│ (8 bytes per field) │ (>100 bytes)        │
│                     │                     │
│ Access time:        │ Access time:        │
│ O(1) = 1 cycle      │ O(log n) = 10+      │
│ arithmetic          │ cycles              │
│                     │                     │
│ Memory:             │ Memory:             │
│ Cache friendly      │ Cache unfriendly    │
│ (hot data)          │ (hashtable)         │
│                     │                     │
│ Scalability:        │ Scalability:        │
│ Same time regardless │ Slower with more   │
│ of field count      │ fields              │
└─────────────────────┴─────────────────────┘
```

---

## Three Types of Offsets - Summary

```
┌──────────────────┬──────────────────┬──────────────────┐
│ FIELD OFFSET     │ BYTECODE OFFSET  │ VTABLE INDEX     │
├──────────────────┼──────────────────┼──────────────────┤
│                  │                  │                  │
│ Calculated:      │ Calculated:      │ Calculated:      │
│ Class loading    │ Method parsing   │ Class loading    │
│                  │                  │                  │
│ What it is:      │ What it is:      │ What it is:      │
│ Bytes from       │ Byte position    │ Slot number in   │
│ object start     │ in bytecode      │ vtable array     │
│                  │                  │                  │
│ Used for:        │ Used for:        │ Used for:        │
│ Field access     │ Debugging,       │ Virtual method   │
│ (getfield/       │ exception        │ dispatch         │
│  putfield)       │ handling         │ (invokevirtual)  │
│                  │                  │                  │
│ Performance:     │ Performance:     │ Performance:     │
│ O(1) access      │ Not critical     │ O(1) lookup      │
│                  │                  │                  │
│ Example:         │ Example:         │ Example:         │
│ myField at 12    │ aload_0 at 0     │ printField at 11 │
│                  │ getfield at 1    │                  │
│                  │ return at 7      │                  │
│                  │                  │                  │
└──────────────────┴──────────────────┴──────────────────┘
```

---

## Key Takeaways

1. **Field Offsets are Pre-Calculated**
   - During class loading, not at access time
   - Stored in class metadata for instant reuse
   - Makes field access O(1) fast

2. **Offset = Simple Arithmetic**
   - `address = base_address + offset`
   - No hashtable lookup
   - No string comparison
   - Cache-friendly

3. **Three Different Offset Types**
   - Field offsets: where data is in instance
   - Bytecode offsets: where instructions are in method
   - VTable indices: where method is in dispatch table

4. **Design Trade-off**
   - Cost: Small CPU work at class load time
   - Benefit: Fast field access repeated billions of times
   - Result: Massive performance gain

5. **Memory Layout Optimization**
   - Fields sorted by size to minimize padding
   - Instance size = header + sum of field sizes
   - Layout is fixed and deterministic

This is why Java's JVM can run efficiently despite using symbolic references!
