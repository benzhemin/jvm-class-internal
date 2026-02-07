# Java Constant Pool: FIELDREF & METHODREF Explained

## Simple Java Code Example

```java
public class SimpleExample {
    public int myField = 10;
    
    public void printField() {
        System.out.println(myField);
    }
}
```

---

## How It Maps to Bytecode

When compiled, the Java source code generates bytecode that references fields and methods. These references use **indices** into the constant pool.

### Field Access Bytecode
In `printField()` method, when accessing `myField`:
```
4: getfield      #7                  // Field myField:I
```
- Bytecode instruction: `getfield #7` means "get the field at constant pool index #7"
- Index #7 is a `CONSTANT_Fieldref` entry

### Method Call Bytecode  
In `printField()` method, when calling `System.out.println()`:
```
7: invokevirtual #19                 // Method java/io/PrintStream.println:(I)V
```
- Bytecode instruction: `invokevirtual #19` means "call method at constant pool index #19"
- Index #19 is a `CONSTANT_Methodref` entry

---

## CONSTANT_Fieldref Example - Tracking myField

### Step 1: The CONSTANT_Fieldref Entry

```
#7 = Fieldref           #8.#9          // SimpleExample.myField:I
```

**Structure breakdown:**
- **Tag**: 9 (identifies this as CONSTANT_Fieldref)
- **class_index**: #8 (points to the class where field is defined)
- **name_and_type_index**: #9 (points to field name and descriptor)

### Step 2: Follow class_index (#8)

```
#8 = Class              #10            // SimpleExample
```

**Structure:**
- **Tag**: 7 (CONSTANT_Class)
- **name_index**: #10 (points to the class name)

### Step 3: Follow name_index (#10)

```
#10 = Utf8              SimpleExample
```

**Result**: Class name is "SimpleExample"

### Step 4: Follow name_and_type_index (#9)

```
#9 = NameAndType        #11:#12        // myField:I
```

**Structure:**
- **Tag**: 12 (CONSTANT_NameAndType)
- **name_index**: #11 (field name)
- **descriptor_index**: #12 (field type descriptor)

### Step 5: Follow name_index (#11) and descriptor_index (#12)

```
#11 = Utf8              myField
#12 = Utf8              I
```

**Results:**
- Field name: "myField"
- Descriptor: "I" (means type `int`)

### Complete Information Extracted from #7

```
┌─ CONSTANT_Fieldref (#7)
│
├─ Class: SimpleExample (from #8 → #10)
├─ Field Name: myField (from #9 → #11)
└─ Field Type: I (int) (from #9 → #12)

Final Result: "SimpleExample.myField:I"
```

### Bytecode Usage

When JVM executes `getfield #7`:
```java
getfield #7  // Retrieves field: SimpleExample.myField of type int
```

The JVM uses this information to:
1. Find the class `SimpleExample` 
2. Look up the field `myField` in that class
3. Know it's an `int` type (for validation)
4. Get the field's value from the object instance

---

## CONSTANT_Methodref Example - Tracking println()

### Step 1: The CONSTANT_Methodref Entry

```
#19 = Methodref          #20.#21        // java/io/PrintStream.println:(I)V
```

**Structure:**
- **Tag**: 10 (identifies this as CONSTANT_Methodref)
- **class_index**: #20 (points to the class where method is defined)
- **name_and_type_index**: #21 (points to method name and descriptor)

### Step 2: Follow class_index (#20)

```
#20 = Class              #22            // java/io/PrintStream
```

**Structure:**
- **Tag**: 7 (CONSTANT_Class)
- **name_index**: #22 (points to the class name)

### Step 3: Follow name_index (#22)

```
#22 = Utf8               java/io/PrintStream
```

**Result**: Class name is "java/io/PrintStream"

### Step 4: Follow name_and_type_index (#21)

```
#21 = NameAndType        #23:#24        // println:(I)V
```

**Structure:**
- **Tag**: 12 (CONSTANT_NameAndType)
- **name_index**: #23 (method name)
- **descriptor_index**: #24 (method signature)

### Step 5: Follow name_index (#23) and descriptor_index (#24)

```
#23 = Utf8               println
#24 = Utf8               (I)V
```

