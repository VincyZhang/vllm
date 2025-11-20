This is a quick tutorial about how to deploy PD disaggregation on Intel XPU based on Kubernetes, and utilize nixl with UCX backedn.

# Setup up Pods in Kubernete Cluster

`ucx-test.yaml` file should be like below example, use `kubectl apply -f ucx-test.yaml` to start prefil pod and decode pod within the same cluster and same namespace.

```bash
# Intel XPU configuration for PD Disaggregation
# This configuration sets up prefill and decode services with Intel XPU optimization
apiVersion: v1
kind: Pod
metadata:
  name: ucx-test-decode
  namespace: ucx-test
spec:
  restartPolicy: Never
  containers:
    - name: ucx-test-decode
      image: ghcr.io/llm-d/llm-d-xpu-dev:pr-357
      command: ["/bin/bash"]
      tty: true
      stdin: true
      securityContext:
        privileged: true
      volumeMounts:
        - name: xpu-device
          mountPath: /dev/dri
      env:
        - name: UCX_TLS
          value: "all"
        - name: VLLM_NIXL_SIDE_CHANNEL_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: VLLM_NIXL_SIDE_CHANNEL_PORT
          value: "5557"
        - name: http_proxy
          value: "http://proxy-dmz.intel.com:912"
        - name: https_proxy
          value: "http://proxy-dmz.intel.com:912"
        - name: no_proxy
          value: "localhost,127.0.0.1,intel.com,.intel.com,goto.intel.com,.maas,10.0.0.0/8,172.16.0.0/16,192.168.0.0/16,134.134.0.0/16,.maas-internal,.svc"
      ports:
        - containerPort: 8200
          name: metrics
          protocol: TCP
      resources:
        limits:
          memory: 64Gi
          cpu: "16"
          gpu.intel.com/i915: "1"
        requests:
          memory: 64Gi
          cpu: "16"
  volumes:
    - name: xpu-device
      hostPath:
        path: /dev/dri
        type: Directory
---
apiVersion: v1
kind: Pod
metadata:
  name: ucx-test-prefil
  namespace: ucx-test
spec:
  restartPolicy: Never
  containers:
    - name: ucx-test-prefil
      image: ghcr.io/llm-d/llm-d-xpu-dev:pr-357
      command: ["/bin/bash"]
      tty: true
      stdin: true
      securityContext:
        privileged: true
      volumeMounts:
        - name: xpu-device
          mountPath: /dev/dri
      env:
        - name: UCX_TLS
          value: "all"
        - name: VLLM_NIXL_SIDE_CHANNEL_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: VLLM_NIXL_SIDE_CHANNEL_PORT
          value: "5557"
        - name: http_proxy
          value: "http://proxy-dmz.intel.com:912"
        - name: https_proxy
          value: "http://proxy-dmz.intel.com:912"
        - name: no_proxy
          value: "localhost,127.0.0.1,intel.com,.intel.com,goto.intel.com,.maas,10.0.0.0/8,172.16.0.0/16,192.168.0.0/16,134.134.0.0/16,.maas-internal,.svc"
      ports:
        - containerPort: 8300
          name: metrics
          protocol: TCP
      resources:
        limits:
          memory: 64Gi
          cpu: "16"
          gpu.intel.com/i915: "1"
        requests:
          memory: 64Gi
          cpu: "16"
  volumes:
    - name: xpu-device
      hostPath:
        path: /dev/dri
        type: Directory
```

And use `kubectl get pods -n test` to check if the pods are running successfully.

# Install UCX and Nixl from Source

Enter prefil pod and decode pod separately to install dependencies.
>>**Note**
>> If you are using P2P with GDR, need to apply patch based on a certain commit, `export NIXL_VERSION=0.5.1` before running installation.
>> If you are using P2P without GDR, just `export NIXL_VERSION=0.7.0`


```bash
kubectl exec -it ucx-test-decode -n test -- /bin/bash
cd /workspace
git clone https://github.com/VincyZhang/vllm.git vllm-forked
cd vllm-forked && git checkout wenxin/test_ucx_nixl
cd tools && python install_nixl_from_source_ubuntu.py
```

# Start vLLM service

Use `--kv-transfer-config '{"kv_connector":"NixlConnector","kv_role":"kv_both", "kv_buffer_device":"xpu"}'` to enable P2P transfer. Start prefil and decode service separately, the port should be the same as defined in `ucx-test.yaml`.

```bash
vllm serve Qwen/Qwen3-8B --port 8300 --block-size 128 --disable-log-requests --disable-uvicorn-access-log --enforce-eager --kv-transfer-config '{"kv_connector":"NixlConnector","kv_role":"kv_both", "kv_buffer_device":"xpu"}'
```

# Start PD Disaggregation Proxy

First need to check the IP for prefil and decode pod by `kubectl get pod -o wide -n test`. And the IP and port will be used to enable proxy.

```bash
cd /workspace/vllm
python examples/online_serving/disaggregated_serving/disagg_proxy_demo.py --model Qwen/Qwen3-8B --prefill $P_IP:8300 --decode $D_IP$:8200 --port 8192
```

# Benchmarking
 In this example, we use sonnet dataset to do a quick benchmarking
```bash
python benchmarks/benchmark_serving.py --backend vllm --model Qwen/Qwen3-8B --trust-remote-code --port 8192 --dataset-path benchmarks/sonnet.txt  --dataset-name sonnet --sonnet-input-len 2048 --sonnet-output-len 2048  --num-prompts 100 --request-rate inf --seed 0 --ignore_eos --save-result --result-filename synapsellm-sonnet.json --host localhost --max-concurrency 32
```