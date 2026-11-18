# LANE 2 report: Foundation (NSA Cryptol specifications)

Fork: https://github.com/Dukotah/Foundation. Upstream:
https://github.com/NationalSecurityAgency/Foundation. Nothing has been
opened upstream.

| Branch | Base | Commits | Purpose |
|---|---|---|---|
| `tests/sp800-185-kats` | upstream/main | 1 (`e6d798a`) | Upstream PR #1 candidate |
| `tests/sha1-pbkdf2-kats` | upstream/main | 1 (`89877a2`) | Upstream PR #2 candidate |
| `lane/hkdf` | upstream/main | HKDF (`027b706`), DevGuide (`493dc33`), this report | Fork extension / possible PR #3 |

All commit subjects start with the placeholder `#NN`. **Replace it with
the real issue number after filing the issue** (`git commit --amend`,
then force-push the branch).

## 1. Upstream contribution rules (CONTRIBUTING.md)

- PRs welcome, but open a dialogue (issue) first for anything beyond a
  small fix; search existing issues first.
- Smallest reasonable change; no refactors, renames or find-and-replace
  changes; isolate independent patches into separate PRs.
- Squash to one commit whose message **starts with the issue number**.
- No CI pipeline or third-party dependency changes from outsiders.
- Cryptol must follow `docs/cryptol-style.md` (70-column lines, cite
  the standard, load without warnings); Rust must pass `cargo fmt` and
  `clippy -Dclippy::pedantic`.
- Contributions are licensed inbound=outbound (Apache-2.0).

## 2. What changed and why

### How tests work in this repo (verified)

Test vectors are Cryptol modules under `algorithms/<Name>/tests/`
containing `property` declarations (see `SHAKE/tests`, `SHA3/tests`,
`HMAC/tests`). CI (`.github/workflows/assurance-main.yml`) only
*loads* every `.cry` file via `.ci/load-all-cry-files.py`; it does not
evaluate the properties (the DevGuide has a "TODO running tests via
`:check-docstrings`"). Properties are run by hand with `:check`. The new
tests follow exactly that convention, so CI will typecheck them and
nothing in CI changes.

### Branch A: `tests/sp800-185-kats`

- 14 new test modules: `cSHAKE/tests/cSHAKE{128,256}.cry`,
  `KMAC/tests/KMAC{128,256}.cry`, `KMAC/tests/KMACXOF{128,256}.cry`,
  `TupleHash/tests/TupleHash{,XOF}{128,256}.cry`,
  `ParallelHash/tests/ParallelHash{,XOF}{128,256}.cry`.
- **All 40 samples** in NIST's seven SP 800-185 "Example Values" PDFs
  (cSHAKE 4, KMAC 6, KMACXOF 6, TupleHash 6, TupleHashXOF 6,
  ParallelHash 6, ParallelHashXOF 6), with key, N/S customization
  strings, tuples and block size B as given. Property names keep NIST's
  sample numbers.
- Vectors were extracted from the PDFs (`curl` + `pdftotext -layout`)
  by a parser that asserts every declared length (data, key, tuple
  count, output length) matches the hex it read, then the generated
  modules were cross-checked: every hex literal in every test file
  occurs in the PDF text (1,132 literals, 0 missing).
- Bibliography entries (`[cSHAKE Examples]` etc.) added.
- **Real bug found and fixed:** `algorithms/ParallelHash/Main.cry`
  declared `module algorithms::TupleHash::Main`. Importing it by path
  fails:

  ```
  [error] .../algorithms/ParallelHash/Main.cry:6:8--6:35:
      File name does not match module name:
        Actual: algorithms::TupleHash::Main
      Expected: algorithms::ParallelHash::Main
  ```

  CI did not catch this because it loads files by path rather than
  importing them. The PR changes that one line.
- No spec failed any NIST vector.

### Branch B: `tests/sha1-pbkdf2-kats`

- `SHA1/tests/Examples.cry`: "abc" and the 448-bit two-block message
  from NIST's SHA-1 example values PDF.
- `SHA1/tests/ShortMsgByte.cry`: all 65 vectors of CAVP
  `SHA1ShortMsg.rsp`, in the SHA-3 `ShortMsgByte` style.
- `PBKDF/tests/HMAC_SHA224.cry`: two PBKDF2-HMAC-SHA-224 vectors from
  NIST's ACVP server JSON (tcId 20: C = 1, 1432-bit key; tcId 33:
  C = 183, 1312-bit key).
