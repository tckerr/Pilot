Pilot
=====

Pilot is a python library for traversing object trees and graphs. It is
currently in pre-alpha.

https://pypi.python.org/pypi/pilot

1. `Usage <#usage>`__
2. `Documentation <#documentation>`__

   -  `Classes <#classes>`__
   -  `Callbacks <#callbacks>`__
   -  `Nodes <#nodes>`__

Usage
=====

Download and include the library:

::

    $ pip install pilot

The first step is to define a callback:

::

    >>> def my_callback(node):
    ...    print node.key, "-->", node.val
    ...

Now, define the pilot object and execute a flight:

::

    >>> data = {'test': 1}
    >>> pilot = Pilot(my_callback)
    >>> new_data = pilot.fly(data)
    None --> {'test': 1}
    test --> 1
    >>> type(new_data)
    <class 'pilot.node.dict_extended__node'>
    >>> new_data.__node__
    <pilot.node.Node object at 0x026CCE90>

See the documentation for more details!

Documentation
=============

Creating a pilot: ``pilot = Pilot(*callbacks, **settings)``
-----------------------------------------------------------

Callbacks and keyword arguments (settings) can be passed into the pilot
constructor.

**Settings options**:

-  ``run_callbacks`` *(true\|false)*: Set this to false to skip
   callbacks completely.
-  ``callbacks``: an array of detailed callback objects. See the
   Callback section for more information. If this key is specified *and*
   there are callbacks passed into the constructor via args, an
   exception will be raised. If finer control on callbacks is required,
   use this setting.
-  ``traversal_mode``: the mode for traversing the tree. Options are
   ``depth`` for *depth-first* processing and ``breadth`` for
   *breadth-first* processing.
-  ``structure``: Options are ``tree`` (default) and ``graph``. Graphs
   add nodes as "neighbors" rather than parent/child.
-  ``node_visit_limit``: An integer that defines how many times to allow
   visits to nodes. (Only applicable to graphs.) Set to ``-1`` to allow
   infinite. Note that infinite trees will process indefinitely - use
   ``Pilot.halt()`` in a callback to end the processing manually.
-  ``callback_sort``: A function that will sort callbacks into an
   execution order. Note that callback sorting will only apply to a set
   of callbacks running on a property and position basis, since that is
   the callback execution stack grouping.
-  ``callback_sort_mode`` *(0\|1)*: This parameter describes the
   callback sorting function's required args. Options are ``0`` for a
   standard lambda sort function and ``1`` for a ``cmp`` function.

The configuration defaults to the following:

::

    structure = "Tree"
    node_visit_limit = 1
    traversal_mode = 'depth'
    run_callbacks = True
    callback_sort: lambda self, x: x.priority
    callback_sort_mode = 0

``pilot.fly(self, object, rootkey=None, rootpath=None, rootparent=None)``
-------------------------------------------------------------------------

Traverse an object and execute callbacks during the traversal. The
flight will convert every property into a ``Node``, and will make that
node available on the property through ``property.__node__``. Note that
this means the process will ocassionally change the underlying classes
of objects to support adding the ``__node__`` property. As of now, most
types are supported -- if the conversion process fails, it may be
because Pilot is unable to modify the underlying class. See the
documentation on the **Node** class for more details.

``Pilot.halt()`` *(static method)*:
-----------------------------------

Calling this method within a callback will halt processing completely.
This allows for early exit of the traversal, which is required for
limited processing of infinite trees.

Callbacks
---------

Callbacks are special functions that execute on the object tree/graph as
it is being traversed, and are constructed from predefined functions.
Each callback executes at different times based on several configuration
options listed below. Callbacks are passed a single argument, the
**Node** for the object. The general form of a callback object is:

::

    def my_callback(node):
        do_some_things()

    from pilot import Callback
    callback = Callback(my_callback)

See the "Usage" section of the documentation for a shorthand way to apss
callbacks into the Pilot.

Here's an example, using custom callbacks:

::

    data = [
        {'name': 'Jim', 'friends': [{},{}]},
        {'name': 'Jane', 'friends': [{},{},{}]},
        {'name': 'Joe', 'friends': []}
    ]

    def count_friends(node):
        try:
            node.val['name']
        except: # not a person        
            return
        person = node.val
        friends = person.get('friends', None)
        if friends:
            print person.get("name") + " has " + str(len(friends)) + " friends."
        else:
            print person.get("name") + " has no friends."

    from pilot import Callback, Pilot
    callback = Callback(count_friends, containers=['dict'])
    pilot = Pilot(callbacks=[callback])
    newdata = pilot.fly(data)

