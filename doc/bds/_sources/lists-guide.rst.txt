***********************
Lists Quick Start Guide
***********************

.. contents::
   :local:

What are BDS lists?
===================

BDS lists are intrusive linked lists based on the `BSD queue(3) library <https://man.openbsd.org/queue.3>`_. That library first appeared in 4.4 BSD, and is available today on most UNIX systems via the header file ``<sys/queue.h>`` (it is also included in glibc and can be found on most Linux systems). Linked lists are used to implement queues in various UNIX kernels, thus the name "queue."

The original queue library has been copied into or reimplemented in many well-known C code bases, including the Linux kernel. Like most data structure libraries written in C, it relies on macros to achieve static polymorphism. BDS lists are C++20 re-implementations that combine the data structure design from BSD queues with the design of STL containers. For example, it achieves static polymorphism using templates instead of macros.

To read more on why intrusive linked lists are a great data structure, see :doc:`lists-rationale`

BDS provides the four data structures from ``queue(3)``:

:slist: Singly-linked lists

:stailq: Singly-linked tail queues; a "tail queue" is a list that keeps track of the last element, which allows fast insertion at the end of the list and fast list concatenation

:tailq: Doubly-linked tail queues, i.e., doubly-linked lists that also have the tail queue features

:list:
   .. warning:: list is deprecated and should be avoided.

   In the original ``queue(3)`` library, there is a difference between a ``LIST`` and a ``TAILQ``, but in BDS there is not. To meet the design goal of being STL-friendly, the C++ implementation of lists and tailqs must be exactly the same (see :ref:`here <lists-diff-with-queue>` for a detailed explanation). If ``<bds/list.h>`` is included, it makes lists available through tailq type aliases, but adds deprecation attributes.

A simple example
================

.. code-block:: c++
   :name: lists-main-example

   #include <iostream>
   #include <bds/slist.h>

   struct ListItem {
     int i;                 // List item's data
     bds::slist_entry link; // Intrusive link that makes us part of a list
   };

   // The head of the list is a class template. The second template
   // parameter is a functor that controls how the list locates the
   // intrusive links inside the list items (in this case, via a member
   // offset calculation).
   using list_t =
       bds::slist_head<ListItem, slist_entry_offset<offsetof(ListItem, link)>, std::size_t>;

   // Create an empty list and two items, then add the items to the list.
   list_t L;
   ListItem item1{.i = 1}, item2{.i = 2};

   L.push_front(&item2); // Note: insertion by pointer. This is explained
   L.push_front(&item1); // in the next section of this guide.

   // Print the items.
   for (const ListItem &item : L)
     std::cout << "This is item #" << item.i << std::endl;

This will print

.. code-block:: none

   This is item #1
   This is item #2

.. _lists-where-are-the-docs:

Where do I find the API documentation?
======================================

There is deliberately very little doxygen documentation for BDS list member functions. They are meant to follow the standard C++ list APIs as much as possible, so you should use your favorite C++ reference (e.g., `cppreference <https://cppreference.com>`_) for `\<list\> <https://cppreference.com/w/cpp/container/list>`_ when referencing tailq and `\<forward_list\> <https://cppreference.com/w/cpp/container/forward_list>`_ when referencing slist or stailq.

However, because BDS lists are intrusive and because they follow the BSD data structure design patterns, there are some differences when compared to normal C++ containers. These changes naturally cause some of the public interface to be different.

The fundamental difference is that BDS lists do not own their elements. A "list" object consists of a small "head" object which points to the first (and possibly last) item in the list. The items themselves have a lifetime distinct from the list head's lifetime and are allocated elsewhere, usually in a memory pool. An illustration of a BDS ``slist`` is shown below:

.. figure:: images/slist-diagram.png
   :alt: BDS slist datastructure diagram

   A BDS list object is little more than a pointer to the intial item; the items are *not* owned by the list, they are only linked into it.

Compare this with node-based lists, like ``std::forward_list``, which own their elements and store them in "node" containers. One possible implementation of this design is illustrated below:

.. figure:: images/node-list-diagram.png
   :alt: A typical node-based list datastructure

   A typical node-based list (i.e., a non-intrusive list) has ownership over the elements, which are typically wrapped inside node objects. The lifetime of node objects is explicitly managed by the list itself.

This fundamental difference gives rise to a variety of smaller API differences, be sure to read the section :ref:`lists-diff-with-stl` for all the details.