- Not included, and why (each one was run):
  - **SHA-1 one million 'a' (FIPS 180-2 A.3):** the heap ran out at the
    1.8 GB cap after 17 s (`cryptol: Heap exhausted`).
  - **RFC 6070 / RFC 7914 PBKDF2 vectors:** 7 of the 8 use salts
    shorter than 128 bits. `PBKDF2`'s type requires `sLen >= 128`
    (SP 800-132 Section 5.1), so they don't typecheck:
    `Unsolvable constraint: 32 >= 128 arising from use of expression
    algorithms::SHA1::Main::PBKDF2`. The one that does typecheck
    (RFC 6070 #5, C = 4096) had not finished after 7 minutes at the
    1.8 GB heap cap, so I stopped it. Memory use goes up with C × blocks
    (tcId 33, about 1,100 HMACs, peaked at 1.47 GB). I suspect
    `iterate step [0, U0i] @ C` in `PBKDF/Main.cry` keeps the whole
    stream in memory, but I haven't confirmed it. This is worth raising
    with the maintainers as a question, not a fix.

### Branch C: `lane/hkdf` (fork extension)

- `algorithms/HKDF/{Bibliography.md, Main.cry, tests/RFC5869.cry}`.
  `HKDF_Extract`, `HKDF_Expand` and `HKDF` take the HMAC as arguments,
  the same way `algorithms/PBKDF/Main.cry` does, so any
  `packaging::Hash` instance works (`SHA256::HMAC`, `SHA1::HMAC`, ...).
  The RFC's restriction `L <= 255 * HashLen` is part of the type.
- Tests: all 7 RFC 5869 Appendix A cases. Each one checks PRK
  (Extract), OKM from the RFC's PRK (Expand) and the full composition.
  There is also a random-testing property that shorter outputs are
  prefixes of longer ones. Hex literals were cross-checked against the
  RFC text (76 literals, 0 missing). As a sanity check, a PRK with its
  last bit flipped evaluates to `False`.
- Separate commit: a draft of the DevGuide section "Developing a new
  algorithm", based on the layout the existing specs already use.
  Offer it in the issue; the maintainers may prefer to write it.
- I did **not** add HKDF to `packaging/Hash.cry` (where `PBKDF2` lives).
  That changes a shared module, so ask the maintainers first.

## 3. Evidence

Toolchain: native Cryptol **3.6.0** (latest GitHub release, the same
version as the `ghcr.io/galoisinc/cryptol:latest` image that CI's
Dockerfiles copy from), `ubuntu-24.04-X64-with-solvers`, in WSL. I
didn't use Docker because the image pull never finished under memory
pressure. Command, once per module:
`CRYPTOLPATH=<repo> cryptol +RTS -M1800m -RTS -b <(:module M; :check)`.
Every run also prints an FFI warning that `Keccak/StepMappings.so` is
"file too short". This is expected: CI builds that `.so`, and the
checked-in one is a placeholder, so Cryptol uses its fallback
implementation.

Branch A (trimmed; each "Passed" line is a separate property):

```
== algorithms::cSHAKE::tests::cSHAKE128      Sample1-2 Passed 1 tests. Q.E.D.   [31.84s]
== algorithms::cSHAKE::tests::cSHAKE256      Sample3-4 Passed ... Q.E.D.        [32.91s]
== algorithms::KMAC::tests::KMAC128          3 x Passed 1 tests.                [48.01s]
== algorithms::KMAC::tests::KMAC256          3 x Passed 1 tests.                [42.24s]
== algorithms::KMAC::tests::KMACXOF128       3 x Passed 1 tests.                [26.56s]
== algorithms::KMAC::tests::KMACXOF256       3 x Passed 1 tests.                [21.81s]
== algorithms::TupleHash::tests::TupleHash128     3 x Passed 1 tests.           [16.60s]
== algorithms::TupleHash::tests::TupleHash256     3 x Passed
== algorithms::TupleHash::tests::TupleHashXOF128  3 x Passed
== algorithms::TupleHash::tests::TupleHashXOF256  3 x Passed                    [12.62s]
== algorithms::ParallelHash::tests::ParallelHash128     Sample1-3 Passed. Q.E.D. [23.76s, maxrss 206936KB]
== algorithms::ParallelHash::tests::ParallelHash256     Sample1-3 Passed. Q.E.D. [24.66s]
== algorithms::ParallelHash::tests::ParallelHashXOF128  Sample1-3 Passed. Q.E.D. [19.99s]
== algorithms::ParallelHash::tests::ParallelHashXOF256  Sample1-3 Passed. Q.E.D. [22.55s]
```
40/40 pass. Before the module-name fix, ParallelHash128 failed with the
"File name does not match module name" error quoted above.

Branch B:

```
== algorithms::SHA1::tests::ShortMsgByte      65 x "Testing... Passed 1 tests."   [6.35s, maxrss 121452KB]
== algorithms::SHA1::tests::Examples OneBlock_check TwoBlock_check  2 x Passed  [1.49s]
== algorithms::PBKDF::tests::HMAC_SHA224
property tc20 Using exhaustive testing. Testing... Passed 1 tests. Q.E.D.
property tc33 Using exhaustive testing. Testing... Passed 1 tests. Q.E.D.
   [68.31s, maxrss 1469224KB]
-- excluded --
== SHA1 LongMessage_check: cryptol: Heap exhausted; ... (1800 MB)  [17.50s]
```

Branch C:

```
== algorithms::HKDF::tests::RFC5869
property TestCase1 .. TestCase7   7 x "Testing... Passed 1 tests. Q.E.D."
property ExpandIsPrefixClosed Using random testing.
Testing... Passed 100 tests.
   [19.27s, maxrss 144020KB]
== algorithms::HKDF::Main   (loads, no warnings; "There are no properties in this module.")
```

Not run: SAW (`theorems/*.saw`). The coordinator ruled it out for
memory reasons, and none of these branches add SAW files.

## 4. Competing work check

`gh issue list` / `gh pr list --state all` on upstream (2026-11-17):
issues #8 (SLH-DSA proposal) and #15 (CI unhelpful for external
partners); PRs #1-#14. None covers SP 800-185, SHA-1, PBKDF or HKDF
tests or specs. Open PRs #5 (SHA-2 FFI), #6 (security hardening) and
#9 (SLH-DSA) don't touch any file these branches change (checked with
`gh pr view --json files`).
Note on #15: CI jobs that push images fail for fork PRs, because
`GITHUB_TOKEN` can't push to ghcr. Expect red image-build checks on
these PRs that have nothing to do with the change. The issue already
tracks this.

