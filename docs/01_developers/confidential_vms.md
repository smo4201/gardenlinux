# Confidential VMs

gardenlinux can be run as a confidential virtual machine (CVM) on AMD SEV-SNP and Intel TDX
hosts.

## Prerequisites

The following prerequisites must be met for confidential guest support

### Intel
- Intel Xeon processor with TDX support (4th Gen Scalable or newer)
- QEMU 10.2.0 or higher
- OVMF / EDK2 edk2-stable202405 or higher
- Host Kernel 6.16 or higher
- Guest Linux Kernel 5.19 or higher
- [Intel TDX enabled](https://cc-enabling.trustedservices.intel.com/intel-tdx-enabling-guide/01/introduction/)
- Attestation:
  [Intel Quote Generation Service (QGS)](https://cc-enabling.trustedservices.intel.com/intel-tdx-enabling-guide/05/host_os_setup/)
  running on host

### AMD-only
- AMD EPYC processor with SNP support (Milan or newer)
- QEMU 9.1 or higher (Adds native KVM SEV-SNP support)
- OVMF / EDK2 edk2-stable202405 or higher
- Host Kernel 6.11 or higher
- Guest Kernel 6.11 or higher
- Host BIOS: [AMD SEV-SNP enabled](https://github.com/AMDESE/AMDSEV/)

## Build

Build gardenlinux with SNP or TDX support:
```sh
./build kvm_snp_dev-amd64
./build kvm_tdx_dev-amd64
```

## Run

Run gardenlinux cvms:
```sh
./bin/start-vm \
      --snp \
      --cvm-fw /path/to/OVMF.fd \
      .build/kvm_snp-amd64-*.qcow2


./bin/start-vm \
      --tdx \
      --cvm-fw /path/to/OVMF.tdx.fd \
      .build/kvm_tdx-amd64-*.raw
```

## Attestation

The `cmc` feature (added automatically by `_snp` and `_tdx`) installs the
`cmcd` attestation daemon and the `cmcctl` attestation client.

### Host Setup

One-time host setup builds OVMF and QEMU and generates a demo PKI.

```sh
# Build QEMU + OVMF firmware and fetch the CMC tools
./hack/attestation-setup

# Generate the dedicated CMC attestation PKI
./hack/attestation-generate-pki

# Start the EST server (serves metadata + enrollment on :9001)
./hack/attestation-run-estserver &
```

### TDX Setup

```sh
# Build the Garden Linux TDX image (the _tdx feature pulls in the cmc feature).
./build kvm_tdx_dev-amd64

# Generate + sign attestation metadata over the TDX firmware. --mac/--ramsize must match the
# start-vm invocation below, because both are measured into RTMR0.
OVMF="$PWD/thirdparty/edk2/Build/OvmfX64/RELEASE_GCC5/FV/OVMF.fd"
./hack/attestation-generate-metadata kvm_dev_tdx-amd64 --ovmf "$OVMF" \
  --mac 02:00:00:ca:fe:01 --ramsize 2Gi

# Boot the confidential VM.
PATH="$PWD/thirdparty/qemu/build:$PATH" \
  ./bin/start-vm \
  --tdx \
  --cvm-fw "$OVMF" \
  --mac 02:00:00:ca:fe:01 \
  --mem 2Gi \
  .build/kvm_dev_tdx-amd64-*.raw
```

### SNP Setup

```sh
# Build the Garden Linux SNP image (the _snp feature pulls in the cmc feature).
./build kvm_snp_dev-amd64

# Generate + sign attestation metadata over the SNP firmware
OVMF="$PWD/thirdparty/edk2/Build/OvmfX64/RELEASE_GCC5/FV/OVMF.fd"
./hack/attestation-generate-metadata kvm_snp_dev-amd64 --ovmf "$OVMF"

# Boot the confidential VM.
PATH="$PWD/thirdparty/qemu/build:$PATH" \
  ./bin/start-vm \
  --snp \
  --cvm-fw "$OVMF" \
  .build/kvm_dev_snp-amd64-*.raw
```

### Starting Confidential guests

`hack/start-confidential-vm` is a convenience script for starting confidential guests with the
latest image. It detects whether the platform supports TDX or SNP and launches a confidential VM.
Supports all of the `bin/start-vm` parameters for customization.
```sh
./hack/start-confidential-vm
```

### Perform Attestation

Run inside the guest:

```sh
# cmcd already runs as a service
cmcctl generate -config /etc/cmcctl-conf.json   # attest  -> /var/attestation-report
cmcctl verify   -config /etc/cmcctl-conf.json   # verify  -> /var/attestation-result.json
```

`cmcd` caches the metadata it fetched from the EST server, so after regenerating reference values
restart it (`systemctl restart cmcd`) before attesting again.

### Reference Values

`attestation-generate-metadata` writes two manifests. The RTM manifest covers what the platform
and firmware measure, the OS manifest what the boot chain measures:

| Register | Events                                                 | Derived from                                                           |
| -------- | ------------------------------------------------------ | ---------------------------------------------------------------------- |
| `MRTD`   | initial TD memory                                      | `--ovmf`                                                               |
| `MRSEAM` | TDX module                                             | list of accepted module hashes in the script, or the `MRSEAM` override |
| `RTMR0`  | TD HOB                                                 | `--ramsize`                                                            |
|          | secure boot variables, firmware blob                   | `--ovmf`                                                               |
|          | `etc/table-loader`, `etc/acpi/rsdp`, `etc/acpi/tables` | `config/cmc/fw-cfg/`                                                   |
|          | `BootOrder`, `Boot####`                                | `config/cmc/efivars/`, MAC repinned from `--mac`                       |
| `RTMR1`  | GPT of the boot disk                                   | `.build/<cname>-*.raw`                                                 |
|          | systemd-boot, kernel                                   | `efi/EFI/BOOT/BOOTX64.EFI`, `efi/Default/*/linux`                      |
| `RTMR2`  | kernel command line (twice), initrd                    | `efi/loader/entries/*.conf`, `efi/Default/*/initrd`                    |

Everything that depends on the *image* is read out of the build artifacts, so reference values
track rebuilds automatically. Two classes of input cannot be:

- **The QEMU/OVMF configuration.** The ACPI blobs and the UEFI boot variables depend on the
  machine type and device topology, not on Garden Linux. `hack/attestation-dump-guest-refs` pulls
  them out of a running guest into `config/cmc/`; re-run it whenever the `start-vm` command line
  changes in a way that alters the virtual hardware. The NIC MAC is the one part that varies per
  run, and `--mac` rewrites it into the dumped template, so pinning the MAC is enough to keep the
  dump valid.
- **The TDX module.** `MRSEAM` is a property of the module the *host* loaded. The script currently
  carries the published hashes of every module from 1.5.34 on.
