# Adaptive Filtering for Active Noise Cancellation

This repository demonstrates core adaptive filtering algorithms used in denoising and active noise cancellation (ANC). It is organized for quick review: the README gives the mathematical map, and the notebooks contain straightforward, standalone implementations.

The project intentionally keeps the code simple. Each notebook includes its own imports, signal generation, adaptive-filter loop, metrics, and plots. There are no helper modules or setup scripts to inspect before the algorithms make sense.

## What This Shows

- FIR adaptive filter implementation from first principles.
- LMS and NLMS update rules with clear signal models and learning curves.
- System identification, one-sensor adaptive line enhancement, and two-sensor feedforward denoising.
- Practical ANC structure using FxLMS/FxNLMS with primary paths, secondary paths, and secondary-path mismatch.
- Quantitative evaluation using residual power and SNR-style improvement metrics.

## Notebooks

| Notebook | Scope | Why it matters |
|---|---|---|
| [`01_LMS_Foundations_and_ANC_Demos.ipynb`](01_LMS_Foundations_and_ANC_Demos.ipynb) | LMS system identification, adaptive line enhancement, and feedforward noise cancellation | Shows the baseline adaptive-filter loop in three increasingly ANC-like settings. |
| [`02_NLMS_Normalized_Adaptive_Filtering.ipynb`](02_NLMS_Normalized_Adaptive_Filtering.ipynb) | NLMS with LMS baselines under changing signal levels | Shows why normalization improves tuning and stability when input power changes. |
| [`03_Filtered_x_Practical_ANC.ipynb`](03_Filtered_x_Practical_ANC.ipynb) | FxLMS, FxNLMS, and an HVAC duct case study | Shows the practical ANC correction needed when a loudspeaker and acoustic secondary path are present. |

Older combined notebooks are preserved in [`archive_exploratory/`](archive_exploratory/) for reference. The three numbered notebooks above are the recruiter-facing demo path.

## Core Theory

Most examples use an adaptive FIR filter with tap vector

$$
\mathbf{x}[n] =
\begin{bmatrix}
x[n] & x[n-1] & \cdots & x[n-M+1]
\end{bmatrix}^T
$$

and coefficient vector

$$
\mathbf{w}_n =
\begin{bmatrix}
w_0[n] & w_1[n] & \cdots & w_{M-1}[n]
\end{bmatrix}^T.
$$

The filter output is

$$
y[n] = \mathbf{w}_n^T\mathbf{x}[n].
$$

For system identification and direct noise estimation, the error is

$$
e[n] = d[n] - y[n].
$$

The optimization target is mean-square error:

$$
J(\mathbf{w}) = E\{e^2[n]\}.
$$

For a stationary input, the optimal Wiener solution satisfies

$$
\mathbf{R}_{xx}\mathbf{w}_o = \mathbf{p}_{xd},
$$

where $\mathbf{R}_{xx}=E\{\mathbf{x}[n]\mathbf{x}^T[n]\}$ and $\mathbf{p}_{xd}=E\{d[n]\mathbf{x}[n]\}$. In practice these statistics are usually unknown or changing, so LMS-style algorithms use the current sample to approximate the gradient.

## LMS

LMS is stochastic gradient descent on $J(\mathbf{w})$:

$$
\mathbf{w}_{n+1} = \mathbf{w}_n + \mu e[n]\mathbf{x}[n].
$$

The step size $\mu$ controls the tradeoff:

| Larger $\mu$ | Smaller $\mu$ |
|---|---|
| Faster convergence | Lower misadjustment |
| Better tracking of changes | More stable under high input power |
| Higher risk of divergence | Slower adaptation |

A common stability guideline is

$$
0 < \mu < \frac{2}{\lambda_{max}},
$$

where $\lambda_{max}$ is the largest eigenvalue of $\mathbf{R}_{xx}$. In simple demos, this is often approximated using input power and filter length.

## NLMS

NLMS normalizes the update by the instantaneous input-vector energy:

$$
\mathbf{w}_{n+1} =
\mathbf{w}_n +
\frac{\tilde{\mu}}{\epsilon + \|\mathbf{x}[n]\|^2}
e[n]\mathbf{x}[n].
$$

The effective step size is

$$
\mu_{eff}[n] =
\frac{\tilde{\mu}}{\epsilon + \|\mathbf{x}[n]\|^2}.
$$

This makes the algorithm less sensitive to amplitude changes. The normalized step $\tilde{\mu}$ is dimensionless, and a common practical range is

$$
0 < \tilde{\mu} < 2.
$$

