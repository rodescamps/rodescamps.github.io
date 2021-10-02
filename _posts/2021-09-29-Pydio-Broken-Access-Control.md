---
title: "Pydio Cells v2.2.9 Broken Access Control"
header:
  teaser: /assets/images/pydio-broken-access-control/teaser.png
  image: /assets/images/pydio-broken-access-control/splash.png
toc: true
toc_sticky: true
tags:
  - web application
  - broken access control
  - privilege escalation
---

# A modern open-source file-sharing solution

[Pydio](https://pydio.com/) developped a file-sharing solution written in Go used by many large organisations worldwide. It is a direct competitor to other known solutions in it domain such as ownCloud, Nextcloud, Seafile, Syncthing, etc. As any web application, Pydio Cells is not immune to security issues, and I decided to target this solution during one week of research to assess its security level. This project seemed to be a perfect example of a modern [open-source web application](https://github.com/pydio/cells) used by many large companies, and I was curious to see how difficult it was to breach its defenses. I did not expect to obtain the results I present hereafter.

The following three vulnerabilities affect **Pydio Cells & Enterprise v2.2.9**.

The detailed proofs of concept will be published soon in this post.

# CVE-2021-41323 - Path traversal through injection in compress feature
Security risk: **Medium**

CVSS: 6.5 (*AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:H/A:N*)

Directory traversal in the Compress feature in Pydio Cells 2.2.9 allows remote authenticated users to overwrite personal files, or Cells files belonging to any user, via the format parameter.

# CVE-2021-41324 - File enumeration through injection in copy/move/delete feature
Security risk: **Medium**

CVSS: 4.3 (*AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:N/A:N*)

Directory traversal in the Copy, Move, and Delete features in Pydio Cells 2.2.9 allows remote authenticated users to enumerate personal files (or Cells files belonging to any user) via the nodes parameter (for Copy and Move) or via the Path parameter (for Delete).

# CVE-2021-41325 - Anonymous privilege escalation
Security risk: **High**

CVSS: 7.1 (*AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:N/A:H*)

Broken access control for user creation in Pydio Cells 2.2.9 allows remote anonymous users to create standard users via the profile parameter. (In addition, such users can be granted several admin permissions via the Roles parameter.)

# Recommendation

Update Pydio Cells to the version 2.2.12 or above.

