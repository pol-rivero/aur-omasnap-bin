# aur-omasnap-bin

Automation for the [`omasnap-bin`](https://aur.archlinux.org/packages/omasnap-bin) AUR package.

[omasnap](https://github.com/tobi/omasnap) is a native Wayland screenshot and
annotation editor for Omarchy and Hyprland. Upstream publishes a prebuilt Arch
Linux tarball with every tagged release, so this package installs that binary
directly instead of rebuilding it — hence the `-bin` suffix.

## How it works

`.github/workflows/cron.yaml` runs hourly (and on demand via
**Actions → Check Omasnap Release → Run workflow**):

1. Reads the latest upstream release tag from the GitHub API.
2. Compares it against `current_version.txt`.
3. If they differ, downloads `omasnap-<version>-archlinux-x86_64.tar.gz`,
   computes its `sha256sum`, and renders `PKGBUILD-template` into a `PKGBUILD`.
4. Pushes the `PKGBUILD` to the AUR.
5. Writes the new version back to `current_version.txt` and commits it.

`current_version.txt` is only updated *after* a successful AUR push, so a failed
run simply retries on the next hour. This matters because upstream creates the
release tag before its build workflow attaches the tarball; a run that lands in
that window fails on the download and retries later.

The `PKGBUILD` is generated, never committed — it is in `.gitignore`.

## Required repository secrets

| Secret | Value |
| --- | --- |
| `AUR_USERNAME` | AUR account username, used as the commit author |
| `AUR_EMAIL` | Email registered with the AUR account |
| `AUR_SSH_PRIVATE_KEY` | Private half of an SSH key registered in your AUR account |

## First publish

`current_version.txt` starts at `0.0.0`, which never matches a real upstream
tag. Once the secrets are set, trigger the workflow manually and it will create
the AUR package at the current latest release.

## Package notes

- `depends` mirrors the shared libraries the released binary actually links,
  as reported by `readelf -d usr/bin/omasnap` (`qt6-base`, `layer-shell-qt`,
  `wayland`, `libglvnd`, `libstdc++`, `libgcc`, `glibc`), plus the helpers it
  shells out to at runtime (`hyprctl` from `hyprland`, `wl-copy` from
  `wl-clipboard`).
- `libstdc++` / `libgcc` rather than `gcc-libs`: `gcc-libs` is now an empty
  metapackage that also pulls in the sanitizer, Fortran and Objective-C
  runtimes, none of which this binary links.
- No `qt6-wayland`: on current Arch, `qt6-base` ships the Wayland *client*
  stack (`libQt6WaylandClient.so.6`, the `libqwayland.so` QPA plugin, the
  shell-integration plugins). `qt6-wayland` is compositor-side only.
- `tesseract` / `tesseract-data-eng` are `optdepends`: they are only needed for
  OCR text extraction.
- `options=('!strip' '!debug')` leaves the released binary byte-for-byte intact.
- The upstream tarball unpacks as a `/usr` tree and does not include the project
  `LICENSE`, so it is fetched separately from the matching tag.

## Testing a build locally

```sh
sed -e 's/\[\[VERSION\]\]/1.20.1/g' -e 's/\[\[TARBALL_SHA256\]\]/SKIP/g' \
  PKGBUILD-template > PKGBUILD
makepkg -si
```

## License

The automation in this repository is MIT licensed (see `LICENSE`). omasnap
itself is MIT licensed by its upstream authors.
