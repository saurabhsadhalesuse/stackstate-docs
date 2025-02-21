---
description: SUSE Observability
---

# SUSE Observability

## Introduction

SUSE Observability, formerly known as StackState can be used for Observability of your Kubernetes clusters and their workloads.

The installation of SUSE Observability, the SUSE Observability UI extension and the SUSE Observability Agents takes about 30 minutes in total.

## Getting help
For support please file a support case in [SUSE Customer Center (SCC)](https://scc.suse.com/).

## Prerequisites

### License key
A license key for SUSE Observability server can be obtained via the SUSE Customer Center and will be shown as "SUSE Rancher Prime - Observability Tech Preview" Registration Code. This license is valid until Oct, 31 2024. Before the end a license will be made available which is valid until the end of your Rancher Prime subscription.

### Requirements
To install SUSE Observability, ensure that the nodes have enough CPU and memory capacity. Below are the specific requirements.

There are different installation options available for SUSE Observability. It is possible to install SUSE Observability either in a High-Availability (HA) or single instance (non-HA) setup. The non-HA setup is recommended for testing purposes or small environments. For production environments, it is recommended to install SUSE Observability in a HA setup.

The HA production setup can support from 150 up to 500 Nodes (a Node is counted as<= 4 vCPU and <= 16GB Memory) under observation.
The Non-HA setup can support up to 100 Nodes under observation.

| | trial | 10 non-HA | 20 non-HA | 50 non-HA | 100 non-HA | 150 HA | 250 HA | 500 HA |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **CPU Requests** | 7,5 | 7,5 | 10,5 | 15 | 25 | 49 | 62 | 86.5 |
| **CPU Limits** | 16 | 16 | 21,5 | 30,5 | 50 | 103 | 128 | 176 |
| **Memory Requests** | 22Gi | 22Gi | 28Gi | 32.5Gi | 51Gi | 67Gi | 143Gi | 161.5Gi |
| **Memory Limits** | 23Gi | 23Gi | 29Gi | 33Gi | 51,5Gi | 131Gi | 147.5Gi | 166Gi |

{% hint style="info" %}
A trial setup is a 10 non-HA setup configured with a 3 day retention and lower disk space requirements.
{% endhint %}

These are just the upper and lower bounds of the resources that can be consumed by SUSE Observability in the different installation options. The actual resource usage will depend on the features used, configured resource limits and dynamic usage patterns, such as Deployment or DaemonSet scaling. For our Rancher Prime customers, we recommend to start with the default requirements and monitor the resource usage of the SUSE Observability components.

{% hint style="info" %}
The minimum requirements do not include spare CPU/Memory capacity to ensure smooth application rolling updates.
{% endhint %}

### Storage

SUSE Observability uses persistent volume claims for the services that need to store data. The default storage class for the cluster will be used for all services unless this is overridden by values specified on the command line or in a `values.yaml` file. All services come with a pre-configured volume size that should be good to get you started, but can be customized later using variables as required.

For our different installation profiles, the following are the defaulted storage requirements:

| | trial | 10 non-HA | 20 non-HA | 50 non-HA | 100 non-HA | 150 HA | 250 HA | 500 HA |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **Retention (days)** | 3 | 30 | 30 | 30 | 30 | 30 | 30 | 30 |
| **Storage requirement** | 125GB | 280GB | 420GB | 420GB | 600GB | 2TB | 2TB | 2.5TB |

For more details on the defaults used, see the page [Configure storage](/setup/install-stackstate/kubernetes_openshift/storage.md).

### Helm

SUSE Observability is installed through Helm, which needs to be installed with a minimum version of 3.13.1.

### The different components

#### SUSE Observability Server

This is the on-prem hosted server part of the installation. It contains a set of services to store observability data:

- Topology (StackGraph)
- Metrics (VictoriaMetrics)
- Traces (ClickHouse)
- Logs (ElasticSearch)

Next to this, it contains a set of services for all the observability tasks. e.g. Notifications, State management, Monitoring, etc.

#### SUSE Observability Agent

The lightweight SUSE Observability agent is installed on your downstream worker nodes. It collects and reports metrics, events, traces and logs, and it provides real-time observability and insights, enabling proactive monitoring and troubleshooting of your IT environment.

The SUSE Observability version of the Agent also uses eBPF as a lightweight way to monitor all your workloads and their communication. It also decodes the RED (Rate, Errors and Duration) signals for most of the common L7 protocols like TCP, HTTP, TLS, Redis, etc.

#### Rancher Prime - Observability UI extension

This is an UI extension to Rancher Manager that integrates the health signals observed by SUSE Observability. It gives direct access to the health of any resource and a link to SUSE Observability's UI for further investigation.

### Where to install SUSE Observability server

SUSE Observability server should be installed in its own downstream cluster intended for Observability. See the below picture for reference.

For StackState to be able to work properly it needs:
* [Kubernetes Persistent Storage](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/manage-clusters/create-kubernetes-persistent-storage) to be available in the observability cluster to store metrics, events, etc.
* the observability cluster to support a way to expose StackState on an HTTPS URL to Rancher, StackState users and the StackState agent. This can be done via an Ingress configuration using an ingress controller, alternatively a (cloud) loadbalancer for the StackState services could do this too, for more information see the [Rancher docs](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-resources-setup/load-balancer-and-ingress-controller).

![Architecture](/.gitbook/assets/k8s/prime/SUSEObservabilityDeployment.png)

### Pre-Installation
Before installing the SUSE Observability server a default storage class must be set up in the cluster where the SUSE Observability server will be installed:

- **For k3s**: The local-path storage class of type rancher.io/local-path is created by default.
- **For EKS, AKS, GKE** a storage class is set by default
- **For RKE2 Node Drivers**: No storage class is created by default. You will need to create one before installing SUSE Observability.

## Installing SUSE Observability

{% hint style="info" %}
**Good to know**

If you created the cluster using Rancher Manager and would like to run the provisioning commands below from a local terminal instead of in the web terminal, just copy or download the kubeconfig from the cluster dashboard, see image below, and paste it (or place the downloaded file) into a file that you can easily find e.g. ~/.kube/config-rancher and set the environment variable KUBECONFIG=$HOME/.kube/config-rancher
{% endhint %}

![Rancher](/.gitbook/assets/k8s/prime/rancher_cluster_dashboard.png)


After meeting the prerequisites you can proceed with the installation. The installation is NOT YET AVAILABLE from the app store. Instead, you can install SUSE Observability via the kubectl shell of the cluster.

You can now follow the instruction below for a HA or NON-HA setup.

{% hint style="info" %}
Be aware upgrading or downgrading from HA to NON-HA and visa-versa is not yet supported.
{% endhint %}


### Installation

1. Get the helm chart
   {% code title="helm_repo.sh" lineNumbers="true" %}
```text
helm repo add suse-observability https://charts.rancher.com/server-charts/prime/suse-observability
helm repo update
```
{% endcode %}

2. Command to generate helm chart values files:

{% code title="helm_template.sh" lineNumbers="true" %}
```text
export VALUES_DIR=.
helm template \
  --set license='<your license>' \
  --set baseUrl='<suse-observability-base-url>' \
  --set sizing.profile='<sizing.profile>' \
  suse-observability-values \
  suse-observability/suse-observability-values --output-dir $VALUES_DIR
```
{% endcode %}

The `baseUrl` must be the URL via which SUSE Observability will be accessible to Rancher, users, and the SUSE Observability agent. The URL must include the scheme, for example `https://observability.internal.mycompany.com`. See also [accessing SUSE Observability](#accessing-suse-observability).

The `sizing.profile` should be one of trial, 10-nonha, 20-nonha, 50-nonha, 100-nonha, 150-ha, 250-ha, 500-ha. Based on this profiles the `sizing_values.yaml` file is generated containing default sizes for the SUSE Observability resources and configuration to be deployed on an Ha or NonHa mode. E.g. 10-nonha will produce a `sizing_values.yaml` meant to deploy a NonHa SUSE Observability instance to observe a 10 node cluster in a Non High Available mode. Currently moving from a nonha to an ha environment is not possible, so if you expect that your environment willrequire to observe around 150 nodes then better to go with ha immediately.

This command will generate a `$VALUES_DIR/suse-observability-values/templates/baseConfig_values.yaml` and a `$VALUES_DIR/suse-observability-values/templates/sizing_values.yaml` file which contains the necessary configuration for installing the SUSE Observability Helm Chart.

{% hint style="info" %}
The SUSE Observability administrator passwords will be autogenerated by the above command and are output as comments in the generated `basicConfig.yaml` file. The actual values contain the `bcrypt` hashes of those passwords so that they're securely stored in the Helm release in the cluster.
{% endhint %}

{% hint style="info" %}
Store the generated `basicConfig.yaml` and `sizing_values.yaml` files somewhere safe. You can reuse this files for upgrades, which will save time and \(more importantly\) will ensure that SUSE Observability continues to use the same API key. This is desirable as it means Agents and other data providers for SUSE Observability won't need to be updated.
The files can be regenerated independently using the switches `basicConfig.generate=false` and `sizing.generate=false` to disable any of them while still keeping the previosuly generated version of the file in the `output-dir`.
{% endhint %}

3. Deploy the SUSE Observability helm chart with the generated values:

{% code title="helm_deploy.sh" lineNumbers="true" %}
```text
helm upgrade --install \
    --namespace suse-observability \
    --create-namespace \
    --values $VALUES_DIR/suse-observability-values/templates/baseConfig_values.yaml \
    --values $VALUES_DIR/suse-observability-values/templates/sizing_values.yaml \
    suse-observability \
    suse-observability/suse-observability
```
{% endcode %}

## Accessing SUSE Observability

The SUSE Observability Helm chart has support for creating an Ingress resource to make SUSE Observability accessible outside of the cluster. Follow [these instructions](setup/install-stackstate/kubernetes_openshift/ingress.md) to set that up when you have an ingress controller in the cluster. Make sure that the resulting URL uses TLS with a valid, not self-signed, certificate.

If you prefer to use a load balancer instead of ingress, expose the `suse-observability-router` service. The URL for the loadbalancer needs to use a valid, not self-signed, TLS certificate.

## Installing UI extensions

To install UI extensions, enable the UI extensions from the rancher UI

![Install](/.gitbook/assets/k8s/prime/ui_extensions.png)

After enabling UI extensions, follow these steps:

1. Navigate to extensions on the rancher UI and under the "Available" section of extensions, you will find the Observability extension.
2. Install the Observability extension.
3. Once installed, on the left panel of the rancher UI, the _SUSE Observability_ section appears.
4. Navigate to the _SUSE Observability_ section and select "configurations". In this section, you can add the SUSE Observability server details and connect it.
5. Follow the instructions as mentioned in *Obtain a service token* section below and fill in the details.

### Obtain a service token:
1. Log into the SUSE Observability instance.
1. From the top left corner, select CLI.
1. Note the API token and install SUSE Observability cli on your local machine.
1. Create a service token by running

{% code %}
```
sts service-token create --name suse-observability-extension --roles stackstate-k8s-troubleshooter
```
{% endcode %}


## Installing the SUSE Observability Agent

1. In the SUSE Observability UI open the main menu and select StackPacks.
2. Select the Kubernetes StackPack.
3. Click on new instance and provide the cluster name of the downstream cluster which you are adding. Make sure you match the name of the Rancher cluster with the name provided here. Click install.
4. In the list of instructions find the section that matches your cluster best
5. Execute the instructions provided to install the agent, these can be run in the `kubectl shell` that you can open for your cluster via the Rancher UI. But it can also be run from a local machine if it has Helm installed and is authorized to connect to the cluster.
6. After you install the agent, the cluster can be seen within the SUSE Observability UI as well as the _SUSE Rancher - Observability UI extension_.

## Single Sign On
To enable Single sign-on with your own authentication provider please [see here](setup/security/authentication/authentication_options.md).

## Frequently asked questions & Observations:
1. Is it mandatory to install a SUSE Observability agent before proceeding with adding the UI extension?
   * No this is not mandatory, the UI extension can be installed independent.
1. Is it mandatory to install SUSE Observability Server before we proceed with UI extensions?
   * Yes this is not mandatory since you need to provide a SUSE Observability endpoint in the configuration
1. Can we install SUSE Observability on a local cluster or on a downstream cluster?
   * Both options are possible.
1. To monitor the downstream clusters, should we install the SUSE Observability agent from the app store or add a new instance from the SUSE Observability UI?
   * Both options are possible depending on users preference.

## Open Issues
1. When you uninstall and reinstall the UI extensions for Observability, we noticed that service token is not deleted and is reused upon reinstallation. Whenever we uninstall the extensions, service token should be removed.
   * This information should be deleted when the UI extensions are uninstalled.
1. After the extensions are installed, the SUSE Observability UI opens in the same tab as the Rancher UI.
   * You can use shift-click to open in a new tab, this will become the default behaviour
1. Be aware upgrading or downgrading from HA to NON-HA and visa-versa is not yet supported.

