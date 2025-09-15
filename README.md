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
            "email": {},
            "gg-hap": {
                "threshold": 0
            }
        }
    }
}'
```

The source code for the e-mail detector can be found on [this repository](TODO: add repository).
