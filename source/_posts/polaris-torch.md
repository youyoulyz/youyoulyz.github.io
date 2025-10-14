---
title: '在AMD RX580(gfx803)上借助旧版ROCm运行GPT-2模型'
date: 2025-10-13 11:00:00
tags:
  - LLM
  - ROCm
  - PyTorch
  - AMD
  - RX580

categories:
  - AMD
---

Thanks:  https://github.com/nikos230/Run-Pytorch-with-AMD-Radeon-GPU


![实施图片](images/polaris-torch-1.png)

### 碎碎念

跟着教程里的脚步走，要注意先把原来所有的hip和rocm软件全部卸载

注意这个实验解除了以后要手动去掉`/etc/environment`的相关配置
```plain text
sudo echo ROC_ENABLE_PRE_VEGA=1 >> /etc/environment
sudo echo HSA_OVERRIDE_GFX_VERSION=8.0.3 >> /etc/environment
```

装完以后ubuntu桌面挂了...修了半天没修好不管了。

可以装个amdgpu_top https://github.com/Umio-Yasuno/amdgpu_top 用来好好看负载占用，比较神奇的是588也能用

### 用来inference的code
```python
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM

# --- 1. 设置模型和设备 ---
# 这里我们选择经典的、无需登录的 gpt2-medium 模型
# 第一次运行时会自动下载（约1.5GB）
model_name = "gpt2-large"

# 检查是否有可用的 GPU (NVIDIA)，否则使用 CPU
device = "cuda" if torch.cuda.is_available() else "cpu"
print(f"正在使用设备: {device}")

# --- 2. 加载 Tokenizer 和模型 ---
print("正在加载 Tokenizer...")
tokenizer = AutoTokenizer.from_pretrained(model_name)

# GPT-2 模型没有官方的 pad_token，但为了避免警告，我们可以将其设置为 eos_token
tokenizer.pad_token = tokenizer.eos_token

print("正在加载模型...")
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    device_map="auto" # 自动将模型加载到硬件上
)
# 注意：GPT-2 这种老模型通常不需要设置 torch_dtype

# --- 3. 准备输入 ---
# GPT-2 是一个续写模型，主要用英文训练，所以我们给一个英文的开头
prompt_text = "Shanghai is one of the most populous cities in the world, located in China. It is famous for"

# 直接将文本编码成 PyTorch 张量 (tensor)
inputs = tokenizer(prompt_text, return_tensors="pt").to(device)


# --- 4. 生成回答 ---
print("正在生成回答...")
# 使用 model.generate() 方法进行推理
# max_length 控制生成的总文本长度（包括你的输入）
outputs = model.generate(
    **inputs,
    max_length=100,  # 生成的总长度
    num_return_sequences=1,
    pad_token_id=tokenizer.eos_token_id # 设置 pad_token_id 来避免警告
)

# --- 5. 解码并输出结果 ---
# 将模型输出的数字 ID 解码成人类可读的文本
result_text = tokenizer.decode(outputs[0], skip_special_tokens=True)

print("\n--- 模型生成结果 ---")
print(result_text)
```

