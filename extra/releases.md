# The Release Process

API Platform follows the [Semantic Versioning](https://semver.org) strategy.

Only 3 versions are maintained at the same time:

* **stable** (currently the **2.3** branch): regular bug fixes are integrated in this version
* **old-stable** (currently **2.2** branch): security fixes are integrated in this version, regular bug fixes are **not** backported in it
* **development** (**master** branch): new features target this branch

Older versions (1.x, 2.0...) **are not maintained**. If you still use them, you must upgrade as soon as possible.

The **old-stable** branch is merged in the **stable** branch on a regular basis to propagate security fixes.
The **stable** branch is merged in the **development** branch on a regular basis to propagate security and regular bug fixes.

New versions of API Platform are released **when they are ready**, on the behalf of the API Platform Core Team.
