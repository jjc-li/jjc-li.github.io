---
layout: project
title: CS180 Project 3 – Photo mosaics
subtitle: Homographies, inverse warps, and panorama blending
last_updated: Oct 7, 2025
---

<section id="overview">
  <h2 id="overview">Overview</h2>
  <p>
    In Part A, I implement a full panorama pipeline: (1) capture a short image sweep with overlap,
    (2) estimate homographies using hand-picked correspondences (normalized DLT),
    (3) warp every image into a common reference frame with inverse mapping, and
    (4) blend the stack into a mosaic using edge-aware feathering with per-pixel weight normalization.
    I show two scenes: <em>Cal</em> (3 images — good for a quick, readable walk-through)
    and <em>Hearst</em> (4 images — richer overlaps and a larger field of view).
  </p>
  <p class="muted">
    All figures live under <code>cs180/proj3/assets/partA/&lt;scene&gt;/</code>.
  </p>
</section>

<section id="shooting">
  <h2 id="shooting">1 — Shooting the Pictures</h2>
  <p>
    I shot each sequence by rotating in place, keeping ~30–50% overlap between neighbors, and avoiding strong parallax.
    To reduce exposure steps in the sky, I tried to keep exposure constant when possible. The <em>California Hall</em> scene uses
    3 images. The <em>Hearst Mining Building</em> scene uses 4 images to demonstrate longer chains.
  </p>

  <!-- Images to include -->
  <div class="grid3">
    <!-- Cal scene thumbnails (3 inputs) -->
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/orig_1.jpg" alt="Cal input 1">
      <figcaption class="caption">Cal — input 1</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/orig_2.jpg" alt="Cal input 2">
      <figcaption class="caption">Cal — input 2</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/orig_3.jpg" alt="Cal input 3">
      <figcaption class="caption">Cal — input 3</figcaption>
    </figure>
  </div>

  <div class="grid">
    <!-- Hearst thumbnails (4 inputs) -->
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/hearst/orig_1.jpg" alt="Hearst input 1">
      <figcaption class="caption">Hearst — input 1</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/hearst/orig_2.jpg" alt="Hearst input 2">
      <figcaption class="caption">Hearst — input 2</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/hearst/orig_3.jpg" alt="Hearst input 3">
      <figcaption class="caption">Hearst — input 3</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/hearst/orig_4.jpg" alt="Hearst input 4">
      <figcaption class="caption">Hearst — input 4</figcaption>
    </figure>
  </div>
</section>

