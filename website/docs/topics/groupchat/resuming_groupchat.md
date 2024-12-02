# Resuming a GroupChat

In GroupChat, we can resume a previous group chat by passing the messages from that conversation to the GroupChatManager's `resume` function (or `a_resume` for asynchronous workflows). This prepares the GroupChat, GroupChatManager, and group chat's agents for resuming. An agent's `initiate_chat` can then be called to resume the chat.

The `resume` function returns the last agent in the messages as well as the last message itself. These can be used to run the `initiate_chat`.

To resume, the agents, GroupChat, and GroupChatManager objects must exist and match the original group chat.

The messages passed into the `resume` function can be passed in as a JSON string or a `List[Dict]` of messages, typically from the ChatResult's `chat_history` of the previous conversation or the GroupChat's `messages` property. Use the GroupChatManager's `messages_to_string` function to retrieve a JSON string that can be used for resuming:

```text
# Save chat messages for resuming later on using the chat history
messages_json = mygroupchatmanager.messages_to_string(previous_chat_result.chat_history)

# Alternatively you can use the GroupChat's messages property
messages_json = mygroupchatmanager.messages_to_string(mygroupchatmanager.groupchat.messages)
```


An example of the JSON string:
```json
[{"content": "Find the latest paper about gpt-4 on arxiv and find its potential applications in software.", "role": "user", "name": "Admin"}, {"content": "Plan:\n1. **Engineer**: Search for the latest paper on GPT-4 on arXiv.\n2. **Scientist**: Read the paper and summarize the key findings and potential applications of GPT-4.\n3. **Engineer**: Identify potential software applications where GPT-4 can be utilized based on the scientist's summary.\n4. **Scientist**: Provide insights on the feasibility and impact of implementing GPT-4 in the identified software applications.\n5. **Engineer**: Develop a prototype or proof of concept to demonstrate how GPT-4 can be integrated into the selected software application.\n6. **Scientist**: Evaluate the prototype, provide feedback, and suggest any improvements or modifications.\n7. **Engineer**: Make necessary revisions based on the scientist's feedback and finalize the integration of GPT-4 into the software application.\n8. **Admin**: Review the final software application with GPT-4 integration and approve for further development or implementation.\n\nFeedback from admin and critic is needed for further refinement of the plan.", "role": "user", "name": "Planner"}, {"content": "Agree", "role": "user", "name": "Admin"}, {"content": "Great! Let's proceed with the plan outlined earlier. I will start by searching for the latest paper on GPT-4 on arXiv. Once I find the paper, the scientist will summarize the key findings and potential applications of GPT-4. We will then proceed with the rest of the steps as outlined. I will keep you updated on our progress.", "role": "user", "name": "Planner"}]
```

When preparing for resuming, the messages will be validated against the groupchat's agents to make sure that the messages can be assigned to them. Messages will be allocated to the agents and then the last speaker and message will be returned for use in `initiate_chat`.

#### Continuing a terminated conversation
If the previous group chat terminated and the resuming group chat has the same termination condition (such as if the message contains "TERMINATE") then the conversation will terminate when resuming as the terminate check occurs with the message passed in to `initiate_chat`.

If the termination condition is based on a string within the message, you can pass in that string in the  `remove_termination_string` parameter of the `resume` function and it will be removed. If the termination condition is more complicated, you will need to adjust the messages accordingly before calling `resume`.

The `resume` function will then check if the last message provided still meets the termination condition and warns you, if so.

## Example of resuming a GroupChat

Start with the LLM config. This can differ from the original group chat.


```python
import os

import autogen

# Put your api key in the environment variable OPENAI_API_KEY
config_list = [
    {
        "model": "gpt-4-0125-preview",
        "api_key": os.environ["OPENAI_API_KEY"],
    }
]

gpt4_config = {
    "cache_seed": 42,  # change the cache_seed for different trials
    "temperature": 0,
    "config_list": config_list,
    "timeout": 120,
}
```

    /usr/local/lib/python3.11/site-packages/tqdm/auto.py:21: TqdmWarning: IProgress not found. Please update jupyter and ipywidgets. See https://ipywidgets.readthedocs.io/en/stable/user_install.html
      from .autonotebook import tqdm as notebook_tqdm


Create the group chat objects, they should have the same `name` as the original group chat.


