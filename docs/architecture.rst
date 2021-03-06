SMQTK Architecture Overview
===========================

SMQTK is mainly comprised of 4 high level components, with additional sub-modules for tests, utilities and other control structures.

.. toctree::
  :maxdepth: 3

  dataabstraction
  algorithms
  webservices
  utilities

Plugin Architecture
-------------------

Each of these main components are housed within distinct sub-modules under :mod:`smqtk` and adhere to a plugin pattern for the dynamic discovery of implementations.

In SMQTK, data structures and algorithms are first defined by an abstract interface class that lays out what that services the data structure, or methods that the algorithm, should provide.
This allows users to treat instances of structures and algorithms in a generic way, based on their defined high level functionality, without needing to knowing what specific implementation is running underneath.
It lies, of course, to the implementations of these interfaces to provide the concrete functionality.

When creating a new data structure or algorithm interface, the pattern is that each interface is defined inside its own sub-module in the ``__init__.py`` file.
This file also defines a function ``get_..._impls()`` (replacing the ``...`` with the name of the interface) that returns a mapping of implementation class names to the implementation class type, by calling the general helper method :func:`smqtk.utils.plugin.get_plugins`.
This helper method looks for modules defined parallel to the ``__init__.py`` file as well as classes defined in modules listed in an environment variable (defined by the specific call to :func:`.get_plugins`).
The function then extracts classes that extend from the specified interface class as denoted by a helper variable in the discovered module or by searching attributes exposed by the module.
See the doc-string of :func:`smqtk.utils.plugin.get_plugins` for more information on how plugin modules are discovered.

Adding a new Interface and Internal Implementation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
For example, lets say we're creating a new data representation interface called :class:`FooBar`.
We would create a directory and :file:`__init__.py` file (python module) to house the interface as follows::

    python/
    └── smqtk/
        └── representation/
            └── foo_bar/          # new
                └── __init__.py   # new

Since we are making a new data representation interface, our new interface should descend from the :class:`smqtk.representation.SmqtkRepresentation` interface (algorithm interfaces would descend from :class:`smqtk.algorithms.SmqtkAlgorithm`).
The :class:`.SmqtkRepresentation` base-class descends from the :class:`.Configurable` interface (interface class sets ``__metaclass__ = abc.ABCMeta``, thus it is not set in the example below).

The :file:`__init__.py` file for our new sub-module might look something like the following, defining a new abstract class:

.. code-block:: python

    import abc

    from smqtk.representation import SmqtkRepresentation
    from smqtk.utils.plugin import Pluggable, get_plugins


    class FooBar (SmqtkRepresentation, Pluggable):
        """
        Some documentation on what this does.
        """
        # Interface methods and/or abstract functionality here.
        # -> See the abc module on how to decorate abstract methods.

        @abc.abstractmethod
        def do_something(self):
            """ Does Something """


    def get_foo_bar_impls(reload_modules=False):
        import os.path as osp
        from smqtk.utils.plugin import get_plugins
        this_dir = osp.abspath(osp.dirname(__file__))
        env_var = 'FOO_BAR_PATH'
        helper_var = 'FOO_BAR_CLASS'
        return get_plugins(__name__, this_dir, env_var, helper_var, FooBar,
                           reload_modules)


When adding a an implementation class, if it is sufficient to be contained in a single file, a new module can be added like::

    python/
    └── smqtk/
        └── representation/
            └── foo_bar/
                ├── __init__.py
                └── some_impl.py  # new

Where :file:`some_impl.py` might look like:

.. code-block:: python

    from smqtk.representation.foo_bar import FooBar

    class SomeImpl (FooBar):
        """
        Some documentation
        """
        # Implementation of abstract methods here


Implementation classes can also live inside of a nested sub-module.
This is useful when an implementation class requires specific or extensive support utilities (for example, see the :class:`.DescriptorGenerator` implementation :class:`.ColorDescriptor`).::

    python/
    └── smqtk/
        └── representation/
            └── foo_bar/
                ├── __init__.py
                ├── some_impl.py
                └── other_impl/      # new
                    └── __init__.py  # new

Where the :file:`__init__.py` file should at least expose concrete implementation classes that should be exported as attributes for the plugin getter to discover.


Both :class:`.Pluggable` and :class:`.Configurable`
"""""""""""""""""""""""""""""""""""""""""""""""""""
It is important to note that our new interface, as defined above, descends from both the :class:`.Configurable` interface (transitive through the :class:`.SmqtkRepresentation` base-class) and the :class:`.Pluggable` interface.

The :class:`.Configurable` interface allows classes to be instantiated via a dictionary with JSON-compliant data types.
In conjunction with the plugin getter function (``get_foo_bar_impls`` in our example above), we are able to select and construct specific implementations of an interface via a configuration or during runtime (e.g. via a transcoded JSON object).
With this flexibility, an application can set up a pipeline using the high-level interfaces as reference, allowing specific implementations to be swapped in an out via configuration.


Reload Use Warning
""""""""""""""""""

While the :func:`smqtk.utils.plugin.get_plugins` function allows for reloading discovered modules for potentially new content, this is not recommended under normal conditions.
When reloading a plugin module after :mod:`pickle` serializing an instance of an implementation, deserialization causes an error because the original class type that was pickled is no longer valid as the reloaded module overwrote the previous plugin class type.


Function and Interface Reference
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. autofunction:: smqtk.utils.plugin.get_plugins

.. autoclass:: smqtk.utils.plugin.Pluggable
   :members:

.. autoclass:: smqtk.utils.configurable_interface.Configurable
   :members:
