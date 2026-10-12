# Running the exporter on a cephadm cluster

On a cluster managed by cephadm, the exporter can run as a
[custom container service](https://docs.ceph.com/en/reef/cephadm/services/custom-container/),
so the orchestrator places it and restarts it like any other service. Written against the
Reef (18.2) documentation.

## 1. Create a read-only RGW user

```bash
radosgw-admin user create --uid=usage-exporter --display-name="Prometheus usage exporter"
radosgw-admin caps add --uid=usage-exporter --caps="buckets=read;metadata=read;usage=read;users=read"
radosgw-admin user info --uid=usage-exporter | jq -r '.keys[0] | .access_key, .secret_key'
```

## 2. Fill in the spec

Edit [`rgw-usage-exporter.yaml`](rgw-usage-exporter.yaml):

- `image`: pin a tag built from a known commit. The image workflow tags every build
  `sha-<short sha>`; avoid `latest`, which changes under you.
- `RADOSGW_SERVER`: an RGW endpoint. `ceph orch ps --daemon-type rgw` lists the RGW daemons
  with their hosts and ports.
- `ACCESS_KEY`, `SECRET_KEY`: the keys from step 1.
- `TIMEOUT`: seconds per request to RGW (default 60). Every scrape lists all buckets and
  fetches each user, so raise it on large clusters, and check any load balancer's server
  timeout too.
- `placement`: one instance is enough; use `hosts:` instead of `count:` to pin it to a host.

`--net=host` in `args` puts the exporter on the host's network, so Prometheus can reach it at
`<host>:9242`. `ports` only opens that port in the host firewall.

## 3. Deploy and check

```bash
ceph orch apply -i rgw-usage-exporter.yaml
ceph orch ps --service_name container.rgw-usage-exporter
curl -s http://<host>:9242/metrics | grep '^radosgw_usage_bucket_bytes' | head
```

## 4. Scrape it

Add a job to your Prometheus. A scrape walks every bucket and user, so scrape rarely and allow
it time:

```yaml
- job_name: radosgw_usage
  scrape_interval: 5m
  scrape_timeout: 4m
  static_configs:
    - targets: ['<host>:9242']
```

## Updating and removing

- To change the image or settings, edit the spec and run `ceph orch apply -i` again. If the
  daemon doesn't restart, run `ceph orch redeploy container.rgw-usage-exporter`.
- To remove it: `ceph orch rm container.rgw-usage-exporter`, then
  `radosgw-admin user rm --uid=usage-exporter`.

## Things to know

- **Secrets in the spec:** cephadm stores the spec, including `SECRET_KEY`, in the monitor's
  config store, and `ceph orch ls --export` prints it. That is why the user in step 1 has
  read-only caps.
- **The usage log:** bucket and user metrics (`radosgw_usage_bucket_*`, `radosgw_usage_user_*`)
  come from separate bucket and user stats requests, so they should work without it. The operation and traffic metrics
  (`radosgw_usage_ops_total`, `radosgw_usage_sent_bytes_total`,
  `radosgw_usage_received_bytes_total`) need `rgw_enable_usage_log`; see the main README.
- **Labels:** `radosgw_user_metadata` carries each user's `display_name` and `email` as labels,
  so they end up in Prometheus.
