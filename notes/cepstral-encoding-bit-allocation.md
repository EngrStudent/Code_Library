# Cepstrum-Informed Encoding: Bit-Allocation Note
**Date:** 2026-10-22
**Context:** Follow-up to Buckingham Pi / dimensionless-number-to-EM mapping work and the streak-transform / kepstrum rabbit trail.

## The core idea
Symbol smearing in a wireless channel is convolution: delayed copies of the transmitted signal stacked by the channel impulse response. The cepstrum is the un-smear operator. Take the log of the magnitude spectrum and the convolution becomes an addition in quefrency, which you can subtract instead of divide. Division by a near-zero spectral bin is numerically unstable; subtraction is not.

Existing technology already does pieces of this:
- **Cepstral vocoders** (MELP, CELP speech coders) split excitation from filter in quefrency and encode each with far fewer bits than the raw waveform.
- **OFDM** is the same move in a different basis: cyclic prefix makes the streak finite, FFT is the inverse streak, each subcarrier comes out clean.
- **A precoder** inverts the channel before transmission. The cepstral version inverts by subtraction in quefrency rather than division in frequency.

## The proposed split
Encode two cepstra separately:
1. **Message cepstrum** (high-quefrency / sparse component) — the information-bearing excitation.
2. **Channel cepstrum** (low-quefrency / smooth component) — the slowly varying filter the signal passed through.

At the receiver, recombine: `recon = IFFT(exp(FFT(cep_low + cep_high)))`. Because the two parts live in different quefrency bands, you can allocate bits independently and exploit the different entropy of each.

## Bit-allocation math (worked example)
Scenario: 20 MHz bandwidth, N = 1024 samples per block, 64-QAM baseline, target MSE ratio 1e-4 of signal power.

Uniform quantizer: `MSE = range^2 / (12 * 2^(2b))`, with `range ~ 6*sigma`. Solve for b:

`b = 0.5 * log2(range^2 / (12 * target_mse_ratio * power))`

Simulation result (naive liftering, real-part log-magnitude only, NO phase handling):
- Raw PCM: 16,384 bits/block (N x 16)
- Cepstral split: ~8,192 bits/block (8 bits low-q, 8 bits high-q)
- OFDM 64-QAM: 6,144 bits/block
- Cepstral saving vs raw: 50%
- Cepstral vs OFDM: -33% (OFDM still wins on this naive version)

**Honest failure modes from the sim:**
- Reconstruction correlation with observed signal: -0.07 (essentially uncorrelated)
- Pulse-location recovery: 1 of 8 true pulse indices recovered

Root cause: liftering only the real part of the log-magnitude spectrum discards phase. The inverse cannot rebuild the waveform without it. A production version needs either (a) separate phase encoding, (b) minimum-phase assumption with all-pass residual handled separately, or (c) complex cepstrum with unwrapping.

## Why this might still beat OFDM
OFDM pays for its clean subcarriers with a cyclic prefix overhead (typically 7-25% of every symbol) and rigid subcarrier orthogonality that breaks under Doppler and phase noise. A cepstral encoder has no prefix tax. If the message cepstrum is sparse (like a pulse train), its entropy can be far below 6 bits per sample even before quantization. The bet is: sparsity in quefrency + stable subtraction-based inversion > rigid FFT-bin orthogonality.

## Open questions / next steps
1. Complex cepstrum with phase unwrapping — does reconstruction correlation go positive?
2. Minimum-phase + all-pass split: encode the min-phase cepstrum, treat all-pass as a separate low-rate parameter.
3. Compare against OFDM under the SAME channel model (multipath + Doppler) rather than the clean synthetic above.
4. Quantify the quefrency sparsity of real 5G channel impulse responses — is the channel cepstrum actually low-entropy enough to justify separate encoding?
5. Connect back to the dimensionless-number frame: fractional bandwidth (Bw/f) sets how many quefrency bins the message occupies; delay spread sets the channel cepstrum's quefrency width. Both are dimensionless governors of the bit split.

## Related threads
- Buckingham Pi -> EM dimensionless groups (Fresnel, ka, fractional bandwidth, Eb/N0, impedance ratio, Q, Doppler, coupling strength) — Oct 20-21 sent mail, "Follow-up: Dimensionless groups organized by the eight frames."
- Streak transform / inverse streak / coherency / kepstrum rabbit trail — Oct 22 voice session.
- 5G-EM-PIML-Toy repo: https://github.com/EngrStudent/5G-EM-PIML-Toy
