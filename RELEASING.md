# Releasing an Andamio dolos build

This fork tracks [`txpipe/dolos`](https://github.com/txpipe/dolos) and layers
an Andamio-specific commit stack on top. Releases are tagged
`vX.Y.Z-andamio.N` where `vX.Y.Z` is the upstream base release and `N` is
the Andamio iteration on top of it.

## When to cut a new release

- **New upstream release** (e.g. txpipe cuts `v1.0.4`) → rebase the stack,
  tag `v1.0.4-andamio.1`.
- **New Andamio change on the current base** (e.g. a fix on top of
  `v1.0.3-andamio.1`) → append commit, tag `v1.0.3-andamio.2`.
- **Hotfix on an older base** — branch from the old tag, add the fix, tag
  `vX.Y.Z-andamio.N+1` on that line. Rare.

## Flow

1. **Sync upstream.** Add `txpipe/dolos` as a remote if you haven't:
   ```
   git remote add upstream git@github.com:txpipe/dolos.git
   git fetch upstream --tags
   ```

2. **Rebase or append.**
   - Upstream bump: `git rebase <new-upstream-tag>` on top of `main`. Resolve
     conflicts; most andamio commits are in `src/serve/grpc/watch.rs` and
     `crates/minibf`, so conflicts are usually contained.
   - New change on same base: commit on top of `main` as usual.

3. **Verify.**
   ```
   cargo build --release
   ```
   Smoke test against a running node with `grpcurl` — at minimum stream
   `WatchTx` and confirm `asOutput` is populated on regular and reference
   inputs, since that is the most load-bearing Andamio change.

4. **Update `ANDAMIO_CHANGELOG.md`.** Add a new `## vX.Y.Z-andamio.N`
   section at the top listing the commits and what they do. Keep older
   sections intact.

5. **Push `main`.** Force-push is expected after an upstream rebase because
   commit SHAs change:
   ```
   git push --force-with-lease origin main
   ```

6. **Tag and release.**
   ```
   git tag -a vX.Y.Z-andamio.N -m "vX.Y.Z-andamio.N — see ANDAMIO_CHANGELOG.md"
   git push origin vX.Y.Z-andamio.N
   gh release create vX.Y.Z-andamio.N \
     --title "vX.Y.Z-andamio.N" \
     --notes-file <(sed -n "/^## vX.Y.Z-andamio.N/,/^## /p" ANDAMIO_CHANGELOG.md | sed '$d')
   ```

7. **Consumer bump.** Update
   [`andamio-dev-kit-internal`](https://github.com/Andamio-Platform/andamio-dev-kit-internal)
   `VERSIONS` → `DOLOS_TAG=vX.Y.Z-andamio.N` and open a PR. The
   `build-push-dolos` target in the Makefile builds from the tag, so no
   other Makefile edits are needed.

## Repo hygiene

Keep `Andamio-Platform/dolos` minimal: only `main` and the `andamio.N`
release tags. Don't let upstream branches or tags accumulate — they make
the release list and branch list unreadable. If you sync tags from
upstream by accident, delete them:

```
gh api repos/Andamio-Platform/dolos/tags --paginate --jq '.[].name' \
  | grep -v '^vX.Y.Z-andamio\.' \
  | xargs -I{} gh api -X DELETE repos/Andamio-Platform/dolos/git/refs/tags/{}
```

(Same pattern works for branches under `refs/heads/`.)
