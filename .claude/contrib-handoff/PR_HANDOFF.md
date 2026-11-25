# Slicer PR handoff

The branches are pushed to `bhargavchintam/Slicer` and based on the current `Slicer/Slicer@main` (4c1a0a9).
For each PR: open the link, paste the title and body, and click **Create pull request** (not draft).

Competing-work check, re-run on 2026-11-25 just before handoff. I searched open and closed PRs for
`9558`, `GetClosestPointPositionAlongCurve`, `9351`, `RUNPATH` and `DCMTK rpath`. Nothing
overlaps: the only DCMTK/rpath hits are closed macOS packaging PRs (#9322, #9323, #9266).
Neither issue has comments, assignees or linked branches.

Local equivalents of the upstream checks pass on both branches:
- `CI (Lint)`: `pre-commit` on the changed files, including clang-format v23.1.1
- `codespell` with `.codespellignore`
- `Commit Message Check`: the exact regexes from `Slicer/slicer-check-commit-message-action@26db1d4`
  (prefix `^(ENH|PERF|BUG|STYLE|DOC|COMP): ([A-Z])+` and line length `^[^#].{1,78}$`)

The full `CI (Build)` job (Docker superbuild plus ctest) has not run. It will run on the PR.

---

## PR 1: #9558

**Open:** https://github.com/Slicer/Slicer/compare/main...bhargavchintam:Slicer:claude/fix-9558-curve-closest-point?expand=1

**Title:** `BUG: Fix wrong index returned by GetClosestPointPositionAlongCurveWorld`

**Body:**

````markdown
Fixes #9558

## Root cause

`vtkMRMLMarkupsCurveNode::GetClosestPointPositionAlongCurve()` has two ways to find the closest
curve point. One brute-force scans all points; the other uses `vtkPointLocator`.
`GetClosestPointPositionAlongCurveWorld()` always takes the locator path
(`TransformedCurvePolyLocator`). On that path the squared distance was computed against the
**output argument**, which has not been written yet:

```cpp
points->GetPoint(closestCurvePointIndex, closestCurvePoint);
closestDistance2 = vtkMath::Distance2BetweenPoints(pos, closestPos);          // before
closestDistance2 = vtkMath::Distance2BetweenPoints(pos, closestCurvePoint);   // after
```

`closestDistance2` is the threshold used next to decide whether a projection onto line segment
(k-1, k) or (k, k+1), with `t ∈ [0, 1]`, beats the closest vertex. Because it was taken from
the caller's buffer, the result depended on what the caller happened to put in that buffer. The
brute-force path was already correct, and the fix makes the locator path match it. The
locator's O(log n) lookup is unchanged.

## User-visible effects

There are two separate failure modes:

1. **Wrong index, correct position.** When the buffer is far from `pos` (a zero buffer and a
   curve away from the origin), the threshold is too large. For `pos` exactly on curve point k,
   the (k-1, k) projection has distance 0 and `t = 0`, so it wins `0 < wrong value`, and
   **k-1** is returned instead of k. This is the case in the issue. With the fix, `0 < 0` is
   false and k is returned.
2. **Wrong position.** When the buffer is *closer* to `pos` than the closest vertex, the
   threshold is too small. Projections that really are closer get rejected and the function
   returns the vertex. `vtkSlicerCurveWidget` (`ProcessControlPointInsert`) passes a zero-filled
   `worldPos` and inserts the new control point at the returned position. So for a curve near the
   world origin, clicking 1.0 mm from the curve could insert the point at a vertex about
   9.55 mm away. Example: polyline `(-10,1,0)–(10,1,0)–(30,1,0)`, `pos = (0.5,0,0)`:
   - before: index 1, position `(10,1,0)`, distance 9.55
   - after: index 0, position `(0.5,1,0)`, distance 1.00

## Tests

The new test `vtkMRMLMarkupsCurveNodeTest1` is registered in `Libs/MRML/Core/Testing/CMakeLists.txt`:
- `TestGetClosestPointPositionAlongCurveWorld`: a 4-control-point curve. For every inner
  curve point it checks that the index and position are the same with a zero-filled buffer and
  with a buffer prefilled with the input, and that they match the brute-force path.
- `TestGetClosestPointPositionAlongCurveWithLocator`: the position error case above.

Before submitting, I checked the test against both versions of the function. The function body was copied
verbatim from `main` and from this branch and built with the real `vtkAddonTestingMacros.h`
`CHECK_*` macros, against VTK 9.1:

| | inner points with wrong index (zero buffer) | position test |
|---|---|---|
| `main` | 29 / 29 | `CheckInt failed: current 1, expected 0` |
| this PR | 0 / 29 | pass |

I have not run the full Slicer build locally, so `CI (Build)` will be the first full run of the new
ctest.

## Notes

- There are two commits: the fix, and a follow-up adding the position test case. I'm happy to squash
  them.
- The bug dates from commit 334cccf19b (2019-08-02), so released versions (5.12.x) are affected.
  This might be a candidate for the stable branch.

I used Claude as my AI assistant for this change. I reviewed and verified it before
submitting.
````

---

## PR 2: #9351

**Open:** https://github.com/Slicer/Slicer/compare/main...bhargavchintam:Slicer:claude/fix-9351-dcmtk-runpath?expand=1

**Title:** `BUG: Remove build-tree RUNPATH from DCMTK libraries in Linux package`

**Body:**

````markdown
Fixes #9351

## Root cause

`External_DCMTK.cmake` sets `INSTALL_COMMAND ""`, so DCMTK is never installed. The packaging
blocks then copy ELF files straight out of `DCMTK-build`:

- `SlicerBlockInstallDCMTKLibs.cmake`: `install(DIRECTORY ${DCMTK_DIR}/lib/ ... PATTERN lib<name>.so*)`
- `SlicerBlockInstallDCMTKApps.cmake`: `install(PROGRAMS ${DCMTK_DIR}/bin/<app>)`

CMake only rewrites RPATH/RUNPATH for `install(TARGETS)`, using `CMAKE_INSTALL_RPATH`. It never
does so for `install(FILES|PROGRAMS|DIRECTORY)`. So the linker-generated
`DT_RUNPATH [/…/Slicer-SuperBuild/DCMTK-build/lib]` ships as is. `slicerStripInstalledLibrary()`
does not help: `strip` removes `.symtab`, not `.dynamic`, and it only runs for `Release`.

DCMTK is the outlier here. I checked the other Linux blocks that copy prebuilt ELF files:
OpenJPEG, PythonQt, QtTesting, LibArchive and OpenSSL all copy from their own `*-install` tree,
and their install step has already rewritten the RPATH. DCMTK copies from `*-build`.

## Fix

This adds `slicerRemoveInstalledLibraryRPath(FILES … COMPONENT …)` to
`CMake/SlicerFunctionInstallLibrary.cmake`. It follows the conventions of
`slicerStripInstalledLibrary()`: the same argument parsing and an `install(CODE)` rule that respects
`$ENV{DESTDIR}` and `CMAKE_INSTALL_PREFIX`. It runs `file(RPATH_REMOVE FILE …)` (available in the
3.28 minimum). It is a no-op on Apple and Windows; macOS run-paths are handled by
`SlicerBundleCloseDeps.py`. Unlike stripping, it runs for every configuration. The DCMTK libs and apps
blocks now call it.

Why runtime lookup does not change: `ld.so` checks `LD_LIBRARY_PATH` before `DT_RUNPATH`, and
the launcher already puts `lib/Slicer-X.Y` on `LD_LIBRARY_PATH`. The removed entry pointed at a
directory that does not exist on user machines, so it never resolved anything.

## Verification

I ran this end-to-end through the real block files, not a copy. I built a mock DCMTK tree with the
same shape: `libofstd` and `libdcmnet` with `VERSION 20.3.7 / SOVERSION 20`, `dcmnet → ofstd`, and
an app `dcmdump → dcmnet`. I then installed it through the unmodified
`SlicerBlockInstallDCMTKLibs.cmake` and `SlicerBlockInstallDCMTKApps.cmake` with `cmake --install`.

| | `readelf -d` on `libdcmnet.so.20.3.7` and `bin/dcmdump` | build-tree path in `strings` |
|---|---|---|
| `main` | `RUNPATH [.../ext-build/lib]` | present |
| this PR | no RPATH/RUNPATH entry | none |

Additional checks:
- `file(RPATH_REMOVE)` on `lib<name>.so` follows the `.so → .so.20 → .so.20.3.7` chain and
  edits the real file.
- A CPack-style `DESTDIR=… cmake --install … --prefix /opt/Slicer` also gives 0 entries.
- `LD_LIBRARY_PATH=<pkg>/lib/Slicer-X.Y bin/dcmdump` still runs and resolves the transitive
  `dcmnet → ofstd` dependency.

I have not built a full Linux package locally; the packaging CI or the nightly factory would confirm it on real
DCMTK.

## Alternative considered

I considered rewriting the RUNPATH to `$ORIGIN` (and `$ORIGIN/../lib/Slicer-X.Y` for the apps) so the
libraries also load without the launcher. I didn't do that here because the rest of the Linux package
relies on the launcher environment. It would be a behavior change worth discussing on its own.

I used Claude as my AI assistant for this change. I reviewed and verified it before
submitting.
````

---

## Comment for #9280 (root cause is in VTK)

**Post at:** https://github.com/Slicer/Slicer/issues/9280

````markdown
This is a VTK 9.6 bug and is not specific to `vtkOrientedImageData`. It reproduces with
the stock `vtk==9.6.2` wheel and no Slicer:

```python
import vtk
vtk.vtkImageData().GetCellTypesArray()   # Segmentation fault
```

`vtkStructuredGrid`, `vtkRectilinearGrid` and `vtkStructuredPoints` crash the same way. The
deprecated `vtkCartesianGrid::GetCellTypesArray()` and its copy in `vtkStructuredGrid.cxx` call
`this->StructuredCellTypes->GetBackend()`. `StructuredCellTypes` is null until the grid has
dimensions; the replacement `GetCellTypes()` correctly returns `None`. Tab completion in the
console evaluates attribute getters, so it reaches the deprecated method. The two function bodies are identical, so the
`vtkStructuredGrid::GetCellTypesArray` frame in the Windows dump is probably MSVC identical-code
folding (`/OPT:ICF`) giving both functions one symbol.

VTK `master` has removed the deprecated method, so a fix would go into the Slicer VTK fork
branch (`slicer-v9.6.2-…`) as a one-line guard in each file:

```diff
-  if (!this->LegacyStructuredCellTypes)
+  if (!this->LegacyStructuredCellTypes && this->StructuredCellTypes)
```

I can open the PR against Slicer/VTK if that is the preferred route.
````

The patch file is `vtk-9.6-GetCellTypesArray-null-guard.patch` in this folder. I have not compiled it.

---

## After opening: CI steps from the `oss-issue-contribution` skill

1. If the checks show **"Approve and run workflows"** (first-time fork contributor), post once:
   > Could a maintainer please approve the workflow runs for this PR? Thank you.
2. If a check fails, compare it with the same job on `main`. Fix only failures this change causes, with
   new commits (no force-push). For a pre-existing failure, post the evidence in a comment.
3. When every required check is green, post once:
   > CI is green: Lint, codespell, Commit Message Check and CI (Build), including the new
   > `vtkMRMLMarkupsCurveNodeTest1`, all pass. Ready for review.
   (For #9351, leave out the test name.)