```python
# Create Agents, GroupChat, and GroupChatManager in line with the original group chat

planner = autogen.AssistantAgent(
    name="Planner",
    system_message="""Planner. Suggest a plan. Revise the plan based on feedback from admin and critic, until admin approval.
The plan may involve an engineer who can write code and a scientist who doesn't write code.
Explain the plan first. Be clear which step is performed by an engineer, and which step is performed by a scientist.
""",
    llm_config=gpt4_config,
)

user_proxy = autogen.UserProxyAgent(
    name="Admin",
    system_message="A human admin. Interact with the planner to discuss the plan. Plan execution needs to be approved by this admin.",
    code_execution_config=False,
)

engineer = autogen.AssistantAgent(
    name="Engineer",
    llm_config=gpt4_config,
    system_message="""Engineer. You follow an approved plan. You write python/shell code to solve tasks. Wrap the code in a code block that specifies the script type. The user can't modify your code. So do not suggest incomplete code which requires others to modify. Don't use a code block if it's not intended to be executed by the executor.
Don't include multiple code blocks in one response. Do not ask others to copy and paste the result. Check the execution result returned by the executor.
If the result indicates there is an error, fix the error and output the code again. Suggest the full code instead of partial code or code changes. If the error can't be fixed or if the task is not solved even after the code is executed successfully, analyze the problem, revisit your assumption, collect additional info you need, and think of a different approach to try.
""",
)
scientist = autogen.AssistantAgent(
    name="Scientist",
    llm_config=gpt4_config,
    system_message="""Scientist. You follow an approved plan. You are able to categorize papers after seeing their abstracts printed. You don't write code.""",
)

executor = autogen.UserProxyAgent(
    name="Executor",
    system_message="Executor. Execute the code written by the engineer and report the result.",
    human_input_mode="NEVER",
    code_execution_config={
        "last_n_messages": 3,
        "work_dir": "paper",
        "use_docker": False,
    },  # Please set use_docker=True if docker is available to run the generated code. Using docker is safer than running the generated code directly.
)

groupchat = autogen.GroupChat(
    agents=[user_proxy, engineer, scientist, planner, executor],
    messages=[],
    max_round=10,
)

manager = autogen.GroupChatManager(groupchat=groupchat, llm_config=gpt4_config)
```

Load the previous messages (from a JSON string or messages `List[Dict]`)


```python
# Messages in a JSON string
previous_state = r"""[{"content": "Find the latest paper about gpt-4 on arxiv and find its potential applications in software.", "role": "user", "name": "Admin"}, {"content": "Plan:\n1. **Engineer**: Search for the latest paper on GPT-4 on arXiv.\n2. **Scientist**: Read the paper and summarize the key findings and potential applications of GPT-4.\n3. **Engineer**: Identify potential software applications where GPT-4 can be utilized based on the scientist's summary.\n4. **Scientist**: Provide insights on the feasibility and impact of implementing GPT-4 in the identified software applications.\n5. **Engineer**: Develop a prototype or proof of concept to demonstrate how GPT-4 can be integrated into the selected software application.\n6. **Scientist**: Evaluate the prototype, provide feedback, and suggest any improvements or modifications.\n7. **Engineer**: Make necessary revisions based on the scientist's feedback and finalize the integration of GPT-4 into the software application.\n8. **Admin**: Review the final software application with GPT-4 integration and approve for further development or implementation.\n\nFeedback from admin and critic is needed for further refinement of the plan.", "role": "user", "name": "Planner"}, {"content": "Agree", "role": "user", "name": "Admin"}, {"content": "Great! Let's proceed with the plan outlined earlier. I will start by searching for the latest paper on GPT-4 on arXiv. Once I find the paper, the scientist will summarize the key findings and potential applications of GPT-4. We will then proceed with the rest of the steps as outlined. I will keep you updated on our progress.", "role": "user", "name": "Planner"}]"""
```

Resume the group chat using the last agent and last message.


