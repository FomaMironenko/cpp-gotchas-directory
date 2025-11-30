# C++ Gotchas Directory

// TODO: introduction



---

## std::vector

// TODO: internal impl, size and capacity, reallocation, complexity, asymptotics


### Iterators invalidation

// TODO: iterators and references are invalidated on erasure and inertion


### Transferring elements on reallocation

// TODO: POD types are `memcpy`-ed (depending on impl), move constructor must be noexcept, otherwise copy. std::vector come-ctor was not noexcept until c++17.


### Push vector elements to the same vector

// TODO: vec.push_back(vec.back())
ref: https://habr.com/ru/articles/816681/


### Boolean vector

// TODO: packs bits into bytes
ref: Scott Meyers


---


## Strings

// TODO: intro


### Small String Optimization

// TODO


### Constructing from `std::nullptr`

// TODO: UB


### C-style string as template parameter

// TODO

---


## `std::unordered_map` / `std::unordered_set`

// TODO: implementation, hash table, rehashing, load factor


### Iterators and pointer invalidation

// TODO: iterators are invalidated on insert, pointers are not!
ref: https://stackoverflow.com/questions/39868640/stdunordered-map-pointers-reference-invalidation


---


## Smart pointers

// TODO: implementation of shared and uniqie pts, control block, strong and weak counters
ref: Scott Meyers


### std::make_shared VS std::shared_ptr(obj_ptr)

// TODO


### Smart pointers require access to destructor

// TODO


### std::make_unique / std::make_shared requires access to constructor

// TODO

---


## Core language

// TODO: intro


### Range-based for over temporaries

// TODO


### Redundant copy with `auto`

// TODO: auto a = f(); for (auto a : vec) {}


### Deleted destructor

// TODO: only cast


### Deleted constructor

// TODO: prohibit objects on stack


### Static initialization order fiasco

// TODO:
ref: https://en.cppreference.com/w/cpp/language/siof.html


### Deleting a pointer to incomplete type

// TODO: UB, can be detected at compile time
ref: https://www.modernescpp.com/index.php/small-safety-improvements-in-the-c-26-core-language/#:~:text=Deleting%20a%20Pointer%20to%20an%20Incomplete%20Type%20should%20be%20ill%2Dformed


### RVO, NRVO

// TODO


### Initialization of unions

// TODO
ref: https://www.youtube.com/watch?v=kaI4R0Ng4E8



--- 


## Inheritance 

// TODO: intro


### Virtual destructors

// TODO


### Virtual function calls in constructors and destructors

// TODO


---


## Exceptions

// TODO: intro


### Exceptions in constructor

// TODO


### Exceptions in destructor

// TODO


### Exceptions inside new

// TODO: memory is deallocated automatically


### Never catch by value

// TODO


---


## Move semantics

// TODO


### std::move has no effect
