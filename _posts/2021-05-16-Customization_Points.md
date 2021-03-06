---
layout: post
title: Customization Points
tags: C++, Customization Points, API, Design
---

Before writing about Customization Points in the sense that most of the Posts and References in the Resource section talk about, I want to quickly summarize which ways I see for the user to customize some specific behavior and what are the advantages and drawbacks. 

# Ways to let the user customize some behavior

## Predicates or Callables

A lot of the STL-algorithms work in the way that you can customize their behavior through a callable (UnaryPredicate or UnaryFunction).

```cpp
std::vector<int> input{1,2,3,4};

std::for_each(std::begin(input), std::end(input), [](int& value){multiplyBy2(value);});
```
Notice that there is no virtual dispatch here, everything is wired at compile time and you don't have a need to implement any interface for your callable.

## Dependency Injection

This is something I regularly use in tests when I have classes building on top of each other. It is useful if you can provide a mock object and inject it to see if the expected methods are called.
```cpp
class Interface
{
public:
    virtual void doSomething() = 0;
}

class SomethingThatDoesSomethingMore
{
public:
    SomethingThatDoesSomethingMore(Interface* interface) : m_customized(interface){};

    void doSomethingMore()
    {
        m_customized->doSomething();
        More();
    }
private:
    interface* m_customized;
}
```

## Policy Based Design

This is very similar to the 'Predicates or Callables' approach, except usually the state is stored somewhere in a class. Examples are `allocators` or `deleters` of the STL.

```cpp
// STL declarations of vector and unique_ptr
template<class T, class Allocator = std::allocator<T>> class vector;
template<class T, class Deleter = std::default_delete<T>> class unique_ptr;
```

