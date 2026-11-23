# scGTN Muraro 基线代码与环境记录

- 官方仓库为 [W-RMSL/scGTN](https://github.com/W-RMSL/scGTN/tree/23fc3985e70a5a7def91057bfea896def83dcea3)，分支为 `main`。Git 克隆遇到 DNS 和 HTTP/2 传输错误，工作区使用 GitHub 官方归档。提交号取自归档头。
- 官方源码位于工作区 `scGTN/`，完整保留。运行入口使用从 `scGTN_main.py` 生成的 `work/scGTN_logged.py`。辅助文件 `work/baseline_logging.py` 负责保存配置、每轮损失与指标、KMeans 初始化记录、预测和权重。
- `work/check_preprocessing.py` 直接调用官方数据加载，并监听 Scanpy 原有步骤与 Transformer 输入。此脚本在独立进程执行。
- 新增依赖使用 `ai4sci` 对应的 `python -m pip`。Transformers 固定为 4.40.2，以匹配官方 `BertAttention` 的位置参数与元组返回值。CPU 前向检查通过。代码兼容性修补为零。
- 每轮 NMI 和 ARI 复用官方 KMeans 的同一组预测，不再次执行聚类。模型训练、参数更新和官方最佳轮次判定均沿用原入口。有限值检查在记录后遇到非有限损失时终止。
- 最佳轮次预测沿用官方评估逻辑，其中嵌入来自该轮更新前的训练模式前向传播，融合权重在优化器步后读取。最佳轮次权重保存于该轮更新后，完成权重另存为 `final.pt`。

## 训练入口的完整差异

```diff
--- scGTN_main.py
+++ scGTN_logged.py
@@ -1,5 +1,6 @@
 # -*- coding: utf-8 -*-
 import argparse
+import baseline_logging as logger
 import numpy as np
 import torch
 import random
@@ -191,6 +192,7 @@
     # Initialize model
     model = scGTN(config)
     model.data = data
+    run = logger.BaselineRun(args, config)
     model = model.to(device)
 
     # Move data to device
@@ -205,6 +207,7 @@
         z, z2 = model.get_embeddings()
         kmeans = KMeans(n_clusters=nclass, n_init=20, random_state=2021)
         kmeans.fit(z.cpu().numpy())
+        run.event('Initial KMeans', kmeans.inertia_)
         model.cluster_layer.data = torch.tensor(kmeans.cluster_centers_, device=device)
 
     # Training
@@ -256,6 +259,7 @@
                             'state': copy.deepcopy(model.state_dict())}
 
         pbar.set_postfix({'loss': f'{loss.item():.4f}'})
+        run.epoch(model, epoch, loss, losses, pred, y_true, acc, use_otc=False)
 
     # Load best pre-training model
     if best_pretrain.get('state'):
@@ -270,6 +274,8 @@
         z_fused = alpha * z + (1 - alpha) * z2
         kmeans = KMeans(n_clusters=nclass, n_init=20, random_state=2021)
         y_pred = kmeans.fit_predict(z_fused.cpu().numpy())
+        run.event('Post-pretraining KMeans', kmeans.inertia_)
+        run.save_state(model, 'pretrain_best_before_center_reset.pt')
         model.cluster_layer.data = torch.tensor(kmeans.cluster_centers_, device=device)
 
     target_distribution = None
@@ -318,6 +324,7 @@
             best_finetune = {'acc': acc, 'nmi': nmi(pred, y_true), 'ari': ari(pred, y_true)}
 
         pbar.set_postfix({'loss': f'{loss.item():.4f}'})
+        run.epoch(model, epoch, loss, losses, pred, y_true, acc, use_otc=True)
 
     t_end = time.time()
 
@@ -328,3 +335,5 @@
     print(f"Pre-training - ACC: {best_pretrain['acc']:.4f}, NMI: {best_pretrain['nmi']:.4f}, ARI: {best_pretrain['ari']:.4f}")
     print(f"Fine-tuning  - ACC: {best_finetune['acc']:.4f}, NMI: {best_finetune['nmi']:.4f}, ARI: {best_finetune['ari']:.4f}")
     print(f"Total time: {t_end - t_begin:.2f}s")
+    run.finish(model, best_pretrain, best_finetune, t_end - t_begin)
+
```

## 关键执行命令

以下命令在工作区根目录运行。预处理与训练的日志分别保存于结果目录。

```zsh
conda list -n ai4sci
py=/opt/miniconda3/envs/ai4sci/bin/python
"$py" --version
"$py" -c 'import sys; print(sys.executable)'
"$py" -m pip check
export PYTHONPATH="$PWD/scGTN"
export SCGTN_RESULTS="$PWD/outputs/results/scgtn_muraro_baseline"
export SCGTN_COMMIT=23fc3985e70a5a7def91057bfea896def83dcea3
export MPLCONFIGDIR="$PWD/work/mpl" XDG_CACHE_HOME="$PWD/work/cache"
export NUMBA_CACHE_DIR="$PWD/work/numba" PYTHONDONTWRITEBYTECODE=1 TZ=Asia/Shanghai
"$py" scGTN/scGTN_main.py --help
"$py" -u work/check_preprocessing.py "$SCGTN_RESULTS" --data_dir scGTN/datasets --device cpu
"$py" work/instrument_entry.py scGTN/scGTN_main.py work/scGTN_logged.py
"$py" -u work/scGTN_logged.py --data_dir scGTN/datasets --device cpu
```
