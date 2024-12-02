# LLM Configuration

In AutoGen, agents use LLMs as key components to understand and react. To configure an agent's access to LLMs, you can specify an `llm_config` argument in its constructor. For example, the following snippet shows a configuration that uses `gpt-4`:


```python
import os

llm_config = {
    "config_list": [{"model": "gpt-4", "api_key": os.environ["OPENAI_API_KEY"]}],
}
```

````{=mdx}
:::warning
It is important to never commit secrets into your code, therefore we read the OpenAI API key from an environment variable.
:::
````

This `llm_config` can then be passed to an agent's constructor to enable it to use the LLM.


```python
import autogen

assistant = autogen.AssistantAgent(name="assistant", llm_config=llm_config)
```


## Introduction to `config_list`

Different tasks may require different models, and the `config_list` allows specifying the different endpoints and configurations that are to be used. It is a list of dictionaries, each of which contains the following keys depending on the kind of endpoint being used:

````{=mdx}
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs>
  <TabItem value="openai" label="OpenAI" default>
    - `model` (str, required): The identifier of the model to be used, such as 'gpt-4', 'gpt-3.5-turbo'.
    - `api_key` (str, optional): The API key required for authenticating requests to the model's API endpoint.
    - `base_url` (str, optional): The base URL of the API endpoint. This is the root address where API calls are directed.
    - `tags` (List[str], optional): Tags which can be used for filtering.

    Example:
    ```json
    [
      {
        "model": "gpt-4",
        "api_key": os.environ['OPENAI_API_KEY']
      }
    ]
    ```
  </TabItem>
  <TabItem value="azureopenai" label="Azure OpenAI">
    - `model` (str, required): The deployment to be used. The model corresponds to the deployment name on Azure OpenAI.
    - `api_key` (str, optional): The API key required for authenticating requests to the model's API endpoint.
    - `api_type`: `azure`
    - `base_url` (str, optional): The base URL of the API endpoint. This is the root address where API calls are directed.
    - `api_version` (str, optional): The version of the Azure API you wish to use.
    - `tags` (List[str], optional): Tags which can be used for filtering.

    Example:
    ```json
    [
      {
        "model": "my-gpt-4-deployment",
        "api_type": "azure",
        "api_key": os.environ['AZURE_OPENAI_API_KEY'],
        "base_url": "https://ENDPOINT.openai.azure.com/",
        "api_version": "2024-02-01"
      }
    ]
    ```
  </TabItem>
  <TabItem value="other" label="Other OpenAI compatible">
    - `model` (str, required): The identifier of the model to be used, such as 'llama-7B'.
    - `api_key` (str, optional): The API key required for authenticating requests to the model's API endpoint.
    - `base_url` (str, optional): The base URL of the API endpoint. This is the root address where API calls are directed.
    - `tags` (List[str], optional): Tags which can be used for filtering.

    Example:
    ```json
    [
      {
        "model": "llama-7B",
        "base_url": "http://localhost:1234"
      }
    ]
    ```
  </TabItem>
</Tabs>
````

---