Now we can for example (ab)use our `unique_ptr` to delete a file (Please don't do that in production): 
```cpp
void delete_file(std::filesystem::path* file)
{
    std::filesystem::remove(*file);
    delete file;
}

using temporary_file = std::unique_ptr<std::filesystem::path, decltype(&delete_file)>;

int main()
{
    temporary_file file(new std::filesystem::path{"./xxx.txt"}, &delete_file);
}
```

There is a problem with this approach however, now each type using a different policy(that's what these things are called in "Modern C++ Design") is a distinct type and they cannot easily interoperate with each other.

This problem was addressed in C++17 with `std::pmr::polymorphic_allocator`. The user implements the `do_allocate` method of the interface `std::pmr::memory_resource` and injects that method into the `std::pmr::polymorphic_allocator`.

```cpp
// Defined in the STL
class std::pmr::memory_resource
{
public:
    void* allocate(size_t bytes, size_t align)
    {
        return do_allocate(bytes, align);
    }
private:
    virtual void* do_allocate(size_t bytes, size_t align) = 0;
}

// This is where you specify how your allocator works
class my_memory_resource : public std::pmr::memory_resource
{
    void do_allocate(size_t bytes, size_t align) override
    {
        return ::operator new(bytes, std::align_val_t(align));
    }
}

// How we can use it:
int main()
{
    my_memory_resource pool;
    std::pmr::vector<int>{ {1,2,3,4}, &pool};
}
```

Because of the type erasure through `std::pmr::memory_resource` which is injected into the allocator we can now use different allocators for containers and still use them interchangeably. This is a quite well designed customization point.

## Specialization, Overload Resolution and ADL

There is another way that is used to be able to customize behavior. Argument Dependent Lookup. This is the way that `std::swap` is implemented for custom types:

```cpp
namespace xxx
{
struct Container
{
    std::vector<int> samples;
};

void swap(Container& lhs, Container& rhs)
{
    using std::swap;
    swap(lhs, rhs);
}
}
```
This works because we defined a function swap in our own namespace which will be found through ADL. We still make `std::swap` available through `using std::swap`, however it will only be used if no better match is found.

This approach has some problems. 
* It is very cumbersome and easy to get wrong
* Sometimes you don't own the namespace of the classes you want to swap(e.g. std or a 3rd party)
* Since we provide our customization through the public interface we cannot enforce any constrainsts that might be useful

Similar to that we could also let the user specialize a behavior by adding an overload, or if we have a templated function using template specialization.

```cpp
template<typename T>
void customizeThisFunction(T&){};

// We can add a specialized template
template<>
void customizeThisFunction(double& in){};

// Or even overload directly
void customizeThisFunction(int& ){};
```

This is alright if you do not have any constraints that you want to enforce on the function, however if you have some requirements or preconditions those customization points will directly bypass them.

 Remember how the `std::pmr::memory_resource` works? Here we have two dedicated entities, one the user has to implement: `do_allocate` and one that will be invoked by the `pmr::allocator`: `allocate`. 

This separation allows us to enforce our invariants and still allow the user to customize the behavior.

## Customizing Behavior and Enforcing a Constraint = Customization Point Object

So what do we actually want? Lets say we have a `serialize` function that should be customizable but enforce that the type is `std::is_trivially_copyable`. Something like this:

```cpp
namespace mine
{
  struct Thing
  {};
  
  void serialize(Thing)
  {}

}

template<typename T>
void serialize(T) requires std::is_trivially_copyable<T>::value 
{
// ...
}

int main()
{
    // We want both of these calls to dispatch first to the above function for constraint checking only then we want it to call the user specified function
    serialize(mine::Thing{});
    serialize(11);
}
```

The trick that we can use is instead of defining our global serialize function as a free function we will instead declare it as a global variable. We use C++17s inline variables to avoid ODR Violations. 

For the functionality we overload `operator()` perfoming the constraint checking and dispatching then to our dedicated implementation:

```cpp
#include <type_traits>
#include <utility>

namespace mine
{
  struct Thing
  {};
  
  void serialize(Thing)
  {}
}

namespace detail
{
  template<typename T>
  void serialize(T)
  {
    // actual implementation that does something!
  }

  struct serialize_fn
  {
      template<typename T>
      constexpr void operator()(T&& obj) const requires std::is_trivially_copyable<T>::value
      {
          serialize(std::forward<T>(obj));
      }
  };
}

inline constexpr detail::serialize_fn serialize{}; 

int main()
{
    // We want all of these calls to dispatch first to the above function for constraint checking
    serialize(mine::Thing{});
    serialize(11);
}

```


### How it works

The secret lies in how C++ performs name lookup. From [cppreference](https://en.cppreference.com/w/cpp/language/lookup):


>For function and function template names, name lookup can associate multiple declarations with the same name, and may obtain additional declarations from argument-dependent lookup. Template argument deduction may also apply, and the set of declarations is passed to overload resolution, which selects the declaration that will be used. Member access rules, if applicable, are considered only after name lookup and overload resolution.

>For all other names (variables, namespaces, classes, etc), name lookup must produce a single declaration in order for the program to compile. Lookup for a name in a scope finds all declarations of that name, with one exception, known as the "struct hack" or "type/non-type hiding": Within the same scope, some occurrences of a name may refer to a declaration of a class/struct/union/enum that is not a typedef, while all other occurrences of the same name either all refer to the same variable, non-static data member (since C++14), or enumerator, or they all refer to possibly overloaded function or function template names. In this case, there is no error, but the type name is hidden from lookup (the code must use elaborated type specifier to access it).

In other words: Now that our name refers to a global function object no adl will be performed and C++ will always select our global function object as entrypoint, just as we intended.

Thats it.

## Resources

[Suggested Design for Customization Points](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4381.html)
[Blog post about the above](https://ericniebler.com/2014/10/21/customization-point-design-in-c11-and-beyond/)
[Code Dive Presentation](https://cdn2-ecros.pl/event/codedive/files/presentations/2017/code%20dive%202017%20-%20Michal%20Dominiak%20-%20Customization%20points%20that%20suck%20less.pdf)
[Niebloids and Customization Point Objects](https://brevzin.github.io/c++/2020/12/19/cpo-niebloid/)
[Customization Point Design for Library Functions](https://quuxplusone.github.io/blog/2018/03/19/customization-points-for-functions/)
[A Customizable Framework](https://akrzemi1.wordpress.com/2016/01/16/a-customizable-framework/)
[Policy Based Design](https://www.foonathan.net/2017/02/policy-based-design-problem/)
