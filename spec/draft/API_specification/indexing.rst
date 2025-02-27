.. _indexing:

Indexing
========

    Array API specification for indexing arrays.

A conforming implementation of the array API standard must adhere to the following conventions.


.. _indexing-single-axis:

Single-axis Indexing
--------------------

To index a single array axis, an array must support standard Python indexing rules. Let ``n`` be the axis (dimension) size.

- An integer index must be an object satisfying `operator.index <https://www.python.org/dev/peps/pep-0357/>`_ (e.g., ``int``).

- Nonnegative indices must start at ``0`` (i.e., zero-based indexing).

- **Valid** nonnegative indices must reside on the half-open interval ``[0, n)``.

  .. note::
    This specification does not require bounds checking. The behavior for out-of-bounds integer indices is left unspecified.

- Negative indices must count backward from the last array index, starting from ``-1`` (i.e., negative-one-based indexing, where ``-1`` refers to the last array index).

  .. note::
    A negative index ``j`` is equivalent to ``n-j``; the former is syntactic sugar for the latter, providing a shorthand for indexing elements that would otherwise need to be specified in terms of the axis (dimension) size.

- **Valid** negative indices must reside on the closed interval ``[-n, -1]``.

  .. note::
    This specification does not require bounds checking. The behavior for out-of-bounds integer indices is left unspecified.

- A negative index ``j`` is related to a zero-based nonnegative index ``i`` via ``i = n+j``.

- Colons ``:`` must be used for `slices <https://docs.python.org/3/library/functions.html#slice>`_: ``start:stop:step``, where ``start`` is inclusive and ``stop`` is exclusive.

  .. note::
    The specification does not support returning scalar (i.e., non-array) values from operations, including indexing. In contrast to standard Python indexing rules, for any index, or combination of indices, which select a single value, the result must be a zero-dimensional array containing the selected value.

Slice Syntax
~~~~~~~~~~~~

The basic slice syntax is ``i:j:k`` where ``i`` is the starting index, ``j`` is the stopping index, and ``k`` is the step (``k != 0``). A slice may contain either one or two colons, with either an integer value or nothing on either side of each colon. The following are valid slices.

::

   A[:]
   A[i:]
   A[:j]
   A[i:k]
   A[::]
   A[i::]
   A[:j:]
   A[::k]
   A[i:j:]
   A[i::k]
   A[:j:k]
   A[i::k]
   A[i:j:k]

.. note::
   Slice syntax can be equivalently achieved using the Python built-in `slice() <https://docs.python.org/3/library/functions.html#slice>`_ API. From the perspective of ``A``, the behavior of ``A[i:j:k]`` and ``A[slice(i, j, k)]`` is indistinguishable (i.e., both retrieve the same set of items from ``__getitem__``).

Using a slice to index a single array axis must select ``m`` elements with index values

::

   i, i+k, i+2k, i+3k, ..., i+(m-1)k

where

::

   m = q + r

and ``q`` and ``r`` (``r != 0``) are the quotient and remainder obtained by dividing ``j-i`` by ``k``

::

   j - i = qk + r

such that

::

   j > i + (m-1)k

.. note::
    For ``i`` on the interval ``[0, n)`` (where ``n`` is the axis size), ``j`` on the interval ``(0, n]``, ``i`` less than ``j``, and positive step ``k``, a starting index ``i`` is **always** included, while the stopping index ``j`` is **always** excluded. This preserves ``x[:i]+x[i:]`` always being equal to ``x``.

.. note::
   Using a slice to index into a single array axis should select the same elements as using a slice to index a Python list of the same size.

Slice syntax must have the following defaults. Let ``n`` be the axis (dimension) size.

- If ``k`` is not provided (e.g., ``0:10``), ``k`` must equal ``1``.
- If ``k`` is greater than ``0`` and ``i`` is not provided (e.g., ``:10:2``), ``i`` must equal ``0``.
- If ``k`` is greater than ``0`` and ``j`` is not provided (e.g., ``0::2``), ``j`` must equal ``n``.
- If ``k`` is less than ``0`` and ``i`` is not provided (e.g., ``:10:-2``), ``i`` must equal ``n-1``.
- If ``k`` is less than ``0`` and ``j`` is not provided (e.g., ``0::-2``), ``j`` must equal ``-n-1``.

