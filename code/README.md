# Getting started
## Requirements
Our implementations heavily rely on `Docker` and the detailed environment setup refers to `Dockerfile` under the `../environments` folder.

By running command `docker-compose build` under the folder `environments`, you can build our main docker image `pytorch-mpi`.


## Use case of distributed training (centralized/decentralized)
Some simple explanation of the arguments used in the code.
* Arguments related to *distributed training*:
    * The `n_mpi_process` and `n_sub_process` indicates the number of nodes and the number of GPUs for each node. The data-parallel wrapper is adapted and applied locally for each node.
        * Note that the exact mini-batch size for each MPI process is specified by `batch_size`, while the mini-batch size used for each GPU is `batch_size/n_sub_process`.
    * The `world` describes the GPU topology of the distributed training, in terms of all GPUs used for the distributed training.
    * The `hostfile` from `mpi` specifies the physical location of the MPI processes.
    * We provide two use cases here:
        * `n_mpi_process=2`, `n_sub_process=1` and `world=0,0` indicates that two MPI processes are running on 2 GPUs with the same GPU id. It could be either 1 GPU at the same node or two GPUs at different nodes, where the exact configuration is determined by `hostfile`.
        * `n_mpi_process=2`, `n_sub_process=2` and `world=0,1,0,1` indicates that two MPI processes are running on 4 GPUs and each MPI process uses GPU id 0 and id 1 (on 2 nodes).
* Arguments related to *communication compression*:
    * The `graph_topology` 
    * The `optimizer` will decide the type of distributed training, e.g., centralized SGD, decentralized SGD
* Arguments related to *learning*:
    * The `lr_scaleup`, `lr_warmup` and `lr_warmup_epochs` will decide if we want to scale up the learning rate, or warm up the learning rate. For more details, please check `pcode/create_scheduler.py`.



### Examples
#### Centralized training (n=16)
The script below trains `ResNet-BN-20` on `CIFAR-10`, as an example of centralized training algorithm `centralized_sgd` for randomly partitioned local data (w/o local data reshuffle). A global momentum (buffer) will be used in this case.
```bash
declare -a list_of_n_processes=16
declare -a list_of_graph_topologies=complete
declare -a optimizer=centralized_sgd

declare -a list_of_seeds=6
declare -a list_of_lr_scale_factors=16

OMP_NUM_THREADS=2 MKL_NUM_THREADS=2 $HOME/conda/envs/pytorch-py3.6/bin/python run.py \
--arch resnet20 --group_norm_num_groups 0 --optimizer ${optimizer} \
--experiment demo --manual_seed ${list_of_seeds} \
--data cifar10 --pin_memory True \
--batch_size 32 --base_batch_size 64 --num_workers 0 \
--num_epochs 300 --partition_data random --reshuffle_per_epoch False --stop_criteria epoch \
--n_mpi_process ${list_of_n_processes} --n_sub_process 1 --world 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 \
--on_cuda True --use_ipc False \
--lr 0.1 --lr_scaleup True --lr_scaleup_factor ${list_of_lr_scale_factors} --lr_warmup True --lr_warmup_epochs 5 \
--lr_scheduler MultiStepLR --lr_decay 0.1 --lr_milestone_ratios 0.5,0.75 \
--weight_decay 1e-4 --use_nesterov True --momentum_factor 0.9 \
--hostfile hostfile --graph_topology ${list_of_graph_topologies} --track_time True --display_tracked_time True \
--python_path $HOME/conda/envs/pytorch-py3.6/bin/python --mpi_path $HOME/.openmpi/ 
```

