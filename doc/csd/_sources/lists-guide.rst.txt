***********************
Lists Quick Start Guide
***********************

.. contents::
   :local:

What are CSD lists?
===================

CSD lists are intrusive linked lists based on the `BSD queue(3) library <https://man.openbsd.org/queue.3>`_. That library first appeared in 4.4 BSD, and is available today on most UNIX systems via the header file ``<sys/queue.h>`` (it is also included in glibc and can be found on most Linux systems). Linked lists are used to implement queues in various UNIX kernels, thus the name "queue."

The original queue library has been copied into or reimplemented in many well-known C code bases, including the Linux kernel. Like most data structure libraries written in C, it relies on macros to achieve static polymorphism. CSD lists are C++20 re-implementations that combine the data structure design from BSD queues with the design of STL containers. For example, it achieves static polymorphism using templates instead of macros.

To learn why intrusive linked lists are great data structures, read :doc:`lists-rationale`

CSD provides the four data structures from ``queue(3)``:

:slist: Singly-linked lists

:stailq: Singly-linked tail queues; a "tail queue" is a list that keeps track of the last element, which allows fast insertion at the end of the list and fast list concatenation

:tailq: Doubly-linked tail queues, i.e., doubly-linked lists that also have the tail queue features

:list:
   .. warning:: list is deprecated and should be avoided.

   In the original ``queue(3)`` library, there is a difference between a ``LIST`` and a ``TAILQ``, but in CSD there is not. To meet the design goal of matching the STL, the C++ implementation of lists and tailqs must be exactly the same (see :ref:`here <lists-diff-with-queue>` for a detailed explanation). If ``<csg/core/list.h>`` is included, it makes lists available as tailq type aliases, but adds deprecation attributes.

A simple example
================

.. code-block:: c++
   :name: lists-main-example

   #include <iostream>
   #include <csg/core/slist.h>

   using namespace csg;

   struct ListItem {
     int i;                        // List item's data
     slist_entry<ListItem> link;   // Intrusive link that makes us part of
                                   // a `ListItem` singly-linked list
   };

   // The head of the list is a class template. The second template
   // parameter is a functor that controls how the list locates the
   // intrusive links inside the list items (in this case, via using
   // a pointer-to-data-member access). Because this template is somewhat
   // verbose, a number of helper macros and type aliases are available;
   // these are explained later in the documentation.
   using list_t = slist_head<
     ListItem,                               // Type of items in list
     invocable_constant<&ListItem::link>,    // How to access list links
     std::size_t>;                           // Type for `size()`

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

There is deliberately very little doxygen documentation for CSD list member functions. They are meant to follow the standard C++ list APIs as much as possible, so you should use your favorite C++ reference (e.g., `cppreference <https://cppreference.com>`_) for `\<list\> <https://cppreference.com/w/cpp/container/list>`_ when referencing tailq and `\<forward_list\> <https://cppreference.com/w/cpp/container/forward_list>`_ when referencing slist or stailq.

However, because CSD lists are intrusive and because they follow the BSD data structure design patterns, there are some differences in the public API compared to standard C++ containers. The fundamental difference is that CSD lists -- like all intrusive data structures -- do not own their elements. A "list" consists of a small "head" object which points to the first (and possibly last) item in the list. The items themselves have a lifetime distinct from the list head's lifetime and are allocated elsewhere, usually in a memory pool. An illustration of a CSD ``slist`` is shown below:

.. figure:: images/lists-guide-concept.png
   :alt: CSD slist datastructure diagram
   :target: _images/lists-guide-concept.png

   A CSD list object is little more than a pointer to the intial item; the items are *not* owned by the list, they are only linked into it (click to enlarge).

Compare this with node-based lists, like ``std::forward_list``, which own their elements and store them in "node" containers that are allocated by the list itself. One possible implementation of this design is illustrated below:

.. figure:: images/lists-node-list.png
   :alt: A typical node-based list datastructure
   :target: _images/lists-node-list.png

   A typical node-based list (i.e., a non-intrusive list) has ownership over the elements, which are typically wrapped inside node objects. The lifetime of node objects is explicitly managed by the list itself (click to enlarge).

This fundamental difference gives rise to a variety of smaller API differences, be sure to read the section :ref:`lists-diff-with-stl` for all the details. In short, the differences are:

