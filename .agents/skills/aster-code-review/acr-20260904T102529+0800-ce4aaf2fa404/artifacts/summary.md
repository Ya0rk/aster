The change appears to improve user-visible memory-management syscall handling by recognizing unmapped ranges for `msync`, and it documents the `Vmar::protect` partial-effect behavior in the API comment.

Remaining issues:

- **Major:** The Linux compatibility documentation for `msync` was not updated to reflect that ranges containing unmapped pages now return `ENOMEM`.
- **Major:** The Linux compatibility documentation and `mprotect.scml` were not updated for the changed `mprotect` behavior, including `ENOMEM` on unmapped holes and the observable partial permission changes before the first hole.
- **Minor:** The `Vmar::protect` API comment should cite the rationale or source for the surprising partial side effect on error, such as Linux `mprotect` semantics or an explicit local design note.

Cross-cutting recommendation: keep syscall implementation changes, API comments, and Linux compatibility artifacts updated together, especially when behavior is user-visible or includes non-atomic side effects.