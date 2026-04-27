### ROBOTIS AI Worker 訓練與部署流程（RL + IL 混合）

#### 1. 訓練階段：RL 主要在模擬環境進行
RL 訓練流程（以 Reach 任務為例）：
- 在 Isaac Lab 裡平行訓練。
- 使用 rsl_rl 演算法訓練策略。
- 訓練完後執行 `play.py` 產生 policy 檔案（神經網路權重）。

**範例命令（在 Docker 容器內執行）**：
```bash
python scripts/reinforcement_learning/rsl_rl/train.py --task RobotisLab-Reach-FFW-BG2-v0 --num_envs=512 --headless
python scripts/reinforcement_learning/rsl_rl/play.py --task RobotisLab-Reach-FFW-BG2-v0 --num_envs=16
```

**為什麼用模擬？**  
因為 RL 需要幾百萬次試錯，在真實機器人上根本不可能（會撞壞、耗電、時間超長）。

#### 2. 部署到真實機器人
ROBOTIS 支援把模擬訓練好的 policy 直接跑在真實 AI Worker 上。

**主要步驟**：

**Step 1：真實機器人硬體啟動**  
使用 **ai_worker** repository 把機器人 bring up（ROS 2 + DYNAMIXEL 馬達 + Jetson Orin）。  
這一步讓實體機器人能接收指令。

**Step 2：資料收集（推薦先用 Imitation Learning）**  
真人透過遠端操作（teleop）在**真實機器人**上示範任務（例如 Pick-and-Place）。  
腳本會同時記錄相機影像 + 機器人狀態。

**範例命令（真實機器人資料收集）**：
```bash
python scripts/sim2real/imitation_learning/recorder/record_demos.py \
--task=RobotisLab-Real-Pick-Place-FFW-SG2-v0 \
--robot_type FFW_SG2 \
--dataset_file ./datasets/ffw_sg2_raw.hdf5 \
--num_demos 4 --enable_cameras
```

**Step 3：資料處理與增強（讓 policy 更穩健）**  
- 把動作轉成末端執行器座標（Convert actions from joint space to end-effector pose）。
- 自動標註（Annotate） + 在模擬裡產生大量合成資料（augmentation）。
- Convert actions back to joint space。
- 最後轉成 LeRobot 格式，方便 physical_ai_tools 訓練。

這一步把少量真人示範變成大量高品質資料，大幅提升 Sim2Real 成功率。

**Step 4：訓練與推理**
- 用 **physical_ai_tools** 訓練 IL 模型（可選 Web UI 或 LeRobot CLI）。
- 訓練 ACT（Action Chunking Transformer）等模型，checkpoint 存到 `outputs/train/`。
- 先在模擬器裡跑 inference 驗證（`inference_demos.py`）。
- RL 則用上面訓練好的 policy 繼續 fine-tune（意思是用這個模型作為基礎來後續強化學習用，在模擬裡優化獎勵）。

**注意**：RL 的模型不能放進 physical_ai_tools 裡面繼續訓練，因為 RL 是 actor-critic 網路架構，Imitation Learning 是 Transformer-based 的 ACT 或 pi0 網路。

**Step 5：最後部署到實體機器人**
- 載入訓練好的 policy 檔案，直接執行 real-robot inference 腳本（physical_ai_tools 的 Web UI）。
- 系統會：接收真實相機/狀態 → policy inference → 輸出動作 → ROS 2 控制 DYNAMIXEL 馬達 + 底座。
- 有 Web UI 方便監控和觸發。

**官方 OMY 機器人範例**（AI Worker 流程完全相同）：
```bash
python scripts/sim2real/reinforcement_learning/inference/OMY/reach/run_omy_reach.py \
--model_dir=logs/rsl_rl/reach_omy/<你的時間戳資料夾>
```

#### 總結
實際部署到真實 AI Worker 上，主要走「Imitation Learning 為基底 + RL 微調 或 直接用 IL」的混合流程，而不是純 RL 從零直接執行。  

官方文件區分兩種 pipeline，RL 的 Sim2Real 範例較簡單（多用在 OMY 單臂 reach 任務），而 AI Worker（尤其是 FFW-SG2/BG2 雙臂/人形版本）更常使用 IL + physical_ai_tools 來實現真實部署，因為 IL 更穩定、安全，且能快速從少量真人示範學到複雜任務。


