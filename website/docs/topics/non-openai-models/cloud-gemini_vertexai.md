# Use AutoGen with Gemini via VertexAI

This notebook demonstrates how to use Autogen with Gemini via Vertex AI, which enables enhanced authentication method that also supports enterprise requirements using service accounts or even a personal Google cloud account.

## Requirements

Install AutoGen with Gemini features:
```bash
pip install pyautogen[gemini]
```

### Install other Dependencies of this Notebook
```bash
pip install chromadb markdownify pypdf
```

### Google Cloud Account
To use VertexAI a Google Cloud account is needed. If you do not have one yet, just sign up for a free trial [here](https://cloud.google.com).

Login to your account at [console.cloud.google.com](https://console.cloud.google.com)

In the next step we create a Google Cloud project, which is needed for VertexAI. The official guide for creating a project is available is [here](https://developers.google.com/workspace/guides/create-project). 

We will name our project Autogen-with-Gemini.

### Enable Google Cloud APIs

If you wish to use Gemini with your personal account, then creating a Google Cloud account is enough. However, if a service account is needed, then a few extra steps are needed.

#### Enable API for Gemini
 * For enabling Gemini for Google Cloud search for "api" and select Enabled APIs & services. 
 * Then click ENABLE APIS AND SERVICES. 
 * Search for Gemini, and select Gemini for Google Cloud. <br/> A direct link will look like this for our autogen-with-gemini project:
https://console.cloud.google.com/apis/library/cloudaicompanion.googleapis.com?project=autogen-with-gemini&supportedpurview=project
* Click ENABLE for Gemini for Google Cloud.

### Enable API for Vertex AI
* For enabling Vertex AI for Google Cloud search for "api" and select Enabled APIs & services. 
* Then click ENABLE APIS AND SERVICES. 
* Search for Vertex AI, and select Vertex AI API. <br/> A direct link for our autogen-with-gemini will be: https://console.cloud.google.com/apis/library/aiplatform.googleapis.com?project=autogen-with-gemini
* Click ENABLE Vertex AI API for Google Cloud.

### Create a Service Account

You can find an overview of service accounts [can be found in the cloud console](https://console.cloud.google.com/iam-admin/serviceaccounts)

Detailed guide: https://cloud.google.com/iam/docs/service-accounts-create

A service account can be created within the scope of a project, so a project needs to be selected.

<div>
<img src="https://github.com/microsoft/autogen/blob/main/website/static/img/create_gcp_svc.png?raw=true" width="1000" />
</div>

Next we assign the [Vertex AI User](https://cloud.google.com/vertex-ai/docs/general/access-control#aiplatform.user) for the service account. This can be done in the [Google Cloud console](https://console.cloud.google.com/iam-admin/iam?project=autogen-with-gemini) in our `autogen-with-gemini` project.<br/>
Alternatively, we can also grant the [Vertex AI User](https://cloud.google.com/vertex-ai/docs/general/access-control#aiplatform.user) role by running a command using the gcloud CLI, for example in [Cloud Shell](https://shell.cloud.google.com/cloudshell):
```bash
gcloud projects add-iam-policy-binding autogen-with-gemini \
    --member=serviceAccount:autogen@autogen-with-gemini.iam.gserviceaccount.com --role roles/aiplatform.user
```

* Under IAM & Admin > Service Account select the newly created service accounts, and click the option "Manage keys" among the items. 
* From the "ADD KEY" dropdown select "Create new key" and select the JSON format and click CREATE.
    * The new key will be downloaded automatically. 
* You can then upload the service account key file to the from where you will be running autogen. 
    * Please consider restricting the permissions on the key file. For example, you could run `chmod 600 autogen-with-gemini-service-account-key.json` if your keyfile is called autogen-with-gemini-service-account-key.json.

### Configure Authentication

Authentication happens using standard [Google Cloud authentication methods](https://cloud.google.com/docs/authentication), <br/> which means
that either an already active session can be reused, or by specifying the Google application credentials of a service account. <br/><br/>
Additionally, AutoGen also supports authentication using `Credentials` objects in Python with the [google-auth library](https://google-auth.readthedocs.io/), which enables even more flexibility.<br/>
For example, we can even use impersonated credentials.

#### <a id='use_svc_keyfile'></a>Use Service Account Keyfile

The Google Cloud service account can be specified by setting the `GOOGLE_APPLICATION_CREDENTIALS` environment variable to the path to the JSON key file of the service account. <br/>

We could even just directly set the environment variable, or we can add the `"google_application_credentials"` key with the respective value for our model in the OAI_CONFIG_LIST.

#### Use the Google Default Credentials

If you are using [Cloud Shell](https://shell.cloud.google.com/cloudshell) or [Cloud Shell editor](https://shell.cloud.google.com/cloudshell/editor) in Google Cloud, <br/> then you are already authenticated. If you have the Google Cloud SDK installed locally,  <br/> then you can login by running `gcloud auth application-default login` in the command line. 

Detailed instructions for installing the Google Cloud SDK can be found [here](https://cloud.google.com/sdk/docs/install).

#### Authentication with the Google Auth Library for Python

The google-auth library supports a wide range of authentication scenarios, and you can simply pass a previously created `Credentials` object to the `llm_config`.<br/>
The [official documentation](https://google-auth.readthedocs.io/) of the Python package provides a detailed overview of the supported methods and usage examples.<br/>
If you are already authenticated, like in [Cloud Shell](https://shell.cloud.google.com/cloudshell), or after running the `gcloud auth application-default login` command in a CLI, then the `google.auth.default()` Python method will automatically return your currently active credentials.

## Example Config List
The config could look like the following (change `project_id` and `google_application_credentials`):
```python
config_list = [
    {
        "model": "gemini-pro",
        "api_type": "google",
        "project_id": "autogen-with-gemini",
        "location": "us-west1"
    },
    {
        "model": "gemini-1.5-pro-001",
        "api_type": "google",
        "project_id": "autogen-with-gemini",
        "location": "us-west1"
    },
    {
        "model": "gemini-1.5-pro",
        "api_type": "google",
        "project_id": "autogen-with-gemini",
        "location": "us-west1",
        "google_application_credentials": "autogen-with-gemini-service-account-key.json"
    },
    {
        "model": "gemini-pro-vision",
        "api_type": "google",
        "project_id": "autogen-with-gemini",
        "location": "us-west1"
    }
]
```



## Configure Safety Settings for VertexAI
Configuring safety settings for VertexAI is slightly different, as we have to use the speicialized safety setting object types instead of plain strings


```python
from vertexai.generative_models import HarmBlockThreshold, HarmCategory

safety_settings = {
    HarmCategory.HARM_CATEGORY_HARASSMENT: HarmBlockThreshold.BLOCK_ONLY_HIGH,
    HarmCategory.HARM_CATEGORY_HATE_SPEECH: HarmBlockThreshold.BLOCK_ONLY_HIGH,
    HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT: HarmBlockThreshold.BLOCK_ONLY_HIGH,
    HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT: HarmBlockThreshold.BLOCK_ONLY_HIGH,
}
```


```python
import os
from typing import Any, Callable, Dict, List, Optional, Tuple, Type, Union

import chromadb
from PIL import Image
from termcolor import colored

import autogen
from autogen import Agent, AssistantAgent, ConversableAgent, UserProxyAgent
from autogen.agentchat.contrib.img_utils import _to_pil, get_image_data
from autogen.agentchat.contrib.multimodal_conversable_agent import MultimodalConversableAgent
from autogen.agentchat.contrib.retrieve_user_proxy_agent import RetrieveUserProxyAgent
from autogen.code_utils import DEFAULT_MODEL, UNKNOWN, content_str, execute_code, extract_code, infer_lang
```


```python
config_list_gemini = autogen.config_list_from_json(
    "OAI_CONFIG_LIST",
    filter_dict={
        "model": ["gemini-1.5-pro"],
    },
)

config_list_gemini_vision = autogen.config_list_from_json(
    "OAI_CONFIG_LIST",
    filter_dict={
        "model": ["gemini-pro-vision"],
    },
)

for config_list in [config_list_gemini, config_list_gemini_vision]:
    for config_list_item in config_list:
        config_list_item["safety_settings"] = safety_settings

seed = 25  # for caching
```


```python
assistant = AssistantAgent(
    "assistant", llm_config={"config_list": config_list_gemini, "seed": seed}, max_consecutive_auto_reply=3
)

user_proxy = UserProxyAgent(
    "user_proxy",
    code_execution_config={"work_dir": "coding", "use_docker": False},
    human_input_mode="NEVER",
    is_termination_msg=lambda x: content_str(x.get("content")).find("TERMINATE") >= 0,
)

result = user_proxy.initiate_chat(
    assistant,
    message="""
    Compute the integral of the function f(x)=x^2 on the interval 0 to 1 using a Python script,
    which returns the value of the definite integral""",
)
```

    [33muser_proxy[0m (to assistant):
    
    
        Compute the integral of the function f(x)=x^2 on the interval 0 to 1 using a Python script,
        which returns the value of the definite integral
    
    --------------------------------------------------------------------------------
    [33massistant[0m (to user_proxy):
    
    Plan:
    1. (code) Use Python's `scipy.integrate.quad` function to compute the integral. 
    
    ```python
    # filename: integral.py
    from scipy.integrate import quad
    
    def f(x):
      return x**2
    
    result, error = quad(f, 0, 1)
    
    print(f"The definite integral of x^2 from 0 to 1 is: {result}")
    ```
    
    Let me know when you have executed this code. 
    
    
    --------------------------------------------------------------------------------
    [31m
    >>>>>>>> EXECUTING CODE BLOCK 0 (inferred language is python)...[0m
    [33muser_proxy[0m (to assistant):
    
    exitcode: 0 (execution succeeded)
    Code output: 
    The definite integral of x^2 from 0 to 1 is: 0.33333333333333337
    
    
    --------------------------------------------------------------------------------
    [33massistant[0m (to user_proxy):
    
    The script executed successfully and returned the definite integral's value as approximately 0.33333333333333337. 
    
    This aligns with the analytical solution. The indefinite integral of x^2 is (x^3)/3. Evaluating this from 0 to 1 gives us (1^3)/3 - (0^3)/3 = 1/3 = 0.33333...
    
    Therefore, the script successfully computed the integral of x^2 from 0 to 1.
    
    TERMINATE
    
    
    --------------------------------------------------------------------------------


## Example with Gemini Multimodal
Authentication is the same for vision models as for the text based Gemini models. <br/>
In this example an object of type `Credentials` will be supplied in order to authenticate.<br/>
Here, we will use the google application default credentials, so make sure to run the following commands if you are not yet authenticated:
```bash
export GOOGLE_APPLICATION_CREDENTIALS=autogen-with-gemini-service-account-key.json
gcloud auth application-default login
gcloud config set project autogen-with-gemini
```
The `GOOGLE_APPLICATION_CREDENTIALS` environment variable is a path to our service account JSON keyfile, as described in the [Use Service Account Keyfile](#use_svc_keyfile) section above.<br/>
We also need to set the Google cloud project, which is `autogen-with-gemini` in this example.<br/><br/>

Note, we could also run `gcloud auth application-default login` to use our personal Google account instead of a service account.
In this case we need to run the following commands:
```bash
gcloud gcloud auth application-default login
gcloud config set project autogen-with-gemini
```


```python
import google.auth

scopes = ["https://www.googleapis.com/auth/cloud-platform"]

credentials, project_id = google.auth.default(scopes)

gemini_vision_config = [
    {
        "model": "gemini-pro-vision",
        "api_type": "google",
        "project_id": project_id,
        "credentials": credentials,
        "location": "us-west1",
        "safety_settings": safety_settings,
    }
]
```


```python
image_agent = MultimodalConversableAgent(
    "Gemini Vision", llm_config={"config_list": gemini_vision_config, "seed": seed}, max_consecutive_auto_reply=1
)

user_proxy = UserProxyAgent("user_proxy", human_input_mode="NEVER", max_consecutive_auto_reply=0)

user_proxy.initiate_chat(
    image_agent,
    message="""Describe what is in this image?
<img https://github.com/ag2ai/ag2/blob/main/website/static/img/autogen_agentchat.png?raw=true>.""",
)
```

    [33muser_proxy[0m (to Gemini Vision):
    
    Describe what is in this image?
    <image>.
    
    --------------------------------------------------------------------------------
    [31m
    >>>>>>>> USING AUTO REPLY...[0m
    [33mGemini Vision[0m (to user_proxy):
    
     The image describes a conversational agent that is able to have a conversation with a human user. The agent can be customized to the user's preferences. The conversation can be in form of a joint chat or hierarchical chat.
    
    --------------------------------------------------------------------------------





    ChatResult(chat_id=None, chat_history=[{'content': 'Describe what is in this image?\n<img https://github.com/microsoft/autogen/blob/main/website/static/img/autogen_agentchat.png?raw=true>.', 'role': 'assistant'}, {'content': " The image describes a conversational agent that is able to have a conversation with a human user. The agent can be customized to the user's preferences. The conversation can be in form of a joint chat or hierarchical chat.", 'role': 'user'}], summary=" The image describes a conversational agent that is able to have a conversation with a human user. The agent can be customized to the user's preferences. The conversation can be in form of a joint chat or hierarchical chat.", cost={'usage_including_cached_inference': {'total_cost': 0.0001995, 'gemini-pro-vision': {'cost': 0.0001995, 'prompt_tokens': 267, 'completion_tokens': 44, 'total_tokens': 311}}, 'usage_excluding_cached_inference': {'total_cost': 0}}, human_input=[])



# Use Gemini via the OpenAI Library in Autogen
Using Gemini via the OpenAI library is also possible once you are already authenticated. <br/>
Run `gcloud auth application-default login` to set up application default credentials locally for the example below.<br/>
Also set the Google cloud project on the CLI if you have not done so far: <br/>
```bash
gcloud config set project autogen-with-gemini
```
The prerequisites are essentially the same as in the example above.<br/>

You can read more on the topic in the [official Google docs](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/call-gemini-using-openai-library).
<br/> A list of currently supported models can also be found in the [docs](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/call-gemini-using-openai-library#supported_models)
<br/>
<br/>
Note, that you will need to refresh your token regularly, by default every 1 hour.


```python
import google.auth

scopes = ["https://www.googleapis.com/auth/cloud-platform"]
creds, project = google.auth.default(scopes)
auth_req = google.auth.transport.requests.Request()
creds.refresh(auth_req)
location = "us-west1"
prompt_price_per_1k = (
    0.000125  # For more up-to-date prices see https://cloud.google.com/vertex-ai/generative-ai/pricing
)
completion_token_price_per_1k = (
    0.000375  # For more up-to-date prices see https://cloud.google.com/vertex-ai/generative-ai/pricing
)

openai_gemini_config = [
    {
        "model": "google/gemini-1.5-pro-001",
        "api_type": "openai",
        "base_url": f"https://{location}-aiplatform.googleapis.com/v1beta1/projects/{project}/locations/{location}/endpoints/openapi",
        "api_key": creds.token,
        "price": [prompt_price_per_1k, completion_token_price_per_1k],
    }
]
```


```python
assistant = AssistantAgent("assistant", llm_config={"config_list": openai_gemini_config}, max_consecutive_auto_reply=3)

user_proxy = UserProxyAgent(
    "user_proxy",
    code_execution_config={"work_dir": "coding", "use_docker": False},
    human_input_mode="NEVER",
    is_termination_msg=lambda x: content_str(x.get("content")).find("TERMINATE") >= 0,
)

result = user_proxy.initiate_chat(
    assistant,
    message="""
    Compute the integral of the function f(x)=x^3 on the interval 0 to 10 using a Python script,
    which returns the value of the definite integral.""",
)
```

    [33muser_proxy[0m (to assistant):
    
    
        Compute the integral of the function f(x)=x^3 on the interval 0 to 10 using a Python script,
        which returns the value of the definite integral.
    
    --------------------------------------------------------------------------------


    [33massistant[0m (to user_proxy):
    
    ```python
    # filename: integral.py
    def integrate_x_cubed(a, b):
      """
      This function calculates the definite integral of x^3 from a to b.
    
      Args:
          a: The lower limit of integration.
          b: The upper limit of integration.
    
      Returns:
          The value of the definite integral.
      """
      return (b**4 - a**4) / 4
    
    # Calculate the integral of x^3 from 0 to 10
    result = integrate_x_cubed(0, 10)
    
    # Print the result
    print(result)
    ```
    
    This script defines a function `integrate_x_cubed` that takes the lower and upper limits of integration as arguments and returns the definite integral of x^3 using the power rule of integration. The script then calls this function with the limits 0 and 10 and prints the result.
    
    Execute the script `python integral.py`, you should get the result: `2500.0`.
    
    TERMINATE
    
    
    --------------------------------------------------------------------------------
