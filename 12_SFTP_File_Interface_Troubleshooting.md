# SFTP and File Interface Troubleshooting
# Permissions, checksums, control files, delivery proof

Use this when a feed is generated but the consumer says it never arrived, or arrived twice.

---

## 1. The path a file takes

1. Job writes a local file and a control file (row count, checksum).
2. Names are checked against the expected pattern.
3. SFTP or copy puts the data file, then the control or `.FIN` marker.
4. The consumer waits for the marker, then reads the data file.
5. Local copy is kept until the consumer confirms the load.

If you send the marker before the data file is complete, the consumer reads a partial file. Put the data file first, then the `.FIN` or `.ctl`.

---

## 2. Local file checks

```bash
ls -l /path/out/D_*.dat /path/out/DWH_*.dat
wc -l /path/out/D_20260925.dat
head -1 /path/out/D_20260925.dat
sha256sum /path/out/nfc_dupes_*.csv
```

Header must match the contract exactly. A missing column or a different delimiter fails the load even when `wc -l` looks fine.

Empty file:

```bash
find /path/out -type f -name '*.dat' -size 0 -print
```

Zero rows can be valid. The control file must say `rows=0`, and the consumer must accept an empty data file. Do not invent a row to "test" a prod path.

---

## 3. SFTP session

```bash
sftp -o BatchMode=yes user@host <<'EOF'
cd /remote/dir
ls -l
bye
EOF
```

Batch mode fails fast when keys are missing. A hang is usually DNS or port 22, not the file contents.

```bash
getent hosts <sftp-host>
timeout 3 bash -c 'echo >/dev/tcp/<sftp-host>/22' && echo OPEN || echo CLOSED
```

Permissions on the remote directory: the SFTP user must be able to create the remote name. `Permission denied` on `put` is the remote directory or disk quota, not your local checksum.

Confirm the file landed:

```bash
sftp user@host <<'EOF'
cd /remote/dir
ls -l DWH_BC_*
bye
EOF
```

Compare size to the local `ls -l`. Size mismatch means a partial put.

---

## 4. Names and sequence

Consumers often accept only a pattern such as `D_*_30007*.dat` plus a companion `name.FIN` (not `name.dat.FIN`).

Before send:

- the new sequence is higher than the last accepted file
- the date in the name is the agreed format (`dd-mm-yyyy` vs `yyyymmdd` — do not mix them)
- you are not reusing a name already on the server

A reused name can be ignored or can overwrite a file the consumer has not loaded yet.

---

## 5. Checksums and control files

```bash
sha256sum data.csv
awk -F= '/^sha256=/{print $2}' data.ctl
awk -F= '/^rows=/{print $2}' data.ctl
echo $(($(wc -l < data.csv) - 1))
```

The checksum in the control file must equal `sha256sum` of the data file. Row count must equal data lines excluding the header.

If they differ, do not SFTP. Regenerate. A consumer that trusts the control file will reject or truncate the load.

---

## 6. Duplicates

| What happened | What to do |
|---|---|
| Script retried and put the same file twice under two names | Tell the consumer which name is authoritative. Ask them to ignore the other |
| Same business data, new sequence, load is not idempotent | Do not send the second file. Confirm whether the first already loaded |
| `.FIN` sent twice | Usually harmless if the data file is the same size. Confirm with the consumer |

Keep the local file. It is the evidence when someone reports missing records later.

---

## 7. Same-host copy instead of SFTP

On a test host that is also the consumer:

```bash
mkdir -p /users/xpiwrk1/TOOLS/HandleDuplicateFutureNcm/in
cp -p "$CSV" "$CTL" /users/xpiwrk1/TOOLS/HandleDuplicateFutureNcm/in/
ls -l /users/xpiwrk1/TOOLS/HandleDuplicateFutureNcm/in/
```

`cp -p` keeps the timestamp. The apply job only needs the files in `in/`. It does not care whether they arrived by `cp` or `scp`.

---

## 8. Ticket note

```text
Local file and size:
sha256:
rows:
Remote host and directory:
Remote name:
Remote size after put:
Marker file name:
Consumer load status:
```
