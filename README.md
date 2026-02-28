## for gpt-oss-20b  
- OS: Fedora Linux 43 (Workstation Edition) x86_64
- Kernel: Linux 6.18.9-200.fc43.x86_64
- GPU 1: NVIDIA GeForce RTX 4090
- Profiling list and NGC info in [line text](https://github.com/kagaho/NVIDIA-NIM-Playground/blob/main/NIM_NGC_profiles.md "NIM_NGC_profiles.md")



- https://build.nvidia.com/openai/gpt-oss-20b/deploy
  
```
podman run -d   --name nim-gpt-oss-20b   --restart=unless-stopped   --device nvidia.com/gpu=all   --shm-size=16GB   -e NGC_API_KEY   -e NIM_MODEL_PROFILE="66fb3113efd2aae1b0a3bfa2a375de5fe1cc1b557abac4eb271730482a26ae8e"   -e VLLM_MAX_MODEL_LEN=8192   -e VLLM_MAX_NUM_SEQS=4   -e VLLM_GPU_MEMORY_UTILIZATION=0.80   -e PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True   -v "$LOCAL_NIM_CACHE:/opt/nim/.cache:Z,U"   -p 8000:8000   nvcr.io/nim/openai/gpt-oss-20b:latest
```
  
```bash
curl -X 'POST' \
'http://0.0.0.0:8000/v1/chat/completions' \
-H 'accept: application/json' \
-H 'Content-Type: application/json' \
-d '{
    "model": "openai/gpt-oss-20b",
    "messages": [{"role":"user", "content":"Which number is larger, 9.11 or 9.8?"}],
    "max_tokens": 64
}' | jq
```
  
- Same as above, but filtering the fields you want with "jq" tool:
  
```bash
curl -X 'POST' 'http://0.0.0.0:8000/v1/chat/completions' -H 'accept: application/json' -H 'Content-Type: application/json' -d '{
    "model": "openai/gpt-oss-20b",
    "messages": [{"role":"user", "content":"Which number is larger, 9.11 or 9.8?"}],
    "max_tokens": 64
}'| jq '{content: .choices[0].message.content, reasoning: .choices[0].message.reasoning_content}'
```
  
- print nicely

```bash
curl -sS   -X POST 'http://0.0.0.0:8000/v1/chat/completions'   -H 'accept: application/json'   -H 'Content-Type: application/json'   -d '{
    "model": "openai/gpt-oss-20b",
    "messages": [{"role":"user","content":"create a go script to calcule factorial of a number. The script send a promptto user, asking the number, if the numberisnegative or not a number, we send a message: DONT FOOL ME."}],
    "max_tokens": 10500
  }' | jq -r '.choices[0].message.content'
Here’s a small, self‑contained Go program that does exactly what you asked for:

```go
// factorial.go
package main

import (
	"bufio"
	"fmt"
	"math/big"
	"os"
	"strconv"
	"strings"
)

// factorial returns n! (for n >= 0) as a *big.Int
func factorial(n int64) *big.Int {
	res := big.NewInt(1)
	for i := int64(2); i <= n; i++ {
		res.Mul(res, big.NewInt(i))
	}
	return res
}

func main() {
	reader := bufio.NewReader(os.Stdin)
	fmt.Print("Enter a non‑negative integer: ")
	line, err := reader.ReadString('\n')
	if err != nil {
		fmt.Println("DONT FOOL ME")
		return
	}

	line = strings.TrimSpace(line)
	if line == "" {
		fmt.Println("DONT FOOL ME")
		return
	}

	num, err := strconv.ParseInt(line, 10, 64)
	if err != nil || num < 0 {
		fmt.Println("DONT FOOL ME")
		return
	}

	fmt.Printf("Factorial of %d is %s\n", num, factorial(num))
}
```

### How to run

```bash
go run factorial.go
```

It prompts you for an integer.  
- If you enter a negative number or something that isn’t an integer, you’ll see `DONT FOOL ME`.  
- For a valid non‑negative integer it prints the factorial (using `math/big` to avoid overflow).  
  

### Track details on the NIM container:
  
```  
❯ curl -sS http://0.0.0.0:8000/v1/models | jq
{
  "object": "list",
  "data": [
    {
      "id": "openai/gpt-oss-20b",
      "object": "model",
      "created": 1770049265,
      "owned_by": "system",
      "root": "/opt/nim/workspace",
      "parent": null,
      "max_model_len": 131072,
      "permission": [
        {
          "id": "modelperm-ff9365e5136743bebdcbbc4e8aec6bac",
          "object": "model_permission",
          "created": 1770049265,
          "allow_create_engine": false,
          "allow_sampling": true,
          "allow_logprobs": true,
          "allow_search_indices": false,
          "allow_view": true,
          "allow_fine_tuning": false,
          "organization": "*",
          "group": null,
          "is_blocking": false
        }
      ]
    }
  ]
}
```

### check container level status:
```bash
curl -sS -i http://0.0.0.0:8000/v1/health/ready
HTTP/1.1 200 OK
content-length: 71
content-type: application/json
```

  
### Prompt and Answer token , pending requests:
```
❯ curl -s http://0.0.0.0:8000/v1/metrics   | grep -E '^vllm:(prompt_tokens_total|generation_tokens_total)\{'   | head
vllm:prompt_tokens_total{engine="0",model_name="openai/gpt-oss-20b"} 1939.0
vllm:generation_tokens_total{engine="0",model_name="openai/gpt-oss-20b"} 1460.0
```

```
❯ curl -s http://0.0.0.0:8000/v1/metrics \
  | awk -F' ' '/^vllm:num_requests_waiting\{/{print $2}' \
  | head -n1
0.0
```

### All prometheus metrics:
- curl -sS http://0.0.0.0:8000/v1/metrics