Here are the keyword args you can supply in a callback configuration,
most of which act as filters:

-  ``containers``: a list of containers to run on. Options are
   ``'dict'``, ``'list'``, and ``'value'``. If unspecifed, the callback
   will run on any container.
-  ``keys``: an array of keys to run on. The callback will check the key
   of the property against this list. If unspecified, the callback will
   run on any key.
-  ``positions``: a list of positions in the traversal to run on.
   Options are ``'pre'`` (before any list/object is traversed), and
   ``'post'`` (after any list/object is traversed). For properties of
   container-type ``'value'``, these two run in immediate succession. If
   unspecifed, the callback will run ``'post'``.
-  ``priority``: an integer that represents the callback's place in the
   execution order. Lower priorities execute first.
-  ``run_for_orphans``: a boolean that determines whether the callback
   should run on oprhan nodes.

If any of the filters are not sufficient, additional checks can be done
in the callback.

Nodes
-----

Node objects represent a single node in the tree, providing metadata
about the value, its parents, siblings, children, and more. Nodes have
the following properties:

-  ``key``: The key of this property as defined on it's parent. For
   example, if this callback is running on the ``'weight'`` property of
   a ``person``, the ``key`` would be ``'weight'``. Note that this will
   be ``None`` for properties in list.
-  ``value``: The value of the property. To use the above example, the
   value would be something like ``'183'``.
-  ``container``: The type of container of the property expressed in an
   integer. See the following enumeration for container types:
   ``from pilot.definitions import ContainerType``
-  ``encountered``: The number of times the pilot has processed the
   node.
-  ``id``: A unique id for the node (within the context of piot).

You may also call the following methods off of a node object. Keep in
mind that some of the node accessors may not be populated based on where
you are in your traversal. If you want to wait for the relationship tree
to be fully established, run operations on the output of a flight, which
will be a data object with each value annotated with a ``__node__``
property:

-  ``is_orphan()``: Returns a boolean of whether the node is an orphan
   (no parents.)

Node relationship accessors
~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  ``parent()``: The first node under which the property exists.
   ``node.parent`` is another instance of node, and will have all the
   same properties. This is a convenience method that's useful in trees,
   where only one parent is possible.
-  ``parents()``: A method that returns parents of a node. An optional
   search parameters object can be passed in to filter the list to all
   who have matching key-values. For example,
   ``node.parents(key='name', val='Tom')`` will return all parents where
   ``key == 'name'`` and ``val == 'Tom'``.
-  ``children()``: A method that returns children of a node (i.e. all
   nodes whose parent is this node.) An optional search parameters
   object can be passed in to filter the list to all who have matching
   key-values. For example, ``node.children(key='name', val='Tom')``
   will return all children where ``key == 'name'`` and
   ``val == 'Tom'``.
-  ``neighbors()``: A method that returns adjacent (non-parent,
   non-child) nodes, for use in graphs. An optional search parameters
   object can be passed in to filter the list to all who have matching
   key-values. For example, ``node.neighbors(key='name', val='Tom')``
   will return all neighbors where ``key == 'name'`` and
   ``val == 'Tom'``.
-  ``siblings()``: A method that returns all nodes that exist alongside
   the current node within its parents. For parents of container
   ``'object'``, this includes all other properties of the parent
   object. For parents of type ``'array'``, this includes all other
   nodes in that array.
-  ``orphans()``: A method that returns all connected root nodes.
-  ``ancestors()``: A method that returns a list of all ancestor nodes,
   going back to the root.
-  ``descendants()``: A method that returns a list of all descendant
   nodes.

Each relationship accessor can be pased the following keyword arguments:

-  ``filters=None``: A dictionary of key:val that will be matched
   against each encountered node's val (as of now this is only useful
   for dict types).
-  ``as_value=False``: Boolean that will make the call return node
   values rather than node objects.
-  ``as_generator=False``: Boolean that will make the call return a
   generator object for the relationship. This is useful when the
   relationship list is large or inifinite, since it will not attempt to
   build the list entirely at call time.