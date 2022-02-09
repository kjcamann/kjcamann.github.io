************
Installation
************

.. contents::
   :local:

CSD is a header-only library, so no explicit build step is needed to use it. However, there are a few useful extra features that need explicit configuration, as described below.

.. _install-assertions:

Enabling assertions
===================

CSD containers are modeled after the STL, and STL containers do not throw exceptions when their API contract is violated. For example, the cppreference documentation for `std::list<T>::erase(iterator pos) <https://en.cppreference.com/w/cpp/container/list/erase>`_ states that:

   The iterator pos must be valid and dereferenceable

and it also shows that the ``Exceptions`` thrown are ``(none)``. Moreover, this non-throwing design decision is explicitly specified in the C++ standard. So what happens if the user violates the API contract and passes an invalid iterator to ``erase``? Unsurprisingly, the effect is undefined.

Some C++ standard library implementations, e.g. LLVM's libc++, contain assertion macros to check for incorrect usage by the caller. For example:

.. code-block:: c++

   _LIBCPP_ASSERT(__p != end(),
           "list::erase(iterator) called with a non-dereferenceable iterator");

For performance reasons these checks are not compiled by default but can (and should) be enabled in debug builds to catch contract violations.

CSG borrows this design from libc++, which the user must opt into by doing two things:

1. You must ``#define`` the macro ``CSG_DEBUG_LEVEL`` to ``1`` via the C preprocessor, otherwise assertions will be ignored.

2. You must provide a free function called ``csg_assert_function`` which will be called when an assertion fails. An example of this function is shown below.

When ``CSG_DEBUG_LEVEL == 1``, the internal macro ``CSG_ASSERT`` will emit calls to ``csg_assert_function``, which is declared but not defined in the CSG header files. No default definition is provided because the "weak linkage" concept is not portable to Windows and it is relatively easy to implement in any case. A simple implementation that prints the violation to stderr and then calls `std::terminate <https://en.cppreference.com/w/cpp/error/terminate>`_ is shown below:

.. code-block:: c++

   extern "C" [[noreturn]] void
   csg_assert_function(const csg::assert_info &info, const char *format, ...) noexcept {
     std::fprintf(stderr, "assertion failed: %s <%s@%s:%d>\n", info.test,
                 info.function, info.file, info.line);

     if (format) {
       std::fprintf(stderr, "cause: ");

       std::va_list args;
       va_start(args, format);
       std::vfprintf(stderr, format, args);
       va_end(args);

       std::fprintf(stderr, "\n");
     }

     std::terminate();
   }

.. _install-gdbvis:

Installing the GDB pretty-printers
==================================

CSD includes a GDB pretty-printer extension to improve the formatting of its types. Like all GDB extensions, pretty-printers are implemented as Python plugins. The CSD pretty-printer functionality is contained in a single file, ``utils/gdb-printers/csd.py``. To use it, you must import this Python module and call the function ``register_csd_pretty_printers()``. Two options for doing this automatically are described below.

Option 1: always load the CSD pretty-printers
---------------------------------------------

If you are a heavy user of CSD, you can add the following Python code directly to your GDB initialization file:

.. code-block:: python

   python

   import sys
   sys.path.insert(0, '<path-to-directory-containing-csd.py>')
   from csd import register_csd_pretty_printers
   register_csd_pretty_printers()

   end

This will register the CSD pretty-printers every time you start ``gdb``. The typical initialization file is loaded from ``$XDG_CONFIG_HOME/gdb/gdbinit``, which defaults to ``$HOME/.config/gdb/gdbinit`` when ``$XDG_CONFIG_HOME`` is not defined. For more information see the GDB documentation `here <https://sourceware.org/gdb/onlinedocs/gdb/Startup.html>`_, specifically `this section <https://sourceware.org/gdb/onlinedocs/gdb/Initialization-Files.html#Home-Directory-Init-File>`_.

Option 2: use GDB's Python auto-loading features
------------------------------------------------

GDB offers a number of ways to automatically load Python extensions that you associate with particular executables or shared libraries. The project's documentation on `Python auto-loading <https://sourceware.org/gdb/onlinedocs/gdb/Python-Auto_002dloading.html#Python-Auto_002dloading>`_ explains these, but the easiest way is shown below.

Suppose you have an executable ``foo`` that uses CSD. Create a file called ``foo-gdb.py`` in the same directory as ``foo``, with these contents:

.. code-block:: python

   import sys
   sys.path.insert(0, '<path-to-directory-containing-csd.py>')
   from csd import register_csd_pretty_printers
   register_csd_pretty_printers()

Now GDB will automatically load the pretty-printers when ``foo`` is loaded. This also works for shared libraries (e.g., with a filename like ``libbar.so-gdb.py``). The user should be aware that by default, GDB auto-loading is restricted for security reasons. Unless you've explicitly relaxed GDB's security security settings, you'll need to add a line like this to your ``gdbinit``:

.. code-block:: none

   add-auto-load-safe-path <path-to-foo-gdb.py>

Note that the autoload safe path can be the path to the file itself or -- for more flexibility and less safety -- to any of its parent directories.

.. _install-cmake:

Building tests and documentation
================================

The included CMake build system is used to build the test suite and the Sphinx documentation. You can also use the ``install`` target to copy the CSD headers into an appropriate location. To build the Sphinx documentation, you must also install `doxygen <https://www.doxygen.org>`_, `breathe <https://breathe.readthedocs.io>`_, the `"Read the Docs" Sphinx theme <https://sphinx-rtd-theme.readthedocs.io/en/latest>`_, and `Sphinx itself <https://www.sphinx-doc.org/en/stable/>`_. Building the test suite will use CMake's `ExternalProject <https://cmake.org/cmake/help/latest/module/ExternalProject.html>`_ command to fetch the `Catch2 <https://github.com/catchorg/Catch2>`_ unit testing framework from Github, so it requires an Internet connection.
