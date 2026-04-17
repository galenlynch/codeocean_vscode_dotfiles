---
name: ants
description: ANTs (Advanced Normalization Tools) registration and transform application patterns and gotchas. Use when working with antsRegistration, antsApplyTransforms, warp fields, SyN output (reg_0GenericAffine.mat, reg_1Warp.nii.gz, reg_1InverseWarp.nii.gz), or when composing multiple transforms for point/image warping.
---

# ANTs

ANTs has many subtle conventions that are easy to get wrong and cost debugging sessions. This skill captures the ones that bite most often.

## Rules to always remember

1. **Point direction is the opposite of image direction.** ANTs transforms act on **points** in the `fixed → moving` direction. This is used to resample the *moving* image into *fixed* space (iterate over fixed voxels, for each one ask "what moving point does this correspond to?").

2. **`antsApplyTransforms` applies transforms in command-line order: leftmost = applied first.** This contradicts what the `--help` text seems to say ("last transform pushed onto the stack"). **Ignore the help text** — the actual behavior is left-to-right. Empirically verified.

3. **Warp fields are stored on the fixed image grid.** The forward warp displaces from fixed space toward "pre-affine moving space"; the affine completes the mapping. Outside the stored extent, the displacement silently extrapolates to zero — which can make a chain degrade to affine-only without warning.

4. **Forward chain**: `-t reg_1Warp.nii.gz -t reg_0GenericAffine.mat` (warp applied first, then affine).
   **Inverse chain**: `-t [reg_0GenericAffine.mat,1] -t reg_1InverseWarp.nii.gz` (inverse affine first, then inverse warp).

5. **Use `GenericLabel`, not `NearestNeighbor`, for segmentations.** Cleaner boundaries, avoids NN aliasing at oblique angles.

6. **Mouse MRI template building convention**: `fixed = template`, `moving = subject`. So the forward point transform is template → subject, the inverse is subject → template.

7. **`-x [fixedMask, movingMask]` is a rejection filter, not a domain extender.** Samples are seeded in the virtual (fixed-image) domain; each one is then rejected if either the fixed-mask *or* the transformed-moving-mask is 0. Both masks can only *further restrict* — a generous moving mask cannot compensate for a tight fixed mask. For SyN, **skull-strip the inputs and dilate the masks by 10–30 voxels**, or drop masks entirely if the background is already clean. Pass `-x` per stage, not globally.

## When to consult reference files

- **Transforms, composition, chains, warp-field semantics** — see [transforms.md](transforms.md). Read this for the *why* behind the rules above, debugging tips (two-step resampling), and the forward/inverse chain table in detail.
- **Polar decomposition of `reg_0GenericAffine.mat`** — see [rigid-decomposition.md](rigid-decomposition.md). Extracting the rigid (rotation + translation) component for building template-aligned frames.
- **`-x` mask semantics inside the optimizer, dilation advice, SyN downsampling behavior, non-monotonic intensity traps** — see [registration-masks.md](registration-masks.md). Read this whenever a SyN registration leaves overshoot or undershoot at a brain boundary.
- **Mask dtype traps, `BinaryThreshold` int-type bug, interpolation choice** — see [masks.md](masks.md). I/O-level mask gotchas, distinct from the optimizer semantics in `registration-masks.md`.
- **`antsApplyTransforms` CLI options, `[path,1]` inversion, `-u` output type, `CreateJacobianDeterminantImage`, displacement magnitude, reference image role** — see [commands.md](commands.md).

## Sanity check before trusting a composed chain

When a resampled image looks *almost* right but is shifted or scaled slightly:

1. Decompose the single-call chain into two antsApplyTransforms calls (e.g. template → subject, then subject → rotated).
2. Compare against the single-chain output.
3. If they differ, the single chain is wrong — usually an order or direction mistake. See [transforms.md](transforms.md) for the full debugging walkthrough.
