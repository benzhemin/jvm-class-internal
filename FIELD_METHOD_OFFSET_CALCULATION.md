# Field and Method Offset Calculation at Runtime

## Overview: Two Different Types of Offsets

When a class instance is created, the JVM uses TWO different offset calculations:

```
┌─────────────────────────────────────────────────────────────────┐
│                      OFFSET TYPES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. FIELD OFFSET                                               │
│     ├─ Where in memory a field value is stored                │
│     ├─ Calculated once during class loading                    │
│     ├─ Used repeatedly when accessing fields                   │
│     └─ Example: myField at offset 12 (bytes from object start)│
│                                                                 │
│  2. METHOD OFFSET (in bytecode)                               │
│     ├─ Where in the bytecode a method instruction is          │
│     ├─ Calculated once during method parsing                   │
│     ├─ Used when traversing code/debugging                     │
│     └─ Example: bytecode offset 3 = start of instruction      │
│                                                                 │
│  3. VTABLE INDEX (method dispatch)                            │
│     ├─ Which slot in the vtable for virtual method dispatch    │
│     ├─ Calculated during class loading                         │
│     ├─ Used during method invocation                           │
│     └─ Example: printField at vtable[10]                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## PART 1: FIELD OFFSET CALCULATION

### What is Field Offset?

Field offset is the **byte distance from the start of an object instance to the actual field data**.

```
SimpleExample instance in memory:
┌────────────────────────────────────────────────────────┐
│ Instance Start: 0x7f8e12345678                         │
├────────────────────────────────────────────────────────┤
│ Bytes 0-7:   Class Pointer                            │
│              (reference to SimpleExample.class)        │
├────────────────────────────────────────────────────────┤
│ Bytes 8-11:  Monitor/Lock information                 │
├────────────────────────────────────────────────────────┤
│ Bytes 12-15: myField (int)  ← OFFSET 12               │
│              This is where the int value is stored     │
├────────────────────────────────────────────────────────┤
│ ... (any additional fields would go here)             │
└────────────────────────────────────────────────────────┘

offset = 12 means:
  Field memory address = Instance base address + 12
```

### How Field Offsets are Calculated (During Class Loading)

#### Step 1: Parse All Fields from .class File

```
SimpleExample.class file contains:

FIELDS SECTION:
┌─────────────────────────────────────────┐
│ field_count: 0x0001                     │
│ Field 1:                                │
│   access_flags: 0x0002 (private)        │
│   name_index: 11 ("myField")            │
│   descriptor_index: 12 ("I")            │
│   attributes_count: 0                   │
└─────────────────────────────────────────┘

JVM creates Field objects:
  Field[0] = {
    name: "myField",
    type: int.class,
    size: 4 bytes,
    access_flags: 0x0002,
    offset: ??? (to be calculated)
  }
```

#### Step 2: Gather All Fields (Including Inherited)

```
SimpleExample inherits from Object, but Object has no instance fields.
So total fields = just the ones defined in SimpleExample.

All instance fields to layout:
  ├─ From Object: (none)
  └─ From SimpleExample:
      └─ myField (int, 4 bytes)
```

#### Step 3: Determine Object Header Size

```
JVM checks object header layout (varies by JVM implementation):

On a 64-bit JVM (typical):
┌────────────────────────────────────┐
│ Class Pointer: 8 bytes             │
├────────────────────────────────────┤
│ Monitor/HashCode: 8 bytes          │ (can be compressed)
├────────────────────────────────────┤
│ Total header: 16 bytes             │
└────────────────────────────────────┘

On some JVMs with pointer compression:
┌────────────────────────────────────┐
│ Class Pointer: 4 bytes (compressed)│
├────────────────────────────────────┤
│ Monitor/HashCode: 4 bytes          │
├────────────────────────────────────┤
│ Total header: 8 bytes              │
└────────────────────────────────────┘

For SimpleExample, we'll use:
  header_size = 12 bytes (common in older JVMs)
