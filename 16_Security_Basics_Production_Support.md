# Security Basics for Production Support
# Secrets, access, and safe diagnostics

Use this so a troubleshooting session does not become a data leak.
You can debug thoroughly and still keep passwords, tokens, and customer payloads out of chat, mail, and tickets that a wide audience can read.

---

## 1. What counts as a secret

- database passwords and connection strings with passwords
- Kafka keystores, private keys, truststore passwords
- API keys, Bearer tokens, refresh tokens, cookies
- `oc` / `kubectl` kubeconfig files
- full heap dumps and thread dumps that include request bodies
- customer identifiers in bulk (files of MSISDNs) when the audience does not need them

A JWT payload can be decoded by anyone who has the token. Treat the token like a password.

---

## 2. Before you paste output

```bash
# show that a variable exists without printing it
if [[ -n "${PGPASSWORD:-}" ]]; then echo PGPASSWORD=set; else echo PGPASSWORD=empty; fi

env | grep -E 'PASS|SECRET|TOKEN|KEY' | sed -E 's/=.*/=***/'
```

Redact before sending:

- `password=...`
- `Authorization: Bearer ...`
- private key blocks
- keystore passwords in `ssl.client.properties`

It is fine to paste hostnames, ports, error codes, and SQL text with one test MSISDN when the ticket already concerns that MSISDN.

---

## 3. Files and permissions

```bash
ls -l /path/ssl.client.properties /path/*.p12
namei -l /path/to/key
umask
```

A private key should not be world-readable (`-rw-r--r--` is already wider than you want for a key). If a job fails to read a key, fix the owner and group. Do not `chmod 777`.

```bash
chmod 600 /path/keystore.key
chmod 640 /path/ssl.client.properties
```

Only do that on files you own, and only when the process user still has access.

Do not commit `.env`, `sm.env`, kubeconfig, or p12 files into git.

---

## 4. Production diagnostics that are safe

| Action | Safer way |
|---|---|
| Need a SQL plan | Run `EXPLAIN`, not a `SELECT *` of the whole customer table into a file on your laptop |
| Need a thread dump | Save it on the server. Search for the nid. Copy the one stack, not the file, if the file contains request bodies |
| Need a heap dump | Keep it on the secured host. Do not attach it to a chat |
| Need to prove a password works | Say "login succeeded". Do not echo the password in the script log |
| Need kubectl access | `kubectl auth can-i`. Do not share your kubeconfig |
| Sample a Kafka message | One message, secrets removed, in the ticket with restricted visibility |

`set -x` in bash prints expanded variables, including passwords. Do not enable it on a script that uses `PGPASSWORD`.

---

## 5. Least privilege

Use a read-only database user for investigation when you are not applying a fix.
Use a personal account, not a shared break-glass account, so the audit log shows who ran the statement.

If you must use a privileged account:

- say why in the ticket
- run the one statement you planned
- do not leave a SQL session open with that user

`DELETE`, `UPDATE`, offset reset, and `kubectl delete` are changes. They belong in the ticket before you run them, except when a Sev1 bridge has already approved the action.

---

## 6. Logs you ship to Splunk

Do not turn on DEBUG that prints full PAN, password, token, or IMSI for a whole day to catch one error.
If a log line already contains a secret, note it and ask the owning team to mask it. Do not forward that line to a wide channel to "show the bug".

---

## 7. If a secret was exposed

1. Tell the owner of that system the same day.
2. Rotate the password, token, or certificate. Hiding the chat message is not rotation.
3. Note the time and the audience in the ticket.
4. Replace scripts that had the password hardcoded with environment variables or the existing decrypt utility.

A hardcoded password in a script under a home directory is still a secret on disk. Prefer `PGPASSWORD` from the environment or the site's decrypt tool, and do not put the value in the script text.
