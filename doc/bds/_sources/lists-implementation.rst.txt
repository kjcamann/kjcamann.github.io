******************************
BDS lists implementation notes
******************************

.. contents::
   :local:

Link encoding
=============

Links to neighboring list items are stored in intrusive "entry" structures like the one seen below:

.. code-block:: c++

   struct ListItem {
     int i;
     bds::tailq_entry<ListItem> e; // Holds links to neighboring list items
   };

The term "entry" comes from the original ``queue(3)`` library macros, e.g., ``TAILQ_ENTRY(ListItem) e`` would be written instead of ``bds::tailq_entry<ListItem> e``. If you look at the definition of ``bds::tailq_entry`` however, it is much more complex. That complexity comes from the flexibility to use either offset-based entry accessors or invocable entry accessors. As :ref:`explained in the list guide <lists-how-to-choose-entryaccess>`, offset-based entry accessors usually result in better code generation but have stricter requirements (they only work with complete standard layout types).

The neighbor links are stored in a different format depending on the type of entry accessor being used. One additional complexity is that we can declare entry objects without having to specify anything about the entry accessor. Thus, the entry object must be able to work with either format, regardless of which one will eventually be used. Consequently, the "neighbor link" type is a union of two different types, but only one will be used depending on the kind of entry accessor that is eventually specified in the ``tailq_head`` or ``tailq_proxy`` type.

The design is eaiser to understand if we look at both link formats separately first. To make the expostion more concrete, we'll pick a particular entry type like ``tailq_entry<ListItem>`` or ``slist_entry<ListItem>>``, depending on what is shown in the nearest figure.

offset_entry_ref
----------------

When using an offset-based entry accessor, BDS stores neighbor links by storing pointers to the neighbors' entry objects (e.g., pointers to neighboring ``tailq_entry<ListItem>`` members) in a special smart pointer called ``offset_entry_ref`` -- so named because it is a reference to an entry used in the "offset" case. This smart pointer isn't very smart -- it adds almost nothing beyond a plain old ``tailq_entry<ListItem>*``. The reason it is a separate type is only to satisfy the C++ standard's notion of when it is safe to read from an inactive union member -- because an ``offset_entry_ref`` lives in a union with ``invocable_tagged_ref`` that we discuss next -- and there is a "dark corner" in the code where we need that to work.

.. figure:: images/lists-offset-entry-ref.png
   :alt: tailq using an offset_entry_ref link encoding
   :target: _images/lists-offset-entry-ref.png

   A ``tailq`` using an offset-based link encoding. This tends to generate the most efficient code. To reduce clutter, only the links of the middle list item have their arrows shown (click to enlarge).

