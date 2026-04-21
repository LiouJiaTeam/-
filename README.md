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

**Step 2：資料收集（推薦先用 Imitation Learning 做基底）**  
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