NLMS is useful when reference microphone power changes, when a fixed LMS step size is hard to tune, or when a demo needs robust behavior across multiple signal levels.

## ANC Signal Models

### Adaptive Line Enhancer

The adaptive line enhancer uses one sensor:

$$
x[n] = d[n] + v[n].
$$

The adaptive filter receives a delayed copy of the same signal:

$$
\mathbf{x}_\Delta[n] =
\begin{bmatrix}
x[n-\Delta] & x[n-\Delta-1] & \cdots
\end{bmatrix}^T.
$$

If $d[n]$ is tonal or predictable and $v[n]$ is broadband, the delay keeps $d[n]$ correlated while decorrelating much of the noise. The filter output estimates the predictable component.

### Feedforward Noise Cancellation

Feedforward denoising uses a reference sensor correlated with the noise source:

$$
d[n] = s[n] + v_p[n],
$$

where $s[n]$ is the desired signal and $v_p[n]$ is the noise reaching the primary microphone. The adaptive filter learns

$$
y[n] \approx v_p[n],
$$

so the cleaned output is

$$
e[n] = d[n] - y[n].
$$

This is a useful denoising model and a conceptual stepping stone toward ANC. It is not yet a physical anti-noise controller because the cancellation signal is subtracted directly in software.

### Physical Feedforward ANC

In real ANC, the controller output drives a loudspeaker. The anti-noise reaches the error microphone through a secondary path $S(z)$:

$$
y_s[n] = s[n] * y[n].
$$

The residual at the error microphone is

$$
e[n] = d[n] + y_s[n].
$$

Because the secondary path changes the gradient seen by the adaptive filter, the reference must be filtered through an estimate of that path:

$$
x'[n] = \hat{s}[n] * x[n].
$$

FxLMS uses

$$
\mathbf{w}_{n+1} = \mathbf{w}_n - \mu e[n]\mathbf{x}'[n],
$$

and FxNLMS uses

$$
\mathbf{w}_{n+1} =
\mathbf{w}_n -
\frac{\tilde{\mu}}{\epsilon + \|\mathbf{x}'[n]\|^2}
e[n]\mathbf{x}'[n].
$$

The sign is negative here because the residual is defined as $e[n]=d[n]+y_s[n]$: the loudspeaker output should destructively interfere with the primary disturbance.

## Method Comparison

| Method | Sensors/paths | Main advantage | Main limitation | Good use cases |
|---|---|---|---|---|
| LMS system identification | Known input, measured output | Clear demonstration of adaptive FIR learning | Requires step-size tuning | Plant modeling, channel estimation, DSP fundamentals |
| LMS adaptive line enhancer | One noisy sensor plus delay | Works without a reference microphone | Assumes the wanted signal is more predictable than the noise | Tone enhancement, rotating-machine signals |
| LMS feedforward cancellation | Reference mic and primary mic | Simple reference-based denoising | Fixed step is sensitive to reference power | Intro ANC demos, sensor denoising |
| NLMS | Same structures as LMS | Normalizes for changing input power | Slightly more computation | Variable-level references, robust demos |
| FxLMS | Reference mic, ANC speaker, error mic, secondary-path model | Corrects the physical secondary-path gradient | Needs a good $\hat{S}(z)$ and careful $\mu$ | Practical feedforward ANC foundations |
| FxNLMS | Same as FxLMS | Adds normalization to Filtered-x adaptation | Still depends on secondary-path estimate quality | Headphones, ducts, exhaust systems, machinery noise |

## Practical Use Cases

| Application | Typical reference | Preferred approach |
|---|---|---|
| Audio or sensor denoising | Reference noise channel | LMS/NLMS feedforward cancellation |
| Narrowband tone recovery | Delayed copy of one sensor | LMS/NLMS adaptive line enhancer |
| Headphones and earbuds | External feedforward mic, internal error mic | FxLMS/FxNLMS, often hybridized with feedback |
| HVAC duct noise | Upstream microphone or fan tachometer | FxLMS/FxNLMS |
| Automotive engine-order control | RPM/tachometer-derived harmonic reference | Frequency-locked FxLMS/FxNLMS variants |
| Exhaust and machinery cancellation | Upstream mic or periodic reference | FxNLMS with secondary-path modeling |

## References

- Monson H. Hayes, *Statistical Digital Signal Processing and Modeling*. Wiley, 1996.
- Simon Haykin, *Adaptive Filter Theory*. Prentice Hall, 2002.
- S. M. Kuo and D. R. Morgan, *Active Noise Control Systems*. Wiley, 1996.
