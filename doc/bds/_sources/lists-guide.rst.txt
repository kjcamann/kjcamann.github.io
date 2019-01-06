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
     int i;                           // List item's data
     bds::slist_entry<ListItem> link; // Intrusive link that makes us part of
                                      // a `ListItem` singly-linked list
   };

   // The head of the list is a class template. The second template
   // parameter is a functor that controls how the list locates the
   // intrusive links inside the list items (in this case, via a member
   // offset calculation). Because this template is somewhat verbose,
   // a number of helper macros and type aliases are available; these
   // are explained later in the documentation.
   using list_t = bds::slist_head<
     ListItem,                                       // Type of items in list
     slist_entry_offset<offsetof(ListItem, link)>,   // How to access list links
     std::size_t>;                                   // Type for `size()`

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

.. figure:: images/lists-guide-concept.png
   :alt: BDS slist datastructure diagram
   :target: _images/lists-guide-concept.png

   A BDS list object is little more than a pointer to the intial item; the items are *not* owned by the list, they are only linked into it (click to enlarge).

Compare this with node-based lists, like ``std::forward_list``, which own their elements and store them in "node" containers that are allocated by the list itself. One possible implementation of this design is illustrated below:

.. figure:: images/lists-node-list.png
   :alt: A typical node-based list datastructure
   :target: _images/lists-node-list.png

   A typical node-based list (i.e., a non-intrusive list) has ownership over the elements, which are typically wrapped inside node objects. The lifetime of node objects is explicitly managed by the list itself (click to enlarge).

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

:SizeMember: The C++ standard container ``<list>`` offers :math:`O(1)` access to its size by storing the list size inline and maintaining it during each modifying operation. The C++ standard container ``<forward_list>``, and the BSD ``queue(3)`` library do not track list size -- getting the size is an :math:`O(n)` operation that fully traverses the list to count each item. BDS lists allow the user to choose which behavior they want. See :ref:`lists-how-to-choose-sizetype` for more information.

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
     bds::slist_entry<ListItem> &getLink() noexcept { return link; }

   private:
     bds::slist_entry<ListItem> link;
   };

   using list_t = bds::slist_head<
     ListItem,
     constexpr_invocable<&ListItem::getLink>,
     std::size_t>;

The implementation calls ``std::invoke(<invocable-template-argument>, <item-of-type-T>)`` to access the list entry. The ``auto I`` template argument is usually a member object pointer, a member function pointer, or a free function taking a single ``T&`` argument (see the documentation of `std::invoke <https://cppreference.com/w/cpp/utility/functional/invoke>`_ for the possibilities).

When to use offset-based vs. invocable accessors
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Offset-based accessors usually generate better code, but not all types can use them. The C++ standard only guarantees that ``offsetof`` will work on standard layout types, although since C++17, implementations are free to allow other types. Because of that freedom, BDS does not enforce the standard layout requirement through a template constraint, so the user must "know what they're doing" when using ``offsetof``.

Invocable accessors are more powerful, and can solve two problems:

1. **Access Protection Problems** -- In most implementations, an ``offsetof(T, <field>)`` appearing outside of T and where ``<field>`` is non-public will fail, unless a friend declaration allows access in the scope where ``offsetof`` appears. Invocables can avoid this by using a public "getter" method to return a reference to a private entry, as in the example above.

2. **Self-Referential List Heads** --
   As of C++17, the expression ``offsetof(C, <member-of-C>)`` is usually invalid when it appears inside the definition of ``class C``. For example, this will not work:

   .. code-block:: c++

      class C {
        bds::slist_entry<C> link;
        bds::slist_head<C, bds::slist_entry_offset<offsetof(C, link)>> children; // Error: C isn't complete
      };

   As far as the author is aware, there is nothing in the C standard that says this cannot work, but none of the popular implementations permit ``offsetof`` inside a type that is still being defined. However, using a pointer-to-member invocable here *will* work, even before the type is complete:

   .. code-block:: c++

      class C {
        bds::slist_entry<C> link;
        bds::slist_head<C, bds::constexpr_invocable<&C::link>> children; // OK
      };

   This may seem like an odd use case, but there are several "tree-like" structures like this in the FreeBSD kernel. As one example, a ``vm_object`` contains a ``LIST_HEAD`` to its shadow ``vm_object`` children, as part of the virtual memory copy-on-write feature. There is still one restriction: the expression ``&C::link`` must appear after the ``bds::slist_entry_link<C>`` member. If the user cannot do this, the :ref:`proxy <lists-head-vs-fwd-head>` pattern can be used instead.

