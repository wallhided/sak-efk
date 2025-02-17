## Elastic Kibana Filebeat

https://helm.elastic.co/


## Prerequisite

We do not include xpack, so kibana does not require authorization and is available to everyone. It is recommended to enable the oauth module for nginx and set the settings for oauth!

Create Google OAuth keys
First, you need to create a Google OAuth Client:

* Go to https://console.developers.google.com/apis/credentials.
* Click Create Credentials, then click OAuth Client ID in the drop-down menu
* Enter the following:
```
    Application Type: Web Application
    Name: Grafana
    Authorized JavaScript Origins: https://grafana.mycompany.com
    Authorized Redirect URLs: https://grafana.mycompany.com/login/google
    Replace https://grafana.mycompany.com with the URL of your Grafana instance.
```
* Click Create
* Copy the Client ID and Client Secret from the ‘OAuth Client’ modal


## Example how add with module
```
module "efk" {
  depends_on = [module.argocd]
  source            = "github.com/provectus/sak-efk"
  cluster_name      = module.kubernetes.cluster_name
  argocd            = module.argocd.state
  domains           = local.domain
  kibana_conf       = {
    "ingress.annotations.kubernetes\\.io/ingress\\.class" = "nginx"
    "ingress.annotations.nginx\\.ingress\\.kubernetes\\.io/auth-url"    = "https://auth.example.com/oauth2/auth"
    "ingress.annotations.nginx\\.ingress\\.kubernetes\\.io/auth-signin" = "https://auth.example.com/oauth2/sign_in?rd=https://$host$request_uri"
  }
  filebeat_conf     = {
    setup.kibana:
      host: https://kibana.example.com:443
    setup.dashboards.enabled: true
    setup.template.enabled: true
    setup.template.name: "filebeat"
    setup.template.pattern: "filebeat-*"
    setup.template.settings:
      index.number_of_shards: 1
    setup.ilm.enabled: auto
    setup.ilm.check_exists: false
    setup.ilm.overwrite: true

    filebeat.modules:
      - module: system
        syslog:
          enabled: true
          #var.paths: ["/var/log/syslog"]
        auth:
          enabled: true
          #var.paths: ["/var/log/authlog"]      

    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      stream: all
      processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: "/var/log/containers/"

    filebeat.autodiscover:
      providers:
        - type: kubernetes
          hints.enabled: true
          templates:
            #Get logs only from pod with label logging=true
            - condition.equals:
                kubernetes.labels.logging: true
              config:
                - type: container
                  paths:
                    - /var/log/containers/*-${data.kubernetes.container.id}.log
                  exclude_lines: ["^\\s+[\\-`('.|_]"]  # drop asciiart lines
            #Parse logs with Nginx grok pattern from container with label logtype=nginx
            - condition.equals:
                kubernetes.labels.logtype: nginx
              config:
                - module: nginx
                  access:
                    enabled: true
                    var.paths: ["/var/log/nginx/access.log*"]
                  error:
                    enabled: true
                    var.paths: ["/var/log/nginx/error.log*"]

    processors:
    - add_cloud_metadata: ~
    - decode_json_fields:
        fields: ["message"]
        process_array: false
        max_depth: 1
        target: ""
        overwrite_keys: true
        add_error_key: true
    #Drop all logs without labels logging=true
    - drop_event:
        when:
          not:
            contains:
              kubernetes.pod.labels.logging: "true"
    #Drop health checks logs
    - drop_event:
        when:
          regexp:
            message: '(?i)kube-probe/1.18+'
    - drop_event:
        when:
          regexp:
            message: '(?i)ELB-HealthChecker/2.0'     
  }
}
```


## Requirements

No requirements.

## Providers

| Name | Version |
|------|---------|
| aws | n/a |
| helm | n/a |
| kubernetes | n/a |
| local | n/a |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| argocd | A set of values for enabling deployment through ArgoCD | `map(string)` | `{}` | no |
| cluster\_name | A name of the Amazon EKS cluster | `string` | `null` | no |
| config\_path | location of the kubeconfig file | `string` | `"~/.kube/config"` | no |
| domains | A list of domains to use for ingresses | `list(string)` | <pre>[<br>  "local"<br>]</pre> | no |
| elasticDataSize | Request pvc size for elastic volume data size | `string` | `"30Gi"` | no |
| elasticMinMasters | Number of minimum elasticsearch master nodes. Keep this number low or equals that Replicas | `string` | `"2"` | no |
| elasticReplicas | Number of elasticsearch nodes | `string` | `"3"` | no |
| elastic\_chart\_version | A Helm Chart version | `string` | `"7.10.1"` | no |
| elastic\_conf | A custom configuration for deployment | `map(string)` | `{}` | no |
| filebeat\_chart\_version | A Helm Chart version | `string` | `"7.10.1"` | no |
| filebeat\_conf | A custom configuration for deployment | `map(string)` | <pre>{<br>  "filebeat.inputs": {<br>    "paths": [<br>      "/var/log/containers/*.log"<br>    ],<br>    "stream": "all",<br>    "type": "container"<br>  }<br>}</pre> | no |
| kibana\_chart\_version | A Helm Chart version | `string` | `"7.10.1"` | no |
| kibana\_conf | A custom configuration for deployment | `map(string)` | `{}` | no |
| module\_depends\_on | For depends\_on queqe | `list` | `[]` | no |
| namespace | A name of the existing namespace | `string` | `""` | no |
| namespace\_name | A name of namespace for creating | `string` | `"logging"` | no |

## Outputs

No output.
