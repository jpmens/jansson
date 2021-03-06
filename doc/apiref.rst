.. _apiref:

*************
API Reference
*************

.. highlight:: c

Preliminaries
=============

All declarations are in :file:`jansson.h`, so it's enough to

::

   #include <jansson.h>

in each source file.

All constants are prefixed ``JSON_`` and other identifiers with
``json_``. Type names are suffixed with ``_t`` and ``typedef``\ 'd so
that the ``struct`` keyword need not be used.


Value Representation
====================

The JSON specification (:rfc:`4627`) defines the following data types:
*object*, *array*, *string*, *number*, *boolean*, and *null*. JSON
types are used dynamically; arrays and objects can hold any other data
type, including themselves. For this reason, Jansson's type system is
also dynamic in nature. There's one C type to represent all JSON
values, and this structure knows the type of the JSON value it holds.

.. type:: json_t

  This data structure is used throughout the library to represent all
  JSON values. It always contains the type of the JSON value it holds
  and the value's reference count. The rest depends on the type of the
  value.

Objects of :type:`json_t` are always used through a pointer. There
are APIs for querying the type, manipulating the reference count, and
for constructing and manipulating values of different types.

Unless noted otherwise, all API functions return an error value if an
error occurs. Depending on the function's signature, the error value
is either *NULL* or -1. Invalid arguments or invalid input are
apparent sources for errors. Memory allocation and I/O operations may
also cause errors.


Type
----

The type of a JSON value is queried and tested using the following
functions:

.. type:: enum json_type

   The type of a JSON value. The following members are defined:

   +--------------------+
   | ``JSON_OBJECT``    |
   +--------------------+
   | ``JSON_ARRAY``     |
   +--------------------+
   | ``JSON_STRING``    |
   +--------------------+
   | ``JSON_INTEGER``   |
   +--------------------+
   | ``JSON_REAL``      |
   +--------------------+
   | ``JSON_TRUE``      |
   +--------------------+
   | ``JSON_FALSE``     |
   +--------------------+
   | ``JSON_NULL``      |
   +--------------------+

   These correspond to JSON object, array, string, number, boolean and
   null. A number is represented by either a value of the type
   ``JSON_INTEGER`` or of the type ``JSON_REAL``. A true boolean value
   is represented by a value of the type ``JSON_TRUE`` and false by a
   value of the type ``JSON_FALSE``.

.. function:: int json_typeof(const json_t *json)

   Return the type of the JSON value (a :type:`json_type` cast to
   :type:`int`). *json* MUST NOT be *NULL*. This function is actually
   implemented as a macro for speed.

.. function:: json_is_object(const json_t *json)
               json_is_array(const json_t *json)
               json_is_string(const json_t *json)
               json_is_integer(const json_t *json)
               json_is_real(const json_t *json)
               json_is_true(const json_t *json)
               json_is_false(const json_t *json)
               json_is_null(const json_t *json)

   These functions (actually macros) return true (non-zero) for values
   of the given type, and false (zero) for values of other types and
   for *NULL*.

.. function:: json_is_number(const json_t *json)

   Returns true for values of types ``JSON_INTEGER`` and
   ``JSON_REAL``, and false for other types and for *NULL*.

.. function:: json_is_boolean(const json_t *json)

   Returns true for types ``JSON_TRUE`` and ``JSON_FALSE``, and false
   for values of other types and for *NULL*.


.. _apiref-reference-count:

Reference Count
---------------

The reference count is used to track whether a value is still in use
or not. When a value is created, it's reference count is set to 1. If
a reference to a value is kept (e.g. a value is stored somewhere for
later use), its reference count is incremented, and when the value is
no longer needed, the reference count is decremented. When the
reference count drops to zero, there are no references left, and the
value can be destroyed.

The following functions are used to manipulate the reference count.

.. function:: json_t *json_incref(json_t *json)

   Increment the reference count of *json* if it's not non-*NULL*.
   Returns *json*.

.. function:: void json_decref(json_t *json)

   Decrement the reference count of *json*. As soon as a call to
   :func:`json_decref()` drops the reference count to zero, the value
   is destroyed and it can no longer be used.

