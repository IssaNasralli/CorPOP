
# CorPOP – Model Training with Adaptive CORAL

## Overview

This folder contains the training implementation of **CorPOP**, a deep learning framework for fine-scale population estimation using multi-source geospatial data.

The model combines a Variational Autoencoder (VAE), an atrous convolutional branch, and a feature-fusion mechanism to extract spatial information at different scales. It also integrates **adaptive CORAL (Correlation Alignment)** to improve the transferability of learned representations between geographically different regions.

The main training script is:

- `test_CORAL.py`

Unlike fixed CORAL regularization, the adaptive approach dynamically adjusts the CORAL contribution to the total loss according to the discrepancy between source and target feature distributions.

## Folder Structure

```text
Model Training of CorPOP/
│
├── test_CORAL.py
├── popVAE_full_Gate_Atrous_Gate_CORAL.py
└── data_atrous.py
```

The training script also imports `evaluate.py`. Make sure this module is available in the project environment or add it to the folder if it is not already provided elsewhere.

## Model Architecture

CorPOP uses a multi-branch architecture to capture spatial information at different scales.

### 1. VAE branch

The VAE branch processes a local spatial patch around each target pixel. It consists of:

- An encoder that extracts a latent representation.
- A latent sampling mechanism based on the mean and log-variance.
- A decoder that reconstructs the input patch.
- A latent representation used by the population prediction branch.

### 2. Atrous convolution branch

The atrous branch processes a larger spatial patch to capture broader geographic context.

It uses parallel dilated convolutions with different dilation rates to extract spatial features at multiple receptive-field scales.

### 3. Feature fusion and prediction

The extracted representations are combined with pixel-level ancillary features. Gating mechanisms are used to modulate the learned representations before fusion.

The resulting feature vector is passed to a prediction head to estimate the population value for each pixel.

## Adaptive CORAL Domain Adaptation

To improve cross-domain generalization, the model uses CORAL to align the statistical distributions of source and target features.

Let \(C_s\) and \(C_t\) denote the source and target feature matrices. Their means and covariance matrices are computed as:

\[
\mu_s = \frac{1}{N}\sum_{i=1}^{N}c_s^{(i)}
\]

\[
\mu_t = \frac{1}{N}\sum_{i=1}^{N}c_t^{(i)}
\]

\[
\Sigma_s =
\frac{1}{N-1}
(C_s-\mu_s)(C_s-\mu_s)^T
\]

\[
\Sigma_t =
\frac{1}{N-1}
(C_t-\mu_t)(C_t-\mu_t)^T
\]

The CORAL loss measures the discrepancy between the source and target distributions:

\[
L_{\mathrm{CORAL}} =
\frac{1}{4d^2}
\|\Sigma_s-\Sigma_t\|_F^2
+
\lambda_\mu\|\mu_s-\mu_t\|_2^2
\]

where \(d\) is the feature dimension and \(\lambda_\mu\) controls the contribution of the mean discrepancy.

### Dynamic regularization coefficient

Instead of using a constant CORAL weight, the training procedure computes a dynamic coefficient:

\[
D_t^2 =
\|\Sigma_s-\Sigma_t\|_F^2
+
\lambda_\mu\|\mu_s-\mu_t\|_2^2
\]

\[
\gamma_t =
\gamma_{\max}
\frac{D_t^2}{D_t^2+\kappa}
\]

where:

- \(D_t^2\) represents the current source-target feature discrepancy.
- \(\gamma_{\max}\) is the maximum CORAL regularization strength.
- \(\kappa\) controls the sensitivity of the adaptive coefficient.

The initial discrepancy \(D_0^2\) is computed before CORAL alignment. The parameter \(\kappa\) is set relative to this initial discrepancy.

As the source-target discrepancy increases, the CORAL contribution becomes stronger. When the discrepancy decreases, the contribution becomes weaker.

The coefficient is bounded:

\[
0 \leq \gamma_t < \gamma_{\max}
\]

This adaptive mechanism allows the model to adjust the strength of domain alignment throughout training.

### Total loss

The total objective combines the population prediction loss, VAE regularization, and adaptive CORAL alignment:

\[
L_{\mathrm{total}} =
\alpha L_{\mathrm{pop}}
+
\beta L_{\mathrm{VAE}}
+
\gamma_t L_{\mathrm{CORAL}}
\]

Here, \(\alpha\) and \(\beta\) control the contributions of population prediction and VAE regularization, while \(\gamma_t\) dynamically controls domain alignment.

## Dataset and Experimental Setup

The framework uses two geographic domains:

- **Source domain:** Tunisia.
- **Target domain:** Île-de-France, France.

The source domain provides the population supervision used to train the prediction model. The target domain provides samples for unsupervised feature alignment through CORAL.

The input data consist of multi-source geospatial raster layers, including ancillary geographic information. The model uses local and global spatial patches centered on the pixels being processed.

The training configuration previously provided includes:

| Parameter | Configuration |
|---|---:|
| Number of input bands | 10 |
| Local patch size | 11 |
| Global patch size | 35 |
| Latent dimension | 30 |
| Batch size | 1024 |
| Number of epochs | 100 |
| Early-stopping patience | 5 |
| Random seed | 42 |
| Optimizer | Adam |

These parameter values should be checked against the current `test_CORAL.py` before running experiments.

## Requirements

The implementation uses Python and TensorFlow/Keras, together with the supporting libraries required for geospatial data processing and numerical computation.

The project modules required by the training script include:

- `popVAE_full_Gate_Atrous_Gate_CORAL.py`
- `data_atrous.py`
- `evaluate.py`

Install the required Python packages in your environment according to the imports used by these modules.

## Running the Training

Make sure that the required Python modules and input datasets are available.

Run the main training script from the project directory:

```bash
python test_CORAL.py
```

The script loads the source and target data, prepares the training samples, builds the CorPOP model, and performs training with adaptive CORAL regularization.

Check the dataset paths and configuration parameters in the script before execution, especially when running the experiment on another machine or geographic region.

## Input Data

The training configuration previously used the following raster and auxiliary files:

| File | Description |
|---|---|
| `tunisia10.tif` | Source-domain ancillary raster |
| `POP_Tunisia.tif` | Source population raster |
| `france10.tif` | Target-domain ancillary raster |
| `ins_population.csv` | Population reference information |

Ensure that all input files are correctly located and that the paths specified in the training script match their actual locations.

## Training Outputs

The training procedure generates model and prediction outputs, including:

- Trained model weights.
- Model architecture information.
- Training logs.
- Population prediction rasters.
- Processed raster outputs.
- Checkpoint information.

The exact output filenames depend on the configuration specified in the training script.

## Notes

- `test_CORAL.py` is the main training entry point documented in this repository.
- The implementation uses adaptive rather than fixed CORAL regularization.
- The target domain is used for unsupervised feature alignment; population labels from the target domain are not required for this alignment.
- The adaptive coefficient is recalculated according to the current source-target feature discrepancy during training.
- Ensure that the required supporting modules and datasets are available before execution.

## Reference

The adaptive CORAL formulation and experimental methodology are described in the CorPOP paper.

The adaptive coefficient follows the formulation:

\[
\gamma_t =
\gamma_{\max}
\frac{D_t^2}{D_t^2+\kappa}
\]

with the initial discrepancy used to determine the sensitivity parameter \(\kappa\).
