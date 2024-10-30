# Tangent Space Causal Inference: Leveraging Vector Fields for Causal Discovery in Dynamical Systems

This repository contains the code for Tangent Space Causal Inference (TSCI), accepted to NeurIPS 2024. TSCI is a new method for causal inference in data generated by deterministic dynamical systems, extended existing cross map-based methods like Convergent Cross Mapping (CCM). Generally speaking, cross map-based methods construct embeddings of two dynamical systems, $x(t)$ and $y(t)$, and check for a smooth "cross map" $F \colon Y \to X$ to infer the causal relationship $X \rightarrow Y$, effectively testing that a "signature" of the cause is detectible in the effect. The existence of this cross map is justified by training a k-nearest neighbors estimator and examining a Pearson correlation score. In TSCI, we instead consider the vector fields induced by the dynamics of $x(t)$ and $y(t)$ on their embeddings, checking the alignment of these vector fields in the corresponding tangent spaces. Algorithmically, this requires the computation of the cross map Jacobian. In the nearest neighbors approach (`tsci_nn(...)`), this corresponds to solving a simple linear system. More generally, this Jacobian may be computed efficiently as a Jacobian-vector product with automatic differentiation (`tsci_torch(...)`).

# Citation
The TSCI paper is available on arXiv, and will appear in the [NeurIPS 2024 Proceedings](https://openreview.net/forum?id=Bj2CpB9Dey). If you use any code or results from this project, please consider citing the orignal paper:

```
@inproceedings{
butler2024tsci,
title={Tangent Space Causal Inference: Leveraging Vector Fields for Causal Discovery in Dynamical Systems},
author={Kurt Butler and Daniel Waxman and Petar M. Djuri\'c},
booktitle={The Thirty-eighth Annual Conference on Neural Information Processing Systems},
year={2024},
url={https://openreview.net/forum?id=Bj2CpB9Dey}
}
```

# Installation and Minimal Example
To install dependencies, use `pip` to install the requirements in `requirements.txt` from the `python` directory:
```
# Optionally, create a virtual environment
python3 -m venv venv 
source venv/bin/activate

# Install dependences
pip install -r requirements.txt
```
Note that `benchmark-mi` requires Python >= 3.10, and may have somewhat strange `jax` dependency issues if you're not using a virtual environment.

A minimal example is as follows:
```
import tsci
import numpy as np
from utils import (
    generate_lorenz_rossler,
    lag_select,
    false_nearest_neighbors,
    delay_embed,
    discrete_velocity,
)

# Generate Data from the Rossler-Lorenz system
C = 1.0
np.random.seed(0)
z0 = np.array([-0.82, -0.8, -0.24, 10.01, -12.19, 10.70])
z0 = z0 + np.random.randn(*z0.shape) * 1e-3
x, y = generate_lorenz_rossler(np.linspace(0, 110, 8000), z0, C)

x_signal = x[:, 1].reshape(-1, 1)
y_signal = y[:, 0].reshape(-1, 1)

# Get embedding hyperparameters and create delay embeddings
tau_x = lag_select(x_signal, theta=0.5)  # X lag
tau_y = lag_select(y_signal, theta=0.5)  # Y lag
Q_x = false_nearest_neighbors(x_signal, tau_x, fnn_tol=0.005)  # X embedding dim
Q_y = false_nearest_neighbors(y_signal, tau_y, fnn_tol=0.005)  # Y embedding dim

x_state = delay_embed(x_signal, tau_x, Q_x)
y_state = delay_embed(y_signal, tau_y, Q_y)
truncated_length = (
    min(x_state.shape[0], y_state.shape[0]) - 100
)  # Omit earliest samples
x_state = x_state[-truncated_length:]
y_state = y_state[-truncated_length:]

# Get velocities with (centered) finite differences
dx_dt = discrete_velocity(x_signal)
dy_dt = discrete_velocity(y_signal)

# Delay embed velocity vectors
dx_state = delay_embed(dx_dt, tau_x, Q_x)
dy_state = delay_embed(dy_dt, tau_y, Q_y)
dx_state = dx_state[-truncated_length:]
dx_state = dx_state
dy_state = dy_state[-truncated_length:]
dy_state = dy_state

############################
####    Perform TSCI    ####
############################
r_x2y, r_y2x = tsci.tsci_nn(
    x_state,
    y_state,
    dx_state,
    dy_state,
    fraction_train=0.8,
    use_mutual_info=False,
)

print(f"r_\u007bX -> Y\u007d: {np.mean(r_x2y):.2f}")
print(f"r_\u007bY -> X\u007d: {np.mean(r_y2x):.2f}")

```