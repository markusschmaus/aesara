.. _tutconditions:

==========
Conditions
==========

IfElse vs Switch
================


- Both ops build a condition over symbolic variables.
- ``IfElse`` takes a *boolean* condition and two variables as inputs.
- ``Switch`` takes a *tensor* as condition and two variables as inputs.
  ``switch`` is an elementwise operation and is thus more general than ``ifelse``.
- Whereas ``switch`` evaluates both *output* variables, ``ifelse`` is lazy and only
  evaluates one variable with respect to the condition.

**Example**


.. testcode::

   from aesara import tensor as at
   from aesara.ifelse import ifelse
   import aesara, time, numpy

   a,b = at.scalars('a', 'b')
   x,y = at.matrices('x', 'y')

   z_switch = at.switch(at.lt(a, b), at.mean(x), at.mean(y))
   z_lazy = ifelse(at.lt(a, b), at.mean(x), at.mean(y))

   f_switch = aesara.function([a, b, x, y], z_switch,
                              mode=aesara.compile.mode.Mode(linker='vm'))
   f_lazyifelse = aesara.function([a, b, x, y], z_lazy,
                                  mode=aesara.compile.mode.Mode(linker='vm'))

   val1 = 0.
   val2 = 1.
   big_mat1 = numpy.ones((10000, 1000))
   big_mat2 = numpy.ones((10000, 1000))

   n_times = 10

   tic = time.clock()
   for i in range(n_times):
       f_switch(val1, val2, big_mat1, big_mat2)
   print('time spent evaluating both values %f sec' % (time.clock() - tic))

   tic = time.clock()
   for i in range(n_times):
       f_lazyifelse(val1, val2, big_mat1, big_mat2)
   print('time spent evaluating one value %f sec' % (time.clock() - tic))

.. testoutput::
   :hide:
   :options: +ELLIPSIS

   time spent evaluating both values ... sec
   time spent evaluating one value ... sec

In this example, the ``IfElse`` op spends less time (about half as much) than ``Switch``
since it computes only one variable out of the two.

.. code-block:: none

   $ python ifelse_switch.py
   time spent evaluating both values 0.6700 sec
   time spent evaluating one value 0.3500 sec

Unless ``linker='vm'`` or ``linker='cvm'`` are used, ``ifelse`` will compute both
variables and take the same computation time as ``switch``. Although the linker
is not currently set by default to ``cvm``, it will be in the near future.

There is no automatic rewrite replacing a ``switch`` with a
broadcasted scalar to an ``ifelse``, as this is not always faster. See
this `ticket <http://www.assembla.com/spaces/theano/tickets/764>`_.

.. note::

   If you use :ref:`test values <test_values>`, then all branches of
   the IfElse will be computed. This is normal, as using test_value
   means everything will be computed when we build it, due to Python's
   greedy evaluation and the semantic of test value. As we build both
   branches, they will be executed for test values. This doesn't cause
   any changes during the execution of the compiled Aesara function.
