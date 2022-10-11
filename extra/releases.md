# The Release Process

API Platform follows the [Semantic Versioning](https://semver.org) strategy.
A new minor version is released every six months, and a new major version is released every two years, along with a last minor version on the previous major one with the same features and an upgrade path.

For example:

- version 3.0 has been released on 15 September, 2022;
- version 3.1 will be released on March, 2023;
- version 3.2 will be released on September, 2023;
- version 3.3 will be released on March, 2024;
- versions 3.4 and 4.0 will be released on September, 2024.

## Maintenance

Only 3 versions are maintained at the same time:

* **stable** (currently the **2.7** branch): regular bug fixes are integrated in this version
* **old-stable** (currently **2.6** branch): security fixes are integrated in this version, regular bug fixes are **not** backported in it
* **development** (**main** branch): new features target this branch

Older versions (1.x, 2.0...) **are not maintained**. If you still use them, you must upgrade as soon as possible.

The **old-stable** branch is merged in the **stable** branch on a regular basis to propagate [security fixes](security.md).
The **stable** branch is merged in the **development** branch on a regular basis to propagate [security](security.md) and regular bug fixes.

New versions of API Platform are released **when they are ready**, on the behalf of the API Platform Core Team.
