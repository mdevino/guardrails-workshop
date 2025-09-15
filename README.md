# Guardrails Workshop
This document contains the steps necessary to run the guardrails ecosystem locally. 
The guardrails framework was built to run on production settings.
However, this tutorial works with small components that can be executed locally.
The intent of this workshop is not to have a production-ready environment, but to illustrate the components of the guardrails framework and their functionalities.


## Download Container Images
The guardrails ecossystem is composed of four component types: generation servers, detectors, chunkers and an orchestrator.
The commands below can be used to download the docker images for each component.

```
docker pull ollama/ollama:0.11.7
docker pull quay.io/mdevin0/guardrails-orchestrator:latest
docker pull quay.io/mdevin0/email-detector:latest
docker pull quay.io/mdevin0/granite-guardian-hap-detector:latest
docker pull quay.io/mdevin0/chunker:latest
```


## Generation Server Setup
**Generation servers** are responsible for serving a generative language model.
At this point in time, the framework supports servers that comply with the Open AI [Completions](https://platform.openai.com/docs/api-reference/completions) and [Chat Completions](https://platform.openai.com/docs/api-reference/chat) APIs.
However, only vLLM was tested as a generation server. In this workshop, we will use [Ollama](https://ollama.com/) as our generation server.

We'll be using the official vLLM docker image through docker compose. To start the service, create a file named `docker-compose.yaml` with the following content:

```yaml
services:
  generation-server:
    image: ollama/ollama:0.11.7
    container_name: generation-server
    ports:
      - "8000:8000"
    volumes:
      - ollama_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0:8000
    restart: unless-stopped

volumes:
  ollama_data:
    driver: local
```

Then, run `docker compose up` to start the service.
The next step is to download the generation model.
To do so, run the following commands on a different terminal:

```
# Get inside generation-server container
docker exec -ti generation-server bash
# Download model
ollama pull qwen3:0.6b
# Exit container
exit

# Test model is listed
curl localhost:8000/v1/models
# The output of this command should be similar to this: {"object":"list","data":[{"id":"qwen3:0.6b","object":"model","created":1757891103,"owned_by":"library"}]}
```

You can run the following command to test the model:

```bash
curl --location 'http://localhost:8000/v1/completions' \
--header 'Content-Type: application/json' \
--data '{
    "model": "qwen3:0.6b",
    "prompt": "/no_think Hi there! How are you?"
}'
````

## Orchestrator Setup
Once we have the generation-server up and running, we can setup the orchestrator and configure it to use the generation server. 
Create a file named `orchestrator.yaml` and add the following content:

```yaml
openai:
  service:
    hostname: generation-server
    port: 8000
detectors:
    email:
        type: text_contents
        service:
            hostname: email-detector
            port: 8001
        chunker_id: whole_doc_chunker
        default_threshold: 0.5
```

The documentation for the orchestrator config can be found [here](https://github.com/foundation-model-stack/fms-guardrails-orchestrator/blob/main/config/config.yaml).
Then, run the following command to allow the file to be accessed by containers:
```sudo chcon -Rt svirt_sandbox_file_t orchestrator.yaml```

Now we need to add the orchestrator service to the compose file. To do so, add the following under `services`:

```yaml
orchestrator:
    image: quay.io/mdevin0/guardrails-orchestrator:latest
    container_name: orchestrator
    ports:
      - "8090:8033"
    volumes:
      - ${PWD}/orchestrator.yaml:/config/config.yaml
    environment:
      - ORCHESTRATOR_CONFIG=/config/config.yaml
    depends_on:
      - generation-server
    restart: unless-stopped
```

After that, execute `docker compose up` to bring up both, the generation server and the orchestrator.
To test the connection, you can run the following command:

```bash
curl --location 'http://localhost:8090/api/v2/text/completions-detection' \
--header 'Content-Type: application/json' \
--data '{
    "model": "qwen3:0.6b",
    "prompt": "/no_think Hi there! How are you?"
}'
```

The documentation for the orchestrator completions endpoint can be found [here](https://foundation-model-stack.github.io/fms-guardrails-orchestrator/?urls.primaryName=Orchestrator+API#/Task%20-%20Completions%2C%20with%20detection/api_v2_text_completions_detection_handler), and its source code can be found in [this repository](https://github.com/foundation-model-stack/fms-guardrails-orchestrator).


## E-mail Detector Setup
Now we have the generation server and the orchestrator setup, but there is no value in using the guardrails framework if there are no detectors. We will configure an e-mail detector now by adding the following `service` to `docker-compose.yaml`:

```yaml
email-detector:
    image: quay.io/mdevin0/email-detector:latest
    container_name: email-detector
    ports:
      - "8091:8001"
    restart: unless-stopped
```

Then, run `docker compose up` to bring up the new services.
To test an input detection, run the following command:

```bash
curl --location 'http://localhost:8090/api/v2/text/completions-detection' \
--header 'Content-Type: application/json' \
--data-raw '{
    "model": "qwen3:0.6b",
    "prompt": "/no_think My e-mail is mateus@example.com.",
    "detectors": {
        "input": {
            "email": {}
        }
    }
}'
```

To test output detection, run the following request:
```bash
curl --location 'http://localhost:8090/api/v2/text/completions-detection' \
--header 'Content-Type: application/json' \
--data '{
    "model": "qwen3:0.6b",
    "prompt": "Could you generate a random e-mail address?",
    "detectors": {
        "output": {
            "email": {}
        }
    }
}'
```

The source code for the e-mail detector can be found on [this repository](https://github.com/mdevino/email-detector).


## Sentence Chunker Setup
Now that we have a detector, we will configure a sentence chunker.
Run the following commands
Add the following service to `docker-compose.yaml`:

```yaml
sentence-chunker:
    image: quay.io/mdevin0/chunker:latest
    container_name: sentence-chunker
    ports:
      - "50052:50051"
    volumes:
      -  nltk_data:/root/nltk_data/
    restart: unless-stopped
