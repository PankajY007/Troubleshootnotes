# Networking and SSL/TLS Troubleshooting
# DNS, ports, certificates, handshake failures

Use this when a client cannot connect, or connects and then fails before HTTP.
A TCP timeout and an SSL handshake error are different faults.

---

## 1. Name to address

```bash
getent hosts <hostname>
nslookup <hostname>
dig +short <hostname>
```

`NXDOMAIN` means the name is not in DNS. Guessing another site's hostname will not help. Use the domain of the host you are on (`hostname -f`, `dnsdomainname`).

```bash
hostname -f
dnsdomainname
grep -i <short-name> /etc/hosts
```

`/etc/hosts` overrides DNS for that exact name. An empty grep means the OS will only use DNS.

---

## 2. Is the port open?

```bash
timeout 3 bash -c 'echo >/dev/tcp/<HOST>/<PORT>' && echo OPEN || echo CLOSED
```

`CLOSED` or hang-then-fail is firewall, routing, or nothing listening. It is not a bad JSON body.

```bash
ss -lntp | grep <PORT>          # on the server that should listen
ss -antp | grep <HOST>
```

From a laptop, `svc.cluster.local` names do not resolve. Those names work only inside the cluster.

Common ports on this estate:

| Port | Typical use |
|---|---|
| 5432 / 5400 | Postgres or a pooler in front of Postgres |
| 9093 | Kafka SSL |
| 8443 | HTTPS microservice |
| 443 | Public or ingress HTTPS |
| 22 | SSH / SFTP |

---

## 3. Path of a connection

1. DNS returns an IP.
2. TCP handshake to IP:port succeeds (`OPEN`).
3. TLS handshake succeeds (certificate trusted, hostname matches, client cert accepted).
4. HTTP status comes back.
5. Application code in the body says success or a business error.

Stop at the first step that fails. Do not change the payload to fix a timeout.

```bash
curl -sv --max-time 15 -o /dev/null 'https://<HOST>:<PORT>/'
```

Read the verbose lines in order:

- `Trying <ip>:<port>` — DNS worked
- `Connected` — TCP worked
- `TLSv1.2 / TLS handshake` — TLS started
- `SSL certificate problem` — trust or hostname
- `HTTP/1.1 401` — TLS worked; auth failed

---

## 4. Certificates

A service presents a certificate. The client must trust the signer, and the hostname you dialed must be in the certificate (CN or SAN).

```bash
echo | openssl s_client -connect <HOST>:443 -servername <HOST> 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

`-servername` sets SNI. Without it, a virtual host can show the wrong certificate.

Dates: `notBefore` / `notAfter`. An expired cert fails even when the port is open.

Java truststores on this estate are often PKCS12, not PEM:

```text
keystore-TC.p12
ra-admin-truststore.p12
ssl.client.properties
```

`kafka-python` wants PEM files. Java tools and `kafka-console-producer.sh` use the properties file that points at the p12. Use the tool that matches the file type.

Inspect a p12 (it will ask for the store password):

```bash
keytool -list -keystore /path/keystore-TC.p12 -storetype PKCS12
openssl pkcs12 -in /path/keystore-TC.p12 -nokeys -info
```

Do not copy private keys into tickets or chat.

---

## 5. Error to cause

| Message | Cause |
|---|---|
| `NXDOMAIN` | Name does not exist in this DNS |
| `Connection timed out` / error 110 | Packet dropped. Wrong site, firewall, or security group |
| `Connection refused` | Host reachable, nothing listening on that port |
| `certificate has expired` | Renew the server or client cert |
| `PKIX path building failed` | Client truststore does not contain the CA |
| `hostname in certificate didn't match` | You used an IP or a short name not listed on the cert |
| `handshake_failure` / `no cipher suites in common` | TLS version or cipher mismatch |
| `unknown_ca` on mTLS | Server does not trust your client cert |

---

## 6. Client certificates (mTLS and Kafka SSL)

Server cert proves the server. Client cert proves you. Kafka on 9093 and many internal HTTPS ports want both.

Check that these files exist and are readable by the user running the job:

```bash
ls -l /users/xpiwrk1/certificates/TOPIC_CREATION/
grep -E 'ssl|security|keystore|truststore|bootstrap' /users/xpiwrk1/certificates/TOPIC_CREATION/ssl.client.properties
```

A cron job that cannot read the key file fails with a permission error, not a broker outage. `ls -l` and `namei -l <path>` show which directory blocks access.

---

## 7. After a network change

```bash
getent hosts <broker-1>
timeout 3 bash -c 'echo >/dev/tcp/<broker-1>/9093' && echo OPEN || echo CLOSED
curl -sv --max-time 10 -o /dev/null 'https://<api-host>/'
```

Then retry the application call. If TCP is closed, stop. The application config is not the fault yet.
