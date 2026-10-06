# Local setup: Tesla energy telemetry

Open this when building dashboards on the energy Grafana. Values were correct when written;
confirm a uid or label before relying on it.

## Grafana

- Host: `https://grafana.mn.tesla.services` (Grafana 13).
- Auth: browser session cookie in `~/.grafana_cookie` (value only, read at run time). It
  expires after roughly 20 minutes; on 401 ask the user to refresh it.
- No image renderer installed: screenshot with Playwright and local Chrome (`channel="chrome"`).
- Telemetry folder uid: `StjiTFfIz`. Ask before writing there.

## Datasources

| Purpose | Name | uid |
|---|---|---|
| Prometheus eng | us-smf00-eng-energy | `ff0ge0ivlhq80e` |
| Prometheus prd | us-smf00-prd-energy | `cfpxg0he8s6iob` |
| Tools Prometheus eng (VAST quota, `vast_quota_*`) | prometheus-us-smf00-eng-energytools | `bepk5qnmvtjpcf` |
| Tools Prometheus prd | prometheus-us-smf00-prd-energytools | `feq8znx1ouu4gd` |
| App stats Influx eng | app-stats-eng-smf00 | `aehiougoe15hcb` |
| App stats Influx prd | app-stats-prd-smf00 | `dehiox39ulo8wd` |

Pick the Prometheus datasource with a datasource variable (regex
`/^us-smf00-(eng|prd)-energy$/`) and derive the rest from its name: `Environment`
(`us-smf00-eng`), `env` (`eng`), log index, tools and Influx datasources by regex.

## Dashboards to link

| Dashboard | uid | Variables |
|---|---|---|
| Pod Detail | `TusNyLKGk` | `cluster` (datasource), `namespace`, `pod_prefix`, `container` |
| Kafka Consumer Detailed (Burrow lag) | `skizAe_mz` | `datasource`, `consumers` (multi), `topics`, `cluster` |
| VAST bucket metrics | `bucketcapacity` (monitoring.tesla.com) | `bucket`, `DS_PROMETHEUS`, `cluster` |
| Common Scala / Basic Consumer | `common-scala-basic-consumer` | model for a shared library dashboard |

Pass the time range with `${__url_time_range}`.

## Link shapes

- Deploy tool (health cell): `https://eps-argocd.${Environment}-energytools.k8s.tesla.com/applications?search=${Environment}-energy-telemetry-<app>`
- Logs (errors cell): `https://splunk.teslamotors.com/en-US/app/search/search?q=search%20<url-encoded query>&earliest=-60m&latest=now`
  with index `eps_kube_eng` / `eps_kube_prod`, `kubernetes.labels.app=<container>`, and
  `level>=40` (levels are numeric: 30 info, 40 warn, 50 error).

## Reference dashboards

latest-api (stats table with health → ArgoCD and Splunk columns, system health timeline) and
kafka-sql-relay (processing rate and error rate per relay) are the house style to match.

## Labels

- Kubernetes metrics carry `namespace`, `pod`, `container`; aggregate by `container`.
- Burrow lag metrics are keyed by consumer group; lag in time is preferred over offsets.