* BDS lists cannot be copied.
* List items are neither copied nor moved into the list -- they are just added to it (their intrusive link member variable is modified to splice the item into the list).
* As a consequence of the above, list items are always inserted *by pointer*, i.e., for a list of type ``T``, the insert function requires a ``T*`` value. To avoid breaking with the standard containers too much, all other *non-insertion* member functions continue to use ``T&``, e.g., ``front()`` and ``back()`` return the item, not a pointer to it.
* A list item cannot be added to multiple lists that use the same entry link.
* The list destructor doesn't actually destroy the items (because it does not own them).

To balance out these limitations, BDS lists provide some useful APIs which are not in the standard containers -- see :ref:`lists-extra-methods`.

Choosing the template parameters
================================

As can be seen in the :ref:`example above <lists-main-example>` ``slist_head`` has three template parameters:

:T: The type of the items in the list.

:EntryAccess: The type of function object that accesses the intrusive link, given an instance of ``T``. In the example above, the intrusive link is the data member ``bds::slist_entry link``. The link object is called an "entry" from the name of the corresponding ``queue(3)`` macros, e.g., ``SLIST_ENTRY``. See :ref:`lists-how-to-choose-entryaccess` for more information.

:SizeType: The C++ standard container ``<list>`` offers :math:`O(1)` access to its size by storing the list size inline and maintaining it during each modifying operation. The C++ standard container ``<forward_list>``, and the BSD ``queue(3)`` library do not track list size -- getting the size is an :math:`O(n)` operation that fully traverses the list to count each item. BDS lists allow the user to choose which behavior they want. See :ref:`lists-how-to-choose-sizetype` for more information.

.. _lists-how-to-choose-entryaccess:

Choosing the EntryAccess parameter
----------------------------------

Although the user can provide any kind of function object (including a stateful one), most of the time you will use one of two special pre-defined functors:

The member offset functor
^^^^^^^^^^^^^^^^^^^^^^^^^

This functor is declared as:

.. code-block:: c++

   template <std::size_t Offset> struct slist_entry_offset;

``slist_entry_offset`` (along with its cousins, ``stailq_entry_offset`` and ``tailq_entry_offset``) is a class template parameterized by a single ``std::size_t`` value, which should be the ``offsetof`` of the entry object in the type ``T``. This functor is used in the :ref:`example above <lists-main-example>`.

The invocable/member-access functor
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This functor is declared as:

.. code-block:: c++

   template <auto I> struct constexpr_invocable;

It wraps a compile-time-constant invocable (like a function member pointer, object member pointer, or free function pointer). The following example uses a function member pointer:

.. code-block:: c++

   class ListItem {
   public:
     int i;
     bds::slist_entry &getLink() noexcept { return link; }

   private:
     bds::slist_entry link;
   };

   using list_t =
       bds::slist_head<ListItem, constexpr_invocable<&ListItem::getLink>, std::size_t>;

The implementation calls ``std::invoke(<invocable-template-argument>, <item-of-type-T>)`` to access the list entry. The ``auto I`` template argument is usually a member object pointer, a member function pointer, or a free function taking a single ``T&`` argument (see the documentation of `std::invoke <https://cppreference.com/w/cpp/utility/functional/invoke>`_ for the possibilities).

When to use offset-based vs. invocable accessors
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Offset-based accessors usually result in better code generation, but not all types can support them. The C++ standard only guarantees that ``offsetof`` will work on standard layout types, although since C++17, implementations can choose to allow other types. Because of that freedom, BDS does not enforce the standard layout requirement through a template constraint, so the user must "know what they're doing" when using ``offsetof``.

Invocable accessors are more general, and can solve access protection problems. In some implementations, an ``offsetof(T, <field>)`` appearing outside of T and where ``<field>`` is non-public will fail, unless a friend declaration allows access in the scope where ``offsetof`` appears. Invocables can avoid this by using a public "getter" method to return a reference to the entry, as in the example above.

Both offset-based and ``constexpr_invocable`` functors are :cpp:concept:`stateless <bds::Stateless>`, and use no storage in the list object (thanks to a ``[[no_unique_address]]`` attribute). However, the user is free to add a stateful functor which can perform arbitrarily complex link access.

.. _lists-how-to-choose-sizetype:

Choosing the SizeType Parameter
-------------------------------
The ``SizeType`` parameter can either be an integral type (in which case, the internal size member will be a field of that type) or it can be the special type ``bds::no_size``, which removes the size member and gives the :math:`O(n)` behavior.

The ``bds::no_size`` behavior is more in-keeping with historical BSD design because typically, the size of most lists will not be greater than the maximum value of ``std::uint32_t``. However, if ``SizeType`` is actually set to ``std::uint32_t``, the user will probably waste 4 bytes of tail padding unless they are careful to pack small members immediately after the list head object. It will not be immediately clear from the code that valuable cache line space might be being wasted, so ``bds::no_size`` is generally preferred.

