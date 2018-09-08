******************
Intrusive Concepts
******************

.. cpp:concept:: template <typename C> bds::Stateless

   A type is stateless if it is both `empty <https://en.cppreference.com/w/cpp/types/is_empty>`_ and `trivially constructible <https://en.cppreference.com/w/cpp/types/is_constructible>`_. Together, these properties ensure that there is no need to store a reference to a specific instance of a stateless class, because constructing a new instance on the fly has no cost and is always equivalent:

   * Both instances have the same data, because they are both empty
   * The act of constructing the new instance is side-effect-free because the constructor is trivial (and is therefore a no-op that will be optimized away)

   .. seealso::
      The documentation for :cpp:class:`bds::compressed_invocable_ref` contains an explanation of how this is used in BDS.

.. cpp:concept:: template <typename T> bds::SizeMember

   This concept is satisfied when the type is `Integral <https://en.cppreference.com/w/cpp/concepts/Integral>`_ type or when it is the special empty struct :cpp:class:`bds::no_size`. The intention is to allow containers to either store their size inline (so that `std::size <https://en.cppreference.com/w/cpp/iterator/size>`_ will be :math:`O(1)`) or to save space and compute it on the fly (but then `std::size <https://en.cppreference.com/w/cpp/iterator/size>`_ will be :math:`O(n)`). This is usually done by accepting the size type as a template parameter, which is constrained to to be a ``SizeMember``. The size member variable itself is usually declared with `[[no_unique_address]] <https://en.cppreference.com/w/cpp/language/attributes/no_unique_address>`_, so that it does not take any space when it is ``bds::no_size``.
