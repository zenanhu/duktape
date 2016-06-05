==============
Duktape typing
==============

Typing overview
===============

TBD.

Options for representing a value
================================

There are four basic alternatives to representing a value:

* **A tagged type with no heap allocation**.  This is the lowest footprint
  alternative, and memory usage is 8 bytes (for a packed ``duk_tval``) or
  (typically) 16 bytes (for a non-packed ``duk_tval``).  Example: undefined,
  null, boolean, number, pointer.

* **A heap allocated custom struct**.  A tagged value points to a heap
  allocated C struct which is customized for a certain purpose.  Flags in
  the object header allow a base C struct to be extended in certain cases.
  Example: fixed buffer, dynamic buffer, external buffer, string.

* **A heap allocated object**.  A tagged value points to a ``duk_hobject``.
  Because a ``duk_hobject`` has a property table, type specific values can
  be easily added to the property table, but properties have a relatively
  high cost.  Example: plain Ecmascript object.

* **A heap allocated extended object**.  A tagged value points to a struct
  extending ``duk_hobject``.  Flags in the shared ``duk_hobject`` header
  allow Duktape internals to detect the extended type and to access further
  fields in an extended C struct.  The extended values may only be available
  internally, but may also be accessible via property reads if the properties
  are virtualized.  Example: Ecmascript function, Duktape/C function, thread,
  buffer object.
