|||
|--|--|
|`.`||
|`├── Makefile`|Useful targets for development|
|`├── images`||
|`│   └── camunda-bpm`||
|`│       └── Dockerfile`|Install more jars like Prometheus exporter|
|`├── ingress-patch.yaml`|Configure ingress for your site|
|`├── kustomization.yaml`||
|`├── namespace.yaml`|Set the namespace for this installation|
|`├── platform`||
|`│   ├── config`||
|`│   │   └── prometheus-jmx.yaml`|Metrics exporter configuration|
|`│   ├── deployment.yaml`||
|`│   ├── ingress.yaml`||
|`│   ├── kustomization.yaml`||
|`│   ├── service.yaml`||
|`│   └── service-monitor.yaml`|For prometheus-operator|
|`├── site-data.yaml`|Set the FQDN|
|`└── skaffold.yaml`|Set the GCP project|
