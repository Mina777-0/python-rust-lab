In Python, memory is managed in two ways:
- Reference Counting: When an object’s count hits zero, it’s deleted immediately. This always happens.
- Cyclic GC: Python periodically scans objects to find "circular references" (e.g., Object A points to B, and B points to A). This scan takes time and 
adds a memory overhead (32 bytes) to every object. Cyclic GC is only in case of list, dict, set, class instance (self uses __dict__ to store elements). 


There are many ways to accelarate how classes or schemas can deal with data:

i) __slots__: stores attributes in a fixed-size array (like a C struct) for memory and performance optimization.
    Significantly reduces memory consumption, especially when instantiating thousands or millions of objects, by avoiding the overhead of a dictionary for 
    each instance.
    ** Cannot dynamically add new attributes not defined in the __slots__ tuple at class definition time.

ii) msgspec: If you are dealing with high-performance schemas (like JSON API responses), msgspec is currently the gold standard. 
    It is significantly faster than Pydantic and even built-in json. It implements custom C-level structs for its types.
    Memory: It uses a specialized representation that is often smaller than standard Python objects.
    Speed: It performs validation and serialization in a single pass using highly optimized C code.

iii) typing.NamedTuple: If your class is essentially just a "bag of data" and doesn't need complex logic, use NamedTuple. It behaves like a tuple 
    (stored as a contiguous block in memory) but allows attribute access by name.

iv) Numpy is the nuclear option: It stores data in a single, contiguous block of memory, which contrasts with standard Python lists that store pointers to 
    scattered objects. This contiguous storage, combined with homogeneous data types and C-based algorithms, minimizes memory overhead and boosts performance 
    through better CPU cache utilization.



# i) __slots__

class PriceUpdateSchema:
    __slots__= ('symbol_id', 'bid_price', 'ask_price', 'volume')

    def __init__(self, symbol_id, bid_price, ask_price, volume):
        self.symbol_id= symbol_id
        self.bid_price= bid_price
        self.ask_price= ask_price
        self.volume= volume


# ii) msgspec instead of dataclasses and struct we can use it 

@dataclass(slots= True)
class PriceSchema:
    symbol_id:int 
    bid_price:float 
    ask_price:float
    volume:int 

    def pack(self):
        return struct.pack(
            '!IddQ',
            self.symbol_id,
            self.bid_price,
            self.ask_price,
            self.volume
        )

# OR
class PriceSchema(msgspec.Struct, gc=False):
    # gc=Fasle means no cyclic garbage collection. No use of __dict__
    # implements custom C-level structs for its types. 
    symbol_id:int 
    bid_price:float
    ask_price:float
    volume:int 


* A dataclass with slots=True eliminates the underlying __dict__ (saving memory), but the attributes are still Python objects managed by the Python runtime.
* msgspec.Struct goes a step further. It stores data in a compact, C-contiguous memory layout. When you access a field in msgspec, it’s often faster because it 
minimizes the pointer-chasing typical of standard Python objects.

* By setting gc=False in msgspec.Struct, you are telling Python: "This object will never point to itself or create a loop." 
The Benefit: You save those 32 bytes per object and the CPU doesn't waste time scanning your millions of packets.
The Result: The memory is still freed as soon as you stop using the object, but the process is much leaner.



# iv) Numpy 

# array= np.array()
# schema= np.dtype()
# The differnece between them is the former creates data and the later does NOT create data, it's basically a schema
# So to create data we use np.empty for the schema. After the data is created, it's treated as ndarray, victorized operation and memory efficiency 

price_dtype= np.dtype([
    ('symbol_id', "i4"),
    ('bid_price', 'f8'),
    ('ask_price', 'f8'),
    ('volume', 'i4')
])

packet_buffer= np.empty(1000000, dtype=price_dtype)



# Numpy vs msgspec in memory management

While msgspec is incredibly fast, it still produces Python Objects. NumPy produces Data.

msgspec.Struct (The "Object" Approach)Even with gc=False, every time you create a msgspec instance, Python has to:
Allocate a new chunk of memory for that specific object.
Initialize a Reference Count (to track when to delete it).
Store a Pointer in your list or array that points to that memory address.
This leads to Pointer Chasing. 
To read 1,000 prices, the CPU has to look at the list, find the memory address of the first object, jump there, read the price, go back to the list, 
find the next address, and jump again.

NumPy (The "Block" Approach)
NumPy allocates one massive, contiguous block of memory.
One Reference Count: The entire array of 1 million packets is one Python object.
No Jump: Because the data is contiguous (right next to each other), the CPU can "prefetch" the next price before you even ask for it. 
This is called Cache Locality.

Feature             msgspec.Struct                  NumPy Array
ObjectCount         1,000,000 objects               1 object
Ref Counting        1,000,000 counters              1 counter
Memory Layout       Scattered (Fragmentation)       Contiguous (Dense)CPU 
Efficiency          Low (Pointer Chasing)           High (SIMD/Vectorization)