There is an important symmetry when using the offset link encoding: given a list item of type ``ListItem``, we can get a pointer to the enclosed ``tailq_entry<ListItem>`` via simple pointer math, i.e., by adding the ``offsetof`` bytes to the base address of the item. We can just as easily go in the reverse direction: given a pointer to a ``tailq_entry<ListItem>``, we can calculate the address of the enclosing ``ListItem`` list item by subtracting that same offset. [#lists-container-of]_

Note that this is different from the way that simple linked lists are written -- consider the typical "white board job interview" doubly-linked list:

.. code-block:: c++

   struct ListItem {
     int i;
     ListItem *next;
     ListItem *prev;
   };

The neighbor links in this kind of list point directly to other ``ListItem`` instances, not to "entry" member subobjects. What advantages does a separate entry data structure give us?

One advantage is that it allows us to work with special entries that do not live inside ListItems. This functionality is important for both singly-linked and doubly-linked lists, but is easier to illustrate with singly-linked lists.

Consider that STL-style singly-linked lists contain a member function called ``before_begin``, which returns an iterator that points "before" the first item. In the implementation, the list head itself contains an entry, which points to the entry in the first item. An iterator that is pointing before the first item is actually pointing directly to this special entry, as seen in the figure below:

.. figure:: images/lists-offset-entry-ref-iterator.png
   :alt: Iterator design for slist, which relies on out-of-item entries.
   :target: _images/lists-offset-entry-ref-iterator.png

   The iterator contains a single ``offset_entry_ref`` pointing to the "current entry," regardless of whether that entry exists inside a list item or exists within the head item (click to enlarge).

This feature is even more critical for the tailq design, which is explained in a later section.

invocable_tagged_ref
--------------------

Although offsetof-based link encoding tends to generate the most compact code, it cannot be used with every type. The C++ standard only guarantees that ``offsetof`` will work on standard layout types. Since C++17, an implementation may choose to support other types if its class layout strategy can guarantee a fixed offset for that type. In addition to supporting polymorphic classes, there are a number of other advantages to using "invocable" entry access functors, as explained in :ref:`lists-how-to-choose-entryaccess`.

The invocable link encoding must support the same three features as the offset-based one:

1. The entry reference type, ``invocable_tagged_ref``, must be the size of a single pointer.
2. It must be possible to obtain both the list item pointer and the entry pointer from the entry reference type.
3. It must be able to work with special entries that do not live inside list items.

The ``invocable_tagged_ref`` is a single `tagged pointer <https://en.wikipedia.org/wiki/Tagged_pointer>`_ which can point to either a ``ListItem`` or directly to an entry (``slist_entry<ListItem>`` in the figure below), depending on the low bit of the pointer. Conceptually, it is the union of a ``ListItem*`` and a ``slist_entry<ListItem>*``. If it points to a ``ListItem``, then the enclosed entry can be obtained by invoking the entry accessor, e.g., ``std::invoke(entryAccessor, *static_cast<ListItem *>(ref))``. If it points directly to an ``slist_entry<ListItem>``, then this is one of the "special case" entries that does not live inside an enclosing ``ListItem``.

.. figure:: images/lists-invocable-tagged-ref.png
   :alt: slist using an invocable_tagged_ref link encoding
   :target: _images/lists-invocable-tagged-ref.png

   When the pointer is untagged (unshaded), it points directly to an ``entry_type``; when the pointer is tagged (shaded), it points to a ``ListItem`` (click to enlarge).

entry_ref_codec and entry_ref_union
-----------------------------------

The code orginally used the term "link" but to improve readability, neighbor links are now called "entry refs." The "ref" part of the name emphasizes that they are not entries themselves, but pointer-like objects that refer to entries. The "entry" part of the name emphasizes that their primary purpose is to refer to structures such as ``slist_entry<ListItem>`` -- although they also simulatenously refer to the enclosing ``ListItem`` when it exists.

``entry_ref_codec`` is a helper traits class that shields the list implementation from having to deal with the differences between ``offset_entry_ref`` and ``invocable_tagged_ref``; it provides a partial specialization for each, but gives them a common interface. For example to get an entry, ``offset_entry_ref`` does not need to do any additional work, whereas ``invocable_tagged_ref`` usually needs to invoke the entry access functor on the ``ListItem``. The ``entry_ref_codec`` class offers a static member function (``get_entry``) which accepts the entry access functor and in the offset case, ignores it, but in the invocable case, uses it to extract the entry.

``entry_ref_union`` is a union of both ``offset_entry_ref`` and ``invocable_tagged_ref`` -- this is the type actually used to store neighbor links in entry structures. As mentioned earlier, a union is needed because we do not want to specify the entry access functor at the time an entry is declared, but that is what determines what encoding will be used. Once the entry access functor is known, the list head will declare a static constexpr member pointer (called ``EntryRefMember``) to access the appropriate union member.

There is one tricky implementation detail regarding ``entry_ref_union``: although the list is usually the only entity that manipulates entries, certain entries (e.g., the one in ``tailq_fwd_head``) must be default-constructed before the entry access functor is known.

These constructors perform their initialization through the ``offset_entry_ref`` member of the union, regardless of which union member will be used by the list. This is safe to do because both ``offset_entry_ref`` and ``invocable_tagged_ref`` have a common initial sequence -- a single ``std::uintptr_t`` that holds their respective encodings. In this case, the C++ standard permits reading from a union member that was not used for the write. All subsequent reads and writes after initialization use the correct union member.

Iterator implementation
-----------------------

List iterators store an entry ref for the "current" entry. As described above, with the help of the ``entry_ref_codec`` (and the ``EntryAccess`` functor, for invocable link encoding), the iterator can access both the entry object and the list item.

The iterator is either the size of a single pointer or two pointers, depending on the ``EntryAccess`` functor. The iterator stores a special "smart pointer" to the invocable object, :cpp:class:`bds::compressed_invocable_ref`, which (using template partial specialization) is empty if the ``EntryAccess`` functor is :cpp:concept:`stateless <bds::Stateless>`, i.e., if it is both an empty class and is trivially-constructible. This smart pointer is also declared with ``[[no_unique_address]]``, so if the accessor is stateless, it will not store anything. This is the common case, because both offset-based and `constexpr_invocable`-based functors are stateless.

List design
===========

The singly-linked list design is straight-forward: a head entry points to the first item, and all the intrusive entries point to the next item. The last item has a ``next`` value of ``nullptr`` (encoded as zero). In the case of ``stailq``, an entry_ref to the last item also stored.

.. _lists-tailq-circular-design:

The ``tailq``'s doubly-linked list design is insipred by the `std::list <https://github.com/llvm-mirror/libcxx/blob/master/include/list>`_ implementation in libc++. In this design, the list is actually circular: the ``tailq_entry`` in the list head links the two ends of the list, and corresponds to the ``end()`` sentinel. Its "previous" link points to the last item in the list, and its "next" item points back around to the first item in the list. This is shown in the figure below, which also illustrates the tagged pointers in the ``invocable_tagged_ref`` encoding:

.. _lists-tailq-invocable-figure:

.. figure:: images/lists-tailq-design.png
   :alt: tailq data structure diagram
   :target: _images/lists-tailq-design.png

When the list is empty, both links in the head entry point to itself:

.. figure:: images/lists-tailq-design-empty.png
   :alt: empty tailq data structure diagram

This is the ideal design given the STL container requirements, e.g., that ``*--end()`` is a valid expression that yields the same value as ``back()``. If you work out how to edit the links to insert and remove items "in the middle" of the list, this same code will work even at the boundary conditions (a list of size 0, size 1, etc.) without any special conditionals that check for those cases.

This interesting property makes the circular doubly-linked list one of the "cleanest" and most "algorithmically satisfying" data structures you're ever likely to encounter -- especially when compared to "white board job interview" linked lists which explicitly use ``nullptr`` at the head/tail of the list, and need ``if`` statements in almost every function to treat the boundary cases differently.

Miscellaneous Notes
===================

``iterator_t`` and ``const_iterator_t``
---------------------------------------

Several classes define seemingly superfluous identity type aliases for their iterators, e.g.

.. code-block:: c++

   class iterator;
   class const_iterator;

   using iterator_t = std::type_identity_t<iterator>;
   using const_iterator_t = std::type_identity_t<const_iterator>;

These are needed to work around a strict policy in clang regarding the use of incomplete types in template class member functions, see `this cfe-dev mailing list post <https://lists.llvm.org/pipermail/cfe-dev/2018-June/058273.html>`_ for details.

Unusual code formatting convention
----------------------------------

The naming of variables, functions, and classes may seem inconsistent: sometimes ``lower_case_snake_case`` identifiers are used, other times ``camelCase`` (replete with ``m_`` member prefixes) is used. There is a pattern though: implementation details (such as a private member variables) use camelCase, whereas names in public interfaces use lower_case_snake_case. Why is this?

Because BDS copies LLVM's style: its normal style is camelCase, but for consistency, it uses the STL formatting rules when implementing STL-compatible containers or helper data structures. This is a useful visual indication that the class in question supports the same interface (namely, an iterator-centric interface) as a "normal" STL class. Since almost everything in BDS is designed to be STL compatible, all parts of the public interface use lower_case_snake_case. Meanwhile, all non-public names use the CSG "house style", which is a camelCase style similar to LLVM.

.. rubric:: Footnotes

.. [#lists-container-of] Several popular C codebases contain a special macro that does this "inverse offsetof" operation to obtain the original structure address from one of the members. It is usually spelled ``container_of``.
