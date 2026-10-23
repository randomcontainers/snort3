# snort3

Container images with [Snort 3](https://www.snort.org/), the network intrusion detection and prevention system, built from the release tag of [snort3/snort3](https://github.com/snort3/snort3) with LibDAQ compiled in. Snort reads packet capture files or live traffic, matches them against your rules and writes alerts as text, JSON or unified2. The images contain no rules. They are rebuilt when Snort publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by Cisco. Report problems with the image in this repository and Snort bugs at [snort3/snort3](https://github.com/snort3/snort3/issues).

## Quick start

Run the rules in `local.rules` against `capture.pcap` and print one line per alert:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/snort3 -q -c /usr/local/etc/snort/snort.lua \
  -R local.rules -r capture.pcap -A alert_fast
```

`-c` loads Snort's default configuration from the image, `-R` adds your rules, `-r` reads a capture file and `-A alert_fast` selects the alert format. `-q` leaves out the startup banner and the statistics at the end. A rule in `local.rules` looks like this:

```
alert http (msg:"Request for /admin"; http_uri; content:"/admin"; sid:1000001; rev:1;)
```

A few more commands:

```sh
# Check that the configuration and the rules load, without reading packets
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/snort3 -c /usr/local/etc/snort/snort.lua -R local.rules -T

# Write the alerts as JSON, one object per line, to alert_json.txt
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/snort3 -q -c /usr/local/etc/snort/snort.lua \
  -R local.rules -r capture.pcap -l . -A alert_json --lua 'alert_json = { file = true }'

# Read every .pcap file under the pcaps directory
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/snort3 -q -c /usr/local/etc/snort/snort.lua \
  -R local.rules --pcap-filter '*.pcap' --pcap-dir pcaps -A alert_fast
```

`--pcap-filter` applies only to the `--pcap-dir` and `--pcap-file` options that come after it.

The entrypoint runs `snort` under `tini` in `/work`, so file names are relative to the directory you mount. Running the image without arguments prints Snort's help overview. `--help-module <name>` describes a module and its parameters, and `--help-options <option>` a command-line option. The [Snort 3 documentation](https://docs.snort.org/) covers rule writing, and the user and reference manuals are attached to each [Snort release](https://github.com/snort3/snort3/releases).

## Rules and configuration

The image has no rules. Download the community, registered or subscriber rule sets for Snort 3 from [snort.org](https://www.snort.org/downloads), or write your own, and pass them with `-R`. `/usr/local/etc/snort/` holds Snort's default `snort.lua`, which includes `snort_defaults.lua`, and the tweak files `balanced.lua`, `connectivity.lua`, `max_detect.lua`, `security.lua`, `inline.lua` and `talos.lua`. Select a tweak file with `--tweaks`, for example `--tweaks security`.

To change the configuration itself, copy the directory out of the image, edit it and load your copy:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint cp \
  ghcr.io/randomcontainers/snort3 -r /usr/local/etc/snort config

docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/snort3 -q -c config/snort.lua -R local.rules -r capture.pcap -A alert_fast
```

`--lua '<chunk>'` changes a setting for one run without editing a file, as in the JSON example above.

## Packets as text

Snort's `hext` DAQ module reads packets written as text instead of a capture file, which is handy for testing a rule against a single request. A `$packet <src> <port> -> <dst> <port>` line gives the packet's addresses and ports, and the quoted lines that follow are the payload. The side with the higher port number is treated as the client:

```
$packet 192.168.1.1 49152 -> 10.1.2.3 80
"GET /admin HTTP/1.1\r\n"
"Host: example.org\r\n"
"\r\n"
```

The module is loaded from `/usr/local/lib/snort/daq`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/snort3 -q -c /usr/local/etc/snort/snort.lua \
  --daq-dir /usr/local/lib/snort/daq --daq hext -r request.hext -R local.rules -A alert_fast
```

The `hext` module handles IPv4 only. `doc/user/daq.txt` in the Snort sources describes the format. The `file` module in the same directory feeds plain files to Snort's file inspection.

## Other tools

- `snort2lua` converts a Snort 2 configuration and its rules to Snort 3: `--entrypoint snort2lua ... -c snort.conf -o snort.lua -r snort3.rules`. Content modifiers such as `http_uri` become sticky buffers.
- `-A unified2` writes `unified2.log.<timestamp>`, or `unified2.log` with `--nostamps`. `u2spewfoo` prints such a log as text, and `u2boat` writes the packets in it to a pcap file: `u2boat unified2.log.<timestamp> out.pcap`.

Run them with `--entrypoint`, as in the `snort2lua` example.

## Inspecting live traffic

Reading capture files works as any user. Opening a network interface needs the `NET_RAW` capability, which a container process only has when it runs as root, so live inspection needs `--user 0`. Docker gives root in a container `NET_RAW` by default and Podman does not; `--cap-add NET_RAW` works with both. `--network host` inspects the host's interfaces, and `--network container:<name>` those of another container:

```sh
docker run --rm --user 0 --cap-add NET_RAW --network host --ulimit core=0 \
  -v "$PWD:/work:ro" ghcr.io/randomcontainers/snort3 -c /usr/local/etc/snort/snort.lua \
  -R local.rules -i eth0 -A alert_fast
```

Stop it with Ctrl-C; Snort then prints its statistics. `--ulimit core=0` keeps a crash from leaving a core dump with captured traffic. Snort lowers only its hard limit for core files at startup. That fails while the soft limit is above 0, which is usually the case with Docker.

## Untrusted capture files

Snort parses traffic that may be hostile, and its decoders and inspectors have had security bugs. Reading a capture file needs no network and no capabilities. For files from unknown sources, drop both, turn off core dumps and mount the directory read-only, with the alerts going to standard output:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  --network none --cap-drop ALL --security-opt no-new-privileges --ulimit core=0 \
  ghcr.io/randomcontainers/snort3 -q -c /usr/local/etc/snort/snort.lua \
  -R local.rules -r untrusted.pcap -A alert_fast
```

## What is in the image

| Area | Details |
|---|---|
| Programs | `snort`, `snort2lua`, `u2spewfoo` and `u2boat` in `/usr/local/bin` |
| LibDAQ | Compiled into `snort` with its `afpacket`, `bpf`, `dump`, `fst`, `gwlb`, `pcap`, `savefile` and `trace` modules, so the image has no `libdaq.so` |
| DAQ modules | Snort's `file` and `hext` modules in `/usr/local/lib/snort/daq`, loaded with `--daq-dir` |
| Configuration | Snort's default `snort.lua`, `snort_defaults.lua`, the tweak files, `file_magic.rules` and `sensitive_data.rules` in `/usr/local/etc/snort` |
| Pattern matching | The Hyperscan API through Vectorscan, for the `hyperscan` search engine and the `regex` and `sd_pattern` rule options; PCRE2 for `pcre` |
| Other libraries | From the distro: LuaJIT, OpenSSL 3, libpcap, hwloc, libdnet, zlib, LZMA, Brotli (for `http_inspect`'s `decompress_brotli`) and libuuid, plus libtirpc on Alpine |

Not included: rules, SnortML (it needs LibML), Snort's interactive shell, the plugins from snort3_extra, the `nfq`, `netmap` and `divert` DAQ modules, jemalloc or tcmalloc, NUMA and libunwind support, `appid_detector_builder.sh` (a Bash script for writing AppId detectors), `show_flows` (it reads flow dumps that only the shell writes), and the headers, pkg-config file and documentation. `snort -V` prints the versions of the libraries, and the commit, the LibDAQ version and the CMake options are in `/usr/local/share/randomcontainers/snort3/buildinfo`.

## Default or slim

Snort's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a Snort release such as `3.12.4.0`. `<x.y.z>`, `<x.y>` and `<x>` are its shorter forms, `3.12.4`, `3.12` and `3`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<x.y.z>`, `<x.y.z>-slim`, `<x.y>`, `<x.y>-slim`, `<x>`, `<x>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<x.y.z>-ubuntu`, `<x.y.z>-slim-ubuntu`, `<x.y>-ubuntu`, `<x.y>-slim-ubuntu`, `<x>-ubuntu`, `<x>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<x.y.z>-alpine`, `<x.y.z>-slim-alpine`, `<x.y>-alpine`, `<x.y>-slim-alpine`, `<x>-alpine`, `<x>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Snort version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Snort writes log files such as `alert_json.txt` and `unified2.log.<timestamp>` to `/work` unless `-l` names another directory, and text alerts go to standard output unless a logger is set to write a file. Live inspection runs as root instead, see [Inspecting live traffic](#inspecting-live-traffic).

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Snort release and are rebuilt when the base image changes. The entrypoint is `["tini", "--", "snort"]`, so `CMD` holds Snort's options. This image carries its own rules and checks every `.pcap` file under the directory mounted at `/work`:

```dockerfile
FROM ghcr.io/randomcontainers/snort3:slim-alpine@sha256:...
COPY local.rules /usr/local/etc/snort/local.rules
CMD ["-q", "-c", "/usr/local/etc/snort/snort.lua", "-R", "/usr/local/etc/snort/local.rules", \
     "--pcap-filter", "*.pcap", "--pcap-dir", "/work", "-A", "alert_fast"]
```

The image runs as UID 1000, so switch to `USER root` to install distro packages, then back to `USER 1000:1000`. The packages Snort needs are listed in `/usr/local/share/randomcontainers/snort3/runtime-deps`. To pick up new Snort releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/snort3:latest \
  --repo randomcontainers/snort3 --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/snort3:latest --format '{{ json .SBOM }}'
```

Snort and LibDAQ are compiled here, so they are not in the SBOM and image scanners do not check them. Their versions and the Snort commit are in `/usr/local/share/randomcontainers/snort3/buildinfo`.

Snort attaches only its manuals to its releases and does not sign its tags, so each version is pinned to the commit of its tag. The build clones the `<version>` tag of [snort3/snort3](https://github.com/snort3/snort3) and stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. LibDAQ is GitHub's archive of its `v<version>` tag, pinned by SHA-256 in `package.yml` after its files were compared with the tagged commit. The build also stops unless the optional libraries listed above are linked and LibDAQ's modules are compiled in.

## Updates

The project checks the releases of [snort3/snort3](https://github.com/snort3/snort3/releases) every 15 minutes. A release is picked up once it is 24 hours old. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

LibDAQ is pinned to a version in `package.yml` and does not follow its upstream releases automatically. Snort releases often need the LibDAQ release published with them: the build of such a release stops with a message that names the LibDAQ version it needs, and the images stay on the previous Snort version until the pin is updated. The weekly and base-image rebuilds of those images stop as well until then, because they build the updated `package.yml`. Updating the pin is a commit to `package.yml`.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  --build-arg LIBDAQ_VERSION=<version> \
  --build-arg LIBDAQ_SHA256=<sha256> \
  -t snort3:local .
```

The `LIBDAQ_*` values are the `version` and `sha256` of the `libdaq` entry under `extra-artifacts` in `package.yml`. Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

Snort and LibDAQ are licensed under the GNU General Public License, version 2 only (GPL-2.0-only); their license files say that no later version applies. Code compiled in under other terms:

- LGPL-2.1-or-later: the `timersub` macro from the GNU C Library in Snort's `src/time/timersub.h`.
- BSD-3-Clause: LibDAQ's `daq_dlt.h`, taken from libpcap (Regents of the University of California).
- BSD-3-Clause-flex: flex's `FlexLexer.h` (Regents of the University of California), whose inline code Snort's JavaScript and PDF normalizers compile in.
- Public domain: the MurmurHash3 code in LibDAQ's `fst` module.

`/usr/local/share/randomcontainers/snort3/licenses/` has Snort's `LICENSE`, LibDAQ's (`LICENSE.libdaq`), the notices of those four (`LICENSE.timersub`, `LICENSE.libdaq-dlt`, `LICENSE.flexlexer`, `LICENSE.libdaq-murmurhash`) and `LGPL-2.1`, the text published by the Free Software Foundation, from [`licenses/`](licenses/) in this repository, since the sources do not include it. The image's license label is `GPL-2.0-only AND LGPL-2.1-or-later AND BSD-3-Clause AND BSD-3-Clause-flex AND LicenseRef-Public-Domain`.

Snort requires OpenSSL, and the image uses OpenSSL 3 from the distro, a separate package licensed under Apache-2.0. Snort's license has no OpenSSL exception, and Snort includes code from contributors other than Cisco. This project treats the distro's OpenSSL as a library that comes with the operating system, under the GPL's system-library exception. OpenSSL and the other Ubuntu and Alpine packages in the image keep their own licenses; the SBOM lists them.

The corresponding source for each image:

- Snort and LibDAQ: every version has a GitHub release in this repository, named `v<version>`, with `snort3-<version>.tar.gz`, an archive of the commit that was compiled, and the LibDAQ archive `v<libdaq version>.tar.gz`. When LibDAQ is updated, the release also keeps the archive of the earlier version. The build applies no patches. `/usr/local/share/randomcontainers/snort3/source` names the upstream commit and archive and links to that release.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure and CMake option.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

Snort is a trademark of Cisco and/or its affiliates. The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
