# BMTools

BMTools is an open-source repository that extends language models using tools and serves as a platform for the community to build and share tools. In this repository, you can (1) easily build a plugin by writing python functions (2) use external Chatgpt-Plugins. 

This project is inspired by the open-source project [LangChain](https://github.com/hwchase17/langchain/) and optimized for the usage of open-sourced tools like [ChatGPT-Plugins](https://openai.com/blog/chatgpt-plugins), striving to achieve the open-source academic version of ChatGPT-plugin.

Current version of BMTools is far from perfect, we will continue to improve it. 


## 1. Setup

```
git clone git@github.com:OpenBMB/BMTools.git
python setup.py develop
```

## 2. Use existing tools

### 2.1 Set up tools

#### 2.1.1 Local tools
Add your api keys to secret_keys.sh, then start the local tools
```
source secret_keys.sh
python host_local_tools.py
```
Then use set the url of plugin to `http://127.0.0.1:8079/tools/{tool_name}/` (Remember the tailing `/`).

#### 2.1.2 Use online ChatGPT-plugin

Just loaded it with the URL pointed to the `.well-known/ai-plugin.json`
For example, 
Set the url to `https://www.klarna.com/`, where `https://www.klarna.com/.well-known/ai-plugin.json` is a valid configuration. 

### 2.2 Use a single tool 

```
from bmtools.agent.singletool import load_single_tools, STQuestionAnswerer

tool_name, tool_url = 'klarna',  'https://www.klarna.com/'
tool_name, tool_config = load_single_tools(tool_name, tool_url)
print(tool_name, tool_config)
stqa =  STQuestionAnswerer()

agent = stqa.load_tools(tool_name, tool_config)
agent("{Your Question}")
```

### 2.3 Use multiple tools
We can use multiple tools at the same time. Basically, the language model will do it recursively. It will treat the whole tool as an API, send questions to it, and the tool calls its sub-APIs to solve the question and send it back to parent tools. This functionality will be useful in the upcoming Chat mode.

Try this functionality using scripts like:

```
from bmtools.agent.tools_controller import load_valid_tools, MTQuestionAnswerer
tools_mappings = {
    "klarna": "https://www.klarna.com/",
    "chemical-prop": "http://127.0.0.1:8079/tools/chemical-prop/",
    "wolframalpha": "http://127.0.0.1:8079/tools/wolframalpha/",
}

tools = load_valid_tools(tools_mappings)

qa =  MTQuestionAnswerer(openai_api_key='', all_tools=tools)

agent = qa.build_runner()

agent("How many benzene rings are there in 9H-Carbazole-3-carboxaldehyde? and what is sin(x)*exp(x)'s plot, what is it integrated from 0 to 1? ")
```

### 2.4 Use the web demo
1. Add your plugin to the mappings at beginning of web_demo.py

2. Start the webdemo
```
python web_demo.py
```




## 3. Use customized tools

### 3.1 Develop a tool locally

To develop a tool locally, you need to write a python function to build the tool and register it to the registry.

For example, you can write a tool that can execute python code and return the result. The following is a sample code:

```python
from bmtools.tools import Tool
from pydantic import BaseModel

class ExecutionQuery(BaseModel):
    code: str

class ExecutionResult(BaseModel):
    result: str

def build_python_tool(config) -> Tool:
    tool = Tool(
        "PythonTool",
        "A plugin that can execute python code",
        name_for_model="python", 
        description_for_model="A plugin that can execute python code",
        contact_email="your@email",
    )

    @tool.post("/execute")
    def execute_python_code(query : ExecutionQuery) -> ExecutionResult:
        return ExecutionResult(
            result=eval(query.code)
        )
    
    return tool
```

Then you need to register the tool to the registry using the following code:

```python
from bmtools.tools import register

@register("python")
def register_python_tool():
    return build_python_tool
```

Here we register the tool with the name `python`. 


### 3.2 Contributing to BMTools

After you have developed a tool, you can contribute it to BMTools by following the steps below:
1. Fork this repository
2. Create a folder in bmtools/tools/{tool_name}
3. Add an api.py to the folder: bmtools/tools/{tool_name}/api.py and a \_\_init\_\_.py to the folder: bmtools/tools/{tool_name}/__init__.py
4. Register the tool in the \_\_init\_\_.py file you created in step 3 using the code in section 3.1
5. Import your tool in the \_\_init\_\_.py file under bmtools/tools
6. Add a test.py to test your tool automatically

## 4. Optimize your tool's prompt
The functions you wrote will be converted into an interface compatible with the OpenAI plugin. The AI models will read the name, description of the tools, and the name, descriptions of the APIs of that tools. You can adjust the following aspect to make your API better understood by AI models.

- (1) `name_for_model` (tells the model what the tool is) 
- (2) `description_for_model` (this will be displayed to the model before the tool is called, and you can include information on how to use the APIs) 
- (3) The function name for each API function, as well as the name in `@tool.get()`. It's best if these two names match, as the name plays an important role in the model's API selection. 
- (4) The function's doc string (can suggest to the model whether to use this API or not) 
- (5) The function's return value, which can provide the model with error messages to guide its next steps, such as retrying or indicating a preferred next step 
- (6) Reduce the errors in your API function.

A sample to refer to is the Wolfram Alpha API.



