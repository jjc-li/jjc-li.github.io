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
    \( R\;=\;\det(M)\; -\; k\, (\operatorname{tr} M)^2,\quad k\in[0.04,0.06]. \)
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
    \( r_i\;=\; \min_j\; \|x_i - x_j\| \;\text{s.t.}\; R_j > c\,R_i,\quad c\approx 1.1. \)
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
      <img src="/cs180/proj3/assets/partB/hearst/topk200_overlay.png" alt="Top-200 by Harris score (clustered)" loading="lazy" />
      <figcaption>Top-200 by raw Harris score — clustered. <span class="muted"></span></figcaption>
    </figure>
    <figure class="card">
      <img src="/cs180/proj3/assets/partB/hearst/anms_k200_overlay.png" alt="ANMS-selected 200 corners (well spread)" loading="lazy" />
      <figcaption>ANMS (K = 200) — well spread across the image. <span class="muted"></span></figcaption>
    </figure>
  </div>
</section>
