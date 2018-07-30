# prometheus-for-pcf
Sample Enviroment. This doc is for deloying prometheus-boshrelease to monitor PCF.
![](https://github.com/tkaburagi/outside-pcfbosh-prometheus-bosh/blob/master/diagram.png)

## Creating BOSH ENV on GCP
Following [instruction](https://bosh.io/docs/init-google/).
```bash
$ bosh2 create-env bosh-deployment/bosh.yml \
    --state=state.json \
    --vars-store=creds.yml \
    -o bosh-deployment/gcp/cpi.yml \
    -v director_name=d-bosh \
    -v internal_cidr=10.0.0.0/24 \
    -v internal_gw=10.0.0.1 \
    -v internal_ip=10.0.0.6 \
    --var-file gcp_credentials_json=<KEY_JSON> \
    -v project_id=<GCP_PROJECT_ID> \
    -v zone=<ZONE> \
    -v tags=[d-bosh] \
    -v network=<GCP_VPC> \
    -v subnetwork=<GCP_VPC_SUBNET>
```

Logging into BOSH ENV. `cred.yml` should be stored the directory which is set as `--vars-store` above.
```bash
$ bosh2 alias-env gcpbosh -e 10.0.0.6 --ca-cert <(bosh2 int /home/tkaburagi/creds.yml --path /director_ssl/ca)
$ export BOSH_CLIENT=admin
$ export BOSH_CLIENT_SECRET=`bosh2 int /home/tkaburagi/creds.yml --path /admin_password`
$ bosh2 -e gcpbosh env
Using environment '10.0.0.6' as client 'admin'

Name      d-bosh
UUID      c672a233-2ab8-4710-ba7d-1845e223419b
Version   266.7.0 (00000000)
CPI       google_cpi
Features  compiled_package_cache: disabled
          config_server: disabled
          dns: disabled
          snapshots: disabled
User      admin

Succeeded
```

## Updating Cloud Config
Let's update Cloud Config Sample one is [here](https://github.com/tkaburagi/outside-pcfbosh-prometheus-bosh/blob/master/cloud-config.yml).
```bash
$ bosh2 -e gcpbosh update-cloud-config
$ bosh2 -e gcpbosh upload-stemcel
```

## Generating UAA Clients for Exporters
According to an official [document](https://github.com/bosh-prometheus/prometheus-boshrelease#monitoring-cloud-foundry), you should apply `add-prometheus-uaa-clients.yml` op file but for PCF, this will be refreshed by `Apply Changes` on Ops Manager, so you need to add clients by manual. 
`<UAA_CLIENT_SECRET>` is on `PAS tile -> Credentials -> UAA -> Admin Client Credentials`. `<UAA_LOGIN_CLIENT_CREDENTIAL>` is on `BOSH tile -> Uaa Admin User Credentials`. `<UAA_ADMIN_USER_CREDENTIAL>` is on `BOSH tile -> Uaa Login Client Credentials`.
```bash
$ uaac target https://uaa.<SYSTEM_DOMAIN> --skip-ssl-validation
$ uaac token client get admin -s <UAA_CLIENT_SECRET>

$ uaac client add firehose_exporter \
  --name firehose_exporter \
  --secret prometheus-client-secret \
  --authorized_grant_types client_credentials,refresh_token \
  --authorities doppler.firehose

$ uaac client add cf_exporter \
  --name cf_exporter  \
  --secret prometheus-client-secret \
  --authorized_grant_types client_credentials,refresh_token \
  --authorities cloud_controller.admin

$ uaac target https://BOSH_DIRECTOR:8443 --skip-ssl-validation
$ uaac token owner get login -s <UAA_LOGIN_CLIENT_CREDENTIAL>
User name:  admin
Password:  <UAA_ADMIN_USER_CREDENTIAL>

$ uaac client add prometheus-bosh \
  --name prometheus-bosh \
  --secret prometheus-client-secret \
  --authorized_grant_types client_credentials,refresh_token \
  --authorities bosh.read \
  --scope bosh.read
```

## Installing node_exporter. 
See below.
https://github.com/bosh-prometheus/node-exporter-boshrelease

Preparations have been done! It's time to install Prometheus! `<UAA_ADMIN_USER_CREDENTIAL>>` is on `BOSH tile -> Uaa Admin User Credentials`. 
## Deploying Prometheus
```bash
$ bosh2 -e gcpbosh -d prometheus deploy manifests/prometheus.yml \
  --vars-store tmp/deployment-vars.yml \
  -o manifests/operators/monitor-bosh.yml \
  -o manifests/operators/monitor-cf.yml \
  -o manifests/operators/monitor-node.yml \
  -o pcf-colocate-firehose_exporter.yml \
  -o pcf-local-exporter.yml \
  -v bosh_url=<PCF_BOSH_URL> \
  -v bosh_username=admin \
  -v bosh_password=<UAA_ADMIN_USER_CREDENTIAL> \
  --var-file bosh_ca_cert=<PATH_TO_OPSMAN_CERT> \
  -v metrics_environment=p-bosh \
  -v metron_deployment_name=cf \
  -v system_domain=<SYSTEM_DOMAIN> \
  -v uaa_bosh_exporter_client_secret=prometheus-client-secret \
  -v uaa_clients_cf_exporter_secret=prometheus-client-secret \
  -v uaa_clients_firehose_exporter_secret=prometheus-client-secret \
  -v traffic_controller_external_port=443 \
  -v skip_ssl_verify=true
```

## Login to Grafana
URL: `http://NGINX_PUBLIC_IP:3000`
USER: `admin`
PASSWORD: Retrive from `tmp/deployment-vars.yml`
