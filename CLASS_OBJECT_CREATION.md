# Class Object Creation: Fields and Methods Details

## Overview: What Happens When SimpleExample.class is Loaded

```
SimpleExample.class (binary file)
        ↓
    [Class Parsing]
        ↓
Class Object Created in Memory
    ├─ className
    ├─ superClass
    ├─ fields[]        ← We'll focus here
    ├─ methods[]       ← And here
    ├─ methodTable[]
    └─ other metadata
```

---

## The SimpleExample.class File Structure

First, let's understand the source of the data:

```
SimpleExample.java SOURCE CODE:
┌──────────────────────────────────────────┐
│ public class SimpleExample {             │
│     private int myField;                 │
│                                          │
│     public void printField() {           │
│         System.out.println(myField);     │
│     }                                    │
│ }                                        │
└──────────────────────────────────────────┘

When compiled to SimpleExample.class:
┌──────────────────────────────────────────────────────────────┐
│ MAGIC, VERSION, CONSTANT_POOL_COUNT                          │
├──────────────────────────────────────────────────────────────┤
│ CONSTANT POOL (29 entries)                                   │
│   Entry #1-6: Object initialization references              │
│   Entry #7-12: SimpleExample.myField references              │
│   Entry #13-18: System.out references                        │
│   Entry #19-24: println(int) references                      │
│   Entry #25-29: Code, attributes, source file               │
├──────────────────────────────────────────────────────────────┤
│ CLASS ATTRIBUTES                                             │
│   Access flags: 0x0001 (public)                              │
│   This class: SimpleExample                                  │
│   Super class: java/lang/Object                              │
├──────────────────────────────────────────────────────────────┤
│ FIELDS SECTION (1 field)                                     │
│   ┌────────────────────────────────────────┐                 │
│   │ Field: myField                         │                 │
│   │   access_flags: 0x0002 (private)       │                 │
│   │   name_index: points to "myField"      │                 │
│   │   descriptor_index: points to "I"      │                 │
│   │   attributes_count: 0                  │                 │
│   └────────────────────────────────────────┘                 │
├──────────────────────────────────────────────────────────────┤
│ METHODS SECTION (2 methods)                                  │
│   ┌────────────────────────────────────────┐                 │
│   │ Method 1: <init>() [constructor]       │                 │
│   │   access_flags: 0x0001 (public)        │                 │
│   │   name_index: points to "<init>"       │                 │
│   │   descriptor_index: points to "()V"    │                 │
│   │   attributes:                          │                 │
│   │     Code attribute:                    │                 │
│   │       bytecode: [invoke Object.<init>] │                 │
│   │       max_stack: 1                     │                 │
│   │       max_locals: 1                    │                 │
│   └────────────────────────────────────────┘                 │
│   ┌────────────────────────────────────────┐                 │
│   │ Method 2: printField()                 │                 │
│   │   access_flags: 0x0001 (public)        │                 │
│   │   name_index: points to "printField"   │                 │
│   │   descriptor_index: points to "()V"    │                 │
│   │   attributes:                          │                 │
│   │     Code attribute:                    │                 │
│   │       bytecode: [getfield, invokevirt] │                 │
│   │       max_stack: 2                     │                 │
│   │       max_locals: 1                    │                 │
│   │       LineNumberTable: ...             │                 │
│   └────────────────────────────────────────┘                 │
├──────────────────────────────────────────────────────────────┤
│ CLASS ATTRIBUTES SECTION                                     │
│   SourceFile: "SimpleExample.java"                           │
│   (other attributes...)                                      │
└──────────────────────────────────────────────────────────────┘
```

---

## STEP 1: Class File Parsing - Fields Section

### Raw Binary Data (Simplified):

```
FIELDS SECTION IN SimpleExample.class:
┌────────────┬──────────────┬──────────┬──────────────┬────────────────┐
│ field_count│ access_flags │ name_idx │ descriptor_idx│ attributes_cnt │
├────────────┼──────────────┼──────────┼──────────────┼────────────────┤
│    0x0001  │   0x0002     │  0x000B  │   0x000C     │     0x0000     │
│ (1 field)  │  (private)   │ entry#11 │   entry#12   │   (no extra)   │
└────────────┴──────────────┴──────────┴──────────────┴────────────────┘

What each field means:
  - field_count: 0x0001 = 1 field total
  - access_flags: 0x0002 = ACC_PRIVATE (field is private)
  - name_index: 0x000B = 11 → points to constant pool entry #11
  - descriptor_index: 0x000C = 12 → points to constant pool entry #12
  - attributes_count: 0x0000 = 0 attributes
```