Both offset-based and ``constexpr_invocable`` functors are :cpp:concept:`stateless <bds::Stateless>`, and use no storage in the list object (thanks to a ``[[no_unique_address]]`` attribute). However, the user is free to use a stateful functor which can perform arbitrarily complex link access.

Syntactic sugar for declaring lists
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Because the list head templates are so verbose, helper alias templates and macros are available. Ideally we would not use any macros, but code patterns involving ``offsetof`` typically cannot be expressed without resorting to macros.

For offset-based accessors:

   .. code-block:: c++

      struct ListItem {
        int i;
        bds::slist_entry<ListItem> link;
      };

      // Instead of this...
      using my_list_t = bds::slist_head<
          ListItem,
          bds::slist_entry_offset<offsetof(ListItem, link)>>;

      // ...write this:
      using my_list_t = BDS_TAILQ_HEAD_OFFSET_T(ListItem, link);

For most constexpr-invocable accessors, we can use a helper type alias:

   .. code-block:: c++

      struct ListItem {
        int i;
        bds::slist_entry<ListItem> link;
      };

      // Instead of this...
      using my_list_t = bds::slist_head<
          ListItem,
          bds::constexpr_invocable<&ListItem::link>>;

      // ...write this:
      using my_list_t = bds::tailq_head_cinvoke_t<&ListItem::link>;

Ideally, this helper alias would work with all invocables, not just constexpr ones. Unfortunately as of C++20, there is no "invocation traits" class that would allow us to discover the type of the first argument to the invocable, no matter what form it has. As of C++17, *constexpr* invocables can take a limited number of forms (free functions, member function pointers, etc.) and a traits class is provided that partially specializes all known syntactic forms and exposes the argument type. This won't cover the additional constexpr invocables that were added in C++20 unfortunately (constexpr non-type arguments, e.g., a constexpr lambda entry accessor). For cases that are not covered, the user must resort to the verbose declaration.

.. sidebar:: A final word about offset accessors

   Earlier we said that ``struct slist_entry_offset`` was a functor template. That was true in earlier releases of BDS, but it is no longer true. ``slist_entry_offset`` is not a functor (it is not callable), but is just syntactic sugar that simplifies writing a more complex template pattern. The verbose way to express the template instance would be:

   .. code-block:: c++

      bds::slist_head<
        ListItem,
        offset_extractor<slist_entry<ListItem>, offsetof(ListItem, link)>>

   ``offset_extractor`` is the actual functor, and it needs to accept the entry type (``slist_entry<ListItem>``) as a template argument. The entry type can be easily inferred however, so the helper template ``struct slist_entry_offset`` is used instead; it is rewritten to the above pattern using a traits class.

.. _lists-how-to-choose-sizetype:

Choosing the SizeMember Parameter
---------------------------------
The ``SizeMember`` parameter can either be an integral type (in which case, the internal size member will be a field of that type) or it can be the special type ``bds::no_size``, which removes the size member and gives the :math:`O(n)` behavior.

The ``bds::no_size`` option is more in-keeping with historical BSD design because typically, the size of most lists will not be greater than the maximum value of ``std::uint32_t``. However, if ``SizeMember`` is actually set to ``std::uint32_t``, the user will probably waste 4 bytes of tail padding unless they are careful to pack small members immediately after the list head object. It will not be immediately clear from the code that valuable cache line space might be being wasted, so ``bds::no_size`` is generally preferred. For this reason, ``no_size`` is also the default template argument for ``SizeMember``, hence why it is unspecified in most of the later examples.

Because the list's items are typically allocated from a pool allocator, there is usually a factory function which allocates items and places them on lists -- this is a natural place for the user's own size maintenance code.

.. _lists-head-vs-fwd-head:

