Basic Usage
===========

In this chapter, we'll walk through basic usage of Deform to render a
form, and capture and validate input.

The steps a developer must take to cause a form to be renderered and
subsequently be ready to accept form submission input are:

- Define a schema

- Create a form object.

- Assign non-default widgets to fields in the form (optional).

- Render the form.

Once the form is rendered, a user will interact with the form in his
browser, and some point, he will submit it.  When the user submits the
form, the data provided by the user will either validate properly, or
the form will need to be rerendered with error markers which help to
inform the user of which parts need to be filled in "properly" (as
defined by the schema).  We allow the user to coninue filling in the
form, submitting, and revalidating indefinitely.

Defining A Schema
-----------------

The first step to using Deform is to create a :term:`schema` which
represents the data structure you wish to be captured via a form
rendering.  

For example, let's imagine you want to create a form based roughly on
a data structure you'll obtain by reading data from a relational
database.  An example of such a data structure might look something
like this:

.. code-block:: python
   :linenos:

   [
   {
    'name':'keith',
    'age':20,
   },
   {
    'name':'fred',
    'age':23,
   },
   ]

In other words, the database query we make returns a sequence of
*people*; each person is represented by some data.  We need to edit
this data.  There won't be many people in this list, so we don't need
any sort of paging or batching to make our way through the list; we
can display it all on one form page.

The name that Deform has a structure like the above is an
:term:`appstruct`.  The term "appstruct" is shorthand for "application
structure", because it's the kind of high-level structure that an
application usually cares about: the data present in an appstruct is
useful directly to an application itself.

.. note:: An appstruct differs from other structures that Deform uses
   (such as :term:`pstruct` and :term:`cstruct` structures): pstructs
   and cstructs are typically only useful during intermediate parts of
   the rendering process.

Usually, given some appstruct, you can divine a :term:`schema` that
would allow you to edit the data related to the appstruct.  Let's
define a schema which will attempt to serialize this particular
appstruct to a form.  Our application has these requirements of the
resulting form:

- It must be possible to add and remove a person.

- It must be possible to change any person's name or age after they've
  been added.

Here's a schema that will help us meet those requirements:

.. code-block:: python
   :linenos:

   import colander

   class Person(colander.MappingSchema):
       name = colander.SchemaNode(colander.String())
       age = colander.SchemaNode(colander.Integer(),
                                 validator=colander.Range(0, 200))

   class People(colander.SequenceSchema):
       person = Person()

   class Schema(colander.MappingSchema):
       people = People()

   schema = Schema()
       
The schemas used by Deform come from a package named :term:`Colander`.
The canonical documentation for Colander exists at
`http://docs.repoze.org/colander <http://docs.repoze.org/colander>`_.
To compose complex schemas, you'll need to read it to get comfy with
the documentation of the default Colander data types.  But for now, we
can play it by ear.

For ease of reading, we've actually defined *three* schemas above, but
we coalesce them all into a single schema instance as ``schema`` in
the last step.  A ``People`` schema is a collection of ``Person``
schema nodes.  As the result of our definitions, a ``Person``
represents:

- A ``name``, which must be a string.

- An ``age``, which must be deserializable to an integer; after
  deserialization happens, a validator ensures that the integer is
  between 0 and 200 inclusive.

Schema Node Objects
~~~~~~~~~~~~~~~~~~~

.. note:: This section repeats and contextualizes the :term:`Colander`
   documentation about schema nodes in order to prevent you from
   needing to switch away from this page to another while trying to
   learn about forms.  But you can also get much the same information
   at `http://docs.repoze.org/colander
   <http://docs.repoze.org/colander>`_

A schema is composed of one or more *schema node* objects, each
typically of the class :class:`colander.SchemaNode`, usually in a nested
arrangement.  Each schema node object has a required *type*, an
optional *validator*, an optional *default*, an optional *title*, an
optional *description*, and a slightly less optional *name*.

The *type* of a schema node indicates its data type (such as
:class:`colander.Int` or :class:`colander.String`).

The *validator* of a schema node is called after deserialization; it
makes sure the deserialized value matches a constraint.  An example of
such a validator is provided in the schema above:
``validator=colander.Range(0, 200)``.  A validator is not called after
schema node serialization, only after node deserialization.

The *default* of a schema node indicates its default value if a value
for the schema node is not found in the input data during
serialization.  It should be the *deserialized* representation.  If a
schema node does not have a default, it is considered a required
schema node.

The *name* of a schema node is used to relate schema nodes to each
other.  It is also used as the title if a title is not provided.

