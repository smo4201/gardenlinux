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
      --cvm-firmware /path/to/OVMF.fd \
      .build/kvm_snp-amd64-*.qcow2


./bin/start-vm \
      --tdx \
      --cvm-firmware /path/to/OVMF.tdx.fd \
      .build/kvm_tdx-amd64-*.raw
```
