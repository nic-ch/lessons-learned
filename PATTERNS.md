[← Back](README.md)

# Patterns

The following are a number of my own interpretation of some patterns and best practices.

## Interface

***Interfaces*** in C++ are abstract classes that implement **no** functionality and only define a set of methods to be implemented by subclasses. *Interfaces* inherit only from other *Interfaces* and are characterized by:

* **No** instance variable.
* Their *destructor* is **public virtual noexcept default** as they can be upcasted to to destruct subobjects.
* Their *constructors* are **protected noexcept default** as they are "constructed" only within subobjects.
* Their *copy, move operator=()* are **protected noexcept default** as they can be "assigned to" only within subobjects.
* **All** their instances methods are **public virtual pure**.

## Mixin

***Mixins*** can be implemented in C++ through classes that implement common generic functionality to be shared by subclasses that do not necessary have anything else in common. *Mixins* inherit only from other *Mixins* and are characterized by:

* **No** instance variable.
* Their *destructor* is **protected noexcept** not virtual **default** as they are never upcasted to and are "destructed" within subobjects.
* Their *constructors* are **protected noexcept** or **private noexcept** only as they are "constructed" only within subobjects.
* Their *copy, move operator=()* are **protected noexcept** only as they can be "assigned to" only within subobjects.

## Base Class

***Base Classes*** implement states and functionality common to their all subclasses. They are characterized by:

* Their *destructor* is **public virtual** as *Base Classes* can be upcasted to to destruct subobjects.
* Their *constructors* are **protected** or **private** only as *Base Classes* are constructed only within subobjects.
* Their *copy, move operator=()* are **protected** only as *Base Classes* are assigned to only within subobjects.

Any standard class with at least one **virtual pure** method is a ***Base Class***.