### JVM Parsing Action:

```
JVM Parser reads the FIELDS section:

1. Read field_count: 1
   "There is 1 field to parse"

2. For field #1:
   
   Read access_flags: 0x0002
   Decode: ACC_PRIVATE (0x0002)
   
   Read name_index: 11
   Lookup constant_pool[11] → CONSTANT_Utf8 → "myField"
   
   Read descriptor_index: 12
   Lookup constant_pool[12] → CONSTANT_Utf8 → "I"
   Parse descriptor: "I" = int (primitive type)
   
   Read attributes_count: 0
   No additional attributes for this field
```

### Field Object Creation:

```
JVM creates Field object:

Field field1 = new Field() {
  access_flags: 0x0002,              // private
  name: "myField",                   // from const_pool[11]
  type: int.class,                   // parsed from "I"
  descriptor: "I",                   // original descriptor string
  
  // Computed during class layout:
  offset: 12,                        // bytes from object header start
  volatility: false,                 // not volatile
  transient: false,                  // not transient
  
  // Metadata:
  declaringClass: SimpleExample.class,
  
  // Initial value (for static fields):
  staticValue: null                  // not static, so null
}

Store in class object:
  SimpleExample.fields[] = [ field1 ]
```

---

## STEP 2: Class File Parsing - Methods Section

### Raw Binary Data (Simplified):

```
METHODS SECTION IN SimpleExample.class:
┌────────────┬──────────────┬──────────┬──────────────┬────────────────┐
│methods_cnt │ access_flags │ name_idx │ descriptor_idx│ attributes_cnt │
├────────────┼──────────────┼──────────┼──────────────┼────────────────┤
│    0x0002  │   0x0001     │  0x0005  │   0x0006     │     0x0001     │
│ (2 methods)│  (public)    │ entry#5  │   entry#6    │   (1 attr)     │
│            │              │"<init>"  │    "()V"     │   Code         │
└────────────┴──────────────┴──────────┴──────────────┴────────────────┘

THEN for each method:

METHOD 1 (Constructor):
┌──────────────┬──────────────┬──────────┬──────────────┬────────────────┬──────────┐
│ access_flags │ name_index   │descriptor│ attr_count   │ attr[0].tag    │attr_data │
├──────────────┼──────────────┼──────────┼──────────────┼────────────────┼──────────┤
│   0x0001     │   0x0005     │  0x0006  │   0x0001     │   "Code"       │ ...      │
│  (public)    │ entry#5      │ entry#6  │   (1 attr)   │  (Code attr)   │ bytecode │
│              │  "<init>"    │  "()V"   │              │                │ & stack  │
└──────────────┴──────────────┴──────────┴──────────────┴────────────────┴──────────┘

METHOD 2 (printField):
┌──────────────┬──────────────┬──────────┬──────────────┬────────────────┬──────────┐
│ access_flags │ name_index   │descriptor│ attr_count   │ attr[0].tag    │attr_data │
├──────────────┼──────────────┼──────────┼──────────────┼────────────────┼──────────┤
│   0x0001     │   0x0017     │  0x0018  │   0x0002     │  "Code"        │ bytecode │
│  (public)    │ entry#23     │ entry#24 │  (2 attrs)   │  "LineNumberTbl"   │ ...      │
│              │"printField"  │  "()V"   │              │                │ line map │
└──────────────┴──────────────┴──────────┴──────────────┴────────────────┴──────────┘
```

### JVM Parsing Action - Method 1 (<init>):

