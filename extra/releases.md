# The Release Process

API Platform follows the [Semantic Versioning](https://semver.org) strategy.

3 versions are maintained at the same time:

* **stable** (currently the **3.0** branch): regular bug fixes are integrated in this version
* **old-stable** (currently **2.7** branch): [security fixes](security.md) are integrated in this version, regular bug fixes are **not** backported in it
* **development** (**main** branch): new features target this branch

Older versions (1.x, 2.6...) **are not maintained**. If you still use them, you must upgrade as soon as possible.

The **old-stable** branch is merged in the **stable** branch on a regular basis to propagate [security fixes](security.md).
The **stable** branch is merged in the **development** branch on a regular basis to propagate [security](security.md) and regular bug fixes.

New major versions of API Platform are released every 2 years.
New minor versions of API Platform are released every 6 months.

The latest minor version of a major branch contains all the new features introduced in the first version of the next major, but also contains deprecated features which are removed in the next major branch.