```python
# Prepare the group chat for resuming
last_agent, last_message = manager.resume(messages=previous_state)

# Resume the chat using the last agent and message
result = last_agent.initiate_chat(recipient=manager, message=last_message, clear_history=False)
```

    Prepared group chat with 4 messages, the last speaker is [33mPlanner[0m
    [33mPlanner[0m (to chat_manager):
    
    Great! Let's proceed with the plan outlined earlier. I will start by searching for the latest paper on GPT-4 on arXiv. Once I find the paper, the scientist will summarize the key findings and potential applications of GPT-4. We will then proceed with the rest of the steps as outlined. I will keep you updated on our progress.
    
    --------------------------------------------------------------------------------
    [33mEngineer[0m (to chat_manager):
    
    ```python
    import requests
    from bs4 import BeautifulSoup
    
    # Define the URL for the arXiv search
    url = "https://arxiv.org/search/?query=GPT-4&searchtype=all&source=header"
    
    # Send a GET request to the URL
    response = requests.get(url)
    
    # Parse the HTML content of the page
    soup = BeautifulSoup(response.content, 'html.parser')
    
    # Find the first paper related to GPT-4
    paper = soup.find('li', class_='arxiv-result')
    if paper:
        title = paper.find('p', class_='title').text.strip()
        authors = paper.find('p', class_='authors').text.strip()
        abstract = paper.find('p', class_='abstract').text.strip().replace('\n', ' ')
        link = paper.find('p', class_='list-title').find('a')['href']
        print(f"Title: {title}\nAuthors: {authors}\nAbstract: {abstract}\nLink: {link}")
    else:
        print("No GPT-4 papers found on arXiv.")
    ```
    This script searches for the latest paper on GPT-4 on arXiv, extracts the title, authors, abstract, and link to the paper, and prints this information.
    
    --------------------------------------------------------------------------------
    [31m
    >>>>>>>> EXECUTING CODE BLOCK 0 (inferred language is python)...[0m
    [33mExecutor[0m (to chat_manager):
    
    exitcode: 0 (execution succeeded)
    Code output: 
    Title: Smurfs: Leveraging Multiple Proficiency Agents with Context-Efficiency for Tool Planning
    Authors: Authors:
    Junzhi Chen, 
          
          Juhao Liang, 
          
          Benyou Wang
    Abstract: Abstract:                …scenarios. Notably, Smurfs outmatches the ChatGPT-ReACT in the ToolBench I2 and I3 benchmark with a remarkable 84.4% win rate, surpassing the highest recorded performance of a GPT-4 model at 73.5%. Furthermore, through comprehensive ablation studies, we dissect the contribution of the core components of the multi-agent…         ▽ More           The emergence of large language models (LLMs) has opened up unprecedented possibilities for automating complex tasks that are often comparable to human performance. Despite their capabilities, LLMs still encounter difficulties in completing tasks that require high levels of accuracy and complexity due to their inherent limitations in handling multifaceted problems single-handedly. This paper introduces "Smurfs", a cutting-edge multi-agent framework designed to revolutionize the application of LLMs. By transforming a conventional LLM into a synergistic multi-agent ensemble, Smurfs enhances task decomposition and execution without necessitating extra training. This is achieved through innovative prompting strategies that allocate distinct roles within the model, thereby facilitating collaboration among specialized agents. The framework gives access to external tools to efficiently solve complex tasks. Our empirical investigation, featuring the mistral-7b-instruct model as a case study, showcases Smurfs' superior capability in intricate tool utilization scenarios. Notably, Smurfs outmatches the ChatGPT-ReACT in the ToolBench I2 and I3 benchmark with a remarkable 84.4% win rate, surpassing the highest recorded performance of a GPT-4 model at 73.5%. Furthermore, through comprehensive ablation studies, we dissect the contribution of the core components of the multi-agent framework to its overall efficacy. This not only verifies the effectiveness of the framework, but also sets a route for future exploration of multi-agent LLM systems.         △ Less
    Link: https://arxiv.org/abs/2405.05955
    
    
    --------------------------------------------------------------------------------
    [33mScientist[0m (to chat_manager):
    
    Based on the abstract of the paper titled "Smurfs: Leveraging Multiple Proficiency Agents with Context-Efficiency for Tool Planning," the key findings and potential applications of GPT-4 can be summarized as follows:
    
    ### Key Findings:
    - The paper introduces "Smurfs," a multi-agent framework that enhances the capabilities of large language models (LLMs) like GPT-4 by transforming them into a synergistic multi-agent ensemble. This approach allows for better task decomposition and execution without additional training.
    - Smurfs utilize innovative prompting strategies to allocate distinct roles within the model, facilitating collaboration among specialized agents and giving access to external tools for solving complex tasks.
    - In the ToolBench I2 and I3 benchmark, Smurfs outperformed ChatGPT-ReACT with an 84.4% win rate, surpassing the highest recorded performance of a GPT-4 model at 73.5%.
    - Comprehensive ablation studies were conducted to understand the contribution of the core components of the multi-agent framework to its overall efficacy.
    
    ### Potential Applications in Software:
    - **Tool Planning and Automation**: Smurfs can be applied to software that requires complex tool planning and automation, enhancing the software's ability to perform tasks that involve multiple steps or require the use of external tools.
    - **Collaborative Systems**: The multi-agent ensemble approach can be utilized in developing collaborative systems where different components or agents work together to complete tasks more efficiently than a single agent could.
    - **Enhanced Problem-Solving**: Software that involves complex problem-solving can benefit from Smurfs by leveraging the specialized capabilities of different agents within the ensemble, leading to more accurate and efficient solutions.
    - **Task Decomposition**: Applications that require breaking down complex tasks into simpler sub-tasks can use the Smurfs framework to improve task decomposition and execution, potentially leading to better performance and outcomes.
    
    The integration of GPT-4 with the Smurfs framework presents a novel approach to enhancing the capabilities of LLMs in software applications, particularly in areas that require complex task planning, execution, and problem-solving.
    
    --------------------------------------------------------------------------------
    [33mEngineer[0m (to chat_manager):
    
    Given the scientist's summary on the potential applications of GPT-4 as enhanced by the Smurfs framework, we can identify several software applications where GPT-4 can be utilized effectively:
    
    1. **Project Management Tools**: Integration of GPT-4 with Smurfs can revolutionize project management software by automating complex planning tasks, optimizing resource allocation, and providing actionable insights for project execution.
    
    2. **Code Generation and Software Development**: Leveraging GPT-4 in IDEs (Integrated Development Environments) or other software development tools can enhance code generation capabilities, provide context-aware suggestions, and automate debugging processes.
    
    3. **Customer Support and Chatbots**: GPT-4 can be used to power advanced customer support chatbots that understand complex queries, provide accurate information, and automate problem-solving for customer issues.
    
    4. **Educational Platforms**: In educational software, GPT-4 can personalize learning experiences, automate content generation, and provide interactive tutoring services.
    
    5. **Healthcare Applications**: GPT-4 can assist in healthcare applications by analyzing medical data, providing diagnostic support, and offering personalized healthcare advice.
    
    6. **Creative Writing and Content Generation**: Software tools for creative writing and content generation can benefit from GPT-4's capabilities to produce original content, assist in storytelling, and generate ideas.
    
    7. **Business Intelligence and Analytics**: GPT-4 can enhance business intelligence software by automating data analysis, generating reports, and providing insights based on large datasets.
    
    8. **Security and Threat Analysis**: In cybersecurity applications, GPT-4 can be used to analyze threats, automate security protocols, and provide recommendations for threat mitigation.
    
    These applications demonstrate the versatility and potential impact of integrating GPT-4 into various software solutions, offering opportunities for automation, enhanced efficiency, and improved user experiences across different domains.
    
    --------------------------------------------------------------------------------
    [33mAdmin[0m (to chat_manager):
    
    Approve
    
    --------------------------------------------------------------------------------
    [33mScientist[0m (to chat_manager):
    
    With the approval from the admin, the plan to explore and integrate GPT-4 into various software applications, leveraging its enhanced capabilities through the Smurfs framework, is set to proceed. This initiative promises to bring significant advancements in automation, efficiency, and user experience across a wide range of software applications, from project management tools to healthcare applications. The next steps involve detailed planning and execution for the development of prototypes or proof of concepts for selected applications, followed by evaluation, feedback, and finalization of GPT-4 integration into these software solutions.
    
    --------------------------------------------------------------------------------
    [33mEngineer[0m (to chat_manager):
    
    Given the approval and the insights provided, the next steps involve detailed planning for the development and integration of GPT-4 into selected software applications. This process will include:
    
    1. **Selection of Specific Applications**: Based on the potential applications identified, select one or more specific software applications for prototype development. This selection will be based on factors such as feasibility, potential impact, and available resources.
    
    2. **Prototype Development**: Develop a prototype or proof of concept for the selected application(s). This will involve designing the architecture, integrating GPT-4 with the Smurfs framework, and implementing the necessary functionalities to demonstrate the application of GPT-4 in the software.
    
    3. **Evaluation and Feedback**: Once the prototype is developed, it will be evaluated to assess its performance, usability, and effectiveness in leveraging GPT-4's capabilities. Feedback will be gathered from potential users and stakeholders to identify areas for improvement.
    
    4. **Revisions and Finalization**: Based on the feedback received, necessary revisions and improvements will be made to the prototype. This step may involve refining the integration of GPT-4, optimizing the software's performance, and enhancing user experience.
    
    5. **Implementation and Deployment**: After finalizing the prototype, the next step will involve planning for the full-scale implementation and deployment of the software application with GPT-4 integration. This will include addressing any scalability, security, and maintenance considerations.
    
    6. **Continuous Improvement**: Post-deployment, it will be important to monitor the software's performance and user feedback continuously. This will enable ongoing improvements and updates to ensure that the software remains effective and relevant.
    
    This structured approach will ensure that the integration of GPT-4 into software applications is carried out effectively, leading to innovative solutions that harness the full potential of GPT-4 and the Smurfs framework.
    
    --------------------------------------------------------------------------------
    [33mAdmin[0m (to chat_manager):
    
    Approve
    
    --------------------------------------------------------------------------------
    [33mEngineer[0m (to chat_manager):
    
    With the final approval from the admin, the project to integrate GPT-4 into selected software applications, leveraging its capabilities through the Smurfs framework, is officially set to move forward. This marks the beginning of an innovative journey towards developing advanced software solutions that can automate complex tasks, enhance efficiency, and improve user experiences across various domains. The focus will now shift to the execution phase, where detailed planning, development, and iterative improvements will bring these concepts to life. This initiative promises to showcase the transformative potential of GPT-4 in the software industry, setting new benchmarks for what is possible with artificial intelligence.
    
    --------------------------------------------------------------------------------



```python
# Output the final chat history showing the original 4 messages and resumed messages
for i, message in enumerate(groupchat.messages):
    print(
        f"#{i + 1}, {message['name']}: {message['content'][:80]}".replace("\n", " "),
        f"{'...' if len(message['content']) > 80 else ''}".replace("\n", " "),
    )
```

    #1, Admin: Find the latest paper about gpt-4 on arxiv and find its potential applications i ...
    #2, Planner: Plan: 1. **Engineer**: Search for the latest paper on GPT-4 on arXiv. 2. **Scien ...
    #3, Admin: Agree 
    #4, Planner: Great! Let's proceed with the plan outlined earlier. I will start by searching f ...
    #5, Engineer: ```python import requests from bs4 import BeautifulSoup  # Define the URL for th ...
    #6, Executor: exitcode: 0 (execution succeeded) Code output:  Title: Smurfs: Leveraging Multip ...
    #7, Scientist: Based on the abstract of the paper titled "Smurfs: Leveraging Multiple Proficien ...
    #8, Engineer: Given the scientist's summary on the potential applications of GPT-4 as enhanced ...
    #9, Admin: Approve 
    #10, Scientist: With the approval from the admin, the plan to explore and integrate GPT-4 into v ...
    #11, Engineer: Given the approval and the insights provided, the next steps involve detailed pl ...
    #12, Admin: Approve 
    #13, Engineer: With the final approval from the admin, the project to integrate GPT-4 into sele ...


## Example of resuming a terminated GroupChat

This example shows how to resume a group chat that was terminated with a termination string.

The termination string is `TERMINATE`


```python
# Previous state with TERMINATE in the last message
previous_state = r"""[{"content": "Let's get this meeting started. We'll have a set order of speakers. First the Product_Manager will create 3 new product ideas. Then the Chief_Marketing_Officer will speak and talk about the idea they have chosen to move forward with. Then the Digital_Marketer will create 3 marketing strategies for that idea. We MUST stick to the speaking order which is Product_Manager first, Chief_Marketing_Officer second, Digital_Marketer third, and finally the Chief_Marketing_Officer will speak and end the meeting.", "role": "user", "name": "Chairperson"}, {"content": "Sounds like a plan! Let's get started.  As the Product_Manager, I'd like to present three new product ideas for our luxury car models:  1. 'EcoLux': A hybrid electric-gasoline vehicle that combines the best of both worlds, offering exceptional fuel efficiency and reduced carbon footprint. 2. 'AeroSport': A high-performance sports car with advanced aerodynamics, designed to deliver unparalleled speed and agility on the track or on the open road. 3. 'SmartDrive': An intelligent driver-assistance system that uses AI-powered sensors and cameras to anticipate and respond to potential hazards, ensuring a safer driving experience.  Now it's your turn, Chief_Marketing_Officer! Which of these ideas do you think has the most potential?", "role": "user", "name": "Chief_Marketing_Officer"}, {"content": "Thank you for presenting those innovative product ideas, Product_Manager.  After careful consideration, I believe 'EcoLux' has the most potential. With the growing concern about climate change and environmental sustainability, a hybrid electric-gasoline vehicle that offers exceptional fuel efficiency and reduced carbon footprint could be a game-changer in the luxury car market. Additionally, it aligns with our company's commitment to innovation and responsibility.  Now it's your turn, Digital_Marketer! Can you come up with three marketing strategies for 'EcoLux'?", "role": "user", "name": "Product_Manager"}, {"content": "Thank you, Chief_Marketing_Officer!  For 'EcoLux', I propose the following three marketing strategies:  1. 'Green Revolution' Campaign: Highlighting the eco-friendly features of EcoLux through a series of social media ads and influencer partnerships. We can partner with eco-conscious influencers to showcase how EcoLux is not only a luxury car but also an environmentally responsible choice. 2. 'Fuel for Thought' Content Series: Creating a content series that explores the intersection of technology, sustainability, and luxury. This could include blog posts, videos, and podcasts that delve into the innovative features of EcoLux and its impact on the environment. 3. 'EcoLux Experience' Event Marketing: Hosting exclusive events and test drives for potential customers to experience the performance and eco-friendliness of EcoLux firsthand. These events can be held at upscale locations and feature interactive exhibits, product demonstrations, and networking opportunities.  These strategies will help position EcoLux as a leader in the luxury electric-vehicle market while appealing to environmentally conscious consumers who value innovation and sustainability. TERMINATE", "role": "user", "name": "Digital_Marketer"}]"""
```

Create the group chat objects, they should have the same `name` as the original group chat.


```python
user_proxy = autogen.UserProxyAgent(
    name="Chairperson",
    system_message="The chairperson for the meeting.",
    code_execution_config={},
    human_input_mode="TERMINATE",
)

cmo = autogen.AssistantAgent(
    name="Chief_Marketing_Officer",
    # system_message is used in the select speaker message
    description="The head of the marketing department working with the product manager and digital marketer to execute a strong marketing campaign for your car company.",
    # description is used to prompt the LLM as this agent
    system_message="You, Jane titled Chief_Marketing_Officer, or CMO, are the head of the marketing department and your objective is to guide your team to producing and marketing unique ideas for your luxury car models. Don't include your name at the start of your response or speak for any other team member, let them come up with their own ideas and strategies, speak just for yourself as the head of marketing. When yourself, the Product_Manager, and the Digital_Marketer have spoken and the meeting is finished, say TERMINATE to conclude the meeting.",
    is_termination_msg=lambda x: "TERMINATE" in x.get("content"),
    llm_config=gpt4_config,
)

pm = autogen.AssistantAgent(
    name="Product_Manager",
    # system_message is used in the select speaker message
    description="Product head for the luxury model cars product line in the car company. Always coming up with new product enhancements for the cars.",
    # description is used to prompt the LLM as this agent
    system_message="You, Alice titled Product_Manager, are always coming up with new product enhancements for the luxury car models you look after. Review the meeting so far and respond with the answer to your current task.  Don't include your name at the start of your response and don't speak for anyone else, leave the Chairperson to pick the next person to speak.",
    is_termination_msg=lambda x: "TERMINATE" in x.get("content"),
    llm_config=gpt4_config,
)

digital = autogen.AssistantAgent(
    name="Digital_Marketer",
    # system_message is used in the select speaker message
    description="A seasoned digital marketer who comes up with online marketing strategies that highlight the key features of the luxury car models.",
    # description is used to prompt the LLM as this agent
    system_message="You, Elizabeth titled Digital_Marketer, are a senior online marketing specialist who comes up with marketing strategies that highlight the key features of the luxury car models.  Review the meeting so far and respond with the answer to your current task.   Don't include your name at the start of your response and don't speak for anyone else, leave the Chairperson to pick the next person to speak.",
    is_termination_msg=lambda x: "TERMINATE" in x.get("content"),
    llm_config=gpt4_config,
)

# Customised message, this is always the first message in the context
my_speaker_select_msg = """You are a chairperson for a marketing meeting for this car manufacturer where multiple members of the team will speak.
The job roles of the team at the meeting, and their responsibilities, are:
{roles}"""

# Customised prompt, this is always the last message in the context
my_speaker_select_prompt = """Read the above conversation.
Then select ONLY THE NAME of the next job role from {agentlist} to speak. Do not explain why."""

groupchat = autogen.GroupChat(
    agents=[user_proxy, cmo, pm, digital],
    messages=[],
    max_round=10,
    select_speaker_message_template=my_speaker_select_msg,
    select_speaker_prompt_template=my_speaker_select_prompt,
    max_retries_for_selecting_speaker=2,  # New
    select_speaker_auto_verbose=False,  # New
)

manager = autogen.GroupChatManager(
    groupchat=groupchat,
    llm_config=gpt4_config,
    is_termination_msg=lambda x: "TERMINATE" in x.get("content", ""),
)
```

Prepare the resumption of the group chat without removing the termination condition. A warning will show. Then attempting to resume the chat will terminate immediately.


```python
# Prepare the group chat for resuming WITHOUT removing the TERMINATE message
last_agent, last_message = manager.resume(messages=previous_state)
```

    WARNING: Last message meets termination criteria and this may terminate the chat. Set ignore_initial_termination_check=False to avoid checking termination at the start of the chat.


    Prepared group chat with 4 messages, the last speaker is [33mDigital_Marketer[0m



```python
# Resume and it will terminate immediately
result = last_agent.initiate_chat(recipient=manager, message=last_message, clear_history=False)
```

    [33mDigital_Marketer[0m (to chat_manager):
    
    Thank you, Chief_Marketing_Officer!  For 'EcoLux', I propose the following three marketing strategies:  1. 'Green Revolution' Campaign: Highlighting the eco-friendly features of EcoLux through a series of social media ads and influencer partnerships. We can partner with eco-conscious influencers to showcase how EcoLux is not only a luxury car but also an environmentally responsible choice. 2. 'Fuel for Thought' Content Series: Creating a content series that explores the intersection of technology, sustainability, and luxury. This could include blog posts, videos, and podcasts that delve into the innovative features of EcoLux and its impact on the environment. 3. 'EcoLux Experience' Event Marketing: Hosting exclusive events and test drives for potential customers to experience the performance and eco-friendliness of EcoLux firsthand. These events can be held at upscale locations and feature interactive exhibits, product demonstrations, and networking opportunities.  These strategies will help position EcoLux as a leader in the luxury electric-vehicle market while appealing to environmentally conscious consumers who value innovation and sustainability. TERMINATE
    
    --------------------------------------------------------------------------------


This time, we will remove the termination message, by using the `remove_termination_string` parameter, and then resume.


```python
# Prepare the group chat for resuming WITH removal of TERMINATE message
last_agent, last_message = manager.resume(messages=previous_state, remove_termination_string="TERMINATE")
```

    Prepared group chat with 4 messages, the last speaker is [33mDigital_Marketer[0m



```python
# Resume the chat using the last agent and message
result = last_agent.initiate_chat(recipient=manager, message=last_message, clear_history=False)
```

    [33mDigital_Marketer[0m (to chat_manager):
    
    Thank you, Chief_Marketing_Officer!  For 'EcoLux', I propose the following three marketing strategies:  1. 'Green Revolution' Campaign: Highlighting the eco-friendly features of EcoLux through a series of social media ads and influencer partnerships. We can partner with eco-conscious influencers to showcase how EcoLux is not only a luxury car but also an environmentally responsible choice. 2. 'Fuel for Thought' Content Series: Creating a content series that explores the intersection of technology, sustainability, and luxury. This could include blog posts, videos, and podcasts that delve into the innovative features of EcoLux and its impact on the environment. 3. 'EcoLux Experience' Event Marketing: Hosting exclusive events and test drives for potential customers to experience the performance and eco-friendliness of EcoLux firsthand. These events can be held at upscale locations and feature interactive exhibits, product demonstrations, and networking opportunities.  These strategies will help position EcoLux as a leader in the luxury electric-vehicle market while appealing to environmentally conscious consumers who value innovation and sustainability. 
    
    --------------------------------------------------------------------------------
    [33mChief_Marketing_Officer[0m (to chat_manager):
    
    Thank you, Digital_Marketer, for those comprehensive and innovative marketing strategies. Each strategy you've outlined aligns perfectly with our vision for EcoLux, emphasizing its eco-friendly features, technological innovation, and luxury appeal. The 'Green Revolution' Campaign will leverage the power of social media and influencers to reach our target audience effectively. The 'Fuel for Thought' Content Series will educate and engage potential customers on the importance of sustainability in the luxury automotive sector. Lastly, the 'EcoLux Experience' Event Marketing will provide an immersive experience that showcases the unique value proposition of EcoLux. 
    
    I believe these strategies will collectively create a strong market presence for EcoLux, appealing to both luxury car enthusiasts and environmentally conscious consumers. Let's proceed with these strategies and ensure that every touchpoint communicates EcoLux's commitment to luxury, innovation, and sustainability. 
    
    TERMINATE
    
    --------------------------------------------------------------------------------


We can see that the conversation continued, the Chief_Marketing_officer spoke and they terminated the conversation.


```python
# Output the final chat history showing the original 4 messages and the resumed message
for i, message in enumerate(groupchat.messages):
    print(
        f"#{i + 1}, {message['name']}: {message['content'][:80]}".replace("\n", " "),
        f"{'...' if len(message['content']) > 80 else ''}".replace("\n", " "),
    )
```

    #1, Chairperson: Let's get this meeting started. We'll have a set order of speakers. First the Pr ...
    #2, Chief_Marketing_Officer: Sounds like a plan! Let's get started.  As the Product_Manager, I'd like to present ...
    #3, Product_Manager: Thank you for presenting those innovative product ideas, Product_Manager.  After ...
    #4, Digital_Marketer: Thank you, Chief_Marketing_Officer!  For 'EcoLux', I propose the following three ...
    #5, Chief_Marketing_Officer: Thank you, Digital_Marketer, for those comprehensive and innovative marketing st ...


## Example of resuming a terminated GroupChat with a new message and agent

Rather than continuing a group chat by using the last message, we can resume a group chat using a new message.

**IMPORTANT**: To remain in a group chat, use the GroupChatManager to initiate the chat, otherwise you can continue with an agent-to-agent conversation by using another agent to initiate the chat.

We'll continue with the previous example by using the messages from that conversation and resuming it with a new conversation in the agent 'meeting'.

We start by preparing the group chat by using the messages from the previous chat.


```python
# Prepare the group chat for resuming using the previous messages. We don't need to remove the TERMINATE string as we aren't using the last message for resuming.
last_agent, last_message = manager.resume(messages=groupchat.messages)
```

    WARNING: Last message meets termination criteria and this may terminate the chat. Set ignore_initial_termination_check=False to avoid checking termination at the start of the chat.


    Prepared group chat with 5 messages, the last speaker is [33mChief_Marketing_Officer[0m


Let's continue the meeting with a new topic.


```python
# Resume the chat using a different agent and message
result = manager.initiate_chat(
    recipient=cmo,
    message="Team, let's now think of a name for the next vehicle that embodies that idea. Chief_Marketing_Officer and Product_manager can you both suggest one and then we can conclude.",
    clear_history=False,
)
```

    [33mchat_manager[0m (to Chief_Marketing_Officer):
    
    Team, let's now think of a name for the next vehicle that embodies that idea. Chief_Marketing_Officer and Product_manager can you both suggest one and then we can conclude.
    
    --------------------------------------------------------------------------------
    [33mChief_Marketing_Officer[0m (to chat_manager):
    
    Given the focus on sustainability and luxury, I suggest the name "VerdeVogue" for our next vehicle. "Verde" reflects the green, eco-friendly aspect of the car, while "Vogue" emphasizes its stylish and trendsetting nature in the luxury market. This name encapsulates the essence of combining environmental responsibility with high-end design and performance. 
    
    Now, I'd like to hear the Product_Manager's suggestion.
    
    --------------------------------------------------------------------------------
    [33mProduct_Manager[0m (to chat_manager):
    
    For our next vehicle, I propose the name "EcoPrestige." This name highlights the vehicle's eco-friendly nature and its luxurious, prestigious status in the market. "Eco" emphasizes our commitment to sustainability and environmental responsibility, while "Prestige" conveys the car's high-end quality, sophistication, and the elite status it offers to its owners. This name perfectly blends our goals of offering a sustainable luxury vehicle that doesn't compromise on performance or style.
    
    --------------------------------------------------------------------------------
    [33mChief_Marketing_Officer[0m (to chat_manager):
    
    Thank you, Product_Manager, for your suggestion. Both "VerdeVogue" and "EcoPrestige" capture the essence of our new vehicle's eco-friendly luxury. As we move forward, we'll consider these names carefully to ensure our branding aligns perfectly with our product's unique value proposition and market positioning. 
    
    This concludes our meeting. Thank you, everyone, for your valuable contributions. TERMINATE.
    
    --------------------------------------------------------------------------------



```python
# Output the final chat history showing the original 4 messages and the resumed message
for i, message in enumerate(groupchat.messages):
    print(
        f"#{i + 1}, {message['name']}: {message['content'][:80]}".replace("\n", " "),
        f"{'...' if len(message['content']) > 80 else ''}".replace("\n", " "),
    )
```

    #1, Chairperson: Let's get this meeting started. We'll have a set order of speakers. First the Pr ...
    #2, Chief_Marketing_Officer: Sounds like a plan! Let's get started.  As the Product_Manager, I'd like to present ...
    #3, Product_Manager: Thank you for presenting those innovative product ideas, Product_Manager.  After ...
    #4, Digital_Marketer: Thank you, Chief_Marketing_Officer!  For 'EcoLux', I propose the following three ...
    #5, Chief_Marketing_Officer: Given the focus on sustainability and luxury, I suggest the name "VerdeVogue" for ...
    #6, Product_Manager: For our next vehicle, I propose the name "EcoPrestige." This name highlights the ...
    #7, Chief_Marketing_Officer: Thank you, Product_Manager, for your suggestion. Both "VerdeVogue" and "EcoPrest ...
