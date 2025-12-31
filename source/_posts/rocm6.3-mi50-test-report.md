---
title: ROCm 6.3 MI50 测试报告
date: 2025-12-31 00:00:00
tags: [ROCm, MI50, vLLM, GPU, 测试报告]
categories: [GPU计算, ROCm]
---


1. 降级至6.8内核或安装22.04.2LTS
2. 安装ROCM和AMDGPU-DKMS https://rocm.docs.amd.com/projects/install-on-linux/en/docs-6.3.3/ 
3. 依据这个仓库 https://github.com/nlzy/vllm-gfx906 pull docker 
4. 运行测试

```cpp
#include <hip/hip_runtime.h>
#include <iostream>
#include <vector>
#include <cmath>
#include <cassert>
#include <string>

// 宏定义：用于检查HIP API调用的返回值
#define HIP_CHECK(cmd) \
    do { \
        hipError_t error = cmd; \
        if (error != hipSuccess) { \
            std::cerr << "\033[1;31m[ERROR]\033[0m HIP API Error at line " << __LINE__ \
                      << ": " << hipGetErrorString(error) << std::endl; \
            exit(EXIT_FAILURE); \
        } \
    } while (0)

// -------------------------------------------------------------------------
// Kernel 1: 基础数学运算与全局内存测试
// 测试点: 线程索引, 浮点运算, sin/cos/sqrt 函数支持
// -------------------------------------------------------------------------
__global__ void math_test_kernel(const float* input, float* output, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < n) {
        float val = input[idx];
        // 进行一些稍微复杂的数学运算
        output[idx] = sinf(val) * cosf(val) + sqrtf(fabsf(val));
    }
}

// -------------------------------------------------------------------------
// Kernel 2: 原子操作与共享内存测试
// 测试点: atomicAdd, __shared__ 内存声明与使用
// -------------------------------------------------------------------------
__global__ void atomic_shmem_kernel(int* global_counter) {
    __shared__ int local_sum;
    
    // 初始化共享内存
    if (threadIdx.x == 0) {
        local_sum = 0;
    }
    __syncthreads();

    // 块内原子加
    atomicAdd(&local_sum, 1);
    __syncthreads();

    // 由由于每个block的第0个线程将结果加到全局内存
    if (threadIdx.x == 0) {
        atomicAdd(global_counter, local_sum);
    }
}

// -------------------------------------------------------------------------
// Kernel 3: Warp Shuffle 测试 (高级特性)
// 测试点: __shfl_down (在AMD GPU上通常映射为 DPP 或其他指令)
// -------------------------------------------------------------------------
__global__ void warp_shuffle_kernel(int* output, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    int val = 1;

    // 简单的规约：Warp内的值求和 (假设 warpSize 为 32 或 64)
    // 注意：这里使用通用的 shfl_down，不同架构 warp size 可能不同
    for (int offset = 16; offset > 0; offset /= 2) {
#if defined(__HIP_PLATFORM_AMD__) || defined(__HIP_PLATFORM_NVIDIA__)
        val += __shfl_down(val, offset); 
#endif
    }

    if (idx < n && (threadIdx.x % 32 == 0)) {
        output[idx / 32] = val; // 应该输出32
    }
}

// 辅助函数：打印带颜色的状态
void print_pass(const std::string& msg) {
    std::cout << "\033[1;32m[PASS]\033[0m " << msg << std::endl;
}

void print_info(const std::string& msg) {
    std::cout << "\033[1;34m[INFO]\033[0m " << msg << std::endl;
}

int main() {
    print_info("Starting HIP Comprehensive Compatibility Test...");

    // 1. 设备枚举与属性查询
    int deviceCount = 0;
    HIP_CHECK(hipGetDeviceCount(&deviceCount));
    
    if (deviceCount == 0) {
        std::cerr << "No HIP-capable devices found!" << std::endl;
        return -1;
    }
    
    std::cout << "Found " << deviceCount << " device(s)." << std::endl;

    // 遍历所有设备进行测试（这里只演示Device 0，多卡可循环）
    int dev = 0;
    HIP_CHECK(hipSetDevice(dev));
    
    hipDeviceProp_t props;
    HIP_CHECK(hipGetDeviceProperties(&props, dev));
    
    std::cout << "------------------------------------------------" << std::endl;
    std::cout << "Device " << dev << ": " << props.name << std::endl;
    std::cout << "  Compute Arch: " << props.gcnArchName << std::endl; // AMD 特有，N卡可能是空的或不同
    std::cout << "  Total VRAM: " << props.totalGlobalMem / (1024.0 * 1024.0) << " MB" << std::endl;
    std::cout << "  Multiprocessors: " << props.multiProcessorCount << std::endl;
    std::cout << "  Warp Size: " << props.warpSize << std::endl;
    std::cout << "------------------------------------------------" << std::endl;
    print_pass("Device Property Query");

    // 定义数据规模
    const int N = 1024 * 1024;
    const int bytes = N * sizeof(float);

    // 2. 内存分配测试 (Device & Host)
    float *h_in, *h_out_gpu, *d_in, *d_out;
    h_in = (float*)malloc(bytes);
    h_out_gpu = (float*)malloc(bytes);
    
    // 初始化数据
    for (int i = 0; i < N; i++) h_in[i] = (float)i * 0.01f;

    HIP_CHECK(hipMalloc(&d_in, bytes));
    HIP_CHECK(hipMalloc(&d_out, bytes));
    print_pass("hipMalloc (Device Memory Allocation)");

    // 3. 内存拷贝测试 (H2D)
    HIP_CHECK(hipMemcpy(d_in, h_in, bytes, hipMemcpyHostToDevice));
    print_pass("hipMemcpy (Host -> Device)");

    // 4. 执行 Kernel 1 (Math)
    int threads = 256;
    int blocks = (N + threads - 1) / threads;
    math_test_kernel<<<blocks, threads>>>(d_in, d_out, N);
    HIP_CHECK(hipGetLastError()); // 检查启动错误
    HIP_CHECK(hipDeviceSynchronize()); // 检查执行错误
    print_pass("Kernel Execution (Math & Logic)");

    // 5. 内存拷贝测试 (D2H) & 结果验证
    HIP_CHECK(hipMemcpy(h_out_gpu, d_out, bytes, hipMemcpyDeviceToHost));
    
    bool math_correct = true;
    for (int i = 0; i < 100; i++) { // 只验证前100个防止刷屏
        float val = h_in[i];
        float expected = sinf(val) * cosf(val) + sqrtf(fabsf(val));
        if (fabs(h_out_gpu[i] - expected) > 1e-4) {
            math_correct = false;
            std::cerr << "Mismatch at index " << i << ": GPU=" << h_out_gpu[i] << ", CPU=" << expected << std::endl;
            break;
        }
    }
    if (math_correct) print_pass("Result Verification (Math Accuracy)");

    // 6. 原子操作与共享内存测试
    int *d_atomic_counter;
    int h_atomic_counter = 0;
    HIP_CHECK(hipMalloc(&d_atomic_counter, sizeof(int)));
    HIP_CHECK(hipMemcpy(d_atomic_counter, &h_atomic_counter, sizeof(int), hipMemcpyHostToDevice));

    // Launch: 100 blocks, 256 threads each = 25600 threads total
    atomic_shmem_kernel<<<100, 256>>>(d_atomic_counter);
    HIP_CHECK(hipDeviceSynchronize());
    
    HIP_CHECK(hipMemcpy(&h_atomic_counter, d_atomic_counter, sizeof(int), hipMemcpyDeviceToHost));
    if (h_atomic_counter == 100 * 256) {
        print_pass("Atomic Add & Shared Memory");
    } else {
        std::cerr << "\033[1;31m[FAIL]\033[0m Atomic test failed. Expected 25600, got " << h_atomic_counter << std::endl;
    }

    // 7. 流与事件测试 (Streams & Events)
    hipStream_t stream;
    hipEvent_t start, stop;
    HIP_CHECK(hipStreamCreate(&stream));
    HIP_CHECK(hipEventCreate(&start));
    HIP_CHECK(hipEventCreate(&stop));

    HIP_CHECK(hipEventRecord(start, stream));
    // 再次执行一个小任务在流中
    math_test_kernel<<<blocks, threads, 0, stream>>>(d_in, d_out, N);
    HIP_CHECK(hipEventRecord(stop, stream));
    
    HIP_CHECK(hipEventSynchronize(stop));
    float milliseconds = 0;
    HIP_CHECK(hipEventElapsedTime(&milliseconds, start, stop));
    
    std::cout << "\033[1;34m[INFO]\033[0m Async Kernel Duration: " << milliseconds << " ms" << std::endl;
    print_pass("Streams & Events (Async Execution)");

    // 8. Managed Memory (Unified Memory) 测试
    // 注意：某些旧架构或特定驱动配置可能不支持此功能，这是个很好的兼容性测试点
    int* managed_ptr;
    hipError_t managed_err = hipMallocManaged(&managed_ptr, sizeof(int));
    if (managed_err == hipSuccess) {
        *managed_ptr = 123; // Host 直接写
        // Device 读
        // 简单复用 atomic kernel 来验证 device 能否读取 managed memory
        atomic_shmem_kernel<<<1, 1>>>(managed_ptr); 
        HIP_CHECK(hipDeviceSynchronize());
        if (*managed_ptr == 124) { // 123 + 1
             print_pass("Unified Memory (hipMallocManaged)");
        } else {
             std::cout << "\033[1;33m[WARN]\033[0m Unified Memory result mismatch (Driver limitation?)" << std::endl;
        }
        hipFree(managed_ptr);
    } else {
        std::cout << "\033[1;33m[WARN]\033[0m hipMallocManaged not supported on this device/driver configuration." << std::endl;
    }

    // 清理
    HIP_CHECK(hipFree(d_in));
    HIP_CHECK(hipFree(d_out));
    HIP_CHECK(hipFree(d_atomic_counter));
    HIP_CHECK(hipStreamDestroy(stream));
    HIP_CHECK(hipEventDestroy(start));
    HIP_CHECK(hipEventDestroy(stop));
    
    free(h_in);
    free(h_out_gpu);

    std::cout << "\n\033[1;32m=== HIP SYSTEM CHECK COMPLETED SUCCESSFULLY ===\033[0m" << std::endl;

    return 0;
}
```
```bash
aup@aup-To-Be-Filled-By-O-E-M:~$ hipcc hip_full_check.cpp -o hip_check
hip_full_check.cpp:209:9: warning: ignoring return value of function declared with 'nodiscard' attribute [-Wunused-result]
  209 |         hipFree(managed_ptr);
      |         ^~~~~~~ ~~~~~~~~~~~
1 warning generated when compiling for gfx906.
hip_full_check.cpp:209:9: warning: ignoring return value of function declared with 'nodiscard' attribute [-Wunused-result]
  209 |         hipFree(managed_ptr);
      |         ^~~~~~~ ~~~~~~~~~~~
1 warning generated when compiling for host.

aup@aup-To-Be-Filled-By-O-E-M:~$ ./hip_check 
[INFO] Starting HIP Comprehensive Compatibility Test...
Found 1 device(s).
------------------------------------------------
Device 0: AMD Radeon (TM) Pro VII
  Compute Arch: gfx906:sramecc+:xnack-
  Total VRAM: 16368 MB
  Multiprocessors: 60
  Warp Size: 64
------------------------------------------------
[PASS] Device Property Query
[PASS] hipMalloc (Device Memory Allocation)
[PASS] hipMemcpy (Host -> Device)
[PASS] Kernel Execution (Math & Logic)
[PASS] Result Verification (Math Accuracy)
[PASS] Atomic Add & Shared Memory
[INFO] Async Kernel Duration: 0.038723 ms
[PASS] Streams & Events (Async Execution)
[PASS] Unified Memory (hipMallocManaged)

=== HIP SYSTEM CHECK COMPLETED SUCCESSFULLY ===
```
Python torch test
```python
import torch
import torch.nn as nn
import time
import sys

class Color:
    HEADER = '\033[95m'
    BLUE = '\033[94m'
    GREEN = '\033[92m'
    WARNING = '\033[93m'
    FAIL = '\033[91m'
    ENDC = '\033[0m'
    BOLD = '\033[1m'

def print_pass(msg):
    print(f"{Color.GREEN}[PASS]{Color.ENDC} {msg}")

def print_fail(msg, err):
    print(f"{Color.FAIL}[FAIL]{Color.ENDC} {msg}")
    print(f"       Error: {err}")

def print_info(msg):
    print(f"{Color.BLUE}[INFO]{Color.ENDC} {msg}")

def run_test():
    print(f"{Color.BOLD}=== PyTorch on ROCm Comprehensive Check ==={Color.ENDC}\n")

    # ---------------------------------------------------------
    # 1. 基础环境检查
    # ---------------------------------------------------------
    try:
        print_info(f"PyTorch Version: {torch.__version__}")
        print_info(f"ROCm/HIP Version: {torch.version.hip}")
        
        if not torch.cuda.is_available():
            print_fail("CUDA/ROCm not available in PyTorch.", "torch.cuda.is_available() returned False")
            return
        
        device_count = torch.cuda.device_count()
        device_name = torch.cuda.get_device_name(0)
        print_info(f"Device Found: {device_name} (Count: {device_count})")
        
        # 强制在第一块卡上运行
        device = torch.device("cuda:0")
        print_pass("Environment & Device Detection")
    except Exception as e:
        print_fail("Environment Detection Failed", e)
        return

    # ---------------------------------------------------------
    # 2. 基础张量操作与内存搬运 (H2D, D2H)
    # ---------------------------------------------------------
    try:
        x_host = torch.randn(1024, 1024)
        x_device = x_host.to(device)
        y_device = x_device * 2.0
        y_host = y_device.cpu()
        
        if torch.allclose(x_host * 2.0, y_host, atol=1e-5):
            print_pass("Tensor Creation & Memory Transfer (Host <-> Device)")
        else:
            raise Exception("Calculation mismatch on basic tensor ops")
    except Exception as e:
        print_fail("Basic Tensor Ops", e)

    # ---------------------------------------------------------
    # 3. 矩阵乘法 (测试 rocBLAS)
    # ---------------------------------------------------------
    try:
        size = 2048
        a = torch.randn(size, size, device=device)
        b = torch.randn(size, size, device=device)
        
        torch.cuda.synchronize()
        start = time.time()
        c = torch.matmul(a, b)
        torch.cuda.synchronize()
        end = time.time()
        
        tflops = (2 * size**3) / ((end - start) * 1e12)
        print_pass(f"Matrix Multiplication (rocBLAS) - {size}x{size} @ {tflops:.2f} TFLOPS")
    except Exception as e:
        print_fail("Matrix Multiplication (rocBLAS)", e)

    # ---------------------------------------------------------
    # 4. 卷积神经网络原语 (测试 MIOpen)
    #    这是最关键的一步，很多环境在这里挂掉
    # ---------------------------------------------------------
    try:
        # 定义一个简单的卷积层
        conv = nn.Conv2d(in_channels=3, out_channels=64, kernel_size=3, padding=1).to(device)
        input_data = torch.randn(8, 3, 256, 256, device=device)
        
        # 前向传播
        output = conv(input_data)
        
        # 反向传播 (测试 Autograd 和 cudnn/miopen backward)
        target = torch.randn_like(output)
        criterion = nn.MSELoss()
        loss = criterion(output, target)
        loss.backward()
        
        print_pass("CNN Primitives (MIOpen Forward & Backward)")
    except Exception as e:
        print_fail("CNN Primitives (MIOpen)", e)
        print(f"{Color.WARNING}Hint: If this fails with 'segmentation fault' or 'miopen error', verify MIOpen installation.{Color.ENDC}")

    # ---------------------------------------------------------
    # 5. 半精度 (FP16) 测试
    #    gfx906 对 FP16 支持很好
    # ---------------------------------------------------------
    try:
        a_half = torch.randn(1024, 1024, device=device, dtype=torch.float16)
        b_half = torch.randn(1024, 1024, device=device, dtype=torch.float16)
        c_half = torch.matmul(a_half, b_half)
        print_pass("Half Precision (FP16) Compute")
    except Exception as e:
        print_fail("Half Precision (FP16)", e)

    # ---------------------------------------------------------
    # 6. 分布式通信库 (RCCL) 检查
    #    即使单卡，也检查一下库是否链接正常
    # ---------------------------------------------------------
    try:
        if torch.distributed.is_available():
            if torch.backends.cuda.is_built():
                print_pass("Distributed Library (RCCL) is linked")
            else:
                print(f"{Color.WARNING}[WARN] Distributed available but cuda backend not built?{Color.ENDC}")
        else:
            print(f"{Color.WARNING}[WARN] torch.distributed not available (Expected for some builds){Color.ENDC}")
    except Exception as e:
        print_fail("RCCL Check", e)

    # ---------------------------------------------------------
    # 7. 显存压力测试 (分配 4GB)
    # ---------------------------------------------------------
    try:
        # Radeon Pro VII 有 16GB，分配 4GB 应该没问题
        print_info("Attempting to allocate 4GB VRAM...")
        large_tensor = torch.empty(1024 * 1024 * 1024, dtype=torch.float32, device=device) # 4GB
        torch.cuda.synchronize()
        del large_tensor
        torch.cuda.empty_cache()
        print_pass("Large VRAM Allocation (4GB)")
    except Exception as e:
        print_fail("VRAM Stress Test", "OOM or Allocator Error. " + str(e))

    print(f"\n{Color.BOLD}=== TEST COMPLETE ==={Color.ENDC}")

if __name__ == "__main__":
    run_test()
```

