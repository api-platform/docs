# Security Policy

This document explains how API Platform security issues are handled by the API Platform core team (API Platform being the code hosted in [the `api-platform` GitHub organization](https://github.com/api-platform)).

## Reporting a Security Issue

If you think that you have found a security issue in API Platform, don't use the bug tracker and don't publish it publicly. Instead, all security issues must be sent to kevin+api-platform-security [at] dunglas.fr.

## Resolving Process

For each report, we first try to confirm the vulnerability. When it is confirmed, the core team works on a solution following these steps:

1. Send an acknowledgment to the reporter;
2. Work on a patch;
3. Get a CVE identifier from [mitre.org](https://cveform.mitre.org);
4. Send the patch to the reporter for review;
5. Apply the patch to all [maintained versions](releases.md) of API Platform;
6. Package new versions for all affected versions;
7. If the affected package is written in PHP, update the public [security advisories database](https://github.com/FriendsOfPHP/security-advisories) maintained by the FriendsOfPHP organization and which is used by the `check:security` command.

While we are working on a patch, please do not reveal the issue publicly.

The resolution takes anywhere between a couple of days to some months depending on its complexity and the coordination with the downstream projects (see next paragraph).

## Security Updates With Tidelift

API PLatform Core is part of [the Tidelift subscription](https://tidelift.com/subscription/pkg/packagist-api-platform-core?utm_source=packagist-api-platform-core&utm_medium=referral&utm_campaign=enterprise): verified updates for zero-day vulnerabilities, coordinated security responses, and immediate notifications of which of your applications are impacted, with the fix prepared for you!

* [Learn more](https://tidelift.com/subscription/pkg/packagist-api-platform-core?utm_source=packagist-api-platform-core&utm_medium=referral&utm_campaign=enterprise)
* [Request a demo](https://tidelift.com/subscription/request-a-demo?utm_source=packagist-api-platform-core&utm_medium=referral&utm_campaign=enterprise)

## Issue Severity

In order to determine the severity of a security issue we take into account the complexity of any potential attack, the impact of the vulnerability and also how many projects it is likely to affect. This score out of 15 is then converted into a level of: Low, Medium, High, Critical, or Exceptional.

### Attack Complexity

Score of between 1 and 5 depending on how complex it is to exploit the vulnerability

* 4 - 5 Basic: attacker must follow a set of simple steps
* 2 - 3 Complex: attacker must follow non-intuitive steps with a high level of dependencies
* 1 - 2 High: A successful attack depends on conditions beyond the attacker's control. That is, a successful attack cannot be accomplished at will, but requires the attacker to invest in some measurable amount of effort in preparation or execution against the vulnerable component before a successful attack can be expected.

### Impact

Scores from the following areas are added together to produce a score. The score for Impact is capped at 6. Each area is scored between 0 and 4.

* Integrity: Does this vulnerability cause non-public data to be accessible? If so, does the attacker have control over the data disclosed? (0-4)
* Disclosure: Can this exploit allow system data (or data handled by the system) to be compromised? If so, does the attacker have control over modification? (0-4)
* Code Execution: Does the vulnerability allow arbitrary code to be executed on an end-users system, or the server that it runs on? (0-4)
* Availability: Is the availability of a service or application affected? Is it reduced availability or total loss of availability of a service / application? Availability includes networked services (e.g., databases) or resources such as consumption of network bandwidth, processor cycles, or disk space. (0-4)

### Affected Projects

Scores from the following areas are added together to produce a score. The score for Affected Projects is capped at 4.

* Will it affect some or all projects using a component? (1-2)
* Is the usage of the component that would cause such a thing already considered bad practice? (0-1)
* How common/popular is the component (e.g. Core vs Distribution vs Schema Generator)? (0-2)
* Are a number of well-known FOSS projects using API Platform affected that requires coordinated releases? (0-1)

### Score Totals

* Attack Complexity: 1 - 5
* Impact: 1 - 6
* Affected Projects: 1 - 4

### Severity levels

* Low: 1 - 5
* Medium: 6 - 10
* High: 11 - 12
* Critical: 13 - 14
* Exceptional: 15

## Credits

This document has been adapted from the [Symfony's security policy](https://symfony.com/doc/current/contributing/code/security.html).
