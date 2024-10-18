.. SPDX-FileCopyrightText: 2019-2020 Intel Corporation
..
.. SPDX-License-Identifier: CC-BY-4.0

.. _onemkl_execution_model:

Execution Model
---------------

This section describes the execution environment common to all oneMKL functionality. The execution environment includes how data is provided to computational routines in :ref:`onemkl_queues`, support for several devices in :ref:`onemkl_device_usage`, synchronous and asynchronous execution models in :ref:`onemkl_asynchronous_synchronous_execution` and :ref:`onemkl_host_thread_safety`.  

.. _onemkl_queues:

Use of Queues
+++++++++++++

The ``sycl::queue`` defined in the oneAPI DPC++ specification is used to specify the device and features enabled on that device on which a task will be enqueued.  There are two forms of computational routines in oneMKL: class based :ref:`onemkl_member_functions` and standalone :ref:`onemkl_nonmember_functions`.  As these may interact with the ``sycl::queue`` in different ways, we provide a section for each one to describe assumptions.


.. _onemkl_nonmember_functions:

Non-Member Functions
********************

Each oneMKL non-member computational routine takes a ``sycl::queue`` reference as its first parameter::

    mkl::domain::routine(sycl::queue &q, ...);

All computation performed by the routine shall be done on the hardware device(s) associated with this queue, with possible aid from the host, unless otherwise specified.
In the case of an ordered queue, all computation shall also be ordered with respect to other kernels as if enqueued on that queue.

A particular oneMKL implementation may not support the execution of a given oneMKL routine on the specified device(s). In this case, the implementation may either perform the computation on the host or throw an exception.  See :ref:`onemkl_exceptions` for the possible exceptions.


.. _onemkl_member_functions:

Member Functions
****************

oneMKL class-based APIs, such as those in the RNG and DFT domains, require a ``sycl::queue`` as an argument to the constructor or another setup routine.
The execution requirements for computational routines from the previous section also apply to computational class methods.

.. _onemkl_device_usage:

Device Usage
++++++++++++

oneMKL itself does not currently provide any interfaces for controlling device usage: for instance, controlling the number of cores used on the CPU, or the number of execution units on a GPU. However, such functionality may be available by partitioning a ``sycl::device`` instance into subdevices, when supported by the device.

When given a queue associated with such a subdevice, a oneMKL implementation shall only perform computation on that subdevice.

.. _onemkl_asynchronous_synchronous_execution:

Asynchronous Execution
++++++++++++++++++++++
The oneMKL API is designed to allow asynchronous execution of computational routines, to facilitate concurrent usage of multiple devices in the system. Each computational routine enqueues work to be performed on the selected device, and may (but is not required to) return before execution completes.

Hence, it is the calling application's responsibility to ensure that any inputs are valid until computation is complete, and likewise to wait for computation completion before reading any outputs. This can be done automatically when using DPC++ buffers, or manually when using Unified Shared Memory (USM) pointers, as described in the sections below.

Unless otherwise specified, asynchronous execution is *allowed*, but not *guaranteed*, by any oneMKL computational routine, and may vary between implementations and/or versions. oneMKL implementations must clearly document whether execution is guaranteed to be asynchronous for each supported routine. Regardless, calling applications shall not launch any oneMKL computational routine with a dependency on a future oneMKL API call, even if this computational routine executes asynchronously (i.e. a oneMKL implementation may assume no antidependencies are present). This guarantee allows oneMKL implementations to reserve resources for execution without risking deadlock.

.. _onemkl_synchronization_with_buffers:

Synchronization When Using Buffers
***********************************

``sycl::buffer`` objects automatically manage synchronization between kernel launches linked by a data dependency (either read-after-write, write-after-write, or write-after-read).

oneMKL routines are not required to perform any additional synchronization of ``sycl::buffer`` arguments.

.. _onemkl_synchronization_with_usm:

Synchronization When Using USM APIs
***********************************

When USM pointers are used as input to, or output from, a oneMKL routine, it becomes the calling application's responsibility to manage possible asynchronicity.

To help the calling application, all oneMKL routines with at least one USM pointer argument also take an optional reference to a list of *input events*, of type ``std::vector<sycl::event>``, and have a return value of type ``sycl::event`` representing computation completion::

    sycl::event mkl::domain::routine(..., std::vector<sycl::event> &in_events = {});

The routine shall ensure that all input events (if the list is present and non-empty) have occurred before any USM pointers are accessed. Likewise, the routine's output event shall not be complete until the routine has finished accessing all USM pointer arguments.

For class methods, "argument" includes any USM pointers previously provided to the object via the class constructor or other class methods.

.. _onemkl_host_thread_safety:

Host Thread Safety
++++++++++++++++++

All oneMKL member and non-member functions shall be *host thread safe*. That is, they may be safely called simultaneously from concurrent host threads. However, oneMKL objects in class-based APIs may not be shared between concurrent host threads unless otherwise specified.
