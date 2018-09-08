******************************
BDS lists implementation notes
******************************

.. contents::
   :local:

Link encoding
=============

Links to neighboring list items are stored in intrusive "entry" structures like this one:

.. code-block:: c++

   struct tailq_entry {
     std::uintptr_t next;
     std::uintptr_t prev;
   };

Note that each link is stored as an opaque, pointer-sized value -- this value will either be a pointer to a neighbor's ``tailq_entry`` or a tagged pointer that uses a more complex encoding that is explained below.

The core list classes, e.g., ``tailq_base``, use a traits class called ``link_encoder`` to read and write link values in the entry. The ``EntryAccess`` template parameter determines what kind of link encoding the list will use, via template partial specialization.

Offset link encoding
--------------------

In the original ``queue(3)`` library, the links are pointers to the neighboring entry structures. To get a pointer to the list item given a pointer to the entry, ``queue(3)`` calculates the address of the enclosing ``T`` object from the address of the entry using the ``offsetof`` macro. In BDS, this is referred to as the "offset link encoding," because it relies on the entry living at a known offset in the list item. A picture is shown below:

.. figure:: images/slist-diagram-offset.png
   :alt: slist using an offset link encoding

   An ``slist`` using an offset-based link encoding. This tends to generate the most efficient code.

The ``link_encoder`` class template has a partial specialization when the entry access functor is ``offset_extractor``:

.. code-block:: c++

   template <typename EntryType, std::size_t Offset, typename T>
   struct link_encoder<EntryType, offset_extractor<EntryType, Offset>, T> {
     // ... specialization using offsetof
   };

To get a ``tailq_entry *`` from a ``T *`` (or vice versa), this specialization just adds or subtracts the offset and performs pointer casts.

Invocable link encoding
-----------------------

Although offset link encoding tends to generate the most compact code, it cannot be used with every item type. The C++ standard only guarantees that ``offsetof`` will work on standard layout types (because ``queue(3)`` is a C library, it only needs to concern itself with such types). Since C++17, an implementation may choose to support other types if its class layout strategy can guarantee a fixed offset for that type.

But there are also other reasons to avoid ``offsetof``, e.g., if the user prefers encapsulation and does not want to use friend declarations:

.. code-block:: c++

   class ListItem {
   public:
     tailq_entry &getEntry() noexcept { return m_entry; }

   private:
     tailq_entry m_entry; // Not accessible by the list, must call getEntry
   };

In this case, the tailq's ``EntryAccess`` template argument will be a member function pointer to ``getEntry`` wrapped in a functor, i.e. ``constexpr_invocable<ListItem::getEntry>``.

Here, the "invocable link encoding" is used. In this scheme, most links store a tagged pointer to the neighboring item but with the low bit set. This low bit is the "value tag", indicating that the pointer refers to a list item.

If the low bit is not set, it means the link directly points to an entry structure. This is needed for the "head" entry, which does not have a corresponding list item. The tagging behavior is shown in later figure, depicting the :ref:`tailq design <lists_tailq_invocable_figure>`. Below we see the invocable link encoding for a simple ``slist``:

.. figure:: images/slist-diagram-invocable.png
   :alt: slist using an invocable link encoding

   An ``slist`` using the invocable link encoding. List entries usually store pointers to items, not to their entry subobject. Entries are obtained by calling the entry accessor function.

To get a ``tailq_entry *`` from a ``T *`` we retrieve the ``T *`` value by masking off the value tag and invoking the entry access functor to yield the entry:

.. code-block:: c++

   tailq_entry &entry = std::invoke(entryAccessor, *(T *)(encoded & ~ValueTag));

Without ``offsetof``, we cannot easily derive a ``T *`` from a ``tailq_entry *``, which is why this encoding is needed in the first place: the implementation must remember the item pointer and poke at the entry using the accessor whenver it needs it.

Iterator implementation
-----------------------

List iterators store an encoded link value for the "current" list item. As described above, with the help of the link encoder (and the ``EntryAccess`` functor, for invocable link encoding), the iterator can access both the ``entry`` object and the list item.

The iterator is either the size of a single pointer or two pointers, depending on the ``EntryAccess`` functor. The iterator stores a special "smart pointer" to the invocable object, :cpp:class:`bds::compressed_invocable_ref`, which (using template partial specialization) is empty if the ``EntryAccess`` functor is :cpp:concept:`stateless <bds::Stateless>`, i.e., if it is both an empty class and is trivially-constructible. This smart pointer is also declared with ``[[no_unique_address]]``, so if the accessor is stateless, it will not store anything. This is the common case, because both offset-based and `constexpr_invocable`-based functors are stateless.

List design
===========

The singly-linked list design is straight-forward: a head entry points to the first item, and all the intrusive entries point to the next item. The last item has a ``next`` value of ``nullptr`` (encoded as zero). In the case of ``stailq``, an encoded link to the last item also stored.

The ``tailq``'s doubly-linked list design is insipred by the `std::list <https://github.com/llvm-mirror/libcxx/blob/master/include/list>`_ implementation in libc++. In this design, the list is actually circular: the ``tailq_entry`` in the list head links the two ends of the list, and corresponds to the ``end()`` sentinel. Its "previous" link points to the last item in the list, and its "next" item points back around to the first item in the list. This is shown in the figure below, which also illustrates the tagged pointers in the invocable link encoding:

.. _lists_tailq_invocable_figure:

.. figure:: images/tailq-diagram-invocable.png
   :alt: tailq data structure diagram

   A ``tailq`` using an invocable link encoding, and demonstrating the circular list design. The shaded pointers are untagged, so they point directly to the ``tailq_entry`` in the head object. The unshaded pointers are "value tagged," so they point directly to list items.

When the list is empty, both links in the head entry point to itself:

.. figure:: images/tailq-empty-diagram.png
   :alt: empty tailq data structure diagram

This is the ideal design given the STL container requirements, e.g., that ``*--end()`` is a valid expression that yields the same value as ``back()``. If you work out how to edit the links to insert and remove items "in the middle" of the list, this same code will work even at the boundary conditions (a list of size 0, size 1, etc.) without any special conditionals that check for those cases.

This interesting property makes the circular doubly-linked list one of the "cleanest" and most "algorithmically satisfying" data structures you're ever likely to encounter -- especially when compared to "job interview white board" linked lists which explicitly use ``nullptr`` at the head/tail of the list, and need ``if`` statements in almost every function to treat the boundary cases differently.
