**************************************
Extractors: the Core Intrusive Concept
**************************************

CSD provides several intrusive containers (:doc:`lists <lists-main>`, hash tables (TODO: link)) and an intrusive smart pointer (TODO: link). To insert an item into an intrusive container, or to use the intrusive smart pointer, a class must provide storage for the intrusive library code to store its internal data. For example, an item inserted into an intrusive list must provide storage for a pointer to the next item.

.. figure:: images/lists-extractor-concept.png
   :alt: CSD intrusive objects must be embedded into user objects
   :target: _images/lists-extractor-concept.png

   A typical CSD intrusive data structure; the user must embed a CSD object (in this case, ``slist_entry``) into their own code, for the library to use. *Extractors* are the concept that enable CSD to find the embedded object.

How does the intrusive library code -- located in CSD -- know how to find the storage inside the user's type, defined in the user's own code? CSD uses two approaches: a more generic approach is used by the containers, and a simpler one is used by the reference-counted smart pointer. In both cases, *extractors* are the fundamental concept. At the source-code level, an "extractor" is just a tiny C++ ``concept`` similar to `std::invocable <https://en.cppreference.com/w/cpp/concepts/invocable>`_ but with an additional guarantee that the result can bind to a particular reference type:

.. code-block:: c++

   template <typename F, typename R, typename T>
   concept extractor = std::invocable<F, T&> &&
       std::common_reference_with<std::invoke_result_t<F, T&>,
                                  std::add_lvalue_reference_t<R>>;

Despite this simplicity, usage of ``csg::extractor`` in code is meant to convey a deeper semantic meaning that is explained in the rest of this section.

Extractor basics
================

As shown in the picture above, intrusive containers are just small "head" objects containing links to the container items. The head object is always a template, and one of the template parameters is an `std::invocable <https://en.cppreference.com/w/cpp/concepts/invocable>`_ type, which knows how to extract the intrusive payload from the user's type.

Consider the case of a singly-linked list:

.. code-block:: c++
   :name: first-intrusive-example

   #include <csg/core/slist.h>

   struct ListItem {
     int i;                            // List item's data
     csg::slist_entry<ListItem> link;  // Intrusive link that makes us part of
                                       // a `ListItem` singly-linked list
   };

   // Define a functor which knows how to extract the link member
   // from the `ListItem` type.
   struct GetLink {
     csg::slist_entry<ListItem> &operator()(const ListItem &i) const noexcept {
       return i.link;
     }
   };

   using list_t = csg::slist_head<ListItem, GetLink>;

Throughout the CSD documentation, an invocable like ``GetLink`` is referred to as an *extractor*, a term borrowed from the `Boost Multi-index library <https://www.boost.org/doc/libs/1_70_0/libs/multi_index/doc/index.html>`_.

In the case of the containers, it is typically called an "entry extractor," because the object it extracts has "_entry" in the name. For example, the ``GetLink`` above extracts a ``csg::slist_entry<ListItem>``. The term "entry" comes from the original `queue(3) <http://man.openbsd.org/queue>`_ C intrusive library from BSD; it is the object that stores the pointers used to link an object into a list, thereby making it an entry in that list.

It is both annoying and unnecessary to write a boilerplate functor like ``GetLink``. Because a pointer-to-data-member can be used with `std::invoke <https://cppreference.com/w/cpp/utility/functional/invoke>`_, we would ideally like to write something like this:

.. code-block:: c++

   using list_t = csg::slist_head<ListItem, &ListItem::link>;

Unforutantely we cannot do this directly.

In C++, a template parameter must be either a *type* parameter, or a *non-type* parameter (i.e., a compile-time value). To make the library as generic as possible, the extractor parameter of ``csg::slist_head`` is a *type*, whereas ``&ListItem::link`` is a constexpr *value*.

To deal with this, a helper template called ``invocable_constant`` is used to wrap a constexpr invocable in a function-like type:

.. code-block:: c++

   template <auto I>
   struct invocable_constant {
     constexpr static auto value = I;

     template <typename... Ts> requires std::invocable<decltype(I), Ts...>
     constexpr decltype(auto) operator()(Ts &&... vs) const noexcept(/*details*/) {
       return std::invoke(I, std::forward<Ts>(vs)...);
     }
   };

The name ``invocable_constant`` is similar to the standard library classes, `integral_constant and bool_constant <https://en.cppreference.com/w/cpp/types/integral_constant>`_, which wrap constexpr values into types, so they can be used as template type arguments.

Now we can define ``list_t`` as:

.. code-block:: c++

   using list_t = csg::slist_head<ListItem, invocable_constant<&ListItem::link>>;

Because this pattern is common -- and because the above is still too verbose -- a helper type alias is defined:

.. code-block:: c++

   using list_t = csg::slist_head_cinvoke_t<&ListItem::link>;

Each container type defines a similar type alias, always ending in "_cinvoke_t", to help the user refer to a **c**\ onstexpr **invoke**\ able **t**\ ype that wraps a constexpr value like the pointer-to-member above.

