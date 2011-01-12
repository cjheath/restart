Reclaiming REST
===============

REST, or REpresentational State Transfer, has become wrongly identified with
HTTP verbs and using URLs to name objects and simple hierarchical collections
of objects.

In fact, a Resource is a much more flexible thing than can be conveniently
described by a URL, and the restriction to a simple set of operations (PUT,
GET, UPDATE and DELETE) is much more important than the method used to
request those operations.

RESTart implements a much more flexible resource-naming scheme, using JSON
objects to describe a query that delineates the resource. The query result
is not a single object or array of objects, but a structured mesh or
constellation of objects.

RESTart works equally well with NoSQL, SQL and fact-oriented databases.
RESTart does not make your table or object structures explicit; instead it
treats a field or column as related to the table/entity in the same way
another table might be related through a foreign key association. All such
relationships are intrinsically bi-directional, so you can just as easily
ask "fetch a post having id 3" as "fetch id 3 having a post".

RESTart supports optimistic locking using a lock hash calculated across the
entire result constellation. When you want to save a changed result set, you
must pass the original resource name and the lock hash value. If that query
would yield the same result, the modification is allowed.

Because the table/object structure cannot be assumed, you must explicitly
request every value you need. There is nothing like SQL's "SELECT * FROM...",
although convenience methods allow simple inclusion of predefined groups.
This also leads to the potential for greater parallelism, because
non-conflicting transactions can update the same physical tuples.

Because a Resource can be arbitrarily complex, traversing into and across
the entire database, there is no need for more than one request in response
to any single user action. No more wandering the database processing queries
to assemble all the items needed to display a dialog; just ask for all the
items up front. This is a radical and effective way to structure even a
large application.

The example code here is for the Javascript binding.

Initialise
----------
  <script src="http://dataconstellation.com/scripts/api_min.1.5.2.js">
    var session = new Session('https://api.mybusiness.com/blog', credentials)
  </script>

Resources
---------

The resource syntax defines a query that yields a fact constellation:

<pre>
  /*
   * Example (omit the comments for valid JSON!). Refer to ORM2 model at
   * http://dataconstellation.com/ActiveFacts/examples/images/Blog.png
   */
  var resource = {
      "post": {         // Get all Posts
        "author": {     // having an Author
          "name": [     // with a Name
            "Bloggs",   // matching this
            "Baggins"   // or this
          ]
        },
        "paragraph_count": {
	  "$count": "paragraph"
	},
        "paragraph": {  // and all paragraphs for the post
          "ordinal": {  // Having Ordinal (paragraph sequence number) &lt;= 3
            "&lt;=": 3
          },
          "content": {
            "style": [],// return the style, if any
            "all": [],  // returning the additional content implied by the "all" grouping
          }
        },
        "comment": [],  // Returning all comments (ok if there are none - outer join)
        "topic": {      // A topic is mandatory
          "parent*": [] // Include parent topic recursively, if any
        }
      }
    };
</pre>

Here are the expressions you may use in a query:

* [...] -> The parent must match any one or more array element. Each array element should be either:
  * a value; the parent value must match.
  * null; the parent value may be null
  * an object {...}; the parent value must pass all tests (see below)
  As a special case, an empty array retrieves the value, whether it's null or any non-null value.

* {...} -> the item must match all content items. Content keys may be:
  * a field/column (role) name for the current parent object
  * an association/relationship name from the current object
  * an operator ("<", "<=", "=", "!=", "<>", ">=", ">", "~")
  * an aggregate operator ("$sum", "$avg", ...) which defines the parent item
    based on the sum, etc, of the content value (in this case, the individual
    values are *not* fetched unless otherwise requested).

Assert
------

The resource used in an assert must have exactly one value for each field/column.

<pre>
  var handlers = {
    on_success: function() { ... },
    on_failure: function(reason) { ... }
  };
  var request = session.assert(resource, handlers);
</pre>

The assert operation will not cause an error if a record already exists.
Any facts in existing records will be modified to the ones you specify.

Create
------

The create operation is just like an assert, except that if a record already exists,
you must not contradict existing facts, or you will get a contradiction error.

<pre>
  var handlers = {
    on_success: function() { ... },
    on_failure: function(reason) { ... }
  };
  var request = session.create(resource, handlers);
</pre>

Retrieve
--------

<pre>
  // What to do when a result constellation has been retrieved:
  var handlers = {
    on_success:
      function(values, lock_hash) {
        for (var post in values) {
          for (paragraph in post.paragraph) ...
          ...
        }
      }
  }

  // Fetch posts:
  var request = session.retrieve(resource, handlers);

  // Also available:
  request.cancel();
</pre>

Update
------

<pre>
  request = session.update(
        resource,         // The resource we're talking about
        values,           // The new value of the resource (replaces the entire old value set)
        lock_hash,        // The resource-version hash we received
        handlers          // Success, failure and any other handlers (e.g. progress)
      );
</pre>

Delete
------

<pre>
  // Delete the top-level objects (posts).
  // The delete will propagate to all objects having a mandatory dependence on those
  request = session.delete(resource, handlers);
</pre>
