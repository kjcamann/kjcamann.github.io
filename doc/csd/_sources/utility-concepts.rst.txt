****************
Utility Concepts
****************

.. cpp:concept:: template <typename T> csg::default_arg_initializable

   A type is "default argument initializable" if it is `default initializable <https://en.cppreference.com/w/cpp/concepts/default_initializable>`_ and `move constructible <https://en.cppreference.com/w/cpp/concepts/move_constructible>`_. This concept is used to enable constructors that take default arguments having a dependent type. For example, consider this simple hash map:

   .. code-block:: c++

      template <typename Key, typename Value,
                hash_function<const Key &> KeyHash,
                std::equivalence_relation<const Key &, const Key &> KeyEqual>
      class example_hash_map {
      public:
        // Constructor overload that allows the user to omit arguments for
        // KeyHash and/or KeyEqual.
        example_hash_map(std::size_t size, KeyHash h = KeyHash{}, KeyEqual e = KeyEqual{})
            requires util::default_arg_initializable<KeyHash> &&
                     util::default_arg_initializable<KeyEqual>
        : m_hash{std::move(h)}, m_keyeq{std::move(e)} {}

         // class body omitted

      private:
        KeyHash m_hash;
        KeyEqual m_keyeq;
      };

   The above contains a C++11 modernization of a pattern found in the original C++98 STL, e.g., `std::map <https://en.cppreference.com/w/cpp/container/map/map>`_ uses the following constructor overload to spare the user from needing to pass a comparator and an allocator most of the time:

   .. code-block:: c++

      template <typename InputIt>
      map(InputIt first, InputIt second,
          const Compare &comp = Compare(),
          const Allocator &alloc = Allocator());

   Where the C++98 STL used const-references like ``const Compare &compare = Compare()``, we use move semantics instead: there are far more "move-only" types than there are "copy-only" types, and the user can easily get copy semantics by passing an lvalue. For example:

   .. code-block:: c++

      MyKeyEqualFunctor equal{foo};
      example_hash_map<K, V, H, MyKeyEqualFunctor> m{size, {}, equal};

   In the above code, the user wants the default-constructed hash function and so passes ``{}``, an empty initializer list. By the copy/move-elision rules of C++17, this will directly initialize the ``h`` argument, and the ``h`` argument will be moved into ``m_hash``. On the other hand, ``equal`` is an l-value. A copy of ``equal`` will be made to initialize the ``e`` argument, then ``e`` will be moved into ``m_keyeq``.

   While potentially expensive, a copy is what the user intends here. Note that the const-reference version of this trick (from the C++98 STL) also requires a copy: although binding to the ``const &`` constructor argument is cheap, we must run the copy constructor to construct the ``m_keyeq`` member variable inside the map -- this cannot be copy-elided.

   The advantage of the modern version is that *all* copies can be eliminated if the types support it and the constructor arguments are x-values. C++11 rules also give us the ability to skip an argument just by writing ``{}`` at the call site.

   To support exotic types that are not default initializable or move constructible, typically the class template in question will provide a constructor overload based on the :cpp:concept:`csg::can_direct_initialize` pattern that can be used instead. The disadvantage of that approach compared to this one is that default arguments cannot be used.

.. cpp:concept:: template <typename T> csg::can_direct_initialize

   This concept has a simple definition:

   .. code-block:: c++

      template <typename Arg, typename T>
      concept can_direct_initialize = std::constructible_from<T, Arg>;

   It is typically used on constructor templates, to check if a single passed-in argument is able to initialize an instance of the needed type. It differs from `std::constructible_from <https://en.cppreference.com/w/cpp/concepts/constructible_from>`_ in that the template parameters are in reverse order, so that it can be used to introduce a constrained template parameter for the argument.

   Continuing with the ``example_hash_map`` example introduced in the :cpp:concept:`csg::default_arg_initializable` documentation, the following constructor overload demonstrates how this concept is used:

   .. code-block:: c++

      template <typename Key, typename Value,
                hash_function<const Key &> KeyHash,
                std::equivalence_relation<const Key &, const Key &> KeyEqual>
      class example_hash_map {
      public:
        template <util::can_direct_initialize<KeyHash> KeyHashInit,
                  util::can_direct_initialize<KeyEqual> KeyEqualInit>
        example_hash_map(std::size_t size, KeyHashInit &&h, KeyEqualInit &&e)
        : m_hash{std::forward<KeyHashInit>(h)}, m_keyeq{std::foward<KeyEqualInit>(e)} {}

         // class body omitted

      private:
        KeyHash m_hash;
        KeyEqual m_keyeq;
      };

   Here, ``KeyHashInit`` and ``KeyEqualInit`` are the types of arbitrary objects the user has passed, determined through template argument deduction, that are capable of initializing ``KeyHash`` and ``KeyEqual`` respectively.

   Constructor overloads that use :cpp:concept:`csg::can_direct_initialize` tend to be the most "powerful": they can even work with types that are not default-initializable, move-constructible, or copy-constructible. As long as the type can be direct-initialized by *something* that is a single argument, that single argument is perfect-forwarded through the ``example_hash_map`` and the construction happens directly in the member variable.

   When our ``example_hash_map`` must accept more than one dependent type at construction time, e.g., with ``KeyHash`` and ``KeyEqual`` above, we normally prefer a constructor based on :cpp:concept:`csg::default_arg_initializable` because it allows the user to skip some arguments and use a reasonable default instead. As mentioned above, however, there are some abnormal types that can only work with forwarding construction, so the default argument constructors may be disabled and perfect-forwarding constructors using :cpp:concept:`csg::can_direct_initialize` must be used instead. The downside of using this more-powerful concept is that default arguments cannot be used: writing a ``{}`` will cause template argument deduction to fail.
