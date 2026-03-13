# CLAUDE.md

本文档为 Claude Code (claude.ai/code) 在此代码库中工作提供指导。

## 概述

NAVSIM 是一个用于自动驾驶评估的伪仿真框架，结合了开环效率和闭环鲁棒性。该代码库实现了智能体（驾驶模型）、指标缓存、PDM评分、交通仿真和两阶段伪闭环评估。它基于 nuPlan devkit 构建，使用 Hydra 进行配置，Ray 进行分布式处理，PyTorch 用于基于学习的智能体。

## 环境设置

### 安装
1. 创建 conda 环境：`conda env create --name navsim -f environment.yml`
2. 激活环境：`conda activate navsim`
3. 以开发模式安装：`pip install -e .`

### 必需的环境变量
在 `~/.bashrc` 中设置以下内容（根据实际情况调整路径）：
```bash
export NUPLAN_MAP_VERSION="nuplan-maps-v1.0"
export NUPLAN_MAPS_ROOT="$HOME/navsim_workspace/dataset/maps"
export NAVSIM_EXP_ROOT="$HOME/navsim_workspace/exp"
export NAVSIM_DEVKIT_ROOT="$HOME/navsim_workspace/navsim"
export OPENSCENE_DATA_ROOT="$HOME/navsim_workspace/dataset"
```

### 数据下载
脚本位于 `download/` 目录中。根据需要下载地图和数据集划分：
```bash
cd download
./download_maps
./download_mini
./download_trainval
./download_test
./download_warmup_two_stage
./download_navhard_two_stage
./download_private_test_hard_two_stage
```

数据应按照 `docs/install.md` 中的描述组织在 `$OPENSCENE_DATA_ROOT` 目录下。

## 常见开发任务

### 指标缓存
为评估预计算特征：
```bash
TRAIN_TEST_SPLIT=navtest
CACHE_PATH=$NAVSIM_EXP_ROOT/metric_cache
python $NAVSIM_DEVKIT_ROOT/navsim/planning/script/run_metric_caching.py \
  train_test_split=$TRAIN_TEST_SPLIT \
  metric_cache_path=$CACHE_PATH
```
参考 `scripts/evaluation/run_metric_caching.sh` 中的示例。

### 训练基于学习的智能体
训练智能体（例如，EgoStatusMLP）：
```bash
TRAIN_TEST_SPLIT=navtrain
python $NAVSIM_DEVKIT_ROOT/navsim/planning/script/run_training.py \
  experiment_name=training_ego_mlp_agent \
  trainer.params.max_epochs=50 \
  train_test_split=$TRAIN_TEST_SPLIT
```
配置由 Hydra 管理；使用 `key=value` 覆盖任何参数。

### 评估智能体
运行 PDM 评分（两阶段伪闭环）：
```bash
TRAIN_TEST_SPLIT=warmup_two_stage
python $NAVSIM_DEVKIT_ROOT/navsim/planning/script/run_pdm_score.py \
  train_test_split=$TRAIN_TEST_SPLIT \
  agent=constant_velocity_agent \
  metric_cache_path=$NAVSIM_EXP_ROOT/metric_cache
```
该脚本输出一个包含每个 token 分数的 CSV 文件以及一个聚合的扩展 PDM 分数。

### 为排行榜创建提交
为给定的数据集划分生成 `submission.pkl`（例如，`private_test_hard_two_stage`）：
```bash
TEAM_NAME="YourTeam"
AUTHORS="Your Name"
EMAIL="your@email.com"
INSTITUTION="Your Institution"
COUNTRY="Your Country"
TRAIN_TEST_SPLIT=private_test_hard_two_stage

python $NAVSIM_DEVKIT_ROOT/navsim/planning/script/run_create_submission_pickle_challenge.py \
  train_test_split=$TRAIN_TEST_SPLIT \
  agent=constant_velocity_agent \
  experiment_name=submission_cv_agent \
  team_name=$TEAM_NAME \
  authors=$AUTHORS \
  email=$EMAIL \
  institution=$INSTITUTION \
  country=$COUNTRY
```
将生成的 `submission.pkl` 作为 Hugging Face 模型上传，并将模型链接提交到相应的排行榜。

