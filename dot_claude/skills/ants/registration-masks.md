# `antsRegistration` mask semantics (`-x [fixedMask, movingMask]`)

How the mask option works inside the optimizer — i.e. what the masks actually *do* to the cost function and the displacement field. This is different from mask I/O (dtypes, thresholding, interpolation), which is in [masks.md](masks.md).

The intuition most people have — "masks define the region where the metric cares, so a permissive moving mask and a restrictive fixed mask combine to give the union" — is **wrong**. Getting this right is the difference between a registration that pulls the brain surface where you want and one that leaves tissue overshooting by several voxels.

## The one-line rule

**Masks in ANTs are rejection filters on a sample set that is seeded in the virtual (fixed-image) domain. They can only *further restrict* the cost-function support, never extend it. The tight fixed mask always wins.**

Corollary: a generous moving mask paired with a tight fixed mask is functionally equivalent to the tight fixed mask alone. Drawing a nice piriform-inclusive moving mask accomplishes nothing if the fixed mask is already 0 there.

## Source-level trace (ITK v5 / ANTs ≥ 2.4)

`itkImageToImageMetricv4::TransformAndEvaluateFixedPoint` / `TransformAndEvaluateMovingPoint` implement rejection sampling with **two** sequential tests per candidate sample:

1. **Draw** a sample in the *virtual domain*. The virtual domain is normally the fixed image's grid at the current stage resolution. The sample set comes from whatever sampling strategy you set (`Regular,0.5`, `Random,0.25`, `None`, …). The fixed mask is *not* used to seed samples — samples are generated first, then tested.

2. **Fixed-mask test** —
   `ITKLatestRelease/Modules/Registration/Metricsv4/include/itkImageToImageMetricv4.hxx:316-326`:
   ```cpp
   this->LocalTransformPoint(virtualPoint, mappedFixedPoint);
   if (this->m_FixedImageMask) {
       pointIsValid = this->m_FixedImageMask->IsInsideInWorldSpace(mappedFixedPoint);
       if (!pointIsValid) return pointIsValid;    // reject, contribute nothing
   }
   ```

3. **Moving-mask test** — *same* virtual point, forward-transformed into moving space:
   `itkImageToImageMetricv4.hxx:364-377`:
   ```cpp
   localMappedMovingPoint = this->m_MovingTransform->TransformPoint(localVirtualPoint);
   if (this->m_MovingImageMask) {
       pointIsValid = this->m_MovingImageMask->IsInsideInWorldSpace(mappedMovingPoint);
       if (!pointIsValid) return pointIsValid;    // reject, contribute nothing
   }
   ```

Both tests must pass. A point whose fixed-mask value is 1 but whose transformed moving-space point lands in moving-mask = 0 is rejected. A point whose fixed-mask value is 0 never even gets to the moving-mask test.

Official ITK docs put it plainly:
> "Combining a mask with sampling is done using a rejection approach. First a sample is generated and then it is accepted or rejected if it is inside or outside the mask."

Brian Avants on the ANTs mailing list:
> "the mask doesnt do anything special except identify where one should or should not compute the similarity metric."
> (sourceforge.net/p/advants/discussion/840261/thread/27216e69/)

## The SyN downsampled-level wrinkle

At downsampled SyN levels (non-zero shrink/smoothing), the metric runs on a smaller grid. The moving mask isn't tested in original moving space there — it gets **transformed through the current moving transform, resampled (nearest-neighbor) into virtual space at the downsampled resolution, and re-set on the metric** at every iteration.
`itkSyNImageRegistrationMethod.hxx:398-424`.

Important implications:

- The moving mask is still only a **restrictor** in virtual space; it's ANDed with the fixed mask after resampling. Nothing about this allows it to extend support beyond the fixed mask.
- Nearest-neighbor downsampling means **thin masks can shrink or develop holes at coarse levels**. A 2-voxel-thick sliver will mostly disappear at 16× shrink. Keep mask features thicker than `16 · shrink_factor` voxels or accept aliasing.
- The moving mask does update every iteration to reflect the current deformation — so it moves with the subject — but this is cosmetic relative to the restriction story.

## The gradient-field smoothing wrinkle (this one actually helps you a little)

Outside the fixed mask, the metric's gradient contribution is zero. But the SyN update field is **not zeroed** there — it's Gaussian-smoothed every iteration, so deformation **bleeds across the mask boundary from inside**. Tustison, asked directly:
> "No, due to smoothing the update and/or total displacement fields."
> (sourceforge.net/p/advants/discussion/840260/thread/7ee77cb3/)

Practical meaning for your SyN parameters `SyN[gradStep, updateSigma, totalSigma]`:

- The **update field smoothing** (middle parameter) determines how far gradient information propagates out of the mask per iteration. `SyN[0.2, 24, 0]` with update-σ = 24 voxels propagates quite far.
- Voxels outside the fixed mask therefore receive a deformation inherited from their nearest inside-mask neighbors — *not* a deformation computed from local intensity mismatch. They're passive passengers.
- Dilating the fixed mask converts near-boundary voxels from "passive passengers" to "active gradient contributors", which is a qualitative improvement, not just a quantitative one. This is why the dilation recommendation is so strong.

## Canonical advice — paraphrased from Tustison/Avants

From the ANTs wiki (`Anatomy-of-an-antsRegistration-call`) and various GitHub/SourceForge threads:

