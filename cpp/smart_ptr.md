# C++11 Smart Pointers

## Intro
One of the most valuable features introduced by C++11 (that you've been
forbidden from using in EECS 280/281) is C++11 smart pointers, which are
RAII-based wrappers around "raw", dynamically allocated pointers.

280 uses raw pointers, allocated through `new` and `delete`, because they're
teaching you dynamic memory from scratch; using a smart pointer that hides
the details of memory management would miss the point. But just as you wouldn't
use your unbalanced binary tree implementation from Project 5 in the real world
instead of `std::map`, you (generally) wouldn't use raw pointers for memory
management if you don't need to.

Bjarne Strousup, creator of the C++ language, actually discourages using "raw"
pointers (with explicit calls to `new` and `delete`) in production code;
instead, he recommends using the C++11 smart pointer types declared in the
`<memory>` header. Google's style guide, likewise, [recommends using smart
pointers](https://google.github.io/styleguide/cppguide.html#Ownership_and_Smart_Pointers)
instead of "raw" pointers.

## Why bother learning for this course?

**Ease of use.** The purpose of the smart pointers is to "handle the legwork
for you": they take responsibility for deallocating memory once they're
destroyed (helping to avoid memory leaks and "use after free" segfaults), and
their design helps to enforce program invariants.

**Style grading.** When used properly, smart pointers simplify your code and
make it more elegant; this works to your advantage during style grading.

**Gain familiarity with the STL.** Leaving aside obvious restrictions (no
`<thread>` in Project 2), you have broad freedom in 482 to use whatever STL
headers you want; many of their facilities are useful. Learning new programming
tools is an essential skill for a programmer and one that you should practice.

**Handy for 482 projects.** There are a couple places in the 482 projects
where proper use of smart pointers can greatly simplify your project
implementation, or just help to avoid common bugs.

## Worked Example: By Playing a Children's Card Game!

### Using `std::unique_ptr`

Consider the polymorphic `class Player` from EECS 280, Project 3.

To construct polymorphic Player objects without having access to the
constructors of `SimplePlayer` or `HumanPlayer`, client code must use the
following function:

```cpp
//EFFECTS: Returns a pointer to a player with the given name and strategy
//To create an object that won't go out of scope when the function returns,
//use "return new Simple(name)" or "return new Human(name)"
//Don't forget to call "delete" on each Player* after the game is over
Player* Player_factory(const std::string& name, const std::string& strategy);
```

Note that you need to explicitly delete the returned `Player*` once you're
finished with it. If you forget, then you'll leak memory.

But what if this function had no documentation, as often happens in real life?
What if it was literally just:

```cpp
// make a simple or human player
Player* Palyer_factry(const std::string& name, const std::string& strategy);
```

Then you have no idea (without looking at the function definition) whether
_you_ should delete the pointer, or if there are some global datastructures
where the pointer is stored and you need to call a `Player_destroy` function
to maintain program correctness, or if the pointer points to an object on the
stack and deleting it will segmentation fault.


A more "self-documenting" way to write this function would be to have it return:

```cpp
std::unique_ptr<Player> Player_factory(const std::string& name,
                                       const std::string& strategy);
```

[`std::unique_ptr`](https://en.cppreference.com/w/cpp/memory/unique_ptr) is
an RAII wrapper around a dynamically allocated pointer. Only one `unique_ptr`
can exist for a managed pointer at a given time; the STL enforces this by
disabling the `unique_ptr`'s copy-assignment operator and copy-constructor.
When the `unique_ptr` goes out of scope, the wrapped pointer is `delete`d by the
`unique_ptr` destructor.

You would use it something like this:

```cpp
vector<unique_ptr<Player>> game_players;

// ...

// looks like you're going to the shadow realm, jimbo
unique_ptr<Player> human = Player_factory("Hugh Neutron", "Human");

game_players.emplace_back(std::move(human));
// OR, assume that game_players is filled with unique_ptr(nullptr),
unique_ptr<Player>& back = game_players.back();
back = std::move(human);

assert(!human);  // human holds nullptr
```

[`std::move()`](https://en.cppreference.com/w/cpp/utility/move), is an STL
function (defined in `<utility>`) that allows you to activate an object's [move
assignment operator.](https://en.wikipedia.org/wiki/Move_assignment_operator)
(Formally, it converts the object into an [rvalue reference.](move_semantics/README.md))
While a copy-assignment operator (like the one taught in EECS 280) typically
copies each member of the "right-hand-side" object into the left-hand-side,
a _move-assignment_ operator "steals" the contents of the RHS and puts them in
the LHS.

This allows you to transfer ownership of uncopyable objects without strictly
needing to use pointers. The object itself is never truly "copied": only one
("real", populated) `std::unique_ptr` object exists at a time. (In both
examples, `std::unique_ptr<Player> human` is left holding a `nullptr`, so its
destructor does nothing when it leaves scope.)

Note that the `mutex` and `cv` classes that we provide have deleted
copy-assignment and copy-constructor operators, but do have (optional to
implement for Project 2) move-assignment and move-constructor operators. This is
not true of standard library `std::mutex`es and `std::condition_variable`s,
which (once constructed) must live in the same memory address until they are
destroyed; if it were necessary to transfer ownership of an `std::mutex` or
`std::condition_variable` between class instances (e.g. in a move-assignment
operator), then one could allocate and manipulate them through
`std::unique_ptr`.

`std::unique_ptr`'s uniqueness guarantee would be useful if you have objects in
your code that you want to be unique, i.e. you want/require them to exist in
only one place at a time.

Actually constructing a `unique_ptr` can be done by:

```cpp
Player* human = new HumanPlayer("Hugh Neutron");
return std::unique_ptr<Player>(human);  // unique_ptr "takes ownership"

// OR,
// same as above, but more concise,
return std::unique_ptr<Player>(new HumanPlayer("Hugh Neutron"));

// OR,
// the "standard" method, comparable to vector::emplace_back, arguments "pass
// through" to HumanPlayer constructor
//
// std::unique_ptr<HumanPlayer> is implicitly convertible to std::unique_ptr<Player>
return std::make_unique<HumanPlayer>("Hugh Neutron");
```

### But what if you want multiple copies of the pointer? (`std::shared_ptr`)

`std::unique_ptr` is (intentionally) fairly restrictive in how it can be used
and shared; only one person "has" the pointer at a time. If you want multiple
objects to have pointers to the object, but still want the benefits of RAII, you
can use `std::shared_ptr`.

Using `shared_ptr` is like relying on Python's garbage collector: the
`shared_ptr` keeps an [internal reference count](https://en.cppreference.com/w/cpp/memory/shared_ptr/use_count)
of how many `shared_ptr` objects exist that point to the managed object. This
reference counter increments whenever a `shared_ptr` is copy-constructed or
copy-assigned and decrements whenever a `shared_ptr` is destroyed. When the
reference count finally drops to zero (i.e.  when the last `shared_ptr` to that
object is destroyed), the managed object is destroyed.

You would construct it like this:

```cpp
Player* human = new HumanPlayer("Hugh Neutron");
return std::shared_ptr<Player>(human);  // shared_ptr "takes ownership"

// OR,
// same as above, but more concise,
return std::shared_ptr<Player>(new HumanPlayer("Hugh Neutron"));

// OR,
// the "standard" method, comparable to vector::emplace_back, arguments "pass
// through" to HumanPlayer constructor
//
// std::shared_ptr<HumanPlayer> is implicitly convertible to std::shared_ptr<Player>
return std::make_shared<HumanPlayer>("Hugh Neutron");
```

And you don't need to rely on move-assignment, since `std::shared_ptr` can be
copy-assigned and copy-constructed.

```cpp
vector<shared_ptr<Player>> game_players;

shared_ptr<Player> human = Player_factory("Hugh Neutron", "Human");
game_players.emplace_back(human);
// OR just,
game_players.push_back(human);

assert(game_players.back() == human);  // true

// OR, assume that game_players is filled with shared_ptr(nullptr),
shared_ptr<Player>& back = game_players.back();
back = human;
```

If a `shared_ptr` to one `Player` is overwritten by a `shared_ptr` to
a _different_ `Player`, the `shared_ptr` [still handles the reference count
correctly](https://en.cppreference.com/w/cpp/memory/shared_ptr/operator%3D)
(refcount for the original `Player` decrements, but increments for the
"new" `Player`).

Note, however, that you should not *construct* two `shared_ptr`
objects from the same dynamically allocated raw pointer:

```cpp
Player* p = new HumanPlayer("Hugh Neutron");
std::shared_ptr<Player> hugh(p);
std::shared_ptr<Player> hugh_clone(p);  // BAD, maintains a different refcount than `hugh`!
```

These two `shared_ptr` objects would maintain *separate* reference counts,
and *both* would call `delete` on `p` when those counts reach zero, causing a
double-free. The same would occur if you were to do this with two
`unique_ptr`s.

### Beware of cyclic references! (`std::weak_ptr`)

Let's say that `HumanPlayer` looked like:

```cpp
class HumanPlayer : public Player
{
        std::string name;

        std::vector<Card> hand;

        // don't you know you are my very...
        std::shared_ptr<Player> best_friend;

public:

        void set_best_friend(std::shared_ptr<Player> fren) {
            best_friend = fren;
        }
```

And that we did something like this:

```cpp
// use dynamic_pointer_cast instead of dynamic_cast to downcast from
// Player to HumanPlayer
std::shared_ptr<HumanPlayer> taki =
    std::dynamic_pointer_cast<HumanPlayer>(game_players[0]);

std::shared_ptr<HumanPlayer> mitsuha =
    std::dynamic_pointer_cast<HumanPlayer>(game_players[1]);

taki->set_best_friend(mitsuha);
mitsuha->set_best_friend(taki);
```

Then `taki` has a `shared_ptr` reference to `mitsuha`, so her refcount will
never go below 1 for as long as the `taki` object is alive. And `mitsuha` has
a `shared_ptr` reference to `taki`, so his refcount will never go below 1 for as
long as `mitsuha` is alive. This is a _cyclical reference:_ the two will keep
each other alive forever! (Or at least, until the program exits.)

As desirable as shared immortality might be in the real world, this can lead to
a _de facto_ memory leak; these `shared_ptr` objects, which were supposed to
take care of destroying themselves, will instead live forever.

We can avoid this using `std::weak_ptr`. `weak_ptr` is a "weak-reference" to the
same object as an `std::shared_ptr`. The `weak_ptr`'s existence does not affect
the `shared_ptr` reference count: in order to use the `weak_ptr`, you have to
`std::weak_ptr::lock()`, which returns a `shared_ptr` to the object if it still
exists, or a `shared_ptr(nullptr)` if it doesn't.

```cpp
// class HumanPlayer
// don't you know you are my very...
std::weak_ptr<Player> best_friend;

void set_best_friend(std::weak_ptr<Player> fren) {
    best_friend = fren;
}
```

```cpp
// can construct a weak_ptr from a shared_ptr
taki->set_best_friend(std::weak_ptr<HumanPlayer>(mitsuha));
mitsuha->set_best_friend(std::weak_ptr<HumanPlayer>(taki));

// the above is unnecessary, though; weak_ptr is implicitly constructible
// from shared_ptr
taki->set_best_friend(mitsuha);
mitsuha->set_best_friend(taki);

std::weak_ptr<Player> takis_friend = taki->get_best_friend();
std::shared_ptr<Player> resolved_takis_friend = takis_friend.lock();
if (resolved_takis_friend) {
    cout << resolved_takis_friend->get_name() << endl;
} else {  // locked shared_ptr is nullptr
    cout << "mitsuha is dead" << endl;
}
```

As a general rule, when "sharing" a `shared_ptr`, you should always pass
`weak_ptr` to avoid cyclical references. Passing a "full" `shared_ptr` for an
object to a second object suggests that the second object should never
"outlive" the first object; this may be useful if the second object needs
the first object in order to work properly, but it should be avoided in the
general case.

## Summary

- C++11 smart pointers, defined in `<memory>`, are the industry-preferred method
  for managing pointers to dynamically-allocated memory.
- Use `unique_ptr` if you need a single, discrete object, but need a pointer
  for whatever reason (e.g. polymorphism, object can't be default-initialized
  and reassigned later, etc.).
- Use `std::move()` from `<utility>` to transfer ownership by "dumping"
  a `unique_ptr`'s contents into another.
- Use `shared_ptr` if multiple objects need a pointer to the object.
- Beware of cyclical references with `shared_ptr`; wherever possible, pass
  `weak_ptr` references to the original `shared_ptr`, to avoid keeping
  a "zombie" object alive in memory for longer than necessary.
