# Kafka Troubleshooting Commands
# Basic to advanced, ready to use

Use the Kafka install and client config from the host you are logged into.
Do not point a script at another site's brokers. A timeout to `host:9093` is usually network or DNS, not a bad JSON payload.

```bash
# Typical on this estate
export KAFKA_HOME=<path-to-kafka>
export PATH="$KAFKA_HOME/bin:$PATH"
export KAFKA_HEAP_OPTS="-Xmx1G -Xms512M"

# Broker list: from env file OR from client properties
grep -iE 'broker|bootstrap.servers' /users/xpiwrk1/ccRoot/Zon1/TC/KAFKA/env.properties
grep -E '^[[:space:]]*bootstrap.servers=' /users/xpiwrk1/certificates/TOPIC_CREATION/ssl.client.properties

BROKER_URL="<host1:9093,host2:9093>"
SSL_CFG="/users/xpiwrk1/certificates/TOPIC_CREATION/ssl.client.properties"
TOPIC="NCM_Activities"
```

SSL client files must match this cluster (truststore + keystore). A PEM path from another site will fail the handshake even after the port opens.

---

## 1. Is the cluster reachable?

```bash
getent hosts <broker-hostname>
timeout 3 bash -c 'echo >/dev/tcp/<broker-host>/9093' && echo OPEN || echo CLOSED
```

`OPEN` means TCP works. It does not prove SSL or ACLs.

```bash
kafka-broker-api-versions.sh \
  --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG"
```

If this hangs or returns `TimeoutException` / `ETIMEDOUT`:

- wrong broker list for this host
- firewall between sites
- DNS NXDOMAIN

If it fails with `SSLHandshakeException` or `PKIX`:

- wrong truststore/keystore
- expired cert
- hostname in the cert does not match the broker name you used

---

## 2. Topic exists, partitions, replication

```bash
kafka-topics.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --list | grep -E 'NCM|error'

kafka-topics.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --describe --topic "$TOPIC"
```

Read each line:

| Field | Healthy | Problem |
|---|---|---|
| `Leader` | a broker id, not `none` | `none` = partition offline |
| `Replicas` | same count on every partition | missing replica |
| `Isr` | equals Replicas | under-replicated; a broker is down or slow |
| `OfflinePartitions` | absent | producers/consumers fail on that partition |

Under-replicated:

```bash
kafka-topics.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --describe --under-replicated-partitions
```

---

## 3. What is the log end? (offsets)

Beginning and end offsets per partition:

```bash
kafka-get-offsets.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --topic "$TOPIC" --time -2

kafka-get-offsets.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --topic "$TOPIC" --time -1
```

`-2` = earliest retained offset. `-1` = latest (next offset to be written).

Message count still in retention for one partition:

```text
latest - earliest
```

Older `GetOffsetShell` (what some of your shell scripts use):

```bash
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list "$BROKER_URL" \
  --topic "$TOPIC" \
  --time -1 \
  --consumer.config "$SSL_CFG"
```

---

## 4. Consumer lag — the main support check

```bash
kafka-consumer-groups.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --list

kafka-consumer-groups.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --describe --group <GROUP_ID>
```

Columns that matter:

| Column | Meaning |
|---|---|
| `CURRENT-OFFSET` | last committed offset for that partition |
| `LOG-END-OFFSET` | latest on the broker |
| `LAG` | how many messages this group has not committed |
| `CONSUMER-ID` | which member owns the partition; empty = no active consumer |
| `HOST` | client host |

Lag growing and consumer present: consumer is slower than produce rate, or processing is stuck.
Lag flat and consumer absent: group is down; messages wait until retention deletes them.
Lag 0: group is caught up. That does not prove the business action succeeded (the app may have committed and then failed downstream).

Members and assignment:

```bash
kafka-consumer-groups.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --describe --group <GROUP_ID> --members --verbose
```

---

## 5. Read messages

Do not use the application's group id. A console consumer with that group commits offsets and steals partitions.

```bash
kafka-console-consumer.sh --bootstrap-server "$BROKER_URL" \
  --consumer.config "$SSL_CFG" \
  --topic "$TOPIC" \
  --from-beginning \
  --max-messages 5 \
  --timeout-ms 20000
```

One partition, from an offset (no group commit if you omit `--group`):

```bash
kafka-console-consumer.sh --bootstrap-server "$BROKER_URL" \
  --consumer.config "$SSL_CFG" \
  --topic "$TOPIC" \
  --partition 0 \
  --offset <OFFSET> \
  --max-messages 3
```

Search a small sample for an MSISDN (stop with Ctrl+C). This scans the topic; keep `--max-messages` set.

```bash
kafka-console-consumer.sh --bootstrap-server "$BROKER_URL" \
  --consumer.config "$SSL_CFG" \
  --topic "$TOPIC" \
  --from-beginning \
  --max-messages 5000 \
  --timeout-ms 30000 \
  | grep -m 5 '998905904003'
```

Pretty-print one JSON line:

```bash
# after you copied one line into /tmp/msg.json
python3 -m json.tool /tmp/msg.json | head -80
```

Keys and timestamps (if the producer sent them):

```bash
kafka-console-consumer.sh --bootstrap-server "$BROKER_URL" \
  --consumer.config "$SSL_CFG" \
  --topic "$TOPIC" \
  --partition 0 --offset <OFFSET> --max-messages 1 \
  --property print.timestamp=true \
  --property print.key=true \
  --property print.partition=true \
  --property print.offset=true
```

