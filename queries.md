# OpenSearch Queries Used

All requests were run in **OpenSearch Dashboards > Dev Tools** (OpenSearch 2.19.7).

## 1. Check the installation

```
GET /
```

Returns the OpenSearch version and cluster name. A `200 - OK` response means OpenSearch is running.

```
GET _cluster/health
```

Shows cluster status. `green` means all shards and replicas are assigned.

## 2. Create the index

```
PUT sample-logs
{
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "ip":        { "type": "keyword" },
      "user":      { "type": "keyword" },
      "event":     { "type": "keyword" },
      "message":   { "type": "text" }
    }
  }
}
```

Creates an empty index with five fields. `keyword` fields are matched exactly. The `text` field is searched by words.

## 3. Index the sample logs

```
POST _bulk?refresh=true
{ "index": { "_index": "sample-logs" } }
{ "timestamp": "2026-10-22T10:01:00", "ip": "203.0.113.5", "user": "admin", "event": "login_failed", "message": "Failed password for admin" }
{ "index": { "_index": "sample-logs" } }
{ "timestamp": "2026-10-22T10:01:05", "ip": "203.0.113.5", "user": "admin", "event": "login_failed", "message": "Failed password for admin" }
{ "index": { "_index": "sample-logs" } }
{ "timestamp": "2026-10-22T10:01:10", "ip": "203.0.113.5", "user": "admin", "event": "login_failed", "message": "Failed password for admin" }
{ "index": { "_index": "sample-logs" } }
{ "timestamp": "2026-10-22T10:01:15", "ip": "203.0.113.5", "user": "admin", "event": "login_failed", "message": "Failed password for admin" }
{ "index": { "_index": "sample-logs" } }
{ "timestamp": "2026-10-22T10:01:20", "ip": "203.0.113.5", "user": "admin", "event": "login_success", "message": "Accepted password for admin" }
{ "index": { "_index": "sample-logs" } }
{ "timestamp": "2026-10-22T10:05:00", "ip": "198.51.100.7", "user": "alice", "event": "login_success", "message": "Accepted password for alice" }
{ "index": { "_index": "sample-logs" } }
{ "timestamp": "2026-10-22T10:06:00", "ip": "198.51.100.8", "user": "bob", "event": "login_success", "message": "Accepted password for bob" }
{ "index": { "_index": "sample-logs" } }
{ "timestamp": "2026-10-22T10:07:00", "ip": "198.51.100.7", "user": "alice", "event": "file_download", "message": "alice downloaded report.pdf" }
{ "index": { "_index": "sample-logs" } }
{ "timestamp": "2026-10-22T10:10:00", "ip": "192.0.2.44", "user": "root", "event": "login_failed", "message": "Failed password for root" }
{ "index": { "_index": "sample-logs" } }
{ "timestamp": "2026-10-22T10:10:04", "ip": "192.0.2.44", "user": "root", "event": "login_failed", "message": "Failed password for root" }
```

Stores 10 log entries. Check the result for `"errors": false`.

Check the total number of logs:

```
GET sample-logs/_count
```

Expected: `"count": 10`.

### Fixing duplicates

Running the bulk request twice stores every log twice, because the logs have no fixed ID. To clear the index and start again:

```
POST sample-logs/_delete_by_query?refresh=true
{
  "query": { "match_all": {} }
}
```

## 4. Search the logs

All failed logins:

```
GET sample-logs/_search
{
  "query": {
    "term": { "event": "login_failed" }
  }
}
```

Result: 6 hits.

Everything from one IP, in time order:

```
GET sample-logs/_search
{
  "query": {
    "term": { "ip": "203.0.113.5" }
  },
  "sort": [ { "timestamp": "asc" } ]
}
```

Result: 5 logs, four `login_failed` followed by one `login_success` for `admin`.

Failed logins, newest first, showing only `ip` and `user`:

```
GET sample-logs/_search
{
  "query": {
    "term": { "event": "login_failed" }
  },
  "sort": [ { "timestamp": "desc" } ],
  "_source": ["ip", "user"]
}
```

Failed logins for only the user `admin` (combining conditions with `bool`):

```
GET sample-logs/_search
{
  "query": {
    "bool": {
      "must":   [ { "term": { "event": "login_failed" } } ],
      "filter": [ { "term": { "user": "admin" } } ]
    }
  },
  "sort": [ { "timestamp": "desc" } ],
  "_source": ["ip", "user"]
}
```

Result: 4 hits.

## 5. Aggregation

Count failed logins per IP address:

```
GET sample-logs/_search
{
  "size": 0,
  "query": {
    "term": { "event": "login_failed" }
  },
  "aggs": {
    "failed_by_ip": {
      "terms": { "field": "ip" }
    }
  }
}
```

Result:

| IP address | Failed logins |
|---|---|
| 203.0.113.5 | 4 |
| 192.0.2.44 | 2 |

## Quick reference

| Query | Use |
|---|---|
| `match_all` | Return everything |
| `match` | Search words in a `text` field |
| `term` | Exact value in a `keyword` field |
| `range` | Numbers or dates between limits |
| `bool` | Combine several conditions |
| `terms` aggregation | Count per value |
