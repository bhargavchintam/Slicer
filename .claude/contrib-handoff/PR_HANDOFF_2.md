# Slicer PR handoff, batch 2: #9540, #9562, #6679

The branches are pushed to `bhargavchintam/Slicer` and based on the current `Slicer/Slicer@main` (4c1a0a9).
For each PR: open the link, paste the title and body, and click **Create pull request**.

## Competing-work check (2026-11-25, re-run right before handoff)

| Issue | Open/closed PRs referencing it | Overlap found | Result |
|---|---|---|---|
| #9540 | none by number. #9081 (open, drag-and-drop reordering) is cross-referenced from the issue | #9081 does **not** modify `reorderItems()` (its only model change is item flags). A reviewer on #9081 (xskere, Oct 8) diagnosed the same root cause and noted it "also happens on main" | Not a duplicate. The PR credits that diagnosis. Three-way merge with #9081: no conflicts |
| #9562 | none | none | Clear. Three-way merge with #9081: no conflicts |
| #6679 | none by number | #9264 (VTK 9.7.2) review notes VTK 9.7 array wrappers are unhashable, so the mixin's dict keys break, and asks for a *separate* PR restructuring the mixin. #9264 does not edit the mixin | Not a duplicate. This PR is that separate change |

Local equivalents of the upstream checks pass on all three branches: `pre-commit` (ruff, clang-format v23.1.1, pyupgrade
and the file hygiene hooks), `codespell` with `.codespellignore`, and the exact `Commit Message Check` regexes.
`CI (Build)` (superbuild plus ctest) has not run; it will run on the PRs.

---

## PR 3: #9540

**Open:** https://github.com/Slicer/Slicer/compare/main...bhargavchintam:Slicer:claude/fix-9540-segment-reorder-widgets?expand=1

**Title:** `BUG: Fix segment table buttons lost after reordering segments`

**Body:**

````markdown
Fixes #9540

## Root cause

The visibility, color and status cells of the segments table are index widgets
(`QAbstractItemView::setIndexWidget`). They are created on `rowsInserted`/`modelReset` and tracked
by the view through `QPersistentModelIndex`es of the proxy model (`qMRMLSortFilterSegmentsProxyModel`).

`qMRMLSegmentsModel::reorderItems()` moved rows with `takeRow()` + `insertRow()` between
`layoutAboutToBeChanged()`/`layoutChanged()`. `takeRow()` is a real row removal: Qt invalidates every
persistent index that points into the removed row, and that happens inside the model,
**independently of `blockSignals(true)`**. When the proxy processes `layoutChanged()`, it remaps its
saved source persistent indexes. The moved rows' indexes are already invalid, so the view drops their
index widgets. The cells are left blank and clicking them does nothing.

This also explains the shape of the bug in the report. Moving the top segment to the end makes the
loop re-insert every *other* row one position up. All of those rows lose their widgets; only the row
that was moved last keeps them. Hence "the cells above the moved segment". A reviewer on #9081 (thanks
@xskere) reached the same diagnosis and noted it reproduces on `main` without that PR.

## Fix

`reorderItems()` now:
1. Refreshes `IndexRole` (already stored on every item by `updateItemDataFromSegment()`) from the
   segmentation, with signals blocked as before. Rows whose segment is not in the segmentation are kept at
   the end in their current order. If the order is unchanged, the function returns without touching the
   model.
2. Moves the rows with `QStandardItemModel::sort()` using `sortRole = IndexRole`, then restores the
   previous sort role. `sort()` is Qt's own row-permutation path: it emits
   `layoutAboutToBeChanged`/`layoutChanged` with `VerticalSortHint` and calls
   `changePersistentIndexList()`. Persistent indexes therefore follow their rows, and index widgets and the
   selection stay attached to their segment. The proxy has no sort column of its own, so it follows the
   source order.

