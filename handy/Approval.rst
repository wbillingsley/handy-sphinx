
Approval
========

``Approval`` is a concise and simple way of doing asynchronous security, that lets you express permission rules of
arbitrary complexity. Security checks can be asynchronous (for instance if they want to make an asynchronous database lookup).
They can delegate to other approvals if find they need to. And permissions are remembered within an approval
appropriately so that it all works very efficiently.


There are four very simple types to understand:

* ``Approved`` is a simple token just to say something has been approved.

* A permission is an object of type ``Perm`` that has a single method, ``resolve``, which will
  decide (possibly asynchronously) whether to approve or refuse the permission.  It returns a ``Ref[Approved]``. The
  permission is approved if that results in an approval (for example ``RefItself(Approved)`` or a ``RefFuture`` that is
  successful).

* ``Approval`` collects these permissions when they are asked for and approved. It has the ``ask`` method that is used to
  ask for a permission. It has also holds a ``Ref[U]`` to the user the permissions are being collected for, in the value
  ``who``.

* ``Refused`` is a failure that says the permission has been refused. If a permission returns ``Refused(...)`` it is
  implicitly promoted to ``RefFailed(Refused(...))``.

Unique permissions
------------------

As a simple first example, let's consider designing a "create course" permission where the user can create the course
if they have the role "Teacher" on the site::

  val createCourse = Perm.unique[User] { (prior) =>
    (
      for {
        user <- prior.who if user.hasRole("Teacher")
      } yield Approved("Yes, you are an author")
    ) orIfNone Refused("You need to have the Teacher role to create courses")
  }


What's going on here?

``Perm.unique`` defines a new permission. You could also do this by directly subclassing ``Perm``.  The resolution
method is passed the ``Approval`` object, called "prior" because it caches all the prior approvals that have been
granted on it.

``prior.who`` is a ``Ref[User]`` -- a reference to the user being approved. Its value is set when the ``Approval`` object
is created.  Typically it's likely to be a ``RefFuture`` as most applications resolve which user is making an HTTP request
by taking the session cookie and querying a database (asynchronously) to see which user has that session open.

The logic in the permission filters ``prior.who``, requiring the "Author" role. This will result in the user if they are
an author, or none if they are not.

As our permission check is likely to be a step in an algorithm, we'd like the result to have a nice failure message rather
than just producing none, so we write ``orIfNone Refused("...")`` to change it from none to a failure.


Using this permission in an algorithm
-------------------------------------

The permission we've defined now fits neatly into monadic algorithms. For example, our algorithm to create a course
might be::

  for {
    approved <- request.approval ask createCourse
    created  <- pageModel.newCourse(json)
    saved    <- db.saveNew(created)
    json     <- toJson(saved)
  } yield Ok(json)

And if the user making the request is not a teacher, the result will be ``RefFailed(Refused("You need to have the Teacher role to create courses"))``


Equal on ID
-----------

Most permissions in a system are about an item. For example, we might want the user to be able to edit a page within a
course. This might be::

  val editPage = Perm.cacheOnId[User, Page] { case (prior, refPage) =>
    ...
  }

This gives us ``editPage``, that can produce permissions asking to edit particular pages.  So,
``editPage(page123.itself)`` and ``editPage(LazyId(123).of[Page])`` are permissions asking to
edit page 123.

Why ``Perm.cacheOnId``?  If the user is approved to ``editPage(page123.itself)``, then a later request for
approval to ``editPage(LazyId(123).of[Page])`` will find the previously granted permission because both permissions
come from ``editPage`` and refer to items with the same ID.


Delegating permissions
----------------------

On many occasions, permissions will want to delegate to other permissions.

For example, suppose there are "protected" and "unprotected" pages in a course, and students are allowed to edit the
unprotected pages, but only moderators can edit the protected ones.  We could simply write this as::

  val editPage = Perm.cacheOnId[User, Page]({ case (prior, refPage) =>
    for {
      page <- refPage
      approved <- {
        if (page.protected) {
          prior ask editProtected(page.course)
        } else {
          prior ask editUnprotected(page.course)
        }
    } yield approved
  })


Summing up
----------

Approval lets us express our security rules in terms of our application classes.

It's independent of the web framework we choose to use, and independent of the datastores we plug in to look up our
data.