```
JVM Parser reads METHOD 1:

1. Read access_flags: 0x0001
   Decode: ACC_PUBLIC (0x0001)

2. Read name_index: 5
   Lookup constant_pool[5] → CONSTANT_Utf8 → "<init>"
   
3. Read descriptor_index: 6
   Lookup constant_pool[6] → CONSTANT_Utf8 → "()V"
   Parse descriptor: "()V" means:
     - Parameters: () = no parameters
     - Return type: V = void

4. Read attributes_count: 1
   "1 attribute to parse"

5. Parse Code attribute:
   ┌─────────────────────────────────────────┐
   │ attribute_name: "Code"                  │
   │ attribute_length: 0x0019 (25 bytes)     │
   │                                         │
   │ max_stack: 0x0001 (1)                   │
   │   "Method needs 1 stack slot max"       │
   │                                         │
   │ max_locals: 0x0001 (1)                  │
   │   "Local variables: 1 (just 'this')"    │
   │                                         │
   │ code_length: 0x0005 (5 bytes)           │
   │ code: [                                 │
   │   0xAA: aload_0       (load 'this')     │
   │   0xB7: invokespecial (invoke special)  │
   │   0x00: index high byte = 0             │
   │   0x01: index low byte = 1              │
   │          (invoke entry #1 in const pool)│
   │   0xB1: return (void return)            │
   │ ]                                       │
   │                                         │
   │ exception_table_length: 0x0000          │
   │   (no exception handlers)               │
   │                                         │
   │ attributes_count: 0x0001 (1 attribute) │
   │   LineNumberTable: line 1 → bytecode 0 │
   └─────────────────────────────────────────┘
```

### Method Object Creation - Method 1:

```
JVM creates Method object:

Method method1 = new Method() {
  access_flags: 0x0001,           // public
  name: "<init>",                 // from const_pool[5]
  descriptor: "()V",              // from const_pool[6]
  
  // Parsed from Code attribute:
  bytecode: byte[5] { 
    0xAA,        // aload_0 (load this)
    0xB7,        // invokespecial
    0x00, 0x01,  // index 1
    0xB1         // return
  },
  
  max_stack: 1,                   // max operand stack depth needed
  max_locals: 1,                  // local variables (just 'this')
  
  // Computed after parsing:
  parameterTypes: [],             // () = no parameters
  returnType: void.class,         // V = void
  parameterCount: 0,              // no parameters
  
  // JIT/Runtime info:
  nativePointer: null,            // will be compiled to native code
  compiled: false,                // not yet compiled
  
  // Metadata:
  declaringClass: SimpleExample.class,
  lineNumberTable: {              // from LineNumberTable attribute
    line 1 → bytecode offset 0
  }
}

Store in class object:
  SimpleExample.methods[] = [ method1 ]
```

### JVM Parsing Action - Method 2 (printField):

```
JVM Parser reads METHOD 2:

1. Read access_flags: 0x0001
   Decode: ACC_PUBLIC

2. Read name_index: 23
   Lookup constant_pool[23] → "printField"

3. Read descriptor_index: 24
   Lookup constant_pool[24] → "()V"
   Parse: no parameters, returns void

4. Read attributes_count: 2
   "2 attributes to parse"

5. Parse Code attribute:
   ┌─────────────────────────────────────────┐
   │ attribute_name: "Code"                  │
   │                                         │
   │ max_stack: 0x0002 (2)                   │
   │   "Need 2 stack slots"                  │
   │   (for 'this' + int value)              │
   │                                         │
   │ max_locals: 0x0001 (1)                  │
   │   "Local variables: 1 (just 'this')"    │
   │                                         │
   │ code_length: 0x0009 (9 bytes)           │
   │ code: [                                 │
   │   0xAA: aload_0        (load 'this')    │
   │   0xB4: getfield       (get field)      │
   │   0x00: index high = 0                  │
   │   0x07: index low = 7                   │
   │          (field from const_pool[7])     │
   │   0xB6: invokevirtual  (invoke virtual) │
   │   0x00: index high = 0                  │
   │   0x13: index low = 19                  │
   │          (method from const_pool[19])   │
   │   0xB1: return                          │
   │ ]                                       │
   │                                         │
   │ exception_table_length: 0x0000          │
   │                                         │
   │ attributes_count: 0x0001                │
   │   (has LineNumberTable)                 │
   └─────────────────────────────────────────┘

6. Parse LineNumberTable attribute:
   ┌──────────────────────────────────────┐
   │ attribute_name: "LineNumberTable"    │
   │                                      │
   │ entries: [                           │
   │   start_pc: 0 → source_line: 4      │
   │   start_pc: 3 → source_line: 4      │
   │   start_pc: 6 → source_line: 5      │
   │ ]                                    │
   │                                      │
   │ (Maps bytecode offsets to line nums) │
   └──────────────────────────────────────┘
```

### Method Object Creation - Method 2:

