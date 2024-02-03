---
layout: post
title: C++ 'Type Erasure' Explained
author: Dave
---

I recently stumbled across this pattern on a Hacker News post.
It's a neat toy, but I had a hard time finding a good explanation (most of the
information I found jumped straight into examples before really motivating 
what was going on).
In this post, I'll try to derive the pattern from first principles instead.

## Polymorphism with Interfaces

If you're fluent in C++, this section should be pretty obvious :)

The typical way to achieve polymorphism in C++ is to define an interface
consisting of pure-virtual methods you want to be able to call.
Then, for each implementation that you want to use polymorphically,
you create a subclass that inherits from the base class and implement those
methods.

As an example, let's implement [See 'n Say](http://www.amazon.com/Fisher-Price-See-Say-Farmer-Says/dp/B00JYCN84A).
We start with an interface class:

~~~cpp
class Animal
{
public:
    virtual const char *see() const = 0;
    virtual const char *say() const = 0;
};
~~~

And add a few concrete implementations:

~~~cpp
class Cow : public Animal
{
public:
    const char *see() const { return "cow"; }
    const char *say() const { return "moo"; }
};

class Pig : public Animal
{
public:
    const char *see() const { return "pig"; }
    const char *say() const { return "oink"; }
};

class Dog : public Animal
{
public:
    const char *see() const { return "dog"; }
    const char *say() const { return "woof"; }
};
~~~

Now we can use these implementations generically, by coding against the
interface:

~~~cpp
void seeAndSay(Animal *animal)
{
    printf("The %s says '%s!'", 
        animal->see(), 
        animal->say());
}
~~~

Not rocket science, right?

## Polymorphism with Templates

Inheritance is a good solution to problems that require polymorphism, as long
as the concrete types you're working with (`Cow`, `Pig` and `Dog` in the example
above) all inherit from a common base (`Animal`), which exposes all the
required functionality.

But sometimes the concrete types you're trying to make polymorphic can't
inherit from a common base.
You may not have control of the concrete types (e.g. think STL types like
`std::string`), or it may not even be possible for the concrete type to inherit
(e.g. built-ins like `int`).

If you're in this situation, however, you're not out of luck!
Even if the concrete types don't share a common base, if they conform to a
common interface (that is, they can be used the same way by a caller), we can
instead use a template to make the types polymorphic:

~~~cpp
template <typename T>
void seeAndSay(T *animal)
{
    printf("The %s says '%s!'", 
        animal->see(), 
        animal->say());
}
~~~

You can call this above method on `Cow`s, `Pig`s, `Dog`s, and anything else
that has zero-argument `see()` and `say()` methods that return strings.
This works due to the way templates are compiled: when you invoke a template on
a type, the compiler compiles a new overload of the method, specialized for the
concrete type you're passing in.
Thus, as long as the method would compile with `T` replaced with the concrete
type (say, `Dog`), the template invocation is valid.

To illustrate this, when you call:

~~~cpp
Dog dog;
seeAndSay(&dog);
~~~

The compiler compiles the method `seeAndSay<Dog>`, by more or less replacing
`T` with `Dog`.
The body of that method would look something like this:

~~~cpp
void seeAndSay<Dog>(Dog *animal)
{
    printf("The %s says '%s!'", 
        animal->see(), 
        animal->say());
}
~~~

If you tried to pass in a type that doesn't conform to the 'interface' (say,
`std::string`), the compiler would hit an error when you tried to compile the 
method call, complaining that `std::string` doesn't have `see` or `say` methods.

## Drawbacks to Template Polymorphism

Although achieving polymorphism with templates is a neat trick, there are two
drawbacks:

First, we can't shove disparate types into an array.
When we were using interfaces, we could store an instance of each of `Cow`,
`Pig` and `Dog` in an array of `Animal`:

~~~cpp
void pullTheString()
{
    Animal *animals[] = { new Cow(), new Pig(), new Dog() };

    size_t len = sizeof(animals) / sizeof(Animal*);
    size_t index = rand() % len;

    seeAndSay(animals[index]);
}
~~~

However, with the template-based polymorphism approach, we couldn't create this
array, because there is no common subtype for the array:

~~~cpp
    ??? animals[] = { new Cow(), new Pig(), new Dog() };
~~~

The second drawback is a little more subtle. 
Anybody who uses the template-based `seeAndSay()` method has two options:

1. If the concrete type is known, the method can explicitly specify the
   concrete type, non-polymorphically.
2. Otherwise, the caller must _also_ be a template, to pass along the template
   type (`typename T`) to `seeAndSay()`.

