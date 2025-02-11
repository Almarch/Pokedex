# Run a LLM with RAG from a Gaming PC

The goal of this repo is to play with natural language processing with relatively limited resources.

It works on an [Open-WebUI](https://github.com/open-webui/open-webui) + [Ollama](https://github.com/ollama/ollama) stack, and encompasses a [Jupyter Notebook](https://github.com/jupyter/notebook).

## Deploy

<img src="https://github.com/user-attachments/assets/b12cbef1-98a9-4b79-bca0-fa1f21cb6f0e" width="200px" align="right"/>

The project is containerized with docker. However, some preliminary steps are required in order to prepare the LLM on the host machine.

```sh
git clone https://github.com/Almarch/NLP-from-a-PC
cd NLP-from-a-PC
docker compose build
```

Cuda is highly recommanded for performance. [Nvidia container toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) is needed.

## Install the LLM on the host machine

<img src="https://github.com/user-attachments/assets/7847f4c8-b8d7-483a-aa43-c00241c15891" width="200px" align="right"/>

The main idea is to run the LLM locally. In order to download and set up the LLM on the Ollama environment, the following steps should be done from the host machine. The host python3 requires:

- `torch`
- `transformers`
- `accelerate`

Use a `venv` if you don't want to mess with your system python.

Let's start by downloading the model from [Hugging face](https://huggingface.co/). A [frugal deepseek model](https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Llama-8B) has been picked but it may be changed for another one.

From Python3:

```py
from transformers import AutoModelForCausalLM, AutoTokenizer, GenerationConfig
import torch
import os

author = "deepseek-ai"
model_name = "DeepSeek-R1-Distill-Llama-8B"

tokenizer = AutoTokenizer.from_pretrained(
    author + "/" + model_name,
    cache_dir="./services/ollama/cache"
)
model = AutoModelForCausalLM.from_pretrained(
    author + "/" + model_name,
    cache_dir="./services/ollama/cache",
    torch_dtype=torch.bfloat16,
    device_map="auto"
)
generation_config = GenerationConfig.from_pretrained(
    author + "/" + model_name,
    cache_dir="./services/ollama/cache",
)
generation_config.pad_token_id = model.generation_config.eos_token_id

os.makedirs("./services/ollama/data/models/" + model_name, exist_ok=True)
tokenizer.save_pretrained("./services/ollama/data/models/" + model_name)
model.save_pretrained("./services/ollama/data/models/" + model_name)
generation_config.save_pretrained("./services/ollama/data/models/" + model_name)
```

Then, we will convert the model to Ollama format. To do so, we will use the `llama.cpp` docker image:

```sh
docker run --rm -v "./services/ollama/data/models:/models" \
    ghcr.io/ggerganov/llama.cpp:full \
    --convert --outtype f16 "/models/DeepSeek-R1-Distill-Llama-8B"
```

In the Ollama framework, a model requires a `.Modelfile`. One is provided for the selected model in this project. Be aware that the `.Modelfile` is specific to each model. Simply copy it in the appropriate folder:

```sh
cp ./services/ollama/.Modelfile \
   ./services/ollama/data/models/DeepSeek-R1-Distill-Llama-8B/.Modelfile
```

Now that the model is converted to the appropriate format and accompanied with a `.Modelfile`, it can be registered to Ollama. Access the running Ollama container, say number `123`:

```sh
docker compose up -d
docker ps
docker exec -it 123 bash
```

And register the model:

```sh
ollama create myModel -f /root/.ollama/models/DeepSeek-R1-Distill-Llama-8B/.Modelfile
```
For information, the `.Modelfile` has been obtained from within the Ollama container by running the following commands. However, they do not have to be run again.

```sh
ollama run llama3.1
ollama show --modelfile llama3.1
```

## That's all folks

The stack can now be launched using Docker:

```
docker compose up
```

The web-ui is available at http://localhost:8080 .

Resource consumptions may be followed-up with:

```
nvtop
```

for the VRAM and GPU; and:

```
htop
```

for the RAM and CPU.

## Tunneling

<img src="https://github.com/user-attachments/assets/86197798-9039-484b-9874-85f529fba932" width="100px" align="right"/>

It is sometimes easier to take a virtual private server (VPS) than obtaining a fixed IP from the Internet provider. We want some services from the gaming machine, let's call it A, to be accessible from anywhere, including from machine C. In the middle, B is the VPS used as a tunnel. 

Name|A  |B  |C  |
---|---|---|---
Description|Gaming machine  |VPS  |Client  |
Role|Host the models  |Host the tunnel  |Plays with NLP  | 
User|userA  |userB  | doesn't matter   | 
IP|doesn't matter  |11.22.33.44  | doesn't matter  | 

The services we need are:
- The web UI and the notebook, available at ports 8080 and 8888 respectively.
- A SSH endpoint. Port 22 of the gaming machine (A) will be exposed through port 2222 of the VPS (B).

### From A) the gaming machine
The ports are pushed to the VPS:

```sh
ssh -N -R 8888:localhost:8888 -R 8080:localhost:8080 -R 2222:localhost:22 userB@11.22.33.44
```

### From B) the VPS
The SSH port 2222 has to be opened.

```sh
sudo ufw allow 2222
sudo ufw reload
```

### From C) the client
The jupyter notebook is pulled from the VPS:

```sh
ssh -N -L 8888:localhost:8888 -L 8080:localhost:8080 -L 7777:localhost:7777  userB@11.22.33.44
```

And the VPS is a direct tunnel to the gaming machine A:

```sh
ssh -p 2222 userA@11.22.33.44
```

Note that `userA`, not `userB`, is required for authentication ; idem for the password.



