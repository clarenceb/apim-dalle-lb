# DALL-E 3 LB with APIM

Basic example of load-balancing multiple DALLE-3 models across different Azure regions to get around the current limits of 6 requests per minute per region, see: https://learn.microsoft.com/en-us/azure/ai-services/openai/quotas-limits#quotas-and-limits-reference

## Setup

Modify paramters in `params.json` to meet your needs.

In particular, number of Azure OpenAI instances to LB across:

```json
"openAIConfig": {
    "value": [
        {
            "name": "dalle3-1",
            "location": "australiaeast",
            "priority": 1,
            "weight": 50
        },
        {
            "name": "dalle3-2",
            "location": "swedencentral",
            "priority": 1,
            "weight": 50

        }
    ]
},
```

Add more regions as required and adjust weights (assuming equal weighting).  The Azure OpenAI instance names must be unique.

Adjust capacity per region (max is current 2 units) for DALL-E 3:

```json
"openAIModelCapacity": {
    "value": 2
},
```

Adjust circuit breaking policy for APIM (if required) here `policy.xml`.

## Deploy models and API Management

```sh
ARM_DEPLOYMENT_NAME=openai-dalle3
RG_NAME=openai-dalle3
RG_LOCATION=australiaeast

az bicep upgrade

az group create -n $RG_NAME -l $RG_LOCATION

az deployment group create --name $ARM_DEPLOYMENT_NAME --resource-group $RG_NAME --template-file "main.bicep" --parameters "params.json"

apim_subscription_key=$(az deployment group show --name $ARM_DEPLOYMENT_NAME -g $RG_NAME --query properties.outputs.apimSubscriptionKey.value -o tsv)

echo "API Subscription key: ${apim_subscription_key}"

apim_resource_gateway_url=$(az deployment group show --name $ARM_DEPLOYMENT_NAME -g $RG_NAME --query properties.outputs.apimResourceGatewayURL.value -o tsv)

echo "API Gateway URL: ${apim_resource_gateway_url}"
```

## Usage

```sh
curl -H "Content-Type: application/json" -H "Ocp-Apim-Subscription-Key: <your-sub-key>" -X POST https://<your-apim>.azure-api.net/openai/deployments/dall-e-3/images/generations?api-version=2024-05-01-preview -d '{"prompt":"a corgi in a field","n":1,"size":"1024x1024","response_format":"url","user":"user123456","quality":"standard","style":"vivid"}'
```

## Credits

Based on the work by Alexandre Vieira and others here: https://github.com/Azure-Samples/AI-Gateway
