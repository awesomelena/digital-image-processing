# Digital Image Processing — Homework Assignments

## Overview

This repository contains three Jupyter Notebook homework assignments completed for the Digital Image Processing course. Each assignment focuses on a different fundamental area of the field, progressing from color space manipulation and tone mapping, through frequency-domain filtering, to morphological segmentation and edge detection.

---

## Repository Structure

```
digital-image-processing/
├── homework1/
│   └── domaci1_22_351.ipynb   # Color spaces, HDR tone mapping, sharpening & CLAHE
├── homework2/
│   └── domaci2_22_351.ipynb   # Gaussian filtering, image restoration, noise estimation & self-guided filter
├── homework3/
│   └── domaci3_22_351.ipynb   # Coin segmentation & Canny edge detection
├── requirements.txt
├── .gitignore
└── README.md
```

> **Note:** Image sequences used as input data are stored locally in a `sekvence/` directory (not included in this repository due to file size). The notebooks reference them via `data_dir = "../sekvence/"`.

---

## Homework 1 — Color Space Manipulation, HDR Tone Mapping, Sharpening & CLAHE

**File:** `homework1/domaci1_22_351.ipynb`

### Part 1 — Bus Recoloring (Blue → Red / Green)

The task was to recolor blue city buses in photographs to red and green using color space analysis.

**Approach:**  
RGB was immediately ruled out because the spectra of blue, red, and green overlap significantly, making clean segmentation impossible. Instead, a **hybrid HSV + YUV masking** strategy was developed:

- **HSV mask** — isolates the blue hue range (`0.55 < H < 0.75`) while constraining saturation and brightness to avoid dark shadows and overexposed regions.
- **YUV mask** — the U channel proved critical for eliminating unwanted reflections on bus windows and wet asphalt, which HSV alone could not suppress.
- The **final mask** is the logical AND of both masks, and only the Hue channel is modified at masked pixels — saturation and brightness are preserved, producing natural-looking results.

Parameters were tuned experimentally per image. A **bonus implementation** for `gsp3.jpg` decomposes the bus into three separate sub-masks (`mask_body`, `mask_roof`, `mask_shadows`) combined with a stricter U-channel threshold, achieving near-artifact-free recoloring.

**Libraries:** `skimage.color` (rgb2hsv, rgb2yuv, hsv2rgb), `matplotlib`

---

### Part 2 — HDR Tone Mapping

The task was to map an HDR image (`sea.hdr`) into the displayable [0, 1] range using multiple methods, with gamma correction (γ = 1/2.2) applied throughout.

**Methods implemented:**

| Method | Description |
|--------|-------------|
| **Linear with saturation** | Clips at the `(100 − s)th` percentile; interactive slider for saturation percentage. Parameters A = 1, B ≈ 12.31, C ≈ 36.35 were identified. |
| **Logarithmic** | `c · log(1 + img)` — compresses highlights smoothly. |
| **Exponential** | `1 − exp(−c · img)` — soft shoulder at high intensities. |
| **Drago** | Adaptive logarithmic operator for perceptually uniform brightness. |
| **Reinhard** | Global photographic tone mapping: `img / (1 + img)`. |

Each method's parameters were explored interactively using `ipywidgets.interact`. Qualitative trade-offs (highlight compression vs. shadow visibility) were analyzed and documented.

**Libraries:** `imageio.v2`, `skimage.exposure`, `ipywidgets`

---

### Part 3 — Image Sharpening (Unsharp Masking)

The task was to sharpen `miner.jpg` in a natural-looking way, without over-emphasizing edges to the point of looking artificial.

**Approach:**  
Sharpening was applied in the **YUV color space**, operating only on the Y (luminance) channel to avoid introducing color artifacts that would occur if sharpening were applied directly in RGB. The **unsharp masking** method was used:

1. The Y channel is blurred with a Gaussian filter (σ = 1.3).
2. The difference between the original and blurred Y channel gives a detail/edge map.
3. This detail map is added back to the original Y channel, scaled by an `amount` factor (1.6).
4. The result is clipped to [0, 1] and converted back to RGB.

Parameters were tuned interactively using `ipywidgets.interact`.

**Libraries:** `skimage.filters`, `skimage.color`, `ipywidgets`

---

### Part 4 — Custom CLAHE Implementation (`dosCLAHE`)

The task was to implement Contrast Limited Adaptive Histogram Equalization (CLAHE) from scratch, including image padding and bilinear interpolation.

**Implementation details:**

- **`image_padding`** — pads the image by repeating the last rows and columns so that it becomes evenly divisible by the tile grid (`numTiles`). This avoids the black-border artifact that zero-padding would introduce, and prevents unwanted effects in the histograms of border tiles.
- **`dosCLAHE(imgIn, numTiles, limit)`** — the main function:
  - For **color images**, only the **V channel** of the HSV representation is processed; H and S are left unchanged. This enhances contrast without distorting colors.
  - The padded image is divided into a `numTiles[0] × numTiles[1]` grid. A local CDF (with clip limit) is computed for each tile.
  - **Bilinear interpolation** between the four surrounding tile CDFs is fully **vectorized** using coordinate matrices, avoiding nested loops and significantly speeding up execution.
  - Results were compared against `skimage.exposure.equalize_adapthist` both visually and by execution time across different tile sizes and clip limits.

