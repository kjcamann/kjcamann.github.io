******************
Intrusive Concepts
******************

.. cpp:concept:: template <typename C> csg::stateless

   A type is stateless if it is `empty <https://en.cppreference.com/w/cpp/types/is_empty>`_, `trivially constructible <https://en.cppreference.com/w/cpp/types/is_constructible>`_, and `trivially destructible <https://en.cppreference.com/w/cpp/types/is_destructible>`_. Together, these properties ensure that there is no need to store a reference to a specific instance of a stateless class, because constructing a new instance on the fly has no cost and is always equivalent:

   * Both instances have the same data, because they are both empty
   * The creation and destruction of the new instance are side-effect-free because the constructor and destructor are trivial (and are therefore no-ops that will be optimized away)

   .. seealso::
      The documentation for :cpp:class:`csg::compressed_invocable_ref` contains an explanation of how this is used in CSD.

.. cpp:concept:: template <typename T> csg::optional_size

   This concept is satisfied when the type is an `integral <https://en.cppreference.com/w/cpp/concepts/integral>`_ type or when it is the special empty struct :cpp:class:`csg::no_size`. The intention is to allow containers to either store their size inline (so that `std::size <https://en.cppreference.com/w/cpp/iterator/size>`_ will be :math:`O(1)`) or to save space and compute it on the fly (but then `std::size <https://en.cppreference.com/w/cpp/iterator/size>`_ will be :math:`O(n)`). This is usually done by accepting the size type as a template parameter, which is constrained to to be an ``optional_size``. The size member variable itself is usually declared with `[[no_unique_address]] <https://en.cppreference.com/w/cpp/language/attributes/no_unique_address>`_, so that it does not take any space when it is ``csg::no_size``.
