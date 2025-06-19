+++
title = "C++ Polymorphism NOT using inheritance or sum types"
date = "2025-06-19"
[taxonomies]
tags = ["cpp", "polymorphism", "type erasure"]
+++

## Motivation and goal
Polymorphism is a day-one-essential in programming. Irrespective of the domain, having to categorize multiple types under an umbrella-type is paradigmatic of programming itself. The choice of programming language usually plays a key role in how we express polymorphism. We usually tend to default to facilities that are natively available. As such, the goal of this post is to highlight a rather obscure approach to polymorphism in C++ called type erasure, which does NOT directly involve inheritance (via abstract base classes) or sum types (via unions, enums, or std::variant).

## Consequences of always using the "hammer"
One might ask why they should bear the mental burden of yet another programming pattern. "Why do it myself when the native facilities of my programming language get the job done?" And they would be right in doing so. The native features are native for a reason after all. And they make things a lot easier. Easier approaches however, don't always lead to simpler solutions, as Rich Hickey [might argue](https://www.youtube.com/watch?v=SxdOUGdseq4). In the case of polymorphism, the native approaches have drawbacks and limitations that make it worthwhile to at least have this other tool in your belt.

### The hammer of inheritance
In his [talk about inheritance and polymorphism](https://www.youtube.com/watch?v=2bLkxj6EVoM), Sean Parent mentions, "Inheritance is instrusive." He then follows it up with a rather profound thinking model about polymorhphism. He explains that polymorphism is not a property of types, but of uses of those types. Strongly coupling polymorphism with inheritance can therefore have negative consequences.

What a type is, can be perverted by how it's used.
```c++															
// Imagine you have a compiler for FooLang.
// `Type` is the base class for primitive types 
// supported in FooLang.
struct Type {
	// `Type` just represents a programming language
	// primitive. Must it really know anything about
	// serialization?
	//
	// `serialize` now also becomes an inherent part of the 
	// identity of all deriving classes. They're not simple
	// type primitives in FooLang anymore (T⌓T).
	virtual void serialize() = 0;
};

```

With inheritance, polymorphic uses become, more or less, mutually exclusive with value semantics.
```c++
// The user ends up bearing the burden of polymorphism
// via inheritance during polymorphic uses.
//
// Sure, this isn't the worst thing in the world. But why
// not do better if we know we can?
std::vector<Type> types;
types.push_back(std::make_unique<IntType>(...));
types.push_back(std::make_unique<FloatType>(...));

for (const auto type : types) {
	type->serialize();
}
```

Inheritance can introduce accidental complexity.
```c++
// `Base` is non-copyable.
struct Base {
	std::unique_ptr<int> ptr;
};

// By way of deriving from `Base`, `Derived`
// loses its copy semantics too. 
struct Derived : public Base {};
```

### The hammer of sum types
Moving on to sum types. What's wrong with sum types? Nothing, really! Only they don't fare too well when we need to be polymorphic over an open set of types. A lot of real world use-cases involve polymorphism over an open set of types. The [MLIR compiler framework](https://mlir.llvm.org/) is one such example. MLIR allows users to create domain-specific IRs (which comprise the open set of types) and at the same time provides transformation and optimization passes that would polymorphically work on any IR.

## Polymorphism via type erasure
We're finally here! Let's see polymormphism via type erasure in action. We will first establish a baseline example, where polymorphism is implemented via inheritance. We will then evolve this example step-by-step, eventually arriving at the type erasure based approach.

### Baseline example
We have a `Shape` type with a pure virtual function called `draw`. And we have types `Circle` and `Square` that extend `Shape` and provide their own implementations of `draw`. The goal is to polymorphically use a `Shape` to call `draw` on `Circle`s and `Square`s.

> **Note** - A similar baseline example can also be constructed via sum types, but I think you'll get the point.

```c++
#include <iostream>
#include <memory>
#include <vector>

// Abstract base class
struct Shape {
	virtual void draw() const = 0;
};

// Inheriting class
struct Circle : public Shape {
	void draw() const override {
		std::cout << "Drawing a circle..." << std::endl;
	}
};

// Another inheriting class
struct Square : public Shape {
	void draw() const override {
		std::cout << "Drawing a square..." << std::endl;
	}
};

int main() {
    // Polymorphic use
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>());
    shapes.push_back(std::make_unique<Square>());

    // Prints -
    // Drawing a circle...
    // Drawing a square...
    for (auto& shape : shapes) {
        shape->draw();
    }
}
```

### Revert inheritance's intrusion
Building upon our baseline example, let's now free `Circle` and `Square` from inheritance and revert them back to simple geometrical primitives. `Circle` and `Square` can retain their respective `draw` methods. But we could also make them completely impervious to the concept of drawing and define free drawing functions instead. We might need to do this if we cannot directly modify `Circle` and `Square`, perhaps because they are defined in another library.

Now, to be able to continue using `Circle` and `Square` polymorphically, we need to introduce polymorphism "externally". We do this using 2 new types - `ExternalDrawInterface` and `ExternalDrawImpl`. The former is the "external" interface, while the latter is the "external" implementation, in the form or a templated wrapper over original types.
```c++
#include <iostream>
#include <memory>
#include <vector>

struct Circle {
    void draw() const {
        std::cout << "Drawing a circle in CIRCLE..." << std::endl;
    }
};

// Let's assume `Square` is a library type not under out control.
struct Square;

// We can implement external polymorphism for it via free functions 
// like this.
void draw(Square const& square) {
    std::cout << "Drawing a square outside SQUARE..." << std::endl;
}

struct ExternalDrawInterface {
    virtual ~ExternalDrawInterface() {}
    virtual void draw() const = 0;
};

// Thanks to C++20 concepts, we can deal with both, types
// implementing `draw` as methods and types that don't,
// in a single implementation of `ExternalDrawInterface::draw`.
template <typename T>
concept HasDrawMethod = requires (T t) {
    { t.draw() } -> std::same_as<void>;
};

template<typename T>
struct ExternalDrawImpl : public ExternalDrawInterface {
    ExternalDrawImpl(T&& object) : object(std::move(object)) {}

    ~ExternalDrawImpl() override {}

    void draw() const override {
        if constexpr (HasDrawMethod<T>) {
            object.draw();
        } else {
            ::draw(object);
        }
    }

    T object;
};

int main() {
    // Polymorphic use
    std::vector<std::unique_ptr<ExternalDrawInterface>> shapes;
    shapes.push_back(std::make_unique<ExternalDrawImpl<Circle>>(Circle{}));
    shapes.push_back(std::make_unique<ExternalDrawImpl<Square>>(Square{}));

    // Prints -		
    // Drawing a circle in CIRCLE...
    // Drawing a square outside SQUARE...
    for (auto& shape : shapes) {
        shape->draw();
    }
}

```

### Bring back value semantics
We have freed `Circle` and `Square` from the burdens of polymorphism, but we haven't freed the user from the inconvenience of using these types polymorphically. We could enable a value semantics based interface and abstract away the need to explicitly allocate and manage memory via pointers.

In order to do so, we will introduce a `Drawable` type that will encapsulate `ExternalDrawInterface` and `ExternalDrawImpl`, and expose simple APIs to construct `Drawable`s and draw them.
```c++
#include <iostream>
#include <memory>
#include <vector>

// Code declaring `Circle` and `Square` types
// ...

template <typename T>
concept HasDrawMethod = requires (T t) {
    { t.draw() } -> std::same_as<void>;
};

// `Drawable` is the new user-facing API.
class Drawable {
private:
    struct ExternalDrawInterface {
        virtual ~ExternalDrawInterface() {}
        virtual void draw() const = 0;
    };

    template<typename T>
    struct ExternalDrawImpl : public ExternalDrawInterface {
        ExternalDrawImpl(T&& object) : object(std::move(object)) {}
        ExternalDrawImpl(T const& object) : object(object) {}

        ~ExternalDrawImpl() override {}

        void draw() const override {
            if constexpr (HasDrawMethod<T>) {
                object.draw();
            } else {
                ::draw(object);
            }
        }

        T object;
    };

    // You might we wondering a few things here.
    //
    // Q1. Why can't we just store a plain, 
    // simple `ExternalDrawImpl<T>` object here instead of 
    // introducing pointers?
    //
    // Ans. We could! But doing so means that `T` has to be a valid 
    // type-parameter. We cannot introduce a valid type-parameter `T`
    // without making `Drawable` a class template.
    //
    // And as soon as we make `Drawable` a template, we undo
    // any form of type erasure and loose the ability to use
    // `Drawable` polymorphically.
    //
    // Q2. Why a `std::shared_ptr` instead of a `std::unique_ptr`?
    //
    // Ans. `std::shared_ptr` lets `Drawable` retain its value semantics.
    // We could also have used a `std::unique_ptr` here, but then would have
    // had to implement special member functions such as the copy constructor
    // and copy assignment operator to recover value semantics.
    std::shared_ptr<ExternalDrawInterface> ptrToDrawable;
public:
    template<typename T>
    Drawable(T&& t): ptrToDrawable(std::make_shared<ExternalDrawImpl<std::remove_reference_t<T>>>(std::forward<T>(t))) {}

    void draw() {
        ptrToDrawable->draw();
    }
};

int main() {
    // Ta daa...a much more ergonomic polymorphic use!
    std::vector<Drawable> shapes;
    shapes.emplace_back(Circle{});
    shapes.emplace_back(Square{});

    // Prints -     
    // Drawing a circle in CIRCLE...
    // Drawing a square outside SQUARE...
    for (auto& shape : shapes) {
        shape.draw();
    }
}
```

## Conclusion and resources for exploring polymorphism via type erasure further
That was a lot, but hopefully it was worth it! If this post piqued your interest, you may want to check out the following resources (some of which I may already have referred to earlier):

1) [Inheritance is the base class of evil](https://www.youtube.com/watch?v=-ss9XGsENPE) - Sean Parent discusses a mental model to think about Polymorphism and decouple it from inheritance via type erasure (although he does not use that exact name).

2) [Breaking dependencies via type erasure](https://www.youtube.com/watch?v=4eeESJQk-mw&t=3s) - Klaus Iglberger really dumbs down the type erasure based approach to polymorphism via his classic shape class hierarchy examples. In fact, the code examples I've used in this post, were inspired by Klaus' talk.

3) [MLIR op interfaces](https://www.youtube.com/watch?v=VCJAmOFvnh4) - Mehdi Amini does a deep dive on how MLIR leverages the type erasure pattern to implement op interfaces. This talk is quite interesting as Mehdi explains the different ways in which MLIR is taking the type erasure pattern further to keep its runtime software performant.
