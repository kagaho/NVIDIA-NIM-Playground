# NGC and NIM

### Sites:  
- NGC:  
https://org.ngc.nvidia.com/setup/installers/cli
  
- NIM:  
https://docs.nvidia.com/nim/large-language-models/latest/getting-started.html
https://build.nvidia.com/openai/gpt-oss-20b/deploy
  
  
### useful commands:  
```
ngc registry image list --format_type ascii nvcr.io/nim/*
```
  
### Check NIM Model profiles
- https://docs.nvidia.com/nim/large-language-models/latest/profiles.html
  
```
docker run --rm --gpus=all -e NGC_API_KEY=$NGC_API_KEY $IMG_NAME list-model-profiles
```
  

example:
```
echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin
export CONTAINER_NAME=llama-3.1-8b-instruct
export Repository=nim/meta/llama-3.1-8b-instruct
export TAG=latest
ngc registry image info --format_type ascii $Repository:$TAG
export IMG_NAME="nvcr.io/$Repository:$TAG"
podman run -it --rm  --gpus all -e NGC_API_KEY=$NGC_API_KEY   -v "$LOCAL_NIM_CACHE:/opt/nim/.cache" $IMG_NAME   list-model-profiles
```

  

- For below , I could not have gtp-oss 20B with TensoRT-LLM, so I had to use VLLM profile
  

#### gpt-oss NIM container (VLLM)
  
```
podman run -d \
  --name nim-gpt-oss-20b \
  --restart=unless-stopped \
  --device nvidia.com/gpu=all \
  --shm-size=16GB \
  -e NGC_API_KEY \
  -e NIM_MODEL_PROFILE="66fb3113efd2aae1b0a3bfa2a375de5fe1cc1b557abac4eb271730482a26ae8e" \
  -e VLLM_MAX_MODEL_LEN=8192 \
  -e VLLM_MAX_NUM_SEQS=4 \
  -e VLLM_GPU_MEMORY_UTILIZATION=0.80 \
  -e PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  -v "$LOCAL_NIM_CACHE:/opt/nim/.cache:Z,U" \
  -p 8000:8000 \
  nvcr.io/nim/openai/gpt-oss-20b:latest
```

https://build.nvidia.com/openai/gpt-oss-20b/deploy
https://docs.nvidia.com/nim/large-language-models/latest/getting-started.html


```
󰣛 soundwave :   ~   14:11  ❯ podman ps
CONTAINER ID  IMAGE                                  COMMAND     CREATED         STATUS         PORTS                                       NAMES
807d1896b124  nvcr.io/nim/openai/gpt-oss-20b:latest              27 seconds ago  Up 27 seconds  0.0.0.0:8000->8000/tcp, 6006/tcp, 8888/tcp  nim-gpt-oss-20b
```
  

wait serve is ready:
```
podman logs -f nim-gpt-oss-20b          # follow logs

<...>
INFO 2026-01-30 13:12:23.158 on.py:62] Application startup complete.
[2026-01-30 13:12:23] INFO on.py:62: Application startup complete.
INFO 2026-01-30 13:12:23.159 server.py:214] Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
[2026-01-30 13:12:23] INFO server.py:214: Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```


Test:
curl -X 'POST' \
'http://0.0.0.0:8000/v1/chat/completions' \
-H 'accept: application/json' \
-H 'Content-Type: application/json' \
-d '{
    "model": "openai/gpt-oss-20b",
    "messages": [{"role":"user", "content":"Which number is larger, 9.11 or 9.8?"}],
    "max_tokens": 64
}'


references: https://build.nvidia.com/openai/gpt-oss-20b/deploy



Note: This NIM is not Triton Server with Tensor-RT:

podman logs --since 10m nim-gpt-oss-20b | grep -E "Backend type|Triton|TensorRT"
```
INFO 2026-01-30 13:11:28.375 vllm_api.py:144] Backend type: vllm
The argument `trust_remote_code` is to be used with Auto classes. It has no effect here and is ignored.
(EngineCore_DP0 pid=253) INFO 2026-01-30 13:11:39.808 cuda.py:398] Using Triton backend on V1 engine.
Loading safetensors checkpoint shards:   0% Completed | 0/3 [00:00<?, ?it/s]
Loading safetensors checkpoint shards:  33% Completed | 1/3 [00:00<00:00,  2.81it/s]
Loading safetensors checkpoint shards:  67% Completed | 2/3 [00:00<00:00,  2.65it/s]
Loading safetensors checkpoint shards: 100% Completed | 3/3 [00:01<00:00,  2.77it/s]
Loading safetensors checkpoint shards: 100% Completed | 3/3 [00:01<00:00,  2.75it/s]
(EngineCore_DP0 pid=253) 
(EngineCore_DP0 pid=253) WARNING 2026-01-30 13:11:49.273 marlin_utils_fp4.py:204] Your GPU does not have native support for FP4 computation but FP4 quantization is being used. Weight-only FP4 compression will be used leveraging the Marlin kernel. This may degrade performance for compute-heavy workloads.
Capturing CUDA graphs (mixed prefill-decode, PIECEWISE): 100%|██████████| 81/81 [00:03<00:00, 27.00it/s]
Capturing CUDA graphs (decode, FULL): 100%|██████████| 19/19 [00:03<00:00,  5.73it/s]
[2026-01-30 13:12:23] INFO on.py:62: Application startup complete.
[2026-01-30 13:12:23] INFO server.py:214: Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```
  
  
Notes:  
- When NIM does involve Triton Inference Server
  
- Some NIMs / profiles are built on TensorRT-LLM and can be deployed behind Triton Inference Server, but that’s a different image/profile path than what you’re running here (your logs explicitly say vLLM).