Using a slice to index a single array axis must adhere to the following rules. Let ``n`` be the axis (dimension) size.

- If ``i`` equals ``j``, a slice must return an empty array, whose axis (dimension) size along the indexed axis is ``0``.

- Indexing via ``:`` and ``::`` must be equivalent and have defaults derived from the rules above. Both ``:`` and ``::`` indicate to select all elements along a single axis (dimension).

  .. note::
    This specification does not require "clipping" out-of-bounds slice indices. This is in contrast to Python slice semantics where ``0:100`` and ``0:10`` are equivalent on a list of length ``10``.

The following ranges for the start and stop values of a slice must be supported. Let ``n`` be the axis (dimension) size being sliced. For a slice ``i:j:k``, the behavior specified above should be implemented for the following:

- ``i`` or ``j`` omitted (``None``).
- ``-n <= i <= n``.
- For ``k > 0`` or ``k`` omitted (``None``), ``-n <= j <= n``.
- For ``k < 0``, ``-n - 1 <= j <= max(0, n - 1)``.

The behavior outside of these bounds is unspecified.

.. note::
   *Rationale: this is consistent with bounds checking for integer indexing; the behavior of out-of-bounds indices is left unspecified. Implementations may choose to clip (consistent with Python* ``list`` *slicing semantics), raise an exception, return junk values, or some other behavior depending on device requirements and performance considerations.*


.. _indexing-multi-axis:

Multi-axis Indexing
-------------------

Multi-dimensional arrays must extend the concept of single-axis indexing to multiple axes by applying single-axis indexing rules along each axis (dimension) and supporting the following additional rules. Let ``N`` be the number of dimensions ("rank") of a multi-dimensional array ``A``.

- Each axis may be independently indexed via single-axis indexing by providing a comma-separated sequence ("selection tuple") of single-axis indexing expressions (e.g., ``A[:, 2:10, :, 5]``).

  .. note::
    In Python, ``A[(exp1, exp2, ..., expN)]`` is equivalent to ``A[exp1, exp2, ..., expN]``; the latter is syntactic sugar for the former.

    Accordingly, if ``A`` has rank ``1``, then ``A[(2:10,)]`` must be equivalent to ``A[2:10]``. If ``A`` has rank ``2``, then ``A[(2:10, :)]`` must be equivalent to ``A[2:10, :]``. And so on and so forth.

- Providing a single nonnegative integer ``i`` as a single-axis index must index the same elements as the slice ``i:i+1``.

- Providing a single negative integer ``i`` as a single-axis index must index the same elements as the slice ``n+i:n+i+1``, where ``n`` is the axis (dimension) size.

- Providing a single integer as a single-axis index must reduce the number of array dimensions by ``1`` (i.e., the array rank must decrease by one; if ``A`` has rank ``2``, ``rank(A)-1 == rank(A[0, :])``). In particular, a selection tuple with the ``m``\th element an integer (and all other entries ``:``) indexes a sub-array with rank ``N-1``.

  .. note::
    When providing a single integer as a single-axis index to an array of rank ``1``, the result should be an array of rank ``0``, not a NumPy scalar. Note that this behavior differs from NumPy.

- Providing a slice must retain array dimensions (i.e., the array rank must remain the same; ``rank(A) == rank(A[:])``).

- Providing `ellipsis <https://docs.python.org/3/library/constants.html#Ellipsis>`_ must apply ``:`` to each dimension necessary to index all dimensions (e.g., if ``A`` has rank ``4``, ``A[1:, ..., 2:5] == A[1:, :, :, 2:5]``). Only a single ellipsis must be allowed. An ``IndexError`` exception must be raised if more than one ellipsis is provided.

- Providing an empty tuple or an ellipsis to an array of rank ``0`` must result in an array of the same rank (i.e., if ``A`` has rank ``0``, ``A == A[()]`` and ``A == A[...]``).

  .. note::
    This behavior differs from NumPy where providing an empty tuple to an array of rank ``0`` returns a NumPy scalar.

- Each ``None`` in the selection tuple must expand the dimensions of the resulting selection by one dimension of size ``1``. The position of the added dimension must be the same as the position of ``None`` in the selection tuple.

  .. note::
    Expanding dimensions can be equivalently achieved via repeated invocation of :func:`~array_api.expand_dims`.

  .. note::
    The constant ``newaxis`` is an alias of ``None`` and can thus be used in a similar manner as ``None``.