<section id="homography">
  <h2 id="homography">2 — Recovering Homographies</h2>

  <p>
    To align two images of the same scene, we compute a <b>homography</b>. 
    A homography is a 3×3 transformation matrix \(H\) that maps a point \(\mathbf{p} = (x,y,1)^\top\) in the first image 
    to a point \(\mathbf{p}' = (u,v,1)^\top\) in the second image:
  </p>

  $$
  \mathbf{p}' \sim H \mathbf{p}
  $$

  <p>
    Here:
  </p>

  <ul>
    <li>\(\mathbf{p} = (x,y,1)^\top\) is a point in the first image (homogeneous coordinates).</li>
    <li>\(\mathbf{p}' = (u,v,1)^\top\) is the corresponding point in the second image.</li>
    <li>\(H\) is the homography matrix:</li>
  </ul>

  $$
  H =
  \begin{bmatrix}
    h_{11} & h_{12} & h_{13} \\
    h_{21} & h_{22} & h_{23} \\
    h_{31} & h_{32} & 1
  \end{bmatrix}
  $$

  <p>
    Notice that \(H\) has 8 unknown parameters (we fix the bottom-right entry to 1, since it is only a scale factor).
  </p>

  <h3>Building the System of Equations</h3>
  <p>
    Expanding the homography gives:
  </p>

  $$
  u = \frac{h_{11}x + h_{12}y + h_{13}}{h_{31}x + h_{32}y + 1}, \quad
  v = \frac{h_{21}x + h_{22}y + h_{23}}{h_{31}x + h_{32}y + 1}.
  $$

  <p>
    Rearranging, each correspondence \((x,y) \mapsto (u,v)\) yields two linear equations:
  </p>

  $$
  \big[-x \;\; -y \;\; -1 \;\; 0 \;\; 0 \;\; 0 \;\; ux \;\; uy \big]\,
  \boldsymbol{\theta} = -u
  $$

  $$
  \big[0 \;\; 0 \;\; 0 \;\; -x \;\; -y \;\; -1 \;\; vx \;\; vy \big]\,
  \boldsymbol{\theta} = -v
  $$

  <p>
    where
  </p>

  $$
  \boldsymbol{\theta} =
  \begin{bmatrix}
    h_{11} & h_{12} & h_{13} & h_{21} & h_{22} & h_{23} & h_{31} & h_{32}
  \end{bmatrix}^\top.
  $$

  <p>
    Stacking all such equations produces a system \(A\theta = b\). 
    With 4 correspondences this system is square but unstable; with more than 4, it is overdetermined. 
    In that case, we solve in the least-squares sense:
  </p>

  $$
  \theta^\star = \arg\min_\theta \lVert A\theta - b \rVert_2
  $$

  <h3>Computing H with SVD</h3>
  <p>
    To solve, we use the Singular Value Decomposition (SVD). 
    Factorize \(A = U\Sigma V^\top\). Then:
  </p>
  <ul>
    <li><b>Least-squares form</b> (\(A\theta=b\)): compute the pseudoinverse \(\theta^\star = V\Sigma^+ U^\top b\).</li>
    <li><b>DLT form</b> (\(Ah=0\)): the solution is the right singular vector of \(A\) corresponding to the smallest singular value. Reshape it into \(H\) and rescale so that \(H_{33}=1\).</li>
  </ul>
  <p>
    Once \(\theta\) (or \(h\)) is recovered, we fill it back into \(H\), which can then be used to warp one image into alignment with the other.
  </p>

  <h3>Correspondences</h3>
  <p>
    We recover \(H\) by supplying pairs of matching points — <b>correspondences</b>. 
    Typically these are corners, edges, or other distinctive features. 
    I used around 15 points distributed across the image, so the estimate becomes more robust.
  </p>

  <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/corr_i1_i2.png" alt="Cal i1→i2 correspondences">
      <figcaption class="caption">Example correspondences between Cal images i1 and i2.</figcaption>
  </figure>



<section id="warp">
  <h2 id="warp">3 — Warp the Images</h2>

  <p>
    With the homography \(H\) estimated, we re-express each source image in the <em>reference</em> image’s coordinate system.
    The safest way to do this is <b>inverse mapping</b>: instead of pushing source pixels forward, we pull values from the source
    for every pixel on the target canvas. This avoids holes and ensures every output pixel receives a value.
  </p>

  <h3>Function signatures (from scratch)</h3>
  <pre><code>imwarped_nn  = warpImageNearestNeighbor(im, H)
imwarped_bil = warpImageBilinear(im, H)</code></pre>
  <p>
    Both functions perform the same geometric step (inverse mapping); they differ only in how the non-integer source location
    \((x,y)\) is sampled (nearest vs. bilinear).
  </p>

  <h3>Inverse mapping, step by step</h3>
  <ol>
    <li><b>Predict output canvas:</b> map the 4 corners of the source image through \(H\) to find the output bounding box.
      Let the source corners be \((0,0),\,(W\!-\!1,0),\,(W\!-\!1,H\!-\!1),\,(0,H\!-\!1)\).
      For each corner \(\mathbf p=(x,y,1)^\top\), compute \(\mathbf p' = H\,\mathbf p\), then divide by the homogeneous scale:
      \( (u, v) = \big(\tfrac{p'_x}{p'_w}, \tfrac{p'_y}{p'_w}\big) \).
      Take \(\text{xmin}=\lfloor\min u\rfloor\), \(\text{xmax}=\lceil\max u\rceil\),
      \(\text{ymin}=\lfloor\min v\rfloor\), \(\text{ymax}=\lceil\max v\rceil\).
      The output size is \(W_{\text{out}}=\text{xmax}-\text{xmin}+1\), \(H_{\text{out}}=\text{ymax}-\text{ymin}+1\).</li>
    <li><b>Loop over output pixels:</b> for each output index \((U,V)\) on the canvas, convert to canvas coordinates
      \((u,v)=(U+\text{xmin},\,V+\text{ymin})\) and form \(\mathbf p'=(u,v,1)^\top\).</li>
    <li><b>Back-project with \(H^{-1}\):</b>
      $$
      \mathbf p \sim H^{-1}\mathbf p', \qquad
      (x,y)=\left(\frac{p_x}{p_w},\,\frac{p_y}{p_w}\right).
      $$
    </li>
    <li><b>Sample the source</b> at \((x,y)\) using one of the interpolation rules below. If \((x,y)\) lies outside
      \([0,W\!-\!1]\times[0,H\!-\!1]\), leave the output pixel empty (e.g., black).</li>
  </ol>

  <p class="muted">
    <b>Coordinate convention:</b> choose a consistent model for pixel locations. It’s often convenient to treat integer grid
    points as pixel <em>centers</em> (so \((0,0)\) is the center of the top-left pixel, and \((0.5,0)\) lies midway to the next column).
    Consistency matters more than the specific choice.
  </p>

  <h3>Nearest neighbor (fastest)</h3>
  <p>
    Round to the closest integer pixel:
  </p>
  $$
  i = \text{round}(x), \quad j = \text{round}(y), \quad I_{\text{out}}(U,V) \gets I_{\text{src}}(j,i).
  $$
  <p>
    Pros: simplest and fast; preserves exact source values. Cons: jagged edges (“staircasing”) and aliasing when geometry shifts by subpixel amounts.
  </p>

  <h3>Bilinear interpolation (smoother)</h3>
  <p>
    Let \(x_0=\lfloor x\rfloor,\ y_0=\lfloor y\rfloor,\ \alpha=x-x_0,\ \beta=y-y_0\).
    Read the four neighbors \(v_{00}=I(y_0,x_0)\), \(v_{10}=I(y_0,x_0+1)\), \(v_{01}=I(y_1,x_0)\), \(v_{11}=I(y_1,x_0+1)\) with \(x_1=x_0+1,\ y_1=y_0+1\).
    The bilinear value is
  </p>
  $$
  I_{\text{out}}(U,V)=
  (1-\alpha)(1-\beta)\,v_{00} + \alpha(1-\beta)\,v_{10} + (1-\alpha)\beta\,v_{01} + \alpha\beta\,v_{11}.
  $$
  <p>
    Apply per channel for color images. Pros: much smoother results; Cons: slightly slower and can soften fine detail.
  </p>

  <h3>Quality vs. speed</h3>
  <ul>
    <li><b>NN:</b> minimal compute, possible jaggies along oblique lines and text.</li>
    <li><b>Bilinear:</b> modest extra cost, cleaner edges and fewer aliasing artifacts; minor blur on high-frequency content.</li>
  </ul>

  <p>
    However, the difference between the two methods could be hard to see if the image is large.
  </p>

  <h3>Examples (warp toward reference)</h3>
  <div class="grid3">
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/warp_nn.png" alt="NN warp">
      <figcaption class="caption">Cal — inverse warp using Nearest Neighbor.</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/warp_bil.png" alt="Bilinear warp">
      <figcaption class="caption">Cal — inverse warp using Bilinear.</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/warp_dif.png" alt="Warp difference">
      <figcaption class="caption">Difference between two images.</figcaption>
    </figure>
  </div>

  <p>
    On high-contrast edge, the "Warp difference" image is brighter, indicating that the difference is larger. 
  </p>

  <h3>Rectification</h3>
  <p>
    "Rectification" on image is helpful to check whether the projection function is working correctly.

  <div class="pair">
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/rect/rect_src1.jpg" alt="Rect source 1">
      <figcaption class="caption">Rectification — source image (example 1).</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/rect/rect_out1.png" alt="Rectified 1">
      <figcaption class="caption">Rectified (example 1).</figcaption>
    </figure>
  </div>

  <div class="pair">
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/rect/rect_src2.jpg" alt="Rect source 2">
      <figcaption class="caption">Rectification — source image (example 2).</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/rect/rect_out2.png" alt="Rectified 2">
      <figcaption class="caption">Rectified (example 2).</figcaption>
    </figure>
  </div>
  
</section>




<section id="blending">
  <h2 id="blending">4 — Blending into a Mosaic</h2>
  <p>
    I warp every image into the <em>center</em> image’s coordinate frame using the composed homography \(H_{i\to c}\) and inverse mapping (bilinear sampling). Each warped result is pasted onto a common panorama canvas at its computed offset. Finally, I perform a weighted average across the stack of aligned layers to produce a single mosaic.
  </p>
  <ol>
    <li><strong>Register to center.</strong> For each image \(I_i\), compose neighbor homographies to obtain \(H_{i\to c}\). Inverse-warp \(I_i\) with \(H_{i\to c}^{-1}\) onto the canvas.</li>
    <li><strong>Stack on a canvas.</strong> Compute a tight bbox for each warp, then place every warped layer on the same global canvas using its offset.</li>
    <li><strong>Blend.</strong> Compute a per-pixel weighted average over all stacked layers to get the final mosaic.</li>
  </ol>
  <p class="muted">
    Notes: I use a simple feathered alpha (edge-aware distance-to-boundary) with a tunable seam width and a slight center-layer boost. Radial feathering is available as a fallback.
  </p>

  <figure class="fig">
    <img class="bigimg" src="/cs180/proj3/assets/partA/cal/pano.png" alt="Cal panorama">
    <figcaption class="caption">California Hall — final panorama (3 images)</figcaption>
  </figure>

  <figure class="fig">
    <img class="bigimg" src="/cs180/proj3/assets/partA/hearst/pano.png" alt="Hearst panorama">
    <figcaption class="caption">Hearst Mining Building — final panorama (4 images)</figcaption>
  </figure>

  <figure class="fig">
    <img class="bigimg" src="/cs180/proj3/assets/partA/doe/pano.png" alt="Doe panorama">
    <figcaption class="caption">Doe Library — final panorama (3 images)</figcaption>
  </figure>

</section>



<section id="reflection">
  <h2 id="reflection">Reflection</h2>
  <p>
    The main challenges were selecting reliable, well-spread correspondences and handling exposure steps in sky regions.
    Edge-aware feathering and per-pixel weight normalization can minimize the seams. When needed, an overlap-based gain match
    further reduced some subtle bandings.
  </p>
</section>

<section id="links">
  <h2 id="links">7 — Links &amp; Assets</h2>
  <ul>
    <li><a href="/cs180/proj3/assets/partA/">Downloads</a> — Original and final images</li>
    <li><a href="/cs180/proj3/partB/">Go to Part B</a> — Autostitching</li>
  </ul>
</section>