The script below trains `distilbert-base-uncased` on `AG News`, as an example of decentralized training algorithm `centralized_adam` for randomly partitioned local data (w/o local data reshuffle). We first synchronize the gradients globally by using All-reduce, before applying them to the model.
```bash
declare -a list_of_n_processes=16
declare -a list_of_graph_topologies=complete
declare -a optimizer=centralized_adam

declare -a list_of_seeds=6
declare -a non_iid_ness=1

OMP_NUM_THREADS=2 MKL_NUM_THREADS=2 $HOME/conda/envs/pytorch-py3.6/bin/python run.py \
--arch distilbert-base-uncased --optimizer ${optimizer} --bert_conf model_scheme=vector_cls_sentence,max_seq_len=128,eval_every_batch=100 \
--experiment demo --manual_seed ${list_of_seeds} \
--data agnews --pin_memory True --batch_size 32 --base_batch_size 32 --num_workers 0 \
--num_epochs 10 --partition_data random --reshuffle_per_epoch False --stop_criteria epoch \
--n_mpi_process ${list_of_n_processes} --n_sub_process 1 --world 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 \
--on_cuda True --use_ipc False \
--lr 1e-5 --lr_scaleup False --lr_warmup False --lr_scheduler MultiStepLR \
--weight_decay 1e-4 \
--hostfile hostfile --graph_topology ${list_of_graph_topologies} --track_time True --display_tracked_time True \
--python_path $HOME/conda/envs/pytorch-py3.6/bin/python --mpi_path $HOME/.openmpi/
```


#### Decentralized training on Ring topology (n=16).
The script below trains `ResNet-GN-20` on `CIFAR-10`, as an example of decentralized training algorithm `decentralized_sgd` on heterogeneous data (with non-iid-ness of 1). Each worker maintains an independent local momentum buffer.
```bash
declare -a list_of_n_processes=16
declare -a list_of_graph_topologies=ring
declare -a optimizer=decentralized_sgd

declare -a list_of_seeds=6
declare -a list_of_lr_scale_factors=16
declare -a non_iid_ness=1

OMP_NUM_THREADS=2 MKL_NUM_THREADS=2 $HOME/conda/envs/pytorch-py3.6/bin/python run.py \
--arch resnet20 --group_norm_num_groups 2 --optimizer ${optimizer} \
--experiment demo --manual_seed ${list_of_seeds} \
--data cifar10 --pin_memory True \
--batch_size 32 --base_batch_size 64 --num_workers 0 \
--num_epochs 300 --partition_data non_iid_dirichlet --non_iid_alpha ${non_iid_ness} --reshuffle_per_epoch False --stop_criteria epoch \
--n_mpi_process ${list_of_n_processes} --n_sub_process 1 --world 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 \
--on_cuda True --use_ipc False \
--lr 0.1 --lr_scaleup True --lr_scaleup_factor ${list_of_lr_scale_factors} --lr_warmup True --lr_warmup_epochs 5 \
--lr_scheduler MultiStepLR --lr_decay 0.1 --lr_milestone_ratios 0.5,0.75 \
--weight_decay 1e-4 --use_nesterov True --momentum_factor 0.9 \
--hostfile hostfile --graph_topology ${list_of_graph_topologies} --track_time True --display_tracked_time True \
--python_path $HOME/conda/envs/pytorch-py3.6/bin/python --mpi_path $HOME/.openmpi/ 
```

The script below trains `ResNet-EvoNorm-20` on `CIFAR-10`, as an example of decentralized training algorithm `decentralized_qg_sgd` on heterogeneous data (with non-iid-ness of 1). Each worker maintains an independent quasi-global momentum buffer.
```bash
declare -a list_of_n_processes=16
declare -a list_of_graph_topologies=ring
declare -a optimizer=decentralized_qg_sgd

declare -a list_of_seeds=6
declare -a list_of_lr_scale_factors=16
declare -a non_iid_ness=1

OMP_NUM_THREADS=2 MKL_NUM_THREADS=2 $HOME/conda/envs/pytorch-py3.6/bin/python run.py \
--arch resnet_evonorm20 --optimizer ${optimizer} \
--experiment demo --manual_seed ${list_of_seeds} \
--data cifar10 --pin_memory True \
--batch_size 32 --base_batch_size 64 --num_workers 0 \
--num_epochs 300 --partition_data non_iid_dirichlet --non_iid_alpha ${non_iid_ness} --reshuffle_per_epoch False --stop_criteria epoch \
--n_mpi_process ${list_of_n_processes} --n_sub_process 1 --world 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 \
--on_cuda True --use_ipc False \
--lr 0.1 --lr_scaleup True --lr_scaleup_factor ${list_of_lr_scale_factors} --lr_warmup True --lr_warmup_epochs 5 \
--lr_scheduler MultiStepLR --lr_decay 0.1 --lr_milestone_ratios 0.5,0.75 \
--weight_decay 1e-4 --use_nesterov True --momentum_factor 0.9 \
--hostfile hostfile --graph_topology ${list_of_graph_topologies} --track_time True --display_tracked_time True \
--python_path $HOME/conda/envs/pytorch-py3.6/bin/python --mpi_path $HOME/.openmpi/
```

