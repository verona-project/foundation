# Verona Foundation

An initial Foundation library written in Verona. It provides explicit allocation,
a segmented arena, binary data, UTF-8 strings, checked numeric utilities, absolute
time, structured errors, and statically specialized collections. Algorithms and
storage management are Verona code; only POSIX allocation/free and wall-clock
access use C calls. No compiler changes, reference counting, Regions, garbage
collector, object headers, runtime registry, or reference tracking are introduced.

## Build and run

Use the repository development shell and configured build:

```sh
cd projects/foundation
../../build/verona build foundation ../../build/foundation
../../build/verona build foundation-tests ../../build/foundation
../../build/verona build foundation-example ../../build/foundation
../../build/foundation/foundation-tests
../../build/foundation/foundation-example
```

`foundation-tests` runs 252 named unit tests and one separate integration test.
See [Testing](TESTING.md) for coverage, allocation checks, and selecting a module
or individual test.
`foundation-example` runs the arena-backed Array<String>, Dictionary<String,u64>,
Set<u64>, and mutable Data example in `tests/test_integration.vrn`. Both return
zero on success. `foundation` builds a static archive, with source modules as the
normal Verona consumption interface; it does not declare a C export API.

Add this package's `src` and `../base/src` to a consumer's module paths. Import
individual modules explicitly; `foundation.vrn` is the library build aggregator.
Each file defines its own namespace. Collection macro expansions refer to `base`,
`array`, `allocator`, and `error`; clients must import those module names too
(aliases alone do not introduce the original qualified names).

## Memory and lifetime contract

- An allocator is a `backend` value with a context pointer and two typed callbacks.
  `allocate`, `resize`, and `deallocate` consistently use that backend. Contexts
  must outlive their users. Layouts require a nonzero power-of-two alignment.
- System zero-size allocation obtains a distinct, freeable allocation of at least
  one byte. Arena zero-size allocation consumes one byte. Deallocation must use
  the same backend and original requested size/alignment. Never free borrowed
  pointers, views, or empty descriptors' sentinel pointers.
- `resize` allocates a replacement, copies the smaller length, and releases the
  original only on success. Failure preserves the original allocation. Checked
  addition/multiplication reject overflow before allocating collection storage.
  Integer overflow guards use Base’s `minBound` and `maxBound`; collection
  doubling checks use half the unsigned maximum.
- Arena creation is lazy. Blocks have at least the configured initial capacity;
  oversized requests get larger blocks. Block alignment satisfies its request.
  Allocation scans reusable blocks and bumps an aligned offset, allocating a new
  segment when needed. `used` includes alignment padding; `capacity` counts
  payload bytes, excluding the 40-byte metadata per block. Blocks are not moved.
- `reset` sets every block's cursor to zero and retains storage; `destroy` frees
  all backing allocations. **Both invalidate every outstanding arena allocation,
  view, string, and collection.** There are no lifetime checks. A destroyed arena
  descriptor becomes empty and can be reused. Arena release callbacks do nothing.
- Keep an arena descriptor at a stable address while a backend points to it.
  Destroy/reset shared arenas only after all their clients have finished.
- Data, String, and collection descriptors own their allocated storage. Plain
  assignment copies the descriptor, not its storage; it does not create a second
  owner. Use explicit copy APIs for independent allocations. Call `destroy`
  exactly once per owner, before its allocator context expires. Empty/repeated
  destruction is allowed on the same descriptor; using a copied owner is not.
- Removal/clear changes logical contents, retaining capacity. Arena-backed
  resize/rehash retains obsolete storage until reset/destroy. Elements are shallow
  values: Foundation never runs destructors. Callers release nested owners and
  external resources explicitly, including removed/replaced values.
- A borrowed view/get pointer remains valid only while its owner remains alive
  and storage has not moved. Growth, removal, replacement, clear, or reset can
  invalidate addresses or their logical contents. Read-only views are a contract;
  the current type system cannot enforce pointer immutability.
