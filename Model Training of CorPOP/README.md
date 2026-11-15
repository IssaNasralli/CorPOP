
# Model Training of CorPOP

This directory contains the implementation of the **CorPOP (Domain-Invariant Representation Learning via Second-Order Feature Alignment)** framework for fine-scale geospatial population estimation.

CorPOP extends the popVAT architecture by integrating **Correlation Alignment (CORAL)** into a dual-branch Variational Autoencoder (VAE) and atrous convolution framework. The objective is to learn transferable feature representations by aligning the second-order statistics of source- and target-domain feature distributions.

The implementation uses Tunisia as the source domain and Île-de-France, France, as the target domain. Target-domain population labels are not required for CORAL feature alignment.

## 1. Repository Structure

The directory contains the following Python scripts:

| File | Description |
|---|---|
| `test_CORAL.py` | Main training script. Handles dataset preparation, source-target batch generation, model optimization, validation, checkpointing, and population prediction. |
| `popVAE_full_Gate_Atrous_Gate_CORAL.py` | Defines the CorPOP model architecture, including the VAE encoder-decoder, gating mechanisms, atrous convolution branches, CORAL loss layer, data generators, and population prediction functions. |
| `data_atrous.py` | Provides geospatial data preprocessing utilities, including raster preprocessing, target data preparation, and administrative district mask loading. |

**Additional dependency:** The training script also imports `evaluate.py` (`import evaluate as ev`) for pixel-level and administrative-level evaluation. This file must be available in the Python import path, even if it is maintained outside this directory.

## 2. Dependencies and Environment

The framework is implemented using Python and TensorFlow/Keras.

Required Python packages:

```bash
tensorflow==2.15.0
numpy
pandas
scikit-learn
rasterio
```

Additional standard Python libraries used include:

```bash
os
random
gc
itertools
time
```

Python version: Python ≥ 3.8.

A GPU is recommended to accelerate model training and raster processing. Sufficient RAM is also recommended when processing large geospatial datasets.

### Installation

Install the required packages using:

```bash
pip install tensorflow==2.15.0 numpy pandas scikit-learn rasterio
```

## 3. Model Architecture

CorPOP adopts a dual-branch architecture that integrates spatial contextual information and domain adaptation.

The architecture consists of the following components:

### 3.1. Variational Autoencoder (VAE)

The VAE branch extracts spatial representations from medium-scale patches.

It consists of:
- A convolutional encoder.
- A latent representation characterized by its mean and log-variance.
- A sampling operation based on the reparameterization trick.
- A convolutional decoder that reconstructs the input patch.

### 3.2. Medium-Scale Gating Mechanism

A gating module (`G_m`) applies sigmoid activation to the latent representation to learn feature-dependent weights.

The resulting modulated latent features are used in the population prediction branch and CORAL alignment.

### 3.3. Atrous Convolution Branch

The global spatial context is extracted using parallel atrous convolution layers with dilation rates:

```python
atrous_rates = [1, 3, 11, 17]
```

Each branch uses 8 convolutional filters, followed by global average pooling.

The resulting feature vectors are concatenated and passed through a second gating module (`G_g`) using sigmoid activation.

### 3.4. Feature Fusion and Population Prediction

The final representation combines:

- Pixel-level ancillary features.
- Gated VAE latent features.
- Gated atrous convolution features.

These representations are concatenated and passed through convolutional layers and a global average pooling operation.

A final dense layer with softplus activation produces the population prediction.

## 4. Domain Adaptation Using CORAL

CorPOP incorporates CORrelation ALignment (CORAL) to reduce the distributional discrepancy between source- and target-domain feature representations.

The method aligns the covariance matrices of the fused source and target features.

Let:

- $C_s$ denote the source-domain feature covariance matrix.
- $C_t$ denote the target-domain feature covariance matrix.
- $d$ denote the dimensionality of the fused feature representation.

The CORAL loss is defined as:

$$
\mathcal{L}_{CORAL} =
\frac{1}{4d^2}
\left\|C_s-C_t\right\|_F^2
$$

where $\|\cdot\|_F$ represents the Frobenius norm.

The fused representations used for alignment consist of:

- Pixel-level ancillary features.
- Gated latent representations.
- Gated global contextual features.

In the current implementation, the CORAL loss is calculated using covariance alignment without an additional mean-alignment term.

### 4.1. Training Objective

The custom training procedure combines four loss components:

1. **Reconstruction loss:** Huber loss between the original and reconstructed source-domain patches.
2. **KL divergence:** Regularizes the VAE latent distribution.
3. **Population prediction loss:** Huber loss between predicted and reference source-domain population values.
4. **CORAL loss:** Aligns source- and target-domain feature covariance matrices.

The total loss is:

$$
\begin{aligned}
\mathcal{L}_{total} =\;&
\lambda_{rec}\mathcal{L}_{rec}
+\lambda_{KL}\mathcal{L}_{KL}\\
&+\lambda_{pred}\mathcal{L}_{pred}
+\lambda_{CORAL}\mathcal{L}_{CORAL}
\end{aligned}
$$

The default weights defined in `compute_total_loss()` are:

```python
weight_reconstruction = 1.0
weight_kl = 1.0
weight_prediction = 2.0
weight_coral = 0.1
```

The CORAL weight is fixed throughout the custom training loop.

## 5. Dataset Preparation

CorPOP uses two geographically distinct datasets.

### 5.1. Source Domain: Tunisia

The source-domain dataset contains the ancillary geospatial information and population ground truth required for supervised population estimation.

The main input files expected by the training script are:

