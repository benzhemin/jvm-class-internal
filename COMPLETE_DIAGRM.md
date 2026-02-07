┌─────────────────────────────────────────────────────────────────┐
│ PHASE 1: COMPILE (javac)                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Input:  SimpleExample.java                                      │
│ Output: SimpleExample.class (binary)                            │
│                                                                 │
│ Constant Pool (symbolic):                                       │
│   #7 = Fieldref(class=8, nameandtype=9)                        │
│   #8 = Class(name=10)                                          │
│   #9 = NameAndType(name=11, desc=12)                          │
│   #10 = Utf8("SimpleExample")                                  │
│   #11 = Utf8("myField")                                        │
│   #12 = Utf8("I")                                              │
│                                                                 │
│ Bytecode:                                                       │
│   getfield #7  ← Just references constant pool index          │
│                                                                 │
│ Status: ALL SYMBOLIC, NO RESOLVED DATA YET                    │
└─────────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 2: CLASS LOADING (JVM ClassLoader)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Input:  SimpleExample.class (binary file)                      │
│ Output: InstanceKlass object in memory                         │
│                                                                 │
│ Step 1: Parse FIELDS Section                                   │
│   Field: myField (int, 4 bytes)                                │
│   ├─ Sort by size                                              │
│   └─ Calculate offset = 12  ✓ KEY!                             │
│                                                                 │
│ Step 2: Parse METHODS Section                                  │
│   Method: <init>, printField                                   │
│   └─ Attach bytecode info                                      │
│                                                                 │
│ Step 3: Build VTable                                           │
│   ├─ Inherit Object methods [0..10]                            │
│   └─ Add printField at [11]                                    │
│                                                                 │
│ Step 4: Create Class Metadata (InstanceKlass)                  │
│   SimpleExample.class {                                         │
│     instance_size: 16  ✓                                        │
│     fields[0].offset: 12  ✓                                     │
│     methods[]: [<init>, printField]                            │
│     vtable[]: [Object methods..., printField]                  │
│     constant_pool[]: [unresolved entries]                      │
│   }                                                             │
│                                                                 │
│ Status: CLASS METADATA READY, CONSTANT POOL STILL SYMBOLIC    │
└─────────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 3: LINKING/RESOLUTION (JVM)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ For each constant pool entry (lazy or eager):                  │
│                                                                 │
│ Entry #7 (Fieldref) RESOLUTION:                               │
│   Before:                                                       │
│     #7 = Fieldref(class_idx=8, nameandtype_idx=9)             │
│   Process:                                                      │
│     1. Load class #8 → SimpleExample.class ✓                   │
│     2. Find field #11:"myField" in fields[] ✓                  │
│     3. Get offset from Field object: 12 ✓                      │
│   After:                                                        │
│     #7 = Fieldref {                                            │
│       resolved_class: SimpleExample.class,                     │
│       resolved_field: Field "myField",                         │
│       field_offset: 12,                                        │
│       resolved: true  ✓ KEY!                                   │
│     }                                                          │
│                                                                 │
│ Entry #19 (Methodref) RESOLUTION:                             │
│   Before:                                                       │
│     #19 = Methodref(class_idx=20, nameandtype_idx=21)        │
│   Process:                                                      │
│     1. Load class #20 → PrintStream.class ✓                    │
│     2. Find method #23:"println(I)V" in methods[] ✓            │
│     3. Get vtable_index from VTable: 42 ✓                      │
│   After:                                                        │
│     #19 = Methodref {                                          │
│       resolved_class: PrintStream.class,                       │
│       resolved_method: Method "println(I)V",                   │
│       vtable_index: 42,                                        │
│       resolved: true  ✓ KEY!                                   │
│     }                                                          │
│                                                                 │
│ Status: CONSTANT POOL NOW FULLY RESOLVED                      │
└─────────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 4: INSTANCE CREATION                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ new SimpleExample():                                            │
│   1. Get SimpleExample.class from classloader cache            │
│   2. Get instance_size = 16 bytes  ✓ From class loading       │
│   3. Allocate 16 bytes @ 0x7f8e12345678                        │
│   4. Initialize:                                                │
│      ├─ Class pointer @ 0                                      │
│      ├─ Monitor @ 8                                            │
│      └─ myField @ 12 (offset from Field object) ✓             │
│   5. Call constructor                                          │
│                                                                 │
│ Instance Memory:                                                │
│   0x7f8e12345678: [Class pointer: 0x7f8c92c50000]            │
│   0x7f8e12345680: [Monitor: 0]                                 │
│   0x7f8e1234568C: [myField: 0] ← offset 12                    │
│                                                                 │
│ Status: INSTANCE CREATED AND READY                            │
└─────────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 5: EXECUTION (Bytecode Execution)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Bytecode: getfield #7                                          │
│                                                                 │
│ Execution:                                                      │
│   1. Lookup constant_pool[7]                                    │
│      → ALREADY RESOLVED (from linking phase)                   │
│      → Contains: field_offset=12 ✓                             │
│   2. obj = 0x7f8e12345678 (from stack)                         │
│   3. address = 0x7f8e12345678 + 12 = 0x7f8e1234568C           │
│   4. value = *(int*)0x7f8e1234568C                             │
│   5. Push to stack                                              │
│                                                                 │
│ Performance: ~5 nanoseconds ✓                                  │
│ Status: FIELD ACCESSED SUCCESSFULLY                           │
└─────────────────────────────────────────────────────────────────┘