Since you're employing polymorphism in the first place, most callers will
likely fall into the second group, meaning large swathes of your program will
need to be implemented in templates.
This can get out of hand quickly, making your program hard to read and hard to
organize. 
Overuse of this technique can make it take longer to compile your program, and
can bloat the size of your program, wasting space and making it take longer to
start your program at runtime.

Yuck!

## Kernel of an Idea

Pretend, for some reason, `Cow`, `Dog` and `Pig` are set in stone, and the
designers originally did not give them a common base class.
We would like to unite them under some common base class ourselves.
And, since we don't control the implementation of `Cow`, `Pig` and `Dog`, it's
not possible for us to simply change them to inherit from a base interface.

Here's a basic plan for fixing this: if we don't have the inheritance chain we
want, and we can't change the objects to make them inherit, then we can build
our own inheritance chain out of wrapper objects.
That is, we define our own interface, and implement it multiple times.
Each implementation of the interface wraps a `Cow`, `Dog` or `Pig`, and calls
into that for all the virtual methods.

In this example, our common interface might be:

~~~cpp
class MyAnimal
{
public:
    virtual const char *see() const = 0;
    virtual const char *say() const = 0;
};
~~~

Then we create wrapper objects which inherit from `MyAnimal`.
Each wrapper does not except but call into the 'real' underlying object:

~~~cpp
class MyCow : public MyAnimal
{
    Cow m_cow;

public:
    const char *see() const { return m_cow.see(); }
    const char *say() const { return m_cow.say(); }
};

class MyPig : public MyAnimal
{
    Pig m_pig;

public:
    const char *see() const { return m_pig.see(); }
    const char *say() const { return m_pig.say(); }
};

class MyDog : public MyAnimal
{
    Dog m_dog;

public:
    const char *see() const { return m_dog.see(); }
    const char *say() const { return m_dog.say(); }
};
~~~

Now we can work with instances of `MyAnimal`, each of which wraps one of `Cow`,
`Pig` or `Dog`:

~~~cpp
void pullTheString()
{
    MyAnimal *animals[] = 
    {
        new MyCow(), 
        new MyPig(), 
        new MyDog()
    };

    size_t len = sizeof(animals) / sizeof(Animal*);
    size_t index = rand() % len;

    seeAndSay(animals[index]);
}

void seeAndSay(MyAnimal *animal)
{
    printf("The %s says '%s!'", 
        animal->see(), 
        animal->say());
}
~~~

This works, but there's a glaring drawback: we have to define one wrapper class
(like `MyCow`) for every concrete type we want to wrap (like `Cow`).
Holy boilerplate, Batman!

However, we've already seen an easy way to have the compiler do this work for
us: by using templates for polymorphism ...

~~~cpp
template <typename T>
class AnimalWrapper : public MyAnimal
{
    const T *m_animal;

public:
    AnimalWrapper(const T *animal)
        : m_animal(animal)
    { }

    const char *see() const { return m_animal->see(); }
    const char *say() const { return m_animal->say(); }
};
~~~

Now we can use the single `AnimalWrapper` template in lieu of `MyCow`, `MyPig`
and `MyDog`, to have the compiler generate the derived class for us:

~~~cpp
void pullTheString()
{
    MyAnimal *animals[] = 
    {
        new AnimalWrapper(new Cow()),
        new AnimalWrapper(new Pig()),
        new AnimalWrapper(new Dog()),
    };

    size_t len = sizeof(animals) / sizeof(Animal *);
    size_t index = rand() % len;

    seeAndSay(animals[index]);
}

void seeAndSay(MyAnimal *animal)
{
    printf("The %s says '%s!'", 
        animal->see(), 
        animal->say());
}
~~~

## The Type Erasure Idiom

What we built above is the basis of the 'type erasure' idiom.
All that's left is to hide all this machinery behind a another class, so that
callers don't have to deal with our custom interfaces and templates:

~~~cpp
class SeeAndSay
{
    // The interface
    class MyAnimal
    {
    public:
        virtual const char *see() const = 0;
        virtual const char *say() const = 0;
    };

    // The derived type(s)
    template <typename T>
    class AnimalWrapper : public MyAnimal
    {
        const T *m_animal;

    public:
        AnimalWrapper(const T *animal)
            : m_animal(animal)
        { }

        const char *see() const { return m_animal->see(); }
        const char *say() const { return m_animal->say(); }
    };

    // Registered animals
    std::vector<MyAnimal*> m_animals;
    
public:
    template <typename T>
    void addAnimal(T *animal)
    {
        m_animals.push_back(new AnimalWrapper(animal));
    }