Understanding the ``head`` vs ``fwd_head + proxy`` templates
================================================================

Suppose you want to create a BDS singly-linked list (slist) head. You have two options:

1. Use the ``slist_head`` class template
2. Use a combination of the ``slist_fwd_head`` and ``slist_proxy`` class templates

Consider this code from the example above:

.. code-block:: c++

   struct ListItem {
     int i;
     bds::slist_entry<ListItem> link;
   };

   using list_t = bds::slist_head<
     ListItem,
     slist_entry_offset<offsetof(ListItem, link)>,
     std::size_t>;

Note that the ``offsetof`` macro requires ``ListItem`` to be a complete type. This means that the ``list_t`` type alias cannot be declared before the ``ListItem`` type is defined, nor can any instance of the list. That is, the following does *not* work:

.. code-block:: c++

   struct ListItem;

   using list_t = bds::slist_head<
     ListItem,
     slist_entry_offset<offsetof(ListItem, link)>, // Error: offsetof into incomplete type
     std::size_t>;

   // Definition is below, but it's too late!
   struct ListItem {
     int i;
     bds::slist_entry<ListItem> link;
   };

This restriction did not exist in the original ``queue(3)`` C library. In exchange, the ``queue(3)`` equivalent of the "entry access functor" (the name of the field for the "entry" object) has to be specified in almost *every* macro that accesses the list, e.g., the ``link`` in ``SLIST_INSERT_AFTER(item_before, item, link)``. BDS avoids this by baking the entry access details into the type definition, via a template argument. This is much friendlier API-wise, but requires the type to be complete.

If a list head must be declared before the item type is complete, it can be declared using the corresponding ``fwd_head`` ("forward head") template, e.g., ``slist_fwd_head``. This template only requires the ``T`` and ``SizeMember`` template parameters. Note that although ``T`` is a required parameter, it can be an incomplete.

To actually use the list, the proxy type ``slist_proxy`` is constructed, taking a reference to the ``slist_fwd_head``, and imbuing it with the ``EntryAccess`` template parameter.

The following example illustrates this:

.. code-block:: c++

   struct ListItem; // Note: not complete yet

   struct S {
     bds::stailq_fwd_head<ListItem, std::size_t> allItemsFwd; // Collection of ListItems
   };

   struct ListItem {
     int i;
     bds::slist_entry<ListItem> link;
   };

   // Now that ListItem is a complete type, define `list_proxy_t` as an
   // `slist_proxy` instead of `slist_head`.
   using list_proxy_t = bds::slist_proxy<
       stailq_fwd_head<ListItem, std::size_t>,        // Type of the fwd head
       slist_entry_offset<offsetof(ListItem, link)>>; // Late-specified EntryAccess

   void printItems(S &s) {
     // Construct a list_t proxy container, passing in the head object.
     list_proxy_t allItems{s.allItemsFwd};

     for (const ListItem &item : allItems)
       std::cout << "This is item #" << item.i << std::endl;
   }

The "proxy" types are extremely cheap to construct (for stateless entry accessors, they only capture a reference to the "forward head") and should be completely elided at higher optimization levels. They are also implicitly constructible from the corresponding ``fwd_head`` type, so the user can usually avoid explicitly constructing them, e.g.:

.. code-block:: c++

   void printItems(const list_proxy_t &allItems);

   void foo(S &s) {
     printItems(s.allItemsFwd); // OK: calls the list_proxy_t converting constructor
   }

