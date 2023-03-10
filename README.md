# Deploy models from Azure Databricks to Azure ML in a secure way

In this repository we will see how to deploy real-time models connecting the Workspaces of Azure Databricks and Azure Machine Learning in a secure way.

As an example, we will use an already-trained Churn prediction model. The model was trained using the XGBoost algorithm and makes a binary prediction to predict Churn (YES/NO). The serialized model can be found in the [directory](churn-deploy/model/).

Below is a diagram demonstrating the deployment process:

![](/images/deployment-flow.jpg)

## Prerequisites

* You must have an Azure Databricks Workspace inside a private vnet and with No-Public IP enable. For more information about this configuration please look in this [doc](https://learn.microsoft.com/en-us/azure/databricks/security/secure-cluster-connectivity).

* You must have an Azure Machine Learning Workspace with no Public Access and the private endpoints configured for the Workspace and all the dependencies. For more information about this configuration please look in this [doc](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-secure-workspace-vnet?tabs=pe%2Ccli).

* A peering between the two vnets (AML and ADB). It is also important to ensure the Azure Databricks vnet has a vnet-link with the Private DNS Zones of AML Workspace and dependencies.

* A Service Principal with contribution permission on AML Workspace.

* An Azure Key Vault configured and linked to ADB Workspace. We also need the Service Principal credentials created as Secrets as well as the other sensitive informations (AML Workspace Name, Subscription Id, etc).

* If you want to use our example model please take it to your ADLS container.

* An ADB Cluster with 11.3 LTS Runtime (or greater) and ``azure-ai-ml`` library installed.

We provide the following architecture demonstrating some of these pre-reqs:

![](/images/architecture-adb-aml-prv.jpg)

## How to Deploy?

- Ensure your ADB cluster has the environment variables ``AZURE_CLIENT_SECRET``, ``AZURE_TENANT_ID`` and ``AZURE_CLIENT_ID`` set with the Service Principal credentials. It is important to keep them safe, so ensure you have them registered in a Key Vault. The image below show how you can do this in Environment variables section of the cluster:

![](/images/environment-variables.jpg)


With the configured cluster you can create a notebook to deploy the model. You can check the full notebook example [here](notebooks/deploy-model-to-aml.ipynb). Following we show the required steps:

- Initiate the `ml_client` to interact with AML Workspace resources:

```python

from azure.ai.ml import MLClient
from azure.identity import DefaultAzureCredential

subscription_id = dbutils.secrets.get(scope = 'akd-adb-prv', key = 'SUBSCRIPTION-ID')
resource_group = dbutils.secrets.get(scope = 'akd-adb-prv', key = 'RESOURCE-GROUP')
workspace = dbutils.secrets.get(scope = 'akd-adb-prv', key = 'AML-WORKSPACE-NAME')

ml_client = MLClient(
    DefaultAzureCredential(), subscription_id, resource_group, workspace
)

```

- Now we need to define the Managed Online Endpoint endpoint's configs. 

We set the ``publick_network_access`` as ``disabled`` to ensure that the inbound (scoring) access will use the private endpoint of the [Azure ML Workspace](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-configure-private-link).

```python
from azure.ai.ml.entities import (ManagedOnlineEndpoint, 
                                 ManagedOnlineDeployment, 
                                 Model, 
                                 CodeConfiguration, 
                                 Environment)

endpoint_name = 'churn-endpoint-01' # YOUR ENDPOINT NAME

endpoint = ManagedOnlineEndpoint(name=endpoint_name,  
                         description='Realtime endpoint to predict churn', 
                         tags={'model': 'XGBoost'}, 
                         auth_mode="key", 
                         public_network_access="disabled" 
)

ml_client.online_endpoints.begin_create_or_update(endpoint)
```

We need to wait to the provisioning_state to be **Succeeded**. In the [notebook](notebooks/deploy-model-to-aml.ipynb) we have an example how to to this.

- Create a Model on Azure Machine Learning with all models' artifacts

It is important to have your model in the `path`. In our example we put the artifact in an ADLS storage and copy them to dbfs. Feel free to customize it according to your requirements.

```python
from azure.ai.ml.entities import Model
from azure.ai.ml.constants import AssetTypes

cloud_model = Model(
    path='/dbfs/tmp/model/model',
    name="churn-model-adb",
    type=AssetTypes.CUSTOM_MODEL,
    description="Model created from cloud path.",
)

model = ml_client.models.create_or_update(cloud_model)
```

- The last stage is to deploy the model according to the required configs.

Here we need to define the `instance_type` and how many nodes we would like to use (`instance_count`).

We set `egress_public_network_access` to disabled as well to ensure that the communication between a deployment and external resources, including Azure resources, will use the private endpoint.

```python
deployment_name = 'blue'

env = Environment(
    conda_file="/dbfs/tmp/model/conda.yml",
    image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04:latest",
)

deployment = ManagedOnlineDeployment(name=deployment_name, 
                                          endpoint_name=endpoint_name, 
                                          model=model, 
                                          code_configuration=CodeConfiguration(code='/dbfs/tmp/onlinescoring/',
                                                                               scoring_script='score.py'),
                                          environment=env, 
                                          instance_type='Standard_DS2_v2', 
                                          instance_count=1, 
                                          egress_public_network_access="disabled"
)

ml_client.online_deployments.begin_create_or_update(deployment=deployment)
```

After the deployment suceeded we can see the endpoint in the AML Worspace. All the requests will be done in a private context. So, ensure your application has the right network configs.

![](/images/endpoint-aml.jpg)