Alternatives I tried in the harness first:
- `takeItem()`/`setItem()` plus `changePersistentIndexList()`: in Qt 5.15 `takeItem()` also invalidates
  persistent indexes, so the proxy had nothing left to remap and every row broke.
- Re-creating all index widgets after a reorder: it works, but it hides the model bug, destroys and
  recreates every button on each reorder, and still loses the selection.

## Verification

I built a headless harness with the same structure as the table: `QStandardItemModel` →
`QSortFilterProxyModel` → `QTableView`, plus one index widget per row tagged with its segment ID. It runs
the verbatim upstream `reorderItems()` loop and the fixed version.

| | top segment moved to end (6 rows) | 500 random permutations of 20 rows, checking order, widget↔segment identity and selection |
|---|---|---|
| `main`, Qt 5.15.13 | 5/6 rows lose their button (all except the moved row) | 500/500 broken |
| this PR, Qt 5.15.13 | 0/6 | 0/500 |
| this PR, Qt 6.4.2 | 0/6 | 0/500 |

`SegmentationWidgetsTest1` is extended. It records each segment's index widgets, moves the first segment
to the end and back with `vtkSegmentation::SetSegmentIndex()`, and asserts that the displayed order changed
and every segment still has the same widgets. I couldn't build Slicer locally, so `CI (Build)` will be the
first full run of that test.

This change was prepared with AI assistance (Claude Code). I reviewed and verified it before
submitting.
````

---

## PR 4: #9562

**Open:** https://github.com/Slicer/Slicer/compare/main...bhargavchintam:Slicer:claude/enh-9562-scroll-to-selected-segment?expand=1

**Title:** `ENH: Scroll segments table to show the selected segment`

**Body:**

````markdown
Fixes #9562

## Change

`qMRMLSegmentsTableView::setSelectedSegmentIDs()` now calls `QTableView::scrollTo()` on the first
selected segment once the selection is applied. The default hint is `EnsureVisible`, so the table
only scrolls when the row is outside the viewport. Clicking a visible row never moves the table.

## Why this is the right place

Every selection change that does not come from a click in the table goes through this method:
- `qMRMLSegmentEditorWidget::updateWidgetFromMRML()` when the editor node's selected segment changes.
  This covers the next/previous segment shortcuts (`selectNextSegment`/`selectPreviousSegment`) and
  `segmentEditorNode.SetSelectedSegmentID()` from Python or extensions, as asked in the forum thread.
- `onRemoveSegment()`, `moveSelectedSegmentsUp/Down()`, restoring the selection after a model reset, and
  `qSlicerSegmentationsModuleWidget`.

The editor already scrolls this way for a newly added segment (`onAddSegment()` calls
`tableWidget()->scrollTo(index)`). This change applies the same behavior to every programmatic selection.
Invalid or filtered-out IDs produce no index, so nothing scrolls. An empty selection still just clears.

## Verification

- Qt harness (Qt 5.15.13 and Qt 6.4.2): 50-row proxy model, 200 px view. Selecting row 40
  programmatically leaves it off-screen on `main` and makes it visible with this change. Selecting an
  already-visible row leaves the scrollbar unchanged (36 → 36).
- New `SegmentationWidgetsTest1` section 5 (`TestSection_05_qMRMLSegmentsTableViewScrollToSelection`):
  40 segments in a 150 px table. It asserts the last segment is not visible, selects it with
  `setSelectedSegmentIDs()`, and asserts it is now inside the viewport. The test is placed as its own
  section so it doesn't collide with the #9540 or #9081 edits to the same file. I checked that both merge
  cleanly.

This change was prepared with AI assistance (Claude Code). I reviewed and verified it before
submitting.
````

---

## PR 5: #6679

**Open:** https://github.com/Slicer/Slicer/compare/main...bhargavchintam:Slicer:claude/fix-6679-observation-mixin-weak-refs?expand=1

**Title:** `BUG: Do not keep objects observed by VTKObservationMixin alive`

**Body:**

````markdown
Fixes #6679

## Background

