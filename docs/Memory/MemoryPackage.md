---
layout: doc_page
---

## Memory Package

Note: this applies to the memory package after version 0.10.0

### Introduction
The DataSketches memory package has its own repository and is released with its own jars in Maven Central. 
To avoid confusion in the documentation the capitalized <i>Memory</i> refers to the code in the 
Java <i>com.yahoo.memory</i> package, and the uncapitalized <i>memory</i> refers to computer memory in general.

The <i>Memory</i> package allows primitive read-write capabilities of data structures in native computer memory, 
which is also referred to as "off-java-heap" or just "off-heap". 
For compatibility and ease-of-use the <i>Memory</i> API can also be used to manage data structures that are 
contained in Java on-heap primitive arrays, memory mapped files, or ByteBuffers.

The hardware systems used in big data environments can be quite large approaching a terabyte 
of RAM and 24 or more CPUs, each of which can manage two threads.
Most of that memory is usually dedicated to selected partitions of data, 
which can even be orders of magnitude larger. 
How the system designers select the partitions of the data to be in RAM over time is quite complex 
and varies considerably based on the specific objectives of the systems platform. 

It is in these very large data environments that managing how the data gets copied into RAM and 
when it is considered obsolete and can be written 
over by newer or different partitions of data is an important aspect of the systems design. 
Having the JVM manage these large chunks of memory is often problematic and often results in large garbage collection 
pauses and poor real-time performance. 
As a result, it is often the case that the system designers need to manage these large chunks of 
memory directly.  

The JVM has a very sophisticated heap management process and works very well for many 
general purpose programming tasks. 
However, for very large systems that have critical latency requirements, 
utilizing off-heap memory becomes a requirement.

Java does not permit normal java processes direct access to off-heap memory. Nonetheless, 
in order to improve performance, many internal Java classes leverage a low-level, restricted
class (unfortunately) called "Unsafe", which does exactly that. The methods of Unsafe
are native methods that are initially compiled into C++ code.  The JIT compiler
replaces this C++ code with assembly language instructions called "intrinsics", which
are often just a single CPU instruction.

The <i>Memory</i> package is essentially an extension of Unsafe and wraps most of the 
primitive get and put methods and a few specialized methods into a convenient API 
organized around an allocated block of native memory.

Using the <i>Memory</i> package cannot be taken lightly, as the systems developer must now be 
aware of the importance of memory allocation and deallocation and make sure these resources 
are managed properly. 

### Architecture
The Memory package consists of two major groups of classes.
  * Classes that provide access to a <i>resource</i>, which is a linear collection of consecutive bytes. 
  * Classes that define an API for reading and writing to a resource using primitives and primitive arrays.

#### Resourses
The Memory package defines 4 <i>Resources</i> that at their most basic level can be viewed as a linear collection of consecutive bytes.
  * Primitive on-heap arrays: byte[], char[], etc.
  * Java _ByteBuffer_s
  * Off-heap memory. Also called "native" or "direct" memory.
  * Memory-mapped files


#### APIs
The Memory package defines 4 APIs for accessing the above resources.
  * Memory - Read-only access using byte offsets from the start of the resource.
  * WritableMemory - Read/write access using byte offsets from the start of the resource.
  * Buffer - Read-only access using user setable byte position values: <i>start</i>, <i>position</i>, and <i>end</i>.
  * WritableBuffer - Read-write access using user setable byte position values: <i>start</i>, <i>position</i>, and <i>end</i>.

#### Mapping a Resource to an API
There are two different ways to map a resource to an API. The first uses static methods:

```java
    //use static methods to map an array of 1024 bytes to the WritableMemory API
    WritableMemory wmem = WritableMemory.allocate(1024);
    //Or, map a ByteBuffer to the WritableMemory API
    WritableMemory wmem2 = WritableMemory.allocate(byteBuffer);
```

For AutoClosable resources, special classes called "Handles" are used to manage the AutoClosable properties. See examples below.


#### Examples for Accessing Primitive On-heap Array Resources
Mapping a primitive array resource to the Memory API:

```java
    byte[] array = new byte[] {1, 0, 0, 0, 2, 0, 0, 0};
    Memory mem = Memory.wrap(array);
    assert mem.getInt(0) == 1;
    assert mem.getInt(4) == 2;
```

This illustrates that the underlying structure of the resource is bytes but we can read it as
ints, longs, char, or whatever. This is similar to a C UNION, which allows multiple data types
to access the underlying bytes. This isn't allowed in Java! So be careful! For example:

```java
    byte[] arr = new byte[16]
    WritableMemory wmem = WritableMemory.wrap(arr);
    wmem.putByte(1, (byte) 1);
    int v = wmem.getInt(0);
    assert ( v == 256 );
    
    arr[9] = 3; //you can also access the backing array directly
    long v2 = mem.getLong(8);
    assert ( v2 == 768L);
```

You have to keep careful track of your own structure and the appropriate byte offsets.

