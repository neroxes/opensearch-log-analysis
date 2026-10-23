# OpenSearch Sample Log Analysis Report

**Prepared by:** Thomson Jose\
**Date:** October 22, 2026

## 1. Objective

To install OpenSearch and OpenSearch Dashboards, store sample security logs, search them, and report what the logs show.

## 2. Environment

- **OpenSearch** version 2.19.7, running in Docker as a two-node cluster (`opensearch-node1`, `opensearch-node2`)
- **OpenSearch Dashboards**, running in Docker, used through the Dev Tools console
- Cluster health: green (all shards assigned, replicas on a separate node)

## 3. What Was Done

1. **Installation:** Installed OpenSearch and OpenSearch Dashboards with Docker and confirmed both work (`GET /` returned 200 OK).
2. **Practice:** Created a `movies` index and practised indexing, searching (`match`, `term`, `bool`) and the difference between `text` and `keyword` fields.
3. **Index creation:** Created an index named `sample-logs` with the fields `timestamp`, `ip`, `user`, `event` and `message`.
4. **Indexing:** Stored 10 sample log entries using the `_bulk` API. (A first attempt created duplicates, 20 entries instead of 10. I deleted them with `_delete_by_query` and re-indexed, and the final count was exactly 10.)
5. **Searching:** Used `term` queries, sorting and `_source` filtering to find failed logins.
6. **Aggregation:** Used a `terms` aggregation to count failed logins per IP address.

## 4. Findings

There were **6 failed logins** in total, from two IP addresses.

| Source IP | User targeted | Failed logins | Successful login afterwards? |
| --- | --- | --- | --- |
| 203.0.113.5 | admin | 4 | **Yes**, at 10:01:20 |
| 192.0.2.44 | root | 2 | No |

**Key finding:** IP `203.0.113.5` made four failed login attempts on the `admin` account at 10:01:00, 10:01:05, 10:01:10 and 10:01:15, exactly 5 seconds apart, then logged in successfully at 10:01:20. The even timing suggests an automated password-guessing tool. This pattern is consistent with a **successful brute-force attack**.

**Secondary finding:** IP `192.0.2.44` made two quick failed attempts on `root` at 10:10:00 and 10:10:04, with no success. This looks like a smaller, unsuccessful attempt.

**Normal activity:** The logins for `alice` and `bob` and the one file download by `alice` looked normal.

## 5. Recommendations

If this were real data, the following steps would be advised:

1. Treat the `admin` account as compromised: reset its password and review what it did after 10:01:20.
2. Block or investigate IP `203.0.113.5` (and `192.0.2.44`) at the firewall.
3. Turn on account lockout or rate limiting after a few failed logins.
4. Require multi-factor authentication for admin accounts.
5. Create an alert in OpenSearch for repeated `login_failed` events from the same IP.

## 6. Conclusion

OpenSearch and Dashboards are installed and working. Sample logs were indexed, searched and aggregated, and the analysis found a likely brute-force attack that succeeded against the `admin` account. This shows how OpenSearch can turn raw logs into clear security findings quickly.\


Proof\
&#91;image: Screenshot 2026-10-22 194056.png\]

&#91;image: image.png\]

&#91;image: image.png\]

*Note: the log data used here is made-up sample data created for practice. The IP addresses come from ranges reserved for documentation.*
