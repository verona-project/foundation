# Compiler compatibility assessment

Assessment preceded implementation and was refined using actual compiler diagnostics.
The available compiler supports nominal products/sums, structural tuples (2–4 fields),
generic functions and protocols, typed function pointers, pointer arithmetic, writable
dereferences, explicit void-pointer casts, and scalar/pointer C ABI declarations.
Modules use filenames and explicit exports. Tests use the bundled `testing` package.

| Capability | Observed result and library adaptation |
| --- | --- |
| Parameterized nominal products/sums | No declaration syntax; use structural descriptors and macro-generated concrete collection declarations |
| Type parameters in local annotations/casts | Generic function signatures work, but Array reserve's generic local/cast failed with `unknown type`; specialize collection functions through existing macros |
| Recursive nominal layouts, even pointers to self | Compiler explicitly rejects them; arena block links use pointer-void with a completed block cast |
| Source-level sizeof/alignof | No supported operation; callers supply collection layouts, and arena metadata is isolated to the known 64-bit ABI (40 bytes, alignment 8) |
| Writable product field places | Compiler reports `expression is not writable`; update the complete value through a writable descriptor pointer |
| Numeric literal inference in generic comparisons | Use explicitly typed constants for literals and typed locals for bounds and conversions; arithmetic and comparisons use existing operators directly |
| Sum construction at the end of `do` | Context is not propagated in every position; use typed result helpers or typed local bindings |
| Qualified sum constructors as imported functions | Not exported declarations; match unqualified constructors against known sum types and expose explicit helper functions |
| Exhaustiveness of partial tuple patterns | Separate true/false tuple patterns do not establish exhaustiveness; use a final whole-tuple binding pattern |
| Loops | Use recursive functions and `match`; no new core constructs |
| Borrow checking, const pointers, automatic destruction | Not provided; ownership and invalidation are explicit caller obligations |

Supported milestone target: 64-bit Darwin/Linux POSIX ABI. Native tests ran on Darwin;
Linux and other targets are not claimed as tested. No compiler changes or C replacement
algorithms are included. All implemented modules are compiled by the Foundation target,
and macro-generated collections are concretely instantiated by the tests. Missing
language features have working adaptations except automatic/enforced layout and
ownership safety, which remain explicit limitations rather than implied guarantees.

Base and compiler operator behavior are unchanged. Foundation calls existing
operators directly without arithmetic compatibility modules.
