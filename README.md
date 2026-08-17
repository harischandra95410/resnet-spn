# Image Denoising with Residual Autoencoder (RAE) – Full Code & Results

This repository provides a complete implementation of an image denoising experiment for brain MRI. The custom **Residual Autoencoder (RAE)** is compared against three baseline models: **DnCNN**, **U‑Net**, and **Transformer‑U‑Net**. The project covers data preparation, noise synthesis, model definition, training, ablation studies, statistical testing, and visualisation.

---

## 1. Setup and Data Loading

The experiment starts by importing all required libraries, including TensorFlow, OpenCV, scikit‑image, and standard data science tools. To ensure reproducibility, a fixed random seed (42) is set and mixed‑precision training (`mixed_float16`) is enabled for faster computation.

Images are loaded from the `train/` and `valid/` folders, resized to 256×256 pixels, and normalised to the range `[0,1]`. The training set is then split into 80% for training and 20% for validation.

Several noise types are implemented: salt‑and‑pepper (SPN), Gaussian, Rician, and Poisson. For salt‑and‑pepper noise, the function creates a random mask and sets a fraction of pixels to either 0 or 1 according to the specified density. Three SPN levels (10%, 20%, and 30%) are generated for both training and validation sets and stored in dictionaries for later use.

---

## 2. Model Architectures

### Residual Autoencoder (RAE)

The RAE is built with an encoder–decoder structure:

- **Encoder**: Four convolutional blocks, each optionally including a residual connection and batch normalisation, followed by max‑pooling layers. The number of filters increases progressively, ending with a bottleneck of 512 filters.
- **Decoder**: Four up‑sampling blocks that use transposed convolutions. Skip connections from the encoder to the corresponding decoder layers can be optionally enabled.
- **Output**: A single‑channel sigmoid activation to produce the denoised image.

The architecture supports toggling skip connections and residual blocks for ablation purposes.

### DnCNN

A plain feed‑forward convolutional network with 9 layers, each using 64 filters of size 3×3, batch normalisation, and ReLU activation. A residual connection adds the input directly to the output, allowing the network to learn the noise residual.

### U‑Net

A classic symmetric encoder‑decoder with skip connections. The encoder uses max‑pooling and increasing filter counts, while the decoder uses up‑sampling and concatenation with corresponding encoder feature maps.

### Transformer‑U‑Net

This variant extends the standard U‑Net by inserting two transformer blocks in the bottleneck. Each transformer block contains a multi‑head self‑attention layer and a feed‑forward network, applied after reshaping the feature map into a sequence.

---

## 3. Loss Functions and Helpers

Two loss functions are used:

- **Tukey loss**: A robust loss with a tunable constant `c` (default 4.685). It behaves like a quadratic loss for small errors but saturates for large residuals, making it less sensitive to outliers.
- **Mean Squared Error (MSE)**: Used for baseline models.

Evaluation metrics are **PSNR** (Peak Signal‑to‑Noise Ratio) and **SSIM** (Structural Similarity Index), computed on the test set. For statistical comparison between models, the **Wilcoxon signed‑rank test** is employed.

---

## 4. Training the Main RAE

The full RAE (with skip connections and residual blocks) is trained on the 30% salt‑and‑pepper noise dataset. It uses the Tukey loss function and is compiled with an appropriate optimiser. The training process includes:

- **ReduceLROnPlateau**: to reduce the learning rate when validation loss plateaus.
- **EarlyStopping**: to halt training if no improvement is seen.
- **ModelCheckpoint**: to save the best weights.

The model is trained for up to 30 epochs with a batch size of 8. Training and validation loss as well as mean absolute error are recorded, and the training curves are saved as an image file.

---

## 5. Ablation Studies

### Architecture Ablation

Four variants of the RAE are evaluated on the 30% SPN task:

1. **No Skip Connections** – skip connections are removed.
2. **No Residual Blocks** – residual blocks in the encoder are disabled.
3. **Tukey → MSE** – the loss is switched from Tukey to MSE.
4. **Standard AE + MSE** – a plain autoencoder without skip connections, residual blocks, and trained with MSE.

Each variant is trained and its performance (PSNR and SSIM) is recorded in a CSV file.

### Tukey Constant Ablation

The constant `c` in the Tukey loss is swept over three values: 3.0, 4.685 (default), and 6.0. The resulting performance metrics are saved for comparison.

---

## 6. Training Baselines

DnCNN, U‑Net, and Transformer‑U‑Net are trained on all three SPN levels (10%, 20%, and 30%) using the MSE loss. For fairness, the RAE models for 10% and 20% SPN are newly trained (the 30% model is reused from the main training). All models are compiled and trained with the same callbacks and hyperparameters as the main RAE.

---

## 7. Full Evaluation and Statistics

All trained models are evaluated on the held‑out test set. For each noise level, the mean and standard deviation of PSNR and SSIM across test images are computed. The Wilcoxon signed‑rank test is then performed to compare the RAE against each baseline at each noise level, producing p‑values and performance differences.

