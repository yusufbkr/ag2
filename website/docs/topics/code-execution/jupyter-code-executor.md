# Jupyter Code Executor

AutoGen is able to execute code in a stateful Jupyter kernel, this is in contrast to the command line code executor where each code block is executed in a new process. This means that you can define variables in one code block and use them in another. One of the interesting properties of this is that when an error is encountered, only the failing code needs to be re-executed, and not the entire script.

To use the [`JupyterCodeExecutor`](/docs/reference/coding/jupyter/jupyter_code_executor#jupytercodeexecutor) you need a Jupyter server running. This can be in Docker, local, or even a remote server. Then, when constructing the [`JupyterCodeExecutor`](/docs/reference/coding/jupyter/jupyter_code_executor#jupytercodeexecutor) you pass it the server it should connect to.

## Dependencies

In order to use Jupyter based code execution some extra dependencies are required. These can be installed with the extra `jupyter-executor`:

```bash
pip install 'pyautogen[jupyter-executor]'
```

## Jupyter Server

### Docker

To run a Docker based Jupyter server, the [`DockerJupyterServer`](/docs/reference/coding/jupyter/docker_jupyter_server#dockerjupyterserver) can be used.


```python
from autogen.coding import CodeBlock
from autogen.coding.jupyter import DockerJupyterServer, JupyterCodeExecutor

with DockerJupyterServer() as server:
    executor = JupyterCodeExecutor(server)
    print(
        executor.execute_code_blocks(
            code_blocks=[
                CodeBlock(language="python", code="print('Hello, World!')"),
            ]
        )
    )
```

    exit_code=0 output='Hello, World!\n' output_files=[]


By default the [`DockerJupyterServer`](/docs/reference/coding/jupyter/docker_jupyter_server#dockerjupyterserver) will build and use a bundled Dockerfile, which can be seen below:


```python
print(DockerJupyterServer.DEFAULT_DOCKERFILE)
```

    FROM quay.io/jupyter/docker-stacks-foundation
    
    SHELL ["/bin/bash", "-o", "pipefail", "-c"]
    
    USER ${NB_UID}
    RUN mamba install --yes jupyter_kernel_gateway ipykernel &&     mamba clean --all -f -y &&     fix-permissions "${CONDA_DIR}" &&     fix-permissions "/home/${NB_USER}"
    
    ENV TOKEN="UNSET"
    CMD python -m jupyter kernelgateway --KernelGatewayApp.ip=0.0.0.0     --KernelGatewayApp.port=8888     --KernelGatewayApp.auth_token="${TOKEN}"     --JupyterApp.answer_yes=true     --JupyterWebsocketPersonality.list_kernels=true
    
    EXPOSE 8888
    
    WORKDIR "${HOME}"
    


#### Custom Docker Image

A custom image can be used by passing the `custom_image_name` parameter to the [`DockerJupyterServer`](/docs/reference/coding/jupyter/docker_jupyter_server#dockerjupyterserver) constructor. There are some requirements of the image for it to work correctly:

- The image must have [Jupyer Kernel Gateway](https://jupyter-kernel-gateway.readthedocs.io/en/latest/) installed and running on port 8888 for the [`JupyterCodeExecutor`](/docs/reference/coding/jupyter/jupyter_code_executor) to be able to connect to it.
- Respect the `TOKEN` environment variable, which is used to authenticate the [`JupyterCodeExecutor`](/docs/reference/coding/jupyter/jupyter_code_executor) with the Jupyter server.
- Ensure the `jupyter kernelgateway` is started with:
    - `--JupyterApp.answer_yes=true` - this ensures that the kernel gateway does not prompt for confirmation when shut down.
    - `--JupyterWebsocketPersonality.list_kernels=true` - this ensures that the kernel gateway lists the available kernels.


If you wanted to add extra dependencies (for example `matplotlib` and `numpy`) to this image you could customize the Dockerfile like so:

```Dockerfile
FROM quay.io/jupyter/docker-stacks-foundation

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER ${NB_UID}
RUN mamba install --yes jupyter_kernel_gateway ipykernel matplotlib numpy &&
    mamba clean --all -f -y &&
    fix-permissions "${CONDA_DIR}" &&
    fix-permissions "/home/${NB_USER}"

ENV TOKEN="UNSET"
CMD python -m jupyter kernelgateway \
    --KernelGatewayApp.ip=0.0.0.0 \
    --KernelGatewayApp.port=8888 \
    --KernelGatewayApp.auth_token="${TOKEN}" \
    --JupyterApp.answer_yes=true \
    --JupyterWebsocketPersonality.list_kernels=true

EXPOSE 8888

WORKDIR "${HOME}"
```

````{=mdx}
:::tip
To learn about how to combine AutoGen in a Docker image while also executing code in a separate image go [here](/docs/topics/code-execution/cli-code-executor#combining-autogen-in-docker-with-a-docker-based-executor).
:::
````

### Local

````mdx-code-block
:::danger
The local version will run code on your local system. Use it with caution.
:::
````

To run a local Jupyter server, the [`LocalJupyterServer`](/docs/reference/coding/jupyter/local_jupyter_server#localjupyterserver) can be used.

````{=mdx}
:::warning
The [`LocalJupyterServer`](/docs/reference/coding/jupyter/local_jupyter_server#localjupyterserver) does not function on Windows due to a bug. In this case, you can use the [`DockerJupyterServer`](/docs/reference/coding/jupyter/docker_jupyter_server#dockerjupyterserver) instead or use the [`EmbeddedIPythonCodeExecutor`](/docs/reference/coding/jupyter/embedded_ipython_code_executor). Do note that the intention is to remove the [`EmbeddedIPythonCodeExecutor`](/docs/reference/coding/jupyter/embedded_ipython_code_executor) when the bug is fixed.
:::
````


```python
from autogen.coding import CodeBlock
from autogen.coding.jupyter import JupyterCodeExecutor, LocalJupyterServer

with LocalJupyterServer() as server:
    executor = JupyterCodeExecutor(server)
    print(
        executor.execute_code_blocks(
            code_blocks=[
                CodeBlock(language="python", code="print('Hello, World!')"),
            ]
        )
    )
```

### Remote

The [`JupyterCodeExecutor`](/docs/reference/coding/jupyter/jupyter_code_executor) can also connect to a remote Jupyter server. This is done by passing connection information rather than an actual server object into the [`JupyterCodeExecutor`](/docs/reference/coding/jupyter/jupyter_code_executor) constructor.

```python
from autogen.coding.jupyter import JupyterCodeExecutor, JupyterConnectionInfo

executor = JupyterCodeExecutor(
    jupyter_server=JupyterConnectionInfo(host='example.com', use_https=True, port=7893, token='mytoken')
)
```

## Image outputs

When Jupyter outputs an image, this is saved as a file into the `output_dir` of the [`JupyterCodeExecutor`](/docs/reference/coding/jupyter/jupyter_code_executor), as specified by the constructor. By default this is the current working directory.

## Assigning to an agent

A single server can support multiple agents, as each executor will create its own kernel. To assign an executor to an agent it can be done like so:


```python
from pathlib import Path

from autogen import ConversableAgent
from autogen.coding.jupyter import DockerJupyterServer, JupyterCodeExecutor

server = DockerJupyterServer()

output_dir = Path("coding")
output_dir.mkdir(exist_ok=True)

code_executor_agent = ConversableAgent(
    name="code_executor_agent",
    llm_config=False,
    code_execution_config={
        "executor": JupyterCodeExecutor(server, output_dir=output_dir),
    },
    human_input_mode="NEVER",
)
```

When using code execution it is critical that you update the system prompt of agents you expect to write code to be able to make use of the executor. For example, for the [`JupyterCodeExecutor`](/docs/reference/coding/jupyter/jupyter_code_executor) you might setup a code writing agent like so:


```python
# The code writer agent's system message is to instruct the LLM on how to
# use the Jupyter code executor with IPython kernel.
code_writer_system_message = """
You have been given coding capability to solve tasks using Python code in a stateful IPython kernel.
You are responsible for writing the code, and the user is responsible for executing the code.

When you write Python code, put the code in a markdown code block with the language set to Python.
For example:
```python
x = 3
```
You can use the variable `x` in subsequent code blocks.
```python
print(x)
```

Write code incrementally and leverage the statefulness of the kernel to avoid repeating code.
Import libraries in a separate code block.
Define a function or a class in a separate code block.
Run code that produces output in a separate code block.
Run code that involves expensive operations like download, upload, and call external APIs in a separate code block.

When your code produces an output, the output will be returned to you.
Because you have limited conversation memory, if your code creates an image,
the output will be a path to the image instead of the image itself."""

import os

code_writer_agent = ConversableAgent(
    "code_writer",
    system_message=code_writer_system_message,
    llm_config={"config_list": [{"model": "gpt-4", "api_key": os.environ["OPENAI_API_KEY"]}]},
    code_execution_config=False,  # Turn off code execution for this agent.
    max_consecutive_auto_reply=2,
    human_input_mode="NEVER",
)
```

Then we can use these two agents to solve a problem:


```python
import pprint

chat_result = code_executor_agent.initiate_chat(
    code_writer_agent, message="Write Python code to calculate the 14th Fibonacci number."
)

pprint.pprint(chat_result)
```

    [33mcode_executor_agent[0m (to code_writer):
    
    Write Python code to calculate the 14th Fibonacci number.
    
    --------------------------------------------------------------------------------
    [33mcode_writer[0m (to code_executor_agent):
    
    Sure. The Fibonacci sequence is a series of numbers where the next number is found by adding up the two numbers before it. We know that the first two Fibonacci numbers are 0 and 1. After that, the series looks like:
    
    0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, ...
    
    So, let's define a Python function to calculate the nth Fibonacci number.
    
    --------------------------------------------------------------------------------
    [33mcode_executor_agent[0m (to code_writer):
    
    
    
    --------------------------------------------------------------------------------
    [33mcode_writer[0m (to code_executor_agent):
    
    Here is the Python function to calculate the nth Fibonacci number:
    
    ```python
    def fibonacci(n):
        if n <= 1:
            return n
        else:
            a, b = 0, 1
            for _ in range(2, n+1):
                a, b = b, a+b
            return b
    ```
    
    Now, let's use this function to calculate the 14th Fibonacci number.
    
    ```python
    fibonacci(14)
    ```
    
    --------------------------------------------------------------------------------
    [33mcode_executor_agent[0m (to code_writer):
    
    exitcode: 0 (execution succeeded)
    Code output: 
    377
    
    --------------------------------------------------------------------------------
    ChatResult(chat_id=None,
               chat_history=[{'content': 'Write Python code to calculate the 14th '
                                         'Fibonacci number.',
                              'role': 'assistant'},
                             {'content': 'Sure. The Fibonacci sequence is a series '
                                         'of numbers where the next number is '
                                         'found by adding up the two numbers '
                                         'before it. We know that the first two '
                                         'Fibonacci numbers are 0 and 1. After '
                                         'that, the series looks like:\n'
                                         '\n'
                                         '0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, '
                                         '...\n'
                                         '\n'
                                         "So, let's define a Python function to "
                                         'calculate the nth Fibonacci number.',
                              'role': 'user'},
                             {'content': '', 'role': 'assistant'},
                             {'content': 'Here is the Python function to calculate '
                                         'the nth Fibonacci number:\n'
                                         '\n'
                                         '```python\n'
                                         'def fibonacci(n):\n'
                                         '    if n <= 1:\n'
                                         '        return n\n'
                                         '    else:\n'
                                         '        a, b = 0, 1\n'
                                         '        for _ in range(2, n+1):\n'
                                         '            a, b = b, a+b\n'
                                         '        return b\n'
                                         '```\n'
                                         '\n'
                                         "Now, let's use this function to "
                                         'calculate the 14th Fibonacci number.\n'
                                         '\n'
                                         '```python\n'
                                         'fibonacci(14)\n'
                                         '```',
                              'role': 'user'},
                             {'content': 'exitcode: 0 (execution succeeded)\n'
                                         'Code output: \n'
                                         '377',
                              'role': 'assistant'}],
               summary='exitcode: 0 (execution succeeded)\nCode output: \n377',
               cost=({'gpt-4-0613': {'completion_tokens': 193,
                                     'cost': 0.028499999999999998,
                                     'prompt_tokens': 564,
                                     'total_tokens': 757},
                      'total_cost': 0.028499999999999998},
                     {'gpt-4-0613': {'completion_tokens': 193,
                                     'cost': 0.028499999999999998,
                                     'prompt_tokens': 564,
                                     'total_tokens': 757},
                      'total_cost': 0.028499999999999998}),
               human_input=[])


Finally, stop the server. Or better yet use a context manager for it to be stopped automatically.


```python
server.stop()
```
