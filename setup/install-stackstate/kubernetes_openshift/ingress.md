---
description: SUSE Observability Self-hosted
---

# Expose SUSE Observability outside of the cluster

## Overview

SUSE Observability can be exposed with a Kubernetes Ingress resource. The example on this page shows how to configure an nginx-ingress controller using [Helm for SUSE Observability running on Kubernetes](ingress.md#configure-ingress-via-the-suse-observability-helm-chart). This page also documents which service/port combination to expose when using a different method of configuring ingress traffic.

When observing the cluster that also hosts SUSE Observability, the agent traffic can be kept entirely within the cluster itself by [changing the agent configuration](./ingress.md#agents-in-the-same-cluster) during agent installation.

## Configure ingress via the SUSE Observability Helm chart

The SUSE Observability Helm chart exposes an `ingress` section in its values. This is disabled by default. The example below shows how to use the Helm chart to configure an nginx-ingress controller with TLS encryption enabled. Note that setting up the controller itself and the certificates is beyond the scope of this document.

To configure the ingress for SUSE Observability, create a file `ingress_values.yaml` with contents like below. Replace `MY_DOMAIN` with your own domain \(that's linked with your ingress controller\) and set the correct name for the `tls-secret`. Consult the documentation of your ingress controller for the correct annotations to set. All fields below are optional, for example, if no TLS will be used, omit that section but be aware that SUSE Observability also doesn't encrypt the traffic.

```text
ingress:
  enabled: true
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  hosts:
    - host: stackstate.MY_DOMAIN
  tls:
    - hosts:
        - stackstate.MY_DOMAIN
      secretName: tls-secret
```

The thing that stands out in this file is the Nginx annotation to increase the allowed `proxy-body-size` to `50m` \(larger than any expected request\). By default, Nginx allows body sizes of maximum `1m`. SUSE Observability Agents and other data providers can sometimes send much larger requests. For this reason, you should make sure that the allowed body size is large enough, regardless of whether you are using Nginx or another ingress controller. Make sure to update the `baseUrl` in the values file generated during initial installation, it will be used by SUSE Observability to generate convenient installation instructions for the agent.

Include the `ingress_values.yaml` file when you run the `helm upgrade` command to deploy SUSE Observability:

```text
helm upgrade --install \
  --namespace "suse-observability" \
  --values "ingress_values.yaml" \
  --values $VALUES_DIR/suse-observability-values/templates/baseConfig_values.yaml \
  --values $VALUES_DIR/suse-observability-values/templates/sizing_values.yaml \  
suse-observability \
suse-observability/suse-observability
```

{% hint style="info" %}
This step assummes that [Generate `baseConfig_values.yaml` and `sizing_values.yaml`](./kubernetes_install.md#generate-baseconfig_values.yaml-and-sizing_values.yaml) was already executed.
{% endhint %}


## Configure Ingress Rule for Open Telemetry Traces via the SUSE Observability Helm chart

The SUSE Observability Helm chart exposes an `opentelemetry-collector` service in its values where a dedicated `ingress` can be created. This is disabled by default. The ingress needed for `opentelemetry-collector` purposed needs to support GRPC protocol. The example below shows how to use the Helm chart to configure an nginx-ingress controller with GRPC and  TLS encryption enabled. Note that setting up the controller itself and the certificates is beyond the scope of this document.

To configure the `opentelemetry-collector` ingress for SUSE Observability, create a file `ingress_otel_values.yaml` with contents like below. Replace `MY_DOMAIN` with your own domain \(that's linked with your ingress controller\) and set the correct name for the `otlp-tls-secret`. Consult the documentation of your ingress controller for the correct annotations to set. All fields below are optional, for example, if no TLS will be used, omit that section but be aware that SUSE Observability also doesn't encrypt the traffic.

```text
opentelemetry-collector:
  ingress:
    enabled: true
    annotations:
      nginx.ingress.kubernetes.io/proxy-body-size: "50m"
      nginx.ingress.kubernetes.io/backend-protocol: GRPC
    hosts:
      - host: otlp-stackstate.MY_DOMAIN
        paths:
          - path: /
            pathType: Prefix
            port: 4317
    tls:
      - hosts:
          - otlp-stackstate.MY_DOMAIN
        secretName: otlp-tls-secret
```

The thing that stands out in this file is the Nginx annotation to increase the allowed `proxy-body-size` to `50m` \(larger than any expected request\). By default, Nginx allows body sizes of maximum `1m`. SUSE Observability Agents and other data providers can sometimes send much larger requests. For this reason, you should make sure that the allowed body size is large enough, regardless of whether you are using Nginx or another ingress controller. Make sure to update the `baseUrl` in the values file generated during initial installation, it will be used by SUSE Observability to generate convenient installation instructions for the agent.

Include the `ingress_otel_values.yaml` file when you run the `helm upgrade` command to deploy SUSE Observability:

```text
helm upgrade \
  --install \
  --namespace "suse-observability" \
  --values "ingress_otel_values.yaml" \
  --values $VALUES_DIR/suse-observability-values/templates/baseConfig_values.yaml \
  --values $VALUES_DIR/suse-observability-values/templates/sizing_values.yaml \ 
suse-observability \
suse-observability/suse-observability
```

{% hint style="info" %}
This step assummes that [Generate `baseConfig_values.yaml` and `sizing_values.yaml`](./kubernetes_install.md#generate-baseconfig_values.yaml-and-sizing_values.yaml) was already executed.
{% endhint %}

## Configure via external tools

To make SUSE Observability accessible outside of the Kubernetes cluster it's installed in, it's enough to route traffic to port `8080` of the `<namespace>-stackstate-k8s-router` service. The UI of SUSE Observability can be accessed directly under the root path of that service (i.e. `http://<namespace>-stackstate-k8s-router:8080`) while agents will use the `/receiver` path (`http://<namespace>-stackstate-k8s-router:8080/receiver`).

Make sure to update the `baseUrl` in the values file generated during initial installation, it will be used by SUSE Observability to generate convenient installation instructions for the agent.

{% hint style="info" %}
When manually configuring an Nginx or similar HTTP server as reverse proxy make sure that it can proxy websockets as well. For Nginx this can be configured by including the following directives in the `location` directive:

```text
proxy_set_header Upgrade                 $http_upgrade;
proxy_set_header Connection              "Upgrade";
```
{% endhint %}

{% hint style="warning" %}
SUSE Observability itself doesn't use TLS encrypted traffic, TLS encryption is expected to be handled by the ingress controller or external load balancers.
{% endhint %}

## Agents in the same cluster

Agents that are deployed to the same cluster as SUSE Observability can of course use the external URL on which SUSE Observability is exposed, but it's also possible to configure the agent to directly connect to the SUSE Observability instance via the Kubernetes internal network only. To do that replace the value of the `'stackstate.url'` in the `helm install` command from the [Agent Kubernetes installation](../../../k8s-quick-start-guide.md) with the internal cluster URL for the router service (see also above): `http://<namespace>-suse-observability-router.<namespace>.svc.cluster.local:8080/receiver/stsAgent` (the `<namespace>` sections need to be replaced with the namespace of SUSE Observability). 

## See also

* [AKS \(learn.microsoft.com\)](https://learn.microsoft.com/en-us/azure/aks/ingress-tls?tabs=azure-cli)
* [EKS Official docs](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html) \(not using nginx\)
* [EKS blog post](https://aws.amazon.com/blogs/opensource/network-load-balancer-nginx-ingress-controller-eks/) \(using nginx\)