```
Method method2 = new Method() {
  access_flags: 0x0001,           // public
  name: "printField",             // from const_pool[23]
  descriptor: "()V",              // from const_pool[24]
  
  // Parsed from Code attribute:
  bytecode: byte[9] {
    0xAA,              // aload_0
    0xB4, 0x00, 0x07,  // getfield #7
    0xB6, 0x00, 0x13,  // invokevirtual #19
    0xB1               // return
  },
  
  max_stack: 2,                   // need 2 stack slots
  max_locals: 1,                  // just 'this'
  
  parameterTypes: [],             // ()
  returnType: void.class,         // V
  parameterCount: 0,
  
  // Metadata:
  declaringClass: SimpleExample.class,
  lineNumberTable: {
    0 → 4,
    3 → 4,
    6 → 5
  }
}

Store in class object:
  SimpleExample.methods[] = [ method1, method2 ]
```

---

## STEP 3: Class Layout - Computing Field Offsets

### Field Layout Algorithm:

```
After parsing all fields, JVM must compute memory offsets for instance fields.

ALGORITHM:
  1. Start with base offset after object header
     (typically 12-16 bytes depending on JVM, varies by compression)
  
  2. Sort fields by size (largest first)
     This minimizes padding and fragmentation
  
  3. For each field, assign next available offset

EXAMPLE FOR SimpleExample:

Source fields (order in .class file):
  Field 1: myField (int)

Sorted by size (largest first):
  Size 4: myField (int)

Layout computation:
  Object header: bytes 0-11 (12 bytes)
  
  myField (int, 4 bytes):
    offset = 12
    size = 4
    occupies bytes 12-15

Final layout:
┌──────────────────────────────────────┐
│ Object Header (12 bytes)             │
│   ├─ Class pointer (8 bytes)         │
│   ├─ Monitor lock (4 bytes)          │
│   └─ (depends on JVM implementation) │
├──────────────────────────────────────┤
│ myField (4 bytes) @ offset 12        │  ← THIS IS WHERE 12 COMES FROM!
├──────────────────────────────────────┤
│ (no more instance fields)            │
├──────────────────────────────────────┤
│ Total size: 16 bytes                 │
└──────────────────────────────────────┘
```

### Update Field Objects with Offsets:

```
JVM updates each Field object:

Before:
  Field field1 {
    name: "myField",
    type: int.class,
    offset: ??? (not yet computed)
  }

After layout:
  Field field1 {
    name: "myField",
    type: int.class,
    offset: 12  ← COMPUTED!
  }

Store in class:
  SimpleExample.fields[0].offset = 12
```

---

## STEP 4: Virtual Method Table (Method Dispatch Table)

### VTable Creation:

```
JVM creates Virtual Method Table for virtual method dispatch:

SimpleExample inherits from Object:
  Object methods:
    - equals(Object):boolean
    - hashCode():int
    - toString():String
    - clone():Object
    - finalize():void
    - (others...)

SimpleExample adds/overrides:
  - printField():void (NEW)
  - <init>():void (OVERRIDES Object's default constructor)

VTable Layout:
┌──────────┬───────────────────────────────────────────────────┐
│ Index    │ Method Reference                                  │
├──────────┼───────────────────────────────────────────────────┤
│ [0]      │ equals(Object):boolean → Object.equals (inherited)│
│ [1]      │ hashCode():int → Object.hashCode (inherited)      │
│ [2]      │ toString():String → Object.toString (inherited)   │
│ [3]      │ clone():Object → Object.clone (inherited)         │
│ [4]      │ finalize():void → Object.finalize (inherited)     │
│ ...      │ (other Object methods)                            │
│ [10]     │ printField():void → SimpleExample.printField ✓NEW│
│ ...      │ (other methods)                                   │
└──────────┴───────────────────────────────────────────────────┘

Each entry is a method pointer:
  - For inherited methods: points to parent's Method object
  - For overridden methods: points to child's Method object
  - For new methods: added to end of table
```

### VTable Stored in Class:

```
SimpleExample.class object {
  ...
  methods[] = [
    Method: <init>():void
    Method: printField():void
  ]
  
  // VTable (for virtual method dispatch)
  methodTable = [
    index 0: pointer to Object.equals
    index 1: pointer to Object.hashCode
    index 2: pointer to Object.toString
    index 3: pointer to Object.clone
    index 4: pointer to Object.finalize
    ...
    index 10: pointer to SimpleExample.printField  ← Direct pointer!
    ...
  ]
}
```

