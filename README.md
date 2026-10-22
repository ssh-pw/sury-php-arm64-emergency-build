# Emergency build of Sury's PHP source packages for arm64 (Debian Bookworm)

> **No warranty — for information only.**
> This document describes how we rebuilt the PHP packages from [deb.sury.org](https://deb.sury.org/) for arm64 ourselves on 2026-10-21 during php2shell (CVE-2026-109618 / CVE-2026-109619), because the official arm64 binaries arrived days after the amd64 packages. It is a field report, not a recommendation and not a supported procedure. Use at your own risk. The resulting packages do not come from the Sury project, are not signed and are not maintained by anyone. Roll back to the official packages as soon as they are available (section G).
>
> If the Sury repository helps you: <https://github.com/sponsors/oerdnj>.

## Background

deb.sury.org builds amd64 first. The arm* builds run under qemu emulation and come later: according to the maintainer, roughly 5 h per arm64 combination, with about 110 combinations in total, so days rather than hours. The **source packages** (`.dsc` plus tarballs), however, are uploaded to the pool together with the amd64 binaries. On a native arm64 host, `sbuild` turns them into the same packages.

The version string of the self-built packages is identical to Sury's. Once Sury ships the arm64 binaries, apt sees the same version and reinstalls nothing. With the next release, Sury's packages take over on their own.

## Scope

- Everything runs as root; run long steps inside `screen`/`tmux`.
- Placeholder `<ver>` = PHP branch, e.g. `8.5`, `8.3`, `7.4`.
- Tested on arm64 with Debian 12 (Bookworm). For other releases, adjust paths and suite names.

**Prerequisites and limits**

- The `.dsc` must be in the Sury pool: `https://packages.sury.org/php/pool/main/p/php<ver>/`.
- Build host: arm64, Bookworm. With 16 cores, 8.5 took about 50 min and 7.4 about 18 min, both without the test suite. On shared vCPUs this is a short burst of full load; expect CPU/load alerts for that period.
- Build dependencies stay inside the chroot. Only `sbuild`, `schroot`, `debootstrap` and `devscripts` are installed on the build host itself. Disk space: about 3 GB for chroot plus build.
- PECL extensions (`php<ver>-yaml`, `php<ver>-redis`, …) are separate source packages. They are **not** rebuilt, stay untouched during the upgrade and keep working against the unchanged API.

## A. Prepare the build host (once, ~15 min)

```
apt-get install -y sbuild schroot debootstrap devscripts
```

```
sbuild-createchroot --include=ca-certificates,eatmydata bookworm /srv/chroot/bookworm-arm64-sbuild http://deb.debian.org/debian
```

**Pitfall:** Bookworm's sbuild has no `--extra-repository-key`. Copy the keyring and the sources line into the chroot by hand. On any host that already uses Sury, the keyring is at `/usr/share/keyrings/` (package `debsuryorg-archive-keyring`):

```
cp /usr/share/keyrings/debsuryorg-archive-keyring.gpg /srv/chroot/bookworm-arm64-sbuild/usr/share/keyrings/
```

```
echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ bookworm main" > /srv/chroot/bookworm-arm64-sbuild/etc/apt/sources.list.d/sury.list
```

Cross-check: `Candidate` must come from packages.sury.org.

```
schroot -c source:bookworm-arm64-sbuild -u root -- sh -c 'apt-get update -qq && apt-cache policy dh-php php-common | grep -E "^[a-z]|Candidate"'
```

## B. Fetch the source package

```
mkdir -p /srv/build && cd /srv/build && dget -u https://packages.sury.org/php/pool/main/p/php<ver>/<php<ver>_…debian12….dsc>
```

**Pitfall:** `dget` saves the `.dsc` with a URL-encoded `%2B`, but the tarballs with `+`. Rename them. Afterwards `.dsc`, `.orig.tar.xz` and `.debian.tar.xz` must sit next to each other:

```
cd /srv/build && for f in *%2B*; do mv "$f" "${f//%2B/+}"; done; ls -la
```

## C. Build

Without the test suite, leaving one core free. The variable only applies to the current shell. The terminal stays quiet during the build; the log goes to the `.build` file.

```
export DEB_BUILD_OPTIONS="nocheck parallel=$(( $(nproc) - 1 ))"
```

```
cd /srv/build && nice -n 15 sbuild -d bookworm --arch=arm64 --apt-update --no-run-lintian <php<ver>_…debian12….dsc> 2>&1 | tee build-php<ver>.log
```

Follow along:

```
tail -f /srv/build/php<ver>_*_arm64.build
```

Check that the test suite really is off:

```
grep -c -E "run-tests.php|^TEST " /srv/build/php<ver>_*_arm64.build; grep -m1 DEB_BUILD_OPTIONS /srv/build/php<ver>_*_arm64.build
```

The first number must be `0`, the second line must contain `nocheck`. The build is done when the summary block of the `.build` file shows `Status: successful`. Result: all binary packages of the source package as `.deb` files in `/srv/build`, including extensions and `-dbgsym`.

## D. Check the package set against Sury

The reference is Sury's amd64 set of the same version (on 2026-10-21: 63 packages for php8.5, 73 for php7.4). Count locally:

```
cd /srv/build && ls php<ver>*.deb libphp<ver>*.deb libapache2-mod-php<ver>*.deb 2>/dev/null | sed -E 's/_.*//' | sort -u | wc -l
```

Reference from the amd64 index, on any machine (adjust the version string):

```
curl -sS -o /tmp/Packages.amd64 https://packages.sury.org/php/dists/bookworm/main/binary-amd64/Packages
```

```
LC_ALL=C awk -v RS= -F'\n' '{p="";v=""; for(i=1;i<=NF;i++){ if($i ~ /^Package: /) p=substr($i,10); if($i ~ /^Version: /) v=substr($i,10)}; if (v ~ /^8\.5\.12-1\+0~20261021/) print p}' /tmp/Packages.amd64 | sort -u | wc -l
```

## E. Roll out on a host

To move the packages to another host, pack only the files you need into a tarball and compare the checksum on both ends:

```
cd /srv/build && tar czf /root/php<ver>-arm64-bookworm.tgz php<ver>*.deb libphp<ver>*.deb libapache2-mod-php<ver>*.deb && sha256sum /root/php<ver>-arm64-bookworm.tgz
```

On the target host, unpack and build a local index.

**Pitfall:** `dpkg-scanpackages` is part of `dpkg-dev`, which is missing on regular hosts. With `2>/dev/null` you silently get an empty `Packages.gz` (20 bytes), and apt does not see the repository. So install `dpkg-dev` and check size and package count:

```
apt-get install -y dpkg-dev && mkdir -p /srv/build && cd /srv/build && tar xzf /root/php<ver>-arm64-bookworm.tgz && dpkg-scanpackages . /dev/null 2>/dev/null | gzip -9 > Packages.gz && ls -la Packages.gz && zcat Packages.gz | grep -c "^Package:"
```

**Save a rollback set** **before** the upgrade, using the version currently available from Sury. The old version disappears from the pool as soon as Sury catches up.

**Pitfall:** `dpkg-query -W 'php<ver>*'` also lists virtual packages (`-calendar`, `-dom`, …) that are only `Provides` of `-common`/`-xml`/`-mysql`. Hence the filter on status `ii`. PECL packages (here `yaml`) are excluded because they are not rebuilt; adjust the pattern to your own PECL extensions.

```
mkdir -p /root/php<ver>-rollback && cd /root/php<ver>-rollback && apt-get download $(dpkg-query -W -f='${binary:Package} ${db:Status-Abbrev}\n' 'php<ver>*' | awk '$2=="ii" && $1!~/yaml/ {print $1"=<old version>"}') && ls *.deb | wc -l
```

Save a baseline:

```
php<ver> -m | sort > /root/php<ver>-modules-before.txt; php<ver> -i | grep -E "Configure Command" > /root/php<ver>-configure-before.txt; wc -l /root/php<ver>-modules-before.txt
```

Add the local repository. `Candidate` must be the new version from `file:/srv/build`:

```
echo "deb [trusted=yes] file:/srv/build ./" > /etc/apt/sources.list.d/local-php-build.list && apt-get update -qq && apt-cache policy php<ver>-fpm | grep -E "Installed|Candidate"
```

Preview, then upgrade only the installed `php<ver>-*` packages (plus `php-common` if due). `--only-upgrade` skips extensions that are not installed. The many `Note, selecting …`/`Skipping …` lines are expected.

```
apt-get -s install --only-upgrade php-common 'php<ver>-*' | grep -E "^Inst"
```

```
apt-get install -y --only-upgrade -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold php-common 'php<ver>-*'
```

Verify:

```
php<ver> -m | sort | diff /root/php<ver>-modules-before.txt - && echo "modules identical"; php<ver> -i | grep "Configure Command" | diff /root/php<ver>-configure-before.txt - && echo "configure identical"; php-fpm<ver> -t; php-fpm<ver> -v | head -1; ls -l /proc/$(systemctl show -p MainPID --value php<ver>-fpm)/exe; systemctl is-active php<ver>-fpm
```

Expected:

- both lines say `identical`
- `test is successful`
- the new version with its build date (Sury sets `SOURCE_DATE_EPOCH` from the changelog, so the stamp is not the compile time)
- `exe` without `(deleted)`, i.e. the FPM master runs the new binary
- `active`

For 7.4, also read the changelog from the installed package. `apt-get changelog` fails for Sury packages with "Changelog unavailable":

```
zcat /usr/share/doc/php7.4-fpm/changelog.Debian.gz | grep -m1 -i "CVE-2026-1096"
```

Then smoke-test your own application.

## F. Rollback

```
cd /root/php<ver>-rollback && rm -f /etc/apt/sources.list.d/local-php-build.list && apt-get update -qq && apt-get install -y --allow-downgrades ./php<ver>-*.deb && php-fpm<ver> -v | head -1 && systemctl is-active php<ver>-fpm
```

Afterwards, check your own `php.ini` changes: dpkg may have replaced conffiles during the downgrade.

## G. Clean up once Sury's arm64 packages are in the index

On every host with the local repository:

```
rm /etc/apt/sources.list.d/local-php-build.list && apt-get update -qq && apt-cache policy php<ver>-fpm | grep -E "Installed|Candidate|packages.sury"
```

`Installed` and `Candidate` stay the same; Sury now shows the same version. Then clean up:

- remove `/srv/build`, `/root/php*-arm64-bookworm.tgz`, `/root/php*-rollback`
- uninstall `dpkg-dev`
- on the build host, also remove `/srv/chroot` and run `apt-get purge sbuild schroot debootstrap devscripts`
- `apt-mark unhold` anything you held for the emergency run

## Has Sury shipped yet?

Check the arm64 index directly, without logging in to a host (adjust the package list):

```
curl -sS -o /tmp/Packages.arm64 https://packages.sury.org/php/dists/bookworm/main/binary-arm64/Packages
```

```
for p in php8.5-fpm php8.3-fpm php7.4-fpm; do printf "%-12s " "$p"; grep -a -A12 "^Package: $p$" /tmp/Packages.arm64 | grep -a -m1 "^Version:"; done
```

## License

[CC0 1.0](LICENSE). Provided as is, without warranty of any kind.