    void pullTheString()
    {
        size_t index = rand() % m_animals.size();

        MyAnimal *animal = m_animals[index];
        printf("The %s says '%s!'", 
            animal->see(), 
            animal->say());
    }
};
~~~

That's all there really is to it!

This pattern is known as the 'type erasure' idiom because we managed to 'erase'
the concrete types we unified (`Cow`, `Pig`, `Dog`) by hiding them behind a
custom interface (`MyAnimal`).
The key to doing so was to implement this interface with a template
(`AnimalWrapper`) that forwards the interface's methods to the wrapped concrete
type.
Since the wrapper is templated, we can generate a wrapper automatically for any
type that corresponds to the correct interface.

Also note that, even though in the example above, `AnimalWrapper::see` and
`AnimalWrapper::say` both forward to methods called `see()` and `say()`
respectively, there's no need for `MyAnimal` to have the same interface as the
concrete types.
For example, we could instead call `see()` '`getAnimalName()`', and `say()`
'`getAnimalSound()`':

~~~cpp
class SeeAndSay
{
    // The interface
    class MyAnimal
    {
    public:
        virtual const char *getAnimalName() const = 0;
        virtual const char *getAnimalSound() const = 0;
    };

    // The derived type(s)
    template <typename T>
    class AnimalWrapper : public MyAnimal
    {
        const T *m_animal;

    public:
        AnimalWrapper(const T *animal)
            : m_animal(animal)
        { }

        const char *getAnimalName() const 
        {
            return m_animal->see();
        }

        const char *getAnimalSound() const
        {
            return m_animal->say();
        }
    };

    // Registered animals
    std::vector<MyAnimal*> m_animals;
    
public:
    template <typename T>
    void addAnimal(T *animal)
    {
        m_animals.push_back(new AnimalWrapper(animal));
    }

    void pullTheString()
    {
        size_t index = rand() % m_animals.size();

        MyAnimal *animal = m_animals[index];
        printf("The %s says '%s!'", 
            animal->getAnimalName(), 
            animal->getAnimalSound());
    }
};
~~~

## Naming

Both `MyAnimal` and `AnimalWrapper` have accepted standard names.

`MyAnimal` is an example of a type erasure **concept**. 
That is, `MyAnimal` captures the *concept* of an animal, which is shared among
all the concrete types we accept (`Cow`, `Dog` and `Pig`).
In the end, a concept is just the interface we program against internally
(for example, in `SeeAndSay::pullTheString()`).

`AnimalWrapper` is an example of a type erasure **model**.
That is, `AnimalWrapper` models the concrete types as instances of the concept.
The model is a templated wrapper object, which implements the concept interface
and forwards all concept methods to the underlying concrete type.

In parting, let's rewrite our original `SeeAndSay` type erasure example to use
the standard parlance.
Nothing needs to be changed except a few type names:

~~~cpp
class SeeAndSay
{
    class AnimalConcept
    {
    public:
        virtual const char *see() const = 0;
        virtual const char *say() const = 0;
    };

    template <typename T>
    class AnimalModel : public AnimalConcept
    {
        const T *m_animal;

    public:
        AnimalModel(const T *animal)
            : m_animal(animal)
        { }

        const char *see() const { return m_animal->see(); }
        const char *say() const { return m_animal->say(); }
    };

    std::vector<AnimalConcept*> m_animals;
    
public:
    template <typename T>
    void addAnimal(T *animal)
    {
        m_animals.push_back(new AnimalModel(animal));
    }

    void pullTheString()
    {
        size_t index = rand() % m_animals.size();

        AnimalConcept *animal = m_animals[index];
        printf("The %s says '%s!'", 
            animal->see(), 
            animal->say());
    }
};
~~~

## More Information

For more information on C++'s type erasure idiom, try ...

* Andrzej's series on type erasure 
  ([Part 1](https://akrzemi1.wordpress.com/2013/11/18/type-erasure-part-i/) , 
   [Part 2](https://akrzemi1.wordpress.com/2013/12/06/type-erasure-part-ii/) , 
   [Part 3](https://akrzemi1.wordpress.com/2013/12/11/type-erasure-part-iii/) , 
   [Part 4](https://akrzemi1.wordpress.com/2014/01/13/type-erasure-part-iv/))
* [This cplusplus.com article](http://www.cplusplus.com/articles/oz18T05o/)
* [Type Erasure with Merged Concepts](http://aherrmann.github.io/programming/2014/10/19/type-erasure-with-merged-concepts/)

## Special Thanks

Thanks to Šimon Bařinka for pointing out an object lifetime bug in a previous draft!
