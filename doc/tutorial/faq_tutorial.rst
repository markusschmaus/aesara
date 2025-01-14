.. _faq_tutorial:

===========================
Frequently Asked Questions
===========================

How to update a subset of weights?
==================================
If you want to update only a subset of a weight matrix (such as
some rows or some columns) that are used in the forward propagation
of each iteration, then the cost function should be defined in a way
that it only depends on the subset of weights that are used in that
iteration.

For example if you want to learn a lookup table, e.g. used for
word embeddings, where each row is a vector of weights representing
the embedding that the model has learned for a word, in each iteration,
the only rows that should get updated are those containing embeddings
used during the forward propagation. Here is how the aesara function
should be written:

Defining a shared variable for the lookup table

.. code-block:: python

   lookup_table = aesara.shared(matrix_ndarray)

Getting a subset of the table (some rows or some columns) by passing
an integer vector of indices corresponding to those rows or columns.

.. code-block:: python

   subset = lookup_table[vector_of_indices]

From now on, use only 'subset'. Do not call lookup_table[vector_of_indices]
again. This causes problems with grad as this will create new variables.

Defining cost which depends only on subset and not the entire lookup_table

.. code-block:: python

   cost = something that depends on subset
   g = aesara.grad(cost, subset)

There are two ways for updating the parameters:
Either use inc_subtensor or set_subtensor. It is recommended to use
inc_subtensor. Some aesara rewrites do the conversion between
the two functions, but not in all cases.

.. code-block:: python

   updates = inc_subtensor(subset, g*lr)

OR

.. code-block:: python

   updates = set_subtensor(subset, subset + g*lr)

Currently we just cover the case here,
not if you use inc_subtensor or set_subtensor with other types of indexing.

Defining the aesara function

.. code-block:: python

   f = aesara.function(..., updates=[(lookup_table, updates)])

Note that you can compute the gradient of the cost function w.r.t.
the entire lookup_table, and the gradient will have nonzero rows only
for the rows that were selected during forward propagation. If you use
gradient descent to update the parameters, there are no issues except
for unnecessary computation, e.g. you will update the lookup table
parameters with many zero gradient rows. However, if you want to use
a different optimization method like rmsprop or Hessian-Free optimization,
then there will be issues. In rmsprop, you keep an exponentially decaying
squared gradient by whose square root you divide the current gradient to
rescale the update step component-wise. If the gradient of the lookup table row
which corresponds to a rare word is very often zero, the squared gradient history
will tend to zero for that row because the history of that row decays towards zero.
Using Hessian-Free, you will get many zero rows and columns. Even one of them would
make it non-invertible. In general, it would be better to compute the gradient only
w.r.t. to those lookup table rows or columns which are actually used during the
forward propagation.
