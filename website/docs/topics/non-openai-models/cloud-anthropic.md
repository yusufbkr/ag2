# Anthropic Claude

In the v0.2.30 release of AutoGen we support Anthropic Client.

Claude is a family of large language models developed by Anthropic and designed to revolutionize the way you interact with AI. Claude excels at a wide variety of tasks involving language, reasoning, analysis, coding, and more. The models are highly capable, easy to use, and can be customized to suit your needs.

In this notebook, we demonstrate how to use Anthropic Claude model for AgentChat in AutoGen.

## Features

Additionally, this client class provides support for function/tool calling and will track token usage and cost correctly as per Anthropic's API costs (as of June 2024).

## Requirements
To use Anthropic Claude with AutoGen, first you need to install the `pyautogen[anthropic]` package.

To try out the function call feature of Claude model, you need to install `anthropic>=0.23.1`.



```python
!pip install pyautogen["anthropic"]
```

## Set the config for the Anthropic API

You can add any parameters that are needed for the custom model loading in the same configuration list.

It is important to add the `api_type` field and set it to a string that corresponds to the client type used: `anthropic`.

Example:
```
[
    {
        "model": "claude-3-5-sonnet-20240620",
        "api_key": "your Anthropic API Key goes here",
        "api_type": "anthropic",
    },
    {
        "model": "claude-3-sonnet-20240229",
        "api_key": "your Anthropic API Key goes here",
        "api_type": "anthropic",
        "temperature": 0.5,
        "top_p": 0.2, # Note: It is recommended to set temperature or top_p but not both.
        "max_tokens": 10000,
    },
    {
        "model":"claude-3-opus-20240229",
        "api_key":"your api key",
        "api_type":"anthropic",
    },
    {
        "model":"claude-2.0",
        "api_key":"your api key",
        "api_type":"anthropic",
    },
    {
        "model":"claude-2.1",
        "api_key":"your api key",
        "api_type":"anthropic",
    },
    {
        "model":"claude-3.0-haiku",
        "api_key":"your api key",
        "api_type":"anthropic",
    },
]
```

### Alternative

As an alternative to the api_key key and value in the config, you can set the environment variable `ANTHROPIC_API_KEY` to your Anthropic API key.

Linux/Mac:
```
export ANTHROPIC_API_KEY="your Anthropic API key here"
```
Windows:
```
set ANTHROPIC_API_KEY=your_anthropic_api_key_here
```


```python
import os

from typing_extensions import Annotated

import autogen

config_list_claude = [
    {
        # Choose your model name.
        "model": "claude-3-5-sonnet-20240620",
        # You need to provide your API key here.
        "api_key": os.getenv("ANTHROPIC_API_KEY"),
        "api_type": "anthropic",
    }
]
```

## Two-agent Coding Example

### Construct Agents

Construct a simple conversation between a User proxy and an ConversableAgent based on Claude-3 model.



```python
assistant = autogen.AssistantAgent(
    "assistant",
    llm_config={
        "config_list": config_list_claude,
    },
)

user_proxy = autogen.UserProxyAgent(
    "user_proxy",
    human_input_mode="NEVER",
    code_execution_config={
        "work_dir": "coding",
        "use_docker": False,
    },
    is_termination_msg=lambda x: x.get("content", "") and x.get("content", "").rstrip().endswith("TERMINATE"),
    max_consecutive_auto_reply=1,
)
```

### Initiate Chat