The *title* of a schema node is metadata about a schema node.  It
shows up in the legend above the form field(s) related to the schema
node.  By default, it is a capitalization of the *name*.

The *description* of a schema node is metadata about a schema node.
It shows up as a tooltip when someone hovers over one a form field.
By default, it is empty.

The name of a schema node that is introduced as a class-level
attribute of a :class:`colander.MappingSchema`,
:class:`colander.TupleSchema` or a :class:`colander.SequenceSchema` is
its class attribute name.  For example:

.. code-block:: python
   :linenos:

   import colander

   class Phone(colander.MappingSchema):
       location = colander.SchemaNode(colander.String(), 
                                      validator=colander.OneOf(['home','work']))
       number = colander.SchemaNode(colander.String())

The name of the schema node defined via ``location =
colander.SchemaNode(..)`` within the schema above is ``location``.
The title of the same schema node is ``Location``.

Schema Objects
~~~~~~~~~~~~~~

In the examples above, if you've been paying attention, you'll have
noticed that we're defining classes which subclass from
:class:`colander.MappingSchema`, and :class:`colander.SequenceSchema`.
It's turtles all the way down: the result of creating an instance of
any of :class:`colander.MappingSchema`, :class:`colander.TupleSchema`
or :class:`colander.SequenceSchema` object is *also* a
:class:`colander.SchemaNode` object.

Instantiating a :class:`colander.MappingSchema` creates a schema node
which has a *type* value of :class:`colander.Mapping`.

Instantiating a :class:`colander.TupleSchema` creates a schema node
which has a *type* value of :class:`colander.Tuple`.

Instantiating a :class:`colander.SequenceSchema` creates a schema node
which has a *type* value of :class:`colander.Sequence`.

Creating Schemas Without Using a Class Statement (Imperatively)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

See `http://docs.repoze.org/colander/#defining-a-schema-imperatively
<http://docs.repoze.org/colander/#defining-a-schema-imperatively>`_
for information about how to create schemas without using a ``class``
statement.

Creating a schema with or without ``class`` statements is purely a
style decision; the outcome of creating a schema without ``class``
statements is the same as creating one with ``class`` statements.

Rendering a Form and Validating Form Submission Data
----------------------------------------------------

Earlier we defined a schema:

.. code-block:: python
   :linenos:

   import colander

   class Person(colander.MappingSchema):
       name = colander.SchemaNode(colander.String())
       age = colander.SchemaNode(colander.Integer(),
                                 validator=colander.Range(0, 200))

   class People(colander.SequenceSchema):
       person = Person()

   class Schema(colander.MappingSchema):
       people = People()

   schema = Schema()

Let's now use this schema to create, render and validate a form.

Creating a Form Object
~~~~~~~~~~~~~~~~~~~~~~

To create a form object, we do this:

.. code-block:: python
   :linenos:

   from deform import Form
   myform = Form(schema, buttons=('submit',))

We used the ``schema`` object (an instance of
:class:`colander.MappingSchema`) we created in the previous section as
the first positional parameter to the :class:`deform.Form` class; we
passed the value ``('submit',)`` as the value of the ``buttons``
keyword argument.  This will cause a single button labeled ``Submit``
to be injected at the bottom of the form rendering.  We chose to pass
in the button names as a sequence of strings, but we could have also
passed a sequence of instances of the :class:`deform.Button` class.
Either is permissible.

Note that the first positional argument to :class:`deform.Form` must
be a schema node representing a *mapping* object (a structure which
maps a key to a value).  We satisfied this constraint above by passing
our ``schema`` object, which we obtained via the
:class:`colander.MappingSchema` constructor, as the ``schema``
argument to the :class:`deform.Form` constructor

Although different kinds of schema nodes can be present in a schema
used by a Deform :class:`deform.Form` instance, a form instance cannot
deal with a schema node representing a sequence, a tuple schema, a
string, an integer, etc. as its first parameter; only a schema node
representing a mapping is permissible.  This typically means that the
object passed as the ``schema`` argument to a :class:`deform.Form`
constructor must be obtained as the result of using the
:class:`colander.MappingSchema` constructor (or the equivalent
imperative spelling).

Rendering the Form
~~~~~~~~~~~~~~~~~~

Once we've created a Form object, we can render it without issue by
calling the :meth:`deform.Field.render` method: the
:class:`deform.Form` class is a subclass of the :class:`deform.Field`
class, so this method is available to a :class:`deform.Form` instance.