**Libraries:** `skimage.exposure`, `skimage.color`, `numpy`, `time`

---

## Homework 2 — Frequency-Domain Filtering, Image Restoration, Noise Estimation & Self-Guided Filter — Gaussian Filtering in the Frequency Domain

**File:** `homework2/domaci2_22_351.ipynb`

### Part 1 — Analytical Derivation

The 2-D discrete Fourier transform of a separable Gaussian was derived analytically. Starting from the continuous Gaussian and applying the Fourier transform, the key result is:

$$F(u,v) = 2\pi\sigma_r^{S}\sigma_c^{S} \cdot \exp\!\left(-2\pi^2\left(\frac{(\sigma_r^{S})^2 u^2}{N_H^2} + \frac{(\sigma_c^{S})^2 v^2}{N_W^2}\right)\right)$$

This gives the relationship between spatial and frequency-domain standard deviations:

$$\sigma_r^{F} = \frac{N_H}{2\pi\sigma_r^{S}}, \qquad \sigma_c^{F} = \frac{N_W}{2\pi\sigma_c^{S}}$$

### Part 2 — Image Restoration (Wiener Filter)

The task was to restore a motion-blurred image (`etf_blur.png`) using its known blur kernel (`blur_kernel.txt`).

**Approach:**  
Restoration was performed in the **frequency domain** using a **Wiener filter**:

- The image is first converted from sRGB to linear light (γ = 2.2 removed) before processing.
- To reduce ringing artifacts caused by FFT's implicit periodicity assumption, the image is **mirror-padded** before transforming.
- The blur kernel is embedded in a zero matrix of the same size as the padded image, then **shifted** with `np.roll` to move its origin to (0,0) as required by FFT convention.
- The Wiener filter is: `W = |H|² / (|H|² + K)`, where K is the inverse SNR. The optimal value K = 0.0005 was found experimentally — larger K over-smooths, smaller K amplifies noise.
- After restoration, the padding is removed and gamma correction is reapplied.

**Libraries:** `numpy.fft`, `scipy.ndimage`, `skimage.util`

---

### Part 3 — Noise Variance Estimation (`estimate_noise_var`)

The task was to estimate the variance of additive Gaussian noise in an image without knowing the noise level in advance.

**Approach:**  
The function `estimate_noise_var` computes **local variance** in small windows across the image, then finds the **peak of the local variance histogram**. The key insight is that in smooth (uniform) image regions, local variance is dominated by noise, while textured regions produce much higher variance. The histogram peak therefore corresponds to the noise variance.

- A **Bessel correction** is applied to get an unbiased estimate.
- The top 1% of variance values are excluded to prevent edge pixels from skewing the histogram.
- In **automatic mode** (win=0), the window size grows from 3×3 upward until the estimate stabilizes within a tolerance, balancing precision and sensitivity to image content.
- Robustness was analyzed across different window sizes and noise levels: larger windows give a tighter, more peaked histogram (lower estimation variance), while higher noise levels introduce more bias.

**Libraries:** `scipy.ndimage`, `skimage.filters`, `numpy`

---

### Part 4 — Self-Guided Filter & Fast Self-Guided Filter

The task was to implement both the standard and fast variants of the Self-Guided Filter, following the reference papers provided with the assignment, and compare their quality and speed.

**`dos_self_guided_filter(img, R, epsilon)`:**  
Implements the standard self-guided filter using `ndi.convolve` with a box kernel of size `(2R+1)×(2R+1)`. For each pixel, local mean and variance are computed, then smoothing coefficients `a` and `b` are derived (`a = var / (var + ε)`, `b = mean − a·mean`) and averaged over the window. `ndi.convolve` was intentionally used instead of `ndi.uniform_filter` to explicitly demonstrate the O((2R+1)²) per-pixel complexity.

**`dos_fast_self_guided_filter(img, R, epsilon, s)`:**  
The fast variant reduces computation by **decimating** the image by factor `s`, computing the filter coefficients on the smaller image, and **upsampling** back to the original resolution via bilinear interpolation. This approximation is valid because the coefficient maps are smooth and low-frequency. A helper `reduce_box` function performs the decimation by block-averaging.

**Comparison:**  
- PSNR between standard and fast outputs was analyzed across a range of R and ε values. Larger R → higher PSNR (less approximation error) because heavy smoothing is low-frequency and survives decimation well.
- The characteristic **zig-zag pattern** in PSNR vs. R graphs is a discretization effect: PSNR peaks when R is divisible by s (the decimated filter geometry aligns perfectly with the original), and dips otherwise due to rounding of the sub-sampled radius.
- Execution time was compared on `lena.tif` and `einstein.tif`, showing significant speedup of the fast variant for large R.

