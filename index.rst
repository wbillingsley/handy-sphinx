.. Handy documentation master file, created by
   sphinx-quickstart on Mon Jan 20 12:11:01 2014.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.


Making complex apps a little bit easier
=======================================

Handy is a small set of libraries that make writing asynchronous reactive apps concise, easy, and functional.

The core of handy is Ref, a monad for the web.

It's especially useful if you are writing the server for a "single page app", such as an Angular.js app, backed by Play and ReactiveMongo, but it is tidily componentised to be useful in other contexts.



Ref is a Monad for the Web
==========================

(If you want a little primer on monads, and why they are useful, I'll add one shortly.)

Monads allow us to wrap functionality around data. This helps us to separate concerns in a generic and typesafe manner.

They also help us chain actions together using ``flatMap`` (or here, using Scala's for notation), producing an algorithm that reads as simply as if it were imperative code::

    for {
      page <- LazyId(pId).of[Page]
      approved <- request.approval ask editPage(page.itself)
      updated <- update(page, json)
      saved <- pageDAO.save(updated)
    } yield saved


But it's better than imperative code. The result of the code above would be a ``Ref[Page]``.
That means the functionality we've wrapped around it -- such as error handling and asynchronicity -- is present in the result.


What does Ref handle
--------------------


* Lazy ID references

  Most applications have some kind of database, with items referenced by an ID within it. Some are synchronous;
  some are asynchronous. We'd like to be able to express algorithms on things whether we've just got their ID, whether
  they are coming from another query, or whether we've already fetched the item.

  And of course items can reference other items. A page can be in a course, written by an author. If we need their
  details, we want to fetch them. But if all we want from the reference is the ID itself, we don't want to go and fetch
  the object from the database if we can help it.

  ``val user = LazyId(123).of[User]``


* Asynchronicity

  If we're fetching data from remote systems (including the database), we may wish to do so asynchonrously rather than
  block the thread. ``Ref`` supports ``Future`` to allow this.


* Absence

  If we look something up by its ID, perhaps there is nothing for that ID.


* Failure

  Perhaps the database is down? Perhaps the user is trying to do something they are not allowed to?


* Plurality

  Usually applications treat one-of-something differently than many (zero-or-more) of something.

  For example, we may want to just return one item from an HTTP request (and give a 404 if there isn't one).
  But we may want to *stream* items from a request for it's children, and it's only a 404 if the parent doesn't exist
  (not if the parent has no children).

  And we might want the code to give us those plural children to be as simple as ``refParent flatMap _.children``


* Enumerators (for feeding iteratees), in handy-play

  This is especially useful if you're working with an asynchronous database driver, such as ReactiveMongo, that can
  stream data out asynchonously.



Seamless asynchronous security
------------------------------

``Approval`` is a concise class for doing asynchronous security.  Security checks can do lookups if they need to. They
can delegate to other approvals if find they need to. And permissions are remembered within an approval appropriately
so that it all works very efficiently.

For example, consider editing a wiki page in a teaching course.

* We want to check for the "edit page" permission on that page.
* If the page is protected, then we need the "edit protected" permission on the course, which requires the user to be
  registered for the course with the "Moderator" role
* If it's not protected, we just need the "edit unprotected" permission on the course, which requires the user to be
  registered for the course with the "Member" role.


A typical call inside some operation might look like::

   for {
     approved <- approval ask editPage(p)
     ... // do stuff monadically
   } yield ... // whatever we want to return


This flow might need to look up the page, the user, and the course to work out the answer. But we'd also like to
remember everything we've already been granted so we don't have to look anything up twice.

``Approval`` makes checks like this incredibly concise, readable, asynchronous, and efficient.




Seamless asynchronous JSON conversion
-------------------------------------

Just as security might need to look something up, so might converting something into JSON.

For instance suppose you want to send a page to the client, but you want to send some permissions with it (so the
client knows whether to enable or disable the edit button in the GUI)::

  def toJsonFor(page:Page, approval:Approval[User]) = {
    for {
      read <- approval askBoolean read(page)
      edit <- approval askBoolean edit(page)
    } yield Json.obj(
      "id" -> page.id,
      "text" -> page.text,
      permissions -> Json.obj(
        "read" -> read,
        "edit" -> edit
      )
    )
  }


Concise support for single-page-apps
------------------------------------

Where handy really shines is in writing the server side of single page apps.

Security, JSON conversion, asynchronicity, and detecting whether this is a JSON request for data or an HTML request
(hitting refresh in the browser) so you can instead send the single page app's HTML, all wrapped up as simply as::

  def editPage(pageId:String) = DataAction.returning.one(parse.json) { implicit request ->
    for {
      page <- LazyId(pageId).of[Page]
      approved <- request.approval ask Permissions.editPage(page.itself)
      updated <- update(page, request.body)
      saved <- pageDAO.save(updated)
    } yield saved
  }

Note: ``page.itself`` is syntactic sugar for ``RefItself(page)``

Streaming all its revisions using HTTP1.1 chunked?
::

  def revisions(pageId:String) = DataAction.returning.many { implicit request =>
    for {
      page <- LazyId(pageId).of[Page]
      approved <- request.approval ask Permissions.readPage(page.itself)
      revision <- page.revisions
    } yield revision
  }


Ties into EventRoom
-------------------

Eventroom is an everso-simple library for apps to do publish and subscribe over websockets or server-sent-events.


Ties into handy-play-oauth
--------------------------

Handy-play-oauth is an everso-simple library for doing social media logins (eg, sign in with GitHub).


Detailed documentation
----------------------

The documentation pages below are being expanded with explanations of how everything works and fits neatly together.


Contents:

.. toctree::
   :maxdepth: 2

   handy/LazyId


Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`