```

#### Step 4: Sort Fields by Size (Field Layout Strategy)

```
WHY SORT?
  - Minimize memory fragmentation
  - Improve cache efficiency
  - Group similar-sized fields together

EXAMPLE: If SimpleExample had multiple fields:
  
  Before sorting (declaration order):
    Field 1: byte myByte (1 byte)
    Field 2: long myLong (8 bytes)
    Field 3: int myInt (4 bytes)

  After sorting (by size, largest first):
    Field 1: long myLong (8 bytes)  ← Largest
    Field 2: int myInt (4 bytes)
    Field 3: byte myByte (1 byte)   ← Smallest

  This avoids padding waste:
    ✓ Good layout:  LLLLLLLL IIII B___ (3 bytes wasted)
    ✗ Bad layout:   B_ LL... IIII (more fragmentation)

For SimpleExample (only one field):
  After sorting: myField (int, 4 bytes)
```

#### Step 5: Assign Field Offsets

```
ALGORITHM:
  current_offset = header_size
  
  for each field (in sorted order):
    field.offset = current_offset
    current_offset += field.size

CALCULATION FOR SimpleExample:

  header_size = 12
  current_offset = 12

  Field: myField (int, 4 bytes)
    myField.offset = 12
    current_offset = 12 + 4 = 16

  instance_size = 16 bytes

RESULT:
  Field offsets calculated:
  ├─ myField: offset = 12
  
  Instance size: 16 bytes
```

#### Step 6: Store Calculated Offsets in Class Object

```
BEFORE offset calculation:
  SimpleExample.class {
    fields[0] = Field {
      name: "myField",
      type: int.class,
      size: 4,
      offset: UNKNOWN  ← Not yet set
    }
    instance_size: UNKNOWN
  }

AFTER offset calculation:
  SimpleExample.class {
    fields[0] = Field {
      name: "myField",
      type: int.class,
      size: 4,
      offset: 12  ← CALCULATED!
    }
    instance_size: 16  ← CALCULATED!
  }
```

### Complete Field Offset Example with Multiple Fields

Let's trace through a more complex example:

```
Imagine SimpleExample had these fields:

  private boolean myBoolean;  (1 byte)
  private long myLong;        (8 bytes)
  private int myInt;          (4 bytes)
  private byte myByte;        (1 byte)

STEP 1: Parse from .class file
  Field[0] = { name: "myBoolean", type: boolean, size: 1 }
  Field[1] = { name: "myLong", type: long, size: 8 }
  Field[2] = { name: "myInt", type: int, size: 4 }
  Field[3] = { name: "myByte", type: byte, size: 1 }

STEP 2: Sort by size (largest first)
  Field[0] = { name: "myLong", type: long, size: 8 }     ← Largest
  Field[1] = { name: "myInt", type: int, size: 4 }
  Field[2] = { name: "myBoolean", type: boolean, size: 1 }
  Field[3] = { name: "myByte", type: byte, size: 1 }     ← Smallest

STEP 3: Calculate offsets
  header_size = 12
  current_offset = 12

  Field "myLong" (8 bytes):
    offset = 12
    current_offset = 12 + 8 = 20

  Field "myInt" (4 bytes):
    offset = 20
    current_offset = 20 + 4 = 24

  Field "myBoolean" (1 byte):
    offset = 24
    current_offset = 24 + 1 = 25

  Field "myByte" (1 byte):
    offset = 25
    current_offset = 25 + 1 = 26

  instance_size = 26 bytes