### 模擬機器人實現步驟 (AI Worker - Robotis Lab)

本指南介紹如何在 Isaac Lab 模擬環境中，針對 FFW-BG2（雙臂行動機器人）進行從資料收集到模型訓練與驗證的完整模仿學習（Imitation Learning）流程。
#### 安裝環境

確保已完成 robotis_lab 的容器配置與 Isaac Lab 環境安裝。
#### 資料收集 (Teleop + Record)

在模擬器裡操作虛擬機器人，利用鍵盤或遙控器完成任務，並記錄感測器數據（相機影像、機器人狀態）。
FFW-BG2 Pick and Place
```bash
python scripts/imitation_learning/isaaclab_recorder/record_demos.py \
  --task RobotisLab-PickPlace-FFW-BG2-IK-Rel-v0 \
  --teleop_device keyboard \
  --dataset_file ./datasets/dataset.hdf5 \
  --num_demos 10 \
  --enable_cameras
```
* 操作說明：使用鍵盤控制（如 WASD/QE 移動手臂，K 開關夾爪）。

* 建議：嘗試多種不同的物體起始位置與角度，增加資料多樣性。

3. 資料處理與標註 (Data Processing)

在訓練之前，需對錄製的原始資料進行檢查與語義標註。
回放檢查錄製資料

建議先回放第 0 筆資料，確認動作與影像錄製是否正常：
```bash
python scripts/imitation_learning/isaaclab_recorder/replay_demos.py \
  --task RobotisLab-PickPlace-FFW-BG2-IK-Rel-v0 \
  --dataset_file ./datasets/dataset.hdf5 \
  --select_episodes 0
```
手動標註 (Annotate)

針對 FFW-BG2 任務進行階段性標註（定義抓取、放置等關鍵點）：
```bash
python scripts/imitation_learning/isaaclab_mimic/annotate_demos.py \
  --device cuda \
  --task RobotisLab-PickPlace-FFW-BG2-Mimic-v0 \
  --input_file ./datasets/dataset.hdf5 \
  --output_file ./datasets/annotated_dataset.hdf5 \
  --enable_cameras
```
* 按鍵說明：n (播放)、b (暫停)、s (保存標註)。

4. 生成 Mimic 資料集 (Data Augmentation)

透過已標註的 Demo，在模擬器中自動生成更多變體的資料集：
```bash
python scripts/imitation_learning/isaaclab_mimic/generate_dataset.py \
  --device cuda \
  --task RobotisLab-PickPlace-FFW-BG2-Mimic-v0 \
  --num_envs 20 \
  --generation_num_trials 300 \
  --input_file ./datasets/annotated_dataset.hdf5 \
  --output_file ./datasets/generated_dataset.hdf5 \
  --enable_cameras \
  --headless
```
5. 開始訓練 (Training)
   
使用 Robomimic 框架進行行為克隆（BC）訓練：
```bash
python scripts/imitation_learning/robomimic/train.py \
  --task RobotisLab-PickPlace-FFW-BG2-IK-Rel-v0 \
  --algo bc \
  --dataset ./datasets/generated_dataset.hdf5 \
  --name bc_rnn_image_ffw_bg2_exp1 \
  --log_dir robomimic
```
6. 模型驗證 (Evaluation）
   
尋找最新 Checkpoint，訓練完成後，可以使用以下指令找到最新的模型權重檔（.pth）：
```bash
find logs/robomimic/RobotisLab-PickPlace-FFW-BG2-IK-Rel-v0 -name "*.pth" | sort | tail -n 1
```
#### 運行推理驗證

載入訓練好的權重，觀察機器人在模擬環境中的自動化表現：
```bash
python scripts/imitation_learning/robomimic/play.py \
  --device cuda \
  --task RobotisLab-PickPlace-FFW-BG2-IK-Rel-v0 \
  --checkpoint logs/robomimic/RobotisLab-PickPlace-FFW-BG2-IK-Rel-v0/bc_rnn_image_ffw_bg2/20260427152308/models/model_epoch_2.pth \
  --num_rollouts 20 \
  --horizon 800 \
  --enable_cameras
```