**Libraries:** `scipy.ndimage`, `numpy`, `skimage`

---

## Homework 3 — Coin Segmentation & Canny Edge Detection

**File:** `homework3/domaci3_22_351.ipynb`

### Part 1 — Robust Coin Segmentation & Counting

The task was to segment Serbian coins (1-dinar and 5-dinar) from a green background, count them, and classify them by size.

#### `coin_mask` — Segmentation

A **hybrid color + texture** approach was designed to handle non-uniform illumination and reflections:

1. **Median pre-filtering** (3×3) reduces impulse noise while preserving edges.
2. **HSV color analysis** — the dominant background hue is found automatically from the histogram. A hue range covering ≥20% of the dominant bin's count is labeled background. An **Otsu threshold on background saturation** (×0.5) further refines the color mask. Everything outside the background color range is a coin candidate.
3. **Texture analysis** — local standard deviation is computed in a window sized to 0.5% of the image's smaller dimension. Pixels above 30% of Otsu's texture threshold are flagged as textured. The texture mask is **morphologically dilated** (radius ≈ 2.5% of image size) to fill coin interiors from their textured edges inward.
4. **Combined mask** = `color_mask AND dilated_texture_mask` — both conditions must hold simultaneously, eliminating background scratches (wrong texture) and smooth regions with unusual color (wrong color).
5. **Morphological cleanup** — opening → hole-filling → closing → hole-filling, with kernel sizes scaled to image resolution.
6. **Border removal** — any region touching the image boundary is discarded (incomplete coins).

All parameters scale relative to image dimensions, making the algorithm **resolution-independent**.

#### `bw_label` — Connected Component Labeling

A custom BFS-based connected component labeler using **8-connectivity** was implemented from scratch (without `scipy.ndimage.label`). 8-connectivity was chosen because 4-connectivity can incorrectly split circular objects along diagonal boundaries.

#### `coin_classification` — Size Classification

Coins are classified by region area (larger area → 5-dinar):

- Regions smaller than 45% of the median area or with circularity < 0.60 are discarded as noise.
- The area threshold between classes is found using the **maximum relative jump** method: the sorted area sequence is scanned for the largest ratio between consecutive values, subject to both groups having at least 10% of total coins.
- **Otsu fallback** on the area array is used when no clear gap exists.

Circularity is computed as $\frac{4\pi A}{P^2}$ (= 1 for a perfect circle).

---

### Part 2 — Canny Edge Detection

A full Canny edge detector was implemented from scratch:

1. **Gaussian smoothing** — configurable σ.
2. **Gradient computation** — Sobel operators for ∂x and ∂y; magnitude and angle calculated.
3. **Non-maximum suppression (NMS)** — gradient direction quantized to 4 directions (0°, 45°, 90°, 135°); local maxima retained, others suppressed.
4. **Double thresholding** — pixels above `high` are strong edges; pixels between `low` and `high` are weak edges.
5. **Hysteresis edge tracking** — BFS from strong edge seeds, accepting connected weak edges.

Results were validated against `skimage.feature.canny`. Differences are attributed to skimage's sub-pixel gradient interpolation during NMS and internal magnitude normalization. The custom implementation faithfully follows the theoretical algorithm from lectures.

**Parameter sensitivity analysis** was performed on `lena.tif` and `cameraman.tif`, varying σ (0.5–3.0) and threshold pairs, with documented trade-offs.

**Libraries:** `skimage.morphology`, `skimage.filters`, `scipy.ndimage`, `collections.deque`, `pandas`

---

## Setup & Usage

### Prerequisites

Python 3.8+ is recommended.

```bash
git clone https://github.com/<your-username>/digital-image-processing.git
cd digital-image-processing
pip install -r requirements.txt
```

### Image data

Place the required image sequences in a `sekvence/` folder one level above the repository root (i.e. at `../sekvence/` relative to each notebook), or update `data_dir` inside each notebook to point to your local image directory.

### Running the notebooks

```bash
jupyter notebook
```

Then open the desired notebook from the `homework1/`, `homework2/`, or `homework3/` folder.

---

## Key Techniques Summary

| Topic | Homework |
|---|---|
| Color space conversion (RGB, HSV, YUV) | HW1 |
| Image segmentation via color masking | HW1 |
| HDR tone mapping (linear, log, Reinhard, Drago) | HW1 |
| Gamma correction | HW1 |
| 2-D DFT of Gaussian — analytical derivation | HW2 |
| Frequency-domain Gaussian low-pass filtering | HW2 |
| Isotropic vs. anisotropic filters | HW2 |
| Spatial vs. frequency-domain performance comparison | HW2 |
| Hybrid color+texture segmentation | HW3 |
| BFS-based connected component labeling | HW3 |
| Otsu thresholding & relative-jump classification | HW3 |
| Canny edge detection from scratch | HW3 |
| Non-maximum suppression & hysteresis tracking | HW3 |