STEP 4: Memory layout
┌─────────────────────────────────────────────────────┐
│ Object Instance (26 bytes)                          │
├─────────────────────────────────────────────────────┤
│ Bytes 0-7:   Class Pointer (8 bytes)               │
├─────────────────────────────────────────────────────┤
│ Bytes 8-11:  Monitor (4 bytes)                      │
├─────────────────────────────────────────────────────┤
│ Bytes 12-19: myLong (8 bytes)  @ offset 12         │
├─────────────────────────────────────────────────────┤
│ Bytes 20-23: myInt (4 bytes)   @ offset 20         │
├─────────────────────────────────────────────────────┤
│ Byte 24:     myBoolean (1 byte) @ offset 24        │
├─────────────────────────────────────────────────────┤
│ Byte 25:     myByte (1 byte)    @ offset 25        │
├─────────────────────────────────────────────────────┤
│ Total: 26 bytes                                     │
└─────────────────────────────────────────────────────┘
```

---

## PART 2: USING FIELD OFFSETS AT INSTANCE CREATION

### Step 1: Allocate Memory

```
Bytecode: new SimpleExample()

JVM execution:

1. Look up SimpleExample.class
   SimpleExample.class.instance_size = 16 bytes

2. Allocate memory from heap
   Allocate 16 bytes
   Base address: 0x7f8e12345678

3. Zero-initialize memory
   Memory[0x7f8e12345678 ... 0x7f8e12345687] = 0
```

### Step 2: Set Class Pointer

```
2. Set class pointer in object header
   
   Field location: offset 0 (first field in header)
   Address: 0x7f8e12345678 + 0 = 0x7f8e12345678
   
   Value to write: SimpleExample.class
   
   Memory state:
   ┌────────────────────────────────────────────────────┐
   │ Bytes 0-7 (8 bytes): 0x7f8c92c50000               │
   │                      (SimpleExample.class pointer) │
   ├────────────────────────────────────────────────────┤
   │ Bytes 8-11: 0x00000000 (monitor lock)              │
   ├────────────────────────────────────────────────────┤
   │ Bytes 12-15: 0x00000000 (field data, not set yet) │
   └────────────────────────────────────────────────────┘
```

### Step 3: Initialize Instance Fields with Default Values

```
3. Initialize all instance fields with default values
   
   For each field in SimpleExample.fields[]:
   
   Field: myField (int)
     offset = 12
     size = 4
     address = base + offset
             = 0x7f8e12345678 + 12
             = 0x7f8e1234568C
     
     default value for int = 0
     *(int*)0x7f8e1234568C = 0
   
   Memory state:
   ┌────────────────────────────────────────────────────┐
   │ Bytes 0-7:   0x7f8c92c50000 (class pointer)       │
   ├────────────────────────────────────────────────────┤
   │ Bytes 8-11:  0x00000000 (monitor)                  │
   ├────────────────────────────────────────────────────┤
   │ Bytes 12-15: 0x00000000 (myField = 0)             │
   └────────────────────────────────────────────────────┘
```

### Step 4: Call Constructor

```
4. Call <init>() constructor
   
   Bytecode: invokespecial #1  (Object.<init>)
   
   The constructor may assign values to fields:
   Example constructor bytecode:
     aload_0          (load 'this')
     bipush 42        (push constant 42)
     putfield #7      (assign to myField)
     return
   
   After constructor executes:
   ┌────────────────────────────────────────────────────┐
   │ Bytes 0-7:   0x7f8c92c50000 (class pointer)       │
   ├────────────────────────────────────────────────────┤
   │ Bytes 8-11:  0x00000000 (monitor)                  │
   ├────────────────────────────────────────────────────┤
   │ Bytes 12-15: 0x0000002A (myField = 42)            │
   │              (0x2A = 42 in hex)                    │
   └────────────────────────────────────────────────────┘
```

---

## PART 3: USING FIELD OFFSETS AT FIELD ACCESS

### Scenario: getfield #7 (access myField)

```
Bytecode:
  getfield #7

Instruction execution:

STEP 1: Resolve constant pool entry #7
  constant_pool[7] = Fieldref (RESOLVED)
  {
    resolved_class: SimpleExample.class
    resolved_field: SimpleExample.fields[0] (myField)
    field_offset: 12  ← THIS IS KEY!
  }

STEP 2: Get object instance from operand stack
  obj_address = stack.pop()  = 0x7f8e12345678

