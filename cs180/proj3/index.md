---
layout: project
title: CS180 Project 3 – Image Warping & Mosaics (Part A)
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
    All figures live under <code>cs180/proj3/assets/partA/&lt;scene&gt;/</code>. See the picture checklist in each section.
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
    To align two images of the same scene, we need to compute a <b>homography</b>. 
    A homography is a 3×3 transformation matrix \(H\) that relates a point \(\mathbf{p} = (x,y,1)^\top\) in the first image 
    to a point \(\mathbf{p}' = (u,v,1)^\top\) in the second image:
  </p>

  $$
  \mathbf{p}' \sim H \mathbf{p}
  $$

  <p>
    Here:
  </p>

  - \(\mathbf{p} = (x,y,1)^\top\) is a point in the first image (in homogeneous coordinates).  
  - \(\mathbf{p}' = (u,v,1)^\top\) is the corresponding point in the second image.  
  - \(H\) is the homography matrix:
  
  $$
  H =
  \begin{bmatrix}
    h_{11} & h_{12} & h_{13} \\
    h_{21} & h_{22} & h_{23} \\
    h_{31} & h_{32} & 1
  \end{bmatrix}
  $$

  <p>
    Notice that \(H\) has 8 unknown parameters (we fix the bottom-right entry to 1, since it is just a scale factor).
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
    Rearranging, each correspondence \((x,y) \mapsto (u,v)\) contributes two linear equations:
  </p>

  $$
  \big[-x \;\; -y \;\; -1 \;\; 0 \;\; 0 \;\; 0 \;\; u x \;\; u y \big]\,
  \boldsymbol{\theta} = -u
  $$

  $$
  \big[0 \;\; 0 \;\; 0 \;\; -x \;\; -y \;\; -1 \;\; v x \;\; v y \big]\,
  \boldsymbol{\theta} = -v
  $$

  where  
  $$
  \boldsymbol{\theta} =
  \begin{bmatrix}
    h_{11} & h_{12} & h_{13} & h_{21} & h_{22} & h_{23} & h_{31} & h_{32}
  \end{bmatrix}^\top.
  $$

  <p>
    Stacking all equations gives a linear system \(A\theta = b\). With 4 correspondences this system is square, but unstable; 
    with more than 4 matches the system is overdetermined and we solve it with least squares:
  </p>

  $$
  \theta^\star = \arg\min_\theta \lVert A\theta - b \rVert_2
  $$

  <h3>Computing H</h3>
  <p>
    Once we solve for \(\theta\), we fill it back into the matrix \(H\). This recovered \(H\) is then used to warp one image into alignment with another.
  </p>

  <h3>Correspondences</h3>
  <p>
    To recover \(H\), we must provide pairs of matching points — called <b>correspondences</b>. 
    Typically these are corners, edges, or other distinctive features selected by hand. 
    For example, if we pick the corner of a window in image 1, we click the same corner of the same window in image 2. 
    With 8–20 such points, the homography is much more reliable.
  </p>

  <div class="grid">
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/corr_i1_i2.png" alt="Cal i1→i2 correspondences">
      <figcaption class="caption">Example correspondences between Cal images i1 and i2.</figcaption>
    </figure>
  </div>

  <h3>Rectification (Optional)</h3>
  <p>
    A useful way to verify a homography is to apply it to a <b>planar patch</b> in one image. 
    For example, if a building facade appears at an angle, we can warp it with the recovered \(H\) so it looks fronto-parallel. 
    This process is called <b>rectification</b>. While not required for Part A, it is a nice sanity check: 
    if the patch becomes a clean rectangle, your homography is working correctly.
  </p>

  <div class="grid">
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/rectify_example.png" alt="Rectification example">
      <figcaption class="caption">Rectification example: a slanted window corrected to a rectangle using the homography.</figcaption>
    </figure>
  </div>

  <h3>Deliverables</h3>
  <ul>
    <li>Implement <code>computeH(im1_pts, im2_pts)</code> that outputs the 3×3 matrix \(H\).</li>
    <li>Show correspondences visualized on the input image pairs.</li>
    <li>Show the linear system setup \(A\theta = b\) (one example row pair).</li>
    <li>Print recovered \(H\) matrices (rounded values).</li>
    <li>(Optional) Show a rectification example.</li>
  </ul>
</section>



<section id="warping">
  <h2 id="warping">3 — Warping & Canvas Assembly</h2>
  <p>
    I warp each source into the center frame using <em>inverse mapping</em>:
    for each destination pixel, sample the source at H<sup>−1</sup>(x). I implemented nearest-neighbor and bilinear;
    mosaics use bilinear for smoother results. For each warped image, I compute a tight bounding box by forward-warping
    the four corners, then paste the tight canvas onto a global panorama using the returned offset.
  </p>

  <!-- Show first few placed layers and their alpha visualizations -->
  <div class="grid">
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/warp_layer_00.png" alt="Center layer">
      <figcaption class="caption">Cal — center layer on panorama canvas</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/warp_alpha_00.png" alt="Alpha center">
      <figcaption class="caption">Cal — alpha (visualized) for center</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/warp_layer_01.png" alt="Neighbor layer 1">
      <figcaption class="caption">Cal — neighbor layer</figcaption>
    </figure>
    <figure class="fig">
      <img src="/cs180/proj3/assets/partA/cal/warp_alpha_01.png" alt="Alpha neighbor 1">
      <figcaption class="caption">Cal — alpha (visualized) for neighbor</figcaption>
    </figure>
  </div>

  <p class="muted">
    Pictures needed here (per scene):
    <code>warp_layer_00.png</code>, <code>warp_alpha_00.png</code>,
    <code>warp_layer_01.png</code>, <code>warp_alpha_01.png</code> (optionally 02 as well).
  </p>
</section>

<section id="blending">
  <h2 id="blending">4 — Blending into a Mosaic</h2>
  <p>
    I use a weighted average with <em>edge-aware feather</em>:
    alphas are proportional to distance from each layer’s boundary (via distance transform),
    then normalized per pixel so weights in overlaps sum to 1 while single-support regions stay weight=1.
    This removes hard seams and preserves exposure in non-overlap regions. (I also support a simple radial feather fallback.)
  </p>
  <figure class="fig">
    <img class="bigimg" src="/cs180/proj3/assets/partA/cal/pano.png" alt="Cal panorama">
    <figcaption class="caption">Cal — final panorama (3 images)</figcaption>
  </figure>

  <p class="muted">
    Picture needed here: <code>pano.png</code> (final mosaic).
  </p>
</section>

<section id="results">
  <h2 id="results">Additional Results (Hearst)</h2>
  <p>
    The Hearst scene demonstrates a longer chain (4 images). I include inputs, correspondence plots,
    a few warped layers with their alphas, and the final mosaic.
  </p>

  <!-- Hearst final -->
  <figure class="fig">
    <img class="bigimg" src="/cs180/proj3/assets/partA/hearst/pano.png" alt="Hearst panorama">
    <figcaption class="caption">Hearst — final panorama (4 images)</figcaption>
  </figure>
</section>

<section id="reflection">
  <h2 id="reflection">Reflection</h2>
  <p>
    The main challenges were selecting reliable, well-spread correspondences and handling exposure steps in sky regions.
    Edge-aware feathering plus per-pixel weight normalization minimized seams; when needed, an overlap-based gain match
    further reduced subtle banding. Future work: automatic feature matching, cylindrical/spherical reprojection, and multiband blending.
  </p>
</section>
