---
title: Metrics
type: docs
weight: 5
---

| Name| Type | Labels |
| --- | --- | --- |
| spegel_mirror_requests_total | Counter | `registry` </br> `cache` |
| spegel_mirror_last_success_timestamp_seconds | Gauge | |
| spegel_resolve_duration_seconds | Histogram | `router` |
| spegel_advertised_keys | Gauge | `registry` |
| http_request_duration_seconds | Histogram | `handler` <br/> `method` <br/> `code` |
| http_response_size_bytes | Histogram | `handler` <br/> `method` <br/> `code` |
| http_requests_inflight | Gauge | `handler` |