STEP 3: Calculate field address
  field_address = obj_address + field_offset
                = 0x7f8e12345678 + 12
                = 0x7f8e1234568C

STEP 4: Read field value
  field_value = *(int*)field_address
              = *(int*)0x7f8e1234568C
              = 42  (or whatever value was stored)

STEP 5: Push to stack
  stack.push(field_value)  = stack.push(42)
```

### Scenario: putfield #7 (assign to myField)

```
Bytecode:
  putfield #7

Instruction execution:

STEP 1: Resolve constant pool entry #7
  field_offset = 12

STEP 2: Pop values from stack
  new_value = stack.pop()  = 99
  obj_address = stack.pop() = 0x7f8e12345678

STEP 3: Calculate field address
  field_address = 0x7f8e12345678 + 12
                = 0x7f8e1234568C

STEP 4: Write field value
  *(int*)field_address = new_value
  *(int*)0x7f8e1234568C = 99

Memory after:
┌────────────────────────────────────────────────────┐
│ Bytes 12-15: 0x00000063 (myField = 99)            │
│              (0x63 = 99 in hex)                    │
└────────────────────────────────────────────────────┘
```

---

## PART 4: METHOD OFFSET CALCULATION (BYTECODE LEVEL)

### What is Method Bytecode Offset?

Method offset is the **byte position within a method's bytecode array**.

```
printField method bytecode:
┌────────────────────────────────────┐
│ Bytecode array (9 bytes total)     │
├────────────────────────────────────┤
│ Offset 0: 0xAA (aload_0)          │ ← Instruction 1
├────────────────────────────────────┤
│ Offset 1: 0xB4 (getfield)         │ ← Instruction 2 (wide)
│ Offset 2: 0x00 (high byte of arg) │
│ Offset 3: 0x07 (low byte of arg)  │
├────────────────────────────────────┤
│ Offset 4: 0xB6 (invokevirtual)    │ ← Instruction 3 (wide)
│ Offset 5: 0x00 (high byte of arg) │
│ Offset 6: 0x13 (low byte of arg)  │
├────────────────────────────────────┤
│ Offset 7: 0xB1 (return)           │ ← Instruction 4
├────────────────────────────────────┤
│ Offset 8: (padding/unused)        │
└────────────────────────────────────┘
```

### How Bytecode Offsets are Calculated (During Parsing)

```
Parsing Algorithm:

  bytecode = [0xAA, 0xB4, 0x00, 0x07, 0xB6, 0x00, 0x13, 0xB1]
  
  offset = 0
  
  while (offset < bytecode.length) {
    opcode = bytecode[offset]
    
    // Look up instruction info
    instruction_info = OPCODE_TABLE[opcode]
    instruction_size = instruction_info.size
    
    // Record this instruction
    instruction[offset] = {
      opcode: opcode,
      offset: offset,
      size: instruction_size,
      ... (other info)
    }
    
    // Move to next instruction
    offset += instruction_size
  }

EXAMPLE TRACE:

  offset = 0
  opcode = 0xAA (aload_0)
  size = 1
  Record: instruction[0] = aload_0
  offset = 0 + 1 = 1

  offset = 1
  opcode = 0xB4 (getfield)
  size = 3 (1 opcode + 2 byte index)
  Record: instruction[1] = getfield #index
  offset = 1 + 3 = 4

  offset = 4
  opcode = 0xB6 (invokevirtual)
  size = 3 (1 opcode + 2 byte index)
  Record: instruction[4] = invokevirtual #index
  offset = 4 + 3 = 7

  offset = 7
  opcode = 0xB1 (return)
  size = 1
  Record: instruction[7] = return
  offset = 7 + 1 = 8

  Done! All instructions parsed.
```

### Bytecode Offset Usage

```
Purpose: Line number debugging, exception handling, etc.

Example: User steps through code in debugger

  Source line 4:
    myField = 42;           ← Points to bytecode offset 0-3

  Source line 5:
    System.out.println(myField);  ← Points to bytecode offset 4-7