- Except in the case of providing a single ellipsis (e.g., ``A[2:10, ...]`` or ``A[1:, ..., 2:5]``), the number of provided single-axis indexing expressions (excluding ``None``) should equal ``N``. For example, if ``A`` has rank ``2``, a single-axis indexing expression should be explicitly provided for both axes (e.g., ``A[2:10, :]``). An ``IndexError`` exception should be raised if the number of provided single-axis indexing expressions (excluding ``None``) is less than ``N``.

  .. note::
    Some libraries, such as SymPy, support flat indexing (i.e., providing a single-axis indexing expression to a higher-dimensional array). That practice is not supported here.

    To perform flat indexing, use ``reshape(x, (-1,))[integer]``.

- An ``IndexError`` exception must be raised if the number of provided single-axis indexing expressions (excluding ``None``) is greater than ``N``.

  .. note::
    This specification leaves unspecified the behavior of providing a slice which attempts to select elements along a particular axis, but whose starting index is out-of-bounds.

    *Rationale: this is consistent with bounds-checking for single-axis indexing. An implementation may choose to set the axis (dimension) size of the result array to* ``0`` *, raise an exception, return junk values, or some other behavior depending on device requirements and performance considerations.*

Integer Array Indexing
----------------------

.. note::
  Integer array indexing, as described in this specification, is a reduced subset of "vectorized indexing" semantics, as implemented in libraries such as NumPy. In vectorized indexing, integers and integer arrays are broadcasted to integer arrays having a common shape before being "zipped" together to form a list of index coordinates. This form of indexing diverges from the multi-axis indexing semantics described above (see :ref:`indexing-multi-axis`) where each element of an indexing tuple comprised of integers and slices independently indexes a particular axis. This latter form of indexing is commonly referred to as "orthogonal indexing" and is the default form of indexing outside of Python in languages such as Julia and MATLAB.

An array must support indexing by an indexing tuple which contains only integers and integer arrays according to the following rules. Let ``A`` be an ``N``-dimensional array with shape ``S1``. Let ``T`` be a tuple ``(t1, t2, ..., tN)`` having length ``N``. Let ``tk`` be an individual element of ``T``.

.. note::
   This specification does not currently address indexing tuples which combine slices and integer arrays. Behavior for such indexing tuples is left unspecified and thus implementation-defined. This may be revisited in a future revision of this standard.

.. note::
   This specification does not currently address indexing tuples which include array-like elements, such as Python lists, tuples, and other sequences. Behavior when indexing an array using array-like elements is left unspecified and thus implementation-defined.

- If ``tk`` is an integer array, ``tk`` should have the default array index data type (see :ref:`data-type-defaults`).

.. note::
   Conforming implementations of this standard may support integer arrays having other integer data types; however, consumers of this standard should be aware that integer arrays having uncommon array index data types such as ``int8`` and ``uint8`` may not be widely supported as index arrays across conforming array libraries. To dynamically resolve the default array index data type, including for that of the current device context, use the inspection API ``default_dtypes()``.

- Providing a zero-dimensional integer array ``tk`` containing an integer index must be equivalent to providing an integer index having the value ``int(tk)``. Conversely, each integer index ``tk`` must be equivalent to a zero-dimensional integer array containing the same value and be treated as such, including shape inference and broadcasting. Accordingly, if ``T`` consists of only integers and zero-dimensional integer arrays, the result must be equivalent to indexing multiple axes using integer indices. For example, if ``A`` is a two-dimensional array, ``T`` is the tuple ``(i, J)``, ``i`` is a valid integer index, and ``J`` is a zero-dimensional array containing a valid integer index ``j``, the result of ``A[T]`` must be equivalent to ``A[(i,j)]`` (see :ref:`indexing-multi-axis`).

- If ``tk`` is an integer array, each element in ``tk`` must independently satisfy the rules stated above for indexing a single-axis with an integer index (see :ref:`indexing-single-axis`).

  .. note::
    This specification does not require bounds checking. The behavior for out-of-bounds integer indices is left unspecified.

- If ``tk`` is an integer array containing duplicate valid integer indices, the result must include the corresponding elements of ``A`` with the same duplication.

  ..
    TODO: once setitem semantics are determined, insert the following note: Given the assignment operation ``x[T] = y[...]``, if ``T`` contains an integer array having duplicate indices, the order in which elements in ``y`` are assigned to the corresponding element(s) in ``x`` is unspecified and thus implementation-defined.