```

And the following volume:
```yaml
  nltk_data:
    driver: local
```

In the `orchestrator.yaml`, add the following entry right at the top level, right below the `openai` block:

```yaml
chunkers:
    sentence_chunker:
        type: sentence
        service:
            hostname: sentence-chunker
            port: 50051
```

The block above registers the chunker in the orchestrator.
We can also register a version of the e-mail detector configured with the sentence chunker by adding the following entry under `detectors`:

```yaml
email-sentence:
    type: text_contents
    service:
        hostname: email-detector
        port: 8001
    chunker_id: sentence_chunker
    default_threshold: 0.5
```

Now we can start the docker compose services and get into the sentence chunker container to download the necessary model.
To do so:

```bash
docker exec -ti sentence-chunker bash
python
import nltk
nltk.download('punkt_tab')
exit
exit
```

Now we can test a call to the e-mail detector configured with the sentence chunker by running the following:

```bash
curl --location 'http://localhost:8090/api/v2/text/completions-detection' \
--header 'Content-Type: application/json' \
--data-raw '{
    "model": "qwen3:0.6b",
    "prompt": "Hey, how'\''s it going? My e-mail address is mateus@example.com.",
    "detectors": {
        "input": {
            "email-sentence": {}
        }
    }
}'
```

The source code for this chunker is available on [this repository](https://github.com/mdevino/sentence-chunker).


## Granite Guardian HAP Detector Setup

The last step in this tutorial is configuring HAP (Hate, Abuse and Profanity) detector based a Granite Guardian model.
Granite is a open source family of models developed by IBM under the Apache 2.0 license, which means these models can be used even in enterprise environments.
We will be using [a Granite Guardian model focused on HAP detection](https://huggingface.co/ibm-granite/granite-guardian-hap-38m).

To enable this detector, add the following entry under `services` in `docker-compose.yaml`:

```yaml
gg-hap-detector:
    image: quay.io/mdevin0/granite-guardian-hap-detector:latest
    container_name: gg-hap-detector
    ports:
      - "8092:8002"
    volumes:
      -  ~/.cache/huggingface/:/root/.cache/huggingface/
    restart: unless-stopped
```

Then, register the detector configured with the sentence chunker in `orchestrator.yaml`, under `detectors`:

```yaml
gg-hap-sentence:
    type: text_contents
    service:
        hostname: gg-hap-detector
        port: 8002
    chunker_id: sentence_chunker
    default_threshold: 0
```

Feel free to also register a version using the whole_doc_chunker.

To test the detection, run the following command:

```bash
curl --location 'http://localhost:8090/api/v2/text/completions-detection' \
--header 'Content-Type: application/json' \
--data '{
    "model": "qwen3:0.6b",
    "prompt": "I love cats! I hate aliens!",
    "detectors": {
        "input": {
            "gg-hap-sentence": {}
        }
    }
}'
```

The source code for this detector can be found on [this repository](https://github.com/mdevino/gg-hap-detector).


## Summary

In this workshop, we configured the Guardrails ecosystem, which can be used to perform detections in both, user input and LLMs output.
While the generation server and orchestrator are ready for production use, the chunker and detectors used in this tutorial are intended for educational purposes only.
Also, depending on your production workload, you'll most likely want to use a generation server like [vLLM](https://docs.vllm.ai/en/stable/index.html), as Ollama is intended mostly for local usage.