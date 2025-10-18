---
layout: project
title: CS180 Project 3 – Autostitching
subtitle: Feature Matching for Autostitching
last_updated: Oct 17, 2025
---

<section id="overview">
  <h2 id="overview">Overview</h2>
  <p>
    To be filled
  </p>
  <p class="muted">
    All figures live under <code>cs180/proj3/assets/partB/&lt;scene&gt;/</code>.
  </p>
</section>

<section id="harris">
  <h2 id="harris">1 — Harris Corner Detection (with ANMS)</h2>
  <p>
    I begin with the single-scale Harris interest point detector and then thin the dense responses using
    <em>Adaptive Non-Maximal Suppression</em> (ANMS). This yields a spatially diverse, high-quality
    set of candidate keypoints for descriptor extraction and matching.
  </p>

  <h3>1.1 Harris Corner Response</h3>
  <p>
    For a grayscale image <em>I</em>, I compute image gradients <em>I</em><sub>x</sub>, <em>I</em><sub>y</sub> (Sobel filters),
    then form the <em>second-moment matrix</em> in a Gaussian window of scale <em>σ</em>:
  </p>
  <p class="math">
    \[
    M(x,y)\;=\; G_{\sigma} * \begin{bmatrix}
      I_x^2 & I_x I_y \\
      I_x I_y & I_y^2
    \end{bmatrix}
    \]
  </p>
  <p>
    The Harris response is
  </p>
  <p class="math">
    \[ 
      R\;=\;\det(M)\; -\; k\, (\operatorname{tr} M)^2
    \]
  </p>
  <p>
    Corners have large positive <em>R</em> (both eigenvalues large). Edges yield one large and one small eigenvalue,
    so <em>R</em> is small. On flat regions, both eigenvalues and <em>R</em> are small.
  </p>

  <!-- Comparison: Original vs Raw Harris -->
  <div class="pair">
    <figure class="card">
      <img src="/cs180/proj3/assets/partB/hearst/orig_1.jpg" alt="Hearst 1" loading="lazy" />
      <figcaption>Original image. <span class="muted">(Hearst Mining Building)</span></figcaption>
    </figure>
    <figure class="card">
      <img src="/cs180/proj3/assets/partB//hearst/harris_overlay.png" alt="Raw Harris corners overlaid" loading="lazy" />
      <figcaption>Raw Harris corners overlaid (pre-ANMS). <span class="muted"></span></figcaption>
    </figure>
  </div>

  <h3>1.2 Adaptive Non-Maximal Suppression (ANMS)</h3>
  <p>
    Dense local-max suppression can cluster many points on certain regions. ANMS retains strong points that are
    also <em>well spread</em>. For each candidate <em>i</em> with response <em>R</em><sub>i</sub>, define its suppression radius:
  </p>
  <p class="math">
    \[
      r_i\;=\; \min_j\; \|x_i - x_j\| \;\text{s.t.}\; R_j > c\,R_i
    \] 
  </p>
  <p>
    Sort by <em>r</em><sub>i</sub> (descending) and keep the top <em>K</em>, yielding points that are both strong and well-spaced.
  </p>

  <p>
    Comparison between ANMS-selected corners (well spread) and simply taking the top 200 by raw Harris score
    (often clustered).
  </p>
  <div class="pair">
    <figure class="card">
      <img src="/cs180/proj3/assets/partB/hearst/topk250_overlay.png" alt="Top-250 by Harris score (clustered)" loading="lazy" />
      <figcaption>Top-250 by raw Harris score — clustered. <span class="muted"></span></figcaption>
    </figure>
    <figure class="card">
      <img src="/cs180/proj3/assets/partB/hearst/anms_k250_overlay.png" alt="ANMS-selected 250 corners (well spread)" loading="lazy" />
      <figcaption>ANMS (K = 250) — well spread across the image. <span class="muted"></span></figcaption>
    </figure>
  </div>
</section>

<section id="descriptors">
  <h2 id="descriptors">2 — Extracting Feature Descriptors</h2>
  <p>
    Once I have the finalized interest points, I extract a simple, lighting‑robust descriptor around each one.
    For every keypoint, I take a <strong>40×40</strong> window centered at the point, then <strong>downsample by 5</strong> to produce an <strong>8×8</strong> patch. I then perform <em>bias/gain
    normalization</em> on that patch so it has zero mean and unit variance, which reduces the effects of
    brightness and contrast changes.
  </p>

  <ul>
    <li><span class="kbd">Window</span>: 40×40 around each keypoint (skip points too close to the image boundary or pad).</li>
    <li><span class="kbd">Downsample</span>: scale factor 5 → 8×8 descriptor patch.</li>
    <li><span class="kbd">Normalize</span>: subtract mean, divide by standard deviation (with small ε to avoid divide‑by‑zero).</li>
  </ul>

  <figure class="card">
    <img src="/cs180/proj3/assets/partB/hearst/patch_norm_8x8.png" alt="Normalized 8x8 patch" loading="lazy" />
    <figcaption>Normalized 8×8 patch used as the descriptor. <span class="muted"></span></figcaption>
  </figure>
</section>