---

## STEP 5: Static Initializer (if any)

```
If class had static fields:
  public static String VERSION = "1.0";

JVM would:
  1. Parse the static field
  2. Create Field object with static flag
  3. Allocate space in the class object
  4. Parse ConstantValue attribute (if present)
  5. Execute <clinit> method during class initialization

But SimpleExample has no static fields, so this step is skipped.
```

---

## Complete Class Object After Parsing

```
SimpleExample CLASS OBJECT IN MEMORY:
┌──────────────────────────────────────────────────────────────┐
│ Class: SimpleExample @ 0x7f8c92c50000                        │
├──────────────────────────────────────────────────────────────┤
│ CLASS METADATA:                                              │
│   className: "SimpleExample"                                │
│   superClass: Object.class                                  │
│   classLoader: sun.misc.Launcher$AppClassLoader             │
│   access_flags: 0x0001 (public)                             │
│                                                              │
│ FIELDS ARRAY:                                               │
│   fields[0] = Field {                                        │
│     name: "myField"                                          │
│     type: int.class                                          │
│     descriptor: "I"                                          │
│     access_flags: 0x0002 (private)                          │
│     offset: 12  ← offset in instance!                       │
│     volatility: false                                        │
│     transient: false                                         │
│   }                                                          │
│                                                              │
│ METHODS ARRAY:                                              │
│   methods[0] = Method {                                      │
│     name: "<init>"                                           │
│     descriptor: "()V"                                        │
│     bytecode: [0xAA, 0xB7, 0x00, 0x01, 0xB1]               │
│     max_stack: 1                                             │
│     max_locals: 1                                            │
│     access_flags: 0x0001 (public)                           │
│     declaringClass: SimpleExample.class                      │
│     nativePointer: null (not yet JIT compiled)              │
│   }                                                          │
│                                                              │
│   methods[1] = Method {                                      │
│     name: "printField"                                       │
│     descriptor: "()V"                                        │
│     bytecode: [0xAA, 0xB4, 0x00, 0x07, 0xB6, 0x00, 0x13...] │
│     max_stack: 2                                             │
│     max_locals: 1                                            │
│     access_flags: 0x0001 (public)                           │
│     declaringClass: SimpleExample.class                      │
│     nativePointer: null (not yet JIT compiled)              │
│     lineNumberTable: [0→4, 3→4, 6→5]                        │
│   }                                                          │
│                                                              │
│ METHOD TABLE (VTable):                                      │
│   methodTable[0]: → Object.equals                           │
│   methodTable[1]: → Object.hashCode                         │
│   methodTable[2]: → Object.toString                         │
│   ... (inherited methods)                                   │
│   methodTable[10]: → SimpleExample.printField  ← Direct!    │
│                                                              │
│ STATIC FIELDS:                                              │
│   (none - no static fields in SimpleExample)                │
│                                                              │
│ CONSTANT POOL:                                              │
│   (same as the .class file's constant pool, now in memory)  │
│                                                              │
│ CLASS INITIALIZATION:                                       │
│   initialized: true                                         │
│   initializing: false                                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Flow: From Binary to In-Memory Class Object

```
SimpleExample.class FILE (Binary):
┌──────────────────────────────────────────────────────────────┐
│ Hex bytes representing:                                      │
│   FIELDS section: 0x0001 0x0002 0x000B 0x000C 0x0000        │
│   METHODS section: 0x0002 [method1 data] [method2 data]     │
│   (+ constant pool references, etc.)                        │
└──────────────────────────────────────────────────────────────┘
          ↓ (JVM ClassFileParser.parseFields())
┌──────────────────────────────────────────────────────────────┐
│ Parse Fields Section:                                        │
│   Read field_count, access_flags, name_index, descriptor... │
│   Look up strings in constant pool                          │
│   Create Field objects                                      │
└──────────────────────────────────────────────────────────────┘
          ↓ (JVM FieldLayout.computeOffsets())
┌──────────────────────────────────────────────────────────────┐
│ Compute Field Offsets:                                       │
│   Sort by size, calculate positions                         │
│   Assign offset=12 to myField                               │
└──────────────────────────────────────────────────────────────┘
          ↓ (JVM ClassFileParser.parseMethods())
