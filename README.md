# Adaptive Filtering for Active Noise Cancellation

Python implementations of adaptive filtering algorithms for denoising and active noise cancellation (ANC). The examples cover LMS, NLMS, adaptive line enhancement, feedforward noise cancellation, and filtered-x ANC.

## Notebooks

- `01_LMS_Foundations_and_ANC_Demos.ipynb`: LMS system identification, adaptive line enhancement, and feedforward noise cancellation.
- `02_NLMS_Normalized_Adaptive_Filtering.ipynb`: NLMS with LMS baselines under changing signal levels.
- `03_Filtered_x_Practical_ANC.ipynb`: FxLMS, FxNLMS, and an HVAC duct ANC case study.

Older combined notebooks are preserved in `archive_exploratory/` for reference.

## Adaptive Filter Model

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

where

$$
\mathbf{R}_{xx}=E\{\mathbf{x}[n]\mathbf{x}^T[n]\}
$$

and

$$
\mathbf{p}_{xd}=E\{d[n]\mathbf{x}[n]\}.
$$

In practice these statistics are usually unknown or changing, so LMS-style algorithms use the current sample to approximate the gradient.

## LMS

LMS is a stochastic gradient descent algorithm:

$$
\mathbf{w}_{n+1} = \mathbf{w}_n + \mu e[n]\mathbf{x}[n].
$$

The step size $\mu$ controls the tradeoff between convergence speed and steady-state error. Larger values generally converge faster but can become unstable, while smaller values are more stable but adapt more slowly.

A common stability guideline is

$$
0 < \mu < \frac{2}{\lambda_{max}},
$$

where $\lambda_{max}$ is the largest eigenvalue of $\mathbf{R}_{xx}$.

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

This makes the algorithm less sensitive to changes in input amplitude. The normalized step $\tilde{\mu}$ is dimensionless, with a common practical range of

$$
0 < \tilde{\mu} < 2.
$$

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

If $d[n]$ is tonal or predictable and $v[n]$ is broadband, the delay keeps $d[n]$ correlated while decorrelating much of the noise. The filter output therefore estimates the predictable component.

### Feedforward Noise Cancellation

Feedforward denoising uses a reference sensor correlated with the noise source:

$$
d[n] = s[n] + v_p[n],
$$

where $s[n]$ is the desired signal and $v_p[n]$ is the noise reaching the primary microphone.

The adaptive filter learns

$$
y[n] \approx v_p[n],
$$

so the cleaned output is

$$
e[n] = d[n] - y[n].
$$

This is a reference-based denoising model and a step toward physical ANC. The cancellation signal is still subtracted directly in software rather than being reproduced through a loudspeaker.

### Physical Feedforward ANC

In physical ANC, the controller output drives a loudspeaker. The anti-noise reaches the error microphone through a secondary path $S(z)$:

$$
y_s[n] = s[n] * y[n].
$$

The residual at the error microphone is

$$
e[n] = d[n] + y_s[n].
$$

Because the secondary path changes the gradient seen by the adaptive filter, the reference is filtered through an estimate of that path:

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

The negative sign follows from the residual definition $e[n]=d[n]+y_s[n]$: the loudspeaker output should destructively interfere with the primary disturbance.

## Practical Notes

- LMS is straightforward but requires tuning of the step size for a given input level.
- NLMS is less sensitive to changes in reference-signal power.
- Adaptive line enhancement works when the desired component is more predictable than the noise.
- Feedforward cancellation depends on a reference signal correlated with the disturbance.
- FxLMS and FxNLMS account for the loudspeaker-to-error-microphone secondary path.
- The quality of the secondary-path estimate affects filtered-x ANC performance.
- The notebooks use residual power and SNR-style metrics to evaluate cancellation.

## References

- Monson H. Hayes, *Statistical Digital Signal Processing and Modeling*. Wiley, 1996.
- Simon Haykin, *Adaptive Filter Theory*. Prentice Hall, 2002.
- S. M. Kuo and D. R. Morgan, *Active Noise Control Systems*. Wiley, 1996.