Because the list's items are typically allocated from a pool allocator, there is usually a factory function which allocates items and places them on lists -- this is a natural place for the user's own size maintenance code.

Understanding the ``head`` vs ``fwd_head + container`` templates
================================================================

A BDS singly-linked list (slist) head can be created in one of two ways:

1. Using the ``slist_head`` class template
2. Using a combination of the ``slist_fwd_head`` and ``slist_container`` class templates

Consider this code from the example above:

.. code-block:: c++

   struct ListItem {
     int i;
     bds::slist_entry link;
   };

   using list_t =
       bds::slist_head<ListItem, slist_entry_offset<offsetof(ListItem, link)>, std::size_t>;

Note that the ``offsetof`` macro requires ``ListItem`` to be a complete type. This means that the ``list_t`` type alias cannot be declared before the ``ListItem`` type is defined, nor can any instance of the list. That is, the following does *not* work:

.. code-block:: c++

   struct ListItem;

   // Error on the next line: offsetof into incomplete type `ListItem`
   using list_t =
       bds::slist_head<ListItem, slist_entry_offset<offsetof(ListItem, link)>, std::size_t>;

   // Definition is below, but it's too late!
   struct ListItem {
     int i;
     bds::slist_entry link;
   };

This restriction did not exist in the original ``queue(3)`` C library. In exchange, the ``queue(3)`` equivalent of the "entry access functor" (the name of the field for the link object) has to be specified in almost *every* macro that accesses the list, e.g., the ``link`` in ``SLIST_INSERT_AFTER(item_before, item, link)``. BDS avoids this by baking the entry access details into the type definition, via a template argument. This is much friendlier API-wise, but requires the type to be complete.

If a list head must be declared before the item type is complete, it can be declared using the corresponding ``_fwd_head`` template, i.e., ``slist_fwd_head``. This is a template only requires the ``SizeType`` template parameter.

To actually use the list, the proxy type ``slist_container`` is constructed, taking a reference to the ``slist_fwd_head``, and imbuing it with the ``T`` and ``EntryAccess`` template parameters.

A code example illustrates this:

.. code-block:: c++

   struct ListItem; // Note: not complete yet

   struct S {
     bds::stailq_fwd_head<std::size_t> allItemsFwd; // Collection of ListItems
   };

   struct ListItem {
     int i;
     bds::slist_entry link;
   };

   // Now that ListItem is a complete type, define `list_t` as an `slist_container`
   // instead of `slist_head`.
   using list_t =
       bds::slist_container<ListItem, slist_entry_offset<offsetof(ListItem, link)>, std::size_t>;

   void printItems(S &s) {
     // Construct a list_t proxy container, passing in the head object.
     list_t allItems{s.allItemsFwd};

     for (const ListItem &item : allItems)
       std::cout << "This is item #" << item.i << std::endl;
   }

The "container" types are extremely cheap to construct (for stateless entry accessors, they only capture a reference to "forward head") and should be completely elided at higher optimization levels. They are also implicitly constructible from the corresponding ``fwd_head`` type, so the user can often avoid explicitly constructing them, e.g.:

.. code-block:: c++

   void printItems(const list_t &allItems);

   void foo(S &s) {
     printItems(s.allItemsFwd);
   }

The (somewhat awkward) need to construct a proxy object each time is roughly analogous to the need to specify the link member each time in the old ``queue(3)`` macros. Thus, when you don't need the flexibility of the ``_container`` variant, prefer the ``_head`` version.

Understand the BDS list concepts
================================

Most of the implementation of an slist is contained in the class template ``slist_base``. Both ``slist_head`` and ``slist_container`` inherit the core slist API from this (`CRTP <https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern>`_) base class. The only difference between ``slist_head`` and ``slist_container`` is that the former is a bit simpler, but at the expense of requiring complete types.

A function taking any kind of slist could be defined like this:

.. code-block:: c++

   template <typename T, typename EntryAccess, typename SizeType>
   void f(bds::slist_base<T, EntryAccess, SizeType> &anySList);

With C++20 concepts, a better option is available:

.. code-block:: c++

   template <bds::SList ListType>
   void f(ListType &anySList);

The single-argument concepts ``TailQ``, ``STailQ``, and ``SList`` evaluate to true if the given type argument is a BDS list of the appropriate kind. The concept ``SListOrQueue`` can be used to check for either ``STailQ`` or ``SList``, which have similar APIs because they are both singly-linked lists.
