---
layout: post
title: "Adventures in the land of reference types"
date: 2012-10-04 19:58
comments: true
categories: [Programming]
---

While reading someone else’s C++ code, I have recently come across something which could perhaps best be described as **passing-by-reference (in a constructor) to an instance reference variable**. Aside from being quite a mouthful, it is not something I had come across before and thought it interesting enough to document here.

Unfortunately this makes this post rather limited to a specific domain of interest. All I can do is apologise for my technical ramblings. Conversely, this may seem blindingly obvious to some. I had genuinely never seen reference types used this way.

### What is passing by reference again? ###

In C/C++ a function will make its own copy of any argument provided to it, then operate on it in a local context (pass-by-value). This is not always desired behaviour -- the overhead of copying could be significant, or you may wish for a function to pluck into the very depths of your data and directly modify it.

To avoid data being copied when passed to a function, you can declare the function as taking a pointer to the data, and so you're effectively passing the *memory location* of the address by value. Doing so, however means you have to use a different notation when addressing the data in the function (*dereferencing*).

In C++ (not C) you have another option: pass-by-reference. A small change in the function declaration and you’re now passing a memory address to the function, but that process is made all but invisible, avoiding pointer notation.

{% pullquote %}
If you’re working in C++, it’s mainly a question of personal preference whether to use pointers or pass-by-reference. {"It is however often considered bad practice to use pass-by-reference if you’re planning to modify the data being passed."} Why is this? Given we’re eschewing any sign of pointer notation, the caller has no indication that their data may be modified unless they inspect the function. 

{% endpullquote %}

<!-- more -->

``` cpp
//a naughty function
int subtractOne(int &anumber) {
   return --anumber;
}

int main(void) {
   int mynumber = 5;
   int newnumber = subtractOne(mynumber); //no indication subtractOne will modify mynumber
   assert( mynumber==5 ); //nuh-huh
}

```

Of course we could just resort to always using pointers in C++, but passing by reference does have the nice feature of making it a bit clearer that a function is not responsible for a variable's memory management. Which is nice. Also we don’t have to deal with pointer notation, which can get tiresome (and ugly), in certain situations.

So a good rule of thumb is: pass-by-reference if it’s an option and if you don’t modify the data being passed. To future proof a pass-by-reference function, a common pattern is to *pass-by-reference to const*, preventing anyone accidentally updating the function in a way that modifies the data. This pattern is [often found in C++ style guides](http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml?showone=Reference_Arguments#Reference_Arguments "Google's style guide").

### Reference Types ###

Now things get interesting. There’s nothing stopping you from actually declaring a variable as a reference type. Like so:

``` cpp
MyType& var;
```

But wait. This won’t compile. A reference type isn’t allowed to refer to nothing! Unlike a pointer, it isn’t even allowed to refer to NULL.

``` cpp
MyType& var = some_other_var;
```

Better. Now have two ways of referring to the same data, var and some_other_var. All this without using a pointer. The variable var will **always** refer to that location in memory. Unlike a pointer, it can’t refer to anything else from this point onwards. Neat! Right? Okay, maybe the benefits of this aren't immediately obvious!

Imagine for a moment that you created a class called Player, that requires the data in a class Settings during the entire lifetime of the Player object.

You could provide your Settings object to the Player object in the Player’s constructor. You may have done so by doing pass-by-value. You may have written lots of code based on this fact.

``` cpp Like so
Player p(name, settings);
```

Your settings class grows over the lifetime of the project. And the number of players you initialise also grows. Suddenly you realise you shouldn’t really be making all these copies of the settings object.

{% pullquote %}
{"Instead of migrating to using pointers and refactoring your code, you could declare a member variable in player as a Settings reference type."} Do you see where this is going? If you then provide the Settings object to each Player you create, by pass-by-reference in the initialisation list, you can set a member variable of type reference to refer to that Settings data. It *has* to be in the initialisation list. Remember, reference types can’t be uninitialised, so you can’t wait around and assign it in the body of your constructor, you lazy oaf.
{% endpullquote %}

``` cpp Instead of this
class Player {
public:
    Player(std::string n, const Settings *s): settings(s), name(n) {}
private:
    const Settings *settings;
    std::string name;    
};

Player p(name, &settings);
```

``` cpp You could do this
class Player {
public:
    Player(std::string n, const Settings &s): settings(s), name(n) {}
private:
    const Settings &settings;
    std::string name;    
};

Player p(name, settings);
```

So there you go, now you can access the Settings in your settings member variable, across the lifetime of your Player object without needing pointer notation and without copying the data. Amazing!

Here’s some example code you can compile and play around with:

```
#include <string>
#include <iostream>

class Settings {
public:
    Settings(int v): value(v){}
    int value;
};

class Player {
public:
    Player(std::string n, const Settings &s): settings(s), name(n) {}
    void printInfo() { std::cout << settings.value << ", " << name << std::endl; }
private:
    const Settings &settings;
    std::string name;
};

int main(void) {
    Settings s(5);
    
    Player p("Me", s);
    s.value = 4;
    p.printInfo();
}
```

### Bad references ###

Note the use of ampersands in the arguments and the type declaration. It’s hugely important to not forget one or the other. Let’s see what happens if you do:

1. You forget the amerspand in the member variable type declaration: You pass-by-reference correctly into the constructor, but now, instead of assigning a *reference* member variable to that data, your initialisation list copies the data that the pass-by-reference points to, into the member variable. Booh!
2. You forget the ampersand in the argument list: Player constructor doesn’t do pass-by-reference and copies your data. The settings member reference variable is assigned to the location of that copy. Once out of the constructor, that copy is released. Congratulations, your reference variable is now addressing random memory. Good thing you declared it const, eh?

Both situations will compile g++ without so much as a warning. Eek! (Valgrind will kick up a fuss on number 2, though).