Functions creating new JSON values set the reference count to 1. These
functions are said to return a **new reference**. Other functions
returning (existing) JSON values do not normally increase the
reference count. These functions are said to return a **borrowed
reference**. So, if the user will hold a reference to a value returned
as a borrowed reference, he must call :func:`json_incref`. As soon as
the value is no longer needed, :func:`json_decref` should be called
to release the reference.

Normally, all functions accepting a JSON value as an argument will
manage the reference, i.e. increase and decrease the reference count
as needed. However, some functions **steal** the reference, i.e. they
have the same result as if the user called :func:`json_decref()` on
the argument right after calling the function. These functions are
suffixed with ``_new`` or have ``_new_`` somewhere in their name.

For example, the following code creates a new JSON array and appends
an integer to it::

  json_t *array, *integer;

  array = json_array();
  integer = json_integer(42);

  json_array_append(array, integer);
  json_decref(integer);

Note how the caller has to release the reference to the integer value
by calling :func:`json_decref()`. By using a reference stealing
function :func:`json_array_append_new()` instead of
:func:`json_array_append()`, the code becomes much simpler::

  json_t *array = json_array();
  json_array_append_new(array, json_integer(42));

In this case, the user doesn't have to explicitly release the
reference to the integer value, as :func:`json_array_append_new()`
steals the reference when appending the value to the array.

In the following sections it is clearly documented whether a function
will return a new or borrowed reference or steal a reference to its
argument.


Circular References
-------------------

A circular reference is created when an object or an array is,
directly or indirectly, inserted inside itself. The direct case is
simple::

  json_t *obj = json_object();
  json_object_set(obj, "foo", obj);

Jansson will refuse to do this, and :func:`json_object_set()` (and
all the other such functions for objects and arrays) will return with
an error status. The indirect case is the dangerous one::

  json_t *arr1 = json_array(), *arr2 = json_array();
  json_array_append(arr1, arr2);
  json_array_append(arr2, arr1);

In this example, the array ``arr2`` is contained in the array
``arr1``, and vice versa. Jansson cannot check for this kind of
indirect circular references without a performance hit, so it's up to
the user to avoid them.

If a circular reference is created, the memory consumed by the values
cannot be freed by :func:`json_decref()`. The reference counts never
drops to zero because the values are keeping the references to each
other. Moreover, trying to encode the values with any of the encoding
functions will fail. The encoder detects circular references and
returns an error status.


True, False and Null
====================

These values are implemented as singletons, so each of these functions
returns the same value each time.

.. function:: json_t *json_true(void)

   .. refcounting:: new

   Returns the JSON true value.

.. function:: json_t *json_false(void)

   .. refcounting:: new

   Returns the JSON false value.

.. function:: json_t *json_null(void)

   .. refcounting:: new

   Returns the JSON null value.


String
======