If we have some existing data already that we'd like to edit using the
form (the form is an "edit form" as opposed to an "add form").  That
data might look like this:

.. code-block:: python
   :linenos:

    appstruct = [
        {
            'name':'keith',
            'age':20,
            },
        {
            'name':'fred',
            'age':23,
            },
        ]

To inject it into the serialized form as the data to be edited, we'd
pass it in to the :meth:`deform.Field.render` method to get a form
rendering:

.. code-block:: python

   form = myform.render(appstruct)

If instead we wanted to render a "read-only" variant of an edit form
using the same appstruct, we'd pass the ``readonly`` flag as ``True``
to the :meth:`deform.Field.render` method.

.. code-block:: python

   form = myform.render(appstruct, readonly=True)

This would cause a page to be rendered in a crude form without any
form controls, so the user it's presented to cannot edit it.

If, finally, we wanted to render an "add" form (a form without initial
data), we'd just omit the appstruct while calling
:meth:`deform.Field.render`.

.. code-block:: python

   form = myform.render()

Once any of the above statements runs, the ``form`` variable is now a
Unicode object containing an HTML rendering of the edit form, useful
for serving out to a browser.  The root tag of the rendering will be
the ``<form>`` tag representing this form (or at least a ``<div>`` tag
that contains this form tag), so the application using it will need to
wrap it in HTML ``<html>`` and ``<body>`` tags as necessary.  It will
need to be inserted as "structure" without any HTML escaping.

Serving up the Rendered Form
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We now have an HTML rendering of a form as the variable named
``form``.  But before we can serve it up successfully to a browser
user, we have to make sure that static resources used by Deform can be
resolved properly. Some Deform widgets (including at least one we've
implied in our sample schema) require access to static resources such
as images via HTTP.

For these widgets to work properly, we'll need to arrange that files
in the directory named ``static`` within the :mod:`deform` package can
be resolved via a URL which lives at the same hostname and port number
as the page which serves up the form itself.  For example, the URL
``/static/images/close.png`` should be willing to return the
``close.png`` image in the ``static/images`` directory in the
:mod:`deform` package and ``/static/scripts/deform.js`` as
``image/png`` content .  How you arrange to do this is dependent on
your web framework.  It's done in :mod:`repoze.bfg` imperative
configuration via:

.. code-block:: python

  config = Configurator(...)
  ...
  config.add_static_view('static', 'deform:static')
  ...

Validating a Form Submission
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once the user seen the form and has chewed on its inputs a bit, he
will eventually submit the form.  When he submits it, the logic you
use to deal with the form validation must do a few things:

- It must detect that a submit button was clicked.

- It must obtain the list of :term:`form controls` from the form POST
  data.

- It must call the :meth:`deform.Form.validate` method with the list
  of form controls.

- It must be willing to catch a :exc:`deform.ValidationError`
  exception and rerender the form if there were validation errors.

For example, using the :term:`WebOb` API for the above tasks, and the
``form`` object we created earlier, such a dance might look like this:

.. code-block:: python
   :linenos:

   if 'submit' in request.POST: # detect that the submit button was clicked

       controls = request.POST.items() # get the form controls

       try:
           appstruct = myform.validate(controls)  # call validate
       except ValidationFailure, e: # catch the exception
           return {'form':e.render()} # re-render the form with an exception

       # the form submission succeeded, we have the data
       return {'form':None, 'appstruct':appstruct}

The above set of statements is the sort of logic every web app that
uses Deform must do.  If the validation stage does not fail, a
variable named ``appstruct`` will exist with the data serialized from
the form to be used in your application.  Otherwise the form will be
rerendered.

Note that by default, when any form submit button is clicked, the form
will send a post request to the same URL which rendered the form.
This can be changed by passing a different ``action`` to the
:class:`deform.Form` constructor.

Seeing it In Action
~~~~~~~~~~~~~~~~~~~

To see an "add form" in action that follows the schema in this
chapter, visit `http://deformdemo.repoze.org/sequence_of_mappings/
<http://deformdemo.repoze.org/sequence_of_mappings/>`_.

To see a "readonly edit form" in action that follows the schema in
this chapter, visit
`http://deformdemo.repoze.org/readonly_sequence_of_mappings/
<http://deformdemo.repoze.org/readonly_sequence_of_mappings/>`_

The application at http://deformdemo.repoze.org is a :mod:`repoze.bfg`
application which demonstrates most of the features of Deform,
including most of the widget and data types available for use within
an application that uses Deform.  
