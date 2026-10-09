# Intel P-State performance governor fix for power-profiles-daemon

A local Ubuntu package rebuild of **power-profiles-daemon 0.30-2** for one verified configuration where selecting the performance profile changed EPP to `performance` but left all CPUFreq policies on the `powersave` governor.

## Hardware and verified environment

- Laptop CPU: **Intel Core i7-1260P** (12th Gen, 16 logical CPUs)
- Distribution: **Ubuntu 26.04.1 LTS (Resolute Raccoon)**
- Kernel: **7.0.0-38-generic**
- Driver: `intel_pstate`, active mode
- Firmware platform profile: unavailable (`PlatformDriver: placeholder`)
- Patched package: `power-profiles-daemon 0.30-2xing1`, `amd64`

This is a machine-specific, experimental rebuild. It is not an Intel batch/stepping diagnosis, an official Ubuntu package, or a general performance guarantee. Other kernels, CPUs, firmware implementations, and PPD versions have not been validated.

## What changes

The bundled patch changes PPD's Intel P-State EPP path:

- On `performance`, set each EPP policy's `scaling_governor` to `performance` and skip an EPP write.
- On `balanced` and `power-saver`, first restore `powersave`, then write the EPP preference selected by PPD.
- No additional service, D-Bus monitor, kernel parameter, or thermal configuration is added.

This ordering avoids writing a non-performance EPP while the active `intel_pstate` performance algorithm owns EPP and may reject the write with `EBUSY`.

## Build and verification record

- Source base: Ubuntu `power-profiles-daemon 0.30-2` source package.
- Debian package version: `0.30-2xing1`.
- PPD test suite: **127 passed, 0 failed** in the recorded local build.
- On the machine above, a live `balanced → performance → balanced` round trip yielded 16 `performance` governors in performance and 16 `powersave` governors with `balance_performance` EPP on AC after returning to balanced. The PPD journal check found no matching `busy`, `failed`, `error`, or `warning` entries in the checked interval.
- No controlled benchmark was run; no throughput or sustained-frequency gain is claimed.

## Install

Download `power-profiles-daemon_0.30-2xing1_amd64.deb` from the GitHub Release and inspect the package before installing. Installation replaces the system PPD package and requires administrator authorization:

```sh
sha256sum -c SHA256SUMS
pkexec dpkg -i ./power-profiles-daemon_0.30-2xing1_amd64.deb
```

After installation, verify the profile and every policy:

```sh
powerprofilesctl set performance
powerprofilesctl get
for f in /sys/devices/system/cpu/cpufreq/policy*/scaling_governor; do cat "$f"; done | sort | uniq -c

powerprofilesctl set balanced
powerprofilesctl get
for f in /sys/devices/system/cpu/cpufreq/policy*/scaling_governor; do cat "$f"; done | sort | uniq -c
for f in /sys/devices/system/cpu/cpufreq/policy*/energy_performance_preference; do cat "$f"; done | sort | uniq -c
journalctl -u power-profiles-daemon --since '10 minutes ago' --no-pager | grep -Ei 'busy|failed|error|warning'
```

Expected on the verified AC setup: 16 `performance` governors in performance; 16 `powersave` governors and 16 `balance_performance` EPP values in balanced. No grep output means no matching entries in that interval only.

## Roll back

The release also contains the unmodified Ubuntu `0.30-2` package for rollback:

```sh
pkexec dpkg -i ./power-profiles-daemon_0.30-2_amd64.deb
powerprofilesctl set balanced
```

## Limits and risk

The performance governor selects the driver's performance algorithm; it does not pin the CPU to its advertised maximum clock or defeat firmware power limits, thermals, or throttling. This package changes privileged system power policy and is not distributed through Ubuntu's repositories. Review the patch and package metadata before use. Keep the original package or another recovery path available.

## Source and license

The patch is against the GPL-3-licensed PPD source; see `COPYING` and `NOTICE` for upstream attribution. This repository contains the source patch, a test/verification record, and binary Debian packages for convenient reproduction/rollback.