**Results:**
- Method name: "println"
- Descriptor: "(I)V" (takes 1 int parameter, returns void)

### Complete Information Extracted from #19

```
┌─ CONSTANT_Methodref (#19)
│
├─ Class: java/io/PrintStream (from #20 → #22)
├─ Method Name: println (from #21 → #23)
└─ Method Signature: (I)V (from #21 → #24)
   └─ Parameters: int
   └─ Return Type: void

Final Result: "java/io/PrintStream.println:(I)V"
```

### Bytecode Usage

When JVM executes `invokevirtual #19`:
```java
invokevirtual #19  // Calls method: java/io/PrintStream.println(int)
```

The JVM uses this information to:
1. Find the class `java/io/PrintStream`
2. Look up the method `println` that takes an `int`
3. Perform virtual method dispatch (find actual implementation in runtime type)
4. Set up parameters and call the method

---

## Visual Mapping Summary

### FIELDREF Resolution Chain

```
Bytecode: getfield #7
    ↓
Constant Pool [#7] = Fieldref
    ├── class_index = #8
    │       ↓
    │   [#8] = Class → #10
    │       ↓
    │   [#10] = Utf8 "SimpleExample"
    │
    └── name_and_type_index = #9
            ↓
        [#9] = NameAndType
            ├── name_index = #11
            │       ↓
            │   [#11] = Utf8 "myField"
            │
            └── descriptor_index = #12
                    ↓
                [#12] = Utf8 "I"

RESOLVED TO: 
  Class: SimpleExample
  Field: myField
  Type: int
```

### METHODREF Resolution Chain

```
Bytecode: invokevirtual #19
    ↓
Constant Pool [#19] = Methodref
    ├── class_index = #20
    │       ↓
    │   [#20] = Class → #22
    │       ↓
    │   [#22] = Utf8 "java/io/PrintStream"
    │
    └── name_and_type_index = #21
            ↓
        [#21] = NameAndType
            ├── name_index = #23
            │       ↓
            │   [#23] = Utf8 "println"
            │
            └── descriptor_index = #24
                    ↓
                [#24] = Utf8 "(I)V"

RESOLVED TO:
  Class: java/io/PrintStream
  Method: println
  Signature: (I)V
    - Parameter 1: int
    - Return: void
```

---

## Key Points

1. **Fieldref/Methodref are INDIRECT references** - they point to other constant pool entries
2. **No direct field/method info** - the actual names and types are stored as UTF8 strings
3. **Linked resolution** - the JVM must follow the chain of indices to resolve complete info
4. **Type safety** - descriptors enable JVM to validate types at runtime
5. **Efficiency** - indices are 2 bytes, more compact than storing full strings everywhere

---

## Instance Class Mapping in Runtime

When the JVM loads `SimpleExample` class:

```java
// Class metadata is stored in JVM memory
class SimpleExample {
    public int myField;
    
    public SimpleExample() { ... }
    public void printField() { ... }
}
```

The constant pool entries map to:

| Entry | Maps To | Runtime Lookup |
|-------|---------|-----------------|
| #7 (Fieldref) | SimpleExample.myField:I | JVM finds field in SimpleExample class metadata |
| #19 (Methodref) | java/io/PrintStream.println:(I)V | JVM finds method in loaded PrintStream class |

When bytecode executes `getfield #7`:
- JVM looks up entry #7 in constant pool
- Resolves the chain: #7 → #8 → #10 → "SimpleExample", and #9 → #11 → "myField", #12 → "I"
- Searches for field "myField" of type "int" in SimpleExample class object
- Gets the field value from the object instance

---

## Binary Representation

If we looked at the raw bytes of entry #7:

```
Offset  Bytes
------- ----
0x00    09           // Tag = 9 (CONSTANT_Fieldref)
0x01    00 08        // class_index = 8 (big-endian u2)
0x03    00 09        // name_and_type_index = 9 (big-endian u2)
```

Entry #9:
```
Offset  Bytes
------- ----
0x00    0C           // Tag = 12 (CONSTANT_NameAndType)
0x01    00 0B        // name_index = 11 (big-endian u2)
0x03    00 0C        // descriptor_index = 12 (big-endian u2)
```

The indices (0x0008, 0x0009, etc.) are 2-byte big-endian integers pointing into the constant pool array.