- Copy, byte assignment, and concatenation require destination storage to be
  distinct from borrowed source storage. Visitor callbacks must not mutate the
  collection being visited. APIs are unsynchronized; callers coordinate access.

## API reference

Every item below is exported by its named module. Functions that mutate an
owner accept a pointer to that owner; successful mutators return `(ok)` unless
their signature states another result type.

### `allocator`

- `error`: `(invalid-layout)`, `(overflow)`, or `(out-of-memory)`.
- `result`: `(allocated (pointer u8))` or `(failed error)`.
- `backend`: an allocation context plus acquire/release callbacks.
- `success(pointer)` and `failure(error)`: construct `result` values.
- `valid-alignment(alignment)`: true for a nonzero power-of-two alignment.
- `allocate(backend, size, alignment)`: delegates to the backend.
- `deallocate(backend, pointer, size, alignment)`: releases a matching allocation.
- `resize(backend, old, old-size, new-size, alignment)`: allocates, copies the
  preserved prefix, and releases the old allocation only after success.
- `copy-bytes(destination, source, count)`: copies `count` bytes; ranges must not overlap.
- `checked-add-size(left, right)` and `checked-multiply-size(left, right)`:
  return `(tuple valid value)` and reject `u64` overflow.

### `system_allocator`

- `create()`: returns the POSIX allocation backend. Its allocations use
  `posix_memalign` and must be returned through the same backend.

### `arena`

- `arena`: segmented allocator state; construct it with `create`.
- `create(backend, initial-capacity)`: returns an empty arena. Zero capacity
  selects the default capacity.
- `allocate(owner, size, alignment)`: returns `allocator:result` from arena storage.
- `allocator(owner)`: exposes the arena as an `allocator:backend`.
- `reset(owner)`: reuses segments and invalidates all outstanding allocations.
- `destroy(owner)`: releases segments, empties the descriptor, and invalidates
  all outstanding allocations.
- `used(arena)` and `capacity(arena)`: report current payload use and retained capacity.

### `data`

- `data`: owned mutable byte storage; `view`: borrowed `(bytes,length)` range.
- `empty(backend)`, `length(data)`, `bytes(data)`, and `borrow(data)`: create
  or inspect storage. `bytes`/`borrow` are invalidated by a moving mutation.
- `slice(view, start, count)`: returns `(tuple valid view)` with checked byte bounds.
- `reserve(owner, capacity)`, `resize(owner, length)`, and `append(owner, byte)`:
  grow or change contents; resize zero-fills newly exposed bytes.
- `assign-bytes(owner, view)`: replaces an empty owner with a copy of the view.
- `copy(source, out)`: creates an independent owner using `out`'s allocator.
- `equal(left-view, right-view)`: bytewise equality.
- `destroy(owner)`: releases owned storage; repeated destruction is safe.

### `string`

- `string`: owned validated UTF-8 storage plus scalar count.
- `empty(backend)`: creates an empty string.
- `validate(view)`: returns `(tuple valid scalar-count)` without allocating.
- `from-utf8(owner, view)`: validates and replaces an empty owner.
- `bytes(string)`, `byte-length(string)`, and `scalar-length(string)`: inspect
  the borrowed UTF-8 bytes and their byte/scalar lengths.
- `equal(left, right)`: exact UTF-8-byte equality.
- `substring(string, start-scalar, scalar-count)`: returns `(tuple valid view)`;
  bounds are Unicode scalar indices, not grapheme indices.
- `find(haystack, needle)`: returns `(tuple found scalar-index)`.
- `copy(source, out)`, `concat(left, right, out)`, and `destroy(owner)`: manage
  owned string storage; `out` selects the allocator for copy/concatenation.

### `number`

- `result`: `(value u64)`, `(invalid)`, or `(overflow)` for decimal parsing.
- `signed-result`: `(signed i64)` or `(overflow)`.
- `parse-u64(view)`: parses nonempty ASCII decimal digits only.
- `to-i64(u64)`: checked conversion to `i64`.
- `format-u64(number, data-owner)`: replaces the destination with decimal ASCII,
  without a terminating NUL.

