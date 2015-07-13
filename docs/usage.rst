.. _usage:

Cerberus Usage
==============

Basic Usage
-----------
You define a validation schema and pass it to an instance of the
:class:`~cerberus.Validator` class: ::

    >>> schema = {'name': {'type': 'string'}}
    >>> v = Validator(schema)

Then you simply invoke the :func:`~cerberus.Validator.validate` to validate
a dictionary against the schema. If validation succeeds, ``True`` is returned:

::

    >>> document = {'name': 'john doe'}
    >>> v.validate(document)
    True

Alternatively, you can pass both the dictionary and the schema to the
:func:`~cerberus.Validator.validate` method: ::

    >>> v = Validator()
    >>> v.validate(document, schema)
    True

Which can be handy if your schema is changing through the life of the
instance.

Unlike other validation tools, Cerberus will not halt and raise an exception on
the first validation issue. The whole document will always be processed, and
``False`` will be returned if validation failed.  You can then access the
:func:`~cerberus.Validator.errors` method to obtain a list of issues.  ::

    >>> schema = {'name': {'type': 'string'}, 'age': {'type': 'integer', 'min': 10}}
    >>> document = {'name': 1337, 'age': 5}
    >>> v.validate(document, schema)
    False
    >>> v.errors
    {'age': 'min value is 10', 'name': 'must be of string type'}

You will still get :class:`~cerberus.SchemaError` and
:class:`~cerberus.ValidationError` exceptions.

.. versionchanged:: 0.4.1
    The Validator class is callable, allowing for the following shorthand
    syntax:

::

    >>> document = {'name': 'john doe'}
    >>> v(document)
    True


Validation Schema
-----------------
A validation schema is a dictionary. Schema keys are the keys allowed in
the target dictionary. Schema values express the rules that must be  matched by
the corresponding target values. ::

    >>> schema = {'name': {'type': 'string', 'maxlength': 10}}