┌──────────────────────────────────────────────────────────────┐
│ Parse Methods Section:                                       │
│   Read method_count, access_flags, name_index, descriptor..│
│   Parse Code attribute (bytecode, stack info, etc.)         │
│   Create Method objects with bytecode                       │
└──────────────────────────────────────────────────────────────┘
          ↓ (JVM ClassLoader.createVtable())
┌──────────────────────────────────────────────────────────────┐
│ Build Virtual Method Table:                                  │
│   Inherit methods from superclass                           │
│   Add/override methods from this class                      │
│   Build dispatch table                                      │
└──────────────────────────────────────────────────────────────┘
          ↓ (JVM InstanceKlass.initialize())
┌──────────────────────────────────────────────────────────────┐
│ Class Object Created in Memory:                              │
│   ├─ className: "SimpleExample"                             │
│   ├─ fields[]: [Field with offset=12]                       │
│   ├─ methods[]: [Method <init>, Method printField]          │
│   ├─ methodTable[]: [vtable pointers]                       │
│   └─ (other metadata)                                       │
│                                                              │
│ Ready for use! Can now:                                      │
│   - Create instances                                        │
│   - Access fields (using offsets)                           │
│   - Call methods (using vtable)                             │
└──────────────────────────────────────────────────────────────┘
```

---

## How Offsets Are Used at Runtime

### When Creating an Instance:

```
Code: SimpleExample obj = new SimpleExample();

JVM execution:
1. Allocate memory for instance
   Size = 16 bytes (header 12 + fields 4)
   Memory address: 0x7f8e12345678

2. Initialize object header
   0x7f8e12345678 (0-7): ClassPointer → SimpleExample.class
   0x7f8e12345680 (8-11): MonitorLock → 0

3. Initialize fields using offsets
   offset = 12
   0x7f8e12345678 + 12 = 0x7f8e1234568C
   Write int value 0 (default for int)

4. Call constructor
   Execute SimpleExample.<init>() bytecode

Final instance:
┌────────────────────────────────────────────────┐
│ Instance @ 0x7f8e12345678                      │
├────────────────────────────────────────────────┤
│ Header (0-11):                                 │
│   ClassPtr: SimpleExample.class                │
│   Monitor: 0                                   │
│                                                │
│ myField (offset 12-15): 0                      │
│                                                │
│ Total size: 16 bytes                           │
└────────────────────────────────────────────────┘
```

### When Accessing a Field:

```
Bytecode: getfield #7  (accesses myField)

JVM execution:
1. Resolve constant_pool[7]
   → Already resolved to:
     - Class: SimpleExample.class
     - Field: myField
     - Offset: 12

2. Get instance pointer from stack: obj @ 0x7f8e12345678

3. Calculate field address:
   field_address = 0x7f8e12345678 + 12
                 = 0x7f8e1234568C

4. Read int value from that address
   value = *(int*)0x7f8e1234568C = 0 (or whatever value was stored)

5. Push value onto stack
```

---

## Summary: What Gets Created

```
INPUT: SimpleExample.class binary file

PARSING OUTPUT:

1. FIELD OBJECTS
   ├─ Name: "myField"
   ├─ Type: int.class
   ├─ Offset: 12 bytes from object start ← COMPUTED
   ├─ Access flags: private
   └─ Stored in: SimpleExample.fields[]

2. METHOD OBJECTS
   ├─ Name: "<init>"
   ├─ Bytecode: [actual bytes]
   ├─ Max stack: 1
   ├─ Max locals: 1
   ├─ Metadata: source line mapping
   └─ Stored in: SimpleExample.methods[0]
   
   ├─ Name: "printField"
   ├─ Bytecode: [actual bytes]
   ├─ Max stack: 2
   ├─ Max locals: 1
   ├─ Metadata: source line mapping
   └─ Stored in: SimpleExample.methods[1]

3. VIRTUAL METHOD TABLE
   ├─ Index pointers to inherited methods from Object
   ├─ Index pointers to SimpleExample methods
   └─ Stored in: SimpleExample.methodTable[]

4. CLASS OBJECT
   ├─ Contains all of above
   ├─ Ready for instantiation
   ├─ Ready for method invocation
   └─ Stored in: Classloader cache
```

This is how raw binary data becomes a usable in-memory class object!