* CSD lists cannot be copied.
* List items are neither copied nor moved into the list -- they are just linked into it (their intrusive link member variable is modified to splice the item into the list).
* As a consequence of the above, list items are always inserted *by pointer*, i.e., for a list of type ``T``, the insert family of functions require a ``T*`` value. To avoid breaking with the standard containers too much, all *non-insertion* member functions continue to use ``T&``, e.g., ``front()`` and ``back()`` continue to return a reference to the item, not a pointer to it.
* A list item cannot be added to multiple lists that use the same entry link.
* The list destructor doesn't actually destroy the items (because it does not own them).

CSD lists also provide some useful APIs which are not in the standard containers -- see :ref:`lists-extra-methods`.

Choosing the template parameters
================================

As can be seen in the :ref:`example above <lists-main-example>` ``slist_head`` has three template parameters:

:T: The type of the items in the list.

:EntryEx: The type of function object that extracts the intrusive link, given an instance of ``T``. In the example above, the intrusive link is the data member ``csg::slist_entry<ListItem> link``. The link object is called an "entry" from the name of the corresponding ``queue(3)`` macros, e.g., ``SLIST_ENTRY``. See :ref:`lists-how-to-choose-extractor` for more information.

:SizeMember: The C++ standard container ``<list>`` offers :math:`O(1)` access to its size by storing the list size inline and maintaining it during each modifying operation. The C++ standard container ``<forward_list>``, and the BSD ``queue(3)`` library do not track list size -- getting the size is an :math:`O(n)` operation that fully traverses the list to count each item. CSD lists allow the user to choose which behavior they want. See :ref:`lists-how-to-choose-sizetype` for more information.

.. _lists-how-to-choose-extractor:

Choosing the EntryEx parameter
------------------------------

All intrusive data structures in CSD are parameterized by a template parameter called an *extractor* -- in this case an **Entry Ex**\ tractor -- that knows how to extract the intrusive book-keeping object from the user's type.

Extractors are covered in detail in :doc:`intrusive-extractors`. That document explains the rationale for extractors and gives advice on how to define them. Below we cover the most common "syntactic sugar" way to declare the two main types of extractors: ``offsetof``-based and ``invocable_constant``-based.

Syntactic sugar for declaring extractors
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Because the list head templates are so verbose -- mainly because of the extractor parameter -- CSD provides helper alias templates and macros. Ideally we would not use any macros, but code patterns involving ``offsetof`` typically cannot be expressed without resorting to macros.

For offsetof-based extractors:

.. code-block:: c++

   struct ListItem {
     int i;
     slist_entry<ListItem> link;
   };

   // Instead of this...
   using my_list_t = slist_head<
       ListItem,
       offset_extractor<slist_entry<ListItem>, offsetof(ListItem, link)>>;

   // ...write this:
   using my_list_t = CSG_TAILQ_HEAD_OFFSET_T(ListItem, link);

For most constexpr-invocable extractors -- usually pointer-to-member based -- we can use a helper type alias:

.. code-block:: c++

   struct ListItem {
     int i;
     slist_entry<ListItem> link;
   };

   // Instead of this...
   using my_list_t = slist_head<
       ListItem,
       invocable_constant<&ListItem::link>>;

   // ...write this:
   using my_list_t = slist_head_cinvoke_t<&ListItem::link>;

Ideally, this helper alias would work with all invocables, not just constexpr ones. Unfortunately as of C++20, there is no "invocation traits" class that would allow us to discover the type of the first argument to the invocable, no matter what form it has. As of C++17, *constexpr* invocables can take a limited number of forms (free functions, member function pointers, etc.) and a traits class is provided that partially specializes all known syntactic forms and exposes the argument type. This won't cover the additional constexpr invocables that were added in C++20 unfortunately (constexpr non-type arguments, e.g., a constexpr lambda extractor). For cases that are not covered, the user must resort to the verbose declaration.

.. _lists-how-to-choose-sizetype:

Choosing the SizeMember parameter
---------------------------------
The ``SizeMember`` parameter can either be an integral type (in which case, the internal size member will be a field of that type) or it can be the special type ``csg::no_size``, which removes the size member and gives the :math:`O(n)` behavior.

The ``csg::no_size`` option is more in-keeping with historical BSD design because typically, the size of most lists will not be greater than the maximum value of ``std::uint32_t``. However, if ``SizeMember`` is actually set to ``std::uint32_t``, the user will probably waste 4 bytes of tail padding unless they are careful to pack small members directly after the list head object. It will not be immediately clear from the code that valuable cache line space might be being wasted, so ``csg::no_size`` is generally preferred. For this reason, ``no_size`` is also the default template argument for ``SizeMember``, hence why it is unspecified in most of the later examples.

