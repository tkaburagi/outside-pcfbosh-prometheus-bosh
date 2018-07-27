# outside-pcfbosh-prometheus-bosh
![](https://github.com/tkaburagi/outside-pcfbosh-prometheus-bosh/blob/master/diagram.png)

```console
bosh create-env
```

```console
bosh2 alias-env 
export BOSH_CLIENT=admin
export BOSH_CLIENT_SECRET
bosh2 -e bosh env
```

```console
bosh2 -e bosh-1 update-cloud-config
bosh2 -e bosh-1 upload-stemcel
```

```console
uaac target
uaac token client
uaac client add firehost_exporter
uaac client add cf_exporter
uaac target
uaac token client
uaac client add bosh_exporter
```
https://github.com/bosh-prometheus/node-exporter-boshrelease

```console
bosh2 -e bosh -d prometheus deploy manifests/prometheus.yml \
  --vars-store tmp/deployment-vars.yml \
  -o manifests/operators/monitor-bosh.yml \
  -o manifests/operators/monitor-cf.yml \
  -o manifests/operators/monitor-node.yml \
  -o pcf-colocate-firehose_exporter.yml \
  -o pcf-local-exporter.yml \
  -v bosh_url=\
  -v bosh_username= \
  -v bosh_password=u \
  --var-file bosh_ca_cert= \
  -v metrics_environment=pcflab \
  -v metron_deployment_name=cf \
  -v system_domain= \
  -v uaa_bosh_exporter_client_secret=prometheus-client-secret \
  -v uaa_clients_cf_exporter_secret=prometheus-client-secret \
  -v uaa_clients_firehose_exporter_secret=prometheus-client-secret \
  -v traffic_controller_external_port=443 \
  -v skip_ssl_verify=true
```
