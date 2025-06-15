# [复现论文 ](https://papers.nips.cc/paper/7395-evolution-guided-policy-gradient-in-reinforcement-learning.pdf )

**Evolution-Guided Policy Gradient in Reinforcement Learning**

## 1、配置环境

### 1. 创建并激活新环境（Linux）
```bash
# 创建Python 3.8的新环境
conda create --name erl_v0 python=3.8.8
# 激活环境
conda activate erl_v0
```

### 2. 安装核心依赖
```bash
# 安装本地PyTorch 2.0.0（确保在whl文件所在目录执行）(2025/5/12能使用)
pip install torch-2.0.0+cu118-cp38-cp38-win_amd64.whl -i https://pypi.mirrors.ustc.edu.cn/simple

# 安装torchvision和torchaudio（与CUDA 11.8兼容的版本）
pip install torchvision==0.15.1+cu118 torchaudio==2.0.1+cu118 --index-url https://download.pytorch.org/whl/cu118

# 安装Numpy（指定版本以确保兼容性）
conda install numpy=1.24.3

# 安装Gymnasium（新版Gym）
pip install gymnasium[all]  # 安装完整版本，包含所有环境

# 安装Mujoco相关
pip install mujoco
pip install gymnasium[mujoco]  # 特别安装mujoco环境支持

# 安装Tensorboard
pip install tensorboard

# 安装必要的基础包
pip install pandas  # 数据处理
pip install matplotlib  # 绘图
pip install scipy  # 科学计算
```

### 3. 开发和调试工具
```bash
# 安装开发工具
pip install pytest  # 用于测试
pip install black  # 用于代码格式化
pip install flake8  # 用于代码检查
pip install ipython  # 交互式Python环境
```

### 4. 验证安装
```bash
# 验证Python版本
python --version

# 验证PyTorch版本和CUDA可用性
python -c "import torch; print(f'PyTorch version: {torch.__version__}'); print(f'CUDA available: {torch.cuda.is_available()}'); print(f'CUDA version: {torch.version.cuda if torch.cuda.is_available() else None}')"

# 验证Gymnasium
python -c "import gymnasium as gym; print(f'Gymnasium version: {gym.__version__}')"

# 验证Mujoco
python -c "import mujoco; print(f'Mujoco version: {mujoco.__version__}')"

# 验证其他核心包
python -c "import numpy as np; import pandas as pd; import matplotlib.pyplot as plt; print(f'Numpy version: {np.__version__}'); print(f'Pandas version: {pd.__version__}')"
```

### 5. 代码迁移注意事项
1. 将原有的`gym`导入改为`gymnasium`
2. 检查并更新PyTorch API的变化：
   - `torch.FloatTensor` -> `torch.tensor(..., dtype=torch.float32)`
   - `detach().numpy()` -> `detach().cpu().numpy()`
   - 检查模型的`state_dict()`保存和加载方式
3. 更新环境包装器：
   - 更新`gym_wrapper.py`以适应gymnasium的新API
   - 检查环境重置函数的返回值（gymnasium返回tuple(observation, info)）
4. 更新模型构造函数：
   - 利用PyTorch 2.0的编译特性提升性能
   - 更新优化器参数

### 6. 备份恢复方案
如果在升级过程中遇到问题，可以：
1. 删除`erl_v2`环境：`conda remove --name erl_v2 --all`
2. 从`backup_original`文件夹恢复代码：`xcopy /E /I /H /Y backup_original\* .`

### 7. 测试计划
1. 基础环境测试：
   - CartPole-v1（离散动作空间）
   - Pendulum-v1（连续动作空间）
2. Mujoco环境测试：
   - Hopper-v4
   - HalfCheetah-v4
   - Humanoid-v4
3. 性能测试：
   - 单次环境步长时间
   - 训练吞吐量（steps/second）
   - GPU内存使用情况
   - 模型收敛速度对比

### 8. 性能优化建议
1. 使用PyTorch 2.0的编译功能：
```python
model = torch.compile(model)  # 在模型定义后添加
```
2. 使用混合精度训练：
```python
from torch.cuda.amp import autocast, GradScaler
scaler = GradScaler()
```
3. 数据加载优化：
   - 使用`pin_memory=True`
   - 适当的`batch_size`
   - 使用`num_workers`进行并行数据加载

### 注意事项
1. 确保CUDA版本与PyTorch版本匹配（当前使用CUDA 11.8）
2. 如果遇到CUDA相关错误，检查环境变量和NVIDIA驱动版本
3. 定期保存模型和训练状态
4. 记录每次重要的代码修改和性能变化 

## 2、阅读代码

- 目标函数：$J(\theta) = \mathbb{E}*{s_t \sim \rho*\pi} [\sum_{t=0}^{\infty} \gamma^t (r(s_t, a_t) + \alpha H(\pi(\cdot|s_t)))]$