````{=mdx}
:::tip
By default this will create a model client which assumes an OpenAI API (or compatible) endpoint. To use custom model clients, see [here](https://github.com/microsoft/autogen/blob/main/notebook/agentchat_custom_model.ipynb).
:::
````

### `OAI_CONFIG_LIST` pattern

A common, useful pattern used is to define this `config_list` via JSON (specified as a file or an environment variable set to a JSON-formatted string) and then use the [`config_list_from_json`](/docs/reference/oai/openai_utils#config_list_from_json) helper function to load it:


```python
config_list = autogen.config_list_from_json(
    env_or_file="OAI_CONFIG_LIST",
)

# Then, create the assistant agent with the config list
assistant = autogen.AssistantAgent(name="assistant", llm_config={"config_list": config_list})
```

This can be helpful as it keeps all the configuration in one place across different projects or notebooks.

This function interprets the `env_or_file` argument as follows:

- If `env_or_file` is an environment variable then:
    - It will first try to load the file from the path specified in the environment variable.
    - If there is no file, it will try to interpret the environment variable as a JSON string.
- Otherwise, it will try to open the file at the path specified by `env_or_file`.

### Why is it a list?

Being a list allows you to define multiple models that can be used by the agent. This is useful for a few reasons:

- If one model times out or fails, the agent can try another model.
- Having a single global list of models and [filtering it](#config-list-filtering) based on certain keys (e.g. name, tag) in order to pass select models into a certain agent (e.g. use cheaper GPT 3.5 for agents solving easier tasks)
- While the core agents, (e.g. conversable or assistant) do not have special logic around selecting configs, some of the specialized agents *may* have logic to select the best model based on the task at hand.

### How does an agent decide which model to pick out of the list?

An agent uses the very first model available in the "config_list" and makes LLM calls against this model. If the model fails (e.g. API throttling) the agent will retry the request against the 2nd model and so on until prompt completion is received (or throws an error if none of the models successfully completes the request). In general there's no implicit/hidden logic inside agents that is used to pick "the best model for the task". However, some specialized agents may attempt to choose "the best model for the task". It is developers responsibility to pick the right models and use them with agents.

### Config list filtering

As described above the list can be filtered based on certain criteria. This is defined as a dictionary of key to filter on and values to filter by. For example, if you have a list of configs and you want to select the one with the model "gpt-3.5-turbo" you can use the following filter:


```python
filter_dict = {"model": ["gpt-3.5-turbo"]}
```


This can then be applied to a config list loaded in Python with [`filter_config`](/docs/reference/oai/openai_utils#filter_config):


```python
config_list = autogen.filter_config(config_list, filter_dict)
```

Or, directly when loading the config list using [`config_list_from_json`](/docs/reference/oai/openai_utils#config_list_from_json):


```python
config_list = autogen.config_list_from_json(env_or_file="OAI_CONFIG_LIST", filter_dict=filter_dict)
```

#### Tags

Model names can differ between OpenAI and Azure OpenAI, so tags offer an easy way to smooth over this inconsistency. Tags are a list of strings in the `config_list`, for example for the following `config_list`:


```python
config_list = [
    {"model": "my-gpt-4-deployment", "api_key": "", "tags": ["gpt4", "openai"]},
    {"model": "llama-7B", "base_url": "http://127.0.0.1:8080", "tags": ["llama", "local"]},
]
```

Then when filtering the `config_list` you can can specify the desired tags. A config is selected if it has at least one of the tags specified in the filter. For example, to just get the `llama` model, you can use the following filter:


```python
filter_dict = {"tags": ["llama", "another_tag"]}
config_list = autogen.filter_config(config_list, filter_dict)
assert len(config_list) == 1
```

### Adding http client in llm_config for proxy

In Autogen, a deepcopy is used on llm_config to ensure that the llm_config passed by user is not modified internally. You may get an error if the llm_config contains objects of a class that do not support deepcopy. To fix this, you need to implement a `__deepcopy__` method for the class.

The below example shows how to implement a `__deepcopy__` method for http client and add a  proxy.


```python
#!pip install httpx
import httpx


class MyHttpClient(httpx.Client):
    def __deepcopy__(self, memo):
        return self


config_list = [
    {
        "model": "my-gpt-4-deployment",
        "api_key": "",
        "http_client": MyHttpClient(proxy="http://localhost:8030"),
    }
]

llm_config = {
    "config_list": config_list,
}
```

### Using Azure Active Directory (AAD) Authentication

Azure Active Directory (AAD) provides secure access to resources and applications. Follow the steps below to configure AAD authentication for Autogen.

#### Prerequisites
- An Azure subscription - [Create one for free](https://azure.microsoft.com/en-us/free/).
- Access granted to the Azure OpenAI Service in the desired Azure subscription.
- Appropriate permissions to register an application in AAD.
- Custom subdomain names are required to enable features like Microsoft Entra ID for authentication.
- Azure CLI - [Installation Guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

For more detailed and up-to-date instructions, please refer to the official [Azure OpenAI documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/).

#### Step 1: Register an Application in AAD
1. Navigate to the [Azure portal](https://azure.microsoft.com/en-us/get-started/azure-portal).
2. Go to `Azure Active Directory` > `App registrations`.
3. Click on `New registration`.
4. Enter a name for your application.
5. Set the `Redirect URI` (optional).
6. Click `Register`.

For detailed instructions, refer to the official [Azure AD Quickstart documentation](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app?tabs=certificate).

#### Step 2: Configure API Permissions
1. After registration, go to `API permissions`.
2. Click `Add a permission`.
3. Select `Microsoft Graph` and then `Delegated permissions`.
4. Add the necessary permissions (e.g., `User.Read`).

For more details, see [API permissions in Microsoft Graph](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)

#### Step 3: Obtain Client ID and Tenant ID
1. Go to `Overview` of your registered application.
2. Note down the `Application (client) ID` and `Directory (tenant) ID`.

For more details, visit [Register an application with the Microsoft identity platform](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app?tabs=certificate)

#### Step 4: Configure Your Application
Use the obtained `Client ID` and `Tenant ID` in your application configuration. Here’s an example of how to do this in your configuration file:
```
aad_config = {
    "client_id": "YOUR_CLIENT_ID",
    "tenant_id": "YOUR_TENANT_ID",
    "authority": "https://login.microsoftonline.com/YOUR_TENANT_ID",
    "scope": ["https://graph.microsoft.com/.default"],
}
```
#### Step 5: Authenticate and Acquire Tokens
Use the following code to authenticate and acquire tokens:

```
from msal import ConfidentialClientApplication

app = ConfidentialClientApplication(
    client_id=aad_config["client_id"],
    client_credential="YOUR_CLIENT_SECRET",
    authority=aad_config["authority"]
)

result = app.acquire_token_for_client(scopes=aad_config["scope"])

if "access_token" in result:
    print("Token acquired")
else:
    print("Error acquiring token:", result.get("error"))
```

For more details, refer to the [Authenticate and authorize in Azure OpenAI Service](https://learn.microsoft.com/en-us/azure/api-management/api-management-authenticate-authorize-azure-openai) and [How to configure Azure OpenAI Service with Microsoft Entra ID authentication](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/managed-identity).


#### Step 6: Configure Azure OpenAI with AAD Auth in AutoGen
To use AAD authentication with Azure OpenAI in AutoGen, configure the `llm_config` with the necessary parameters.

Here is an example configuration:

```
llm_config = {
    "config_list": [
        {
            "model": "gpt-4",
            "base_url": "YOUR_BASE_URL",
            "api_type": "azure",
            "api_version": "2024-02-01",
            "max_tokens": 1000,
            "azure_ad_token_provider": "DEFAULT"
        }
    ]
}
```

For more details, refer to the [Authenticate and authorize in Azure OpenAI Service](https://learn.microsoft.com/en-us/azure/api-management/api-management-authenticate-authorize-azure-openai) and [How to configure Azure OpenAI Service with Microsoft Entra ID authentication](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/managed-identity).

In this configuration:
- `model`: The Azure OpenAI deployment name.
- `base_url`: The base URL of the Azure OpenAI endpoint.
- `api_type`: Should be set to "azure".
- `api_version`: The API version to use.
- `azure_ad_token_provider`: Set to "DEFAULT" to use the default token provider.

#### Example of Initializing an Assistant Agent with AAD Auth
```
import autogen

# Initialize the assistant agent with the AAD authenticated config
assistant = autogen.AssistantAgent(name="assistant", llm_config=llm_config)
```

#### Troubleshooting
If you encounter issues, check the following:
- Ensure your `Client ID` and `Tenant ID` are correct.
- Verify the permissions granted to your application.
- Check network connectivity and Azure service status.

This documentation provides a complete guide to configure and use AAD authentication with Azure OpenAI in the AutoGen.


## Other configuration parameters

Besides the `config_list`, there are other parameters that can be used to configure the LLM. These are split between parameters specifically used by Autogen and those passed into the model client.

### AutoGen specific parameters

- `cache_seed` - This is a legacy parameter and not recommended to be used unless the reason for using it is to disable the default caching behavior. To disable default caching, set this to `None`. Otherwise, by default or if an int is passed the [DiskCache](/docs/reference/cache/disk_cache) will be used. For the new way of using caching, pass a [Cache](/docs/reference/cache/) object into [`initiate_chat`](/docs/reference/agentchat/conversable_agent#initiate_chat).

### Extra model client parameters

It is also possible to passthrough parameters through to the OpenAI client. Parameters that correspond to the [`OpenAI` client](https://github.com/openai/openai-python/blob/d231d1fa783967c1d3a1db3ba1b52647fff148ac/src/openai/_client.py#L67) or the [`OpenAI` completions create API](https://github.com/openai/openai-python/blob/d231d1fa783967c1d3a1db3ba1b52647fff148ac/src/openai/resources/completions.py#L35) can be supplied.

This is commonly used for things like `temperature`, or `timeout`.

## Example




```python
llm_config = {
    "config_list": [
        {
            "model": "my-gpt-4-deployment",
            "api_key": os.environ.get("AZURE_OPENAI_API_KEY"),
            "api_type": "azure",
            "base_url": os.environ.get("AZURE_OPENAI_API_BASE"),
            "api_version": "2024-02-01",
        },
        {
            "model": "llama-7B",
            "base_url": "http://127.0.0.1:8080",
            "api_type": "openai",
        },
    ],
    "temperature": 0.9,
    "timeout": 300,
}
```

## Other helpers for loading a config list

- [`get_config_list`](/docs/reference/oai/openai_utils#get_config_list): Generates configurations for API calls, primarily from provided API keys.
- [`config_list_openai_aoai`](/docs/reference/oai/openai_utils#config_list_openai_aoai): Constructs a list of configurations using both Azure OpenAI and OpenAI endpoints, sourcing API keys from environment variables or local files.
- [`config_list_from_models`](/docs/reference/oai/openai_utils#config_list_from_models): Creates configurations based on a provided list of models, useful when targeting specific models without manually specifying each configuration.
- [`config_list_from_dotenv`](/docs/reference/oai/openai_utils#config_list_from_dotenv): Constructs a configuration list from a `.env` file, offering a consolidated way to manage multiple API configurations and keys from a single file.

See [this notebook](https://github.com/microsoft/autogen/blob/main/notebook/config_loader_utility_functions.ipynb) for examples of using the above functions.