The LineNumberTable attribute maps:
  bytecode_offset → source_line_number

LineNumberTable for printField:
┌─────────────────────┐
│ offset 0 → line 4  │
│ offset 3 → line 4  │
│ offset 4 → line 5  │
│ offset 7 → line 5  │
└─────────────────────┘

This allows debugger to show which source line you're on.
```

---

## PART 5: VTABLE INDEX CALCULATION (METHOD DISPATCH)

### What is VTable Index?

VTable index is the **slot number in the virtual method table** used to find the correct method implementation.

```
VTable layout:

┌────────────────┬──────────────────────────────────────┐
│ VTable Index   │ Method                               │
├────────────────┼──────────────────────────────────────┤
│ [0]            │ Object.equals(Object):boolean        │
│ [1]            │ Object.hashCode():int                │
│ [2]            │ Object.toString():String             │
│ [3]            │ Object.clone():Object                │
│ [4]            │ Object.finalize():void               │
│ ...            │ (other Object methods)               │
│ [10]           │ SimpleExample.printField():void      │
│ ...            │ (other SimpleExample methods)        │
└────────────────┴──────────────────────────────────────┘
```

### How VTable Indices are Assigned (During Class Loading)

```
ALGORITHM:

1. Start with parent's vtable
   Object has N methods → vtable slots [0..N-1]

2. For each method in SimpleExample:
   
   If method overrides parent method:
     Use same slot (replaces parent)
   
   If method is new (not in parent):
     Assign new slot (append to vtable)

EXAMPLE FOR SimpleExample:

Step 1: Get Object's vtable
  Object.vtable[0] = equals
  Object.vtable[1] = hashCode
  Object.vtable[2] = toString
  Object.vtable[3] = clone
  Object.vtable[4] = finalize
  ... (11 total methods from Object)
  
  SimpleExample.vtable = copy of Object.vtable

