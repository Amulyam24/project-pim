# vLLM

> This directory contains two variants: the standard **ppc64le CPU-based** vLLM image and an **AMD ROCm GPU** variant. Jump to [AMD ROCm variant](#amd-rocm-variant) if that is what you need.


vLLM example allows you to deploy vLLM inference engine that exposes OpenAI API server on a partition which allows you to leverage the GenAI capabilities on your on prem environment.

## Architecture
![alt text](vLLM_Arch.png)

## Steps to setup e2e flow

### Step 1: Preparing the images

#### Use Pre-built images
##### Application Image
- Container image that can run vLLM's OpenAI API server 
- Recommended to use the vLLM application image built by IBM Linux on Power team. Image details are updated over a blog post [here](https://community.ibm.com/community/user/blogs/priya-seth/2023/04/05/open-source-containers-for-power-in-icr)
```
icr.io/ppc64le-oss/vllm-ppc64le:0.10.1.dev852.gee01645db.d20250827
```
##### PIM Bootc Image
- Bootc image to bringup the AI partition that can run the above vLLM application container.
- Recommended to use the pre-built PIM Bootc image and its available to consume directly via below image.
```
quay.io/powercloud/pim:vllm
``` 

#### Build from source
If you wish to build your own version, you can follow below steps to build it.

##### Step 1: Build Application image

Follow the instructions in the [README](app) to build the vLLM application's container image. It has a script that pulls open-source vLLM code base and builds a container image.

##### Step 2: Build PIM Base image

Follow the steps provided [here](../../base-image) to build the base image or use the pre-built base-image `quay.io/powercloud/pim:base`

##### Step 3: Build PIM Bootc image

Ensure to replace the `FROM` image in [Containerfile](Containerfile) with the base image you have built before building this image.

```shell
podman build -t <your registry>/pim:vllm

podman push <your registry>/pim:vllm
```

---

## AMD ROCm Variant

[`Containerfile.rocm`](Containerfile.rocm) builds a vLLM PIM bootc image for AMD ROCm on ppc64le.

### Architecture

```
base-image/Containerfile.rocm      →  quay.io/<account>/pim:base-rocm
        ↓ (FROM)
examples/vllm/Containerfile.rocm   →  quay.io/<account>/pim:vllm-rocm
        ↓ (bootc deploy)
AMD GPU machine running vllm-rocm.service on port 8000
```

### Required local assets

Before building, populate `custom-rocm/` inside `examples/vllm/` with the torch/triton/vllm wheels (the ROCm SDK wheels belong in `base-image/custom-rocm/`):

```
examples/vllm/
├── Containerfile.rocm
├── vllm-rocm.service
└── custom-rocm/
    ├── torch-*.whl
    ├── torchvision-*.whl
    ├── torchaudio-*.whl
    ├── triton-*.whl
    └── vllm-*.whl
```

### Step 1: Build the base ROCm image

Follow the steps in [`base-image/README.md`](../../base-image/README.md#amd-rocm-variant) to build and push `pim:base-rocm` first, then update the `FROM` line in `Containerfile.rocm` with your registry tag.

### Step 2: Build the vLLM ROCm bootc image

```shell
# Run from the examples/vllm/ directory
podman build -t localhost/pim-vllm-rocm -f Containerfile.rocm .

podman tag localhost/pim-vllm-rocm quay.io/<account-id>/pim:vllm-rocm
podman push quay.io/<account-id>/pim:vllm-rocm
```

### Step 3: Deploy via PIM

In `config.ini`, update the two fields below. Everything else (`[partition]`, `[network]`, `[storage]`, `[ssh]`) stays the same as any other PIM deployment.

```ini
[ai]
  image = "quay.io/<account-id>/pim:vllm-rocm"
  config-json = """"""    # leave empty — model and args are baked into vllm-rocm.service
  auth-json = """"""      # add registry credentials here if your image is in a private registry
  [[validation]]
    request = "yes"
    url = "http://<partition-ip>:8000/v1/chat/completions"
    method = "POST"
    headers = """{"Content-Type": "application/json"}"""
    payload = """{"model": "ibm-granite/granite-3.3-8b-instruct", "messages": [{"role": "user", "content": "What is the capital of France?"}]}"""
```

`config-json` can be left empty — the CLI treats it as `{}` and only adds `workloadImage` to it automatically. `llmImage` / `llmArgs` / `llmEnv` are not needed because the model and runtime args are baked directly into [`vllm-rocm.service`](vllm-rocm.service). To change the model or flags after deployment, update the service file, rebuild, push, and run `python3 cli/pim.py upgrade`.

Then run the standard launch:

```shell
python3 cli/pim.py launch
```

The vLLM OpenAI API server will be available on port `8000` once the partition boots.

### Step 2: Setting up PIM partition

Follow this [deployer guide](../../docs/deployer-guide.md) to setup PIM cli, configuring your AI partition and launching it.
Regarding the configuration of your vLLM application, below configs are supported. Please read through them and use them as per your requirement in config file detailed in deployer guide.

#### llmImage
- Use `llmImage` param to pass the vLLM application's container image to be used in your AI partition. 
- Look at the image section [here](#step-1-preparing-the-images) to decide the image to be used.
- This is given as a configurable option so that in future if there is a newer version of vLLM image available, we can just update the stack via [update-config](../../docs/deployer-guide.md#update-config)
#### llmArgs
- Arguments you want to pass it to your vLLM inference engine
#### llmEnv
- Environment variables that you want to set while running vLLM inference engine
#### modelSource
- A JSON object that specifies the source from which you want to download the model. 
- Use this parameter to use a offline model loaded within the local network instead of downloading from hugging face over the internet. This is suitable for environment which restricts outside connection. 
- Follow the steps [here](local-model-server.md) to bring up self-hosted HTTP server which serves the offline models.

**Sample config:**
```ini
config-json = """
  {
        "llmImage": "icr.io/ppc64le-oss/vllm-ppc64le:0.10.1.dev852.gee01645db.d20250827",
        "llmArgs": "--model ibm-granite/granite-3.3-8b-instruct --max-model-len=8192 --max-num-batched-tokens=8192",
        "llmEnv": "OMP_NUM_THREADS=16,VLLM_CPU_OMP_THREADS_BIND=all",
        "modelSource": { "url": "http://<Host/ip>/models--ibm-granite--granite-3.2-8b-instruct.tar.gz" }
  }
  """
```