- #6406: `VTKObservationMixin.__observations` uses observed objects as dict keys. That holds a reference,
  so observed objects are never deleted until every observation is removed explicitly, which leaks
  temporary objects.
- #6409 switched to `weakref.WeakKeyDictionary`. The key was then the *Python wrapper*, which can be garbage
  collected while the VTK object lives on (e.g. owned by a `vtkCollection` or the scene). The entry vanished
  with the wrapper and the observer could no longer be removed (#6610). #6612 reverted it and marked
  `test_releaseNodes` as expected to fail.

## Approach

This follows option 1 from the issue: weak references that respect VTK reference counting.

- **Key:** each observed VTK object is keyed by `obj.__this__`, the address of the C++ object. That string
  is the same for every Python wrapper of the object.
- **Reference:** only a `vtkWeakReference` to the object is kept (the Python-visible `vtkWeakPointer` the
  issue asks for). `Get()` returns a live wrapper of the most-derived type while the C++ object exists,
  and `None` after it is deleted.
- **Deleted objects:** `Observations`, `observer()` and `removeObservers()` skip and forget entries whose
  weak reference is null.
- **Address reuse:** if an entry is found under a key but its weak reference is null, the object that
  was there was deleted and a new one now sits at the same address. The stale entry is discarded, so it
  can't be mistaken for the old one.
- **Non-VTK objects:** objects without `__this__` are still referenced exactly as before.
- **Lookups:** a weak reference is only created when an observation is added. `hasObserver`/`getObserver`
  are often called in GUI update loops and allocate nothing.

The public API (`addObserver`, `removeObserver`, `removeObservers`, `getObserver`, `hasObserver`,
`observer`, `Observations`) and its semantics are unchanged. The only behavior change is that the mixin
no longer keeps observed objects alive. None of the 23 in-tree `addObserver` call sites rely on that:
none observes a temporary.

Related: in the #9264 (VTK 9.7) review, @ebrahimebrahim pointed out that VTK 9.7 array wrappers are no
longer hashable, so using observed objects as dict keys breaks, and suggested a separate PR restructuring the
mixin internals. With this change the key is the `__this__` string, so wrapper hashability no longer
matters.

## Verification

Ran the real `test_slicer_util_VTKObservationMixin.py` against the upstream and patched `util.py` with VTK
9.6.2. The four `test_module*` cases need Slicer's Qt module classes and were not run locally.

| | upstream `util.py` | this PR |
|---|---|---|
| `test_releaseNodes` (was `@expectedFailure`, decorator now removed) | FAIL: `DeleteEvent` callback never called | pass |
| new `test_observedObjectNotKeptAlive` | FAIL: object still alive after `del` | pass |
| `test_removeObserver_issue6610` (C++-owned object, no Python ref) | pass | pass |
| other 6 tests | pass | pass |

Extra checks on the patched code:
- an observation added through one wrapper is found and removed through a fresh wrapper of the same
  object;
- `observer()` returns a live object;
- `removeObservers()` after an observed object was deleted does not fail;
- non-VTK observable objects keep the old behavior.

This change was prepared with AI assistance (Claude Code). I reviewed and verified it before
submitting.
````

---

## Considered and not taken in this batch

- **#8603** (transformed ROI marked modified after load): likely mechanism found. A parent-transform
  `ModifiedEvent` calls `UpdateROIFromControlPoints()`, whose `SetSize` round trip rewrites the control
  points and bumps their MTime past the stored time. I couldn't verify it without building MRML Core (needs
  ITK, Teem and a newer VTK), so I didn't submit an untested change.
- **#7445** (multiprocessing under PythonSlicer): three separate platform failures. The Linux one is in
  PythonQt's stdin redirect (external), and none can be verified without a Slicer runtime.
- **#9537**: needs a maintainer decision on re-bundling PyGithub.
- **#9279**: the issue says a fix is being prepared on the #9266 branch, and #9478 overlaps.
