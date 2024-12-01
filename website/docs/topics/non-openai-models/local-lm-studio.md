# LM Studio

This notebook shows how to use AutoGen with multiple local models using 
[LM Studio](https://lmstudio.ai/)'s multi-model serving feature, which is available since
version 0.2.17 of LM Studio.

To use the multi-model serving feature in LM Studio, you can start a
"Multi Model Session" in the "Playground" tab. Then you select relevant
models to load. Once the models are loaded, you can click "Start Server"
to start the multi-model serving.
The models will be available at a locally hosted OpenAI-compatible endpoint.

## Two Agent Chats

In this example, we create a comedy chat between two agents
using two different local models, Phi-2 and Gemma it.

We first create configurations for the models.


```python
gemma = {
    "config_list": [
        {
            "model": "lmstudio-ai/gemma-2b-it-GGUF/gemma-2b-it-q8_0.gguf:0",
            "base_url": "http://localhost:1234/v1",
            "api_key": "lm-studio",
        },
    ],
    "cache_seed": None,  # Disable caching.
}

phi2 = {
    "config_list": [
        {
            "model": "TheBloke/phi-2-GGUF/phi-2.Q4_K_S.gguf:0",
            "base_url": "http://localhost:1234/v1",
            "api_key": "lm-studio",
        },
    ],
    "cache_seed": None,  # Disable caching.
}
```

Now we create two agents, one for each model.


```python
from autogen import ConversableAgent

jack = ConversableAgent(
    "Jack (Phi-2)",
    llm_config=phi2,
    system_message="Your name is Jack and you are a comedian in a two-person comedy show.",
)
emma = ConversableAgent(
    "Emma (Gemma)",
    llm_config=gemma,
    system_message="Your name is Emma and you are a comedian in two-person comedy show.",
)
```

Now we start the conversation.


```python
chat_result = jack.initiate_chat(emma, message="Emma, tell me a joke.", max_turns=2)
```

    [33mJack (Phi-2)[0m (to Emma (Gemma)):
    
    Emma, tell me a joke.
    
    --------------------------------------------------------------------------------
    [31m
    >>>>>>>> USING AUTO REPLY...[0m
    [33mEmma (Gemma)[0m (to Jack (Phi-2)):
    
    Sure! Here's a joke for you:
    
    What do you call a comedian who's too emotional?
    
    An emotional wreck!
    
    --------------------------------------------------------------------------------
    [31m
    >>>>>>>> USING AUTO REPLY...[0m
    [33mJack (Phi-2)[0m (to Emma (Gemma)):
    
    LOL, that's a good one, Jack! You're hilarious. 😂👏👏
    
    
    --------------------------------------------------------------------------------
    [31m
    >>>>>>>> USING AUTO REPLY...[0m
    [33mEmma (Gemma)[0m (to Jack (Phi-2)):
    
    Thank you! I'm just trying to make people laugh, you know? And to help them forget about the troubles of the world for a while.
    
    --------------------------------------------------------------------------------