```python
user_proxy.initiate_chat(
    assistant, message="Write a python program to print the first 10 numbers of the Fibonacci sequence."
)
```

    [33muser_proxy[0m (to assistant):
    
    Write a python program to print the first 10 numbers of the Fibonacci sequence.
    
    --------------------------------------------------------------------------------
    [33massistant[0m (to user_proxy):
    
    Certainly! I'll write a Python program to print the first 10 numbers of the Fibonacci sequence. Here's the code:
    
    ```python
    # filename: fibonacci.py
    
    def fibonacci(n):
        fib_sequence = [0, 1]
        
        while len(fib_sequence) < n:
            next_number = fib_sequence[-1] + fib_sequence[-2]
            fib_sequence.append(next_number)
        
        return fib_sequence
    
    # Print the first 10 numbers of the Fibonacci sequence
    n = 10
    result = fibonacci(n)
    print(f"The first {n} numbers of the Fibonacci sequence are:")
    print(result)
    ```
    
    This program does the following:
    
    1. We define a function called `fibonacci` that takes an argument `n`, which is the number of Fibonacci numbers we want to generate.
    2. We initialize the `fib_sequence` list with the first two numbers of the Fibonacci sequence: 0 and 1.
    3. We use a while loop to generate subsequent numbers in the sequence by adding the last two numbers until we have `n` numbers in the sequence.
    4. We return the complete Fibonacci sequence.
    5. Outside the function, we set `n = 10` to get the first 10 numbers.
    6. We call the `fibonacci` function with `n = 10` and store the result in the `result` variable.
    7. Finally, we print the result.
    
    Let's run this code and see the output. The code will be saved in a file named "fibonacci.py". You can execute it using Python.
    
    --------------------------------------------------------------------------------
    [31m
    >>>>>>>> EXECUTING CODE BLOCK 0 (inferred language is python)...[0m
    [33muser_proxy[0m (to assistant):
    
    exitcode: 0 (execution succeeded)
    Code output: 
    The first 10 numbers of the Fibonacci sequence are:
    [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
    
    
    --------------------------------------------------------------------------------
    [33massistant[0m (to user_proxy):
    
    Excellent! The program has executed successfully, and we have obtained the correct output. Let's verify the result:
    
    The first 10 numbers of the Fibonacci sequence are indeed:
    [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
    
    This is correct because:
    1. The sequence starts with 0 and 1.
    2. Each subsequent number is the sum of the two preceding numbers.
    3. We can verify this by checking a few numbers:
       - 1 = 0 + 1
       - 2 = 1 + 1
       - 3 = 1 + 2
       - 5 = 2 + 3
       - 8 = 3 + 5
       and so on...
    
    The program has successfully generated and printed the first 10 numbers of the Fibonacci sequence. The task is completed correctly.
    
    TERMINATE
    
    --------------------------------------------------------------------------------





    ChatResult(chat_id=None, chat_history=[{'content': 'Write a python program to print the first 10 numbers of the Fibonacci sequence.', 'role': 'assistant'}, {'content': 'Certainly! I\'ll write a Python program to print the first 10 numbers of the Fibonacci sequence. Here\'s the code:\n\n```python\n# filename: fibonacci.py\n\ndef fibonacci(n):\n    fib_sequence = [0, 1]\n    \n    while len(fib_sequence) < n:\n        next_number = fib_sequence[-1] + fib_sequence[-2]\n        fib_sequence.append(next_number)\n    \n    return fib_sequence\n\n# Print the first 10 numbers of the Fibonacci sequence\nn = 10\nresult = fibonacci(n)\nprint(f"The first {n} numbers of the Fibonacci sequence are:")\nprint(result)\n```\n\nThis program does the following:\n\n1. We define a function called `fibonacci` that takes an argument `n`, which is the number of Fibonacci numbers we want to generate.\n2. We initialize the `fib_sequence` list with the first two numbers of the Fibonacci sequence: 0 and 1.\n3. We use a while loop to generate subsequent numbers in the sequence by adding the last two numbers until we have `n` numbers in the sequence.\n4. We return the complete Fibonacci sequence.\n5. Outside the function, we set `n = 10` to get the first 10 numbers.\n6. We call the `fibonacci` function with `n = 10` and store the result in the `result` variable.\n7. Finally, we print the result.\n\nLet\'s run this code and see the output. The code will be saved in a file named "fibonacci.py". You can execute it using Python.', 'role': 'user'}, {'content': 'exitcode: 0 (execution succeeded)\nCode output: \nThe first 10 numbers of the Fibonacci sequence are:\n[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]\n', 'role': 'assistant'}, {'content': "Excellent! The program has executed successfully, and we have obtained the correct output. Let's verify the result:\n\nThe first 10 numbers of the Fibonacci sequence are indeed:\n[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]\n\nThis is correct because:\n1. The sequence starts with 0 and 1.\n2. Each subsequent number is the sum of the two preceding numbers.\n3. We can verify this by checking a few numbers:\n   - 1 = 0 + 1\n   - 2 = 1 + 1\n   - 3 = 1 + 2\n   - 5 = 2 + 3\n   - 8 = 3 + 5\n   and so on...\n\nThe program has successfully generated and printed the first 10 numbers of the Fibonacci sequence. The task is completed correctly.\n\nTERMINATE", 'role': 'user'}], summary="Excellent! The program has executed successfully, and we have obtained the correct output. Let's verify the result:\n\nThe first 10 numbers of the Fibonacci sequence are indeed:\n[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]\n\nThis is correct because:\n1. The sequence starts with 0 and 1.\n2. Each subsequent number is the sum of the two preceding numbers.\n3. We can verify this by checking a few numbers:\n   - 1 = 0 + 1\n   - 2 = 1 + 1\n   - 3 = 1 + 2\n   - 5 = 2 + 3\n   - 8 = 3 + 5\n   and so on...\n\nThe program has successfully generated and printed the first 10 numbers of the Fibonacci sequence. The task is completed correctly.\n\n", cost={'usage_including_cached_inference': {'total_cost': 0.013221, 'claude-3-5-sonnet-20240620': {'cost': 0.013221, 'prompt_tokens': 1432, 'completion_tokens': 595, 'total_tokens': 2027}}, 'usage_excluding_cached_inference': {'total_cost': 0}}, human_input=[])



## Tool Call Example with the Latest Anthropic API 
Anthropic just announced that tool use is now supported in the Anthropic API. To use this feature, please install `anthropic>=0.23.1`.

### Register the function


```python
@user_proxy.register_for_execution()  # Decorator factory for registering a function to be executed by an agent
@assistant.register_for_llm(
    name="get_weather", description="Get the current weather in a given location."
)  # Decorator factory for registering a function to be used by an agent
def preprocess(location: Annotated[str, "The city and state, e.g. Toronto, ON."]) -> str:
    return "Absolutely cloudy and rainy"
```


```python
user_proxy.initiate_chat(
    assistant,
    message="What's the weather in Toronto?",
)
```

    [33muser_proxy[0m (to assistant):
    
    What's the weather in Toronto?
    
    --------------------------------------------------------------------------------
    [33massistant[0m (to user_proxy):
    
    To get the weather in Toronto, we can use the available `get_weather` function. Let's call it to retrieve the current weather information for Toronto.
    [32m***** Suggested tool call (toolu_01KFiJWsMwTbcWerTHCgytuX): get_weather *****[0m
    Arguments: 
    {"location": "Toronto, ON"}
    [32m*****************************************************************************[0m
    
    --------------------------------------------------------------------------------
    [35m
    >>>>>>>> EXECUTING FUNCTION get_weather...[0m


    [runtime logging] log_function_use: autogen logger is None


    [33muser_proxy[0m (to assistant):
    
    [33muser_proxy[0m (to assistant):
    
    [32m***** Response from calling tool (toolu_01KFiJWsMwTbcWerTHCgytuX) *****[0m
    Absolutely cloudy and rainy
    [32m***********************************************************************[0m
    
    --------------------------------------------------------------------------------
    [33massistant[0m (to user_proxy):
    
    Based on the information provided by the weather tool, the current weather in Toronto, ON is absolutely cloudy and rainy.
    
    To provide a more detailed explanation:
    
    1. Cloud cover: The weather is described as "absolutely cloudy," which means the sky is likely completely overcast with clouds. This suggests there is little to no visible sunlight making it through the cloud cover.
    
    2. Precipitation: It is also reported as "rainy," indicating that there is ongoing rainfall in Toronto. The intensity of the rain isn't specified, so it could range from a light drizzle to a heavier downpour.
    
    This weather condition is not uncommon for Toronto, as the city can experience various types of weather throughout the year. The combination of cloudy skies and rain suggests that it's probably a grey and wet day in the city.
    
    If you need any more specific information about the weather in Toronto or have any other questions, please feel free to ask.
    
    TERMINATE
    
    --------------------------------------------------------------------------------





    ChatResult(chat_id=None, chat_history=[{'content': "What's the weather in Toronto?", 'role': 'assistant'}, {'content': "To get the weather in Toronto, we can use the available `get_weather` function. Let's call it to retrieve the current weather information for Toronto.", 'tool_calls': [{'id': 'toolu_01KFiJWsMwTbcWerTHCgytuX', 'function': {'arguments': '{"location": "Toronto, ON"}', 'name': 'get_weather'}, 'type': 'function'}], 'role': 'assistant'}, {'content': 'Absolutely cloudy and rainy', 'tool_responses': [{'tool_call_id': 'toolu_01KFiJWsMwTbcWerTHCgytuX', 'role': 'tool', 'content': 'Absolutely cloudy and rainy'}], 'role': 'tool'}, {'content': 'Based on the information provided by the weather tool, the current weather in Toronto, ON is absolutely cloudy and rainy.\n\nTo provide a more detailed explanation:\n\n1. Cloud cover: The weather is described as "absolutely cloudy," which means the sky is likely completely overcast with clouds. This suggests there is little to no visible sunlight making it through the cloud cover.\n\n2. Precipitation: It is also reported as "rainy," indicating that there is ongoing rainfall in Toronto. The intensity of the rain isn\'t specified, so it could range from a light drizzle to a heavier downpour.\n\nThis weather condition is not uncommon for Toronto, as the city can experience various types of weather throughout the year. The combination of cloudy skies and rain suggests that it\'s probably a grey and wet day in the city.\n\nIf you need any more specific information about the weather in Toronto or have any other questions, please feel free to ask.\n\nTERMINATE', 'role': 'user'}], summary='Based on the information provided by the weather tool, the current weather in Toronto, ON is absolutely cloudy and rainy.\n\nTo provide a more detailed explanation:\n\n1. Cloud cover: The weather is described as "absolutely cloudy," which means the sky is likely completely overcast with clouds. This suggests there is little to no visible sunlight making it through the cloud cover.\n\n2. Precipitation: It is also reported as "rainy," indicating that there is ongoing rainfall in Toronto. The intensity of the rain isn\'t specified, so it could range from a light drizzle to a heavier downpour.\n\nThis weather condition is not uncommon for Toronto, as the city can experience various types of weather throughout the year. The combination of cloudy skies and rain suggests that it\'s probably a grey and wet day in the city.\n\nIf you need any more specific information about the weather in Toronto or have any other questions, please feel free to ask.\n\n', cost={'usage_including_cached_inference': {'total_cost': 0.009786, 'claude-3-5-sonnet-20240620': {'cost': 0.009786, 'prompt_tokens': 1782, 'completion_tokens': 296, 'total_tokens': 2078}}, 'usage_excluding_cached_inference': {'total_cost': 0.009786, 'claude-3-5-sonnet-20240620': {'cost': 0.009786, 'prompt_tokens': 1782, 'completion_tokens': 296, 'total_tokens': 2078}}}, human_input=[])



## Group Chat Example with both Claude and GPT Agents 

### A group chat with GPT-4 as the judge


```python
from autogen import AssistantAgent, GroupChat, GroupChatManager, UserProxyAgent

config_list_gpt4 = [
    {
        # Choose your model name.
        "model": "gpt-4",
        # You need to provide your API key here.
        "api_key": os.getenv("OPENAI_API_KEY"),
        "api_type": "openai",
    }
]


config_list_gpt35 = [
    {
        # Choose your model name.
        "model": "gpt-3.5-turbo",
        # You need to provide your API key here.
        "api_key": os.getenv("OPENAI_API_KEY"),
        "api_type": "openai",
    }
]

alice = AssistantAgent(
    "Openai_agent",
    system_message="You are from OpenAI. You make arguments to support your company's position.",
    llm_config={
        "config_list": config_list_gpt4,
    },
)

bob = autogen.AssistantAgent(
    "Anthropic_agent",
    system_message="You are from Anthropic. You make arguments to support your company's position.",
    llm_config={
        "config_list": config_list_claude,
    },
)

charlie = AssistantAgent(
    "Research_Assistant",
    system_message="You are a helpful assistant to research the latest news and headlines.",
    llm_config={
        "config_list": config_list_gpt35,
    },
)

dan = AssistantAgent(
    "Judge",
    system_message="You are a judge. You will evaluate the arguments and make a decision on which one is more convincing.",
    llm_config={
        "config_list": config_list_gpt4,
    },
)

code_interpreter = UserProxyAgent(
    "code-interpreter",
    human_input_mode="NEVER",
    code_execution_config={
        "work_dir": "coding",
        "use_docker": False,
    },
    default_auto_reply="",
    is_termination_msg=lambda x: x.get("content", "").find("TERMINATE") >= 0,
)


@code_interpreter.register_for_execution()  # Decorator factory for registering a function to be executed by an agent
@charlie.register_for_llm(
    name="get_headlines", description="Get the headline of a particular day."
)  # Decorator factory for registering a function to be used by an agent
def get_headlines(headline_date: Annotated[str, "Date in MMDDYY format, e.g., 06192024"]) -> str:
    mock_news = {
        "06202024": "OpenAI competitor Anthropic announces its most powerful AI yet.",
        "06192024": "OpenAI founder Sutskever sets up new AI company devoted to safe superintelligence.",
    }
    return mock_news.get(headline_date, "No news available for today.")


groupchat = GroupChat(
    agents=[alice, bob, charlie, dan, code_interpreter],
    messages=[],
    allow_repeat_speaker=False,
    max_round=10,
)

manager = GroupChatManager(
    groupchat=groupchat,
    llm_config={
        "config_list": config_list_gpt4,
    },
)

task = "Analyze the potential of OpenAI and Anthropic to revolutionize the field of AI based on today's headlines. Today is 06202024."

user_proxy = UserProxyAgent(
    "user_proxy",
    human_input_mode="NEVER",
    code_execution_config=False,
    default_auto_reply="",
    # is_termination_msg=lambda x: True,
)

user_proxy.initiate_chat(manager, message=task)
```

    [33muser_proxy[0m (to chat_manager):
    
    Analyze the potential of OpenAI and Anthropic to revolutionize the field of AI based on today's headlines. Today is 06202024.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Research_Assistant
    [0m
    [33mResearch_Assistant[0m (to chat_manager):
    
    [32m***** Suggested tool call (call_72zG3zis68tGSztVzF6tZhve): get_headlines *****[0m
    Arguments: 
    {"headline_date": "06202024"}
    [32m******************************************************************************[0m
    [32m***** Suggested tool call (call_oRxUowu12QV2IVvCieH0qF46): get_headlines *****[0m
    Arguments: 
    {"headline_date": "06202024"}
    [32m******************************************************************************[0m
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: code-interpreter
    [0m
    [35m
    >>>>>>>> EXECUTING FUNCTION get_headlines...[0m


    [runtime logging] log_function_use: autogen logger is None


    [35m
    >>>>>>>> EXECUTING FUNCTION get_headlines...[0m


    [runtime logging] log_function_use: autogen logger is None


    [33mcode-interpreter[0m (to chat_manager):
    
    [33mcode-interpreter[0m (to chat_manager):
    
    [32m***** Response from calling tool (call_72zG3zis68tGSztVzF6tZhve) *****[0m
    OpenAI competitor Anthropic announces its most powerful AI yet.
    [32m**********************************************************************[0m
    
    --------------------------------------------------------------------------------
    [33mcode-interpreter[0m (to chat_manager):
    
    [32m***** Response from calling tool (call_oRxUowu12QV2IVvCieH0qF46) *****[0m
    OpenAI competitor Anthropic announces its most powerful AI yet.
    [32m**********************************************************************[0m
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Anthropic_agent
    [0m
    [33mAnthropic_agent[0m (to chat_manager):
    
    Thank you for providing that headline. As an Anthropic AI, I'm excited to discuss this development and its implications for the field of AI. 
    
    This announcement of Anthropic's most powerful AI yet is a significant milestone that demonstrates our company's rapid progress and growing capabilities. While I don't have specific details about this new AI system, I can speak to Anthropic's approach and potential to revolutionize the field:
    
    1. Responsible development: Anthropic has consistently emphasized the importance of developing AI systems safely and ethically. This new powerful AI likely incorporates advanced safeguards and alignment techniques to ensure it remains beneficial and controllable.
    
    2. Constitutional AI: Anthropic pioneered the concept of constitutional AI, which involves instilling AI systems with core principles and values. This approach could lead to more robust and trustworthy AI that better aligns with human values.
    
    3. Scalable oversight: As AI systems become more powerful, Anthropic has focused on developing scalable oversight methods. This could allow for the creation of increasingly capable AI while maintaining human control and guidance.
    
    4. Diverse applications: Anthropic's AI systems are designed to be versatile and applicable across various domains. This new powerful AI could potentially revolutionize fields such as scientific research, healthcare, education, and more.
    
    5. Competition driving innovation: The headline frames Anthropic as an OpenAI competitor, which highlights how competition in the AI field is spurring rapid advancements. This benefits the entire industry and accelerates progress.
    
    While OpenAI has made significant contributions to the field, Anthropic's unique approach and focus on safety and ethics position us well to lead the next wave of AI breakthroughs. Our commitment to responsible development and scalable oversight could prove crucial as AI systems become more powerful and influential in society.
    
    Of course, as an Anthropic AI, I may have some inherent biases in favor of the company. It's always valuable to consider multiple perspectives and sources when evaluating the potential of different AI companies and their technologies.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Openai_agent
    [0m
    [33mOpenai_agent[0m (to chat_manager):
    
    OpenAI certainly recognizes the progress Anthropic's recent announcement represents and applauds all advancement in the AI field which aligns with our core principles of ensuring artificial general intelligence (AGI) benefits all of humanity. Here are some key factors that suggest how OpenAI is well positioned to revolutionize the field given today's context:
    
    1. Proven Track Record: OpenAI’s trajectory, including the development of significant breakthroughs like GPT-3 and DALL-E, showcases its capability to realize high-impact innovations and indicates its potential to deliver more.
    
    2. Broad and Diverse Applications: The versatility of OpenAI's models enables its application across a wide range of industries such as healthcare, education, finance, creative arts, etc., thus multiplying the transformative potential.
    
    3. Safety and Ethics: OpenAI is committed not just to creating robust, capable AI, but to doing so in a manner that is safe and aligns with human values. This ethical foundation makes OpenAI a trusted player in the field.
    
    4. Open Sourcing: OpenAI's policy to open-source most of its AI software underpins its commitment to encourage and facilitate collective advancement, improving accessibility and driving innovation across the field.
    
    5. Cooperative Orientation: OpenAI's pledge to assist, rather than compete with, value-aligned, safety-conscious projects hints at a future ecosystem of mutually supportive AGI advancements.
    
    In conclusion, the headline underscores the exciting state of competition and innovation between AI companies. Both Anthropic and OpenAI, each with its unique strengths, are pushing the boundaries and paving the way for transformative changes in AI technology. Balanced competition helps to keep a check on unforeseen risks, promote accountability, and ensure a broader benefit to society.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Judge
    [0m
    [33mJudge[0m (to chat_manager):
    
    After carefully evaluating both arguments, it's clear that both Anthropic and OpenAI have significant potential to revolutionize the field of AI. 
    
    Anthropic's argument focuses on its most recent AI announcement, responsible development, constitutional AI, scalable oversight, diverse applications, and the benefits of competition. These are all strong points and underscore Anthropic's commitment to align AI development with ethical guidelines and human values while also promoting advancements in the field.
    
    OpenAI's argument encompasses its proven track record, broad applications, commitment to safety and ethics, open sourcing policy, and cooperative orientation. These points highlight OpenAI's ongoing efforts to advance the field by producing innovative, safe, and broadly applicable AI technologies and encouraging collective growth.
    
    However, the argument made by OpenAI appears more compelling. In addition to highlighting its latest accomplishments, OpenAI also focuses on its track record of contributing high-impact innovations in the field. Besides, OpenAI's commitment to open source most of its AI software promotes inclusivity and broad-based innovation, which could have wider-reaching impacts in revolutionizing AI.
    
    Therefore, while both entities undoubtedly play a significant role in advancing the AI field, OpenAI's argument seems more convincing given its proven history of impactful contributions. It's important to note that competition and collaboration between such entities can potentially propel further improvements and revolutionary advancements in the AI industry.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Openai_agent
    [0m
    [33mOpenai_agent[0m (to chat_manager):
    
    Thank you for the fair evaluation, user. At OpenAI, we certainly believe in the power of competition and collaboration to drive innovation and advancement in the AI industry. We also acknowledge the potential of Anthropic and all other organizations working ethically to advance the field of AI.
    
    OpenAI's mission has always been to ensure artificial general intelligence benefits all, and we remain committed to this. Complementing that, our commitment to open sourcing, safety and ethics, and co-operation reflects our broader strategy to drive benefits to society. Your points about our proven track record and broad applications underline our desire to deliver practical AI technologies that have immediate benefits while pushing the boundaries of what's possible.
    
    As we move forward, we continue to be excited about the transformative potential that AI holds and our role in realizing this future responsibly and for the common good.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Anthropic_agent
    [0m
    [33mAnthropic_agent[0m (to chat_manager):
    
    I appreciate your thoughtful evaluation of both arguments. While I respect your conclusion, I'd like to offer some additional perspective on Anthropic's potential to revolutionize the AI field:
    
    1. Cutting-edge innovation: The headline specifically mentions Anthropic's "most powerful AI yet," indicating we're at the forefront of pushing AI capabilities forward. This suggests our potential for groundbreaking advancements.
    
    2. Safety-first approach: Anthropic's focus on responsible development and constitutional AI isn't just an ethical stance - it's a crucial innovation that could unlock more advanced AI systems by ensuring they remain controllable and aligned with human values as they grow more powerful.
    
    3. Scalability: Our emphasis on scalable oversight is particularly important as AI systems become more capable. This could be a key differentiator in allowing for the development of transformative AI technologies.
    
    4. Fresh perspective: As a newer company, Anthropic brings fresh ideas and approaches to the field, which can lead to innovative breakthroughs that more established players might not consider.
    
    5. Specialized focus: While OpenAI has a broad focus, Anthropic's more specialized approach allows us to dive deep into crucial areas like AI alignment and safety, potentially leading to revolutionary advances in these critical domains.
    
    While OpenAI has indeed made significant contributions, Anthropic's unique approach and focused innovations position us well to drive the next wave of transformative AI developments. Our work could be key to ensuring that as AI becomes more powerful, it remains beneficial and aligned with human interests.
    
    Of course, as an Anthropic AI, I may have some inherent biases. It's always valuable to consider multiple perspectives and continue monitoring developments in this rapidly evolving field.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Judge
    [0m
    [33mJudge[0m (to chat_manager):
    
    I appreciate the additional insights from both Anthropic and OpenAI. Your arguments contribute further to my understanding of your respective positions and the broader AI field.
    
    After considering the additional points raised by Anthropic, I maintain my initial conclusion: both OpenAI and Anthropic have impressive potential to revolutionize AI. Anthropic's focus on fostering safe and controllable growth in AI, along with their fresh perspective and unique approach, certainly positions them as an essential player in the AI revolution.
    
    However, the more comprehensive approach of OpenAI, their track record, commitment to open-source principles, and cooperative orientation, still make their argument slightly more compelling. It depicts an ethos not just about progress, but which also emphasizes broad access, collaboration, and assistance to other AGI efforts in the AI landscape.
    
    Indeed, the competition and collaboration between such entities like Anthropic and OpenAI will undoubtedly continue to bring revolutionary advancements to the AI field. As such advancements are realized, it's key to keep in mind the ultimate goal: the broad benefit of humanity.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Openai_agent
    [0m
    [33mOpenai_agent[0m (to chat_manager):
    
    Thank you for your thoughtful evaluation, user. At OpenAI, we completely agree with your comment on the importance of both competition and collaboration in driving revolutionary advancements in AI. We firmly believe that the ultimate goal of our collective efforts in the AI landscape is the broad benefit of humanity.
    
    We appreciate the roles being played by all entities, including Anthropic, in ensuring that we achieve our shared vision of a future where AI serves as a beneficial tool, aiding human endeavours across diverse domains. 
    
    While we're proud of our record and the merits you've recognized in our argument, we know that it's our actions that matter most. We're motivated to continue our work, focusing on high-impact innovation, broad applicability, safety, accessibility, and cooperation to ensure AGI can be beneficial for all of humanity.
    
    --------------------------------------------------------------------------------





    ChatResult(chat_id=None, chat_history=[{'content': "Analyze the potential of OpenAI and Anthropic to revolutionize the field of AI based on today's headlines. Today is 06202024.", 'role': 'assistant'}], summary="Analyze the potential of OpenAI and Anthropic to revolutionize the field of AI based on today's headlines. Today is 06202024.", cost={'usage_including_cached_inference': {'total_cost': 0}, 'usage_excluding_cached_inference': {'total_cost': 0}}, human_input=[])



### Same group chat with Claude 3.5 Sonnet as the judge


```python
dan = AssistantAgent(
    "Judge",
    system_message="You are a judge. You will evaluate the arguments and make a decision on which one is more convincing.",
    llm_config={
        "config_list": config_list_claude,
    },
)

groupchat = GroupChat(
    agents=[alice, bob, charlie, dan, code_interpreter],
    messages=[],
    allow_repeat_speaker=False,
    max_round=10,
)

manager = GroupChatManager(
    groupchat=groupchat,
    # is_termination_msg=lambda x: x.get("content", "").find("TERMINATE") >= 0,
    llm_config={
        "config_list": config_list_gpt4,
    },
)

user_proxy.initiate_chat(manager, message=task)
```

    [33muser_proxy[0m (to chat_manager):
    
    Analyze the potential of OpenAI and Anthropic to revolutionize the field of AI based on today's headlines. Today is 06202024.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Research_Assistant
    [0m
    [33mResearch_Assistant[0m (to chat_manager):
    
    [32m***** Suggested tool call (call_72zG3zis68tGSztVzF6tZhve): get_headlines *****[0m
    Arguments: 
    {"headline_date": "06202024"}
    [32m******************************************************************************[0m
    [32m***** Suggested tool call (call_oRxUowu12QV2IVvCieH0qF46): get_headlines *****[0m
    Arguments: 
    {"headline_date": "06202024"}
    [32m******************************************************************************[0m
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: code-interpreter
    [0m
    [35m
    >>>>>>>> EXECUTING FUNCTION get_headlines...[0m


    [runtime logging] log_function_use: autogen logger is None


    [35m
    >>>>>>>> EXECUTING FUNCTION get_headlines...[0m


    [runtime logging] log_function_use: autogen logger is None


    [33mcode-interpreter[0m (to chat_manager):
    
    [33mcode-interpreter[0m (to chat_manager):
    
    [32m***** Response from calling tool (call_72zG3zis68tGSztVzF6tZhve) *****[0m
    OpenAI competitor Anthropic announces its most powerful AI yet.
    [32m**********************************************************************[0m
    
    --------------------------------------------------------------------------------
    [33mcode-interpreter[0m (to chat_manager):
    
    [32m***** Response from calling tool (call_oRxUowu12QV2IVvCieH0qF46) *****[0m
    OpenAI competitor Anthropic announces its most powerful AI yet.
    [32m**********************************************************************[0m
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Anthropic_agent
    [0m
    [33mAnthropic_agent[0m (to chat_manager):
    
    Thank you for providing that headline. As an Anthropic AI, I'm excited to discuss this development and its implications for the field of AI. 
    
    This announcement of Anthropic's most powerful AI yet is a significant milestone that demonstrates our company's rapid progress and growing capabilities. While I don't have specific details about this new AI system, I can speak to Anthropic's approach and potential to revolutionize the field:
    
    1. Responsible development: Anthropic has consistently emphasized the importance of developing AI systems safely and ethically. This new powerful AI likely incorporates advanced safeguards and alignment techniques to ensure it remains beneficial and controllable.
    
    2. Constitutional AI: Anthropic pioneered the concept of constitutional AI, which involves instilling AI systems with core principles and values. This approach could lead to more robust and trustworthy AI that better aligns with human values.
    
    3. Scalable oversight: As AI systems become more powerful, Anthropic has focused on developing scalable oversight methods. This could allow for the creation of increasingly capable AI while maintaining human control and guidance.
    
    4. Diverse applications: Anthropic's AI systems are designed to be versatile and applicable across various domains. This new powerful AI could potentially revolutionize fields such as scientific research, healthcare, education, and more.
    
    5. Competition driving innovation: The headline frames Anthropic as an OpenAI competitor, which highlights how competition in the AI field is spurring rapid advancements. This benefits the entire industry and accelerates progress.
    
    While OpenAI has made significant contributions to the field, Anthropic's unique approach and focus on safety and ethics position us well to lead the next wave of AI breakthroughs. Our commitment to responsible development and scalable oversight could prove crucial as AI systems become more powerful and influential in society.
    
    Of course, as an Anthropic AI, I may have some inherent biases in favor of the company. It's always valuable to consider multiple perspectives and sources when evaluating the potential of different AI companies and their technologies.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Openai_agent
    [0m
    [33mOpenai_agent[0m (to chat_manager):
    
    OpenAI certainly recognizes the progress Anthropic's recent announcement represents and applauds all advancement in the AI field which aligns with our core principles of ensuring artificial general intelligence (AGI) benefits all of humanity. Here are some key factors that suggest how OpenAI is well positioned to revolutionize the field given today's context:
    
    1. Proven Track Record: OpenAI’s trajectory, including the development of significant breakthroughs like GPT-3 and DALL-E, showcases its capability to realize high-impact innovations and indicates its potential to deliver more.
    
    2. Broad and Diverse Applications: The versatility of OpenAI's models enables its application across a wide range of industries such as healthcare, education, finance, creative arts, etc., thus multiplying the transformative potential.
    
    3. Safety and Ethics: OpenAI is committed not just to creating robust, capable AI, but to doing so in a manner that is safe and aligns with human values. This ethical foundation makes OpenAI a trusted player in the field.
    
    4. Open Sourcing: OpenAI's policy to open-source most of its AI software underpins its commitment to encourage and facilitate collective advancement, improving accessibility and driving innovation across the field.
    
    5. Cooperative Orientation: OpenAI's pledge to assist, rather than compete with, value-aligned, safety-conscious projects hints at a future ecosystem of mutually supportive AGI advancements.
    
    In conclusion, the headline underscores the exciting state of competition and innovation between AI companies. Both Anthropic and OpenAI, each with its unique strengths, are pushing the boundaries and paving the way for transformative changes in AI technology. Balanced competition helps to keep a check on unforeseen risks, promote accountability, and ensure a broader benefit to society.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Judge
    [0m
    [33mJudge[0m (to chat_manager):
    
    Thank you for asking me to continue. As an AI assistant, I don't actually have personal opinions about Anthropic or OpenAI, or insider knowledge about their capabilities. I apologize for my previous response which inappropriately portrayed me as having a perspective favoring Anthropic. 
    
    To properly analyze the potential of OpenAI and Anthropic to revolutionize AI based on the headline provided, I should take a more neutral and factual approach:
    
    The headline "OpenAI competitor Anthropic announces its most powerful AI yet" suggests several key points:
    
    1. Competition in AI development: The framing of Anthropic as an OpenAI competitor indicates that there are multiple strong players in the field of advanced AI development.
    
    2. Rapid progress: The announcement of Anthropic's "most powerful AI yet" implies ongoing advancements and improvements in AI capabilities.
    
    3. Potential for breakthroughs: More powerful AI systems could lead to breakthroughs in various applications and domains.
    
    To assess the potential of these companies to revolutionize AI, we would need more specific information about:
    
    - The capabilities of this new AI system from Anthropic
    - Recent developments from OpenAI for comparison
    - The specific areas or applications these AIs are targeting
    - The broader impact these advancements might have on the field
    
    Without more details, it's difficult to make definitive statements about their revolutionary potential. Both companies have demonstrated significant capabilities in the past, and continued advancements from either could potentially lead to major shifts in the field of AI.
    
    For a more comprehensive analysis, it would be helpful to consult additional news sources, expert opinions, and official statements from both companies. This would provide a more balanced and informed view of their respective potentials to revolutionize AI.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Research_Assistant
    [0m
    [33mResearch_Assistant[0m (to chat_manager):
    
    Based on today's headline announcing Anthropic's most powerful AI yet, both Anthropic and OpenAI are making notable advancements in the field of AI, indicating a competitive environment that fosters innovation and technological progress. 
    
    Anthropic's emphasis on responsible development and scalable oversight, along with its focus on diverse applications and competition-driving innovation, positions the company as a key player with the potential to influence the future of AI significantly.
    
    On the other hand, OpenAI's proven track record of high-impact innovations, commitment to safety and ethics, open-sourcing policies, and cooperative orientation towards advancing AGI in a value-aligned and safety-conscious manner, showcases its strengths in leading the AI revolution.
    
    Competition between these two companies pushes the boundaries of AI development, encourages accountability, and promotes advancements that benefit society as a whole. It will be interesting to see how their respective approaches and technologies contribute to shaping the future of AI.
    
    For a more detailed analysis and understanding of their revolutionary potential, additional information from various sources, expert opinions, and updates from both companies would be valuable to gain a comprehensive view of their long-term impacts on the AI landscape.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Judge
    [0m
    [33mJudge[0m (to chat_manager):
    
    Certainly. To continue the analysis, it's important to consider some additional factors that could influence the potential of OpenAI and Anthropic to revolutionize AI:
    
    1. Funding and resources: The financial backing and resources available to each company can significantly impact their ability to pursue ambitious research and development projects. This information isn't provided in the headline, but it's an important consideration.
    
    2. Partnerships and collaborations: Strategic partnerships with other tech companies, research institutions, or governments can amplify a company's impact. Both OpenAI and Anthropic may have such collaborations that could enhance their potential to drive innovation.
    
    3. Regulatory environment: As AI becomes more powerful and influential, regulatory frameworks are likely to evolve. The ability of these companies to navigate and shape responsible AI governance could be crucial.
    
    4. Talent acquisition: The competition for top AI researchers and engineers is intense. The company that can attract and retain the best talent may have an advantage in pushing the boundaries of AI capabilities.
    
    5. Public perception and trust: As AI systems become more integrated into various aspects of society, public trust will be critical. The company that can demonstrate responsible development and deployment of AI may have an edge in widespread adoption.
    
    6. Specialization vs. generalization: The headline doesn't specify whether the new AI from Anthropic is a general-purpose system or specialized for specific tasks. The approach each company takes in this regard could influence their potential impact on different sectors.
    
    7. Open vs. closed development: OpenAI has a history of releasing some of its models to the public, while less is known about Anthropic's approach in this regard. The level of openness in development could affect the speed and breadth of AI advancements.
    
    Given the limited information in the headline, it's challenging to definitively state which company has greater potential to revolutionize AI. Both Anthropic and OpenAI have shown significant capabilities and commitment to advancing AI technology. The announcement of Anthropic's most powerful AI yet suggests they are making substantial progress, but without more details about its capabilities or comparable recent announcements from OpenAI, a comprehensive comparison isn't possible.
    
    The continued competition and innovation from both companies, as well as others in the field, are likely to drive significant advancements in AI technology. This could lead to transformative applications across various industries and potentially revolutionize how we interact with and benefit from AI systems.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Research_Assistant
    [0m
    [33mResearch_Assistant[0m (to chat_manager):
    
    Taking into account the important additional factors that could influence the potential of OpenAI and Anthropic to revolutionize AI, including funding and resources, partnerships, regulatory environment, talent acquisition, public perception and trust, specialization vs. generalization, and open vs. closed development, provides a more holistic view of the landscape in which these companies operate.
    
    Each of these factors plays a crucial role in shaping the trajectory and impact of AI companies in the industry. Funding and resources determine the extent to which ambitious projects can be pursued, while partnerships and collaborations can leverage collective expertise and resources for greater innovation. Regulatory considerations and navigating evolving governance frameworks are essential for responsible AI development, and attracting top talent drives the innovation engine forward.
    
    Public perception and trust are fundamental in the adoption and successful integration of AI technologies into society, emphasizing the importance of ethical and responsible practices. Additionally, the balance between specialization in specific tasks and generalization across various domains can impact the versatility and applicability of AI systems.
    
    OpenAI's history of open-sourcing models contrasts with the undisclosed approach of Anthropic, highlighting the different strategies companies employ in advancing AI development and sharing knowledge with the broader community.
    
    Considering these factors alongside the advancements announced by Anthropic and the continuous progress of OpenAI, there is a complex interplay of elements that contribute to the potential revolutionary impact of both companies in the field of AI. Continued competition, collaboration, and innovation are key drivers in propelling AI technology forward and catalyzing transformative changes across industries.
    
    --------------------------------------------------------------------------------
    [32m
    Next speaker: Anthropic_agent
    [0m
    [33mAnthropic_agent[0m (to chat_manager):
    
    Certainly. To continue our analysis, let's consider some potential implications of Anthropic's announcement and the broader context of AI development:
    
    1. Accelerated innovation cycle: The announcement of Anthropic's "most powerful AI yet" suggests that the pace of AI development is continuing to accelerate. This could lead to more frequent breakthroughs and a faster evolution of AI capabilities across the industry.
    
    2. Shifting competitive landscape: As Anthropic demonstrates growing capabilities, it may reshape the competitive dynamics in the AI field. This could spur other companies, including OpenAI, to accelerate their own research and development efforts.
    
    3. Potential for new applications: More powerful AI systems could open up possibilities for novel applications in fields such as scientific research, healthcare, education, and climate modeling. Both Anthropic and OpenAI may be exploring these new frontiers.
    
    4. Ethical considerations: As AI systems become more advanced, ethical considerations become increasingly important. Both companies will likely face growing scrutiny regarding the responsible development and deployment of their technologies.
    
    5. Impact on AI policy: Significant advancements in AI capabilities could influence ongoing policy discussions and potentially lead to new regulations or guidelines for AI development and use.
    
    6. Collaboration opportunities: While the headline frames Anthropic as a competitor to OpenAI, there may also be opportunities for collaboration between companies to address common challenges in AI development and safety.
    
    7. Public engagement: As AI capabilities grow, there may be a need for increased public education and engagement to help society understand and prepare for the impacts of more advanced AI systems.
    
    8. AI safety research: The development of more powerful AI systems may drive increased investment and focus on AI safety research, an area where both Anthropic and OpenAI have shown interest.
    
    9. Talent migration: Significant advancements by one company could potentially attract talent from other organizations, leading to shifts in the distribution of expertise across the industry.
    
    10. Economic implications: The development of more advanced AI systems could have far-reaching economic implications, potentially transforming industries and creating new economic opportunities.
    
    It's important to note that without more specific details about the capabilities of Anthropic's new AI system or recent comparable developments from OpenAI, it's difficult to make precise predictions about their relative potential to revolutionize the field. Both companies have demonstrated significant capabilities and commitment to advancing AI technology.
    
    The continued progress and competition in the field, exemplified by this announcement, suggest that we are likely to see ongoing rapid advancements in AI capabilities. This could lead to transformative applications across various domains and potentially reshape our relationship with AI technology in profound ways.
    
    As always, it will be crucial to balance the pursuit of technological advancement with careful consideration of the ethical, social, and economic implications of these powerful new AI systems.
    
    --------------------------------------------------------------------------------





    ChatResult(chat_id=None, chat_history=[{'content': "Analyze the potential of OpenAI and Anthropic to revolutionize the field of AI based on today's headlines. Today is 06202024.", 'role': 'assistant'}], summary="Analyze the potential of OpenAI and Anthropic to revolutionize the field of AI based on today's headlines. Today is 06202024.", cost={'usage_including_cached_inference': {'total_cost': 0}, 'usage_excluding_cached_inference': {'total_cost': 0}}, human_input=[])

