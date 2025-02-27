---
title: Configure Container insights agent data collection | Microsoft Docs
description: This article describes how you can configure the Container insights agent to control stdout/stderr and environment variables log collection.
ms.topic: conceptual
ms.date: 08/25/2022
ms.reviewer: aul
---

# Configure agent data collection for Container insights

Container insights collects stdout, stderr, and environmental variables from container workloads deployed to managed Kubernetes clusters from the containerized agent. You can configure agent data collection settings by creating a custom Kubernetes ConfigMaps to control this experience. 

This article demonstrates how to create ConfigMap and configure data collection based on your requirements.

## ConfigMap file settings overview

A template ConfigMap file is provided that allows you to easily edit it with your customizations without having to create it from scratch. Before starting, you should review the Kubernetes documentation about [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) and familiarize yourself with how to create, configure, and deploy ConfigMaps. This will allow you to filter stderr and stdout per namespace or across the entire cluster, and environment variables for any container running across all pods/nodes in the cluster.

>[!IMPORTANT]
>The minimum agent version supported to collect stdout, stderr, and environmental variables from container workloads is ciprod06142019 or later. To verify your agent version, from the **Node** tab select a node, and in the properties pane note value of the **Agent Image Tag** property. For additional information about the agent versions and what's included in each release, see [agent release notes](https://github.com/microsoft/Docker-Provider/tree/ci_feature_prod).

### Data collection settings

The following table describes the settings you can configure to control data collection:

| Key | Data type | Value | Description |
|--|--|--|--|
| `schema-version` | String (case sensitive) | v1 | This is the schema version used by the agent<br> when parsing this ConfigMap.<br> Currently supported schema-version is v1.<br> Modifying this value is not supported and will be<br> rejected when ConfigMap is evaluated. |
| `config-version` | String |  | Supports ability to keep track of this config file's version in your source control system/repository.<br> Maximum allowed characters are 10, and all other characters are truncated. |
| `[log_collection_settings.stdout] enabled =` | Boolean | true or false | This controls if stdout container log collection is enabled. When set to `true` and no namespaces are excluded for stdout log collection<br> (`log_collection_settings.stdout.exclude_namespaces` setting below), stdout logs will be collected from all containers across all pods/nodes in the cluster. If not specified in ConfigMaps,<br> the default value is `enabled = true`. |
| `[log_collection_settings.stdout] exclude_namespaces =` | String | Comma-separated array | Array of Kubernetes namespaces for which stdout logs will not be collected. This setting is effective only if<br> `log_collection_settings.stdout.enabled`<br> is set to `true`.<br> If not specified in ConfigMap, the default value is<br> `exclude_namespaces = ["kube-system"]`. |
| `[log_collection_settings.stderr] enabled =` | Boolean | true or false | This controls if stderr container log collection is enabled.<br> When set to `true` and no namespaces are excluded for stdout log collection<br> (`log_collection_settings.stderr.exclude_namespaces` setting), stderr logs will be collected from all containers across all pods/nodes in the cluster.<br> If not specified in ConfigMaps, the default value is<br> `enabled = true`. |
| `[log_collection_settings.stderr] exclude_namespaces =` | String | Comma-separated array | Array of Kubernetes namespaces for which stderr logs will not be collected.<br> This setting is effective only if<br> `log_collection_settings.stdout.enabled` is set to `true`.<br> If not specified in ConfigMap, the default value is<br> `exclude_namespaces = ["kube-system"]`. |
| `[log_collection_settings.env_var] enabled =` | Boolean | true or false | This setting controls environment variable collection<br> across all pods/nodes in the cluster<br> and defaults to `enabled = true` when not specified<br> in ConfigMaps.<br> If collection of environment variables is globally enabled, you can disable it for a specific container<br> by setting the environment variable<br> `AZMON_COLLECT_ENV` to **False** either with a Dockerfile setting or in the [configuration file for the Pod](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/) under the **env:** section.<br> If collection of environment variables is globally disabled, then you cannot enable collection for a specific container (that is, the only override that can be applied at the container level is to disable collection when it's already enabled globally.). It’s strongly recommended to secure log analytics workspace access with the default [log_collection_settings.env_var] enabled = true. If sensitive data is stored in environment variables, it is mandatory and very critical to secure log analytics workspace. |
| `[log_collection_settings.enrich_container_logs] enabled =` | Boolean | true or false | This setting controls container log enrichment to populate the Name and Image property values<br> for every log record written to the ContainerLog table for all container logs in the cluster.<br> It defaults to `enabled = false` when not specified in ConfigMap. |
| `[log_collection_settings.collect_all_kube_events] enabled =` | Boolean | true or false | This setting allows the collection of Kube events of all types.<br> By default the Kube events with type *Normal* are not collected. When this setting is set to `true`, the *Normal* events are no longer filtered and all events are collected.<br> It defaults to `enabled = false` when not specified in the ConfigMap |

### Metric collection settings

The following table describes the settings you can configure to control metric collection:

| Key | Data type | Value | Description |
|--|--|--|--|
| `[metric_collection_settings.collect_kube_system_pv_metrics] enabled =` | Boolean | true or false | This setting allows persistent volume (PV) usage metrics to be collected in the kube-system namespace. By default, usage metrics for persistent volumes with persistent volume claims in the kube-system namespace are not collected. When this setting is set to `true`, PV usage metrics for all namespaces are collected. By default, this is set to `false`. |

ConfigMaps is a global list and there can be only one ConfigMap applied to the agent. You cannot have another ConfigMaps overruling the collections.

## Configure and deploy ConfigMaps

Perform the following steps to configure and deploy your ConfigMap configuration file to your cluster.

1. Download the [template ConfigMap YAML file](https://aka.ms/container-azm-ms-agentconfig) and save it as container-azm-ms-agentconfig.yaml. 

2. Edit the ConfigMap yaml file with your customizations to collect stdout, stderr, and/or environmental variables.

    - To exclude specific namespaces for stdout log collection, you configure the key/value using the following example: `[log_collection_settings.stdout] enabled = true exclude_namespaces = ["my-namespace-1", "my-namespace-2"]`.
    
    - To disable environment variable collection for a specific container, set the key/value `[log_collection_settings.env_var] enabled = true` to enable variable collection globally, and then follow the steps [here](container-insights-manage-agent.md#how-to-disable-environment-variable-collection-on-a-container) to complete configuration for the specific container.
    
    - To disable stderr log collection cluster-wide, you configure the key/value using the following example: `[log_collection_settings.stderr] enabled = false`.
    
    Save your changes in the editor.

3. Create ConfigMap by running the following kubectl command: `kubectl apply -f <configmap_yaml_file.yaml>`. 
    
    Example: `kubectl apply -f container-azm-ms-agentconfig.yaml`. 

The configuration change can take a few minutes to finish before taking effect, and all omsagent pods in the cluster will restart. The restart is a rolling restart for all omsagent pods, not all restart at the same time. When the restarts are finished, a message is displayed that's similar to the following and includes the result: `configmap "container-azm-ms-agentconfig" created`.

## Verify configuration

To verify the configuration was successfully applied to a cluster, use the following command to review the logs from an agent pod: `kubectl logs omsagent-fdf58 -n kube-system`. If there are configuration errors from the omsagent pods, the output will show errors similar to the following:

``` 
***************Start Config Processing******************** 
config::unsupported/missing config schema version - 'v21' , using defaults
```

Errors related to applying configuration changes are also available for review. The following options are available to perform additional troubleshooting of configuration changes:

- From an agent pod logs using the same `kubectl logs` command. 

- From Live logs. Live logs show errors similar to the following:

    ```
    config::error::Exception while parsing config map for log collection/env variable settings: \nparse error on value \"$\" ($end), using defaults, please check config map for errors
    ```

- From the **KubeMonAgentEvents** table in your Log Analytics workspace. Data is sent every hour with *Error* severity for configuration errors. If there are no errors, the entry in the table will have data with severity *Info*, which reports no errors. The **Tags** property contains more information about the pod and container ID on which the error occurred and also the first occurrence, last occurrence and count in the last hour.

After you correct the error(s) in ConfigMap, save the yaml file and apply the updated ConfigMaps by running the command: `kubectl apply -f <configmap_yaml_file.yaml`.

## Applying updated ConfigMap

If you have already deployed a ConfigMap on clusters and you want to update it with a newer configuration, you can edit the ConfigMap file you've previously used and then apply using the same command as before, `kubectl apply -f <configmap_yaml_file.yaml`.

The configuration change can take a few minutes to finish before taking effect, and all omsagent pods in the cluster will restart. The restart is a rolling restart for all omsagent pods, not all restart at the same time. When the restarts are finished, a message is displayed that's similar to the following and includes the result: `configmap "container-azm-ms-agentconfig" updated`.

## Verifying schema version

Supported config schema versions are available as pod annotation (schema-versions) on the omsagent pod. You can see them with the following kubectl command: `kubectl describe pod omsagent-fdf58 -n=kube-system`

The output will show similar to the following with the annotation schema-versions:

```
    Name:           omsagent-fdf58
    Namespace:      kube-system
    Node:           aks-agentpool-95673144-0/10.240.0.4
    Start Time:     Mon, 10 Jun 2019 15:01:03 -0700
    Labels:         controller-revision-hash=589cc7785d
                    dsName=omsagent-ds
                    pod-template-generation=1
    Annotations:    agentVersion=1.10.0.1
                  dockerProviderVersion=5.0.0-0
                    schema-versions=v1 
```

## Next steps

- Container insights does not include a predefined set of alerts. Review the [Create performance alerts with Container insights](./container-insights-log-alerts.md) to learn how to create recommended alerts for high CPU and memory utilization to support your DevOps or operational processes and procedures.

- With monitoring enabled to collect health and resource utilization of your AKS or hybrid cluster and workloads running on them, learn [how to use](container-insights-analyze.md) Container insights.

- View [log query examples](container-insights-log-query.md) to see pre-defined queries and examples to evaluate or customize for alerting, visualizing, or analyzing your clusters.