<section id="matching">
  <h2 id="matching">3 — Feature Matching</h2>
  <p>
    With 8×8 patches in hand, I match points between image pairs using a nearest-neighbor search. To weed out ambiguous correspondences, I apply a <em>ratio test</em>: a match is kept
    only if the best distance is sufficiently smaller than the second best.
  </p>
  <ul>
    <li><span class="kbd">Distance</span>: Euclidean on mean/variance-normalized 8×8 patches.</li>
    <li><span class="kbd">Ratio test</span>: keep if <code>d<sub>1</sub>/d<sub>2</sub> &lt; τ</code> (I use <code>τ = 0.7</code>).</li>
    <li><span class="kbd">Mutual check</span>: optional cross-check (A→B and B→A) to remove asymmetric pairings.</li>
  </ul>

  <figure class="card">
    <img src="/cs180/proj3/assets/partB/hearst/matches_ratio.png"
          alt="Matches after applying ratio test" loading="lazy" />
    <figcaption>Matches after applying ratio test. <span class="muted"></span></figcaption>
  </figure>
  

</section>

<section id="ransac">
  <h2 id="ransac">4 — RANSAC for Robust Homography</h2>
  <p>
    Matches inevitably include outliers. I estimate a homography with <strong>RANSAC</strong>, repeatedly sampling
    minimal 4-point sets, fitting <em>H</em>, and counting inliers under a geometric error threshold in the target image.
    The model with the most inliers is then re-fit on its inlier set to produce the final transform.
  </p>
  <ul>
    <li><span class="kbd">Sample size</span>: 4 correspondences per iteration (normalized DLT solve).</li>
    <li><span class="kbd">Inlier test</span>: reprojection error \( \|x' - \hat x'\| &lt; \varepsilon \) (e.g., 2–4 px).</li>
    <li><span class="kbd">Iterations</span>: run to a max or early-stop once the inlier count stabilizes.</li>
    <li><span class="kbd">Refit</span>: recompute <em>H</em> using all inliers; skip degenerate samples (near-collinear quads).</li>
  </ul>


  <figure class="card">
    <img src="/cs180/proj3/assets/partB/hearst/matches_inliers.png"
          alt="Inlier matches after RANSAC" loading="lazy" />
    <figcaption>Inlier set after RANSAC; outliers removed. <span class="muted"></span></figcaption>
  </figure>
  
</section>

<section id="results">
  <h2 id="results">5 — Final Mosaics (Manual vs. Automatic)</h2>
  <p>
    Side‑by‑side comparison of the Part A (manual correspondences) mosaics and the Part B (automatic) results
    for three scenes.
  </p>

  <!-- Hearst -->
  <h3>5.1 Hearst Mining Building</h3>
  <div class="pair">
    <figure class="card">
      <img src="/cs180/proj3/assets/partB/hearst/manual_pano.png" alt="Hearst manual panorama (Part A)" loading="lazy" />
      <figcaption>Part A — Manual correspondences.</figcaption>
    </figure>
    <figure class="card">
      <img src="/cs180/proj3/assets/partB/hearst/auto_pano.png" alt="Hearst automatic panorama (Part B)" loading="lazy" />
      <figcaption>Part B — Automatic.</figcaption>
    </figure>
  </div>

  <!-- Cal -->
  <h3>5.2 California Hall</h3>
  <div class="pair">
    <figure class="card">
      <img src="/cs180/proj3/assets/partB/cal/manual_pano.png" alt="Cal manual panorama (Part A)" loading="lazy" />
      <figcaption>Part A — Manual correspondences.</figcaption>
    </figure>
    <figure class="card">
      <img src="/cs180/proj3/assets/partB/cal/auto_pano.png" alt="Cal automatic panorama (Part B)" loading="lazy" />
      <figcaption>Part B — Automatic.</figcaption>
    </figure>
  </div>

  <!-- Doe -->
  <h3>5.3 Doe Library</h3>
  <div class="pair">
    <figure class="card">
      <img src="/cs180/proj3/assets/partB/doe/manual_pano.png" alt="Doe manual panorama (Part A)" loading="lazy" />
      <figcaption>Part A — Manual correspondences.</figcaption>
    </figure>
    <figure class="card">
      <img src="/cs180/proj3/assets/partB/doe/auto_pano.png" alt="Doe automatic panorama (Part B)" loading="lazy" />
      <figcaption>Part B — Automatic.</figcaption>
    </figure>
  </div>
</section>

<section id="reflection">
  <h2 id="reflection">6 — Reflection</h2>
  <p>
    Building the automatic pipeline clarified how each stage supports the next: Harris finds many plausible corners;
    ANMS keeps them well distributed; simple normalized 8×8 patches are surprisingly effective when paired with a
    ratio test; and RANSAC turns noisy correspondences into a reliable homography. The biggest tuning levers were the
    ANMS <em>K</em> (too small → holes; too large → many weak matches), the ratio threshold (trade‑off between recall and
    precision), and the inlier tolerance in RANSAC. Failure cases mostly came from parallax and large moving objects;
    multiband blending helped hide exposure seams but cannot fix geometry. Overall, the automatic results closely
    tracked my Part A mosaics while removing the manual correspondence step.
  </p>
</section>

<section id="links">
  <h2 id="links">7 — Links &amp; Assets</h2>
  <ul>
    <li><a href="/cs180/proj3/partB/assets/">Downloads</a> — Original and final images</li>
    <li><a href="/cs180/proj3/partA/">Go to Part A</a> — manual correspondences + warping/blending writeup.</li>
  </ul>
</section>


