# ADO Agents Container Apps

Version 0.0.1

Date 15/03/23

| Title                     | Document Author    | Product Manager  | Iteration                                                                                                      |
| ------------------------- | ------------------ | ---------------- | -------------------------------------------------------------------------------------------------------------- |
| ADO Agents Container Apps | Arnas Klimasauskas | James Waterfield | [Azure 2.2](https://allenovery.visualstudio.com/Platform%20Team/_wiki/wikis/Platform-Team.wiki/7280/Azure-2.2) |

## Introduction

Currently, whenever Platform team members run a pipeline, ADO Agent(s) are assigned from an Agent pool defined inside the pipeline definition to complete the jobs. The Agent replicas inside Agent pools are deployed in Azure Container Instances. While the current configuration is reliable and stable, it can not keep up with heavier workloads (large amounts of jobs waiting in queue) since Azure Container Instances do not support Agent autoscaling.


## Design considerations and high-level recommendations

Following requirements were considered:
- ADO authentication using PAT (Personal Access token)
- Ability to auto-scale containerized agent pools based on queue 
- KEDA integration with Container Applications

## Design

In order to tackle performance issues during heavier workloads and make a refinement to current configuration, it was considered to explore Azure Container Applications _([Explore Azure container apps option which should be Azure Container instances v2 for easy scalability](https://allenovery.visualstudio.com/Platform%20Team/_workitems/edit/139968))_.

## Azure Container Application and use case

When talking about replacing Container Instances (CI) with Container Applications (CA), one might ask - how is CA better than CI? To shortly answer that question, CA enables us to build serverless microservices based on containers while also supporting technologies like Dapr or KEDA (Kubernetes Event Driven Autoscaler). 
In our case, using CA is beneficial, because KEDA allows us to scale ADO Agent replicas (e.g. increase Agent replica count when 1 or more jobs are in queue inside the Agent Pool) without losing any of the functionalities from CI. Besides that, CA could potentially be more cost effective (would need more testing), because there wouldn't be a fixed amount of online agents, but only a specified amount (1 or more) that would scale depending on given workloads.

## Container Application deployment prerequisites

In order to create a Container Application, it is required to have a _[Container App environment](https://learn.microsoft.com/en-us/azure/container-apps/environment)_ and _[Log Analytics Workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-workspace-overview)_ resources. A CA environment is meant to deploy multiple CAs inside the same environment while also being in the same virtual network and write logs to the same _Log Analytics workspace_.

To deploy a CA which authenticates to ADO and creates Agent replicas inside an Agent Pool, we need:

- ADO Agent Pool name, id;
- ADO organization url;
- ADO PAT (Personal Access Token);
- Azure Container Registry with an ADO Agent image;
- A UAMI with permissions of:
    - Microsoft.ManagedIdentity/userAssignedIdentities/assign/action;
    - Microsoft.App/containerApps/write;
    - Microsoft.ContainerRegistry/registries/pull/read.

## Networking

As mentioned above, a CA requires a Managed Environment (ME) resource. When deploying a ME resource, there is an option to choose your own networking solution - a subnet from a Virtual Network resource ([reference](https://learn.microsoft.com/en-us/azure/container-apps/networking#custom-vnet-configuration)). The assigned network use case regarding the project would be useful, because we could assign a subnet which would allow communication between azure resources inside the internal network (this configuration is already being used in CIs). 

For testing purposes, an ME was created and a Virtual Network with its' subnet was assigned successfully in a _dev_ environment.

## Container Application scaling

When we want to enable CA autoscaling using _[KEDA](https://keda.sh/)_, we define **scale**
object inside the CA resource module ([Full resource](https://allenovery.visualstudio.com/Platform%20Team/_git/Platform?path=/tooling/azure-pipelines-agent/deploy-ca.bicep&version=GB139968-create-container-app&line=169&lineEnd=170&lineStartColumn=1&lineEndColumn=1&lineStyle=plain&_a=contents)):

```
scale: {
  maxReplicas: scaleMaxReplicas
  minReplicas: scaleMinReplicas
    rules: [
      {
        name: 'ado-agent'
        custom: {
          type: 'azure-pipelines'
          metadata: {
            poolName: AZP_POOL
            poolID: AZP_POOLID
            targetPipelinesQueueLength: '1'
          }
          auth: [
            {
              secretRef: 'azp-token'
              triggerParameter: 'personalAccessToken'
            }
            {
              secretRef: 'azp-url'
              triggerParameter: 'organizationURL'
            }
          ]
        }
      }
   ]
}
```

Here, we need to define the minimum and maximum amount of ADO Agent replicas. During no workload, the minimum amount of Agent replicas will be online. Inside the rule object _ado-agent_ we are using a custom Scale rule _[azure-pipelines](https://keda.sh/docs/2.10/scalers/azure-pipelines/)_ from KEDA. Required parameters for succesful Agent replica creation:

- poolName - ADO Agent pool name
- poolID - ADO Agent pool id
- targetPipelinesQueueLength - how many jobs to be in queue to activate scaling
- auth:
    - secretRef - reference to a CA secret
    - organizationURL - The URL of the ADO organization.
    - personalAccessToken - The PAT for ADO

## Performance

Let's say we have deployed a CA with a minimum of 1 Agent replica and the maximum amount is 10 replicas. 
Then, if we start a pipeline with 30 jobs where each one of them takes a minute to finish - using CA autoscaling, the amount of time it would take to finish would be around 5 minutes (3 minutes for the jobs and around 2 minutes for the additional Agent replicas to be deployed).
On the other hand, if for example we use a _PlatformUbuntu_ CI Agent pool (at the moment of testing, it consisted of 4 Agent replicas) to complete the same 30 jobs, it would take around 8 minutes and that's to say all of the online Agent replicas are currently on _idle_ status.

One thing to notice is that CA supports a maximum amount of 30 Agent replicas per revision (Maximum amount of revisions is 100). With that in mind, it's safe to say that the job completion time could be even faster.

## Consensus

To summarize, after creating a Container Application resource module, deploying it to Azure with parameters and running pipelines to test out how it handles big amounts of jobs - it is safe to say that Container Applications is the way to go forward. It does not have any takeaways from the previous configuration of using Container Instances to deploy ADO Agents and moreover, it consists of benefits such as: scalability, cost effectiveness and ease of deployment. 

## Resources

- https://github.com/Azure/ResourceModules/tree/main/modules/Microsoft.App/containerApps
- https://learn.microsoft.com/en-us/azure/container-apps/environment
- https://learn.microsoft.com/en-us/azure/container-apps/networking#custom-vnet-configuration
- https://keda.sh/
- https://keda.sh/docs/2.10/scalers/azure-pipelines/
- https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-workspace-overview
