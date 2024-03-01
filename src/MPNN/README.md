# MPNN

## Config file

We evaluate the MPNN on 1D and 2D time dependent problems. The config files are saved in `config` directory. The explanations of some MPNN specific args:

* PDE information args:
    * `temporal_domain`: tuple, the value range of temporal domain.
    * `resolution_t`: int, the original temporal resolution of data.
    * `spatial_domain`: string can be parsed into the list of tuple with `eval` function, the value range of spatial domain.
    * `resolution`: int, the original spatial resolution of data.
    * `variables`: dict, contains PDE parameters.
    * `num_outputs`: int, the number of variables to be solved.

* training args:
    * `neighbors`: int, the number of neighbors used for message passing. By default, we set it to 3 for 1D problems and 1 for 2D problems.
    * `time_window`: int, the number of input/output timesteps. We set it to 10 that equals to the value of `initial_step` args in other autoregressive methods for fair comparison.
    * `unrolling`: int, the number of unrolled forward steps. (default 1)
    * `unroll_step`: int, the number of time steps to backpropagate during pushforward training. Because there are slight differences (read [here](https://github.com/zhouzy36/PDENNEval/tree/main/src/MPNN#training-strategy) for details) in training strategy between MPNN and other autoregressive methods, this arg is borrowed from other autoregressive methods to ensure the number of iterations in one MPNN training epoch is almost equal to others. (default 1)

* model args:
    * `hidden_features`: int, the number of node feature dimensions.
    * `hidden_layer`: int, the number of GNN layers.
    
For reproducibility, we give our training hyperparameters for solving different PDEs in the table below.

| PDE Names                   | spatial resolution / downsample rate | temporal resolution / downsample rate | lr    | batch size | weight decay | number of neighbors | epochs | lr schedule        |
| :-------------------------- | :----------------- | :------------------ | :---- | :--------- | :----------- | :------------------ | :----- | :----------------- |
| 1D Advection                | 1024/4             | 201/5               | 1.e-4 | 64         | 1.e-8        | 3                   | 500    | "StepLR", 100, 0.5 |
| 1D Diffusion-Reaction       | 1024/4             | 101/1               | 1.e-4 | 64         | 1.e-8        | 3                   | 500    | "StepLR", 100, 0.5 |
| 1D Burgers                  | 1024/4             | 201/5               | 1.e-4 | 64         | 1.e-8        | 3                   | 500    | "StepLR", 100, 0.5 |
| 1D Diffusion-Sorption       | 1024/4             | 101/1               | 1.e-4 | 64         | 1.e-8        | 3                   | 500    | "StepLR", 100, 0.5 |
| 1D Compressible NS          | 1024/4             | 101/1               | 1.e-4 | 64         | 1.e-8        | 3                   | 500    | "StepLR", 100, 0.5 |
| 1D Allen Cahn               | 1024/4             | 101/1               | 1.e-4 | 64         | 1.e-8        | 3                   | 500    | "StepLR", 100, 0.5 |
| 1D Cahn Hilliard            | 1024/4             | 101/1               | 1.e-4 | 64         | 1.e-8        | 3                   | 500    | "StepLR", 100, 0.5 |
| 2D Shallow Water            | 128/2              | 101/1               | 1.e-4 | 8          | 1.e-8        | 1                   | 500    | "StepLR", 100, 0.5 |
| 2D Compressible NS          | 128/2              | 21/1                | 1.e-4 | 16         | 1.e-8        | 1                   | 500    | "StepLR", 100, 0.5 |
| 2D Burgers                  | 128/2              | 101/1               | 1.e-4 | 8          | 1.e-8        | 1                   | 500    | "StepLR", 100, 0.5 |
| 2D Allen-Cahn               | 128/2              | 101/1               | 1.e-4 | 8          | 1.e-8        | 1                   | 500    | "StepLR", 100, 0.5 |
| 2D Black-Scholes-Barenblatt | 128/2              | 101/1               | 1.e-4 | 8          | 1.e-8        | 1                   | 500    | "StepLR", 100, 0.5 |


## Loss function

MPNN employs the autoregressive approach to solve PDEs, similar to U-Net, FNO and U-NO. The model predicts the solution of the next time steps $\hat{u}^{k+1}$ based on the solution of previous $l$ (`time_window`) time steps $\{\hat{u}^{k-l+
1},...,\hat{u}^k\}$. This process can be formalized as 
$$
\hat{u}^{k+1} = f_{\theta}(\hat{u}^{k-l+1},...,\hat{u}^k).
$$

The loss function has the form:

$$
\mathcal{L}=\frac{1}{N}\sum_{k=1}^{N}l(f_{\theta}(\hat{u}^{k-l+1},...,\hat{u}^k), u^k),
$$

where $u^{k}$ is ground truth data generated by high-order numerical methods and $l$ is MSE in our implementation.

## Training strategy

We train MPNN with temporal bundling trick and pushforward trick.

**Temporal bundling trick**: Unlike other autoregressive method only predicts solution of the next time steps in one forward calculation, MPNN predicts $l$ (`time_window`) steps together: $(\hat{u}^{k+1},...,\hat{u}^{k+l}) = f_{\theta}(\hat{u}^{k-l+1},...,\hat{u}^k)$. This
reduces the number of solver calls by a factor of $l$ and so reduces the inference time.

**Pushforward trick**: 

## Train

1. Check the following args in the config file:
    1. The path of `file_name` and `saved_folder` are correct;
    2. `if_training` is `True`;
2. Set hyperparameters for training, such as `lr`, `batch size`, etc. You can use the default values we provide;
3. Run command:
```bash
CUDA_VISIBLE_DEVICES=0 python train.py ./config/${config file name}
# Example: CUDA_VISIBLE_DEVICES=0 python train.py ./configs/config_1D_Advection.yaml
```

## Resume training

1. Modify config file:
    1. Make sure `if_training` is `True`;
    2. Set `continue_training` to `True`;
    3. Set `model_path` to the checkpoint path where traing restart;
2. Run command:
```bash
CUDA_VISIBLE_DEVICES=0 python train.py ./config/${config file name}
```

## Test

1. Modify config file:
    1. Set `if_training` to `False`;
    2. Set `model_path` to the checkpoint path where the model to be evaluated is saved.
2. Run command:
```bash
CUDA_VISIBLE_DEVICES=0 python train.py ./config/${config file name}
```