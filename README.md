# lecture9lab3
Lab 3: Frequency-Domain Filtering with fft2
1) Magnitude spectrum (Figure 1)

fft2 breaks the image into 2D sinusoids. Each value F(u,v) gives how strongly one spatial frequency is present.

fftshift moves the zero-frequency (DC) term to the centre of the spectrum.
log(1 + |F|) compresses the range of values. The DC term is several orders of magnitude larger than everything else, so without the log you would see one white dot on a black background.

What the spectrum shows:

A bright centre with brightness falling off outward. Most of the image’s energy is in the low frequencies: smooth areas and slow changes in brightness.
High frequencies (edges, fine texture, noise) sit toward the outside and are much weaker.
Faint horizontal and vertical lines through the centre. The FFT treats the image as periodic. Where the left edge meets the right edge, and the top meets the bottom, there is a sudden jump in brightness, and those jumps create the lines. Strong horizontal and vertical structures in the scene, such as the table edge, add to them.
2) Ideal & Gaussian low-pass filters in frequency

D(u,v) is the distance of each frequency from the centre.

Ideal LP: H = 1 if D ≤ D0 = 40, otherwise 0. This is a hard circular cutoff, a cylinder.
Gaussian LP: H = exp(−D² / 2σ²) with σ = 20. The response falls off smoothly, with no sharp edge.
3) Applying the low-pass filters (Figure 2)

The filtering is done by multiplying in the frequency domain: G = H · F. Then ifftshift and ifft2 take the result back to an image. By the convolution theorem, this gives the same result as convolving in the spatial domain:

f * h ⟷ F · H

Ideal LP: blurred, with ringing: faint ripples or halos run parallel to strong edges such as the garlic outlines, the table edge and the pepper contours.
Gaussian LP: smooth, natural blur with no ringing. It looks blurrier than the ideal LP because the Gaussian has already fallen to 13.5% at D = 40 (exp(−2)).
real() is needed because rounding leaves tiny imaginary parts (around 1e-16) after ifft2.
4) High-pass via complement (Figure 3)

H_HP = 1 − H_LP keeps the high frequencies and removes the low ones, including DC.

The result keeps edges and fine texture only. Flat areas such as the background and the smooth surfaces of the peppers become uniform.
It looks gray with an embossed appearance. Removing DC makes the mean 0, and mat2gray then maps 0 to mid-gray. Edges appear as light and dark lines on either side.
A high-pass filter is the frequency-domain equivalent of the edge detectors from Lab 2.
5) Spatial vs frequency-domain Gaussian LP (Figure 4)

The two results should match, but the frequency-domain version is clearly blurrier. This is because the two filters have different strengths:

Spatial: σ_s = 1.2 pixels, a light blur.
Frequency: σ_f = 20 frequency units, a much stronger blur.

A Gaussian stays a Gaussian under the Fourier transform, and its width is inverted: σ_f = N / (2π·σ_s). A narrow spatial Gaussian becomes a wide frequency Gaussian. So σ_f = 20 corresponds to σ_s ≈ 512/(2π·20) ≈ 4.1 px horizontally and 384/(2π·20) ≈ 3.1 px vertically. The image is not square, so u and v use different frequency units, and the filter is not quite isotropic.

Fix: build the frequency filter from the same σ_s as the spatial one:

matlab
sigma_s = 1.2;
H_match = exp(-2*pi^2*sigma_s^2 * ((u/N).^2 + (v/M).^2));
G_match = real(ifft2(ifftshift(H_match .* Fshift)));
figure; montage({I_spatial_gauss, G_match}, 'Size', [1 2]);
title('Spatial Gaussian LP | Matched frequency-domain Gaussian LP');
fprintf('Max difference: %g\n', max(abs(I_spatial_gauss(:) - G_match(:))));

With this change the two images look identical. Small differences remain only:

at the borders, because the FFT wraps around (circular) while imfilter used 'replicate', and
from the truncation of the 7×7 spatial kernel.

In your report, include the new figure and explain why the original pair didn’t match. That explanation shows you understand the convolution theorem.

Reflections

1) Why does an ideal low-pass filter cause ringing (Gibbs phenomenon)?
The ideal LP is a sharp-edged cylinder in the frequency domain. Its inverse Fourier transform, the spatial impulse response, is a 2D sinc (jinc) function with a central peak and oscillating sidelobes that alternate in sign and decay slowly. Multiplying in frequency is the same as convolving the image with this kernel, so each sharp edge is copied with ripples beside it, and you see overshoot and ringing next to edges. This is the Gibbs phenomenon: when a signal with discontinuities is rebuilt from a truncated set of frequencies, it overshoots by about 9% near each jump. Raising D0 makes the ripples narrower, but the overshoot does not go away. A Gaussian has no hard cutoff, its transform is another Gaussian with no sidelobes, and so it produces no ringing.

2) What does fftshift do visually?
fft2 puts the DC (zero-frequency) term at the top-left corner (index 1,1), with the low frequencies spread over the four corners. fftshift swaps the quadrants diagonally, which moves DC to the centre. Low frequencies then sit in the middle and high frequencies at the edges. The spectrum becomes symmetric and much easier to read, and a radial filter can be built directly from the distance D measured from the centre. ifftshift reverses the swap before ifft2.

3) When is frequency-domain filtering computationally preferable?

Spatial convolution costs O(MN·K²) for a K×K kernel, or O(MN·2K) if the kernel is separable.
FFT filtering costs O(MN·log(MN)), which does not depend on the kernel size. For a 384×512 image, log₂(MN) ≈ 17.6.

The frequency domain is better:

for large kernels (roughly K > 10–15), where K² outgrows the FFT overhead,
for filters that are defined in frequency: ideal, Butterworth, or notch filters (for example, removing periodic noise),
when you apply many filters to the same image, since the FFT is computed once and reused,
for deconvolution and inverse filtering, where you divide by H.

The spatial domain is better for small or separable kernels (3×3, 7×7), for local or adaptive filters, and when you want boundary handling other than wrap-around. FFT filtering is circular, so you need zero padding (to 2M×2N) to avoid wrap-around artifacts.
