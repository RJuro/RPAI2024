### Building a simple chatbot API with LangServe and deployment on Huggingface Spaces using Docker

We are following the tutorial for the pirate-speak app from langserve: https://github.com/langchain-ai/langchain/blob/master/templates/README.md
Follow the official tutorial there - however:

We are changing up folowing! 
- we use the `meta-llama/Llama-3-70b-chat-hf` model and the prompt from the `langchain_together` package.
- we add api-key handling using .env files

The `server.py` file looks like this after adding the pirate-speak chain:

```python

from fastapi import FastAPI
from fastapi.responses import RedirectResponse
from langserve import add_routes
from pirate_speak.chain import chain as pirate_speak_chain

app = FastAPI()


@app.get("/")
async def redirect_root_to_docs():
    return RedirectResponse("/docs")


# Edit this to add the chain you want to add
add_routes(app, 
           pirate_speak_chain, 
           path="/pirate-speak",
           playground_type='chat') # for nice chat-playground

if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=8000)

```

The `chain.py` file looks like this:

```python
#from langchain_community.chat_models import ChatOpenAI
from langchain_together import Together
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

import os
#from dotenv import load_dotenv
#load_dotenv()
together_api_key = os.getenv("TOGETHER_API_KEY")

_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "Du er en ung mand fra Randers og du opfylder alle stereotyper. Du svarer som sådan en ville gøre. Du svarer relativ kort og stopper efter et svar.",
        ),
        MessagesPlaceholder("chat_history"),
        ("human", "{text}"),
    ]
)
_model  = Together(
    model="meta-llama/Llama-3-70b-chat-hf",
    temperature=0.7,
    top_k=50,
    top_p=0.7,
    repetition_penalty=1,
    together_api_key=together_api_key
)


# if you update this, you MUST also update ../pyproject.toml
# with the new `tool.langserve.export_attr`
chain = _prompt | _model

    
```

Once you have created the chain and added it to the app (server.py) you can run the app with the following command:

```bash
langchain serve
```
make sure that you are in an environment with langchain installed.

You can now access the playground at `http://localhost:8000/pirate-speak/playground` and test it out.
The API is available at `http://localhost:8000/pirate-speak/

You can use a RemoteRunnable to access the API from your code. Here is an example:

```python

from langserve.client import RemoteRunnable

# Initialize the RemoteRunnable with your API 
rag_app = RemoteRunnable("http://127.0.0.1:8000/rag-chroma/")

# call the API with a question
answer = rag_app.invoke({"text": "Hej Mads, hvad så, skal du have en ny bil?", "chat_history": []})

print(answer)

```

Optional:
You can test if the container is functional by running it locally.

```bash
docker build . -t YOUR-GREAT-NAME
```

```bash
docker run -p 8080:8080 -e PORT=8080 YOUR-GREAT-NAME
```

You can deploy on Huggingface Spaces as a Docker API
This requires that you start up a space on Huggingface and clone it into a repository on your local machine.

you will then have to copy all files from your project directory into this one. Make sure to not overwrite the README.md that comes from huggingface. It contains the instructions for the deployment that HF uses to start the container.

To be able to push you will need to set your token in the git remote setup

```bash
git remote set-url origin https://USENAME:HF_TOKEN@huggingface.co/spaces/USER/REPO-ID
```

You will need to edit the dockerfile to include the together api key at build time. First add it to the secrets in the HF space. 
You also need to change up the uvicorn command (from the initial one created by langchain)


```Dockerfile
RUN --mount=type=secret,id=TOGETHER_API_KEY,mode=0444,required=true 

CMD ["uvicorn", "app.server:app", "--host", "0.0.0.0", "--port", "7860"]
```

Since we are adding new libraries to the project, you will need to add them to the pyproject.toml file. You can do this by editing the following lines - pydantic needs to be upversioned to 2.6.0 due to langchain_together requirements.

```toml
pydantic = "2.6.0"
pirate-speak = {path = "packages/pirate-speak", develop = true}
python-dotenv = "1"
langchain-together = "0.1.0"
```

Now you should be able to push the changes to HF Spaces, which should trigger a build and deployment of the API. You can access the API at the URL provided by HF.


You can try out the deployment on HF from the workshop:

```python
from langserve.client import RemoteRunnable
rag_app = RemoteRunnable("https://rjuro-rpai2024-bot.hf.space/pirate-speak/") # Roman's deployment. You can also check out the file structure there.
result = rag_app.invoke({"text": "Hej Mads, hvad så, skal du have en ny bil?", "chat_history": []})
print(result)
```

### Running the same with a local model:

you can easily swap out the together.ai model for a local one:

```python
from langchain_community.llms import Ollama

_model = Ollama(
    model="llama3"
)
```