The ``fwd_head`` and ``proxy`` classes separate the list's *data storage* (in the ``fwd_head``) from the list's *behavior* (in the ``proxy``), which allows list storage to be declared before the item type is completed [#lists-proxy-implies-stateless]_. A ``fwd_head`` and ``proxy`` pair are somewhat like a `std::mutex <https://en.cppreference.com/w/cpp/thread/mutex>`_ and a `std::unique_lock <https://en.cppreference.com/w/cpp/thread/unique_lock>`_: the former generally stores the state of the object, and the latter is "constructed around it," as a means of operating on it.

The (somewhat awkward) need to construct a proxy object each time is roughly analogous to the need to specify the linkage member each time in the old ``queue(3)`` macros. Thus, when you don't need the flexibility of the ``fwd_head + proxy`` pattern, prefer the "all-in-one" alternative, e.g., ``slist_head``.

As with the "head" versions, helper macros and alias templates are available that make proxy type declarations less verbose:

.. code-block:: c++

   struct ListItem {
     int i;
     bds::slist_entry<ListItem> link;
   };

   using mylist_proxy_t = BDS_TAILQ_PROXY_OFFSET_T(ListItem, link);

   // or when using std::invoke:

   using mylist_proxy_t = bds::tailq_proxy_cinvoke_t<&ListItem::link>;

Are ``fwd_head`` instances movable?
-----------------------------------

For most BDS intrusive containers, the "forward head" types are `moveable <https://en.cppreference.com/w/cpp/concepts/Movable>`_, to permit code like this: [#lists-careful-moving]_

.. code-block:: c++

   struct ListItem;

   struct T {
     T() = default;
     T(T &&) = default; // OK, `fwdItems` is movable

     bds::slist_fwd_head<ListItem> fwdItems;
   };

Unfortunately this cannot work for certain data structures, namely ``tailq_fwd_head``:

.. code-block:: c++

   struct ListItem;

   struct T {
     T() = default;
     T(T &&) = default; // Error: tailq_fwd_head's move constructor is deleted

     bds::tailq_fwd_head<ListItem> fwdItems;
   };

This is a consequence of a :ref:`tailq implementation detail <lists-tailq-circular-design>`, namely that tailq's are circular lists with the ``end()`` sentinel connecting the head to the tail.

Because the head and tail list items must point to the address of the ``tailq_fwd_head`` instance (which plays the same role as ``tailq_head`` in :ref:`this diagram <lists-tailq-invocable-figure>`), the move logic needs to access the ``EntryAccess`` functor so it can edit the head and tail entries, namely to rewrite their linkage into the sentinel node. In short, a ``tailq_fwd_head`` by itself lacks the information needed to move the list; we must construct proxies to help move it.

To work around this limitation, the user must define a custom move constructor:

.. code-block:: c++

   struct ListItem;

   struct T {
     T() = default;
     T(T &&) noexcept;

     bds::tailq_fwd_head<ListItem> fwdItems;
   };

   struct ListItem {
     int i;
     bds::tailq_entry link;
   };

   using list_proxy_t = bds::tailq_proxy<tailq_fwd_head<ListItem>,
       bds::tailq_entry_offset<offsetof(ListItem, link)>>;

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

Understand the BDS list concepts
================================

Most of the implementation of an slist is contained in the class template ``slist_base``. Both ``slist_head`` and ``slist_proxy`` inherit the core slist API from this (`CRTP <https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern>`_) base class. The only difference between ``slist_head`` and ``slist_proxy`` is that the former is more "elegant," at the expense of requiring complete types.

A function taking any kind of slist could be defined like this:

.. code-block:: c++

   template <typename T, typename EntryAccess, typename SizeMember>
   void f(bds::slist_base<T, EntryAccess, SizeMember> &anySList);

With C++20 concepts, a better option is available:

.. code-block:: c++

   template <bds::SList ListType>
   void f(ListType &anySList);

The single-argument concepts ``TailQ``, ``STailQ``, and ``SList`` evaluate to true if the given type argument is a BDS list of the appropriate kind. The concept ``SListOrQueue`` can be used to check for either ``STailQ`` or ``SList``, which have similar APIs because they are both singly-linked lists. All BDS lists satisfy the ``LinkedList`` concept.

.. rubric:: Footnotes

.. [#lists-careful-moving] The user must be careful when using movable ``fwd_head`` instances. Any `proxy`` instances still attached to the forward head will now affect the moved-from head. The effects of doing this are undefined.

.. [#lists-proxy-implies-stateless] In :ref:`lists-how-to-choose-entryaccess` we note that the user is free to use a stateful entry access functor. These can be difficult to work with when using the ``fwd_head + proxy`` pattern, because the accessor state must be passed to each ``proxy`` instance (``proxy`` instances are ephemeral wrapper types with short life-spans). For entry access functors which contain move-only types (e.g., a ``unique_ptr``) it should still be possible, but may not be worth the effort.
