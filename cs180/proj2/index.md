---
layout: project
title: CS180 Project 2 - Fun with Filters and Frequencies!
subtitle: Sharpening, hybrid images, and frequency-based blending
last_updated: Oct 3, 2025
---

<section id="overview">
  <h2 id="overview">Overview</h2>
  <p>
    This project explores how linear filtering and coarse-to-fine frequency analysis let us sculpt images. I implemented
    finite-difference edge detectors, derivative-of-Gaussian filters, unsharp masking, the hybrid-image pipeline, Gaussian/Laplacian
    stacks, and multiresolution blending. This page mirrors the assignment structure so each deliverable slots next to the required
    figures.
  </p>
  <p class="muted">
    All figures live in <code>cs180/proj2/assets/</code>. Thumbnails below link to the full-resolution versions for grading.
  </p>
</section>

<section id="implementation">
  <h2 id="implementation">Implementation Notes</h2>
  <p>
    I relied on NumPy for array math and SciPy/scikit-image for Gaussian kernels. Every filter is implemented as a convolution with
    either separable 1D kernels (for Gaussians) or small finite-difference masks. For stacks and blending I build matrices of images
    and masks at progressively lower resolutions, normalize each band, then upsample and sum to reconstruct. Frequency-domain plots
    come from <code>np.fft.fftshift</code> with magnitude log scaling so high-frequency structure is visible.
  </p>
  <p>
    The interactive alignment utility from Project&nbsp;1 reappears here to line up hybrid-image pairs before computing low/high
    frequency components. All results are stored as float32 in [0, 1] during processing and converted to uint8 only when exporting
    JPEG/PNG assets for the report.
  </p>

  <h3 id="core-code">Core filtering snippet</h3>
  <pre><code class="language-python">
  <!-- code to be inserted -->
  </code></pre>
</section>