### `date`

- `instant` and `duration`: signed Unix-second products.
- `result`: `(at instant)`, `(overflow)`, or `(clock-error)`; `duration-result`:
  `(elapsed duration)` or `(overflow)`.
- `now()`: reads wall-clock Unix seconds.
- `before(left, right)`: strict instant comparison.
- `add(instant, duration)` and `subtract(instant, duration)`: checked arithmetic.
- `between(start, end)`: checked `end - start` duration.

### `error`

- `status`: `(ok)`, `(allocation-error allocator:error)`, `(bounds)`,
  `(invalid-input)`, or `(overflow)`.
- `context`: `(code i32, description pointer, detail pointer)` for borrowed
  NUL-terminated diagnostics.
- `success()`, `invalid()`, `out-of-bounds()`, `too-large()`, and
  `allocation(allocator-error)`: construct `status` values.
- `description(status)`: returns a fixed NUL-terminated description.

### Collections

- `array:layout(size, alignment, backend)` describes concrete element storage.
  `array:define PREFIX ELEMENT` generates `PREFIX-empty`, `-length`, `-capacity`,
  `-reserve`, `-get`, `-append`, `-insert`, `-remove`, `-replace`, `-clear`,
  `-copy-into`, `-visit`, and `-destroy`.
- `dictionary:define PREFIX KEY VALUE HASH EQUAL SLOT-SIZE SLOT-ALIGNMENT`
  generates `PREFIX-table`, `-empty`, `-count`, `-capacity`, `-lookup`, `-get`,
  `-contains`, `-insert`, `-remove`, `-clear`, `-visit`, and `-destroy`.
- `set:define PREFIX ELEMENT HASH EQUAL SLOT-SIZE SLOT-ALIGNMENT` generates the
  dictionary operations for a boolean payload plus `PREFIX-add`.

Foundation uses the compiler's concrete default operators directly: `+`, `-`,
`*`, `/`, `=`, `<`, `<=`, `>`, and `>=`. Base's constrained `equality`
protocol uses the same `=` spelling, so generic and concrete equality agree.

A successful mutation returns `error:status` case `(ok)`; errors are sum values.
Allocator, Number, and Date additionally have module-specific result/error sums.
Sum constructor names in matches are unqualified and resolved from the matched
type. Constructors are not exported functions; use module helper functions where
needed. Context descriptions are borrowed NUL-terminated UTF-8/ASCII pointers.

Data's `assign-bytes` constructs/replaces content in an initially empty owner;
`resize` zero-fills newly exposed bytes. `reserve` grows capacity without changing
logical length or contents; streaming clients can reserve geometrically before
copying a chunk. Slices are checked borrowed views, indexed
in bytes. Out parameters are existing descriptors initialized with the desired
allocator, making destination allocation policy explicit.

String stores validated UTF-8, including embedded NUL. Validation rejects
continuation errors, truncation, overlong encodings, surrogates, and scalars above
U+10FFFF. `substring(start,count)` indexes **Unicode scalar values**, not bytes or
grapheme clusters, and returns a borrowed byte view with complete scalar boundaries.
`find` returns `(tuple found scalar-index)`, with the first match; an empty needle
matches at zero. Equality compares exact UTF-8 bytes. `bytes` is not necessarily
NUL-terminated. Unicode normalization and locale collation are outside this version.

Number accepts nonempty ASCII decimal digits only (no sign, whitespace, separators,
or prefixes), detecting u64 overflow. Formatting replaces destination Data with
ASCII decimal digits without a terminating NUL. Other primitive arithmetic,
comparison, and conversions remain in Verona's `base`; Foundation adds no boxing.

Date uses signed Unix seconds since 1970-01-01T00:00:00Z, with one-second wall-clock
resolution. Durations are signed seconds. `between(start,end)` computes end minus
start. All arithmetic checks overflow, including minimum signed values. `now`
reports a clock error for the platform's -1 failure sentinel. These are instants,
not local calendar dates; wall-clock readings need not increase monotonically.