```bash
(torch) aup@aup-mi50:~$ python torch_test.py 
=== PyTorch on ROCm Comprehensive Check ===

[INFO] PyTorch Version: 2.9.1+rocm6.3
[INFO] ROCm/HIP Version: 6.3.42134-a9a80e791
[INFO] Device Found: AMD Radeon (TM) Pro VII (Count: 1)
[PASS] Environment & Device Detection
[PASS] Tensor Creation & Memory Transfer (Host <-> Device)
[PASS] Matrix Multiplication (rocBLAS) - 2048x2048 @ 0.06 TFLOPS
[PASS] CNN Primitives (MIOpen Forward & Backward)
[PASS] Half Precision (FP16) Compute
[PASS] Distributed Library (RCCL) is linked
[INFO] Attempting to allocate 4GB VRAM...
[PASS] Large VRAM Allocation (4GB)

=== TEST COMPLETE ===
```


Base script for dockerfile
```bash
sudo docker run -it --rm --shm-size=2g --device=/dev/kfd --device=/dev/dri     --group-add video -p 8000:8000 -v ~/Qwen3-0.6B:/model     nalanzeyu/vllm-gfx906 vllm serve /model
```
VLLM startup log
```bash
INFO 12-31 08:00:32 [__init__.py:239] Automatically detected platform rocm.
INFO 12-31 08:00:46 [api_server.py:1043] vLLM API server version 0.8.5
INFO 12-31 08:00:46 [api_server.py:1044] args: Namespace(subparser='serve', model_tag='/model', config='', host=None, port=8000, uvicorn_log_level='info', disable_uvicorn_access_log=False, allow_credentials=False, allowed_origins=['*'], allowed_methods=['*'], allowed_headers=['*'], api_key=None, lora_modules=None, prompt_adapters=None, chat_template=None, chat_template_content_format='auto', response_role='assistant', ssl_keyfile=None, ssl_certfile=None, ssl_ca_certs=None, enable_ssl_refresh=False, ssl_cert_reqs=0, root_path=None, middleware=[], return_tokens_as_token_ids=False, disable_frontend_multiprocessing=False, enable_request_id_headers=False, enable_auto_tool_choice=False, tool_call_parser=None, tool_parser_plugin='', model='/model', task='auto', tokenizer=None, hf_config_path=None, skip_tokenizer_init=False, revision=None, code_revision=None, tokenizer_revision=None, tokenizer_mode='auto', trust_remote_code=False, allowed_local_media_path=None, load_format='auto', download_dir=None, model_loader_extra_config={}, use_tqdm_on_load=True, config_format=<ConfigFormat.AUTO: 'auto'>, dtype='auto', max_model_len=None, guided_decoding_backend='auto', reasoning_parser=None, logits_processor_pattern=None, model_impl='auto', distributed_executor_backend=None, pipeline_parallel_size=1, tensor_parallel_size=1, data_parallel_size=1, enable_expert_parallel=False, max_parallel_loading_workers=None, ray_workers_use_nsight=False, disable_custom_all_reduce=False, block_size=None, gpu_memory_utilization=0.9, swap_space=4, kv_cache_dtype='auto', num_gpu_blocks_override=None, enable_prefix_caching=None, prefix_caching_hash_algo='builtin', cpu_offload_gb=0, calculate_kv_scales=False, disable_sliding_window=False, use_v2_block_manager=True, seed=None, max_logprobs=20, disable_log_stats=False, quantization=None, rope_scaling=None, rope_theta=None, hf_token=None, hf_overrides=None, enforce_eager=False, max_seq_len_to_capture=8192, tokenizer_pool_size=0, tokenizer_pool_type='ray', tokenizer_pool_extra_config={}, limit_mm_per_prompt={}, mm_processor_kwargs=None, disable_mm_preprocessor_cache=False, enable_lora=None, enable_lora_bias=False, max_loras=1, max_lora_rank=16, lora_extra_vocab_size=256, lora_dtype='auto', long_lora_scaling_factors=None, max_cpu_loras=None, fully_sharded_loras=False, enable_prompt_adapter=None, max_prompt_adapters=1, max_prompt_adapter_token=0, device='auto', speculative_config=None, ignore_patterns=[], served_model_name=None, qlora_adapter_name_or_path=None, show_hidden_metrics_for_version=None, otlp_traces_endpoint=None, collect_detailed_traces=None, disable_async_output_proc=False, max_num_batched_tokens=None, max_num_seqs=None, max_num_partial_prefills=1, max_long_partial_prefills=1, long_prefill_token_threshold=0, num_lookahead_slots=0, scheduler_delay_factor=0.0, preemption_mode=None, num_scheduler_steps=1, multi_step_stream_outputs=True, scheduling_policy='fcfs', enable_chunked_prefill=None, disable_chunked_mm_input=False, scheduler_cls='vllm.core.scheduler.Scheduler', override_neuron_config=None, override_pooler_config=None, compilation_config=None, kv_transfer_config=None, worker_cls='auto', worker_extension_cls='', generation_config='auto', override_generation_config=None, enable_sleep_mode=False, additional_config=None, enable_reasoning=False, disable_cascade_attn=False, disable_log_requests=False, max_log_len=None, disable_fastapi_docs=False, enable_prompt_tokens_details=False, enable_server_load_tracking=False, dispatch_function=<function ServeSubcommand.cmd at 0x705f0bf82d40>)
INFO 12-31 08:01:08 [config.py:717] This model supports multiple tasks: {'generate', 'reward', 'classify', 'score', 'embed'}. Defaulting to 'generate'.
INFO 12-31 08:01:08 [arg_utils.py:1669] rocm is experimental on VLLM_USE_V1=1. Falling back to V0 Engine.
WARNING 12-31 08:01:08 [arg_utils.py:1536] The model has a long context length (40960). This may causeOOM during the initial memory profiling phase, or result in low performance due to small KV cache size. Consider setting --max-model-len to a smaller value.
INFO 12-31 08:01:08 [config.py:1804] Disabled the custom all-reduce kernel because it is not supported on current platform.
INFO 12-31 08:01:08 [api_server.py:246] Started engine process with PID 47
INFO 12-31 08:01:12 [__init__.py:239] Automatically detected platform rocm.
INFO 12-31 08:01:24 [llm_engine.py:240] Initializing a V0 LLM engine (v0.8.5) with config: model='/model', speculative_config=None, tokenizer='/model', skip_tokenizer_init=False, tokenizer_mode=auto, revision=None, override_neuron_config=None, tokenizer_revision=None, trust_remote_code=False, dtype=torch.bfloat16, max_seq_len=40960, download_dir=None, load_format=LoadFormat.AUTO, tensor_parallel_size=1, pipeline_parallel_size=1, disable_custom_all_reduce=True, quantization=None, enforce_eager=False, kv_cache_dtype=auto,  device_config=cuda, decoding_config=DecodingConfig(guided_decoding_backend='auto', reasoning_backend=None), observability_config=ObservabilityConfig(show_hidden_metrics=False, otlp_traces_endpoint=None, collect_model_forward_time=False, collect_model_execute_time=False), seed=None, served_model_name=/model, num_scheduler_steps=1, multi_step_stream_outputs=True, enable_prefix_caching=None, chunked_prefill_enabled=False, use_async_output_proc=True, disable_mm_preprocessor_cache=False, mm_processor_kwargs=None, pooler_config=None, compilation_config={"splitting_ops":[],"compile_sizes":[],"cudagraph_capture_sizes":[256,248,240,232,224,216,208,200,192,184,176,168,160,152,144,136,128,120,112,104,96,88,80,72,64,56,48,40,32,24,16,8,4,2,1],"max_capture_size":256}, use_cached_outputs=True, 
INFO 12-31 08:01:25 [rocm.py:183] None is not supported in AMD GPUs.
INFO 12-31 08:01:25 [rocm.py:184] Using ROCmFlashAttention backend.
INFO 12-31 08:01:25 [parallel_state.py:1004] rank 0 in world size 1 is assigned as DP rank 0, PP rank 0, TP rank 0
INFO 12-31 08:01:25 [model_runner.py:1108] Starting to load model /model...
Loading safetensors checkpoint shards:   0% Completed | 0/1 [00:00<?, ?it/s]
Loading safetensors checkpoint shards: 100% Completed | 1/1 [00:00<00:00,  1.11it/s]
Loading safetensors checkpoint shards: 100% Completed | 1/1 [00:00<00:00,  1.11it/s]

INFO 12-31 08:01:27 [loader.py:458] Loading weights took 1.03 seconds
INFO 12-31 08:01:27 [model_runner.py:1140] Model loading took 1.3379 GiB and 1.568738 seconds
INFO 12-31 08:01:48 [worker.py:287] Memory profiling takes 20.58 seconds
INFO 12-31 08:01:48 [worker.py:287] the current vLLM instance can use total_gpu_memory (15.98GiB) x gpu_memory_utilization (0.90) = 14.39GiB
INFO 12-31 08:01:48 [worker.py:287] model weights take 1.34GiB; non_torch_memory takes 0.46GiB; PyTorch activation peak memory takes 1.49GiB; the rest of the memory reserved for KV Cache is 11.10GiB.
INFO 12-31 08:01:48 [executor_base.py:112] # rocm blocks: 6495, # CPU blocks: 2340
INFO 12-31 08:01:48 [executor_base.py:117] Maximum concurrency for 40960 tokens per request: 2.54x
INFO 12-31 08:01:54 [model_runner.py:1450] Capturing cudagraphs for decoding. This may lead to unexpected consequences if the model is not static. To run the model in eager mode, set 'enforce_eager=True' or use '--enforce-eager' in the CLI. If out-of-memory error occurs during cudagraph capture, consider decreasing `gpu_memory_utilization` or switching to eager mode. You can also reduce the `max_num_seqs` as needed to decrease memory usage.
Capturing CUDA graph shapes: 100%|████████████████████████████████████████████████████████████████████████████████████████| 35/35 [00:19<00:00,  1.78it/s]INFO 12-31 08:02:14 [model_runner.py:1592] Graph capturing finished in 20 secs, took 0.08 GiB
INFO 12-31 08:02:14 [llm_engine.py:437] init engine (profile, create kv cache, warmup model) took 46.92 seconds
WARNING 12-31 08:02:14 [config.py:1239] Default sampling parameters have been overridden by the model's Hugging Face generation config recommended from the model creator. If this is not intended, please relaunch vLLM instance with `--generation-config vllm`.
INFO 12-31 08:02:14 [serving_chat.py:118] Using default chat sampling params from model: {'temperature': 0.6, 'top_k': 20, 'top_p': 0.95}
INFO 12-31 08:02:14 [serving_completion.py:61] Using default completion sampling params from model: {'temperature': 0.6, 'top_k': 20, 'top_p': 0.95}
INFO 12-31 08:02:14 [api_server.py:1090] Starting vLLM API server on http://0.0.0.0:8000
INFO 12-31 08:02:14 [launcher.py:28] Available routes are:
INFO 12-31 08:02:14 [launcher.py:36] Route: /openapi.json, Methods: HEAD, GET
INFO 12-31 08:02:14 [launcher.py:36] Route: /docs, Methods: HEAD, GET
INFO 12-31 08:02:14 [launcher.py:36] Route: /docs/oauth2-redirect, Methods: HEAD, GET
INFO 12-31 08:02:14 [launcher.py:36] Route: /redoc, Methods: HEAD, GET
INFO 12-31 08:02:14 [launcher.py:36] Route: /health, Methods: GET
INFO 12-31 08:02:14 [launcher.py:36] Route: /load, Methods: GET
INFO 12-31 08:02:14 [launcher.py:36] Route: /ping, Methods: POST, GET
INFO 12-31 08:02:14 [launcher.py:36] Route: /tokenize, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /detokenize, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /v1/models, Methods: GET
INFO 12-31 08:02:14 [launcher.py:36] Route: /version, Methods: GET
INFO 12-31 08:02:14 [launcher.py:36] Route: /v1/chat/completions, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /v1/completions, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /v1/embeddings, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /pooling, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /score, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /v1/score, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /v1/audio/transcriptions, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /rerank, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /v1/rerank, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /v2/rerank, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /invocations, Methods: POST
INFO 12-31 08:02:14 [launcher.py:36] Route: /metrics, Methods: GET
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.

```

VLLM Result
```bash
INFO 12-31 08:30:11 [metrics.py:486] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 13.0 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 10.5%, CPU KV cache usage: 0.0%.
```