## 5. Draft upstream text

### Issue (file first; then put its number in each commit subject)

Title: Known-answer tests missing for SP 800-185 family, SHA-1 and PBKDF2 (offering PRs; also HKDF if wanted)

> Several specifications have no `tests/` directory: cSHAKE, KMAC,
> TupleHash, ParallelHash, SHA1 and PBKDF. I'd like to contribute
> known-answer tests in the same style as `SHAKE/tests` and
> `SHA3/tests` (modules of `property` declarations, citing the vector
> source in `Bibliography.md`):
>
> 1. All 40 samples from NIST's SP 800-185 "Example Values" documents
>    (cSHAKE, KMAC, KMACXOF, TupleHash, TupleHashXOF, ParallelHash,
>    ParallelHashXOF). All of them pass `:check` on Cryptol 3.6.0.
>    While doing this I found that `algorithms/ParallelHash/Main.cry`
>    declares `module algorithms::TupleHash::Main`. Importing it fails
>    with "File name does not match module name", so the PR also fixes
>    that line. CI loads files by path, which is why this wasn't caught.
> 2. SHA-1: the two NIST example messages plus all 65 CAVP
>    `SHA1ShortMsg.rsp` vectors. PBKDF2: two ACVP PBKDF2-HMAC-SHA-224
>    vectors. The RFC 6070 / RFC 7914 vectors don't typecheck, because
>    their salts are shorter than the 128 bits that `PBKDF2`'s type
>    requires. Also, PBKDF2 gets expensive in the interpreter as C
>    grows: with HMAC-SHA-1 and a 25-byte output, C = 256 took about
>    35 s and 560 MB, and C = 4096 (RFC 6070 #5) exhausted a 1.8 GB
>    heap. Is that expected from the `iterate ... @ C` formulation,
>    or is there a better way to run it?
>
> I'd send each as its own small PR. If it's of interest, I also have a
> Cryptol HKDF (RFC 5869) spec in the repository layout. It's built on
> the existing HMAC instances, the same way PBKDF is, and passes all
> seven RFC 5869 Appendix A test cases. I also have a draft of the
> DevGuide's "Developing a new algorithm" section, based on the layout
> the existing specs follow. Happy to open either, or to leave them
> with you if you'd rather write them.

### PR #1 (branch `tests/sp800-185-kats`)

Title: `#NN Add NIST SP 800-185 example-value tests`

> Adds `tests/` modules for cSHAKE, KMAC, TupleHash and ParallelHash
> (including the XOF variants), covering all 40 samples in NIST's
> SP 800-185 Example Values documents, in the style of `SHAKE/tests`.
> The sources are cited in each `Bibliography.md`.
>
> It also fixes the module header of `algorithms/ParallelHash/Main.cry`.
> The header said `algorithms::TupleHash::Main`, so the module could not
> be imported.
>
> Checked locally with Cryptol 3.6.0: `:module
> algorithms::<Alg>::tests::<Fn>` then `:check`. 40/40 properties
> pass.
>
> 🤖 Generated with [Claude Code](https://claude.com/claude-code)

### PR #2 (branch `tests/sha1-pbkdf2-kats`)

Title: `#NN Add SHA-1 and PBKDF2 known-answer tests`

> Adds `SHA1/tests/Examples.cry` (the NIST SHA-1 example messages),
> `SHA1/tests/ShortMsgByte.cry` (all 65 CAVP SHA1ShortMsg vectors) and
> `PBKDF/tests/HMAC_SHA224.cry` (two ACVP PBKDF vectors, C = 1 and
> C = 183). All pass `:check` on Cryptol 3.6.0.
>
> The RFC 6070 / RFC 7914 PBKDF2 vectors are not included. Their salts
> are shorter than the 128 bits that `PBKDF2`'s type requires, and the
> one long-salt vector (C = 4096) exhausted a 1.8 GB heap in the
> interpreter. So did the FIPS 180-2 "one million a" SHA-1 example.
>
> 🤖 Generated with [Claude Code](https://claude.com/claude-code)

### PR #3 (only if the issue reply invites it; cherry-pick `027b706` onto upstream/main, without this report)

Title: `#NN Add HKDF (RFC 5869) specification`

> Adds `algorithms/HKDF` (Bibliography, `Main.cry`, `tests/RFC5869.cry`).
> `HKDF_Extract`, `HKDF_Expand` and `HKDF` take the HMAC as arguments,
> the same way `algorithms/PBKDF/Main.cry` does, so any
> `packaging::Hash` instance can be used. `L <= 255 * HashLen` is
> enforced in the type. All 7 RFC 5869 Appendix A test cases pass
> (checking Extract, Expand and the composition for each), plus a
> prefix-consistency property. Checked with Cryptol 3.6.0.
>
> 🤖 Generated with [Claude Code](https://claude.com/claude-code)

## 6. Land probability and next steps

- **A: High.** Tests-only, in the existing convention, all from
  primary NIST sources, plus a one-line real bug fix. The maintainer
  merged four small outside PRs the same day last week.
- **B: High.** Same reasoning. The PBKDF2 performance and salt-length
  question may start a discussion, but it doesn't block merging.
- **C: Med.** A new spec is a bigger ask, and CONTRIBUTING prefers
  dialogue first. They may also want HKDF wired into `packaging::Hash`
  or written differently.

Owner next steps:
1. File the issue above. Replace `#NN` in the A and B commit subjects
   with its number: `git commit --amend`, then `git push -f origin
   <branch>`.
2. Open PR #1 from `tests/sp800-185-kats` and PR #2 from
   `tests/sha1-pbkdf2-kats`.
3. Only if the maintainers reply positively: open PR #3 from a fresh
   branch with just the HKDF commit cherry-picked, and offer the
   DevGuide draft in the issue thread.
4. Commit trailers: the original work was done by Claude Opus 5.5
   and its trailer is kept. Commits amended during review carry an
   additional `Co-Authored-By: Claude Fable 5.1` trailer, which is
   the line LANE-RULES asks for. `tests/sha1-pbkdf2-kats` was not
   amended, so it still ends with the Opus trailer; add the Fable
   line when you amend it for the issue number if you want the exact
   LANE-RULES text.

## 7. Review notes (2026-11-17, Fable 5.1)

Verified independently before submission; details in
`../REVIEW-foundation-dnf.md`.

- All 18 new test modules re-run with the image CI's Dockerfiles
  copy Cryptol from (`ghcr.io/galoisinc/cryptol:latest`, reports
  Cryptol 3.6.0): 40 + 65 + 2 + 2 + 8 properties, all pass. On an
  idle machine each SP 800-185 module takes 3-6 s, the PBKDF module
  30 s.
- Every hex field of every vector was re-extracted from the primary
  sources (the seven NIST example-value PDFs, CAVP `SHA1ShortMsg.rsp`,
  NIST's `SHA1.pdf`, ACVP `PBKDF-1.0/internalProjection.json`,
  RFC 5869) and compared with the `.cry` files: keys, data, N/S,
  tuple elements, B, L and full outputs. No differences. Note the
  ParallelHash PDFs restart sample numbering at 1 for the 256-bit
  strength, so `ParallelHash256.cry` correctly uses `Sample1`-`3`.
- The `ParallelHash/Main.cry` module-name bug reproduces on
  `upstream/main` exactly as described (`:load` by path succeeds, an
  `import algorithms::ParallelHash::Main` fails with "File name does
  not match module name"). Nothing else in the repo referenced the
  wrong name.
- Changes made during review: the header comments of the 14 SP 800-185
  test modules were rewrapped to the style guide's 70 columns
  (comments only, no code or data changed); the pseudocode in the
  `HKDF_Expand` docstring was fenced with triple backticks, which
  `:check-docstrings` executes as REPL input (3 failures), so it is
  now indented plain text (`:check-docstrings` passes); the DevGuide
  draft gained one sentence on that. The PBKDF2 performance sentence
  in the drafts above was replaced with measured numbers (C = 256:
  35 s / 560 MB; C = 4096: heap exhausted at 1.8 GB after 59 s) in
  place of the earlier "did not finish in 7 minutes".
