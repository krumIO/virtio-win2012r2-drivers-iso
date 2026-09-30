# Building the 2012 R2 VirtIO substitution ISO from scratch

Prepared for the Coriolis migration pilot (Harvester / SUSE Virtualization destination). 2026-09-30. Working notes.

## TL;DR

> This walks through producing `virtio-win-2012r2.iso` starting from an empty directory: download upstream virtio-win 0.1.189, extract it, copy nine driver components out of their per-component `2k12R2` directories into one flat directory matching VMDP's layout, unpack the guest agent from its MSI, and build an ISO9660 + Joliet image from the result. The finished tree is 43 files and the image is roughly 10 MB. Everything here is file copying and image building, with no compilation and no re-signing, because re-signing is precisely what is impossible.

---

## Why this is necessary

SUSE's Virtual Machine Driver Pack ships its Windows Server 2012 R2 drivers signed by `SUSE LLC` under a Sectigo chain. `winload` rejects them at boot with `0xc0000428`, because kernel-mode code signing on Windows versions before 10 requires a chain terminating at a Microsoft cross-certificate. SUSE's own download page states the distinction plainly: in the community package, the Windows 10 and Server 2016 and later drivers are signed by SUSE and certified by Microsoft, while all other drivers are signed by SUSE only.

Two separate signature checks apply to a Windows driver, and passing the first says nothing about the second:

- **Installation** (DISM, `pnputil`, or an MSI) validates the `.cat` against user-mode certificate stores. An untrusted publisher produces a prompt, not a failure, and installers commonly import their own certificate into TrustedPublisher first, so the step completes quietly either way. Coriolis compounds this by invoking `dism /add-driver` with `/forceunsigned`, which suppresses the check entirely. This is why manually mounting an ISO and running an installer appears to succeed.
- **Boot** applies kernel-mode code signing rules in `winload`, before the kernel exists. It consults none of the stores the installer populated and has no prompt to dismiss. This is the check that produces `0xc0000428`, and it is the only one that determines whether the machine starts on a virtio disk.

The two driver sets differ exactly there:

```
viostor.sys (virtio-win 0.1.189)
  CN=Red Hat, Inc.
    -> CN=Symantec Class 3 SHA256 Code Signing CA - G2
      -> CN=VeriSign Universal Root Certification Authority
        -> CN=Microsoft Code Verification Root

pvvxblk.sys (VMDP 2.5.5.1)
  CN=SUSE LLC
    -> CN=Sectigo Public Code Signing CA EV R36
      -> CN=Sectigo Public Code Signing Root R46
        -> CN=AAA Certificate Services
```

The upstream chain reaches `Microsoft Code Verification Root`, the cross-signing root. The VMDP chain terminates in a commercial CA that kernel-mode code signing does not accept for this Windows generation.

