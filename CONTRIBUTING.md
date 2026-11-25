# Contributing

## Commit messages

Commit subjects follow [Conventional Commits](https://www.conventionalcommits.org/).
This repository's deliverable is the test suite, so a change's type is judged by its effect on the suite: on what the tests assert, and on how the suite is built and run against a backend.

A change that leaves all code untouched except its comments, the code being the headers under `include/`, the `Makefile` and the scripts under `scripts/` that build and run the suite, takes `docs` when it changes only documentation, meaning Markdown files such as `README.md` or this one and comments in the code, and `chore` otherwise, as for a change to `LICENSE` or `.gitignore`.
For any other change, the following types are read against its effect, and a change whose effect one of them describes takes that type:

- `feat`: new coverage, such as a new test case or a newly covered API, or new suite infrastructure, such as adapters, tolerance categories, skip macros or the integration check.
- `fix`: a test that asserts the wrong thing, including one that asserts behavior TCAPI V1 does not define, or any other bug in the suite or in how it is built or run.
- `refactor`: any other change that leaves what the tests assert, and how the suite is built and run, unchanged, such as a reformat or a rename.

`test` and `build` are not used: `test` would give one type to a new test and a corrected one, which `feat` and `fix` tell apart here, and a change to how the suite is built is typed by its effect like any other.

## Formatting

C and C++ files are formatted by clang-format in LLVM style as adjusted by `.clang-format`, and Markdown files by mdformat.
Shell scripts are checked by shellcheck.
[pre-commit](https://pre-commit.com/) ([installation](https://pre-commit.com/#install)) runs these, along with checks for trailing whitespace, missing final newlines, YAML syntax and leftover merge-conflict markers, on every commit once the hook is installed:

```sh
pre-commit install
```

To format every file at once:

```sh
pre-commit run --all-files
```

Tool versions are pinned in `.pre-commit-config.yaml`.

`.git-blame-ignore-revs` lists commits that reformat the suite.
To have `git blame` skip them, run once:

```sh
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

## Local integration check with the cytnx backend

`make integration-check` verifies the current tcapict HEAD against the [cytnx
backend](https://github.com/r-ccs-cms/tcapi-backend-cytnx)
without the manual "copy headers / build / `git checkout`" routine.

### Default behavior

```sh
make integration-check
```

This:

0. Refuses to run while tracked files of this repo have uncommitted changes, or when `git diff` fails while checking for them.
1. Clones cytnx (with submodules) into `./build/tcapi-backend-cytnx/` if it does not yet exist, then runs `git fetch`, `git reset --hard $CYTNX_REF`, and `git submodule update --init --recursive --force` on every invocation so the checkout always tracks `$CYTNX_REF` (default under "Environment variables" below).
2. Refuses to continue unless the cytnx working tree's `.gitmodules` registers a submodule at the path `external/tcapict` and that submodule is initialized, since a cytnx revision that vendors this suite under another path cannot take the override.
3. Replaces `external/tcapict/` content with this repo's tracked HEAD via `git archive HEAD | tar -xmf -`; `-m` stamps the extracted files with the current time, so the build recompiles against them.
4. Builds the cytnx test target (`TCAPITests`).

`build/` is gitignored, so the cloned backend and its cmake artefacts stay untracked.

### Use an existing cytnx checkout

```sh
make integration-check CYTNX_BACKEND_DIR=$HOME/path/to/tcapi-backend-cytnx
```

When `CYTNX_BACKEND_DIR` is set in the environment, the script skips step 1: it does **not** clone, fetch or reset that checkout. Steps 0 and 2–4 still run, so the script overrides `external/tcapict/` and configures and builds in `$CYTNX_BACKEND_DIR/build`, and the checkout must pass step 2 and provide the signatures the suite calls; see `CYTNX_REF` under "Environment variables". To return to the auto-clone default, unset the variable rather than passing the default value explicitly.

### Run the tests

```sh
make integration-test
```

Builds, then runs `$CYTNX_BACKEND_DIR/build/test/TCAPITests` directly.

### Restore the cytnx submodule

```sh
make integration-restore
```

Resets `$CYTNX_BACKEND_DIR/external/tcapict` to cytnx's pinned submodule revision (`git submodule update --init --force external/tcapict`) and runs `git clean -ffd` inside that path. The clean step is required: the override adds files that do not exist at the submodule pin (Makefile, CONTRIBUTING.md, scripts/, .gitignore), and a plain `submodule update --force` would leave those files as untracked.

### Extra cmake configure args

Some platforms need more than `-DTCAPI_BUILD_TESTS=ON` to configure the cytnx build. Pass the additional flags via `INTEGRATION_CMAKE_ARGS`:

```sh
# macOS + Homebrew toolchain (matches cytnx's CMakePresets.json `brew-debug`)
INTEGRATION_CMAKE_ARGS='--preset brew-debug' make integration-check

# Or set the toolchain file directly
INTEGRATION_CMAKE_ARGS='-DCMAKE_TOOLCHAIN_FILE=cmake/toolchains/macos-homebrew.cmake -DUSE_HPTT=ON' make integration-check
```

`INTEGRATION_CMAKE_ARGS` is appended verbatim to the `cmake -S ... -B ...` line.

### Caveats

- The override copies tcapict **HEAD** (last committed state), which is why step 0 refuses on uncommitted changes, in `make integration-check` and `make integration-test` alike; commit or stash them first. Untracked files are not shipped either.
- The `TCAPICT_SKIP_*` skip-macro matrix (cross-build with various skip-macro combinations) is deferred to a follow-up issue.
- `make integration-restore` acts on the same checkout (`CYTNX_BACKEND_DIR`, or the default path), which must exist and be a git checkout that meets step 2's first condition, the `.gitmodules` registration; its second does not apply, so the submodule need not be initialized.

### Environment variables

| Variable | Default | Effect |
| -- | -- | -- |
| `CYTNX_BACKEND_DIR` | `./build/tcapi-backend-cytnx` | cytnx checkout to override into; **set** disables auto-clone/sync |
| `CYTNX_REPO_URL` | `https://github.com/r-ccs-cms/tcapi-backend-cytnx.git` | URL used only when auto-cloning the default path |
| `CYTNX_REF` | `origin/main` | ref the default-path checkout is reset to on every run; it must provide the TCAPI V1 signatures this suite calls, such as `concatenate`, `stack` and `linear_combine` over `List<CRef<TenT>>` |
| `INTEGRATION_CMAKE_ARGS` | (empty) | extra args appended to the cmake configure command |

## Conventions for writing tests

These apply to every header under `include/tcapict/`; comments in the code refer to them by section name.

### Failures

In this section a test is a test function together with the construction of the `tcapi_test_fixture` it runs on.
At run time, a test reports a failure by letting a `std::exception` leave it.
A failed assertion throws `tcapict::assertion_error`.
A `std::exception` the backend throws that leaves the test does so as it was thrown, or, where the test converts it, as an `assertion_error`; `TCAPICT_ASSERT_NOTHROW` and `TCAPICT_NOTHROW_VALUE` in `include/tcapict/assertion.h` convert one from the expression they run and report it at their own line.
So a harness that runs the tests, whether an adapter under `include/tcapict/adapters/` or one a backend writes, must count every `std::exception` that leaves a test as a failure, as doctest does, and must not take an `assertion_error` to mean that an assertion failed.
What it does with an exception of any other type is its own choice; doctest counts that as a failure too.
A handler in the suite, by contrast, must catch nothing wider than `std::exception`, for the reason the docstring of `detail::nothrow_call` in `include/tcapict/assertion.h` gives.

A call to `concatenate`, `stack` or `linear_combine` must go through `TCAPICT_NOTHROW_VALUE`, so that a throw from it is reported at its own line.
Any other backend call may go through `TCAPICT_ASSERT_NOTHROW` or `TCAPICT_NOTHROW_VALUE` where reporting a throw at the call is worth it.

### What a test must accept

A test must compile for every backend that conforms to TCAPI V1, and pass for it on the inputs the test uses at an `epsilon()` the backend can set, as the next paragraph describes.
So an assertion must assert only what a TCAPI V1 clause states, must assert nothing about behavior TCAPI V1 leaves unspecified, and must use the backend's types only as TCAPI V1 says they can be used.
Where another section here states how the suite reads a TCAPI V1 clause, that reading is what the clause states throughout this section, in what conforming to TCAPI V1 means as well.

TCAPI V1 gives no error bound for any operation, so how close a floating-point result must be is this suite's policy, not TCAPI V1's: `tolerance()` in `include/tcapict/fixture.h` returns it for each `tol_category`: 0 for `exact`, and a multiple of the fixture's `epsilon()`, which a backend sets by specializing `tcapi_test_fixture`, for the others.
A larger `epsilon()` must only loosen an assertion: an assertion a conforming backend passes at one `epsilon()` it must pass at every larger one.

A review finding that shows an assertion breaking this section is fixed in the change at hand when that is within the change's purpose, and filed as a follow-up issue otherwise.

An assertion that a conforming backend can fail is exempt from the requirements above exactly when its test's comment, or the section of these conventions that covers it, says so and gives the reason it is kept, such as a fault in the class under "What a test must reject" that it rejects and would let through if it were weakened until every conforming backend passed it.

### What a test must reject

No finite set of tests rejects every backend that does not conform to TCAPI V1: one that is correct on the inputs the suite uses and wrong on every other input passes all of them.
So a test is judged against a class of faults, decided by what the faulty backend does, not by how the fault is worded.

A fault is in the class when the backend applies one rule, the same for every input of the clause the test checks, and that rule differs from the clause in one of these ways:

1. an operator, a quantity, or the order of two steps is replaced by another;
2. a step of the clause's computation is done wrong, such as an index off by one or a sum over the wrong range;
3. a parameter the clause gives a role is ignored.

Every other fault is out of the class.
Among them are a fault that cannot be stated as one such rule without dividing the inputs into cases the clause does not distinguish, such as a rule that applies only when some argument is positive where the clause treats every value of that argument alike, and a backend that recognizes the suite's inputs.
What decides is how the fault can be stated, not which inputs show it: a rule applied to every input, differing from the clause in one of the ways above, is in the class even when it differs from the clause only on some inputs, such as visiting the stored entries instead of every element, which differs only for a compactly stored tensor.
So is moving a case boundary the clause draws itself, such as `<=` in place of `<` at `s_min`, which is item 1.

Each input parameter whose effect on an overload TCAPI V1 specifies must have at least one test that calls that overload with an expected result depending on that parameter.
A review finding that names a fault in the class that no test rejects is a gap, and is fixed or filed as under "What a test must accept".
A finding that names a fault some test already rejects, or a fault out of the class, needs no change, though a test may still be added that rejects the latter.

Applied to faults proposed for `trunc_svd`'s overload (1), whose clause reads "Fixed maximum bond dimension kept: keep at most `chi_max` singular values after discarding those below `s_min`", and to the truncation error returned in `trunc_err`, whose numerator TCAPI V1 defines as the sum of the squares of every singular value not retained:

- dropping the `chi_max` cap only when `s_min` is positive divides the inputs by the sign of `s_min`, which the clause does not, so it is out of the class; dropping the cap for every input ignores `chi_max`, item 3;
- a cap off by one only when `s_min` is positive is out of the class for the same reason; a cap off by one for every input is item 2;
- ignoring `s_min` is item 3;
- leaving the values `s_min` discards out of the truncation error's numerator is item 2: summing over only the values the cap removes is one rule for every input, although it differs from the clause only when `s_min` discards something.

### Out-of-place inputs

TCAPI V1: "In-place variants mutate the supplied tensor operand(s), whereas out-of-place variants leave all inputs unmodified".
The tests that read their input back after an out-of-place call assert that "leave all inputs unmodified" for the elements they read.

### Inputs to multi-input operations

`concatenate`, `stack` and `linear_combine` take their inputs as `List<CRef<TenT>>`, and TCAPI V1's entry for each of them says: "In the C++ interface, `ins` is a list of non-owning const references to existing tensors; the referenced tensor objects are not copied, consumed, or modified and must remain valid for the duration of the call."
Each test reads every input back after its last call to the operation and asserts that its shape and elements are unchanged.
That asserts the "not ... modified"; the read-back cannot detect a copy, so "not copied" is not asserted.
An input whose elements all hold one value would pass that read-back even after being reordered in place, so each input element gets a value no other input element has.
Each input other than a zeroth-order one is built from a coordinate-to-value function, through `TCAPICT_SET_ELEMS` in `include/tcapict/elem_sweep.h`, and both the read-back and the check of every element of the result are computed from the same functions, through `TCAPICT_EXPECT_ELEMS`; `ramp_over` there gives distinct values to distinct coordinates.
A zeroth-order input has one element, which no reordering can move, so it needs only a value no other input holds; it is set through `TCAPICT_SET_ZEROTH_ORDER` in `include/tcapict/zeroth_order.h`, and read back, as is the result, through `TCAPICT_EXPECT_ZEROTH_ORDER`.

Each of the three operations, and each `linear_combine` overload, is called at least once through a named `tcapi::List<tcapi::CRef<TenT>>` variable, not only through a braced `{std::cref(a), ...}` list.
When `TenT` is copy-constructible, a braced list of `std::cref`s also initializes a `List<TenT>`, so a braced call alone compiles against a backend that still takes `List<TenT>`.
A named `List<CRef<TenT>>` does not convert to `List<TenT>`, so that call compiles only against TCAPI V1's parameter type.
A call with a braced list names `TenT` explicitly, as in `tcapi::stack<TenT>(ctx, {std::cref(a), std::cref(b)}, 0)`, because no parameter of these functions deduces it from a braced list.

### Callbacks the backend invokes

Coordinate maps, generators and traversal callbacks run inside the backend's frames, which a conformance suite cannot assume are able to carry an exception.
The suite reads TCAPI V1 as invoking them one at a time: TCAPI V1's own `for_each` example accumulates into a plain variable, `[&tot](const float el) { tot += el; }`, and gives the result as `tot == 3.0`.
So they do not assert, do not allocate, and read a coordinate's components only after checking the coordinate: that it has as many components as the tensor has bonds, and that each is less than its bond's dimension.
Whatever a callback observes for the test, such as a flag for a malformed coordinate, a visit count or a running sum, it keeps in variables the test reads after the call returns, and the test asserts on those.
A callback that receives a second-order coordinate makes that check with `coors_2d` in `include/tcapict/elem_helper.h` before reading its components, and records a coordinate `coors_2d` rejects in a flag the test asserts is clear.
Tests call `to_range` and `assign_from_range` on second-order tensors through `to_range_2d` and `assign_from_range_2d` in `include/tcapict/range_2d.h`.

### Zeroth-order tensors

TCAPI V1, in the introduction to its core functions: "A zeroth-order tensor has empty shape, size 1, and a single empty coordinate sequence."
A callback that receives a coordinate of a zeroth-order tensor expects the empty one and flags any other, under "Callbacks the backend invokes", through `to_range_0d`, `assign_from_range_0d` or `visit_record_0d` in `include/tcapict/zeroth_order.h`.

TCAPI V1 goes on: "Operations whose definitions require selecting, reordering, inserting, removing, slicing, or partitioning existing bonds are not applicable to zeroth-order tensors unless explicitly specified by the operation."
It names no such operation. The suite reads `transpose`, `svd`, `qr`, `eig`, `expand`, `shrink`, `extract_sub` and `replace_sub` as such operations, and none of their entries specifies a zeroth-order case.
TCAPI V1 defines no exception for them, so nothing about calling them on a zeroth-order tensor is assertable, and no test calls them on one.
No test calls the following on a zeroth-order tensor either, since the suite reads the same sentence as covering them, and their entries specify no zeroth-order case: `concatenate`, which selects a bond, `stack`, which inserts one, and the operations that matricize, which partition bonds into rows and columns (`trunc_svd`, `lq`, `eigh`, `eigvals`, `eigvalsh`, `exp`, `inverse`).
Nor does any call `diag`, whose input TCAPI V1 requires to be first- or second-order.
The suite reads the operations its zeroth-order tests do call as requiring no existing bond of a zeroth-order operand; `contract` is among them, since such an operand takes the empty label list `""`.

The zeroth-order tests assert nothing about `TenT{}`, the state `clear` and `move` leave, since TCAPI V1 does not say what queries return on it; the `order == 0` assertions the existing `clear` and `move` tests make on it, which a backend conforming to TCAPI V1 can therefore fail, are left as they are, pending the open question on [#51](https://github.com/ultimatile/tcapi-conformance-test/issues/51).
Of `size_bytes` they assert only that it returns, as the tests of the other ranks do.

### Diagonal tensors

TCAPI V1, in the introduction to its core functions: "TCAPI treats a diagonal tensor as an ordinary second-order tensor with shape {N,N} whose off-diagonal logical tensor elements are zero." and "Unless stated otherwise, TCAPI operations are defined by logical tensor elements, not by storage representation."
Its entries name these diagonal tensors: `eye` returns "the N×N identity tensor as a diagonal tensor", `diag` of a first-order input and the `sigma` of `trunc_svd` and `lambda_mat` of `eig` are each "a second-order diagonal tensor", and the `sigma` of `svd` and `lambda_mat` of `eigh` are each "a real second-order diagonal tensor".
Off-diagonal elements must be exactly zero, since TCAPI V1 defines a diagonal tensor as one "whose off-diagonal logical tensor elements are zero".
TCAPI V1 fixes the order of the diagonal for `svd` ("non-increasing order") and for `eigh` ("ascending order").
It states none for `eig`, nor for the `sigma` of `trunc_svd`, whose entry says that the routine "follows the same matricization and SVD convention as svd" and retains the singular values "in descending order" but states no order for the diagonal of its output `sigma`; so the `trunc_svd` tests, and the `eig` tests wherever the eigenvalues are not all equal, compare the diagonal as a multiset, through `diag_order::any` in `include/tcapict/diagonal.h`.