Additionally, the generalisation capability of the RAE (trained only on 30% SPN) is tested on unseen noise types: Gaussian, Rician, and Poisson. Full evaluation results are compiled into a comprehensive CSV file.

---

## 8. Computational Efficiency

Inference time and model parameter counts are measured for all models. Inference time is averaged over multiple runs (with a warm‑up phase) on a dummy input of the same shape as the test images. The results are presented in a table for quick comparison of speed and model size.

---

## 9. Visualisations

Two types of visualisations are generated:

- **Sample restoration**: A side‑by‑side comparison of the noisy input, the RAE’s denoised output, and the ground‑truth image for a few representative test samples.
- **Performance curves**: Plots showing PSNR and SSIM as a function of noise level for all models, illustrating relative performance trends.

Both figures are saved as PNG files.

---

## 10. Final Summary

The final cell prints a concise summary of all generated output files and highlights the key improvements achieved by the RAE over the baselines, as observed in the results.

---

## 📊 Results Summary

### SPN Performance (PSNR / SSIM)

| Model               | 10% SPN               | 20% SPN               | 30% SPN               |
|---------------------|-----------------------|-----------------------|-----------------------|
| **RAE**             | 33.3±1.90 / 0.911     | 32.5±1.80 / 0.915     | **33.3±2.18 / 0.924** |
| DnCNN               | 33.6±1.87 / 0.917     | 31.7±1.76 / 0.836     | 29.2±1.77 / 0.816     |
| U‑Net               | 36.9±2.31 / 0.973     | 34.0±2.23 / 0.944     | 32.7±2.34 / 0.928     |
| Transformer‑U‑Net   | 35.8±2.17 / 0.967     | 33.6±2.25 / 0.946     | 32.0±2.29 / 0.928     |

### Architecture Ablation (30% SPN)

| Variant                | PSNR (dB) | SSIM   |
|------------------------|-----------|--------|
| **Full RAE**           | **33.27** | **0.924** |
| No Skip Connections   | 24.36     | 0.721   |
| No Residual Blocks    | 27.81     | 0.813   |
| Tukey → MSE Loss      | 30.22     | 0.881   |
| Standard AE + MSE     | 23.13     | 0.632   |

### Tukey Constant `c` Ablation

| `c`  | PSNR (dB) | SSIM   |
|------|-----------|--------|
| 3.0  | 30.67     | 0.884  |
| 4.685| 30.22     | 0.882  |
| 6.0  | 30.33     | 0.868  |

### Generalisation (RAE trained on 30% SPN)

| Noise Type | PSNR (dB) | SSIM  |
|------------|-----------|-------|
| Gaussian   | 22.5      | 0.450 |
| Rician     | 22.4      | 0.561 |
| **Poisson**| **30.1**  | **0.809** |

### Computational Efficiency

| Model               | Params    | Inference (ms) |
|---------------------|-----------|----------------|
| **RAE**             | 11.7M     | 67.47          |
| DnCNN               | 298k      | 61.84          |
| U‑Net               | 7.78M     | 66.08          |
| Transformer‑U‑Net   | 7.00M     | 66.04          |

### Statistical Significance (Wilcoxon, 30% SPN)
- **RAE vs DnCNN**: Δ = +4.10 dB, p < 0.001  
- **RAE vs U‑Net**: Δ = +0.61 dB, p < 0.001  
- **RAE vs Transformer‑U‑Net**: Δ = +1.24 dB, p < 0.001

---

## 📁 Output Files

All results are saved to `/kaggle/working/0601/`:

| File                         | Description |
|------------------------------|-------------|
| `Full_Results.csv`           | Full evaluation metrics |
| `ablation_architecture.csv`  | Architecture ablation |
| `ablation_tukey_c.csv`       | Tukey constant sweep |
| `training_curves_rae_30.png` | Loss/MAE curves |
| `sample_restoration.png`     | Visual comparison |
| `performance_curves.png`     | Performance vs. noise level |
| `best_rae_model.keras`       | Trained RAE weights |

---

## 🛠️ Dependencies

The following packages are required (versions specified are recommended):

- TensorFlow (>=2.20.0)
- NumPy (>=1.26.0)
- OpenCV‑Python (>=4.5.0)
- scikit‑image (>=0.21.0)
- scikit‑learn (>=1.5.0)
- SciPy (>=1.10.0)
- Pandas (>=2.0.0)
- Matplotlib (>=3.5.0)

---

## 🚀 How to Run

1. Place your dataset in the expected directory structure (see below).  
2. Execute the notebook cells in order (1–10).  
3. Results and figures will be saved in `/kaggle/working/0601/`.

### Dataset Structure
/path/to/dataset/
├── train/ (2803 images)
└── valid/ (203 images)


All images are resized to 256×256 and normalised to `[0,1]`.

---

## 📄 License

This project is for research purposes.  
Use of the BraTsss dataset must comply with its original license.