
Approval
========

``Approval`` is a concise class for doing asynchronous security.  Security checks can do lookups if they need to. They
can delegate to other approvals if find they need to. And permissions are remembered within an approval appropriately
so that it all works very efficiently.

For example, we could define the "edit page" permission as::

  val editPerm = Perm.cacheOnId[User, Page]({ case (prior, refPage) =>
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

And then if we ask for it::

    val p = LazyId(1).of[Page]
    for {
      approved <- approval ask editPage(p)
    }

If we've already been approved to edit that page, it'll just return ``Approved("Already approved")``
It will look up the page, and delegate the request to either ask for ``editProtected`` or ``editUnprotected`` on the course, etc.