- `tunisia10.tif`: Source-domain ancillary raster.
- `POP_Tunisia.tif`: Source-domain population raster.
- `ins_population.csv`: Population statistics from Tunisia's National Institute of Statistics (INS).

### 5.2. Target Domain: Île-de-France, France

The target-domain dataset is used to extract contextual features and calculate the CORAL alignment loss.

The main target input file is:

- `france10.tif`: Target-domain ancillary raster.

Target-domain population labels are not used in the custom CORAL training objective.

Administrative masks and other geospatial resources are loaded through the preprocessing functions implemented in `data_atrous.py`.

### 5.3. Data Preprocessing

The input rasters are processed using the functions provided by `data_atrous.py`.

The training script performs the following operations:

1. Loads and preprocesses the source-domain raster.
2. Loads source-domain administrative district masks.
3. Loads population statistics and the source population raster.
4. Loads and preprocesses the target-domain ancillary raster.
5. Extracts valid pixel coordinates for both domains.
6. Constructs medium-scale and global contextual patches.

Each training sample consists of:

- A medium-scale contextual patch.
- A global contextual patch.
- Pixel-level ancillary features.
- The corresponding source-domain population value.

Target-domain samples contain the corresponding contextual and pixel-level features but do not require population labels.

## 6. Training Configuration

The default training configuration is defined in the `main()` function of `test_CORAL.py`.

| Parameter | Value |
|---|---|
| Random seed | 42 |
| Source domain | Tunisia |
| Target domain | Île-de-France, France |
| Number of feature bands | 10 |
| Latent dimension | 30 |
| Medium patch size | 11 × 11 |
| Global patch size | 35 × 35 |
| Batch size | 1024 |
| Maximum epochs | 100 |
| Early stopping patience | 5 epochs |
| Optimizer | Adam |
| Learning rate | 0.001 |
| CORAL weight | 0.1 |
| Gating activation | Sigmoid |
| Atrous dilation rates | [1, 3, 11, 17] |

### 6.1. Dataset Splitting

The source-domain pixel indices are divided into three subsets using `train_test_split` with `random_state=42`.

The implemented splitting procedure is:

- **70% training**
- **15% validation**
- **15% testing**

The target-domain dataset is independently sampled and used to provide target feature batches during training.

## 7. Training Procedure

Training is performed using a custom GradientTape-based optimization procedure.

For each training iteration:

1. A source-domain batch is extracted from the training generator.
2. A target-domain batch is extracted from the target generator.
3. Both batches are passed through the shared VAE encoder.
4. Medium-scale and global gating mechanisms are applied.
5. Atrous convolution features are extracted from both domains.
6. Fused source and target representations are constructed.
7. The population prediction is calculated for the source domain.
8. The VAE reconstruction and KL divergence losses are calculated.
9. The CORAL loss is computed using source and target feature covariance matrices.
10. The total loss is backpropagated using TensorFlow's `GradientTape`.
11. Model parameters are updated using the Adam optimizer.

### 7.1. Validation and Checkpointing

After each training epoch:

- Target-domain feature representations are extracted.
- The validation generator is updated with the target-domain features.
- Validation loss is evaluated.
- The best model weights are saved.
- Early stopping is applied if the validation loss does not improve for five consecutive epochs.

Training progress is recorded in:

```text
training_log.txt
```

## 8. Running the Training Script

Before starting training, make sure that:

- All required Python dependencies are installed.
- The three main Python modules are available.
- The additional `evaluate.py` dependency is accessible.
- The source- and target-domain raster datasets are available.
- The required population data and administrative masks are accessible.

Run the training script using:

```bash
python test_CORAL.py
```

The script initializes the model, prepares the datasets, performs training and validation, saves the best model weights, and generates population predictions.

## 9. Output Files

The training script generates several output files.

| Output | Description |
|---|---|
| `best_weights_popVA_full_Huber_...h5` | Saved model weights associated with the best validation loss. |
| `model_architecture_popVAE_Huber_...json` | Serialized model architecture. |
| `training_log.txt` | Training and validation progress. |
| `*_predicted.tif` | Predicted population density raster. |
| `checkpoint_*.txt` | Stores the last processed raster coordinates to support prediction resumption. |
| `*_processed.tif` | Preprocessed ancillary raster. |

The exact filenames depend on the configuration specified in `test_CORAL.py`.

## 10. Population Prediction and Evaluation

After training, the best model weights are used to generate pixel-level population predictions.

The prediction procedure is implemented in:

```python
popVAE.predict_and_reconstruct()
```

Predictions are generated over the source-domain raster and saved as a GeoTIFF file.

The evaluation script is called through:

```python
ev.calculate_pixel_r2(target_data, predicted_image)

ev.calculate_district_r2(
    predicted_image,
    district_masks,
    ins_population
)
```

These functions assess prediction performance at pixel and administrative district levels.

## 11. Important Implementation Notes

- The current `test_CORAL.py` implementation uses a fixed CORAL regularization weight of 0.1 in its custom training procedure.
- The current implementation performs covariance-based CORAL alignment; it does not implement the adaptive CORAL coefficient described in the CorPOP paper.
- The source-domain training, validation, and testing proportions are 70%, 15%, and 15%, respectively, according to the supplied script.
- The target domain is used for unsupervised feature alignment without target population supervision in the custom training loop.
- The model architecture and training configuration can be modified directly in `test_CORAL.py` and `popVAE_full_Gate_Atrous_Gate_CORAL.py`.

---

**CorPOP: Domain-Invariant Representation Learning via Second-Order Feature Alignment: Application to Geospatial Population Estimation.**
