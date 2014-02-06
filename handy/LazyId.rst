
Lazy Ids
========

Unsurprisingly, a ``LazyId`` is a referernce to something by its Id. It has two parts -- a key, and how to look the item up::

    LazyId(id).of(lookupMethod)

Why is it split into two calls like that? It lets us take better advantage of Scala's implicits, if we have an implicit
lookup method in scope. For instance, we could write::

    LazyId(id).of[User]

And if there is an implicit lookup method for Users (and that kind of key) in scope, then it will be used.


Equality for Lazy IDs
---------------------

Two ``LazyId``s are equal if they have the same key and the same lookup method. This typically means that::

    LazyId(1).of[User] != LazyId(1).of[Course]


LazyIds are lazy Refs that memoise their result
-----------------------------------------------

A ``LazyId[User]`` is a subtype of ``Ref[User]``. If we call ``map``, ``flatMap``, or many of the other functions on the
LazyId, it will call the lookup method. This uses a ``lazy val``, so that once the item has been looked up, it will be
remembered.

For instance, if your lookup method is asynchronous, and returns a ``Future`` (wrapped as a ``RefFuture``), then that
future will be memoised in the lazy val.

If you don't want the memoised result (for instance you have made an update that has changed the value in the database),
you can always use either ``lazyId.copy``, which will produce a duplicate lazy id with the same key and lookup method
but with the lazy val not memoised yet.  Or you could use ``RefById``, which is like ``LazyId`` but without the lazy val
(ie, no memoisation of the result).


Look up methods return another Ref
----------------------------------

A lookup method for a single item::

    trait LookUpOne[T, K] {
      def lookUpOne[KK <: K](r:RefById[T]):Ref[T]
    }

It takes the key, and can return any kind of ``Ref``. So, it could work asynchronously, returning a ``RefFuture[T]``.
Or it could work synchronously, returning ``RefItself[T]``, ``RefFailed``, or ``RefNone``.

As the lookups for different types of item can be independent of one another, it is simple to write applications where
different items live in different databases.


Getting the key
---------------

``LazyId[T]`` is also an ``ImmediateId[T]``. This means that the ID for this Ref, if there is one, is immediately
available::

    val id = lazyId.getId

If you just use string IDs everywhere (and your data uses the trait ``HasStringId``) you won't need to worry about this
next bit, but we also have a mechanism to "canonicalise" ids.

Should ``LazyId("1").of[User]``, ``LazyId(1).of[User]`` and ``LazyId(1L).of[User]`` all refer to the same item?

Or, if you're using something like MongoDB, what about converting strings to an ``ObjectId``?

And if we have the item itself (a ``RefItself``), then how do we extract the id from it? Is it in ``id``, or (as in
MongoDB objects) in ``_id``, or somewhere else?

``LazyId.getId`` takes an implicit parameter of type ``GetsId[T, KK]``. This is an object that knows how to get the ID
of that kind of item, and knows how to convert ids that might be in the wrong type into a "canonical" form.


Getting the key of a Future lookup
----------------------------------

Suppose we want the ID of a reference, but we don't have a ``LazyId``, we've got a database query::

    val rUser:Ref[User] = UserDAO.byName("Algernon Moncrieff")

And suppose our database is asynchronous, and gives us a ``Future`` (wrapped as a ``RefFuture``).

Clearly for this reference, we can't get the ID immediately -- we'd need to wait for the database to have finished
running the query, ie for the ``Future`` to complete.

So instead we would use the ``refId`` method that is defined for all ``Ref``s::

    val rId = rUser.refId

This will give us a ``Ref[K]`` where ``K`` is the type of ID this item has.

If we want to wait (block) and get that as an ``Option``, we can::

    val blockingId = rId.fetch.toOption

but we can also keep working asynchronously, because ``Ref`` is a monad that is happy with asynchronicity::

    for {
      id <- rId
    } yield ...

Generally doing the latter (using the monadic ``map`` and ``flatMap`` methods, or ``for`` notation) is better. If the
ID is immediately available, it will run immediately (negligible cost); if it is a Future computation it will run when
the Future is complete.


Lookup caches
-------------

Although ``LazyId``s memoise their results, sometimes you want to ensure that even if you have another ``LazyId`` to the
same item, it won't repeat the lookup.

This is what ``LookupCache`` is for.  The class is available to you; it's up to you when you use it.

Remember, two ``LazyId``s are equal if their IDs and their lookup methods are the same. So, a ``LookupCache`` keeps a
simple concurrent mutable map of ``LazyId`` to looked up ``Ref``.

A typical usage might be::

    for {
      item <- cache.lookUp(lazyId)
    } ...

This will return the cached ``Ref`` if it is available, or cache the ``LazyId`` (which memoises its result) if there is
not already a value in the cache.

If you have an item, you can also pre-seed a cache with it::

    val item = User(id=1, name="Algernon Moncrieff)
    cache.remember[Int, User](item.itself)

This uses an implicit lookup method and an implicit ``GetsId``. The reason being that we need to construct a ``LazyId``
of the canonical type, including the lookup method it would use, and then store this ``Ref`` as the cached value for it.


Lookup catalogs
---------------

``LazyId(1).of[User]`` requires a look up method to be passed in (implicitly) at compile time.

It may be that you'd rather delay that until runtime -- keep a registry or catalog of lookup methods, and have your
start-up configuration register lookup methods that will be used.

This is what ``LookUpCatalog`` is for::

    val catalog = new LookUpCatalog
    val ref = catalog.lazyId(classOf[User], 1)

And separately::

    catalog.registerLookUp(classOf[User], userLookup)

You'll notice that when using ``LookUpCatalog``, we pass in a ``Class`` object when we want to register a look up
method, or create a ``LazyId`` that uses the catalog.  This is because of type erasure in Scala (and Java).

Two ``LazyId``s from a ``LookUpCatalog`` are considered equal if they have the same ID, the same class (passed in), and
come from the same catalog.


