# KUSAMA Validator Simple Monitoring #

## Simple logging for KUSAMA and POLKADOT Validator based on the level="info" logs by the application. ##
### KUSAMA GRAFANA LOKI PROMTAIL ###
* KUSAMA is WILD!
* Grafana is a feature-rich metrics dashboard and graph editor
* Loki is the logging engine.
* Promtail sends logs to Loki

Distributor ID:	Debian
Description:	Debian GNU/Linux 10 (buster)
Release:	10
Codename:	buster
  
## Installation ##
### If you already running Grafana just add Loki as a Data Source and install the Promtail agent on the KUSAMA/PolkaDOT 'Step 3'. ###
### Grafana dashboard can be installed from https://grafana.com/grafana/dashboards/14899 ###

### Dashboard Preview ###
![KSMNETWORK](https://singular.rmrk.app/_next/image?url=https%3A%2F%2Frmrk.mypinata.cloud%2Fipfs%2Fbafybeihumnzmwxvvq7foexpebhpaoqcqnpykqubq44yeoxmddidfvnlapy&w=3840&q=90)

* Logs are obtained by the scrapping job "journal" from the systemd-journal, 
* The relabel_configs will provide you with a simple "unit" & "hostname" labels where you can separate multi-nodes by hostname 
* as a unique value. Most of the logs are dropped since this monitoring is focused only on the PolkaDOT default logs level="info" 
* but you can always drop more if you detect some.. by adding a match stage to the pipeline with a selector unit 
* '{job="systemd-journal", unit="<UNIT.NAME>"}'
```
  - match:
      selector: '{job="systemd-journal", unit="init.scope"}'
      action: drop
      drop_counter_reason: promtail_logs
```
* Logging... well regex ".*", shall explain everything.
* Grafana Dashboard is quite simple as it looks, more like the host terminal with some extra visual statistics,
* don't forget the "{{hostname}}" the label is quite helpful on multi-node setups. 
* Query limit configuration in the setting by default is 1000 if you want to adjust it, go do it!
* Labels set at the queries and then they are transformed to fields, filtered by names to suite the graphs needs.


### Simple query for Unit Service Logs ###
* https://grafana.com/docs/loki/latest/logql/
```
=: exactly equal
!=: not equal
=~: regex matches
!~: regex does not match
Regex log stream examples:
```
* Example
```
{unit="polkadot.service"} |= " Idle "
```

---
### Grafana ###
```
sudo apt-get install -y apt-transport-https \
&& sudo apt-get install -y software-properties-common wget \
&& wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add - \
&& echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list \
&& sudo apt-get update \
&& sudo apt-get install grafana \
&& sudo systemctl daemon-reload \
&& sudo systemctl start grafana-server \
&& sudo systemctl status grafana-server \
&& sudo systemctl enable grafana-server.service
```

---
### Loki ###
```
apt install unzip \
&& mkdir /opt/loki \
&& cd /opt/loki
```

```
wget https://github.com/grafana/loki/releases/download/v2.2.1/loki-linux-amd64.zip \
&& sudo unzip loki-linux-amd64.zipsudo unzip loki-linux-amd64.zip
```

```
cat <<EOF >/etc/systemd/system/loki.service
[Unit] 
Description=Loki service 
After=network.target 
 
[Service] 
Type=simple 
#User=loki 

ExecStart=/opt/loki/loki-linux-amd64 -config.file /opt/loki/loki-local-config.yaml 
Restart=always 
 
[Install] 
WantedBy=multi-user.target
EOF
```

```
cat <<EOF>/opt/loki/loki-local-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

ingester:
  wal:
    enabled: true
    dir: /tmp/wal
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 1h       
  max_chunk_age: 1h           
  chunk_target_size: 1048576  
  chunk_retain_period: 30s    
  max_transfer_retries: 0
schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /tmp/loki/boltdb-shipper-active
    cache_location: /tmp/loki/boltdb-shipper-cache
    cache_ttl: 24h
    shared_store: filesystem
  filesystem:
    directory: /tmp/loki/chunks

compactor:
  working_directory: /tmp/loki/boltdb-shipper-compactor
  shared_store: filesystem

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s

ruler:
  storage:
    type: local
    local:
      directory: /tmp/loki/rules
  rule_path: /tmp/loki/rules-temp
  alertmanager_url: http://localhost:9093
  ring:
    kvstore:
      store: inmemory
  enable_api: true
EOF
```

```
sudo systemctl enable loki.service \
&& sudo systemctl start loki.service \
&& sudo systemctl status loki.service 
```

---
### Promtail ###
```
apt install unzip \
&& sudo mkdir /opt/promtail \
&& cd /opt/promtail \
&& wget https://github.com/grafana/loki/releases/download/v2.2.1/promtail-linux-amd64.zip \
&& sudo unzip promtail-linux-amd64.zip
```

```
cat <<EOF>/opt/promtail/polkadot-config.yaml
server:
  http_listen_port: 8579
  grpc_listen_port: 0
clients:
  - url: http://<HOST>:3100/loki/api/v1/push
    tenant_id: 'v1'
positions:
  filename: /tmp/positions.yaml
scrape_configs:
- job_name: journal
  journal:
    max_age: 60s
    labels:
      job: systemd-journal
  relabel_configs:
    - source_labels: ['__journal__systemd_unit']
      target_label: 'unit'
    - source_labels: ['__journal__hostname']
      target_label: 'hostname'
  pipeline_stages:
  # Drop this logs
  - match:
      selector: '{job="systemd-journal", unit="ifup@eth0.service"}'
      action: drop
      drop_counter_reason: ifup_logs
  - match:
      selector: '{job="systemd-journal", unit="init.scope"}'
      action: drop
      drop_counter_reason: promtail_logs
  - match:
      selector: '{job="systemd-journal", unit="cron.service"}'
      action: drop
      drop_counter_reason: cron_logs
  - match:
      selector: '{job="systemd-journal", unit="ssh.service"}'
      action: drop
      drop_counter_reason: ssh_logs
  - match:
      selector: '{job="systemd-journal", unit="systemd-logind.service"}'
      action: drop
      drop_counter_reason: systemd_logs
  # Get this logs
  - match:
      selector: '{job="systemd-journal", unit="polkadot.service"}'
      stages:
      - regex:
          expression: '.*(<output>)'
      - labels:
          level:

      - output:
          source: output
EOF
```

```
cat <<EOF >/etc/systemd/system/promtail.service
[Unit] 
Description=Promtail service 
After=network.target 
 
[Service] 
Type=simple 
#User=promtail 

ExecStart=/opt/loki/promtail-linux-amd64 -config.file /opt/promtail/polkadot-config.yaml 
Restart=always 
 
[Install] 
WantedBy=multi-user.target
EOF
```

```
systemctl enable promtail.service \
&& systemctl start promtail.service \
&& systemctl status promtail.service
```

---
### Grafana Dashboard Template ###
```
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "target": {
          "limit": 100,
          "matchAny": false,
          "tags": [],
          "type": "dashboard"
        },
        "type": "dashboard"
      },
      {
        "datasource": null,
        "enable": true,
        "expr": "{job=\"systemd-journal\", unit=\"polkadot.service\", hostname=~\".*\"}",
        "iconColor": "#00b781",
        "name": "Hostname",
        "tagKeys": "{{hostname}}",
        "target": {},
        "textFormat": "{{hostname}}",
        "titleFormat": "Name"
      }
    ]
  },
  "description": "A basic monitoring based on the Polkadot log level='info' ",
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": 4,
  "links": [
    {
      "asDropdown": false,
      "icon": "external link",
      "includeVars": false,
      "keepTime": false,
      "tags": [],
      "targetBlank": true,
      "title": "For support & nomination please use the following KUSAMA (KSM) Crypto Address: H1bSKJxoxzxYRCdGQutVqFGeW7xU3AcN6vyEdZBU7Qb1rsZ",
      "tooltip": "KSM",
      "type": "link",
      "url": "https://ksm.network"
    }
  ],
  "panels": [
    {
      "datasource": null,
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "noValue": "No Blocks W#@% Dude? ",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "dark-red",
                "value": null
              },
              {
                "color": "red",
                "value": 0
              },
              {
                "color": "#EAB839",
                "value": 1
              },
              {
                "color": "#6ED0E0",
                "value": 2
              },
              {
                "color": "#EF843C",
                "value": 3
              },
              {
                "color": "#E24D42",
                "value": 4
              },
              {
                "color": "#1F78C1",
                "value": 5
              },
              {
                "color": "#BA43A9",
                "value": 10
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 10,
        "w": 3,
        "x": 0,
        "y": 0
      },
      "id": 12,
      "options": {
        "colorMode": "background",
        "graphMode": "area",
        "justifyMode": "center",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "/^imported$/",
          "values": false
        },
        "text": {},
        "textMode": "value_and_name"
      },
      "pluginVersion": "8.1.1",
      "targets": [
        {
          "expr": "count_over_time( {unit=\"polkadot.service\", hostname=~\".*\"} |= \"Imported\" | regexp \"(?P<time>\\\\d+-\\\\d+-\\\\d+\\\\s+\\\\d+:\\\\d+:\\\\d+)\\\\W+\\\\w+\\\\W+(?P<imported>[0-9]+)\"[6s])",
          "hide": false,
          "legendFormat": "",
          "refId": "C"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Imported Blocks Per/s",
      "transformations": [
        {
          "id": "labelsToFields",
          "options": {}
        },
        {
          "id": "organize",
          "options": {
            "excludeByName": {
              "id": true,
              "job": true,
              "line": true,
              "ts": true,
              "tsNs": true,
              "unit": true
            },
            "indexByName": {},
            "renameByName": {
              "Time": "time",
              "Value": "imported",
              "id": "Id",
              "imported": "Imported",
              "job": "Job",
              "line": "Line",
              "time": "time",
              "ts": "TS",
              "tsNs": "tsNs",
              "unit": "Unit"
            }
          }
        }
      ],
      "type": "stat"
    },
    {
      "datasource": null,
      "description": "",
      "fieldConfig": {
        "defaults": {
          "color": {
            "fixedColor": "#27b27e",
            "mode": "fixed"
          },
          "mappings": [],
          "noValue": "W$@#?",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "#580000",
                "value": null
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 10,
        "w": 18,
        "x": 3,
        "y": 0
      },
      "id": 10,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "center",
        "orientation": "vertical",
        "reduceOptions": {
          "calcs": [
            "last"
          ],
          "fields": "/.*/",
          "limit": 1,
          "values": false
        },
        "text": {},
        "textMode": "value_and_name"
      },
      "pluginVersion": "8.1.1",
      "repeat": null,
      "targets": [
        {
          "expr": "{unit=\"polkadot.service\", hostname=~\".*\"} |= \" Idle \" | regexp \"^(?s)(?P<time>\\\\S+\\\\d+\\\\s+\\\\d+:\\\\d+:\\\\d+) \\\\W+\\\\s? Idle\\\\W+(?P<peers>\\\\d+) peers\\\\)\\\\, best\\\\:\\\\s+\\\\W(?P<best>\\\\d+)\\\\s+\\\\W(?P<hash>[a-zA-Z0-9]+\\\\W[a-zA-Z0-9]+)\\\\W\\\\, finalized\\\\s\\\\W(?P<finalized>[0-9]+)\\\\s+\\\\W(?P<fhash>[a-zA-Z0-9]+\\\\W[a-zA-Z0-9]+)\\\\W+(?P<download>[0-9]+\\\\W[0-9]+[a-zA-Z]+\\\\Ws\\\\s+)\\\\W+(?P<upload>[0-9]+\\\\W[0-9]+[a-zA-Z]+\\\\Ws)\"",
          "instant": false,
          "legendFormat": "{{hostname}}",
          "range": true,
          "refId": "A"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Statistics",
      "transformations": [
        {
          "id": "labelsToFields",
          "options": {}
        },
        {
          "id": "filterFieldsByName",
          "options": {
            "include": {
              "names": [
                "upload",
                "download",
                "best",
                "finalized"
              ]
            }
          }
        },
        {
          "id": "calculateField",
          "options": {
            "alias": "Count",
            "mode": "reduceRow",
            "reduce": {
              "include": [
                "best"
              ],
              "reducer": "sum"
            },
            "replaceFields": false
          }
        },
        {
          "id": "organize",
          "options": {
            "excludeByName": {},
            "indexByName": {
              "Count": 5,
              "best": 1,
              "download": 4,
              "finalized": 2,
              "peers": 0,
              "upload": 3
            },
            "renameByName": {
              "Count": "Blocks Counter",
              "best": "Best Block",
              "download": "Download",
              "finalized": "Finalized Block",
              "peers": "Peers",
              "upload": "Upload"
            }
          }
        }
      ],
      "type": "stat"
    },
    {
      "datasource": null,
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "noValue": "No Peers",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "#dd051b",
                "value": null
              },
              {
                "color": "semi-dark-red",
                "value": 5
              },
              {
                "color": "dark-orange",
                "value": 10
              },
              {
                "color": "light-yellow",
                "value": 15
              },
              {
                "color": "yellow",
                "value": 25
              },
              {
                "color": "dark-yellow",
                "value": 35
              },
              {
                "color": "dark-green",
                "value": 50
              },
              {
                "color": "semi-dark-green",
                "value": 65
              },
              {
                "color": "green",
                "value": 80
              },
              {
                "color": "purple",
                "value": 100
              },
              {
                "color": "dark-purple",
                "value": 200
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 10,
        "w": 3,
        "x": 21,
        "y": 0
      },
      "id": 14,
      "options": {
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "last"
          ],
          "fields": "/^peers$/",
          "limit": 3,
          "values": false
        },
        "showThresholdLabels": true,
        "showThresholdMarkers": true,
        "text": {}
      },
      "pluginVersion": "8.1.1",
      "targets": [
        {
          "expr": "{unit=\"polkadot.service\", hostname=~\".*\"} |= \" Idle \" | regexp \"^(?s)(?P<time>\\\\S+\\\\d+\\\\s+\\\\d+:\\\\d+:\\\\d+) \\\\W+\\\\s? Idle\\\\W+(?P<peers>\\\\d+) peers\\\\)\\\\, best\\\\:\\\\s+\\\\W(?P<best>\\\\d+)\\\\s+\\\\W(?P<hash>[a-zA-Z0-9]+\\\\W[a-zA-Z0-9]+)\\\\W\\\\, finalized\\\\s\\\\W(?P<finalized>[0-9]+)\\\\s+\\\\W(?P<fhash>[a-zA-Z0-9]+\\\\W[a-zA-Z0-9]+)\\\\W+(?P<download>[0-9]+\\\\W[0-9]+[a-zA-Z]+\\\\Ws\\\\s+)\\\\W+(?P<upload>[0-9]+\\\\W[0-9]+[a-zA-Z]+\\\\Ws)\"",
          "refId": "A"
        }
      ],
      "title": "Peers",
      "transformations": [
        {
          "id": "labelsToFields",
          "options": {}
        },
        {
          "id": "organize",
          "options": {
            "excludeByName": {
              "best": true,
              "download": true,
              "fhash": true,
              "finalized": true,
              "hash": true,
              "id": true,
              "job": true,
              "line": true,
              "ts": true,
              "tsNs": true,
              "unit": true,
              "upload": true
            },
            "indexByName": {
              "best": 3,
              "download": 4,
              "fhash": 8,
              "finalized": 2,
              "hash": 10,
              "id": 7,
              "job": 12,
              "line": 6,
              "peers": 1,
              "time": 0,
              "ts": 11,
              "tsNs": 13,
              "unit": 9,
              "upload": 5
            },
            "renameByName": {}
          }
        }
      ],
      "type": "gauge"
    },
    {
      "datasource": null,
      "gridPos": {
        "h": 26,
        "w": 13,
        "x": 0,
        "y": 10
      },
      "id": 8,
      "options": {
        "dedupStrategy": "none",
        "enableLogDetails": true,
        "prettifyLogMessage": false,
        "showCommonLabels": true,
        "showLabels": true,
        "showTime": false,
        "sortOrder": "Descending",
        "wrapLogMessage": false
      },
      "pluginVersion": "8.1.1",
      "targets": [
        {
          "expr": "{unit=\"polkadot.service\"} != \" Dispute \" != \" Mismatch \" != \"error=\"",
          "maxLines": null,
          "refId": "A"
        }
      ],
      "title": "LOGS",
      "type": "logs"
    },
    {
      "datasource": null,
      "gridPos": {
        "h": 26,
        "w": 11,
        "x": 13,
        "y": 10
      },
      "id": 17,
      "options": {
        "dedupStrategy": "none",
        "enableLogDetails": true,
        "prettifyLogMessage": false,
        "showCommonLabels": true,
        "showLabels": true,
        "showTime": false,
        "sortOrder": "Descending",
        "wrapLogMessage": false
      },
      "pluginVersion": "8.1.1",
      "targets": [
        {
          "expr": "{unit=\"polkadot.service\"}  != \" Idle \" != \"Dispute coordinator confirmation lost\" != \"Imported\" != \"Discovered new external address for our node\"",
          "maxLines": null,
          "refId": "A"
        }
      ],
      "title": "Unknown logs ",
      "type": "logs"
    }
  ],
  "refresh": "",
  "schemaVersion": 30,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-5m",
    "to": "now"
  },
  "timepicker": {
    "nowDelay": "1m"
  },
  "timezone": "",
  "title": "Kusama Validator Monitoring With Grafana Loki & Promtail",
  "uid": "skglLKzRz",
  "version": 130
}
```

## Stress less project ###
* Grafana Cloud Monitoring for KUSAMA/PolkaDOT Validators with a single agent installation, PROMTAIL v2.3.0
* No need to install Grafana
* No need to setup Loki
* Serveless Setup
* URL https://github.com/TGReaper/kusama-cloud

---
# For Support && Nominations #
* Display name. KSMNETWORK && KSMNETWORK-WEST 
* Email w3f@ksm.network
* Riot @gtoocool:matrix.org

* KUSAMA (KSM) Address
* ```H1bSKJxoxzxYRCdGQutVqFGeW7xU3AcN6vyEdZBU7Qb1rsZ```

* PolkaDOT (DOT) Address:
* ```15FxvBFDd3X7H9qcMGqsiuvFYEg4D3mBoTA2LQufreysTHKA```

* https://ksm.network
