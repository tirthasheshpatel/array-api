.. _data-interchange:

Data interchange mechanisms
===========================

This section discusses the mechanism to convert one type of array into another.
As discussed in the :ref:`assumptions-dependencies <Assumptions>` section,
*functions* provided by an array library are not expected to operate on
*array types* implemented by another library. Instead, the array can be
converted to a "native" array type.

The interchange mechanism must offer the following:

1. Data access via a protocol that describes the memory layout of the array
   in an implementation-independent manner.

   *Rationale: any number of libraries must be able to exchange data, and no
   particular package must be needed to do so.*

2. Support for all dtypes in this API standard (see :ref:`data-types`).

3. Device support. It must be possible to determine on what device the array
   that is to be converted lives.

   *Rationale: there are CPU-only, GPU-only, and multi-device array types;
   it's best to support these with a single protocol (with separate
   per-device protocols it's hard to figure out unambiguous rules for which
   protocol gets used, and the situation will get more complex over time
   as TPU's and other accelerators become more widely available).*

4. Zero-copy semantics where possible, making a copy only if needed (e.g.
   when data is not contiguous in memory).

   *Rationale: performance.*

5. A Python-side and a C-side interface, the latter with a stable C ABI.

   *Rationale: all prominent existing array libraries are implemented in
   C/C++, and are released independently from each other. Hence a stable C
   ABI is required for packages to work well together.*

The best candidate for this protocol is
`DLPack <https://github.com/dmlc/dlpack>`_, and hence that is what this
standard has chosen as the primary/recommended protocol. Note that the
``asarray`` function also supports the Python buffer protocol (CPU-only) to
support libraries that already implement buffer protocol support.

.. note::
   The main alternatives to DLPack are device-specific methods:

   - The `buffer protocol <https://docs.python.org/dev/c-api/buffer.html>`_ on CPU
   - ``__cuda_array_interface__`` for CUDA, specified in the Numba documentation
     `here <https://numba.pydata.org/numba-doc/0.43.0/cuda/cuda_array_interface.html>`_
     (Python-side only at the moment)

   An issue with device-specific protocols are: if two libraries both
   support multiple device types, in which order should the protocols be
   tried? A growth in the number of protocols to support each time a new
   device gets supported by array libraries (e.g. TPUs, AMD GPUs, emerging
   hardware accelerators) also seems undesirable.

   In addition to the above argument, it is also clear from adoption
   patterns that DLPack has the widest support. The buffer protocol, despite
   being a lot older and standardized as part of Python itself via PEP 3118,
   hardly has any support from array libraries. CPU interoperability is
   mostly dealt with via the NumPy-specific ``__array__`` (which, when called,
   means the object it is attached to must return a ``numpy.ndarray``
   containing the data the object holds).

   See the `RFC to adopt DLPack <https://github.com/data-apis/consortium-feedback/issues/1>`_
   for discussion that preceded the adoption of DLPack.


DLPack support
--------------

.. note::
   DLPack is a standalone protocol/project and can therefore be used outside of
   this standard. Python libraries that want to implement only DLPack support
   are recommended to do so using the same syntax and semantics as outlined
   below. They are not required to return an array object from ``from_dlpack``
   which conforms to this standard.

   DLPack itself has no documentation currently outside of the inline comments in
   `dlpack.h <https://github.com/dmlc/dlpack/blob/main/include/dlpack/dlpack.h>`_.
   In the future, the below content may be migrated to the (to-be-written) DLPack docs.


Syntax for data interchange with DLPack
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The array API will offer the following syntax for data interchange:

1. A ``from_dlpack(x)`` function, which accepts (array) objects with a
   ``__dlpack__`` method and uses that method to construct a new array
   containing the data from ``x``.
2. ``__dlpack__(self, stream=None)`` and ``__dlpack_device__`` methods on the
   array object, which will be called from within ``from_dlpack``, to query
   what device the array is on (may be needed to pass in the correct
   stream, e.g. in the case of multiple GPUs) and to access the data.


Semantics
~~~~~~~~~

DLPack describe the memory layout of strided, n-dimensional arrays.
When a user calls ``y = from_dlpack(x)``, the library implementing ``x`` (the
"producer") will provide access to the data from ``x`` to the library
containing ``from_dlpack`` (the "consumer"). If possible, this must be
zero-copy (i.e. ``y`` will be a *view* on ``x``). If not possible, that library
may make a copy of the data. In both cases:

- the producer keeps owning the memory
- ``y`` may or may not be a view, therefore the user must keep the recommendation to avoid mutating ``y`` in mind - see :ref:`copyview-mutability`.
- Both ``x`` and ``y`` may continue to be used just like arrays created in other ways.

If an array that is accessed via the interchange protocol lives on a
device that the requesting library does not support, it is recommended to
raise a ``TypeError``.

Stream handling through the ``stream`` keyword applies to CUDA and ROCm (perhaps
to other devices that have a stream concept as well, however those haven't been
considered in detail). The consumer must pass the stream it will use to the
producer; the producer must synchronize or wait on the stream when necessary.
In the common case of the default stream being used, synchronization will be
unnecessary so asynchronous execution is enabled.


Implementation
~~~~~~~~~~~~~~

*Note that while this API standard largely tries to avoid discussing
implementation details, some discussion and requirements are needed
here because data interchange requires coordination between
implementers on, e.g., memory management.*

.. image:: /_static/images/DLPack_diagram.png
  :alt: Diagram of DLPack structs

*DLPack diagram. Dark blue are the structs it defines, light blue
struct members, gray text enum values of supported devices and data
types.*

The ``__dlpack__`` method will produce a ``PyCapsule`` containing a
``DLManagedTensor``, which will be consumed immediately within
``from_dlpack`` - therefore it is consumed exactly once, and it will not be
visible to users of the Python API.

The producer must set the ``PyCapsule`` name to ``"dltensor"`` so that
it can be inspected by name, and set ``PyCapsule_Destructor`` that calls
the ``deleter`` of the ``DLManagedTensor`` when the ``"dltensor"``-named
capsule is no longer needed.

The consumer must transer ownership of the ``DLManangedTensor`` from the
capsule to its own object. It does so by renaming the capsule to
``"used_dltensor"`` to ensure that ``PyCapsule_Destructor`` will not get
called (ensured if ``PyCapsule_Destructor`` calls ``deleter`` only for
capsules whose name is ``"dltensor"``), but the ``deleter`` of the
``DLManagedTensor`` will be called by the destructor of the consumer
library object created to own the ``DLManagerTensor`` obtained from the
capsule.

Note: the capsule names ``"dltensor"`` and ``"used_dltensor"`` must be
statically allocated.

When the ``strides`` field in the ``DLTensor`` struct is ``NULL``, it indicates a
row-major compact array. If the array is of size zero, the data pointer in
``DLTensor`` should be set to either ``NULL`` or ``0``.

DLPack version used must be ``0.2 <= DLPACK_VERSION < 1.0``. For further
details on DLPack design and how to implement support for it,
refer to `github.com/dmlc/dlpack <https://github.com/dmlc/dlpack>`_.

.. warning::
   DLPack contains a ``device_id``, which will be the device
   ID (an integer, ``0, 1, ...``) which the producer library uses. In
   practice this will likely be the same numbering as that of the
   consumer, however that is not guaranteed. Depending on the hardware
   type, it may be possible for the consumer library implementation to
   look up the actual device from the pointer to the data - this is
   possible for example for CUDA device pointers.

   It is recommended that implementers of this array API consider and document
   whether the ``.device`` attribute of the array returned from ``from_dlpack`` is
   guaranteed to be in a certain order or not.
