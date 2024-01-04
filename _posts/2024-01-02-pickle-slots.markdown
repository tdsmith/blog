---
title: "Pickle, slots, attrs, and inheritance"
layout: "post"
---

When slotted [attrs](https://www.attrs.org/en/stable/) classes are derived from a non-attrs base class, the attributes of the base class do not survive pickling.

What does this mean? Why would one do such a thing? What can one do about it?

For a contrived example, consider these two classes:

```python
class MyMixin:
    def __init__(self):
        self.mixed = True

@attr.define
class A(MyMixin):
    spam: str = "eggs"

    def __attrs_post_init__(self):
        super().__init__()
```

Here, `MyMixin` is just some class that doesn't depend on `attrs`.<sup>1</sup>

This generally works the way you probably expect; instances of `A`
all get decorated with a `mixed` attribute.

```python
>>> a = A()
>>> a
A(spam='eggs')
>>> a.mixed
True
```

Wonderful.

The problems don't begin until later, when one discovers that [one has cause](https://blog.nelhage.com/post/pickles-and-ml/) to use
Python's `pickle` module to serialize an instance of the class `A`.<sup>2</sup>

It _seems_ to work:

```python
>>> a_prime = pickle.loads(pickle.dumps(a))  # Round-trip `a` through pickle
>>> a_prime
A(spam='eggs')
```

But the integration tests break violently!
Other code depends on the mixin's attributes surviving serialization, but the mixin attributes _don't_ survive serialization:

```python
>>> a_prime.mixed
AttributeError: 'A' object has no attribute 'mixed'
```

That's funny. Where'd our attribute go?

### The case of the missing member

If we start experimenting, we may notice a couple of things:

* `mixed` gets dropped if we decorate `A` with `@attr.define`.
But in code where we used the old-style `@attr.s` instead of `@attr.define`,
instances round-trip through pickle just fine, preserving their `mixed` attribute. 🧐
* If `MyMixin` is __itself__ an attrs class, the `mixed` attribute always survives a round-trip, whether we use `@attr.define` or `@attr.s`.

If you've been following attrs development, you might have recalled that one of the [key differences](https://github.com/python-attrs/attrs/issues/487) between the "cute" `attr.s` API and the "next generation" `attr.define` API is that the next-generation API defaults to creating __slotted classes__. The [slotted data model](https://docs.python.org/3/reference/datamodel.html#slots) for classes comes with some performance advantages and better hygiene: it prevents users from creating undeclared attributes on an instance at runtime. With attrs, it also comes with a long list of edge cases, [mercifully summarized](https://www.attrs.org/en/stable/glossary.html#term-slotted-classes) in the attrs docs. Could slots be to blame?

Indeed, redefining `A` with `@attr.define(slots=False)` restores our ability to round-trip the `mixed` attribute through pickling.

One of the caveats in the attrs docs might stand out on review:

>  Slotted classes must implement `__getstate__` and `__setstate__` to be serializable with `pickle` protocol 0 and 1. Therefore, *attrs* creates these methods automatically for slotted classes.

Peeking at the [implementation](https://github.com/python-attrs/attrs/blob/23.2.0/src/attr/_make.py#L1031) of these methods explains our observations: the `__getstate__` method that attrs writes for slotted classes works from a list of _attrs properties_ belonging to the type to be serialized and its superclasses. It doesn't try to include any properties of an instance that weren't defined as attrs attribs.

How does this interact with inheritance? Mostly, it doesn't. There is no established mechanism to invoke any `__getstate__ ` or `__setstate__` methods associated with a class's superclasses, and the attrs-generated methods do not try.

### These united states

So, what can we do to help pickle find our missing properties?

A few options are:

* Make the base type (i.e. `MyMixin`) an attrs class. This may not be appropriate if the other children of the base type don't want to use attrs, but it's very easy and only requires a single line-of-code change.
* Somehow customize `__setstate__` and `__getstate__` on every leaf class we want to pickle. It's not sufficient to define these on the base type; even though attrs will avoid overwriting these methods if they are already defined on a class, it ignores inherited implementations. You could imagine implementing this with a class decorator or something similarly cute, but that would still require changes at each point of use of the base class, which would make the example mixin much more annoying to use.
* Types can define a [`__reduce__` method](https://docs.python.org/3/library/pickle.html#object.__reduce__) which (empirically!) takes precedence over `__getstate__` and `__setstate__`. We can implement `__reduce__` on the base type and make it responsible for collecting and setting the right state about itself and any subtypes.

For both options 2 and 3, it's awkward that `MyMixin` acquires responsibility for serializing subtypes that it doesn't know or care about. One possibility is to delegate to any `__(get|set)state__` methods that we find on the instance it receives, which must belong to subclasses, and can take responsibility for their own properties.

### Writing a reduce method for the base type

`__reduce__` is expected to return a weird tuple, of length between 2 and 6. Quoting the pickle docs for each member, with annotations:

> 1&nbsp;. A callable object that will be called to create the initial version of the object.

This is usually `object.__new__`. That'll work for us.

> 2&nbsp;. A tuple of arguments for the callable object. An empty tuple must be given if the callable does not accept any argument.

`object.__new__` receives the type of the object to create. We can use `(type(self),)`.

> 3&nbsp;. Optionally, the object’s state, which will be passed to the object’s [`__setstate__()`](https://docs.python.org/3/library/pickle.html#object.__setstate__) method as previously described.

__This is effectively where we override `__getstate__`.__ The object's state is held in `self.__dict__`, and in any slots on either this type or its children. If we assume that all slotted children are attrs classes, we'll know that know that any subtypes that use slots will have a customized `__getstate__()` method.

What we can do is:
- If `__getstate__` exists, call it, and assume that it handles all of the subtype's responsibilities.
- Otherwise, grab the instance `__dict__`.
- Add any state specific to the slots of the base class.

> 4&nbsp;. Optionally, an iterator (and not a sequence) yielding successive items. These items will be appended to the object either using obj.append(item) or, in batch, using obj.extend(list_of_items).

We don't care about this; we can set it to `None`.

> 5&nbsp;. Optionally, an iterator (not a sequence) yielding successive key-value pairs. These items will be stored to the object using obj[key] = value.

We don't care about this; we can set it to `None`.

> 6&nbsp;. Optionally, a callable with a (obj, state) signature. This callable allows the user to programmatically control the state-updating behavior of a specific object, instead of using obj’s static [`__setstate__()`](https://docs.python.org/3/library/pickle.html#object.__setstate__) method. If not None, this callable will have priority over obj’s [`__setstate__()`](https://docs.python.org/3/library/pickle.html#object.__setstate__).

__This is where we override `__setstate__`.__ We get to provide a callable that accepts a brand new instance of the class, and feeds it the state that the instance needs to contain.

If the instance has a `__setstate__` method, we can accomplish most of our work by calling it, and then slide in our base-class state on top of it.

Otherwise, we're responsible for the whole kit and caboodle. This is a little bit subtle in the usual attrs ways; we need to circumvent any property setters and frozen class behaviours. We can do this by using `object.__setattr__` to assign each of the properties.

### Denouement

A concrete implementation that rescues pickling round-trips for our MyMixin example is:

```python
class MyMixin:
    def __init__(self):
        self.mixed = True

    def __reduce__(self) -> tuple:
        if hasattr(self, "__getstate__"):
            # Subtypes with slots should have customized their pickling behaviour.
            # Rely on the subtype to handle the things it's responsible for.
            state = self.__getstate__()
        else:
            # The subtype must not have slots. Grab its instance dictionary.
            state = self.__dict__
        # Sprinkle our own properties on top:
        if hasattr(self, "mixed"):
            state["mixed"] = self.mixed
        return (
            object.__new__,
            (type(self),),
            state,
            None,
            None,
            _setstate,
        )


def _setstate(obj: object, state: dict[str, Any]) -> None:
    """
    Inspired by https://github.com/python-attrs/attrs/blob/336ea8b691f148b81db62e24bb55c7ca11ddbc6c/src/attr/_make.py#L931
    """
    # Use object.__setattr__ to actually assign in state, to avoid any property setters
    # or frozen class behaviours.
    # I think the precise form of this (instead of repeatedly calling
    # object.__setattr__(obj, key, value)) is a performance microoptimization.
    _bound_setattr = object.__setattr__.__get__(obj)

    if hasattr(obj, "__setstate__"):
        # If the instance has a setstate method, rely on it to handle almost everything.
        obj.__setstate__(state)  # type: ignore
        # Restore our MyMixin state to the object
        if "mixed" in state:
            _bound_setattr("mixed", state["mixed"])
        return

    # The object doesn't know how to restore its state, so we're responsible for it all.
    for k, v in state.items():
        if k == "__weakref__":
            continue
        _bound_setattr(k, v)

@attr.define
class A(MyMixin):
    spam: str = "eggs"

    def __attrs_post_init__(self):
        super().__init__()
```

Finally, we can write:

```python
>>> a = A()
>>> a_prime = pickle.loads(pickle.dumps(a))
>>> a_prime.mixed
True
```

### Endnotes

1: A "mixin" class [provides some functionality](https://stackoverflow.com/questions/533631/what-is-a-mixin-and-why-is-it-useful) to child types, to which the mixin class may not be tightly coupled. It's not important that the base class here is a mixin, instead of some other kind of base class, but it was the motivating example in the codebase where I encountered this strange circumstance.

2: To avoid slandering pickle, running into problems serializing this data layout isn't a pickle-specific problem. A glance at the default [`cattrs`](https://catt.rs/en/stable/) representation of an instance of A is revealing:

```python
>>> cattr.unstructure(a)
{'spam': 'eggs'}
```

There's no attempt to represent the `mixed` attribute here, so it follows that you would have to similarly customize how cattrs structures and unstructures this type to preserve any interesting properties belonging to the mixin class.