Because the list's items are typically allocated from a pool allocator, there is usually a factory function which allocates items and places them on lists -- this is a natural place for the user's own size maintenance code.

.. _lists-head-vs-fwd-head:

Understanding the ``head`` vs ``fwd_head + proxy`` templates
============================================================

Suppose you want to create a CSD singly-linked list (slist) head. You have two options:

1. Use the ``slist_head`` class template
2. Use a combination of the ``slist_fwd_head`` and ``slist_proxy`` class templates

Consider this code from the example above:

.. code-block:: c++

   struct ListItem {
     int i;
     slist_entry<ListItem> link;
   };

   using list_t = slist_head<
     ListItem,
     invocable_constant<&ListItem::link>,
     std::size_t>;

Note that the expression ``&ListItem::link`` requires ``ListItem`` to be a complete type. [#lists-complete-type]_ This means that the ``list_t`` type alias cannot be declared before the ``ListItem`` type is defined, nor can any instance of the list. That is, the following does *not* work:

.. code-block:: c++

   struct ListItem;

   using list_t = slist_head<
     ListItem,
     invocable_constant<&ListItem::link>, // Error: incomplete type in nested-name specifier
     std::size_t>;

   // Definition is below, but it's too late!
   struct ListItem {
     int i;
     slist_entry<ListItem> link;
   };

This restriction did not exist in the original ``queue(3)`` C library. In exchange, the ``queue(3)`` equivalent of the "entry extractor" (the name of the field for the "entry" object) has to be specified in almost *every* macro that accesses the list, e.g., the ``link`` field name would need to be specified when performing an insert: ``SLIST_INSERT_AFTER(item_before, item, link)``. CSD avoids this by baking the entry extractor details into the type definition, via a template argument. This is much friendlier API-wise, but requires the type to be complete.

If a list head must be declared before the item type is complete, it can be declared using the corresponding ``fwd_head`` ("forward head") template, e.g., ``slist_fwd_head``. This template only requires the ``T`` and ``SizeMember`` template parameters. Note that although ``T`` is a required parameter, it can be an incomplete.

To actually use the list, the proxy type ``slist_proxy`` is constructed, taking a reference to the ``slist_fwd_head``, and imbuing it with the ``EntryEx`` template parameter.

The following example illustrates this:

.. code-block:: c++

   struct ListItem; // Note: not complete yet

   struct S {
     stailq_fwd_head<ListItem, std::size_t> allItemsFwd; // Collection of ListItems
   };

   struct ListItem {
     int i;
     slist_entry<ListItem> link;
   };

   // Now that ListItem is a complete type, define `list_proxy_t` as an
   // `slist_proxy` instead of `slist_head`.
   using list_proxy_t = slist_proxy<
       stailq_fwd_head<ListItem, std::size_t>, // Type of the fwd head
       invocable_constant<&ListItem::link>>;   // Late-specified EntryEx

   void printItems(S &s) {
     // Construct a list_t proxy container, passing in the head object.
     list_proxy_t allItems{s.allItemsFwd};

     for (const ListItem &item : allItems)
       std::cout << "This is item #" << item.i << std::endl;
   }

As usual, a helper type alias makes the verbose template type easier to declare:

.. code-block:: c++

   using list_proxy_t = slist_proxy_cinvoke_t<&ListItem::link, std::size_t>;

The "proxy" types are extremely cheap to construct (for stateless entry extractors, they only capture a reference to the "forward head") and should be completely elided at higher optimization levels. They are also implicitly constructible from the corresponding ``fwd_head`` type, so the user can usually avoid explicitly constructing them, e.g.:

.. code-block:: c++

   void printItems(const list_proxy_t &allItems);

   void foo(S &s) {
     printItems(s.allItemsFwd); // OK: calls the list_proxy_t converting constructor
   }

The ``fwd_head`` and ``proxy`` classes separate the list's *data storage* (in the ``fwd_head``) from the list's *behavior* (in the ``proxy``), which allows list storage to be declared before the item type is completed [#lists-proxy-implies-stateless]_. A ``fwd_head`` and ``proxy`` pair are somewhat like a `std::mutex <https://en.cppreference.com/w/cpp/thread/mutex>`_ and a `std::unique_lock <https://en.cppreference.com/w/cpp/thread/unique_lock>`_: the former generally stores the state of the object, and the latter is "constructed around it," as a means of operating on it.

The (somewhat awkward) need to construct a proxy object each time is roughly analogous to the (somewhat awkward) need to specify the linkage member each time in the old ``queue(3)`` macros. Thus, when you don't need the flexibility of the ``fwd_head + proxy`` pattern, prefer the "all-in-one" alternative, e.g., ``slist_head``.

As with the "head" versions, a helper macro is available to declare proxy types that use ``offsetof``-based extractors:

.. code-block:: c++

   struct ListItem {
     int i;
     slist_entry<ListItem> link;
   };

   using mylist_proxy_t = CSG_TAILQ_PROXY_OFFSET_T(ListItem, link);

Are ``fwd_head`` instances movable?
-----------------------------------

For most CSD intrusive containers, the "forward head" types are `moveable <https://en.cppreference.com/w/cpp/concepts/Movable>`_, to permit code like this: [#lists-careful-moving]_

.. code-block:: c++

   struct ListItem;

   struct T {
     T() = default;
     T(T &&) = default; // OK, `fwdItems` is movable

     slist_fwd_head<ListItem> fwdItems;
   };

Unfortunately this cannot work for certain data structures, namely ``tailq_fwd_head``:

.. code-block:: c++

   struct ListItem;

   struct T {
     T() = default;
     T(T &&) = default; // Error: tailq_fwd_head's move constructor is deleted

     tailq_fwd_head<ListItem> fwdItems;
   };

This is a consequence of a :ref:`tailq implementation detail <lists-tailq-circular-design>`, namely that tailq's are circular lists with the ``end()`` sentinel connecting the head to the tail.

Because the head and tail list items must point to the address of the ``tailq_fwd_head`` instance (which plays the same role as ``tailq_head`` in :ref:`this diagram <lists-tailq-invocable-figure>`), the move logic needs to access the ``EntryEx`` extractor so it can edit the head and tail entries, namely to rewrite their linkage into the sentinel node. In short, a ``tailq_fwd_head`` by itself lacks the information needed to move the list; we must construct proxies to help move it.

To work around this limitation, the user must define a custom move constructor:

.. code-block:: c++

   struct ListItem;

   struct T {
     T() = default;
     T(T &&) noexcept;

     tailq_fwd_head<ListItem> fwdItems;
   };

   struct ListItem {
     int i;
     tailq_entry link;
   };

   using list_proxy_t = tailq_proxy<tailq_fwd_head<ListItem>,
       tailq_entry_offset<offsetof(ListItem, link)>>;

   T::T(T &&other) noexcept {
     list_proxy_t ourItems{this->fwdItems, list_proxy_t{other.fwdItems}};
   }

   void f() {
     S s[] = { {.i=1}, {.i=2} };
     T t1;

     // Construct a tailq container proxy so we can insert items into t1's list.
     list_proxy_t items1{t1.fwdItems, {&s[0]. &s[1]}};

     T t2{std::move(t1)}; // Invoke our custom move logic

     assert(std::size(items1) == 0 && std::size(list_t{t2.fwdItems}) == 2); // OK
   }

Understand the CSD list concepts
================================

Most of the implementation of an slist is contained in the class template ``slist_base``. Both ``slist_head`` and ``slist_proxy`` inherit the core slist API from this base class using the `CRTP <https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern>`_. The only difference between ``slist_head`` and ``slist_proxy`` is that the former is more "elegant," at the expense of requiring complete types.

A function taking any kind of slist can be defined like this:

.. code-block:: c++

   template <csg::slist ListType>
   void f(ListType &anySList);

The single-argument concepts ``tailq``, ``stailq``, and ``slist`` evaluate to true if the given type argument is a CSD list of the appropriate kind. The concept ``singly_linked_list`` can be used to check for either ``stailq`` or ``slist``, which have similar APIs because they are both singly-linked lists. All CSD lists satisfy the ``linked_list`` concept.

.. rubric:: Footnotes

.. [#lists-complete-type] As explained in :doc:`intrusive-extractors`, ``&ListItem::link`` may also appear inside the body of ``ListItem`` while it is still being defined, as long as the field declaration has already occurred.

.. [#lists-careful-moving] The user must be careful when using movable ``fwd_head`` instances. Any `proxy`` instances still attached to the forward head will now affect the moved-from head. The effects of doing this are undefined.

.. [#lists-proxy-implies-stateless] In :ref:`lists-how-to-choose-extractor` we note that the user is free to use a stateful entry extractor. These can be difficult to work with when using the ``fwd_head + proxy`` pattern, because the extractor state must be passed to each ``proxy`` instance (``proxy`` instances are ephemeral wrapper types with short life-spans). For entry extractors which contain move-only types (e.g., a ``unique_ptr``) it should still be possible, but may not be worth the effort.