The script below trains `distilbert-base-uncased` on `AG News`, as an example of decentralized training algorithm `decentralized_qg_adam` on heterogeneous data (with non-iid-ness of 1). Each worker maintains two independent quasi-global moment buffers.
```bash
declare -a list_of_n_processes=16
declare -a list_of_graph_topologies=ring
declare -a optimizer=decentralized_qg_adam

declare -a list_of_seeds=6
declare -a non_iid_ness=1

OMP_NUM_THREADS=2 MKL_NUM_THREADS=2 $HOME/conda/envs/pytorch-py3.6/bin/python run.py \
--arch distilbert-base-uncased --optimizer ${optimizer} --bert_conf model_scheme=vector_cls_sentence,max_seq_len=128,eval_every_batch=100 \
--experiment demo --manual_seed ${list_of_seeds} \
--data agnews --pin_memory True --batch_size 32 --base_batch_size 32 --num_workers 0 \
--num_epochs 10 --partition_data non_iid_dirichlet --non_iid_alpha ${non_iid_ness} --reshuffle_per_epoch False --stop_criteria epoch \
--n_mpi_process ${list_of_n_processes} --n_sub_process 1 --world 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 \
--on_cuda True --use_ipc False \
--lr 1e-5 --lr_scaleup False --lr_warmup False --lr_scheduler MultiStepLR \
--weight_decay 1e-4 \
--hostfile hostfile --graph_topology ${list_of_graph_topologies} --track_time True --display_tracked_time True \
--python_path $HOME/conda/envs/pytorch-py3.6/bin/python --mpi_path $HOME/.openmpi/
```


#### Decentralized training on Social Network topology (n=32).
The script below trains `ResNet-EvoNorm-20` on `CIFAR-10`, as an example of decentralized training algorithm `decentralized_qg_sgd` on heterogeneous data (with non-iid-ness of 1). Each worker maintains an independent quasi-global momentum buffer.
```bash
declare -a list_of_n_processes=32
declare -a list_of_graph_topologies=social
declare -a optimizer=decentralized_qg_sgd

declare -a list_of_seeds=6
declare -a list_of_lr_scale_factors=32
declare -a non_iid_ness=1

OMP_NUM_THREADS=2 MKL_NUM_THREADS=2 $HOME/conda/envs/pytorch-py3.6/bin/python run.py \
--arch resnet_evonorm20 --optimizer ${optimizer} \
--experiment demo --manual_seed ${list_of_seeds} \
--data cifar10 --pin_memory True \
--batch_size 32 --base_batch_size 64 --num_workers 0 \
--num_epochs 300 --partition_data non_iid_dirichlet --non_iid_alpha ${non_iid_ness} --reshuffle_per_epoch False --stop_criteria epoch \
--n_mpi_process ${list_of_n_processes} --n_sub_process 1 --world 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 \
--on_cuda True --use_ipc False \
--lr 0.1 --lr_scaleup True --lr_scaleup_factor ${list_of_lr_scale_factors} --lr_warmup True --lr_warmup_epochs 5 \
--lr_scheduler MultiStepLR --lr_decay 0.1 --lr_milestone_ratios 0.5,0.75 \
--weight_decay 1e-4 --use_nesterov True --momentum_factor 0.9 \
--hostfile hostfile --graph_topology ${list_of_graph_topologies} --track_time True --display_tracked_time True \
--python_path $HOME/conda/envs/pytorch-py3.6/bin/python --mpi_path $HOME/.openmpi/
```