Jansson uses UTF-8 as the character encoding. All JSON strings must be
valid UTF-8 (or ASCII, as it's a subset of UTF-8). Normal null
terminated C strings are used, so JSON strings may not contain
embedded null characters. All other Unicode codepoints U+0001 through
U+10FFFF are allowed.

.. function:: json_t *json_string(const char *value)

   .. refcounting:: new

   Returns a new JSON string, or *NULL* on error. *value* must be a
   valid UTF-8 encoded Unicode string.

.. function:: json_t *json_string_nocheck(const char *value)

   .. refcounting:: new

   Like :func:`json_string`, but doesn't check that *value* is valid
   UTF-8. Use this function only if you are certain that this really
   is the case (e.g. you have already checked it by other means).

   .. versionadded:: 1.2

.. function:: const char *json_string_value(const json_t *string)

   Returns the associated value of *string* as a null terminated UTF-8
   encoded string, or *NULL* if *string* is not a JSON string.

.. function:: int json_string_set(const json_t *string, const char *value)

   Sets the associated value of *string* to *value*. *value* must be a
   valid UTF-8 encoded Unicode string. Returns 0 on success and -1 on
   error.

   .. versionadded:: 1.1

.. function:: int json_string_set_nocheck(const json_t *string, const char *value)

   Like :func:`json_string_set`, but doesn't check that *value* is
   valid UTF-8. Use this function only if you are certain that this
   really is the case (e.g. you have already checked it by other
   means).

   .. versionadded:: 1.2


Number
======

The JSON specification only contains one numeric type, "number". The C
programming language has distinct types for integer and floating-point
numbers, so for practical reasons Jansson also has distinct types for
the two. They are called "integer" and "real", respectively. For more
information, see :ref:`rfc-conformance`.

.. type:: json_int_t

   This is the C type that is used to store JSON integer values. It
   represents the widest integer type available on your system. In
   practice it's just a typedef of ``long long`` if your compiler
   supports it, otherwise ``long``.

   Usually, you can safely use plain ``int`` in place of
   ``json_int_t``, and the implicit C integer conversion handles the
   rest. Only when you know that you need the full 64-bit range, you
   should use ``json_int_t`` explicitly.

``JSON_INTEGER_IS_LONG_LONG``

   This is a preprocessor variable that holds the value 1 if
   :type:`json_int_t` is ``long long``, and 0 if it's ``long``. It
   can be used as follows::

       #if JSON_INTEGER_IS_LONG_LONG
       /* Code specific for long long */
       #else
       /* Code specific for long */
       #endif

``JSON_INTEGER_FORMAT``

   This is a macro that expands to a :func:`printf()` conversion
   specifier that corresponds to :type:`json_int_t`, without the
   leading ``%`` sign, i.e. either ``"lld"`` or ``"ld"``. This macro
   is required because the actual type of :type:`json_int_t` can be
   either ``long`` or ``long long``, and :func:`printf()` reuiqres
   different length modifiers for the two.

   Example::

       json_int_t x = 123123123;
       printf("x is %" JSON_INTEGER_FORMAT "\n", x);


.. function:: json_t *json_integer(json_int_t value)

   .. refcounting:: new

   Returns a new JSON integer, or *NULL* on error.

.. function:: json_int_t json_integer_value(const json_t *integer)

   Returns the associated value of *integer*, or 0 if *json* is not a
   JSON integer.

.. function:: int json_integer_set(const json_t *integer, json_int_t value)

   Sets the associated value of *integer* to *value*. Returns 0 on
   success and -1 if *integer* is not a JSON integer.

   .. versionadded:: 1.1

.. function:: json_t *json_real(double value)

   .. refcounting:: new

   Returns a new JSON real, or *NULL* on error.

.. function:: double json_real_value(const json_t *real)

   Returns the associated value of *real*, or 0.0 if *real* is not a
   JSON real.

.. function:: int json_real_set(const json_t *real, double value)

   Sets the associated value of *real* to *value*. Returns 0 on
   success and -1 if *real* is not a JSON real.

   .. versionadded:: 1.1

In addition to the functions above, there's a common query function
for integers and reals:

.. function:: double json_number_value(const json_t *json)

   Returns the associated value of the JSON integer or JSON real
   *json*, cast to double regardless of the actual type. If *json* is
   neither JSON real nor JSON integer, 0.0 is returned.


Array
=====

A JSON array is an ordered collection of other JSON values.

.. function:: json_t *json_array(void)

   .. refcounting:: new

   Returns a new JSON array, or *NULL* on error. Initially, the array
   is empty.

.. function:: size_t json_array_size(const json_t *array)

   Returns the number of elements in *array*, or 0 if *array* is NULL
   or not a JSON array.

.. function:: json_t *json_array_get(const json_t *array, size_t index)

   .. refcounting:: borrow

   Returns the element in *array* at position *index*. The valid range
   for *index* is from 0 to the return value of
   :func:`json_array_size()` minus 1. If *array* is not a JSON array,
   if *array* is *NULL*, or if *index* is out of range, *NULL* is
   returned.

.. function:: int json_array_set(json_t *array, size_t index, json_t *value)

   Replaces the element in *array* at position *index* with *value*.
   The valid range for *index* is from 0 to the return value of
   :func:`json_array_size()` minus 1. Returns 0 on success and -1 on
   error.

.. function:: int json_array_set_new(json_t *array, size_t index, json_t *value)

   Like :func:`json_array_set()` but steals the reference to *value*.
   This is useful when *value* is newly created and not used after
   the call.

   .. versionadded:: 1.1

.. function:: int json_array_append(json_t *array, json_t *value)

   Appends *value* to the end of *array*, growing the size of *array*
   by 1. Returns 0 on success and -1 on error.

.. function:: int json_array_append_new(json_t *array, json_t *value)

   Like :func:`json_array_append()` but steals the reference to
   *value*. This is useful when *value* is newly created and not used
   after the call.

   .. versionadded:: 1.1

.. function:: int json_array_insert(json_t *array, size_t index, json_t *value)

   Inserts *value* to *array* at position *index*, shifting the
   elements at *index* and after it one position towards the end of
   the array. Returns 0 on success and -1 on error.

   .. versionadded:: 1.1

.. function:: int json_array_insert_new(json_t *array, size_t index, json_t *value)

   Like :func:`json_array_insert()` but steals the reference to
   *value*. This is useful when *value* is newly created and not used
   after the call.

   .. versionadded:: 1.1

.. function:: int json_array_remove(json_t *array, size_t index)

   Removes the element in *array* at position *index*, shifting the
   elements after *index* one position towards the start of the array.
   Returns 0 on success and -1 on error.

   .. versionadded:: 1.1

.. function:: int json_array_clear(json_t *array)

   Removes all elements from *array*. Returns 0 on sucess and -1 on
   error.

   .. versionadded:: 1.1

.. function:: int json_array_extend(json_t *array, json_t *other_array)

   Appends all elements in *other_array* to the end of *array*.
   Returns 0 on success and -1 on error.

   .. versionadded:: 1.1


Object
======

A JSON object is a dictionary of key-value pairs, where the key is a
Unicode string and the value is any JSON value.

.. function:: json_t *json_object(void)

   .. refcounting:: new

   Returns a new JSON object, or *NULL* on error. Initially, the
   object is empty.

.. function:: size_t json_object_size(const json_t *object)

   Returns the number of elements in *object*, or 0 if *object* is not
   a JSON object.

   .. versionadded:: 1.1

.. function:: json_t *json_object_get(const json_t *object, const char *key)

   .. refcounting:: borrow

   Get a value corresponding to *key* from *object*. Returns *NULL* if
   *key* is not found and on error.

.. function:: int json_object_set(json_t *object, const char *key, json_t *value)

   Set the value of *key* to *value* in *object*. *key* must be a
   valid null terminated UTF-8 encoded Unicode string. If there
   already is a value for *key*, it is replaced by the new value.
   Returns 0 on success and -1 on error.

.. function:: int json_object_set_nocheck(json_t *object, const char *key, json_t *value)

   Like :func:`json_object_set`, but doesn't check that *key* is
   valid UTF-8. Use this function only if you are certain that this
   really is the case (e.g. you have already checked it by other
   means).

   .. versionadded:: 1.2

.. function:: int json_object_set_new(json_t *object, const char *key, json_t *value)

   Like :func:`json_object_set()` but steals the reference to
   *value*. This is useful when *value* is newly created and not used
   after the call.

   .. versionadded:: 1.1

.. function:: int json_object_set_new_nocheck(json_t *object, const char *key, json_t *value)

   Like :func:`json_object_set_new`, but doesn't check that *key* is
   valid UTF-8. Use this function only if you are certain that this
   really is the case (e.g. you have already checked it by other
   means).

   .. versionadded:: 1.2

.. function:: int json_object_del(json_t *object, const char *key)

   Delete *key* from *object* if it exists. Returns 0 on success, or
   -1 if *key* was not found.


.. function:: int json_object_clear(json_t *object)

   Remove all elements from *object*. Returns 0 on success and -1 if
   *object* is not a JSON object.

   .. versionadded:: 1.1

.. function:: int json_object_update(json_t *object, json_t *other)

   Update *object* with the key-value pairs from *other*, overwriting
   existing keys. Returns 0 on success or -1 on error.

   .. versionadded:: 1.1


The following functions implement an iteration protocol for objects:

.. function:: void *json_object_iter(json_t *object)

   Returns an opaque iterator which can be used to iterate over all
   key-value pairs in *object*, or *NULL* if *object* is empty.

.. function:: void *json_object_iter_at(json_t *object, const char *key)

   Like :func:`json_object_iter()`, but returns an iterator to the
   key-value pair in *object* whose key is equal to *key*, or NULL if
   *key* is not found in *object*. Iterating forward to the end of
   *object* only yields all key-value pairs of the object if *key*
   happens to be the first key in the underlying hash table.

   .. versionadded:: 1.3

.. function:: void *json_object_iter_next(json_t *object, void *iter)

   Returns an iterator pointing to the next key-value pair in *object*
   after *iter*, or *NULL* if the whole object has been iterated
   through.

.. function:: const char *json_object_iter_key(void *iter)

   Extract the associated key from *iter*.

.. function:: json_t *json_object_iter_value(void *iter)

   .. refcounting:: borrow

   Extract the associated value from *iter*.

.. function:: int json_object_iter_set(json_t *object, void *iter, json_t *value)

   Set the value of the key-value pair in *object*, that is pointed to
   by *iter*, to *value*.

   .. versionadded:: 1.3

.. function:: int json_object_iter_set_new(json_t *object, void *iter, json_t *value)

   Like :func:`json_object_iter_set()`, but steals the reference to
   *value*. This is useful when *value* is newly created and not used
   after the call.

   .. versionadded:: 1.3

The iteration protocol can be used for example as follows::

   /* obj is a JSON object */
   const char *key;
   json_t *value;
   void *iter = json_object_iter(obj);
   while(iter)
   {
       key = json_object_iter_key(iter);
       value = json_object_iter_value(iter);
       /* use key and value ... */
       iter = json_object_iter_next(obj, iter);
   }


Encoding
========

This sections describes the functions that can be used to encode
values to JSON. Only objects and arrays can be encoded, since they are
the only valid "root" values of a JSON text.

By default, the output has no newlines, and spaces are used between
array and object elements for a readable output. This behavior can be
altered by using the ``JSON_INDENT`` and ``JSON_COMPACT`` flags
described below. A newline is never appended to the end of the encoded
JSON data.

Each function takes a *flags* parameter that controls some aspects of
how the data is encoded. Its default value is 0. The following macros
can be ORed together to obtain *flags*.

``JSON_INDENT(n)``
   Pretty-print the result, using newlines between array and object
   items, and indenting with *n* spaces. The valid range for *n* is
   between 0 and 32, other values result in an undefined output. If
   ``JSON_INDENT`` is not used or *n* is 0, no newlines are inserted
   between array and object items.

``JSON_COMPACT``
   This flag enables a compact representation, i.e. sets the separator
   between array and object items to ``","`` and between object keys
   and values to ``":"``. Without this flag, the corresponding
   separators are ``", "`` and ``": "`` for more readable output.

   .. versionadded:: 1.2

``JSON_ENSURE_ASCII``
   If this flag is used, the output is guaranteed to consist only of
   ASCII characters. This is achived by escaping all Unicode
   characters outside the ASCII range.

   .. versionadded:: 1.2

``JSON_SORT_KEYS``
   If this flag is used, all the objects in output are sorted by key.
   This is useful e.g. if two JSON texts are diffed or visually
   compared.

   .. versionadded:: 1.2

``JSON_PRESERVE_ORDER``
   If this flag is used, object keys in the output are sorted into the
   same order in which they were first inserted to the object. For
   example, decoding a JSON text and then encoding with this flag
   preserves the order of object keys.

   .. versionadded:: 1.3

The following functions perform the actual JSON encoding. The result
is in UTF-8.

.. function:: char *json_dumps(const json_t *root, size_t flags)

   Returns the JSON representation of *root* as a string, or *NULL* on
   error. *flags* is described above. The return value must be freed
   by the caller using :func:`free()`.

.. function:: int json_dumpf(const json_t *root, FILE *output, size_t flags)

   Write the JSON representation of *root* to the stream *output*.
   *flags* is described above. Returns 0 on success and -1 on error.
   If an error occurs, something may have already been written to
   *output*. In this case, the output is undefined and most likely not
   valid JSON.

.. function:: int json_dump_file(const json_t *json, const char *path, size_t flags)

   Write the JSON representation of *root* to the file *path*. If
   *path* already exists, it is overwritten. *flags* is described
   above. Returns 0 on success and -1 on error.


Decoding
========

This sections describes the functions that can be used to decode JSON
text to the Jansson representation of JSON data. The JSON
specification requires that a JSON text is either a serialized array
or object, and this requirement is also enforced with the following
functions. In other words, the top level value in the JSON text being
decoded must be either array or object.

See :ref:`rfc-conformance` for a discussion on Jansson's conformance
to the JSON specification. It explains many design decisions that
affect especially the behavior of the decoder.

.. type:: json_error_t

   This data structure is used to return information on decoding
   errors from the decoding functions. Its definition is repeated
   here::

      #define JSON_ERROR_TEXT_LENGTH  160

      typedef struct {
          char text[JSON_ERROR_TEXT_LENGTH];
          int line;
      } json_error_t;

   *line* is the line number on which the error occurred, or -1 if
   this information is not available. *text* contains the error
   message (in UTF-8), or an empty string if a message is not
   available.

   The normal usef of :type:`json_error_t` is to allocate it normally
   on the stack, and pass a pointer to a decoding function. Example::

      int main() {
          json_t *json;
          json_error_t error;

          json = json_load_file("/path/to/file.json", 0, &error);
          if(!json) {
              /* the error variable contains error information */
          }
          ...
      }

   Also note that if the decoding succeeded (``json != NULL`` in the
   above example), the contents of ``error`` are unspecified.

   All decoding functions also accept *NULL* as the
   :type:`json_error_t` pointer, in which case no error information
   is returned to the caller.

The following functions perform the actual JSON decoding.

.. function:: json_t *json_loads(const char *input, size_t flags, json_error_t *error)

   .. refcounting:: new

   Decodes the JSON string *input* and returns the array or object it
   contains, or *NULL* on error, in which case *error* is filled with
   information about the error. See above for discussion on the
   *error* parameter. *flags* is currently unused, and should be set
   to 0.

.. function:: json_t *json_loadf(FILE *input, size_t flags, json_error_t *error)

   .. refcounting:: new

   Decodes the JSON text in stream *input* and returns the array or
   object it contains, or *NULL* on error, in which case *error* is
   filled with information about the error. See above for discussion
   on the *error* parameter. *flags* is currently unused, and should
   be set to 0.

.. function:: json_t *json_load_file(const char *path, size_t flags, json_error_t *error)

   .. refcounting:: new

   Decodes the JSON text in file *path* and returns the array or
   object it contains, or *NULL* on error, in which case *error* is
   filled with information about the error. See above for discussion
   on the *error* parameter. *flags* is currently unused, and should
   be set to 0.


Equality
========

Testing for equality of two JSON values cannot, in general, be
achieved using the ``==`` operator. Equality in the terms of the
``==`` operator states that the two :type:`json_t` pointers point to
exactly the same JSON value. However, two JSON values can be equal not
only if they are exactly the same value, but also if they have equal
"contents":

* Two integer or real values are equal if their contained numeric
  values are equal. An integer value is never equal to a real value,
  though.

* Two strings are equal if their contained UTF-8 strings are equal,
  byte by byte. Unicode comparison algorithms are not implemented.

* Two arrays are equal if they have the same number of elements and
  each element in the first array is equal to the corresponding
  element in the second array.

* Two objects are equal if they have exactly the same keys and the
  value for each key in the first object is equal to the value of the
  corresponding key in the second object.

* Two true, false or null values have no "contents", so they are equal
  if their types are equal. (Because these values are singletons,
  their equality can actually be tested with ``==``.)

The following function can be used to test whether two JSON values are
equal.

.. function:: int json_equal(json_t *value1, json_t *value2)

   Returns 1 if *value1* and *value2* are equal, as defined above.
   Returns 0 if they are inequal or one or both of the pointers are
   *NULL*.

   .. versionadded:: 1.2


Copying
=======

Because of reference counting, passing JSON values around doesn't
require copying them. But sometimes a fresh copy of a JSON value is
needed. For example, if you need to modify an array, but still want to
use the original afterwards, you should take a copy of it first.

Jansson supports two kinds of copying: shallow and deep. There is a
difference between these methods only for arrays and objects. Shallow
copying only copies the first level value (array or object) and uses
the same child values in the copied value. Deep copying makes a fresh
copy of the child values, too. Moreover, all the child values are deep
copied in a recursive fashion.

.. function:: json_t *json_copy(json_t *value)

   .. refcounting:: new

   Returns a shallow copy of *value*, or *NULL* on error.

   .. versionadded:: 1.2

.. function:: json_t *json_deep_copy(json_t *value)

   .. refcounting:: new

   Returns a deep copy of *value*, or *NULL* on error.

   .. versionadded:: 1.2