Reading and writing multibyte primitives requires an assumption about byte ordering or endianness. 
The default endianness is _ByteOrder.nativeOrder()__, which for most CPUs is _ByteOrder.LITTLE_ENDIAN_.
Additional APIs are being developed to enable reading and writing in non-native endianness.

All of the APIs provide a useful toHexString(...) method to assist you in viewing the data in your resources.

#### Examples for Accessing ByteBuffers
Mapping a ByteBuffer resource to the WritableMemory API.  
Here we write the WritableBuffer and read from both the ByteBuffer and the WritableBuffer.

```java
    @Test
    public void simpleBBTest() {
      int n = 1024; //longs
      byte[] arr = new byte[n * 8];
      ByteBuffer bb = ByteBuffer.wrap(arr);
      bb.order(ByteOrder.nativeOrder());
      
      WritableBuffer wbuf = WritableBuffer.wrap(bb);
      for (int i = 0; i < n; i++) { //write to wbuf
        wbuf.putLong(i);
      }
      wbuf.resetPosition();
      for (int i = 0; i < n; i++) { //read from wbuf
        long v = wbuf.getLong();
        assertEquals(v, i);
      }
      for (int i = 0; i < n; i++) { //read from BB
        long v = bb.getLong();
        assertEquals(v, i);
      }
    }
```

#### Examples for Accessing Off-Heap Resources  
Direct allocation of off-heap resources requires that the resource be closed when finished.
This is accomplished using a _WritableDirectHandle_ that implements the Java _AutoClosable_ interface. 
Note that this example leverages the try-with-resources statement to properly close the resource.

```java
    @Test
    public void simpleAllocateDirect() {
      int longs = 32;
      try (WritableDirectHandle wh = WritableMemory.allocateDirect(longs << 3)) {
        WritableMemory wMem1 = wh.get();
        for (int i = 0; i<longs; i++) {
          wMem1.putLong(i << 3, i);
          assertEquals(wMem1.getLong(i << 3), i);
        }
      }
    }
```

Note that these direct allocations can be larger than 2GB.

#### Examples for Memory Mapped File Resources
Memory-mapped files are resources that also must be closed when finished.
This is accomplished using a _MapHandle_ that implements the Java _AutoClosable_ interface.
In the src/test/resources directory of the memory-X.Y.Z-test-sources.jar there is a file called GettysburgAddress.txt.
Note that this example leverages the try-with-resources statement to properly close the resource.
To print out Lincoln's Gettysburg Address:

```java
    @Test
    public void simpleMap() throws Exception {
      File file = new File(getClass().getClassLoader().getResource("GettysburgAddress.txt").getFile());
      try (MapHandle h = Memory.map(file)) {
        Memory mem = h.get();
        byte[] bytes = new byte[(int)mem.getCapacity()];
        mem.getByteArray(0, bytes, 0, bytes.length);
        String text = new String(bytes);
        System.out.println(text);
      }
    }
```

The following test does the following:

1. Creates a off-heap WritableMemory and preloads it with 4GB of consecutive longs as a candidate source.
2. Creates an empty file, and maps it to a memory-mapped space also of 4GB as the destination.
3. Copies the source to the destination in a single operation. No extra copies required.

```java
    @Test
    public void copyOffHeapToMemoryMappedFile() throws Exception {
      long bytes = 1L << 32; //4GB
      long longs = bytes >>> 3;
  
      File file = new File("TestFile.bin");
      if (file.exists()) { file.delete(); }
      assert file.createNewFile();
      assert file.setWritable(true, false);
      assert file.isFile();
      file.deleteOnExit();  //comment out if you want to examine the file.
  
      try (
          WritableMapHandle dstHandle
            = WritableMemory.writableMap(file, 0, bytes, ByteOrder.nativeOrder());
          WritableDirectHandle srcHandle = WritableMemory.allocateDirect(bytes)) {
  
        WritableMemory dstMem = dstHandle.get();
        WritableMemory srcMem = srcHandle.get();
  
        for (long i = 0; i < (longs); i++) {
          srcMem.putLong(i << 3, i); //load source with consecutive longs
        }
  
        srcMem.copyTo(0, dstMem, 0, srcMem.getCapacity()); //off-heap to off-heap copy
  
        dstHandle.force(); //push any remaining to the file
  
        //check end value
        assertEquals(dstMem.getLong((longs - 1L) << 3), longs - 1L);
      }
    }
```

#### Regions and WritableRegions
Similar to the _ByteBuffer slice()_, one can create a region or writable region, 
which is a view into the same underlying resource. 

```java
    @Test
    public void checkRORegions() {
      int n = 16;
      int n2 = n / 2;
      long[] arr = new long[n];
      for (int i = 0; i < n; i++) { arr[i] = i; }
      Memory mem = Memory.wrap(arr);
      Memory reg = mem.region(n2 * 8, n2 * 8);
      for (int i = 0; i < n2; i++) {
        assertEquals(reg.getLong(i * 8), i + n2);
      }
    }
```
