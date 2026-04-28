# crowdsec-grafana-dashboard
Based on freedf dashboard - Fixes the amount of "bans" shown in the dashboard (https://freefd.github.io/articles/8_cyber_threat_insights_with_crowdsec_victoriametrics_and_grafana/)

## Create the VictoriaMetrics server (if you don't have one)
To make VictoriaMetrics work in your configuration, please check that it can be reached by Grafana and by Crowdsec - I recommand putting the container in the same network as Grafana and Crowdsec. Also set a static volume to store the notifications sent by Crowdsec).

Here an example of my container, running alongside of Traefik and Crowdsec in a Docker Compose:
```
victoriametrics:
    image: victoriametrics/victoria-metrics:latest
    networks:
      - monitoring
    volumes:
      - victoria_data:/storage
    command:
      - '-storageDataPath=/storage'
      - '-httpListenAddr=:8428'
```

## Create the notification to push logs to VictoriaMetrics:
Add to /etc/crowdsec/notifications/http.yaml file the following configuration:
```yaml
type: http
name: victoria_metrics
log_level: info
format: >
  {{- range $Alert := . -}}
  {{- range .Decisions -}}
  {"metric":{"__name__":"cs_lapi_decision","instance":"traefik-evangelion","country":"{{$Alert.Source.Cn}}","asname":"{{$Alert.Source.AsName}}","asnumber":"{{$Alert.Source.AsNumber}}","latitude":"{{$Alert.Source.Latitude}}","longitude":"{{$Alert.Source.Longitude}}","iprange":"{{$Alert.Source.Range}}","scenario":"{{.Scenario}}","type":"{{.Type}}","duration":"{{.Duration}}","scope":"{{.Scope}}","ip":"{{.Value}}"},"values": [1],"timestamps":[{{now|unixEpoch}}000]}
  {{- end }}
  {{- end -}}
url: http://<victoriametrics-container>:8428/api/v1/import
method: POST
headers:
  Content-Type: application/json
```

Still in Crowdsec configuration, in /etc/crowdsec/profiles.yaml, add the notification configuration we have created to be triggered:
```yaml
name: default_ip_remediation
...
#duration_expr: Sprintf('%dh', (GetDecisionsCount(Alert.GetValue()) + 1) * 4)
notifications:
  - victoria_metrics
on_success: break
---
name: default_range_remediation
...
notifications:
  - victoria_metrics
on_success: break
```

## Add VictoriaMetrics to Grafana
In Grafana, create a new Data Source using the "Prometheus" Data Source. Set the address of VictoriaMetrics (depending on the container name) with the port 8428.

## Import the Dashboard in Grafana
Import the file or copy-paste the content of it, and deploy the dashboard. Select the Data Source we have created before.

Only new bans are visible on the dashboard.