In the example above we define a target dictionary with only one key, ``name``,
which is expected to be a string not longer than 10 characters. Something like
``{'name': 'john doe'}`` would validate, while something like ``{'name': 'a
very long string'}`` or ``{'name': 99}`` would not.

By definition all keys are optional unless the `required`_ rule is set for
a key.

Validation Rules
----------------
The following rules are currently supported:

.. _type:

type
~~~~
Data type allowed for the key value. Can be one of the following:
    * ``string``
    * ``integer``
    * ``float``
    * ``number`` (integer or float)
    * ``boolean``
    * ``datetime``
    * ``dict`` (formally ``collections.mapping``)
    * ``list`` (formally ``collections.sequence``, excluding strings)
    * ``set``

A list of types can be used to allow different values: ::

    >>> v = Validator({'quotes': {'type': ['string', 'list']}})
    >>> v.validate({'quotes': 'Hello world!'})
    True
    >>> v.validate({'quotes': ['Do not disturb my circles!', 'Heureka!']})
    True

    >>> v = Validator({'quotes': {'type': ['string', 'list'], 'schema': {'type': 'string'}}})
    >>> v.validate({'quotes': 'Hello world!'})
    True
    >>> v.validate({'quotes': [1, 'Heureka!']})
    False
    >>> v.errors
    {'quotes': {0: 'must be of string type'}}

You can extend this list and support custom types, see :ref:`new-types`.

.. note::

    Please note that type validation is performed before any other validation
    rule which might exist on the same field (only exception being the
    ``nullable`` rule). In the occurrence of a type failure subsequent
    validation rules on the field will be skipped and validation will continue
    on other fields. This allows to safely assume that field type is correct
    when other (standard or custom) rules are invoked.

.. versionchanged:: 0.9
   If a list of types is given, the key value must match *any* of them.

.. versionchanged:: 0.7.1
   ``dict`` and ``list`` typechecking are now performed with the more generic
   ``Mapping`` and ``Sequence`` types from the builtin ``collections`` module.
   This means that instances of custom types designed to the same interface as
   the builtin ``dict`` and ``list`` types can be validated with Cerberus. We
   exclude strings when type checking for ``list``/``Sequence`` because it
   in the validation situation it is almost certain the string was not the
   intended data type for a sequence.

.. versionchanged:: 0.7
   Added the ``set`` data type.

.. versionchanged:: 0.6
   Added the ``number`` data type.

.. versionchanged:: 0.4.0
   Type validation is always executed first, and blocks other field validation
   rules on failure.

.. versionchanged:: 0.3.0
   Added the ``float`` data type.

required
~~~~~~~~
If ``True`` the key/value pair is mandatory. Validation will fail when it is
missing, unless :func:`~cerberus.Validator.validate` is called with
``update=True``:

::

    >>> schema = {'name': {'required': True, 'type': 'string'}, 'age': {'type': 'integer'}}
    >>> v = Validator(schema)
    >>> document = {'age': 10}
    >>> v.validate(document)
    False
    >>> v.errors
    {'name': 'must be of string type'}

    >>> v.validate(document, update=True)
    True

.. note::

   String fields with empty values will still be validated, even when
   ``required`` is set to ``True``. If you don't want to accept empty values,
   see the empty_ rule. Also, if dependencies_ are declared for the field, its
   ``required`` rule will only be validated if all dependencies are
   included with the document.

.. versionchanged:: 0.8
   Check field dependencies.

readonly
~~~~~~~~
If ``True`` the value is readonly. Validation will fail if this field is present
in the target dictionary.

nullable
~~~~~~~~
If ``True`` the field value can be set to ``None``. It is essentially the
functionality of the ``ignore_none_values`` parameter of the :ref:`validator`,
but allowing for more fine grained control down to the field level. ::

    >>> schema = {'a_nullable_integer': {'nullable': True, 'type': 'integer'}, 'an_integer': {'type': 'integer'}}
    >>> v = Validator(schema)

    >>> v.validate({'a_nullable_integer': 3})
    True
    >>> v.validate({'a_nullable_integer': None})
    True

    >>> v.validate({'an_integer': 3})
    True
    >>> v.validate({'an_integer': None})
    False
    >>> v.errors
    {'an_integer': 'must be of integer type'}

.. versionchanged:: 0.7 ``nullable`` is valid on fields lacking type definition.
.. versionadded:: 0.3.0

minlength, maxlength
~~~~~~~~~~~~~~~~~~~~
Minimum and maximum length allowed for ``string`` and ``list`` types.

min, max
~~~~~~~~
Minimum and maximum value allowed for ``integer``, ``float`` and ``number``
types.

.. versionchanged:: 0.7
   Added support for ``float`` and ``number`` types.

allowed
~~~~~~~
Allowed values for ``string``, ``list`` and ``int`` types. Validation will fail
if target values are not included in the allowed list.::

    >>> schema = {'role': {'type': 'list', 'allowed': ['agent', 'client', 'supplier']}}
    >>> v = Validator(schema)
    >>> v.validate({'role': ['agent', 'supplier']})
    True

    >>> v.validate({'role': ['intern']})
    False
    >>> v.errors
    {'role': "unallowed values ['intern']"}

    >>> schema = {'role': {'type': 'string', 'allowed': ['agent', 'client', 'supplier']}}
    >>> v = Validator(schema)
    >>> v.validate({'role': 'supplier'})
    True

    >>> v.validate({'role': 'intern'})
    False
    >>> v.errors
    {'role': 'unallowed value intern'}

    >>> schema = {'a_restricted_integer': {'type': 'integer', 'allowed': [-1, 0, 1]}}
    >>> v = Validator(schema)
    >>> v.validate({'a_restricted_integer': -1})
    True

    >>> v.validate({'a_restricted_integer': 2})
    False
    >>> v.errors
    {'a_restricted_unteger': 'unallowed value 2'}

.. versionchanged:: 0.5.1
   Added support for the ``int`` type.

empty
~~~~~
Only applies to string fields. If ``False`` validation will fail if the value
is empty. Defaults to ``True``. ::

    >>> schema = {'name': {'type': 'string', 'empty': False}}
    >>> document = {'name': ''}
    >>> v.validate(document, schema)
    False

    >>> v.errors
    {'name': 'empty values not allowed'}

.. versionadded:: 0.0.3

.. _items_dict:

items (dict)
~~~~~~~~~~~~
.. deprecated:: 0.0.3
   Use :ref:`schema` instead.

When a dictionary, ``items`` defines the validation schema for items in
a ``list`` type: ::

    >>> schema = {'rows': {'type': 'list', 'items': {'sku': {'type': 'string'}, 'price': {'type': 'integer'}}}}
    >>> document = {'rows': [{'sku': 'KT123', 'price': 100}]}
    >>> v.validate(document, schema)
    True

.. note::

    The :ref:`items_dict` rule is deprecated, and will be removed in a future release.

items (list)
~~~~~~~~~~~~
When a list, ``items`` defines a list of values allowed in a ``list`` type of
fixed length in the given order: ::

    >>> schema = {'list_of_values': {'type': 'list', 'items': [{'type': 'string'}, {'type': 'integer'}]}}
    >>> document = {'list_of_values': ['hello', 100]}
    >>> v.validate(document, schema)
    True
    >>> document = {'list_of_values': [100, 'hello']}
    >>> v.validate(document, schema)
    False

See :ref:`schema` rule below for dealing with arbitrary length ``list`` types.

.. _schema:

schema (dict)
~~~~~~~~~~~~~
Validation rules for ``dict`` fields. ::

    >>> schema = {'a_dict': {'type': 'dict', 'schema': {'address': {'type': 'string'}, 'city': {'type': 'string', 'required': True}}}}
    >>> document = {'a_dict': {'address': 'my address', 'city': 'my town'}}
    >>> v.validate(document, schema)
    True

.. note::

    If all keys should share the same validation rules you probably want to use :ref:`valueschema` instead.

schema (list)
~~~~~~~~~~~~~
You can also use this rule to validate arbitrary length ``list`` items. ::

    >>> schema = {'a_list': {'type': 'list', 'schema': {'type': 'integer'}}}
    >>> document = {'a_list': [3, 4, 5]}
    >>> v.validate(document, schema)
    True

The `schema` rule on ``list`` types is also the prefered method for defining
and validating a list of dictionaries. ::

    >>> schema = {'rows': {'type': 'list', 'schema': {'type': 'dict', 'schema': {'sku': {'type': 'string'}, 'price': {'type': 'integer'}}}}}
    >>> document = {'rows': [{'sku': 'KT123', 'price': 100}]}
    >>> v.validate(document, schema)
    True

.. versionchanged:: 0.0.3
   Schema rule for ``list`` types of arbitrary length

.. _valueschema:

valueschema
~~~~~~~~~~~
Validation schema for all values of a ``dict``. The ``dict`` can have arbitrary
keys, the values for all of which must validate with given schema: ::

    >>> schema = {'numbers': {'type': 'dict', 'valueschema': {'type': 'integer', min: 10}}}
    >>> document = {'numbers': {'an integer': 10, 'another integer': 100}}
    >>> v.validate(document, schema)
    True

    >>> document = {'numbers': {'an integer': 9}}
    >>> v.validate(document, schema)
    False

    >>> v.errors
    {'numbers': {'an integer': 'min value is 10'}}

.. versionadded:: 0.7
.. versionchanged:: 0.9
   renamed ``keyschema`` to ``valueschema``

propertyschema
~~~~~~~~~~~~~~

This is the counterpart to ``valueschema`` that validates the `keys` of a ``dict``. For historical reasons
it is `not` named ``keyschema``. ::

    >>> schema = 'a_dict': {'type': 'dict', 'propertyschema': {'type': 'string', 'regex': '[a-z]+'}}
    >>> document = {'a_dict': {'key': 'value'}}
    >>> v.validate(document, schema)
    True

    >>> document = {'a_dict': {'KEY': 'value'}}
    >>> v.validate(document, schema)
    False

.. versionadded:: 0.9

regex
~~~~~
Validation will fail if field value does not match the provided regex rule. Only applies to string fiels. ::

    >>> schema = {'email': {'type': 'string', 'regex': '^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$'}}
    >>> document = {'email': 'john@example.com'}
    >>> v.validate(document, schema)
    True

    >>> document = {'email': 'john_at_example_dot_com'}
    >>> v.validate(document, schema)
    False

    >>> v.errors
    {'email': 'value does not match regex "^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$"}

For details on regex rules, see `Regular Expressions Syntax`_ on Python official site.

.. versionadded:: 0.7

dependencies
~~~~~~~~~~~~
This rule allows for either a list or dict of dependencies. When a list is
provided, all listed fields must be present in order for the target field to be
validated. ::

    >>> schema = {'field1': {'required': False}, 'field2': {'required': False, 'dependencies': ['field1']}}
    >>> document = {'field1': 7}
    >>> v.validate(document, schema)
    True

    >>> document = {'field2': 7}
    >>> v.validate(document, schema)
    False

    >>> v.errors
    {'field2': 'field "field1" is required'}

When a dictionary is provided, then not only all dependencies must be present,
but also any of their allowed values must be matched. ::

    >>> schema = {'field1': {'required': False}, 'field2': {'required': True, 'dependencies': {'field1': ['one', 'two']}}}
    >>> document = {'field1': 'one', 'field2': 7}
    >>> v.validate(document, schema)
    True

    >>> document = {'field1': 'three', 'field2': 7}
    False

    >>> v.errors
    {'field2': "field 'field1' is required with values: ['one', 'two']"}


    >>> # same as using a dependencies list
    >>> document = {'field2': 7}
    >>> v.validate(document, schema)
    {'field2': "field 'field1' is required"}

    >>> # one can also pass a single dependency value
    >>> schema = {'field1': {'required': False}, 'field2': {'dependencies': {'field1': 'one'}}}
    >>> document = {'field1': 'one', 'field2': 7}
    >>> v.validate(document, schema)
    True

    >>> document = {'field1': 'two', 'field2': 7}
    False

    >>> v.errors
    {'field2': "field 'field1' is required with values: one"}

Dependencies on sub-document fields are also supported: ::

    >>> schema = {
    ...   'test_field': {'dependencies': ['a_dict.foo', 'a_dict.bar']},
    ...   'a_dict': {
    ...     'type': 'dict',
    ...     'schema': {
    ...       'foo': {'type': 'string'},
    ...       'bar': {'type': 'string'}
    ...     }
    ...   }
    ... }

    >>> document = {'test_field': 'foobar', 'a_dict': {'foo': 'foo'}}
    >>> v.validate(document, schema)
    False

    >>> v.errors
    {'test_field': "field 'a_dict.bar' is required"}

.. versionchanged:: 0.8.1 Support for sub-document fields as dependencies.

.. versionchanged:: 0.8 Support for dependencies as a dictionary.

.. versionadded:: 0.7

anyof
~~~~~
This rule allows you to list multiple sets of rules to validate against. The field will be considered valid if it validates against one set in the list.
For example, to verify that a property is a number between 0 and 10 or 100 and 110, you could do the following: ::

    >>> schema = {'prop1':
    ...           {'type': 'number',
    ...            'anyof':
    ...            [{'min': 0, 'max': 10}, {'min': 100, 'max': 110}]}}
    >>> doc = {'prop1': 5}
    >>> v.validate(document, schema)
    True
    >>> doc = {'prop1': 105}
    >>> v.validate(document, schema)
    True
    >>> doc = {'prop1': 55}
    >>> v.validate(document, schema)
    False
    >>> print v.errors
    {'prop1': {'anyof': 'no definitions validated', 'definition 1': 'min value is 100', 'definition 0': 'max value is 10'}}

.. versionadded:: 0.9

The `anyof` rule works by creating a new instance of a schema for each item in the list. The above schema is equivalent to creating two separate schemas, ::

    >>> schema1 = {'prop1': {'type': 'number', 'min':   0, 'max':  10}}
    >>> schema2 = {'prop1': {'type': 'number', 'min': 100, 'max': 110}}
    >>> doc = {'prop1': 5}
    >>> valid = v.validate(document, schema1) or v.validate(document, schema2)
    >>> valid
    True
    >>> doc = {'prop1': 105}
    >>> valid = v.validate(document, schema1) or v.validate(document, schema2)
    >>> valid
    True
    True
    >>> doc = {'prop1': 55}
    >>> valid = v.validate(document, schema1) or v.validate(document, schema2)
    >>> valid
    False

allof
~~~~~
Same as ``anyof``, except that all rule collections in the list must validate.

.. versionadded:: 0.9

noneof
~~~~~~
Same as ``anyof``, except that it requires no rule collections in the list to validate.

.. versionadded:: 0.9

oneof
~~~~~
Same as ``anyof``, except that only one rule collections in the list can validate.

.. versionadded:: 0.9

Allowing the Unknown
--------------------
By default only keys defined in the schema are allowed: ::

    >>> schema = {'name': {'type': 'string', 'maxlength': 10}}
    >>> v.validate({'name': 'john', 'sex': 'M'})
    False
    >>> v.errors
    {'sex': 'unknown field'}

However, you can allow unknown key/value pairs by either setting
``allow_unknown`` to ``True``: ::

    >>> v = Validator(schema={})
    >>> v.allow_unknown = True
    >>> v.validate({'name': 'john', 'sex': 'M'})
    True

Or you can set ``allow_unknown`` to a validation schema, in which case
unknown fields will be validated against it: ::

    >>> v = Validator(schema={})
    >>> v.allow_unknown = {'type': 'string'}
    >>> v.validate({'an_unknown_field': 'john'})
    True
    >>> v.validate({'an_unknown_field': 1})
    False
    >>> v.errors
    {'an_unknown_field': 'must be of string type'}

``allow_unknown`` can also be set at initialization: ::

    >>> v = Validator(schema=schema, allow_unknown=True)
    >>> v.validate({'name': 'john', 'sex': 'M'})
    True

``allow_unknown`` can also be set for nested dictionaries ::

    >>> # by default allow_unknown is False for the whole document.
    >>> v = Validator()
    >>> v.allow_unknown
    False

    >>> # we can switch it on (or set it to a validation schema) for individual subdocuments
    >>> schema = {
    ...   'name': {'type': 'string'},
    ...   'a_dict': {
    ...     'type': 'dict',
    ...     'allow_unknown': True,
    ...     'schema': {
    ...       'address': {'type': 'string'}
    ...     }
    ...   }
    ... }

    >>> v.validate({'name': 'john', 'a_dict':{'an_unknown_field': 'is allowed'}}, schema)
    True

    >>> # this fails as allow_unknown is still False for the parent document.
    >>> v.validate({'name': 'john', 'an_unknown_field': 'is not allowed', 'a_dict':{'an_unknown_field': 'is allowed'}}, schema)
    False

    >>> v.errors
    {'an_unknown_field': 'unknown field'}

.. versionchanged:: 0.9
   ``allow_unknown`` can also be set for nested dict fields.

.. versionchanged:: 0.8
   ``allow_unknown`` can also be set to a validation schema.

.. _type-coercion:

Type Coercion
-------------
Type coercion allows you to apply a callable to a value before any other
validators run.  The return value of the callable replaces the new value in
the document.  This can be used to convert values or sanitize data before it is
validated. ::

    >>> v = Validator({'amount': {'type': 'integer'}})
    >>> v.validate({'amount': '1'})
    False

    >>> v = Validator({'amount': {'type': 'integer', 'coerce': int}})
    >>> v.validate({'amount': '1'})
    True
    >>> v.document
    {'amount': 1}

    >>> to_bool = lambda v: v.lower() in ['true', '1']
    >>> v = Validator({'flag': {'type': 'boolean', 'coerce': to_bool}})
    >>> v.validate({'flag': 'true'})
    True
    >>> v.document
    {'flag': True}

.. versionadded:: 0.9

`validated` Method
------------------
There's a wrapper-method ``validated`` that returns the validated document. It
can be useful for flows like this: ::

    >>> v = Validator(schema)
    >>> valid_documents = [x for x in [v.validated(y) for y in documents] if x is not None]

If a coercion callable raises a ``TypeError`` or ``ValueError`` then the
exception will be caught and the validation with fail.  All other exception
pass through.

.. versionadded:: 0.9

`normalized` Method
-------------------
Similary, the ``normalized``-method returns a normalized copy of a document without validating it: ::

    >>> v = Validator()
    >>> norm_doc = v.normalized(doc, schema)

.. versionadded:: FIXME

Vanilla Python
--------------
Cerberus schemas are built with vanilla Python types: `dict`, `list`, `string`, etc. Even user-defined validation rules are invoked in the schema by name, as a string.
A useful side effect of this design is that schemas can be defined in a number of ways, for example with YAML. ::

    >>> import yaml
    >>> schema_text = '''
    ...name:
    ...  type: string
    ...age':
    ...  type: integer
    ...  min: 10
    ...'''
    >>> schema = yaml.load(schema_text)
    >>> document = {'name': 1337, 'age': 5}
    >>> v.validate(document, schema)
    False
    >>> v.errors
    {'age': 'min value is 10', 'name': 'must be of string type'}

You don't have to use YAML of course, you can use your favorate serializer. JSON for example. As long as there is a decoder thant can produce a nested `dict`, you
can use it to define a schema.


.. _`Regular Expressions Syntax`: https://docs.python.org/2/library/re.html#regular-expression-syntax