Step 2: Check SimpleExample's methods
  
  Method: <init>():void
    Check: Does Object have <init>?
    No, but it's constructor, special handling
    vtable_index = N/A (constructors don't use vtable)
  
  Method: printField():void
    Check: Does Object have printField?
    No, new method
    Assign new slot: vtable_index = 11  (next available)
    
Step 3: Final vtable
  SimpleExample.vtable[10] = Object.equals
  SimpleExample.vtable[11] = Object.hashCode
  ...
  SimpleExample.vtable[11] = SimpleExample.printField  ← NEW!
```

### VTable Index Usage at Method Invocation

```
Bytecode: invokevirtual #19

This is calling println(int) on PrintStream

Execution:

STEP 1: Resolve constant pool #19
  constant_pool[19] = Methodref
  {
    method: java/io/PrintStream.println(I)V
    vtable_index: 42  (or whatever println's index is)
  }

STEP 2: Get object from stack
  obj = System.out  (the PrintStream instance)
  
STEP 3: Get actual class of object
  actual_class = obj.class
  (If obj is PrintStream, actual_class is PrintStream.class)
  (If obj is subclass of PrintStream, actual_class is the subclass)
  
STEP 4: Look up method in vtable
  method = actual_class.vtable[vtable_index]
  
STEP 5: Call the method
  method.invoke(obj, arguments)

WHY VTABLE?
  ✓ Handles polymorphism
  ✓ Each subclass can override with its own implementation
  ✓ Still get O(1) method lookup (array index)
```

### VTable Index Calculation Example

```
Consider a subclass hierarchy:

  Animal (abstract)
    ├─ makeSound():void  → vtable[0]
    └─ move():void       → vtable[1]

  Dog extends Animal
    ├─ makeSound():void  → Overrides Animal, vtable[0]
    └─ fetch():void      → New method, vtable[2]

  Cat extends Animal
    ├─ makeSound():void  → Overrides Animal, vtable[0]
    └─ scratch():void    → New method, vtable[2] (same slot as Dog.fetch!)
                          (No conflict because different classes)

Polymorphic call:

  Animal animal = getDogOrCat();
  animal.makeSound();  // Calls Dog.makeSound or Cat.makeSound
  
  How does it work?
  
  1. Look up "makeSound" in Animal's vtable definition
     vtable_index = 0
  
  2. Get actual class: actual_class = animal.class
     (Could be Dog or Cat)
  
  3. Get method from actual class's vtable[0]
     If animal is Dog: method = Dog.vtable[0] → Dog.makeSound
     If animal is Cat: method = Cat.vtable[0] → Cat.makeSound
  
  4. Call the correct method!
     This is polymorphism!
```

---

## PART 6: COMPLETE EXAMPLE - FROM SOURCE TO EXECUTION

### Source Code

```java
public class Account {
    private long balance;        // 8 bytes
    private String owner;        // reference (8 bytes)
    private int accountNumber;   // 4 bytes
    
    public Account(long initialBalance, String name) {
        this.balance = initialBalance;
        this.owner = name;
        this.accountNumber = generateNumber();
    }
    
    public void withdraw(long amount) {
        this.balance -= amount;
    }
}
```

### After Compilation (Simplified .class structure)

```
FIELDS SECTION:
┌─────────────────────────────────────┐
│ Field 1: balance (long, 8 bytes)    │
│ Field 2: owner (String, 8 bytes)    │
│ Field 3: accountNumber (int, 4 bytes)│
└─────────────────────────────────────┘

METHODS SECTION:
┌─────────────────────────────────────┐
│ Method 1: <init>(long, String)V     │
│ Method 2: withdraw(long)V           │
└─────────────────────────────────────┘
```

### Class Loading - Field Offset Calculation

```
Step 1: Parse fields
  Field[0] = { name: "balance", type: long, size: 8 }
  Field[1] = { name: "owner", type: String, size: 8 }
  Field[2] = { name: "accountNumber", type: int, size: 4 }

Step 2: Sort by size (largest first)
  Field[0] = { name: "balance", type: long, size: 8 }
  Field[1] = { name: "owner", type: String, size: 8 }
  Field[2] = { name: "accountNumber", type: int, size: 4 }

Step 3: Assign offsets
  header_size = 12
  
  balance:
    offset = 12
    next = 12 + 8 = 20
  
  owner:
    offset = 20
    next = 20 + 8 = 28
  
  accountNumber:
    offset = 28
    next = 28 + 4 = 32

Step 4: Final class object
  Account.class.instance_size = 32 bytes
  Account.fields[0].offset = 12  (balance)
  Account.fields[1].offset = 20  (owner)
  Account.fields[2].offset = 28  (accountNumber)
```

### Instance Creation

```
Code:
  Account acc = new Account(1000L, "John");

JVM execution:

1. Allocate 32 bytes from heap
   base_address = 0x7f8e12345000

2. Set class pointer
   *(long*)0x7f8e12345000 = Account.class

3. Initialize fields with defaults
   *(long*)(0x7f8e12345000 + 12) = 0L      // balance
   *(long*)(0x7f8e12345000 + 20) = null    // owner
   *(int*)(0x7f8e12345000 + 28) = 0        // accountNumber

4. Call <init>(1000L, "John")
   
   Bytecode for <init>:
     aload_0              // load this
     lload_1              // load balance (1000L)
     putfield balance[12] // this.balance = 1000L
     
     aload_0              // load this
     aload_2              // load owner ("John")
     putfield owner[20]   // this.owner = "John"
     
     aload_0              // load this
     invokespecial generateNumber() // call method
     putfield accountNumber[28]     // this.accountNumber = result

Memory after construction:
┌───────────────────────────────────────────────────┐
│ Bytes 0-7:     0x7f8c92c50000 (Account.class)     │
│ Bytes 8-11:    0x00000000 (monitor)               │
│                                                   │
│ Bytes 12-19:   0x00000000000003E8 (balance=1000) │
│                (0x3E8 = 1000 in hex)              │
│                                                   │
│ Bytes 20-27:   0x7f8e54321000 (owner="John")     │
│                (reference to String object)      │
│                                                   │
│ Bytes 28-31:   0x000012AB (accountNumber=4779)   │
│                                                   │
│ Total: 32 bytes                                   │
└───────────────────────────────────────────────────┘
```

### Field Access During Method Execution

```
Code in withdraw(long amount):
  this.balance -= amount;

Bytecode:
  aload_0              // load this (onto stack)
  aload_0              // load this again
  getfield balance[12] // get balance value
  lload_1              // load amount parameter
  lsub                 // subtract
  putfield balance[12] // set balance

Execution trace:

1. aload_0 → Push obj @ 0x7f8e12345000

2. aload_0 → Push obj @ 0x7f8e12345000

3. getfield balance[12]
   obj = pop() = 0x7f8e12345000
   offset = 12
   field_address = 0x7f8e12345000 + 12 = 0x7f8e12345000 + 12
   value = *(long*)field_address = 1000L
   push(1000L)

4. lload_1 → Push amount = 100L

5. lsub → Pop 100, pop 1000, push (1000-100) = 900L

6. putfield balance[12]
   new_value = pop() = 900L
   obj = pop() = 0x7f8e12345000
   offset = 12
   field_address = 0x7f8e12345000 + 12
   *(long*)field_address = 900L
```

---

## Summary Table: Offset Types and Calculations

```
┌────────────────┬──────────────────┬──────────────────┬─────────────────┐
│ Offset Type    │ Calculated When  │ What It Is        │ Used For        │
├────────────────┼──────────────────┼──────────────────┼─────────────────┤
│                │                  │                  │                 │
│ FIELD OFFSET   │ Class loading    │ Byte distance    │ Field access    │
│ (e.g., 12)     │ (once per class) │ from object      │ getfield        │
│                │                  │ start to field   │ putfield        │
│                │                  │ data             │                 │
│                │                  │                  │                 │
├────────────────┼──────────────────┼──────────────────┼─────────────────┤
│                │                  │                  │                 │
│ METHOD OFFSET  │ Method parsing   │ Byte position    │ Bytecode        │
│ (e.g., 0, 3, 7)│ (once per method)│ within bytecode  │ traversal       │
│                │                  │ array            │ Debug/Exception │
│                │                  │                  │ handling        │
│                │                  │                  │                 │
├────────────────┼──────────────────┼──────────────────┼─────────────────┤
│                │                  │                  │                 │
│ VTABLE INDEX   │ Class loading    │ Slot number in   │ Virtual method  │
│ (e.g., 0, 10)  │ (once per class) │ virtual method   │ dispatch        │
│                │                  │ table            │ invokevirtual   │
│                │                  │                  │                 │
└────────────────┴──────────────────┴──────────────────┴─────────────────┘
```

---

## Key Points Summary

```
1. FIELD OFFSETS are calculated ONCE during class loading
   ├─ Parse all fields from .class file
   ├─ Sort by size (largest first)
   ├─ Assign offsets relative to object header
   └─ Used repeatedly when accessing fields

2. FIELD OFFSETS enable fast memory access
   ├─ Simple arithmetic: base_address + offset
   ├─ No hashtable lookup needed
   ├─ O(1) access time
   └─ Works for both reading and writing

3. METHOD OFFSETS are in bytecode, not field access
   ├─ Different concept from field offsets
   ├─ Used for debugging, not field access
   └─ Not as critical for performance

4. VTABLE enables polymorphism with fast dispatch
   ├─ Each subclass can override methods
   ├─ Still get O(1) lookup (array index)
   ├─ Runtime checks actual object's class
   └─ Calls correct implementation

5. ALL CALCULATIONS happen at LOAD TIME
   ├─ Class loading: field offsets, vtable
   ├─ Method parsing: bytecode offsets
   └─ Runtime: just uses pre-calculated values
```