## Collection specialization

Verona currently needs macro specialization for owning collections; see
[the compatibility assessment](COMPATIBILITY.md). This is existing compiler syntax:

```lisp
(import base)
(import allocator)
(import error)
(import array)
(import dictionary)
(import set)
(import system_allocator :as system)

(array:define integers u64)
; integers-empty takes a typed witness pointer and array:layout.
; The witness is never dereferenced while length/capacity are zero.

(base:function hash ((key u64)) u64 key)
(base:function equal ((left u64) (right u64)) bool (= left right))
(dictionary:define scores u64 u64 hash equal 32 8)
(set:define identifiers u64 hash equal 32 8)
```

`(array:define PREFIX ELEMENT)` generates `PREFIX-empty`, `-length`, `-capacity`,
`-reserve`, `-get`, `-append`, `-insert`, `-remove`, `-replace`, `-clear`, `-copy-into`,
`-visit`, and `-destroy`. An array is `(tuple (pointer ELEMENT) u64 u64 array:layout)`.
`layout` takes allocation size per element, alignment, and allocator. `reserve`
provides initial capacity; append doubles capacity starting at four. `get` returns
`(tuple valid pointer)`; never dereference the pointer on failure. Insert accepts
index equal to length; access/remove/replace require index less than length.
`copy-into` preserves the destination allocator. Visitors take one element by value.

`(dictionary:define PREFIX KEY VALUE HASH EQUAL SLOT-SIZE SLOT-ALIGNMENT)` generates
`PREFIX-table`, `-empty`, `-count`, `-capacity`, `-lookup`, `-get`, `-contains`,
`-insert`, `-remove`, `-clear`, `-visit`, and `-destroy`. Helper declarations also
exist in the expanding module; treat them as private implementation details.
`lookup` returns `(tuple found slot-index)`; indices are invalid after mutation.
`get` writes a supplied value pointer on success and leaves it unchanged on absence.
`insert` updates existing keys. `remove` returns whether an entry existed.
Visitors take a key and value by value. Generated declarations can be re-exported
by their containing module using ordinary Verona exports.

Dictionary uses open addressing, linear probing, tombstones, cached hashes, and
rehashing at 50% live occupancy. Growth is transactional on allocation failure.
Iteration order is unspecified and can change after any mutation. Equality must
be an equivalence relation; equal keys **must** have equal hashes. Stored keys and
hash/equality behavior must stay unchanged while entries exist. The table owns
its slot allocation, not separately allocated resources inside keys/values.

`(set:define PREFIX ELEMENT HASH EQUAL SLOT-SIZE SLOT-ALIGNMENT)` generates the same
table and query/removal/visit operations and adds `PREFIX-add(owner,value)`. The
shared internal `-insert` operation has a boolean payload; normal Set callers use
`-add`. Dictionary-style visitors receive that boolean as their second argument.

**Layout is an explicit unsafe caller contract.** Element/slot allocation sizes
must be at least their actual ABI size, and alignment must satisfy the actual
ABI alignment. Pointer arithmetic uses the compiler's true type stride. Wrong
layouts can corrupt memory. On this compiler's 64-bit ABI, u64 arrays need 8/8;
u64/u64 and u64/bool hash slots need 32/8; String arrays need 72/8; String/u64
hash slots need 96/8. No automatic size or alignment discovery is claimed.

## Verification and remaining work

Implemented and exercised: each module, growth, collisions/rehash/tombstones,
duplicate updates, allocator failures including failed arena block acquisition and
failed rehash, arena reuse/reset/destruction, binary copying/slices, UTF-8 validity
and scalar boundaries, integer boundaries/formatting, time arithmetic, and shared
arena integration. Tests never access invalidated arena allocations.

The suite and static library compile with the available LLVM-backed compiler;
verification was performed on this Darwin host. Linux uses the same POSIX calls
and 64-bit layout assumptions but has not been executed here. Other operating
systems and 32-bit targets are unsupported. Large recursion depth may exhaust the
stack at low optimization; tail-call elimination is not a public guarantee.