<section id="part1">
  <h2 id="part1">Part 1 — Fun with Filters</h2>

  <section id="part1-0">
    <h3 id="part1-0">1.0 Convolutions from Scratch</h3>
    <p>
      I started with a simple 5&times;5 smoothing kernel applied to one of my photos. Convolving with this mean filter removes small
      blemishes while keeping overall structure, confirming that my padding and convolution utilities behave as expected.
    </p>
    <div class="pair">
      <figure>
        <img class="fit" src="./assets/part1_1/jli_original.jpg" alt="Original portrait" />
        <figcaption>Original portrait.</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part1_1/jli_filtered.jpg" alt="Filtered portrait" />
        <figcaption>After 5&times;5 mean filtering (detail softened).</figcaption>
      </figure>
    </div>
  </section>

  <section id="part1-1">
    <h3 id="part1-1">1.1 Finite difference operator</h3>
    <p>
      I applied horizontal and vertical finite-difference kernels <code>[-1, 1]</code> and its transpose to the classic Cameraman image.
      Taking the absolute gradient magnitude reveals strong edges, but direct differencing amplifies sensor noise. Thresholding the
      magnitude at 0.18 produces a binary edge map aligned with textbook results.
    </p>
    <div class="grid">
      <article class="card"><figure>
        <img class="fit" src="./assets/part1_2/input/cameraman.png" alt="Cameraman" />
        <figcaption>Input.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part1_2/cam_dx.jpg" alt="dx" />
        <figcaption>Horizontal derivative.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part1_2/cam_dy.jpg" alt="dy" />
        <figcaption>Vertical derivative.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part1_2/cam_edge_noise.jpg" alt="Gradient magnitude" />
        <figcaption>Gradient magnitude (note amplified noise).</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part1_2/cam_edge_binarize.jpg" alt="Thresholded edges" />
        <figcaption>Binary edge map (threshold&nbsp;&approx;&nbsp;0.18).</figcaption>
      </figure></article>
    </div>
  </section>

  <section id="part1-2">
    <h3 id="part1-2">1.2 Derivative of Gaussian (DoG)</h3>
    <p>
      Blurring before differentiating damps high-frequency noise. Instead of a two-pass blur-then-differentiate, I convolve the
      finite-difference filters with a Gaussian (&#963; = 1.0) to produce derivative-of-Gaussian kernels. Using DoG drastically cleans
      up the gradient magnitude while preserving edge localization, so the binarized edge map is much smoother than the direct
      difference result.
    </p>
    <div class="grid">
      <article class="card"><figure>
        <img class="fit" src="./assets/part1_3/dxog.png" alt="DoG dx" />
        <figcaption>d/dx of Gaussian kernel.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part1_3/dyog.png" alt="DoG dy" />
        <figcaption>d/dy of Gaussian kernel.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part1_3/cam_dog_binarize.jpg" alt="DoG edges" />
        <figcaption>Edges after DoG + thresholding. Noise is greatly reduced.</figcaption>
      </figure></article>
    </div>
  </section>
</section>

<section id="part2">
  <h2 id="part2">Part 2 — Fun with Frequencies</h2>

  <section id="part2-1">
    <h3 id="part2-1">2.1 Image sharpening</h3>
    <p>
      I implemented unsharp masking by subtracting a Gaussian blur (&#963; = 2) scaled by an amount <code>&alpha;</code> from the original image.
      The tree example uses <code>&alpha; = 1.0</code>, while the Taj Mahal and the Berkeley hills panorama explore stronger settings of 3 and 6.
      The blurred intermediate and isolated high frequencies help visualize what gets amplified.
    </p>
    <div class="pair">
      <figure>
        <img class="fit" src="./assets/part2_1_sharpen/input/tree.jpeg" alt="Tree original" />
        <figcaption>Tree — original.</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_1_sharpen/tree_blurred.jpg" alt="Tree blurred" />
        <figcaption>Gaussian blur (&#963; = 2).</figcaption>
      </figure>
    </div>
    <div class="pair">
      <figure>
        <img class="fit" src="./assets/part2_1_sharpen/tree_sharp.jpg" alt="Tree sharpened" />
        <figcaption>Sharpened result (&#945;=1.0).</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_1_sharpen/view_high_freq.jpg" alt="High-frequency" />
        <figcaption>Extracted high-frequency layer for the hills panorama.</figcaption>
      </figure>
    </div>
    <div class="grid">
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_1_sharpen/input/view.jpg" alt="View original" />
        <figcaption>Panorama — original.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_1_sharpen/view_blurred.jpg" alt="View blurred" />
        <figcaption>Blurred baseline.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_1_sharpen/view_sharp_1.jpg" alt="View sharp 1" />
        <figcaption>Sharpened with &#945;=1.0.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_1_sharpen/view_sharp_3.jpg" alt="View sharp 3" />
        <figcaption>Sharpened with &#945;=3.0 (crisper but slightly haloed).</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_1_sharpen/input/taj.jpg" alt="Taj original" />
        <figcaption>Taj Mahal — original.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_1_sharpen/taj_sharp_3.jpg" alt="Taj sharp 3" />
        <figcaption>&#945;=3.0 (balanced).</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_1_sharpen/taj_sharp_6.jpg" alt="Taj sharp 6" />
        <figcaption>&#945;=6.0 (aggressive, halos visible).</figcaption>
      </figure></article>
    </div>
  </section>

  <section id="part2-2">
    <h3 id="part2-2">2.2 Hybrid images</h3>
    <p>
      Hybrid images combine low frequencies from one subject with high frequencies from another. My pipeline aligns each pair,
      applies Gaussian low-pass/high-pass filters (cutoffs around 9–11 pixels), and recombines them. Viewing up close reveals the
      high-frequency subject; stepping back reveals the low-frequency subject. I include frequency-magnitude plots for one pair to
      show how information occupies complementary bands.
    </p>

    <h4 id="hybrid-derek">Derek &amp; Nutmeg</h4>
    <div class="pair">
      <figure>
        <img class="fit" src="./assets/part2_2_hybrid/derek_nutmeg/DerekPicture_aligned.png" alt="Derek low" />
        <figcaption>Derek (low frequencies).</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_2_hybrid/derek_nutmeg/nut_aligned.jpg" alt="Nutmeg high" />
        <figcaption>Nutmeg (high frequencies).</figcaption>
      </figure>
    </div>
    <div class="pair">
      <figure>
        <img class="fit" src="./assets/part2_2_hybrid/derek_nutmeg/hybrid.jpg" alt="Hybrid Derek Nutmeg" />
        <figcaption>Hybrid result — Nutmeg up close, Derek from afar.</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_2_hybrid/derek_nutmeg/derek_aligned.jpg" alt="Alignment debug" />
        <figcaption>Alignment overlay check (channels stacked).</figcaption>
      </figure>
    </div>

    <h4 id="hybrid-binotto">Frederic Vasseur &amp; Binotto the Clown</h4>
    <p>
      This playful pair uses a low-pass portrait of Ferrari team principal Frédéric Vasseur and the high-pass features of a clown.
      The combination exaggerates the smile and hat while keeping the lighting from the portrait.
    </p>
    <div class="grid">
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_2_hybrid/binotto_clown/binotto_aligned.png" alt="Binotto low" />
        <figcaption>Low-pass portrait.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_2_hybrid/binotto_clown/clown_high.png" alt="Clown high" />
        <figcaption>High-pass clown details.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_2_hybrid/binotto_clown/binotto_clown.png" alt="Hybrid result" />
        <figcaption>Hybrid — the clown grin mapped onto Binotto.</figcaption>
      </figure></article>
    </div>

    <h4 id="hybrid-sauber">Sauber C44 &amp; Tractor</h4>
    <p>
      Motorsports meets farming equipment: the low frequencies come from a Sauber C44 Formula&nbsp;1 car, the high frequencies from a
      tractor. The FFT visualizations confirm the low-pass filter concentrates energy near the origin while the high-pass spectrum
      surrounds it with a hollow annulus.
    </p>
    <div class="grid">
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_2_hybrid/sauber_tractor/sauber_aligned.png" alt="Sauber low" />
        <figcaption>Sauber (low frequency).</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_2_hybrid/sauber_tractor/tractor_high.png" alt="Tractor high" />
        <figcaption>Tractor (high frequency).</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_2_hybrid/sauber_tractor/sauber_tractor.png" alt="Hybrid car tractor" />
        <figcaption>Hybrid — from afar it reads as the car.</figcaption>
      </figure></article>
    </div>
    <div class="grid">
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_2_hybrid/sauber_tractor/sauber_fft.png" alt="Sauber FFT" />
        <figcaption>Low-pass FFT (energy near DC).</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_2_hybrid/sauber_tractor/tractor_fft.png" alt="Tractor FFT" />
        <figcaption>Source FFT with broader support.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_2_hybrid/sauber_tractor/sauber_tractor_fft.png" alt="Hybrid FFT" />
        <figcaption>Hybrid FFT, combining both bands.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_2_hybrid/sauber_tractor/tractor_aligned_fft.png" alt="High-pass FFT" />
        <figcaption>High-pass FFT (hollow annulus).</figcaption>
      </figure></article>
    </div>
  </section>

  <section id="part2-3">
    <h3 id="part2-3">2.3 Gaussian and Laplacian stacks</h3>
    <p>
      I constructed nine-level Gaussian and Laplacian stacks for the classic apple-or-orange example. Each subsequent Gaussian level
      downsamples by a factor of two, while the Laplacian levels capture band-pass information (Gaussian level minus the next level
      upsampled). The matched stack stores the mask at each scale so blending can be performed consistently.
    </p>
    <div class="pair">
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/apple_orange.jpg" alt="Hybrid fruit" />
        <figcaption>Blend target: apple on the left, orange on the right.</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/input/apple.jpeg" alt="Apple input" />
        <figcaption>Source A (apple).</figcaption>
      </figure>
    </div>
    <div class="pair">
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/input/orange.jpeg" alt="Orange input" />
        <figcaption>Source B (orange).</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/matched_level0.jpg" alt="Mask level 0" />
        <figcaption>Mask level&nbsp;0 (full resolution).</figcaption>
      </figure>
    </div>
    <p class="muted">Representative stack levels (top row: Gaussian, middle row: Laplacian, bottom row: mask).</p>
    <div class="stack-grid">
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/apple_level0.jpg" alt="Apple level 0" />
        <figcaption>Gaussian L0</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/apple_level3.jpg" alt="Apple level 3" />
        <figcaption>Gaussian L3</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/apple_level6.jpg" alt="Apple level 6" />
        <figcaption>Gaussian L6</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/blended_level0.jpg" alt="Laplacian level 0" />
        <figcaption>Laplacian L0</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/blended_level3.jpg" alt="Laplacian level 3" />
        <figcaption>Laplacian L3</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/blended_level6.jpg" alt="Laplacian level 6" />
        <figcaption>Laplacian L6</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/matched_level3.jpg" alt="Mask level 3" />
        <figcaption>Mask L3</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/matched_level6.jpg" alt="Mask level 6" />
        <figcaption>Mask L6</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/matched_level8.jpg" alt="Mask level 8" />
        <figcaption>Mask L8</figcaption>
      </figure>
    </div>
  </section>

  <section id="part2-4">
    <h3 id="part2-4">2.4 Multiresolution blending</h3>
    <p>
      With stacks in place, blending becomes a per-level interpolation followed by reconstruction. I produced two blends: Apple-or-
      Orange (the classic example) and two custom composites — Foothill sunset stitched across a city skyline and a hand holding the
      Earth. The Laplacian and mask stacks confirm that only band-limited seams appear in the final image.
    </p>

    <h4 id="blend-apple">Apple &amp; Orange</h4>
    <div class="pair">
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/input/apple.jpeg" alt="Apple" />
        <figcaption>Source A.</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/input/orange.jpeg" alt="Orange" />
        <figcaption>Source B.</figcaption>
      </figure>
    </div>
    <div class="pair">
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/matched_level0.jpg" alt="Mask" />
        <figcaption>Feathered vertical mask.</figcaption>
      </figure>
      <figure>
        <img class="fit" src="./assets/part2_3_stacks/fruit/apple_orange.jpg" alt="Final blend" />
        <figcaption>Final blend — smooth transition across the seam.</figcaption>
      </figure>
    </div>

    <h4 id="blend-foothill">Foothill Skyline</h4>
    <p>
      I blended two Berkeley skyline shots captured minutes apart. The stack visualizations show how the Laplacian bands capture
      cloud texture while the mask concentrates along the seam to avoid ghosting.
    </p>
    <div class="grid">
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/foothill/left0.jpg" alt="Foothill left" />
        <figcaption>Left image (golden hour).</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/foothill/right0.jpg" alt="Foothill right" />
        <figcaption>Right image (blue hour).</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/foothill/combined.jpg" alt="Foothill blend" />
        <figcaption>Final blend.</figcaption>
      </figure></article>
    </div>
    <div class="grid">
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/foothill/mathed_level2.jpg" alt="Mask level 2" />
        <figcaption>Mask level&nbsp;2 (feathered seam).</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/foothill/left3.jpg" alt="Left Laplacian" />
        <figcaption>Left Laplacian level&nbsp;3.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/foothill/right3.jpg" alt="Right Laplacian" />
        <figcaption>Right Laplacian level&nbsp;3.</figcaption>
      </figure></article>
    </div>

    <h4 id="blend-hand">Hand &amp; Earth</h4>
    <p>
      For a more dramatic composite, I blended a rendered Earth with a studio-lit hand using a radial mask. Higher Laplacian levels
      capture the planetary highlights without introducing seams along the fingers.
    </p>
    <div class="grid">
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/hand_earth/hand_level0.jpg" alt="Hand" />
        <figcaption>Hand — level 0.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/hand_earth/earth_level0.jpg" alt="Earth" />
        <figcaption>Earth — level 0.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/hand_earth/hand_earth.jpg" alt="Hand holding Earth" />
        <figcaption>Final blend.</figcaption>
      </figure></article>
    </div>
    <div class="grid">
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/hand_earth/matched_level2.jpg" alt="Mask level 2" />
        <figcaption>Mask level&nbsp;2 (radial feather).</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/hand_earth/hand_level3.jpg" alt="Hand Laplacian" />
        <figcaption>Hand Laplacian level&nbsp;3.</figcaption>
      </figure></article>
      <article class="card"><figure>
        <img class="fit" src="./assets/part2_4_blend/hand_earth/earth_level3.jpg" alt="Earth Laplacian" />
        <figcaption>Earth Laplacian level&nbsp;3.</figcaption>
      </figure></article>
    </div>
  </section>
</section>

<section id="bells">
  <h2 id="bells">Bells &amp; Whistles</h2>
  <p>
    I plan to explore color-space sharpening (performing unsharp masking in LAB instead of RGB) and potentially Poisson blending for
    more natural composites. Results will be added here if time permits.
  </p>
</section>

<section id="reflection">
  <h2 id="reflection">Discussion &amp; Reflection</h2>
  <p>
    Working in the frequency domain makes it clear why simple convolution tricks create powerful perceptual effects. High-frequency
    halos appear quickly if the sharpening amount is too large, and hybrid images depend critically on spatial alignment before
    filtering. Multiresolution blending was the most satisfying deliverable: once the stacks are built, swapping in new sources and
    masks becomes a one-line change.
  </p>
  <p>
    One challenge was selecting consistent thresholds and mask feather widths so binary decisions do not introduce seams. Future
    improvements include adaptive thresholding for edge detection and automated alignment (e.g., phase correlation) to streamline
    hybrid image creation.
  </p>
</section>

<section id="assets">
  <h2 id="assets">Assets &amp; Downloads</h2>
  <ul>
    <li><a href="./assets/">Full-resolution figures and intermediate levels</a></li>
    <li>Source notebooks and scripts (to be linked after cleanup)</li>
  </ul>
</section>
