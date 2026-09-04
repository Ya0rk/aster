---
date: 2026-09-04
mode: diff
base: eee6708ae
head: 7feb803ea
branch: HEAD
---

# Summary

The change appears to improve user-visible memory-management syscall handling by recognizing unmapped ranges for `msync`, and it documents the `Vmar::protect` partial-effect behavior in the API comment.

Remaining issues:

- **Major:** The Linux compatibility documentation for `msync` was not updated to reflect that ranges containing unmapped pages now return `ENOMEM`.
- **Major:** The Linux compatibility documentation and `mprotect.scml` were not updated for the changed `mprotect` behavior, including `ENOMEM` on unmapped holes and the observable partial permission changes before the first hole.
- **Minor:** The `Vmar::protect` API comment should cite the rationale or source for the surprising partial side effect on error, such as Linux `mprotect` semantics or an explicit local design note.

Cross-cutting recommendation: keep syscall implementation changes, API comments, and Linux compatibility artifacts updated together, especially when behavior is user-visible or includes non-atomic side effects.

## Maintainability

### `kernel/src/vm/vmar/vmar_impls/protect.rs` line 13

> ```diff
> @@
> +    /// If the range contains unmapped pages, an [`ENOMEM`] error will be returned.
> +    /// Note that pages before the unmapped hole are still protected.
> ```

`cite-sources` (minor): The new `Vmar::protect` contract documents a non-obvious syscall-facing behavior: `protect` can return `Errno::ENOMEM` after already changing pages before the first unmapped hole. That behavior is surprising for a `Result<()>` API and appears to be defined by external `mprotect`/Linux semantics, but the doc comment does not cite the source or rationale. Future readers have to rediscover whether the partial side effect is intentional or an implementation accident.

**Fix.** Shared with the other `mprotect-unmapped-range-docs` comment: document the new `mprotect`/`Vmar::protect` behavior consistently in both the API comment and the Linux compatibility artifacts. In `kernel/src/vm/vmar/vmar_impls/protect.rs`, cite the source or rationale for returning `ENOMEM` after partially protecting pages before the first unmapped hole, e.g. Linux `mprotect` semantics or an explicit local design note. Also update `book/src/kernel/linux-compatibility/syscall-flag-coverage/memory-management/README.md` under `### mprotect` and `book/src/kernel/linux-compatibility/syscall-flag-coverage/memory-management/mprotect.scml` to state that ranges with unmapped holes return `ENOMEM` and that permission changes before the hole may already be visible.

## Documentation

### `kernel/src/syscall/msync.rs` line 38

> ```diff
> @@ -35,6 +35,12 @@ pub fn sys_msync(addr: Vaddr, len: usize, flag: i32, ctx: &Context) -> Result<Sy
> +    if !query_guard.is_fully_mapped() {
> +        return_errno_with_message!(
> +            Errno::ENOMEM,
> +            "the range contains pages that are not mapped"
> +        );
> +    }
> ```

`linux-compat-docs` (major): This change enhances the user-visible `msync` syscall behavior by recognizing ranges that contain unmapped pages and returning `ENOMEM`, but the Linux Compatibility docs and `msync.scml` are unchanged. The documented syscall coverage can therefore understate or misrepresent the implemented `msync` behavior.

**Fix.** Update `book/src/kernel/linux-compatibility/syscall-flag-coverage/memory-management/README.md` under `### msync` and `book/src/kernel/linux-compatibility/syscall-flag-coverage/memory-management/msync.scml` to document the newly supported `msync` behavior for unmapped pages, including that a range containing unmapped pages returns `ENOMEM`.

### `kernel/src/vm/vmar/vmar_impls/protect.rs` line 13

> ```diff
> @@ -9,6 +9,11 @@ impl Vmar {
> -    /// Also, the range must be completely mapped.
> +    ///
> +    /// If the range contains unmapped pages, an [`ENOMEM`] error will be returned.
> +    /// Note that pages before the unmapped hole are still protected.
> ```

`linux-compat-docs` (major): This change alters the user-visible `mprotect` syscall behavior for ranges containing unmapped pages, including the observable partial effect before the first hole. The matching Linux Compatibility docs and `mprotect.scml` were not updated, so the compatibility artifact no longer reflects the syscall behavior exposed to users.

**Fix.** Shared with the other `mprotect-unmapped-range-docs` comment: document the new `mprotect`/`Vmar::protect` behavior consistently in both the API comment and the Linux compatibility artifacts. In `kernel/src/vm/vmar/vmar_impls/protect.rs`, cite the source or rationale for returning `ENOMEM` after partially protecting pages before the first unmapped hole, e.g. Linux `mprotect` semantics or an explicit local design note. Also update `book/src/kernel/linux-compatibility/syscall-flag-coverage/memory-management/README.md` under `### mprotect` and `book/src/kernel/linux-compatibility/syscall-flag-coverage/memory-management/mprotect.scml` to state that ranges with unmapped holes return `ENOMEM` and that permission changes before the hole may already be visible.

## Retracted by verification

- `c2-8c33f6785b95`: A mapping that can be merged into the just-protected mapping must already have the requested access permissions, and such stale entries are skipped before the unwrap; the provided panic path is not reachable as stated.