Blocked by compiler capabilities: automatic layouts, ordinary parameterized named
collection types, generic bodies containing local type-parameter declarations or
casts, field-place mutation, and enforced immutable/lifetime-safe borrowed views.
Whole-descriptor writes and macro specialization provide the working alternatives.

Deferred enhancements: signed integer parsing/formatting, additional checked numeric
conversions, lexicographic string ordering, Unicode normalization/collation,
higher-resolution/monotonic clocks, amortized arena block indexing, stable iterators,
resource cleanup callbacks, and performance/large-input stress benchmarks. Next,
add compiler layout queries and generic local-type resolution, then remove manual
layouts and simplify collection specialization; verify the suite on Linux.

## Changed files and verification record

All new files are under `projects/foundation/`:

- `src/allocator.vrn`, `src/system_allocator.vrn`, `src/arena.vrn`: memory backends and arena.
- `src/data.vrn`, `src/string.vrn`, `src/number.vrn`, `src/date.vrn`, `src/error.vrn`: primitive Foundation APIs.
- `src/array.vrn`, `src/dictionary.vrn`, `src/set.vrn`: collection specialization and algorithms.
- `src/foundation.vrn`: build aggregation.
- `tests/test_allocator.vrn`, `tests/test_system_allocator.vrn`, `tests/test_arena.vrn`, `tests/test_data.vrn`,
  `tests/test_string.vrn`, `tests/test_number.vrn`, `tests/test_array.vrn`,
  `tests/test_dictionary.vrn`, `tests/test_set.vrn`, `tests/test_date.vrn`,
  `tests/test_error.vrn`, `tests/test_compat_arithmetic.vrn`: module and default-operator regression tests.
- `tests/support.vrn`, `tests/test_integration.vrn`, `tests/suite.vrn`,
  `tests/example.vrn`: deterministic failing allocator, shared-arena integration,
  complete runner, and executable example.
- `verona.build`, `README.md`, `COMPATIBILITY.md`, `TESTING.md`: build targets, API/ownership
  documentation, and compiler capability assessment.

The root Verona Makefile builds compiler tools but has no Foundation-specific
test target. Build Foundation targets directly through this package's manifest,
as shown above.

Current verification on the available host/compiler:

- `verona build foundation`: static archive produced.
- `verona build foundation-tests` followed by its executable: all 253 named
  cases pass.
- `verona build foundation-example` and its executable: exit 0.
- `git diff --check`: clean.

Implemented/tested functionality is described above. Implemented but not independently
verified here: Linux execution, exact address-modulo alignment inspection (tests
request aligned allocations but the source language cannot inspect pointer bits),
and large-input performance/stack behavior. Automatic layouts and stronger typed
ownership remain compiler-blocked; the other enhancements listed above are deferred.

The formatter application additionally exercises large Foundation buffers. Byte
copying, Array copying, Data zero-fill/equality, and UTF-8 validation now use
bounded-depth traversal, avoiding reliance on pointer-bearing tail-call
optimization. Other recursive APIs retain the stack limitations noted above.

## Standalone checkout

Foundation depends on Verona Base. Place its source checkout beside this
repository as `base/`; the tests also require a sibling `testing/` checkout.
With a compatible Verona compiler on your PATH, run from this repository:

```sh
verona build foundation ./build
verona build foundation-tests ./build
./build/foundation-tests
verona build foundation-example ./build
./build/foundation-example
```

The commands above run from the main
[Verona](https://github.com/verona-project/verona) checkout, where this package
is pinned at `projects/foundation`. Use its pinned Nix development environment
for native verification. Build the formatter separately through the root
manifest when exercising that consumer.

Public clones use HTTPS. To push Foundation changes over SSH:

```sh
git remote set-url --push origin git@github.com:verona-project/foundation.git
```

Commit changes here before updating Verona's submodule reference.
Released under the Unlicense; see `LICENSE`.