- **Skull-strip both images**, then use **dilated brain masks** for SyN. Don't rely on tight brain masks on unstripped images — you lose edge contrast while still including background noise.
- **Dilate masks by 10–30 voxels** for deformable registration. Tustison uses this range consistently. At 25 µm mouse brain, that's 250–750 µm; at 100 µm template spacing, 1–3 mm.
- **No mask for rigid/affine stages**, restrictive mask only for SyN. Rigid/affine are global metrics and work fine on whole skull-stripped images.
- **Smaller masked-out regions are safer than tight inclusion masks.** Tustison: "Masking out smaller regions is less likely to cause problems than attempting to draw a mask around the entire structure of interest." Use a mask to exclude bad regions (artifacts, non-anatomy), not to define the anatomy of interest.
- **Per-stage `-x` is safer than one global `-x`** (see below).

## Issue #254: global-vs-per-stage mask ROI bug

Historical quirk: when `-x` is passed *once* for the whole registration (as a single global option before the first `-t`), the SyN stage uses as its ROI "the intersection of fixed-mask and **un-transformed** moving-mask in their initial spaces" — i.e. *before* the initial/affine transform has been applied. If the moving image starts far from the fixed image, this intersection can be empty or tiny.

Fix: always pass `-x` **per stage** (once before each `-t`), not globally. `antsRegistration` supports this explicitly (issue #186):
> Tustison: "Two options are allowed: 1) the user specifies a single mask to be used for all stages or 2) the user specifies a mask for each stage."

```
-x "[NULL,NULL]"    -t Rigid[...]  -m Mattes[...] ...
-x "[NULL,NULL]"    -t Affine[...] -m Mattes[...] ...
-x "[fix,mov]"      -t SyN[...]    -m Mattes[...] ...
```

The per-stage form avoids the issue #254 trap entirely and lets you scope the restriction to the stage where it matters.

## Quick flowchart: what to do when a SyN registration leaves overshoot

Symptom: after the SyN stage, the moving brain has tissue that sits *outside* the fixed brain's extent — the warp fails to pull the surface in, even though the deformation elsewhere looks good.

1. **Is the fixed mask tight at the pial surface?** If yes, dilate it by 10–20 voxels (soft minimum; more for bigger overshoot). This is almost always the right move.
2. **Is the moving mask doing anything for you?** If the moving image is already skull-stripped (background is clean black), consider dropping the moving mask entirely — `-x [dilatedFixed,NULL]`. Fewer rejection tests, simpler semantics.
3. **Is the issue orthogonal to masks — i.e. the metric is being pulled the wrong way by intensity mis-correspondence?** Separate problem, separate fix (usually increase the gradient-image metric weight or add a distance-map MSE term). See below.

## Orthogonal trap: cross-modal intensity metric non-monotonicity

Related problem that *looks* like a mask issue but isn't: when fixed and moving have a non-monotonic intensity relationship (e.g., MRI T1 → CCF autofluorescence, where cell-dense cortical layers are bright in MRI but *dark* in autofluorescence, while fiber tracts go the other way), the Mattes-MI cost surface has false minima that can pull anatomical features onto the wrong correspondence.

Fixes, in the same per-stage style:

- **Register on spatial-gradient images** instead of intensity. Gradient magnitude is indifferent to dark→bright vs bright→dark transitions, so it handles non-monotonic contrast gracefully. Weight it at ≥ 2 in the SyN metric mix.
- **Add a distance-map MSE term** between the two brain masks. This pins the brain surfaces regardless of interior intensity.
- **Reduce the intensity-MI weight** if it's currently dominant.

You can mix all three as separate `-m` entries in a single SyN stage; ANTs sums the weighted metric contributions.

## Key source-code citations

- `ITKLatestRelease/Modules/Registration/Metricsv4/include/itkImageToImageMetricv4.hxx:316-326` — fixed mask rejection test
- `ITKLatestRelease/Modules/Registration/Metricsv4/include/itkImageToImageMetricv4.hxx:364-377` — moving mask rejection test (in moving space via forward transform)
- `ITKLatestRelease/Modules/Registration/Metricsv4/include/itkCorrelationImageToImageMetricv4GetValueAndDerivativeThreader.hxx:186-227` — the two-tests-in-sequence pattern for the CC metric threader
- `ITKLatestRelease/Modules/Registration/RegistrationMethodsv4/include/itkSyNImageRegistrationMethod.hxx:340-343` — mask propagation at non-downsampled levels
- `ITKLatestRelease/Modules/Registration/RegistrationMethodsv4/include/itkSyNImageRegistrationMethod.hxx:379-424` — moving-mask transform+resample at downsampled levels
- `ANTs/Examples/antsRegistrationTemplateHeader.h:328-398` — `-x` parsing, per-stage storage
- `ANTs/Examples/itkantsRegistrationHelper.hxx:1161-1166` — `imageMetric->SetFixedImageMask(...)` / `SetMovingImageMask(...)` call sites

## Authoritative external references

- ANTs wiki: "Anatomy of an antsRegistration call" — canonical dilation advice
  https://github.com/ANTsX/ANTs/wiki/Anatomy-of-an-antsRegistration-call
- ANTs wiki: "Tips for improving registration results" — per-stage mask strategy
  https://github.com/ANTsX/ANTs/wiki/Tips-for-improving-registration-results
- ANTs issue #254 — global-mask ROI intersection quirk
  https://github.com/ANTsX/ANTs/issues/254
- ANTs issue #186 — per-stage masks supported, single or one-per-stage
  https://github.com/ANTsX/ANTs/issues/186
- ANTs issue #483 — dilated brain mask on skull-stripped images pattern
  https://github.com/ANTsX/ANTs/issues/483
- ITK docs: `ImageToImageMetricv4` sampling
  https://docs.itk.org/projects/doxygen/en/stable/classitk_1_1ImageToImageMetricv4.html
- ITK discourse: "sampling strategy in ImageRegistrationMethodv4"
  https://discourse.itk.org/t/sampling-strategy-in-imageregistrationmethodv4/1038
