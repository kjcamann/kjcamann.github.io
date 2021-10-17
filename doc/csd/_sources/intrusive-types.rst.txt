***************
Intrusive Types
***************

.. FIXME: This cannot be done yet because of doxygen misparses
   .. doxygenclass:: csg::invocable_constant
   :members:

.. doxygenclass:: csg::compressed_invocable_ref
   :members:

In many cases, e.g. :doc:`intrusive lists <lists-main>`, an iterator will need to access a member sub-object inside the item it currently points to. This means the iterator must also store a reference to the functor that accesses the intrusive link.

That functor is often :cpp:concept:`stateless <csg::stateless>` -- for example, when accessing the item via a ``reinterpret_cast`` at a fixed offset (obtained through ``offsetof``) that is baked into the functor type as a non-type template argument. Functors are usually stored in objects having a `[[no_unique_address]] <https://en.cppreference.com/w/cpp/language/attributes/no_unique_address>`_ attribute, so that they do not require any storage.

However, a *reference* (or pointer) to the functor must still be passed into the iterator, so the iterator can use it. Reference semantics are needed because in the general case, the functor *might* be stateful (it depends on the template argument), so we might need the exact functor instance owned by the container. The problem is that even when the functor is empty, the reference is still pointer-sized. This is solved using this helper class, ``csg::compressed_invocable_ref``, which has an explicit template specialization for stateless functors. Since all stateless instances are fungible, the specialization just returns references to a static inline instance of the functor, and the object itself does not store anything.

A ``compressed_invocable_ref`` is usually a member variable declared with ``[[no_unique_address]]``, so that it requires no storage when it is a reference to a stateless functor.