---

## 6. Publish one payload (test)

Dry-run by printing the file first. Send only when the JSON is one message per line (JSONL).

```bash
wc -l /path/in/add_offer_*.jsonl
head -1 /path/in/add_offer_*.jsonl

kafka-console-producer.sh \
  --bootstrap-server "$BROKER_URL" \
  --topic "$TOPIC" \
  --producer.config "$SSL_CFG" \
  < /path/in/add_offer_YYYYMMDD.jsonl
```

`acks` and retries belong in `ssl.client.properties` or an extra producer properties file:

```properties
acks=all
retries=5
enable.idempotence=true
```

Then:

```bash
kafka-console-producer.sh \
  --bootstrap-server "$BROKER_URL" \
  --topic "$TOPIC" \
  --producer.config /tmp/producer-ssl.properties \
  < /tmp/one-message.json
```

Confirm the offset moved:

```bash
kafka-get-offsets.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --topic "$TOPIC" --time -1
```

---

## 7. When a consumer is stuck

1. `kafka-consumer-groups.sh --describe` — which partition has lag, is `CONSUMER-ID` empty?
2. On the app host: is the process up? (`ps`, pod status).
3. App log for that time: deserialization error, poison message, DB timeout.
4. If one partition lags and others are 0, the slow partition's key (MSISDN/customer) is hot, or one worker is blocked.

Do not reset offsets until you know whether replaying is safe. Replay of `ADD_SERVICE` / charge notifications can double-apply.

Show the offset the group would reset to (does not change anything):

```bash
kafka-consumer-groups.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --group <GROUP_ID> \
  --topic "$TOPIC" \
  --reset-offsets --to-latest \
  --dry-run
```

Execute only with approval:

```bash
# group must be stopped / empty first
kafka-consumer-groups.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --group <GROUP_ID> \
  --topic "$TOPIC" \
  --reset-offsets --to-datetime 2026-09-16T00:00:00.000 \
  --execute
```

`--to-offset`, `--to-earliest`, `--to-latest` exist. `--execute` without `--dry-run` changes the group.

---

## 8. Broker-side errors you will see

| Error | Meaning | What to do |
|---|---|---|
| `TimeoutException` / error 110 | TCP to broker failed | DNS, route, correct site brokers |
| `UNKNOWN_TOPIC_OR_PARTITION` | topic name wrong or metadata stale | `--list`, exact case |
| `NOT_LEADER_OR_FOLLOWER` | client hit a broker that is not leader | retry; if it persists, cluster rolling restart |
| `NOT_ENOUGH_REPLICAS` | `acks=all` and ISR < min.insync.replicas | disk/broker down; do not keep producing blindly |
| `RECORD_TOO_LARGE` | message > `max.message.bytes` | shrink payload or DBA raises the topic limit |
| `SASL` / `SSL` / `AUTHENTICATION_FAILED` | cert or principal | use this cluster's client properties |
| `GROUP_AUTHORIZATION_FAILED` | ACL | different principal or ticket to platform team |
| `OFFSET_OUT_OF_RANGE` | committed offset was deleted by retention | group must reset to earliest or a time still retained |

Retention: if earliest offset is already past your event, the message is gone from Kafka. Look in app logs or the audit table, not the topic.

---

## 9. Produce / consume test on a scratch topic

Only on a non-prod topic you are allowed to create.

```bash
kafka-topics.sh --bootstrap-server "$BROKER_URL" \
  --command-config "$SSL_CFG" \
  --create --topic support_ping_test \
  --partitions 1 --replication-factor 1

echo "ping $(date -u +%FT%TZ)" | kafka-console-producer.sh \
  --bootstrap-server "$BROKER_URL" \
  --producer.config "$SSL_CFG" \
  --topic support_ping_test

kafka-console-consumer.sh --bootstrap-server "$BROKER_URL" \
  --consumer.config "$SSL_CFG" \
  --topic support_ping_test \
  --from-beginning --max-messages 1 --timeout-ms 15000
```

---

## 10. How this maps to NCM notification work

1. Generate JSONL locally (`Activity` must be the real one: `ADD_SERVICE`, `ACTIVE_SUBSCR`, not a guess).
2. Copy or SCP the jsonl + manifest to the inbox on the host that can reach brokers.
3. `BROKER_URL` from that host's `env.properties` or `bootstrap.servers`.
4. `kafka-console-producer.sh` with `ssl.client.properties`.
5. Prove it with `--time -1` offset increase, then the consumer group lag, then the business table (future charge / view), not only "producer returned".

A successful producer ack means the broker stored the record. It does not mean NCM applied the offer.

---

## 11. Command card

```bash
kafka-topics.sh --bootstrap-server "$BROKER_URL" --command-config "$SSL_CFG" --describe --topic "$TOPIC"
kafka-get-offsets.sh --bootstrap-server "$BROKER_URL" --command-config "$SSL_CFG" --topic "$TOPIC" --time -1
kafka-consumer-groups.sh --bootstrap-server "$BROKER_URL" --command-config "$SSL_CFG" --describe --group <GROUP>
kafka-console-consumer.sh --bootstrap-server "$BROKER_URL" --consumer.config "$SSL_CFG" --topic "$TOPIC" --partition 0 --offset <N> --max-messages 1
kafka-console-producer.sh --bootstrap-server "$BROKER_URL" --producer.config "$SSL_CFG" --topic "$TOPIC" < one.json
```