## 代码架构

### 核心模块
- **`navsim.agents`** – 抽象智能体接口（`AbstractAgent`）和具体实现（`ConstantVelocityAgent`、`EgoStatusMLPAgent`、`TransfuserAgent`、`HumanAgent`）。智能体定义传感器配置、特征/目标构建器（用于学习）和轨迹计算。
- **`navsim.common`** – 共享数据类（`AgentInput`、`SensorConfig`、`Trajectory`、`PDMResults`）、数据加载器（`SceneLoader`、`MetricCacheLoader`）和枚举。
- **`navsim.planning`** – 伪仿真管道的核心：
  - **`metric_caching`** – 为高效评估预计算特征。
  - **`simulation`** – 实现两阶段伪闭环仿真（`PDMSimulator`、`PDMScorer`）和交通智能体策略。
  - **`training`** – 轻量级训练框架，包含特征/目标构建器和 PyTorch Lightning 集成。
  - **`script`** – 入口点脚本（`run_metric_caching.py`、`run_training.py`、`run_pdm_score.py`、`run_create_submission_pickle*.py`）和 Hydra 配置树。
- **`navsim.evaluate`** – PDM 评分逻辑（`pdm_score.py`）。
- **`navsim.traffic_agents_policies`** – 反应式交通智能体的抽象接口和策略。

### 配置系统
项目使用 **Hydra**，配置文件位于 `navsim/planning/script/config/` 目录下。关键的配置组包括：
- `agent` – 智能体特定参数（例如，`constant_velocity_agent`、`ego_status_mlp_agent`、`transfuser_agent`）。
- `train_test_split` – 定义数据集划分（`navtrain`、`navtest`、`warmup_two_stage`、`navhard_two_stage`、`private_test_hard_two_stage`）。
- `simulator`、`scorer`、`traffic_agents_policy` – 仿真和评分设置。
- `worker` – 并行化后端（Ray、顺序等）。

通过命令行 `key=value` 或组合 YAML 文件来覆盖任何配置参数。

### 智能体开发
要创建一个新的智能体：
1. 继承自 `AbstractAgent`。
2. 实现 `name()`、`get_sensor_config()`、`initialize()` 和 `compute_trajectory()`（或 `forward()` + 特征/目标构建器用于学习型智能体）。
3. 如果可训练，提供 `get_feature_builders()`、`get_target_builders()`、`compute_loss()`、`get_optimizers()`。
4. 在 `config/agent/` 下添加 Hydra 配置，并在 `scripts/` 中添加脚本包装器。

参考 `navsim/agents/ego_status_mlp_agent.py` 和 `navsim/agents/transfuser/` 中的示例。

## 代码规范与格式化
代码库使用 `black`、`isort`、`autoflake` 和 `flake8` 通过 pre-commit 钩子进行管理。
- 安装钩子：`pre-commit install`
- 对所有文件运行：`pre-commit run --all-files`
- 行长度：120 个字符。

## 有用的脚本
预制的脚本位于 `scripts/` 目录中：
- `scripts/evaluation/` – 不同智能体的指标缓存和评分。
- `scripts/training/` – 学习型智能体的训练流程。
- `scripts/submission/` – 生成提交 pickle 文件。

根据需要调整 `TRAIN_TEST_SPLIT` 和路径。

## 注意事项
- 主分支包含 NAVSIM v2（2025 挑战赛）。对于 v1（navtest 排行榜），请切换到 `v1.1` 分支。
- 评估采用**两阶段伪闭环**：第一阶段使用原始场景，第二阶段使用在规划轨迹附近生成的合成场景。
- 评分前需要指标缓存；缓存依赖于数据集划分。
- 排行榜提交需要有效的 `TEAM_NAME`、`AUTHORS` 等信息包含在提交 pickle 文件中。