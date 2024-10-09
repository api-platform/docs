# The Release Process

API Platform follows the [Semantic Versioning](https://semver.org) strategy.
A new minor version is released every six months, and a new major version is released every two years, along with a last minor version on the previous major one with the same features and an upgrade path.

For example:

- version 3.0 has been released on 15 September 2022;
- version 3.1 has been released on 23 January 2023;
- version 3.2 has been released on 12 October 2023;
- version 3.3 has been released on 9 April 2024 (we were a little late, it should have been published in March);
- versions 3.4 and 4.0 will be released on September 2024.

## Maintenance

3 versions are maintained at the same time:

- **stable** (currently the **4.0** branch): regular bugfixes are integrated in this version
- **old-stable** (are the last 2 minor branches: currently **3.4** and **3.3** branches): [security fixes](security.md) are integrated in this version, regular bugfixes are **not** backported in it
- **development** (**main** branch): new features target this branch

Older versions (1.x, 2.6...) **are not maintained**. If you still use them, you must upgrade as soon as possible.

There's a [crowdfunding for a 2.7 LTS version](https://opencollective.com/api-platform/projects/27-lts).

The **old-stable** branch is merged in the **stable** branch on a regular basis to propagate [security fixes](security.md).
The **stable** branch is merged in the **development** branch on a regular basis to propagate [security](security.md) and regular bugfixes.

New major versions of API Platform are released every 2 years.
New minor versions of API Platform are released every 6 months.

The latest minor version of a major branch contains all the new features introduced in the first version of the next major, but also contains deprecated features which are removed in the next major branch.
