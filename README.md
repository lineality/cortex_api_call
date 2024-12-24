# cortex_api_call

# cortex.cpp cheatsheet  https://github.com/janhq/cortex.cpp

## For using cortex.cpp within python code:
1. This has worked on ubuntu with cuda installed, .gguf models within 1:1 size of gpu-ram are blazing fast.
2. This guide does NOT work with the jan-cortex api (sadly). I message saying:
"{"message":"Model has not been loaded, please load model into cortex.llamacpp"}" is returned,
and I can find zero documentation or clues about how to load a model through the web-api.


# Steps:
## 1. In bash terminal, before running this, start the cortex server:

If not done, install
```bash
curl -s https://raw.githubusercontent.com/janhq/cortex/main/engine/templates/linux/install.sh | sudo bash -s -- --deb_local
```

2. To be safe, fresh install huggingface model (note if license is appropriate for your use, and authors trusted)
```bash
cortex pull bartowski/Mistral-Nemo-Instruct-2407-GGUF
```
or
```bash
cortex pull janhq/trinity-v1.2-GGUF
```
Note: for a ~custom model you may need to make your own yaml file to match the cortex system, 
hopefully very straightforward.

3. Load//Start/Run model:
```bash
cortex run Mistral-Nemo-Instruct-2407-GGUF
```

4. Exit prompt but keep cortex server running:
(This step may be optional.)
```bash
exit()
```

5. Check that model is running & inspect:
```bash
cortex ps
```
This will show you the 'name' notation that you need to use to call the model.


# Advocacy for Better Documentation
Note: Documentation for 
A. the (totally amazing) cortex cli, and 
B. the hopefully someday working Jan-cortex api, 
is not sufficient to get the code working.

1. The notation system for actually connecting to a model should be this:

Convert the last three parts of the full file path in an undocumented colon-demarcated notation. 
The cortex "name" of your model is the last three parts of the model path
separated by a colon.

### e.g.
The path to your model will be something like this:
```
/home/YOURCOMPUTERNAME/cortexcpp/models/huggingface.co/bartowski/Mistral-Nemo-Instruct-2407-GGUF/Mistral-Nemo-Instruct-2407-Q6_K_L.gguf
```

# Use the last three path segments with a colon ":" between them:
```
    So, "/Mistral-Nemo-Instruct-2407-GGUF/Mistral-Nemo-Instruct-2407-Q6_K_L.gguf" becomes ->
    "model": "bartowski:Mistral-Nemo-Instruct-2407-GGUF:Mistral-Nemo-Instruct-2407-Q6_K_L.gguf",

    or:

    "model": "janhq:trinity-v1.2-GGUF:trinity-v1.2.Q8_0.gguf"
```
# using a path will NOT work "model": "/home/YOURCOMPUTERNAME/cortexcpp/models/huggingface.co/bartowski/Mistral-Nemo-Instruct-2407-GGUF/Mistral-Nemo-Instruct-2407-Q6_K_L.gguf",

A. Documentation here:
https://cortex.so/docs/quickstart/
says to use: "model": "llama3.1:8b-gguf",
as the model name which is not a system that has worked for me.

B. The Built-in documentation: localhost:39281/
gives you these specs to follow:
```
curl http://127.0.0.1:39281/v1/chat/completions \
  --request POST \
  --header 'Content-Type: application/json'
```
This crashes cortex every time, and says nothing about model notation.

C. The jan cortex api fastify documentation: http://localhost:1337/static/index.html
says to use this:
{
  "messages": [
    {
      "content": "You are a helpful assistant.",
      "role": "system"
    },
    {
      "content": "Hello!",
      "role": "user"
    }
  ],
  "model": "tinyllama-1.1b",
  "stream": true,
  "max_tokens": 2048,
  "stop": [
    "hello"
  ],
  "frequency_penalty": 0,
  "presence_penalty": 0,
  "temperature": 0.7,
  "top_p": 0.95
}
the "stop" command seems to break the api by design, 
the notation for model-id uses no colon,

2. The "stop" field is routinely configured to make output impossible
- Do not use Null
- leave stop out entirely, or use []



Can use:

http://127.0.0.1:39281/v1/models

Example of "id" without quotes:
http://127.0.0.1:39281/v1/models/bartowski:Mistral-Nemo-Instruct-2407-GGUF:Mistral-Nemo-Instruct-2407-Q6_K_L.gguf

```python
import requests
import json


def call_cortex_api(
    cortex_prompt,
    max_output_tokens=600,
    cortex_model="bartowski:Mistral-Nemo-Instruct-2407-GGUF:Mistral-Nemo-Instruct-2407-Q6_K_L.gguf",
    # jan_api_cortext=False,
):
    """
    Before using, start local Cortex 'server':
        run 
        ```bash
        cortex ps
        ```
    to see the name/notation for models 'names'
    
    Requires:
        import requests
        import json
    """
    
    # if jan_api_cortext is True:
    #     local_cortex_domain = "http://localhost:1337/"
    
    # else:
    #     local_cortex_domain = "http://localhost:39281/"
    
    local_cortex_domain = "http://localhost:39281/"

    try: 
        url = f"{local_cortex_domain}v1/chat/completions"
        
        headers = {
            "Content-Type": "application/json"
        }
        
        data = {
            # use last three path segments with : between
            "model": cortex_model,
            # using path will NOT work "model": "/home/YOURCOMPUTERNAME/cortexcpp/models/huggingface.co/bartowski/Mistral-Nemo-Instruct-2407-GGUF/Mistral-Nemo-Instruct-2407-Q6_K_L.gguf",
            "messages": [
            {
              "role": "user",
              "content": cortex_prompt
            },
            ],
            "stream": False,
            "max_tokens": max_output_tokens,
            "stop": [],
            "frequency_penalty": 1,
            "presence_penalty": 1,
            "temperature": .5,
            "top_p": 1
        }
        
        response = requests.post(url, headers=headers, data=json.dumps(data))
        
        response_dict = response.json()
        
        cortext_output_string = response_dict["choices"][0]["message"]["content"]
        
        return (cortext_output_string, True)

    except Exception as e:
        return (f"Failed, Error: {str(e)}", False)

#######
# Test
#######
# # Timer: For Start of Script
start_the_timer = start_timer()

# Cortext Call Function
cortext_output_string = call_cortex_api("Are birds dinosaurs?")

print(cortext_output_string)

# # Timer: For End of Script
end_timer(start_the_timer)
```