Although the user can provide any `std::invocable <https://en.cppreference.com/w/cpp/concepts/invocable>`_ type (including a stateful one), most of the time the user will want a member object pointer, a member function pointer, or a free function taking a single ``T&`` argument (see the documentation of `std::invoke <https://cppreference.com/w/cpp/utility/functional/invoke>`_ for the possibilities). Because references to particular free functions or members are compile-time constants, the "_cinvoke_t" helpers are the most common way to declare a container type in CSD.

``offsetof``-based Extractors
-----------------------------

Another important class of extractors are the *offset extractors*: they extract an intrusive payload using the ``offsetof`` macro and ``std::bit_cast``. They are specializations of the template functor ``offset_extractor``:

.. code-block:: c++

   template <typename R, std::size_t Offset>
   struct offset_extractor {
     constexpr static std::size_t offset = Offset;

     template <typename T>
     constexpr std::remove_cvref_t<R> &operator()(T &t) const noexcept {
       constexpr auto *const baseAddr = std::bit_cast<std::byte *>(std::addressof(t));
       return *std::bit_cast<R *>(baseAddr + Offset);
     }
   };

Like ``invocable_constant``, manual use of ``offset_extractor`` is cumbersome. Unfortunately we cannot write a helper alias like ``slist_head_cinvoke_t`` that simplifies it, because of limitations in the C++ language.

Instead, we use a helper *macro*:

.. code-block:: c++

   using list_t = CSG_SLIST_HEAD_OFFSET_T(ListItem, link);

Each container type defines a similar macro, always ending in "_OFFSET_T", to help the user work with offset extractors. The extractor type itself can be referred to using the macro ``CSG_OFFSET_EXTRACTOR(Class, Field)``. For example, the above definition of ``list_t`` is shorthand for:

.. code-block:: c++

   using list_t = csg::slist_head<ListItem, CSG_OFFSET_EXTRACTOR(ListItem, link)>;

.. note::

   In both the documentation and the code, extractors are referred to as being either "offset extractors" or "invocable extractors." Yet you may notice that ``offset_extractor`` is an invocable type. Why aren't offset extractors considered "invocable"?

   Although ``offset_extractor`` is a functor, ``std::invoke`` is *not* used in its primary task of performing extraction. Instead, various helper classes in the implementation have partial template specializations for ``offset_extractor``. These helper classes use a different strategy for offset extractors than for all other extractor types (for details, see the link encoding notes in the :doc:`implementation notes <lists-implementation>`). This is actually the "secret sause" of ``offset_extractor`` that gives it a slight edge during code generation. ``offset_extractor`` is modeled as a functor because it simplifies certain other parts of the code.

.. _intrusive-when-to-use-offset-extractors:

When to use offset-based extractors
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Offset-based extractors are similar to pointer-to-data-member extractors, but they usually generate better code. Unfortunately, not all types can use them. The C++ standard only guarantees that ``offsetof`` will work on standard layout types, although since C++17, implementations are free to allow other types. Because of that freedom, CSD does not enforce the standard layout requirement -- so users must "know what they're doing" when using ``offsetof``.

Pointer-to-member-extractors are more powerful, and can solve two problems:

1. **Access Protection Problems** -- In most C++ compilers, an ``offsetof(T, <field>)`` appearing outside of T and where ``<field>`` is non-public will fail, unless a friend declaration allows access in the scope where ``offsetof`` appears. This can be fixed by adding a public "getter" method and using that as the extractor, e.g.

   .. code-block:: c++

      class ListItem {
      public:
        slist_entry<ListItem> &getLink() noexcept { return m_link; }

      private:
        int i;
        slist_entry<ListItem> m_link;
      };

      using list_t = slist_head_cinvoke_t<&ListItem::getLink>;

2. **Self-Referential List Heads** --
   The expression ``offsetof(C, <member-of-C>)`` is usually invalid when it appears inside the definition of ``class C``. For example, this will not work:

   .. code-block:: c++

      class C {
        slist_entry<C> link;
        CSG_SLIST_HEAD_OFFSET_T(C, link) children; // Error: C isn't a complete type
      };

   As far as the author is aware, there is nothing in the C standard that says this cannot work, but none of the popular implementations permit ``offsetof`` inside a type that is still being defined. However, using a pointer-to-member-based extractor *will* work, even before the type is complete:

   .. code-block:: c++

      class C {
        slist_entry<C> link;
        slist_head_cinvoke_t<&C::link> children; // OK
      };

   This may seem like an odd use case, but there are several "tree-like" structures like this in the FreeBSD kernel. As one example, a ``vm_object`` contains a ``LIST_HEAD`` to its shadow ``vm_object`` children, as part of the virtual memory copy-on-write feature. There is still one restriction: the expression ``&C::link`` must appear after the ``csg::slist_entry_link<C>`` member. If the user does not want to do this, the :ref:`proxy <lists-head-vs-fwd-head>` pattern can be used instead.

Both offset-based and ``invocable_constant`` extractors are :cpp:concept:`stateless <csg::stateless>`, and use no storage in the head object (thanks to a ``[[no_unique_address]]`` attribute). However, the user is free to use a stateful functor which can perform arbitrarily complex link access.