- If ``T`` contains at least one non-zero-dimensional integer array, all elements of ``T`` must be broadcast against each other to determine a common shape ``S2 = (s1, s2, ..., sN)`` according to standard broadcasting rules (see :ref:`broadcasting`). If one or more elements in ``T`` are not broadcast-compatible with the others, an exception must be raised.

- After broadcasting elements of ``T`` to a common shape ``S2``, the resulting tuple ``U = (u1, u2, ..., uN)`` must only contain integer arrays having shape ``S2`` (i.e., ``u1 = broadcast_to(t1, S2)``, ``u2 = broadcast_to(t2, S2)``, et cetera).

- Each element in ``U`` must specify a multi-dimensional index ``v_i = (u1[i], u2[i], ..., uN[i])``, where ``i`` ranges over ``S2``. The result of ``A[U]`` must be constructed by gathering elements from ``A`` at each coordinate tuple ``v_i``. For example, let ``A`` have shape ``(4,4)`` and ``U`` contain integer arrays equivalent to ``([0,1], [2,3])``, with ``u1 = [0,1]`` and ``u2 = [2,3]``. The resulting coordinate tuples must be ``(0,2)`` and ``(1,3)``, respectively, and the resulting array must have shape ``(2,)`` and contain elements ``A[(0,2)]`` and ``A[(1,3)]``.

- The result of ``A[U]`` must be an array having the broadcasted shape ``S2``.

Boolean Array Indexing
----------------------

.. admonition:: Data-dependent output shape
   :class: admonition important

   For common boolean array use cases (e.g., using a dynamically-sized boolean array mask to filter the values of another array), the shape of the output array is data-dependent; hence, array libraries which build computation graphs (e.g., JAX, Dask, etc.) may find boolean array indexing difficult to implement. Accordingly, such libraries may choose to omit boolean array indexing. See :ref:`data-dependent-output-shapes` section for more details.

An array must support indexing where the **sole index** is an ``M``-dimensional boolean array ``B`` with shape ``S1 = (s1, ..., sM)`` according to the following rules. Let ``A`` be an ``N``-dimensional array with shape ``S2 = (s1, ..., sM, ..., sN)``.

  .. note::
     The prohibition against combining boolean array indices with other single-axis indexing expressions includes the use of ``None``. To expand dimensions of the returned array, use repeated invocation of :func:`~array_api.expand_dims`.

- If ``N >= M``, then ``A[B]`` must replace the first ``M`` dimensions of ``A`` with a single dimension having a size equal to the number of ``True`` elements in ``B``. The values in the resulting array must be in row-major (C-style order); this is equivalent to ``A[nonzero(B)]``.

  .. note::
    For example, if ``N == M == 2``, indexing ``A`` via a boolean array ``B`` will return a one-dimensional array whose size is equal to the number of ``True`` elements in ``B``.

- If ``N < M``, then an ``IndexError`` exception must be raised.

- The size of each dimension in ``B`` must equal the size of the corresponding dimension in ``A`` or be ``0``, beginning with the first dimension in ``A``. If a dimension size does not equal the size of the corresponding dimension in ``A`` and is not ``0``, then an ``IndexError`` exception must be raised.

- The elements of a boolean index array must be iterated in row-major, C-style order, with the exception of zero-dimensional boolean arrays.

- A zero-dimensional boolean index array (equivalent to ``True`` or ``False``) must follow the same axis replacement rules stated above. Namely, a zero-dimensional boolean index array removes zero dimensions and adds a single dimension of length ``1`` if the index array's value is ``True`` and of length ``0`` if the index array's value is ``False``. Accordingly, for a zero-dimensional boolean index array ``B``, the result of ``A[B]`` has shape ``S = (1, s1, ..., sN)`` if the index array's value is ``True`` and has shape ``S = (0, s1, ..., sN)`` if the index array's value is ``False``.

Return Values
-------------

The result of an indexing operation (e.g., multi-axis indexing, boolean array indexing, etc) must be an array of the same data type as the indexed array.

.. note::
   The specified return value behavior includes indexing operations which return a single value (e.g., accessing a single element within a one-dimensional array).
