# Attestation Workflow

## Extended Attestation Workflow

![Extended attestation diagram](https://github.com/virtee/snpguest/blob/main/docs/extended.PNG?raw=true)

### Step 1. Request an attestation report

```bash
snpguest report $ATT_REPORT_PATH $REQUEST_FILE [-v, --vmpl] $VMPL [-r, --random]
```

Request an attestation report from the AMD Secure Processor (ASM) with the two mandatory parameters:

- `$ATT_REPORT_PATH`: path to which the report is stored
- `$REQUEST_FILE`: path to the request file

The request file is a 64-byte binary which will be embedded in the `REPORT_DATA` field of the attestation report.

The optional parameter `-v, --vmpl` specifies the vmpl level for the attestation report and is set to `1` by default.
It must be greater than or equal to the true VMPL value.

The flag `-r, --random` generates random data to be used as request data for the attestation report.

### Step 2. Request host-cached certificates

```bash
snpguest certificates $ENCODING $CERTS_DIR
```

Request the certificate chain from the extended memory, through the ASP.
This command requires the following parameters:

- `$ENCODING` which specifies whether to use PEM or DER encoding to store the certificates
- `$CERTS_DIR` which specifies the path in the user's directory where the certificates will be saved

In some environments, only the leaf certificate (VCEK/VLEK) can be obtained.
In this case, you must follow the procedure described in [Regular Attestation Workflow](#regular-attestation-workflow) to obtain the AMD CA certificates.

### Step 3. Verify the certificate chain

```bash
snpguest verify certs $CERTS_DIR
```

Verify the certificates saved in the directory `$CERTS_DIR` at Step 2.

### Step 4. Verify the attestation report

```bash
snpguest verify attestation $CERTS_DIR $ATT_REPORT_PATH [-t, --tcb] [-s, --signature]
```

Verify the attestation report by providing the following mandatory parameters:

- `$CERTS_DIR` which specifies the path in the user's directory where the certificates were saved from Step 2
- `$ATT_REPORT_PATH` which is the path pointing to the stored attestation report which the user wishes to verify.

The optional parameters `-t, --tcb` is used to verify only the Reported TCB contents of the attestaion report and `-s, --signature` is used to verify just the signature of the attestaion report.

## Regular Attestation Workflow

![Regular attestation diagram](https://github.com/virtee/snpguest/blob/main/docs/regular.PNG?raw=true)

### Step 1. Request an attestation report

```bash
snpguest report $ATT_REPORT_PATH $REQUEST_FILE [-v, --vmpl] $VMPL [-r, --random]
```

### Step 2-1. Request CA certificates from AMD KDS

```bash
snpguest fetch ca $ENCODING $CERTS_DIR $PROCESSOR_MODEL [-e, --endorser] $ENDORSER
```

Request the AMD CA certificates from the AMD Key Distribution Service (KDS) by providing the following mandatory parameters:

- `$ENCODING` which specifies whether to use PEM or DER encoding to store the certificates
- `$CERTS_DIR` which specifies the path in the user's directory where the certificates will be saved
- `$PROCESSOR_MODEL` - which specifies the AMD processor model for which the certificates are to be fetched

The optional `-e, --endorser` argument specifies the type of attestation signing key (defaults to VCEK).

This command yields the following certificates:

- AMD Root Key (ARK)
- AMD SEV Key (ASK) or AMD SEV-VLEK Key (ASVK), depending on the signing key type (VCEK or VLEK)

ARK is the root CA certificate, while ASK/ASVK are intermediate CA certificates issued for each CPU model.

### Step 2-2. Request leaf certificate from AMD KDS

```bash
snpguest fetch vcek $ENCODING $CERTS_DIR $ATT_REPORT_PATH [-p, --processor-model] $PROCESSOR_MODEL
```

Request the leaf certificate from AMD KDS with the following mandatory parameters:

- `$ENCODING` which specifies whether to use PEM or DER encoding to store the
  certificates
- `$CERTS_DIR` which specifies the path in the user's directory where the
  certificates will be saved
- `$ATT_REPORT_PATH` which is the path pointing to the stored attestation report
  for detecting Chip ID and Reported TCB Version.

The optional `-p, --processor-model` argument specifies the AMD processor model
for which the certificates are to be fetched. This option is mandatory when the
processor model cannot be determined by the report contents.

This yields the leaf certificate called the Versioned Chip Endorsement Key (VCEK)
certificate.
A VCEK is a certificate issued for each combination of CPU chip and TCB version.

Note that the AMD KDS does not provide an API for obtaining Versioned Loaded Key (VLEK) certificates, which are commonly used in cloud confidential computing platforms.
To obtain a VLEK, you must follow the instructions provided by the cloud provider.

### Step 3. Verify the certificate chain

```bash
snpguest verify certs $CERTS_DIR
```

### Step 4. Verify the attestation report

```bash
snpguest verify attestation $CERTS_DIR $ATT_REPORT_PATH [-t, --tcb] [-s, --signature]
```

## Attestation Flow for Azure Confidential VMs

AMD SEV-SNP provides Virtual Machine Privilege Levels (VMPLs) to separate components within a guest VM, with VMPL 0 being the most privileged level.
The architecture described in [Remote attestation of confidential VMs using ephemeral vTPMs](https://doi.org/10.1145/3627106.3627112) places a Secure VM Service Module (SVSM) with an ephemeral virtual Trusted Platform Module (vTPM) at VMPL 0 and the guest OS at VMPL 1. The SVSM requests attestation reports from the ASP on behalf of the guest.

Azure Confidential VMs based on AMD SEV-SNP use a similar separation: the [OpenHCL](https://github.com/microsoft/openvmm) paravisor, which hosts the vTPM, runs at VMPL 0, while the guest OS runs at VMPL 2.
The guest accesses attestation through the vTPM's non-volatile (NV) indices, with the paravisor mediating communication with the AMD-SP. Microsoft's [guest attestation design](https://github.com/MicrosoftDocs/azure-docs/blob/3ba4b2a3845f15f2136eb6d8af775e77d687cc54/articles/confidential-computing/guest-attestation-confidential-virtual-machines-design.md) defines the NV indices and report format used in this flow:

1. The guest writes up to 64 bytes of user data to the Report Data NV index, `0x01400002`, using `TPM2_NV_Write`.
2. The guest reads the Attestation Report NV index, `0x01400001`, using `TPM2_NV_Read`.
3. To generate the report, the paravisor constructs Runtime Claims as a JSON object containing the vTPM Attestation Key (AK) and Endorsement Key (EK) public keys, selected VM configuration, and the supplied user data.
   These appear in the `keys`, `vm-configuration`, and `user-data` fields, respectively; the user data is encoded as a hexadecimal string.
4. The paravisor hashes the serialized Runtime Claims and requests a SEV-SNP attestation report from the AMD-SP with that digest in the `REPORT_DATA` field.
   It packages the returned signed report with an Azure-defined header and Runtime Data containing the claims, and stores the result at NV index `0x01400001`.
5. The guest retrieves this packaged report through the NV read.
   The embedded SEV-SNP report can then be extracted for verification, while the Runtime Claims must be retained to check their binding to `REPORT_DATA`.

The user data is therefore bound to the signed report through the Runtime Claims
digest.
Verification must check the report's signature and certificate chain, recompute the digest over the original Runtime Claims bytes and compare it with `REPORT_DATA` using the report format's padding rules, and check that the `user-data` claim matches the expected input.

At present, `snpguest` only partially supports the OpenHCL-vTPM attestation flow.

### Step 0. Grant permission to access vTPM

```bash
sudo usermod -aG tss $USER
newgrp tss
```

Grant permission to access the TPM device file (`/dev/tpm0` or `/dev/tpmrm0`).

### Step 1. Request an attestation report

```bash
snpguest report $ATT_REPORT_PATH $REQUEST_FILE --openhcl
```

Request an attestation report from the paravisor. It requires the following parameters:

- `$ATT_REPORT_PATH`: path to which the report is stored
- `$REQUEST_FILE`: path to the request file

The request file is a 64-byte binary which will be embedded in the `user-data` field of the Runtime Claims JSON, not in the `REPORT_DATA` field of the attestation report.

Since the SEV-SNP report is requested by the paravisor at VMPL0 from ASP, the optional parameter `-v, --vmpl` is ignored.
The VMPL value in the retrieved report is always set to `0`.

**Note.** For the complete verification of the attestation report, the Runtime Claims JSON data is required in addition to the SEV-SNP report.
Currently, `snpguest` does not provide the functionality to retrieve the Runtime Claims JSON.

### Step 2. Request certificates from AMD KDS

```bash
snpguest fetch ca $ENCODING $CERTS_DIR $PROCESSOR_MODEL [-e, --endorser] $ENDORSER

snpguest fetch vcek $ENCODING $CERTS_DIR $ATT_REPORT_PATH [-p, --processor-model] $PROCESSOR_MODEL
```

**Note.** In an Azure SEV-SNP VM, the guest can retrieve the certificate chain cached on the host via the endpoint `/metadata/THIM/amd/certification` at the link-local address `169.254.169.254`.
Currently, `snpguest` does not provide the functionality to retrieve it via this route.

### Step 3. Verify the certificate chain

```bash
snpguest verify certs $CERTS_DIR
```

### Step 4. Verify the attestation report

```bash
snpguest verify attestation $CERTS_DIR $ATT_REPORT_PATH [-t, --tcb] [-s, --signature]
```
