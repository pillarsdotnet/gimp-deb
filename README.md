# gimp-deb

A patched rebuild of Ubuntu's `gimp`, published to the
[Forgejo Debian registry](https://forgejo.bobbalisa.org/pillarsdotnet/-/packages)
for **resolute** (26.04) and **stonking** (26.10).

Five patches fix three bugs that make GIMP 3.2 crash in ordinary use:

| Patch | Fixes |
| --- | --- |
| `ppa-0003` | Accept a `NULL` `GBytes` where empty was assumed |
| `ppa-0004` | Read an empty bytes parameter back as `NULL` (GIMP [#16821]) |
| `ppa-0005` | Assert on an empty transient handle |
| `ppa-0006` | Retry a parent window handle that is not ready yet (GIMP [#16825]) |
| `ppa-0007` | Stop `script-fu` crashing on a `script-fu-path` shorter than two folders |

[#16821]: https://gitlab.gnome.org/GNOME/gimp/-/issues/16821
[#16825]: https://gitlab.gnome.org/GNOME/gimp/-/issues/16825

`ppa-0007` was found because a `libgimp` test crashed GIMP on startup and
reported success anyway. That is why this build runs the test suite.

## What is in here, and what is not

This repository holds **only the delta**: the patches, and the CI that applies
them. There is no copy of gimp and no `debian/` directory.

```
patches/<series>/
    ppa-0003-...patch  ...  ppa-0007-...patch
    series             # applied in this order, after the distribution's own
    base               # the archive version they were last rebased against
.forgejo/
    workflows/registry.yml
    registry-upload    # the whole build
```

The base package comes from the archive at build time:

```
apt-get source gimp  ->  copy patches into debian/patches
                     ->  append to debian/patches/series
                     ->  write a changelog entry
                     ->  dpkg-buildpackage -b   (tests included)
                     ->  upload the debs to the registry
```

So every distribution security update is inherited without anyone editing
this repository — which matters for gimp, where 3.2.6-1 alone closed about two
dozen CVEs. A weekly cron run picks those up. The cost of the arrangement is
that an archive move can leave a patch no longer applying; `dpkg-source` then
fails the build loudly. See *Refreshing* below — that failure is also how you
find out a fix has landed upstream and the patch can be dropped.

**noble is not a target.** It carries gimp 2.10, and these are GIMP 3
patches. Adding a `patches/noble/` directory and a matrix entry would be the
whole change if that ever stops being true.

## Versions

```
<archive version>+ppa1+git<stamp>.<sha>~ubuntu<release>.1
```

for example `3.2.6-1+ppa1+git20261105132214.2251799~ubuntu26.10.1`. This sorts
above the archive's own version, so apt prefers it; above the
`+ppa1~ubuntu<release>.1` uploads of October 2026; and in archive order across
series, so a host upgrading resolute → stonking still moves forward.

The `+git` is a `+` and not a `~` for a reason that cost a build to find. With
`+ppa1~git<stamp>…`, the segment after `+ppa1~` compares `git…` against the
older uploads' `ubuntu…`, and `g` sorts before `u` — so every build here would
have ranked *below* the October upload it was meant to replace, and apt would
never have installed one. Check any change to the scheme with
`dpkg --compare-versions` against the archive version, the October stonking
upload `3.2.6-1+ppa1~ubuntu26.10.1`, and the October resolute upload
`3.2.2-1ubuntu0.26.04.1+ppa1`.

Because the string embeds both the archive version and the commit, "is this
version already in the registry?" is a complete answer to "is there anything
to build?" — a new archive upload changes it, and so does a new commit here.
A run with nothing to do exits in seconds, before the source download.

## Refreshing the patches after an archive update

When the build fails at `dpkg-source` with a patch that does not apply:

```sh
mkdir /tmp/gimp && cd /tmp/gimp
apt-get source gimp                       # the new base
cd gimp-*/
cp ~/src/forgejo/gimp-deb/patches/<series>/ppa-*.patch debian/patches/
cat ~/src/forgejo/gimp-deb/patches/<series>/series >>debian/patches/series
QUILT_PATCHES=debian/patches quilt push -a      # fix up what rejects
QUILT_PATCHES=debian/patches quilt refresh      # per patch that moved
```

Copy the refreshed patches back, update `patches/<series>/base` to the new
archive version, and commit. If a patch now applies as a no-op, the fix has
landed in the distribution: delete it from the directory and from `series`.

Three of the five patches were byte-identical between the 3.2.2 and 3.2.6
bases; `ppa-0005` and `ppa-0006` differed only in line offsets. Expect a
refresh, not a rewrite.

## Installing

The registry is already an apt source on the fleet, added by the `sources`
role in `kubuntu_bootstrap_ansible`. On a host that has it:

```sh
sudo apt update && sudo apt install gimp
```

## Building by hand

The CI script is self-contained and takes a series and a release. Run it in a
container of the target series, with a token that can write packages:

```sh
docker run --rm -it --shm-size=1g \
  --add-host forgejo.bobbalisa.org:10.0.0.2 \
  -v "$PWD:/src:ro" -w /src \
  -e REGISTRY_TOKEN=... ubuntu:26.10 \
  sh -c 'apt-get update -qq &&
         apt-get install -y -qq --no-install-recommends \
           git ca-certificates curl devscripts dpkg-dev python3 &&
         git config --global --add safe.directory /src &&
         .forgejo/registry-upload stonking 26.10'
```

The `--add-host` is needed when building on a LAN host: the host itself
resolves `forgejo.bobbalisa.org` to `10.0.0.2`, but a container asks public
DNS and gets the external address, which does not come back through NAT. The
script fails early and says so rather than building first and discovering the
problem at upload time. The CI runner is on homelab and does not need it.

Mounting the checkout read-only is deliberate — the build happens entirely in
a temporary directory, and the changelog entry is written there, so nothing
should ever be written back here.

Leave `REGISTRY_TOKEN` unset to build without uploading — the uploads will
report HTTP 401 and the script will exit non-zero, after the debs are built.

## CI

`.forgejo/workflows/registry.yml` builds both series in parallel on the
homelab runner, on every push to `main`, every Monday at 07:00 UTC, and on
demand. It needs the user-level Actions secret `REGISTRY_TOKEN`
(`write:package`), which this repository inherits.

The containers run with `--shm-size=1g`: gimp's test suite forks a full GIMP
under `xvfb`, and Docker's default 64 MB of `/dev/shm` is not enough for it.