# List vs Numpy and Victorization 

* A list in Python isn't a linked list it's a dynamic array of pointers 
arr = [1, 2, 3]
[ ptr ] -> PyLongObject(1)
[ ptr ] -> PyLongObject(2)
[ ptr ] -> PyLongObject(3)


All objects in Python (str, int, dict, function, class) for instance an integer isn't just stored as int with 2 bytes or 4 bytes, there is:
Object header
Reference count
Type pointer
Actual value

So instead of 4 bytes, it might use 28 bytes or more.
So when a for loop happens, the interpreter loops itself first before reaching the actual value.

In Numpy
arr = np.array([1,2,3], dtype=np.int32)
No Python objects
No pointers
Just raw contiguous memory
Same datatype
Fixed size

* Numpy doesn't loop over elements.  
It calls a C function
That C function loops in C
Often compiled with:
SIMD instructions
Vector registers (AVX, SSE)
CPU auto-vectorization

SIMD: Add 8 integers in ONE CPU instruction instead of Add 1 integer per instruction

* Victorization is:
Removing interpreter overhead
Moving loop into compiled C
Using CPU-level optimizations

Python loop: Python VM → bytecode → object → dynamic → slow
Numpy: Python → C function → raw memory → SIMD → fast

Python: You -> Python VM -> C runtime -> CPU
C: You -> CPU

Vectorization is a programming technique that replaces explicit for loops with optimized, internal array operations to process entire datasets at once rather 
than one element at a time. It dramatically increases performance by pushing iteration down to low-level, compiled code (typically C or Fortran) and, 
in many cases, leveraging parallel SIMD (Single Instruction, Multiple Data) CPU instructions. 

Vectorization in General
In computer science, vectorization is the process of transforming a scalar (one-at-a-time) operation into a vector operation 
(operating on many values simultaneously). 

Mechanism: Modern CPUs have "wide" registers capable of storing multiple data points. A vectorized operation, such as adding two arrays, sends a single 
instruction to the CPU to operate on all these elements at once, rather than looping 100 times to add 100 numbers.
Why it is faster:
Reduced Loop Overhead: Eliminates the overhead of interpreting loop control logic (e.g., in Python) at every iteration.
Parallelism (SIMD): Allows the processor to work on multiple data points simultaneously.
Cache Friendliness: Data is organized in contiguous memory blocks, reducing time spent waiting for data to load from RAM. 

Vectorization in Python
Because Python is an interpreted language, explicit for loops are relatively slow. Vectorization in Python is the standard approach for scientific computing and 
data science to achieve high performance. 

Primary Tool: NumPy. Libraries like NumPy, Pandas, and TensorFlow implement vectorization by using optimized C code under the hood.
Key Concept: Instead of iterating through a list, you define operations on a numpy.ndarray (N-dimensional array).



# List vs bytearray vs array / dict vs slots / msgspec vs Numpy 


List is a dynamic array of pinters that point to a boxed data (i.e. Object header, Reference count, Type pointer, Actual value). It can hold hetergenous tpyes
of data.
array.array finds a way to store basic data type in a packed, c-style format. Fixed type numbers (long, int, float) 
bytearray stores raw binary data (0-255) in a contiguous block. No pointers 

dict is a hashtable where the "buckets" store the hash, the key pointer, and the value pointer.
The index of the hash is contiguous but the pointers(keys) and pointers(values) are scattered pyobjects. 
slots don't use dict, use fixed-size contiguous array of pointers point to boxed pyobjects. Save the hash overhead.

Numpy: no pyobject, no pointers just contiguous block of C array or Rust vec<f64>



# referecnce counting and smart pointers 

Pointer in general is a variable that stores memory address of another variable which contains the actual data. It allwos only borrowing those data not own them
the same concept in Rust. Smart pointers on the other hand allow data ownership by using reference counting. This the concept of Python pointers.
Like array is an array of pointers; smart pointers that allow data they point at to be owned while keeping track through reference counting.

Automatic Tracking: Every Python object contains a built-in counter that tracks how many references point to it.
Increment/Decrement: The count increases when you assign an object to a new variable or add it to a container (like a list). It decreases when a variable is 
deleted, rebound to a different object, or goes out of scope
Immediate Deallocation: When the reference count reaches zero, the memory is immediately reclaimed by the interpreter. 

Safety: Python abstracts memory management to prevent common errors like memory leaks, buffer overflows, and "dangling pointers".
Variables as Labels: Instead of being "boxes" that hold data, Python variables are "labels" bound to objects.

When a variable let say x = 1 is created. This creates a smart pointer that points at a box in the heap. The interpreter automatically manages the life cycle
of the object it points to through reference counting. Technically, reference counting start with value 1. It is not 1 because there are internal caching integers 
added.