> [!NOTE]
> SUSE is not doing anything wrong. Microsoft [ended cross-signing on 1 July 2021](https://learn.microsoft.com/en-us/windows-hardware/drivers/install/deprecation-of-software-publisher-certificates-and-commercial-release-certificates) and the cross-certificates expired at the same time, so no new kernel-mode signature valid on 2012 R2 can be produced by anyone today. Attestation signing, the replacement, is valid only on Windows 10 and later, which is why VMDP's Server 2016 set is accepted and its 2012 R2 set is not. The 0.1.189 binaries work because they were signed in August 2020, inside the window.

This is also why the problem cannot be solved by building from source. [SUSE/vmdp](https://github.com/SUSE/vmdp) is BSD-2-Clause and compiles cleanly, but the resulting binaries could not be signed acceptably for 2012 R2. Only already-signed binaries from before the cutoff are usable, which leaves repackaging as the only route.

## What this accomplishes

The finished ISO carries a functionally equivalent driver set, component for component, that `winload` accepts, arranged in VMDP's directory layout so tooling expecting VMDP's structure can consume it without inheriting VMDP's signing problem. Nothing is renamed to VMDP's driver names, which is not possible (see the warning under Assembling the driver directory).

> [!WARNING]
> The VMDP layout this procedure produces is **not** consumable by Coriolis OS morphing. `_add_virtio_drivers` in [`coriolis/osmorphing/windows.py`](https://github.com/cloudbase/coriolis/blob/master/coriolis/osmorphing/windows.py) probes `<drive>:\Balloon\<osdir>\<arch>` to discover which OS directory to use, then builds every component path the same way. Against a flat layout that probe misses for every candidate and the method raises rather than degrading. An ISO intended for `windows_virtio_iso_url` must keep the upstream per-component structure instead, which means skipping the assembly step below and building the image directly from the extracted upstream tree.

## Prerequisites

| Platform | Needed for | Notes |
|---|---|---|
| Any | Downloading | ~477 MB over HTTP |
| macOS | Extract and build | `hdiutil`, built in. `dot_clean`, built in. |
| Linux | Extract and build | `xorriso` (or `genisoimage` / `mkisofs`), plus loop mount or `bsdtar` to read the source ISO |
| Windows | Extract and build | `Mount-DiskImage` in stock PowerShell; `oscdimg` from the Windows ADK Deployment Tools for the build step |
| Windows or msitools | Guest agent | `msiexec /a` on Windows, or `msiextract` from msitools elsewhere |

The guest agent is optional. `viostor` and `vioscsi` are the only components that affect boot, so a driver-only ISO is a complete answer to the `0xc0000428` problem on its own.

VMDP itself is not required. It is useful only as a reference layout, for verifying component coverage, or if you want its signed `qemu-ga.exe` in place of upstream's unsigned one, and it needs a [SUSE download](https://www.suse.com/download/suse-vmdp/) (the community build is available without a subscription; current installers need a SUSE Customer Center login).

## Step 1: download upstream virtio-win 0.1.189

0.1.189 is the last release carrying `2k12R2` directories, so the version is not a preference. It is dated 10 August 2020, which is what puts it inside the cross-signing window.

```bash
curl -LO https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.189-1/virtio-win.iso
```

The same directory also publishes `virtio-win-0.1.189.iso`, an identically sized copy under the versioned name, plus the guest tools MSIs and the `virtio-win-guest-tools.exe` bundle. Only the ISO is needed; it contains the MSIs.

## Step 2: extract the ISO

The goal is a directory named `virtio-win-0-1-189/` holding the ISO's contents.

**macOS:**

```bash
hdiutil attach virtio-win.iso -nobrowse -readonly -mountpoint /tmp/virtio-src
mkdir -p virtio-win-0-1-189
cp -R /tmp/virtio-src/. virtio-win-0-1-189/
hdiutil detach /tmp/virtio-src
```

**Linux:**

```bash
mkdir -p virtio-win-0-1-189
bsdtar -xf virtio-win.iso -C virtio-win-0-1-189
```

Or, with root, `mount -o loop,ro virtio-win.iso /mnt/virtio-src` and copy from there.

**Windows:**

```powershell
$img = Mount-DiskImage -ImagePath C:\virtio\virtio-win.iso -PassThru
$drive = ($img | Get-Volume).DriveLetter
New-Item -ItemType Directory -Force -Path C:\virtio\virtio-win-0-1-189 | Out-Null
Copy-Item "${drive}:\*" -Destination C:\virtio\virtio-win-0-1-189 -Recurse -Force
Dismount-DiskImage -ImagePath C:\virtio\virtio-win.iso
```

The result has one top-level directory per component (`viostor/`, `NetKVM/`, `Balloon/`, and so on), each subdivided by OS and architecture, plus `guest-agent/` holding the two `qemu-ga` MSIs.

## Step 3: know which components map to what

VMDP and virtio-win implement the same virtio device classes under different driver names. `pvvx` is SUSE's combined virtio and Xen driver family. This table is what the copy step below is executing:

| VMDP | virtio-win | 0.1.189 source path | Device class |
|---|---|---|---|
| `pvvxblk` | `viostor` | `viostor/2k12R2/amd64/` | virtio-blk storage. **Boot-critical.** |
| `pvvxscsi` | `vioscsi` | `vioscsi/2k12R2/amd64/` | virtio-scsi storage. **Boot-critical.** |
| `pvvxnet` | `netkvm` | `NetKVM/2k12R2/amd64/` | virtio-net network adapter |
| `pvvxbn` | `balloon` | `Balloon/2k12R2/amd64/` | memory balloon |
| `fwcfg` | `qemufwcfg` | `qemufwcfg/2k16/amd64/` | QEMU `fw_cfg` firmware configuration interface |
| `pvcrash_notify` | `pvpanic` | `pvpanic/2k12R2/amd64/` | guest crash notification to the host |
| `pvvxsvc.exe` | `blnsvr.exe` | `Balloon/2k12R2/amd64/` | user-mode service companion |
| `virtio_fs` | `viofs` | `viofs/2k12R2/amd64/` | virtio-fs shared filesystem |
| `virtio_rng` | `viorng` | `viorng/2k12R2/amd64/` | virtio-rng entropy source |
| `virtio_serial` | `vioser` | `vioserial/2k12R2/amd64/` | virtio-serial channel |

Two entries in that table are not straightforward substitutions:

- **`qemufwcfg` has no `2k12R2` directory at all.** The files come from `2k16/amd64/`, and that package is INF and CAT only with no `.sys`, which is upstream's own packaging rather than an incomplete copy. Untested on 2012 R2.
- **`pvvxbn` has no clean equivalent.** Its source indicates ballooning, but it registers in the `Boot Bus Extender` group and its directory also carries hypervisor detection code, suggesting it doubles as VMDP's paravirtual bus enumerator. Upstream has no counterpart: virtio devices are exposed as ordinary PCI devices claimed directly by their function drivers, with no intermediate bus node.

`README.md` covers the remaining mapping caveats and the confidence attached to each one.

Within the Windows 8 generation, upstream ships byte-identical binaries under `w8`, `w8.1`, `2k12` and `2k12R2`, so the choice of `2k12R2` is for clarity rather than because the others differ.

## Step 4: assemble the driver directory

Every component's files go into one flat directory, `merged/Server2012r2-Win8.1/x64/`, which is the structure VMDP uses. The only files excluded are the `.pdb` debug symbols and `NetKVM`'s `readme.doc`, neither of which is referenced by an INF or hashed by a catalog.

Four of the components ship an identical copy of `WdfCoInstaller01011.dll`, so the copy is done with a no-clobber flag and one copy serves all four.

**macOS or Linux**, run from the parent directory (`/Users/stephen/virtio`):

```bash
SRC=virtio-win-0-1-189
DEST=merged/Server2012r2-Win8.1/x64
mkdir -p "$DEST"
for c in viostor/2k12R2 vioscsi/2k12R2 NetKVM/2k12R2 Balloon/2k12R2 \
         pvpanic/2k12R2 viofs/2k12R2 viorng/2k12R2 vioserial/2k12R2 \
         qemufwcfg/2k16; do
  find "$SRC/$c/amd64" -maxdepth 1 -type f \
    ! -name '*.pdb' ! -name 'readme.doc' -exec cp -n {} "$DEST/" \;
done
ls -1 "$DEST" | wc -l   # expect 32
```

**Windows:**

```powershell
$Src  = "C:\virtio\virtio-win-0-1-189"
$Dest = "C:\virtio\merged\Server2012r2-Win8.1\x64"
New-Item -ItemType Directory -Force -Path $Dest | Out-Null
$Components = @(
    "viostor\2k12R2", "vioscsi\2k12R2", "NetKVM\2k12R2", "Balloon\2k12R2",
    "pvpanic\2k12R2", "viofs\2k12R2",   "viorng\2k12R2", "vioserial\2k12R2",
    "qemufwcfg\2k16"
)
foreach ($c in $Components) {
    Get-ChildItem (Join-Path $Src "$c\amd64") -File |
        Where-Object { $_.Extension -ne ".pdb" -and $_.Name -ne "readme.doc" } |
        ForEach-Object {
            $target = Join-Path $Dest $_.Name
            if (-not (Test-Path -LiteralPath $target)) { Copy-Item $_.FullName $target }
        }
}
(Get-ChildItem $Dest -File).Count   # expect 32
```

The 32 files are the INF, CAT and SYS triplets for the nine components (`qemufwcfg` contributing only two), plus six supporting files that are easy to overlook because upstream keeps them in the per-component directories rather than alongside the drivers:

| File | Needed by | Why |
|---|---|---|
| `WdfCoInstaller01011.dll` | `balloon`, `pvpanic`, `viorng`, `vioser` | KMDF co-installer, named in `SourceDisksFiles` and hashed by each catalog. A package missing it fails to install. |
| `viorngci.dll`, `viorngum.dll` | `viorng` | Named in `viorng.inf`. |
| `virtiofs.exe` | `viofs` | User-mode service. The driver installs without it but virtio-fs does not function. |
| `netkvmco.dll` | `netkvm` | Not referenced by the INF. Upstream ships it for adapter property configuration. |
| `blnsvr.exe` | `balloon` | The balloon service, and the counterpart to VMDP's `pvvxsvc.exe`. |

> [!NOTE]
> Neither boot-critical driver depends on any of these. `viostor` and `vioscsi` are plain SCSI miniports with no KMDF dependency and reference only their own `.sys`, so the storage path is complete as soon as those six files land, whatever happens with the rest.

> [!IMPORTANT]
> Signatures survive copying but not editing. The `.cat` hashes both the `.inf` and the binaries it covers, so whole directories may be relocated freely, while any modification to a file inside one invalidates the catalog and drops the package to unsigned. In particular the upstream files **cannot** be renamed to VMDP's driver names, which is why this procedure reproduces VMDP's directory structure but not its filenames.

## Step 5: unpack the guest agent

Upstream publishes no standalone executable installer. `guest-agent/` on the ISO contains only `qemu-ga-i386.msi` and `qemu-ga-x86_64.msi`, and the only `.exe` on the ISO is `virtio-win-guest-tools.exe`, a WiX Burn bundle covering the whole guest tools set.

Carrying the unpacked files rather than the MSI means the agent installs the same way the drivers do, by copying files and registering a service.

**Windows.** An administrative install writes the payload out without installing anything:

```
msiexec /a C:\virtio\virtio-win-0-1-189\guest-agent\qemu-ga-x86_64.msi TARGETDIR=C:\virtio\qga-extract /qn
```

The files land under `TARGETDIR` in the directory structure the MSI would have installed into, so locate them with a recursive listing rather than assuming a path:

```powershell
Get-ChildItem C:\virtio\qga-extract -Recurse -File | Select-Object FullName
```

**macOS or Linux**, using msitools (`brew install msitools`, or `dnf install msitools`):

```bash
msiextract -C qga-extract virtio-win-0-1-189/guest-agent/qemu-ga-x86_64.msi
```

Either way, 11 payload files come out. Split them into VMDP's two-folder convention:

```bash
mkdir -p merged/qemu-ga/x64 merged/qemu-ga/mingw64
```

| `merged/qemu-ga/x64/` | `merged/qemu-ga/mingw64/` |
|---|---|
| `qemu-ga.exe` | `libgcc_s_seh-1.dll` |
| `qga-vss.dll` | `libglib-2.0-0.dll` |
| `qga-vss.tlb` | `libintl-8.dll` |
| | `libssp-0.dll` |
| | `libwinpthread-1.dll` |
| | `iconv.dll` |
| | `gspawn-win64-helper.exe` |
| | `gspawn-win64-helper-console.exe` |

> [!NOTE]
> The x64 and mingw64 split is a distribution convention, not a runtime one. The MSI installs every file into a single directory (`%ProgramFiles%\Qemu-ga`), so a manual install on the guest should put them all in one place rather than reproducing the two folders on disk.

The two runtime sets are not identical to VMDP's. VMDP ships `libiconv.dll` where upstream ships `iconv.dll`, VMDP additionally carries `libpcre2-8-0.dll` and `libstdc++-6.dll` that upstream's build does not need, and VMDP has no counterpart to the two `gspawn` helpers. These reflect different mingw packaging, not a missing file on either side.

Neither `qemu-ga.exe` nor `qga-vss.dll` imports any of the mingw DLLs; both are statically linked, and the DLLs form a separate group (`libglib-2.0-0.dll` imports `libintl-8.dll`). The executable will therefore start without them. They are shipped because glib code paths can load helpers at runtime, so keeping them is the safe default.

> [!IMPORTANT]
> Upstream's agent binaries carry no Authenticode signature, and neither does the MSI. VMDP's `qemu-ga.exe` is signed by `SUSE LLC`. Nothing enforces this for a user-mode service, so the unsigned files install and run normally, but an environment with a policy requiring signed executables should use VMDP's agent instead. The kernel-mode signing constraint that rules out VMDP's drivers does not apply here, so mixing VMDP's agent with upstream's drivers is sound.

### Installing the agent on a guest

Replicating what the MSI does, from a directory holding all 11 files:

```
qemu-ga.exe -s vss-install
sc create QEMU-GA binPath= "C:\Program Files\Qemu-ga\qemu-ga.exe -d --retry-path" DisplayName= "QEMU Guest Agent" start= auto
sc start QEMU-GA
```

The service name, display name, start type and arguments above are the values the MSI itself writes. The VSS provider registration is a separate step because it is a custom action in the MSI rather than part of the service installation.

Where the MSI is available, it remains the simplest path, since it registers the service itself:

```
msiexec /i qemu-ga-x86_64.msi /qn /norestart
```

accepting exit code 0 or 3010.

## Step 6: build the ISO

At this point `merged/` holds 43 files: 32 drivers plus 11 agent files.

```bash
find merged -type f | wc -l   # expect 43
```

The image needs ISO9660 with Joliet extensions. Plain ISO9660 truncates names to 8.3 uppercase, which breaks the filename relationships the catalogs depend on, and a catalog that no longer matches its files is an unsigned package. Nothing here needs to be bootable, so no El Torito options are involved.

**macOS**, using the built-in `hdiutil`:

```bash
dot_clean -m merged && find merged -name '.DS_Store' -delete
hdiutil makehybrid -o virtio-win-2012r2.iso -iso -joliet -default-volume-name "VIRTIO-WIN" merged
```

`hdiutil` skips `.DS_Store` on its own, but AppleDouble sidecars (`._name`) written when copying from SMB or removable media are included, which is what `dot_clean` is clearing first.

**Linux**, using `xorriso` (or `genisoimage` / `mkisofs`, which take the same options):

```bash
xorriso -as mkisofs -J -joliet-long -V VIRTIO-WIN -o virtio-win-2012r2.iso merged
```

`-J` enables Joliet and `-joliet-long` lifts the 64-character path limit, which matters if the tree is ever restructured into the deeper per-component layout.

**Windows**, using `oscdimg` from the Windows ADK Deployment Tools:

```
oscdimg -j1 -m -lVIRTIO-WIN C:\virtio\merged C:\virtio\virtio-win-2012r2.iso
```

`-j1` writes Joliet names alongside ISO9660 ones, `-m` lifts the default size ceiling, and `-l` takes the volume label with no space after it. Without the ADK, the IMAPI2FS COM interface builds an image from stock PowerShell; `coriolis-worker-build.ps1` in the migration repo carries a working `New-IsoFile` implementation.

The result is roughly 10 MB.

## Step 7: verify the image

Confirm the file count matches the source tree and that names came through with their casing intact. A lowercase or truncated name means Joliet did not take, and the catalogs no longer match their files.

```bash
# macOS
hdiutil attach virtio-win-2012r2.iso -nobrowse -readonly
find /Volumes/VIRTIO-WIN -type f | wc -l
hdiutil detach /Volumes/VIRTIO-WIN
```

> [!TIP]
> If a previous mount is still attached, macOS mounts the new image as `/Volumes/VIRTIO-WIN 1` and a verification reading the fixed path reports zero files on a perfectly good image. Check with `ls -d /Volumes/VIRTIO-WIN*` before concluding anything, and detach any stragglers.

```bash
# Linux
xorriso -indev virtio-win-2012r2.iso -find
```

```powershell
# Windows
$img = Mount-DiskImage -ImagePath C:\virtio\virtio-win-2012r2.iso -PassThru
$drive = ($img | Get-Volume).DriveLetter
Get-ChildItem "${drive}:\" -Recurse -File | Select-Object FullName
Dismount-DiskImage -ImagePath C:\virtio\virtio-win-2012r2.iso
```

For a stronger check than a file count, compare checksums against the source tree. Every file on the image should be byte-identical to its upstream original, since nothing was modified:

```bash
diff <(cd merged && find . -type f | sort) <(cd /Volumes/VIRTIO-WIN && find . -type f | sort)
```

## What this deliberately does not produce

These are choices, not omissions, but they matter if anything expects a drop-in VMDP replacement:

- **No installer.** VMDP's `setup.exe`, `VMDP-WIN-*.exe`, `lang/`, `utilities/`, `v2v_scripts/` and `vd_agent/` have no upstream equivalent. Driver installation is by hand, by `pnputil`, or by `dism /add-driver /recurse`.
- **x86 is absent.** VMDP ships `Server2012r2-Win8.1/x86/` as well. Only x64 is assembled here; the same procedure against the `x86` source directories would produce it.
- **Other OS generations are absent.** VMDP ships `Server2012-Win8`, `Server2016-19-Win10` and `Server2022-25-Win11` trees. Only 2012 R2 has the signing problem this addresses, so only 2012 R2 is built.
- **No MSI on the image.** The agent is carried unpacked, so `qemu-ga-x86_64.msi` is not included. It remains available in the extracted upstream tree.

## Sources

- [virtio-win 0.1.189-1 archive directory, Fedora People](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.189-1/)
- [Deprecation of software publisher certificates and commercial release certificates, Microsoft Learn](https://learn.microsoft.com/en-us/windows-hardware/drivers/install/deprecation-of-software-publisher-certificates-and-commercial-release-certificates)
- [SUSE Linux Enterprise Virtual Machine Driver Pack download page, SUSE](https://www.suse.com/download/suse-vmdp/)
- [SUSE/vmdp, GitHub](https://github.com/SUSE/vmdp)
- [virtio-win/kvm-guest-drivers-windows, GitHub](https://github.com/virtio-win/kvm-guest-drivers-windows)
- [coriolis/osmorphing/windows.py, GitHub](https://github.com/cloudbase/coriolis/blob/master/coriolis/osmorphing/windows.py)
- [Oscdimg command-line options, Microsoft Learn](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/oscdimg-command-line-options)

## Confidence

**Confirmed by direct observation**

- The assembly loop in Step 4 reproduces the 32-file driver directory exactly, verified by SHA-256 against the existing `merged/Server2012r2-Win8.1/x64/` tree.
- The source file sets in each component's `2k12R2/amd64/` directory, and that the only files excluded by the loop are `.pdb` symbols and `NetKVM/readme.doc`.
- `qemufwcfg` in 0.1.189 contains only `.inf` and `.cat`, and has no `2k12R2` directory.
- 0.1.189 is the last upstream release carrying `2k12R2` directories, and its files are dated 10 August 2020.
- The signing chains quoted above, extracted from the PE certificate tables of `viostor.sys`, `vioscsi.sys` and `pvvxblk.sys`.
- VMDP's own `qemu-ga/` tree uses the `x64` and `mingw64` split this procedure reproduces.
- The MSI installs to `%ProgramFiles%\Qemu-ga` and registers service `QEMU-GA` ("QEMU Guest Agent") as `LocalSystem` with arguments `-d --retry-path`, and registers the VSS provider through a custom action running `qemu-ga.exe -s vss-install`.
- The agent's 11 payload files and their division between the two folders.
- Neither `qemu-ga.exe` nor `qga-vss.dll` imports any mingw DLL.
- Coriolis probes `Balloon\<osdir>\<arch>` and raises when no candidate directory exists.
- The upstream download URL in Step 1 resolves and serves a 477 MB image.

**Confirmed by vendor documentation**

- Cross-signing ended on 1 July 2021 and the cross-certificates expired, so no new kernel-mode signature valid on 2012 R2 can be produced. Attestation signing is valid only on Windows 10 and later.
- SUSE's download page states that only the Windows 10 and Server 2016 and later drivers in the community package are certified by Microsoft, with the rest signed by SUSE only.
- `oscdimg` syntax: `-j1` for Joliet alongside ISO9660, `-m` to lift the size ceiling, `-l<label>` with no space.

**Reasoned, not yet validated**

- The `msiextract` invocation in Step 5. msitools is not installed on the machine these notes were written on; the files currently in `merged/qemu-ga/` were produced by extracting the MSI's MSZIP cabinet directly and renaming to the File table's long names. The administrative-install route on Windows is the documented mechanism and is the recommended path.
- The component mapping for `fwcfg`, `pvcrash_notify`, `virtio_fs`, `virtio_rng` and `virtio_serial`, derived from matching source directories in the two projects rather than from testing.

**Needs live validation**

- Whether a migrated guest boots using the drivers in this ISO, as opposed to a guest prepared before migration.
- Whether the KubeVirt and Harvester plugin uses the open-core driver path logic, which determines whether the flat VMDP layout is usable at all.
- Whether `qemufwcfg` from the `2k16` package functions on 2012 R2.