### 可能需要的pip list
```plain text
absl-py==2.1.0
accelerate==1.10.1
addict==2.4.0
aiofiles==23.1.0
ajsonrpc==1.2.0
ament-cmake-test==1.3.11
ament-index-python==1.4.0
ament-package==0.14.0
anyio==3.6.2
asttokens==2.4.1
attrs==23.2.0
autopep8==2.0.1
black==24.3.0
blinker==1.7.0
bottle==0.12.25
cachetools==5.3.3
certifi==2025.10.5
charset-normalizer==3.4.3
click==8.1.3
comm==0.2.2
ConfigArgParse==1.7
coolgpus==0.23
cycler==0.12.1
dash==2.16.1
dash-core-components==2.0.0
dash-html-components==2.0.0
dash-table==5.0.0
descartes==1.1.0
easydict==1.13
exceptiongroup==1.2.0
executing==2.0.1
fastjsonschema==2.19.1
filelock==3.20.0
fire==0.6.0
Flask==3.0.2
fonttools==4.60.1
fsspec==2025.9.0
grpcio==1.62.1
h11==0.14.0
hf-xet==1.1.10
huggingface-hub==0.35.3
idna==3.11
imageio==2.34.0
importlib_metadata==7.0.2
ipython==8.22.2
ipywidgets==8.1.2
itsdangerous==2.1.2
jedi==0.19.1
Jinja2==3.1.6
joblib==1.3.2
jsonschema==4.21.1
jsonschema-specifications==2023.12.1
jupyter_core==5.7.2
jupyterlab_widgets==3.0.10
kiwisolver==1.4.9
lazy_loader==0.3
llvmlite==0.42.0
lyft-dataset-sdk==0.0.8
markdown-it-py==3.0.0
MarkupSafe==2.1.5
marshmallow==3.19.0
matplotlib==3.5.3
matplotlib-inline==0.1.6
mdurl==0.1.2
mmdet3d==1.4.0
mmengine==0.10.3
mpmath==1.3.0
mypy-extensions==1.0.0
nbformat==5.10.3
nest-asyncio==1.6.0
networkx==3.4.2
numba==0.59.1
numpy==1.26.4
nuscenes-devkit==1.1.11
nvidia-cublas-cu11==11.10.3.66
nvidia-cuda-nvrtc-cu11==11.7.99
nvidia-cuda-runtime-cu11==11.7.99
nvidia-cudnn-cu11==8.5.0.96
open3d==0.18.0
opencv-python==4.9.0.80
packaging==25.0
pandas==2.2.1
parso==0.8.3
pathspec==0.12.1
pillow==10.2.0
platformdirs==4.2.0
platformio==6.1.7
plotly==5.20.0
pluggy==1.5.0
plyfile==1.0.3
prompt-toolkit==3.0.43
protobuf==4.25.3
psutil==7.1.0
pure-eval==0.2.2
pycocotools==2.0.7
pycodestyle==2.10.0
pyelftools==0.29
Pygments==2.17.2
pyparsing==3.2.5
pypcd==0.1.1
pyquaternion==0.9.9
pyserial==3.5
pytest==8.0.0
pytest-cover==3.0.0
pytest-html==4.1.1
pytest-metadata==3.1.1
python-dateutil==2.9.0.post0
python-lzf==0.2.4
pytz==2025.2
PyYAML==6.0.3
pyzmq==26.0.3
rcutils==5.1.6
referencing==0.34.0
regex==2025.9.18
requests==2.32.5
retrying==1.3.4
rich==13.7.1
rosidl-adapter==3.1.6
rosidl-cli==3.1.6
rpds-py==0.18.0
safetensors==0.6.2
scikit-image==0.22.0
scikit-learn==1.4.1.post1
scipy==1.15.3
seaborn==0.13.2
semantic-version==2.10.0
Shapely==1.8.5.post1
six==1.17.0
sniffio==1.3.0
stack-data==0.6.3
starlette==0.26.1
sympy==1.14.0
tabulate==0.9.0
tenacity==8.2.3
tensorboard==2.16.2
tensorboard-data-server==0.7.2
tensorboardX==2.6.2.2
termcolor==2.4.0
threadpoolctl==3.3.0
tifffile==2024.2.12
tokenizers==0.15.2
tomli==2.0.1
torch @ file:///home/youyoulyz/rocm/torch-2.1.1-cp310-cp310-linux_x86_64.whl#sha256=5a14a80cdc6608f33fd9d8d413be5400c1aec5eb82870770fd830863b6c92fe5
torchaudio==0.13.1
torchvision==0.14.1
tqdm==4.66.2
traitlets==5.14.2
transformers==4.36.2
trimesh==4.2.0
typing_extensions==4.5.0
tzdata==2024.1
urllib3==2.5.0
uvicorn==0.22.0
wcwidth==0.2.13
websockets==12.0
Werkzeug==3.0.1
widgetsnbextension==4.0.10
wsproto==1.2.0
yapf==0.40.2
zmq==0.0.0
```


### idea
也许可以给MI50/100 5700XT重新编译一版torch？感觉应该只是改环境变量的问题。

接下来会试试MI50/5700XT/6800XT这些老卡的推理支持

### 总结
纯toy，因为transformer的版本到4.36.2，https://pypi.org/project/transformers/4.36.2/  没有新模型可以用，chat的最高就是llama-2，缺乏实用性