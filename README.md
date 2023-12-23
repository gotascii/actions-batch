# actions-batch

Time-sharing supercomputer built on GitHub Actions.

In the 1970s - or so I hear, [time-sharing](https://en.wikipedia.org/wiki/Time-sharing) was all the rage, with users being able to submit tasks or batch jobs to large computers, and to collect the results when the jobs where done.

[Read the blog post: GitHub Actions as a time-sharing supercomputer](https://blog.alexellis.io/github-actions-timesharing-supercomputer/)

## Goal

Run a shell script in an isolated, immutable environment, collect the logs or results.

This works well with self-hosted runners managed by actuated (which use a full VM) or GitHub's hosted runners. It may also work with container-based runners, with some limitations on what software can be used securely i.e. `docker`.

See an example:

[![https://pbs.twimg.com/media/GB3AqMoXwAASieh?format=jpg&name=medium](https://pbs.twimg.com/media/GB3AqMoXwAASieh?format=jpg&name=medium)](https://twitter.com/alexellisuk/status/1737757819288314091/photo/2)

> Example output from [examples/govulncheck.sh](examples/govulncheck.sh):

## What's supported?

* [x] Public repos
* [x] Private repos
* [x] Self-hosted runners
* [x] GitHub-hosted runners
* [x] Using secrets via repository secrets
* [x] Downloading the results of a build as an artifact

## You might also be interested in

* [alexellis/run-job](https://github.com/alexellis/run-job) - run one-shot jobs on Kubernetes
* [OpenFaaS](https://github.com/openfaas/faas) - portable, open-source FaaS for Kubernetes and [standalone containerd](https://github.com/openfaas/faasd)

## How it works

1. You write a bash script like the ones in [examples](/examples) and pass it in as an argument
1. A new repo is created with a random name in the specified organisation
2. A workflow file is written to the repo along with the shell script, the workflow's only job is to run the shell script and exit
3. The workflow is triggered and you can check the results

`-owner` is intended to be a GitHub organisation, but this can be adapted to a personal account by passing `--org false` to the command.

```bash
git clone git@github.com:alexellis/actions-batch
cd actions-batch
```

You'll need a Personal Access Token (PAT) with: delete_repo, repo, workflow, write:packages.

You can download a binary from the [Releases page](https://github.com/alexellis/actions-batch/releases) or build it from source.

### Generate some ASCII art with your personal repo

For a personal account using a hosted runner and a public repo:

```bash
actions-batch \
  --owner alexellis \
  --token-file ~/pat.txt \
  --runs-on ubuntu-latest \
  --file examples/cowsay.sh
```

### Run an LVM using llama and a model from HuggingFace

Use [examples/llama.sh](/examples/llama.sh), snippet below:

```python3
from llama_cpp import Llama
LLM = Llama(model_path="./llama-2-7b-chat.Q5_K_M.gguf")
   
# create a text prompt
prompt = "Q: What are the names of the days of the week? A:"

# generate a response (takes several seconds)
output = LLM(prompt,max_tokens=300, stop=[])
```

Use the `--out ./out` flag, then `cat ./out/output.txt` to see the results.

```
 The names of the days of the week are: Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, and Sunday.
Q: How many days are in a week? A: There are 7 days in a week.

[{'text': ' The names of the days of the week are: Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, and Sunday.\nQ: How many days are in a week? A: There are 7 days in a week.', 'index': 0, 'logprobs': None, 'finish_reason': 'stop'}]
```

### Run Docker slim against an image using a self-hosted runner and a GitHub organisation

For an organisation, using an Arm-based private repo and custom runner with 32vCPU and 256GB of RAM

```bash
actions-batch \
  --private \
  --owner actuated-samples \
  --token-file ~/pat.txt \
  --runs-on actuated-arm64-32cpu-256gb \
  --file examples/slim.sh
```

Example of what's written to the repo:

```bash
name: workflow

# Generated by alexellis/actuated-batch at: 2022-10-07 20:14:30.547822 +0100 BST m=+0.002729293
# Job requested by alex

on:
  pull_request:
    branches:
      - '*'
  push:
    branches:
      - master
      - main

jobs:
  workflow:
    name: agitated_solomon5
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - name: Run the job
        run: |
          chmod +x ./job.sh
          ./job.sh
```

### Consume secrets for a job

A folder can be given, where each file is a secret, the name will be the filename made uppercase, with `-` replaced by `_`.

Take the following example to log into OpenFaaS:

```bash
#!/bin/bash

set -e -x -o pipefail

curl -sLS https://get.arkade.dev | sudo sh

arkade get faas-cli --quiet
sudo mv $HOME/.arkade/bin/faas-cli /usr/local/bin/
sudo chmod +x /usr/local/bin/faas-cli 

echo "${OPENFAAS_GATEWAY_PASSWORD}" | faas-cli login -g "${OPENFAAS_URL}"/function/printer -u admin --password-stdin
curl -h "X-Url: ${OPENFAAS_URL}" https://openfaas.o6s.io/function/printer 
```

Therefore just create two files:

```bash
mkdir -p .secrets
echo "admin password" > .secrets/openfaas-gateway-password
echo "https://gateway.example.com" > .secrets/openfaas-url
```

Then run the script passing in that folder:

```bash
actions-batch \
  --private=false \
  --owner alexellis \
  --token-file ~/batch \
  --runs-on ubuntu-latest \
  --org=false \
  --file examples/secret.sh \
  --secrets-from .secrets
```

[OpenFaaS can be exposed over the Internet using an inlets tunnel](https://inlets.dev/blog/2020/10/15/openfaas-public-endpoints.html).

### Download files from a job

You may be processing a video with ffmpeg, building a binary, a container image, PDF, or even an ISO.

Place the result in a folder named `uploads` and the contents will be uploaded as an artifact to GitHub Actions, before being downloaded to your machine and extracted for you to view.

Bear in mind that if your artifact is confidential or private, then you will need to use the `--private` flag to create a private repo.

A good example is [examples/youtubedl.sh](examples/youtubedl.sh) which downloads a video from YouTube.

```bash
actions-batch \
  --private \
  --owner actuated-samples \
  --token-file ~/batch \
  --runs-on actuated \
  --file ./examples/youtubedl.sh
```

Output:

```bash
Wrote 43.84kB to /tmp/uploads1127046452
Extracting: /tmp/artifacts-968188/video.flv
2023/12/21 16:04:01 extracted zip into /tmp/artifacts-968188: 1 files, 0 dirs (672.583µs)
FILE      SIZE
video.flv 47.39kB

xdg-open /tmp/artifacts-968188/video.flv
```

[![Example video playing](https://pbs.twimg.com/media/GB4eGfzXcAAQOlE?format=jpg&name=medium)](https://twitter.com/alexellisuk/status/1737859786413322477/)

### Build a Linux Kernel and download it a working folder

```bash
mkdir -p kernel-bin
actions-batch \
  --owner alexellis \
  --org=false \
  --file examples/linux-kernel.sh \
  --runs-on ubuntu-latest \
  --out ./kernel-bin

du -h kernel-bin/*
```

### Build a Docker image remotely and import it to your library

```bash
actions-batch \
--private=false \
--owner actuated-samples \
--token-file ~/batch \
--runs-on ubuntu-latest \
--file ./examples/export-docker-image.sh 

Wrote 5.665MB to /tmp/uploads2533248860
Extracting: /tmp/artifacts-888719363/curl.tar
2023/12/21 17:19:32 extracted zip into /tmp/artifacts-888719363: 1 files, 0 dirs (77.250512ms)
FILE     SIZE
curl.tar 12.37MB

QUEUED DURATION TOTAL
4s     15s      22s

docker load -i /tmp/artifacts-888719363/curl.tar
08db363dedea: Loading layer [==================================================>]  4.687MB/4.687MB
Loaded image: curl:latest

docker run -t curl:latest --version
```

## What's left

The main thing I'd like is more examples of workloads that can be run on a Linux system

* [Browse the examples](/examples)
* [See the issue tracker](https://github.com/alexellis/actions-batch/issues)

## License

MIT

DCO - a Signed-off-by message will be required in each commit message i.e. `git commit --signoff`
