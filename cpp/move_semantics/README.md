# C++ Move Semantics

## Introduction

One of the major features introduced by C++11 is the concept of **move
semantics.** Move semantics make it possible to efficiently (i.e. usually in
constant time) move objects between functions, objects, and scopes without
needing to use pointers or dynamic memory. Copy semantics (as supported by
copy-constructors and the copy-assignment operator) produce full copies of the
original object, which is often unnecessary, if not outright impossible.

This tutorial provides an overview of move semantics: their purpose, as well as
their general implementation.

## Review: Copy Semantics

EECS 280 teaches the ["Big Three"](https://en.wikipedia.org/wiki/Rule_of_three_(C%2B%2B_programming))
operators for writing classes: the destructor, copy-constructor, and
copy-assignment operator. When writing classes that manage non-trivial
resources, such as dynamic memory, implementing these operators is necessary to
avoid subtle bugs and memory leaks.

With the inclusion of the _move-construction_ and _move-assignment_ operators,
the Big Three becomes the "Big Five."

We'll start off with the Big Three. Take, for example, a vector-like class with
the following interface:

```cpp
class IntVector {
 public:
  int& operator[] (std::size_t idx);

  void push_back(int to_append);
  void erase(std::size_t idx);
  std::size_t size() const;
 private:
  std::size_t size_{0};
  int* data_{nullptr};
};  // class IntVector
```

If a user were to pass an `IntVector` into a function by value, they would
expect that changes made to the `IntVector` _copy_ would not affect the
_original_ IntVector.

```cpp
void delete_contents(IntVector copy) {
  while (copy.size()) {
    copy.erase(copy.size() - 1);
  }
}

int main() {
  IntVector vec;
  vec.push_back(0);
  vec.push_back(1);
  vec.push_back(2);

  cout << vec.size() << endl;  // prints 3
  delete_contents(vec);
  cout << vec.size() << endl;  // should still print 3
}
```

A simple implementation of the "Big Three" for this class might look like:

```cpp
  ~IntVector() {
    delete[] data_;
  }

  IntVector(const IntVector& rhs) : size_{rhs.size_} {
    if (!size_) return;  // no work needed if the RHS was empty
    data_ = new int[size_];  // allocate a new array
    for (std::size_t i = 0; i < size_; ++i) {
      data_[i] = rhs.data_[i];
    }
  }

  IntVector& operator=(const IntVector& rhs) {
    IntVector rhs_copy(rhs);
    // dump our contents into the rhs_copy and steal its contents (copied from
    // rhs); when rhs_copy leaves scope, its destructor runs and cleans up our
    // old resources
    this->swap(rhs_copy);
    return *this;
  }

  void swap(IntVector& with) {
    std::swap(size_, with.size_);
    std::swap(data_, with.data_);
  }
```

Note the addition of a `::swap()` member function, to facilitate use of the
[copy-swap idiom.](https://stackoverflow.com/questions/3279543/what-is-the-copy-and-swap-idiom)
Conceptually, this lets us "reuse" the copy-constructor and destructor inside of
the copy-assignment operator, rather than having to write out the "if the RHS
is empty, do nothing; otherwise, allocate an array and copy each element..."
logic all over again.

<details><summary>Also note that the explicit `this->` is unnecessary.</summary>

We could just write `swap(rhs_copy)` instead of `this->swap(rhs_copy)`; we only
write the `this->` for clarity. In the real world, it may be necessary to
prevent the compiler from mistakenly invoking `std::swap()` (if there had been
a `using namespace std;` earlier in the file, or similar), but in a well-written
header file, that shouldn't be a major concern.
</details>

Also notice that the copy-construction and copy-assignment operators run in
linear time, i.e. they are both O(n) with respect to the size of the RHS: both
operations require iterating over the RHS's `data_` array and copying each
integer one-by-one.

## Benefits of Move Semantics: Efficiency

Imagine that a user were to use `IntVector` like this:

```cpp
void some_function() {
  IntVector vec;

  // ...lots of other work...

  if (user_input_passed_validation) {
    vec = IntVector{1, 2, 3, 4, 5, 6};
  }
}
```

<details><summary>Note</summary>

Use of the braced initializer list (`IntVector{1, 2,...`) requires the
definition of an [initializer-list constructor](http://www.cplusplus.com/reference/initializer_list/initializer_list/).
`std::initializer_list` objects provide `begin()` and `end()` member functions
that return pointers to a temporary array containing (copies of) the values from
the braced initializer list provided by the user.

An initializer list constructor for `IntVector` might look like:

```cpp
  IntVector(std::initializer_list<int> vals)
      : size_(vals.end() - vals.begin()) {
    data_ = new int[size_];
    int* our_datum = data_;
    for (const int* datum = vals.begin(); datum != vals.end(); ) {
      *our_datum++ = *datum++;
    }
  }
```
</details>

With our current `IntVector`, this would result in:

1. The construction of a new, temporary `IntVector`, which dynamically
   allocates a 6-element array for the given values,
2. Deallocation of `vec`'s original array,
3. Allocation of a new 6-element array for `vec`,
4. Copy-assignment of all 6 elements of the temporary `IntVector` into `vec`'s
   new data array,
5. Deallocation and destruction of the temporary `IntVector`.

Construction of the initial, temporary `IntVector` runs in
linear time (with respect to the size of the braced initializer), but copying
from the temporary into `vec` results in _another_ linear time copy operation.

This is needlessly inefficient.  We perform a deep copy in `IntVector`'s copy
operators to prevent changes to a copy from modifying the original vector, but
in this case, there's no possible way to directly reference or modify the
temporary `IntVector`: it can only be used as the "right-hand side" of an
operation, i.e. an **rvalue.**

We don't care about the contents or "future life" of an rvalue (though its
destruction shouldn't cause resource leaks, segmentation faults, etc.) With that
in mind, rather than copying its contents integer-by-integer, we could simply
"steal" its array in its entirety after it finishes construction.

To achieve this, we can write a **move-assignment operator** as follows:

```cpp
  // the double-ampersand denotes an "rvalue reference"
  // (an ordinary, single ampersand denotes an "lvalue reference")
  IntVector& operator=(IntVector&& rhs) {
    // steal the contents of rhs and dump our contents into it
    // when rhs eventually leaves scope, it'll handle cleanup of our old data_
    // array.
    this->swap(rhs);
    return *this;
  }
```

Now, execution of `some_function()` from above will result in:

1. The construction of a new, temporary `IntVector`, which dynamically
   allocates a 6-element array for the given values,
2. The temporary `IntVector` and `vec` exchanging contents: the temporary winds
   up with `vec`'s original `data_`, while `vec` winds up with the new `data_`
   from the braced initializer list.
3. The temporary `IntVector` goes out of scope, and its destructor deallocates
   `vec`'s original `data_`.

Note that, ignoring the initial construction of the temporary `IntVector`, this
whole operation takes place in constant time.

### Why the reliance on the `::swap()` function?

#### Because the RHS's destructor still runs!

As you likely learned during EECS 280 and EECS 281, writing copy operators can
be tricky and bug-prone. The same is true of writing move operators, especially
for a novice.

One might wonder: couldn't we just implement move-assignment operator like this:

```cpp
  // DON'T DO THIS
  IntVector& operator=(IntVector&& rhs) {
    size_ = rhs.size_;
    data_ = rhs.data_;
    return *this;
  }
```

**This won't work.** To understand why, remember that `*this` and `rhs` are two
distinct objects, with two distinct lifetimes.

For sake of example, assume that `*this` and `rhs` enter the move-assignment
operator with the following values

```javascript
// this
{
  "size_": 3,
  "data_": 0xA6 -> [4, 5, 6],
}

// rhs
{
  "size_": 5,
  "data_": 0x4D -> [1, 2, 3, 4, 5],
}
```

By the time the move-assignment operator finishes execution, `*this` and `rhs`
will look like:

```javascript
// this
{
  "size_": 5,
  "data_": 0x4D -> [1, 2, 3, 4, 5],
}

// rhs
{
  "size_": 5,
  "data_": 0x4D -> [1, 2, 3, 4, 5],
}
```

**`rhs` is an rvalue,** so at some point, it will go out of scope and its
destructor will run.

<details><summary>For example,</summary>

In the example `some_function()` from above (in which `vec` is `*this` and the
temporary `IntVector` is `rhs`):

```cpp
  if (user_input_passed_validation) {
    vec = IntVector{1, 2, 3, 4, 5, 6};
  }  // <-- by this point, the temporary IntVector was destroyed!
```
</details>

Once this happens, `*this` and `rhs` will look like:

```javascript
// this
{
  "size_": 5,
  "data_": 0x4D -> [MEMORY JUNK],
}

// rhs
{
  "size_": XXX,
  "data_": 0xXXXXXX -> [MEMORY JUNK],
}
```

Further operations on `*this` will likely result in a segmentation fault and/or
other undefined behavior.


#### Because we need to clean up our original array!

Knowing that the `rhs`'s destructor will still run, we might try to overwrite its
contents, so that it won't delete our newly stolen array "from beyond the
grave."

```cpp
  // DON'T DO THIS EITHER
  IntVector& operator=(IntVector&& rhs) {
    size_ = rhs.size_;
    data_ = rhs.data_;
    rhs.size_ = 0;
    rhs.data_ = nullptr;
    return *this;
  }
```

Now, when `rhs`'s destructor runs, it'll call `delete[]` on a `nullptr`, which
is a no-op. It's now possible for us to safely use `*this`'s newly stolen array
without worrying about segmentation faults.

But, we've still forgotten something important. After `rhs` has been destroyed,
`*this` and `rhs` will look like:

```javascript
// this
{
  "size_": 5,
  "data_": 0x4D -> [1, 2, 3, 4, 5],
}

// rhs
{
  "size_": XXX,
  "data_": 0xXXXXXX -> [MEMORY JUNK],
}

// ...

  0xA6 -> [4, 5, 6]  // <-- oops, forgot about that
```

We never called `delete[]` on our original array!

To fix this, we could manually deallocate our array, like:

```cpp
  // this works, but it's clunky and inelegant
  IntVector& operator=(IntVector&& rhs) {
    delete[] data_;
    size_ = rhs.size_;
    data_ = rhs.data_;
    rhs.size_ = 0;
    rhs.data_ = nullptr;
    return *this;
  }
```

But it's easier -- that is, it's better practice and needs less code -- just to
dump our old array into `rhs` as we steal its "new" one.

```cpp
  void swap(IntVector& with) {
    std::swap(size_, with.size_);
    std::swap(data_, with.data_);
  }

  IntVector& operator=(IntVector&& rhs) {
    // steal the contents of rhs and dump our contents into it
    // when rhs eventually leaves scope, it'll handle cleanup of our old data_
    // array.
    this->swap(rhs);
    return *this;
```

## Writing a Move Constructor

Imagine that we were to try constructing an `IntVector` like this:

```cpp
  IntVector vec(IntVector{1, 2, 3, 4, 5, 6});
```

We run into the same efficiency problems that we mentioned earlier; the major
difference is that now, we're (unnecessarily) invoking the copy-constructor
rather than the copy-assignment operator.

To achieve this, we can write a **move-constructor.**

```cpp
  IntVector(IntVector&& rhs) {
    this->swap(rhs);
  }
```

Again, note the use of a double-ampersand (`&&`) to denote an rvalue reference.
This identifies the move-constructor to the compiler.

**The earlier concerns about `rhs`'s destructor still apply.** The reason the
move constructor above works is because our class definition defines its member
variables like this:

```cpp
 private:
  std::size_t size_{0};  // defaults to 0
  int* data_{nullptr};  // defaults to nullptr
};  // class IntVector
```

So, _before_ we execute `this->swap()`, it's already guaranteed that `*this` looks like:

```javascript
// this
{
  "size_": 0,
  "data_": 0x0 -> (null)
}
```

So `rhs` will wind up calling `delete[]` on a `nullptr`.

However, if we defined our member variables like this:

```cpp
 private:
  std::size_t size_;  // undefined value
  int* data_;  // undefined value
};  // class IntVector
```

Then these member variables are [default-initialized](https://en.cppreference.com/w/cpp/language/default_initialization).
Primitive values are effectively default-initialized with memory junk ("whatever
was there" when memory was allocated for the new class instance). In this case,
before our `this->swap()` call, `*this` might look like:

```javascript
// this
{
  "size_": 542524156,
  "data_": 0x16A3BE -> [MEMORY JUNK]
}
```

When these values get passed to `rhs`'s destructor, `data_` is _not_
a `nullptr`; calling `delete[]` on `data_` _won't_ be a no-op, but will try to
deallocate memory that had (probably) never been allocated, causing
a segmentation fault.

To avoid this, we need to put `*this` into a valid (but empty) state _before_ we
`swap()` with `rhs`. We could initialize our member variables in our class
definition (like we did earlier), but a more generalizable approach would be to
write the class's default constructor, and use it to "start" the
move-constructor.

```cpp
  IntVector() : size_{0}, data_{nullptr} {}

  // delegate to the default constructor before move-construction
  IntVector(IntVector&& rhs) : IntVector{} {
    this->swap(rhs);
  }

 private:
  std::size_t size_;
  int* data_;
};  // class IntVector
```

The `IntVector{}` in the move constructor's initializer list performs
[constructor delegation](https://stackoverflow.com/questions/308276/can-i-call-a-constructor-from-another-constructor-do-constructor-chaining-in-c)
to the default constructor, which (in this case) runs the default constructor
before execution of the move constructor. The default constructor sets `size_`
and `data_` to zero, guaranteeing that `swap()` will `rhs` in a safe state for
deletion.


## Moving from Non-Temporary ("Named") Objects

While an rvalue is (conventionally) an "anonymous" object on the right-hand side
of an assignment, it's possible to forcibly move-assign from an "ordinary"
named variable. This involves use of the `std::move()` function from the
`<utility>` header.

```cpp
#include <utility>

// ...

void some_function() {
  IntVector vec;

  // ...lots of other work...

  if (user_input_passed_validation) {
    IntVector temporary{1, 2, 3, 4, 5, 6};
    vec = std::move(temporary);

    std::cout << vec.size() << std::endl;  // prints 6
    std::cout << temporary.size() << std::endl;  // prints 0
  }
}
```

This can be used to place named variables inside of a container without
inefficient and unnecessary copy operations.

```cpp
void some_function() {

  // ...

  std::map<std::string, IntVector> names_to_vectors;
  names_to_vectors["billy magic"] = std::move(vec);

  std::cout << names_to_vectors["billy magic"] << std::endl;  // prints 6
  std::cout << vec.size() << std::endl;  // prints 0

  // ...
}
```

If we were to simply `names_to_vectors["billy magic"] = vec;`, then we would
perform a linear-time copy-assignment from `vec` into the `std::map`, which is
likely not our intention. Move-assignment is a more efficient alternative --
just remember that it leaves the moved-from object in an "empty", unspecified
state.

Practically all STL containers -- [including](https://en.cppreference.com/w/cpp/container/vector/vector)
[`std::vector`](https://en.cppreference.com/w/cpp/container/vector/operator%3D),
the (vastly superior) real-world counterpart of our toy `IntVector` -- support
move operations. If one needs to perform non-trivial initialization work for an
STL container prior to placing it in another container, then one can simply
manipulate it like a regular variable before `std::move()`ing it into its final
location.

## Benefits of Move Semantics: Manipulation of Non-Copyable Objects

It often makes sense to "delete" the copy- operators. When this is done, any
operation that would copy an instance of the class (e.g. passing an instance of
it by value) will cause a compilation error.

Typically, this is done when copying an object is intrinsically unsafe, or when
it just wouldn't make any sense conceptually. When the copy operators are
deleted, but the move operators are not, then `std::move()` becomes the only way
to "move" the object from place to place without relying on pointers or
references.

[`std::thread`](https://en.cppreference.com/w/cpp/thread/thread) is one example.
An `std::thread` serves as a handle to a running thread of execution. The
constructor of `std::thread` takes a function and (optionally) arguments for
that function, spawning a new thread within the operating system that calls that
function with those arguments.

What would it mean to copy an `std::thread`? Would it spawn another "fresh"
thread, one that calls the same function with the same arguments? Or would it
"clone" the currently running thread, capturing its entire state in a parallel
thread of execution? Maybe it would be a shallow copy, and the new `std::thread`
would refer to the same thread of execution as the original.

Of these possibilities, the latter is plausible, but the C++ standard library
guarantees that ["No two std::thread objects may represent the same thread of
execution."](https://en.cppreference.com/w/cpp/thread/thread/thread)
Consequently, `std::thread`'s copy-constructor and copy-assignment operators are
both `delete`d.

If we wanted to construct an `std::thread` as a named variable _and then_ place
it in a container, it would not be possible to naively write code like the
following:

```cpp
void do_work(std::string arg1, std::string arg2);

std::vector<std::thread> threads;
std::thread a_thread(do_work, "abc", "def");

// threads.push_back(a_thread);  // ERROR
```

We could, however, write:
```cpp
void do_work(std::string arg1, std::string arg2);

std::vector<std::thread> threads;
std::thread a_thread(do_work, "abc", "def");

threads.push_back(std::move(a_thread));
```

Note that, once again, this would leave `a_thread` "empty"; calling
`a_thread.join()` would not accomplish anything meaningful after the
`std::move()`.


## Working with Uncopyable, Immovable Objects

Some objects cannot be copied _and_ cannot be moved. This may occur for a number
of reasons, one of which may be that the object won't work correctly if one
tries to move it to another memory address.

An example of an uncopyable, immovable object is [`std::mutex`](https://en.cppreference.com/w/cpp/thread/mutex).
While it's possible to write a safely movable mutex (such as the `mutex` class
that you'll implement in Project 2), [in some operating systems](https://stackoverflow.com/questions/7557179/move-constructor-for-stdmutex),
`std::mutex` contains a low-level synchronization primitive as an "ordinary",
non-pointer member variable; in POSIX systems, this is a `pthread_mutex_t`,
which often [cannot be safely moved to another memory address](https://stackoverflow.com/questions/14614523/can-a-pthread-mutex-t-be-moved-in-memory)
after initialization.

In order to "move" such an object from place to place, it becomes necessary to
allocate the object at a single address (e.g. dynamically) and instead
manipulate a _pointer_ to that object, which is trivially copyable and movable.
Semantically, it would make sense to treat this pointer as
a movable-but-not-copyable object, which can be achieved by dynamically
allocating it through an [`std::unique_ptr`.](../smart_ptr.md)

```cpp
std::vector<std::mutex> mutexes;
std::mutex m;
// mutexes.push_back(std::move(m));  // ERROR


std::vector<std::unique_ptr<std::mutex>> mutex_ptrs;
std::unique_ptr<std::mutex> m_ptr = std::make_unique<std::mutex>();
mutex_ptrs.push_back(std::move(m_ptr));  // works
```
