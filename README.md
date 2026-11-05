# SBOM and Vulnerability Analysis of Container Base Images

How much security risk does a single dependency choice bring into a software supply chain? This project generates Software Bills of Materials (SBOMs) for two Node.js container images and scans them for known vulnerabilities, comparing an end-of-life image with a current one.

**Headline result:** switching from `node:16` to `node:22-alpine` reduced vulnerability findings from **2,046 to 26** (−98.7%), Critical findings from **44 to 0**, and actively exploited vulnerabilities from **3 to 0**.

## Why this matters

Regulations such as UNECE R155, ISO/SAE 21434 and the EU Cyber Resilience Act require manufacturers to identify and manage cybersecurity risk in components they receive from suppliers. An SBOM is the inventory of what a piece of software contains. A vulnerability scan of that SBOM turns the inventory into risk information that can be acted on.

A container image works like supplier-delivered software: it arrives as a finished package, and the receiver did not choose most of what is inside it.

## Method

| Tool | Purpose |
|---|---|
| [Syft](https://github.com/anchore/syft) | Generates the SBOM (CycloneDX and SPDX JSON) |
| [Grype](https://github.com/anchore/grype) | Matches SBOM components against known vulnerabilities |

Images analysed:

- `node:16`: Node.js 16 on Debian 10. Both are past end of life.
- `node:22-alpine`: a current Node.js LTS release on a minimal Alpine Linux base.

Commands used (Windows, PowerShell):

```powershell
syft registry:node:16 -o cyclonedx-json=node16.cdx.json
syft registry:node:22-alpine -o cyclonedx-json=node22-alpine.cdx.json
syft registry:node:22-alpine -o spdx-json=node22-alpine.spdx.json

grype sbom:node16.cdx.json -o table --file node16-vulns.txt
grype sbom:node22-alpine.cdx.json -o table --file node22-alpine-vulns.txt
grype sbom:node16.cdx.json --only-fixed -o table --file node16-fixable.txt
```

Scans were run in November 2026. Results reflect the vulnerability database on that date.

## Results

| | node:16 | node:22-alpine |
|---|---|---|
| Software packages in SBOM | 641 | 219 |
| Total findings | 2,046 | 26 |
| Unique vulnerabilities | 727 | 23 |
| Critical | 44 | 0 |
| High | 440 | 10 |
| Medium | 495 | 15 |
| Low | 118 | 1 |
| Negligible / Unknown | 949 | 0 |
| On CISA Known Exploited Vulnerabilities (KEV) list | 3 | 0 |
| Fix available | 706 | 23 |
| Marked "won't fix" | 351 | 0 |

A single vulnerability can appear several times when it affects several packages, so "total findings" is higher than "unique vulnerabilities". The CycloneDX files list 1,947 and 500 components in total, but these totals include individual file entries. The package counts above exclude them. For `node:22-alpine`, the CycloneDX and SPDX SBOMs agree once files are separated out: 219 packages and 281 files in both formats.

## Key findings

**1. Actively exploited vulnerabilities were found in seconds.** The `node:16` image contains three vulnerabilities on CISA's KEV list, meaning attackers are known to exploit them: CVE-2023-4863 (libwebp image parsing), CVE-2023-44487 (HTTP/2 "Rapid Reset", in both Node.js and nghttp2) and a Critical Node.js flaw (CVE-2026-111497). Without an SBOM, these would be invisible to anyone receiving the image.

**2. Most risk came from the base layer, not the application runtime.** In `node:16`, 1,607 of 2,046 findings (79%) were in Debian operating system packages. Bundled components such as Python 2.7/3.7, binutils, curl and OpenSSL produced more findings than Node.js itself. The choice of base image mattered more than the application.

**3. End-of-life components create permanent risk.** 351 findings in `node:16` are marked "won't fix" because Debian 10 no longer receives security updates. No amount of patching discipline fixes these. The only remedy is replacing the component.

**4. Fewer components means less attack surface.** `node:22-alpine` contains 219 software packages against 641 in `node:16` (−66%). Every component left out is one that cannot carry a vulnerability.

**5. Even a current image carries inherited risk.** 23 of the 26 findings in `node:22-alpine` are in the dependencies of the bundled npm package manager (for example `brace-expansion`, `ip-address`, `sigstore`, `pacote`), not in Node.js or the operating system. The remaining 3 are a single BusyBox vulnerability (CVE-2025-60876) with no fix available yet.

## What this means for supply chain assurance

- **Visibility first.** A receiver cannot assess risk in components it cannot see. SBOMs make transitive dependencies visible.
- **Track lifecycle, not just vulnerabilities.** End-of-life status predicted most of the risk here. Supplier requirements should cover support periods, not only current vulnerability counts.
- **Prioritise with exploit data.** Of 727 unique vulnerabilities in `node:16`, three were known to be exploited. KEV and EPSS scores help focus effort where it matters.
- **An SBOM is a snapshot.** New vulnerabilities are published daily, so an SBOM needs to be rescanned continuously, not checked once at delivery.
- **SBOMs must be trusted.** In a real supply chain, the receiver depends on the supplier's SBOM being complete and accurate. Signing and attestation are needed to verify that.

## Limitations

- Only two images were compared, so this is an illustration, not a statistical study.
- Grype matches by package name and version. Some findings may be false positives, for example where a vendor has backported a fix without changing the version number.
- Findings were not manually verified for exploitability in this specific context.
- Results depend on the scan date and vulnerability database version.

## Repository contents

| File | Description |
|---|---|
| `node16.cdx.json` | CycloneDX SBOM for `node:16` |
| `node22-alpine.cdx.json` | CycloneDX SBOM for `node:22-alpine` |
| `node22-alpine.spdx.json` | SPDX SBOM for `node:22-alpine` |
| `node16-vulns.txt` | Grype results for `node:16` |
| `node22-alpine-vulns.txt` | Grype results for `node:22-alpine` |
| `node16-fixable.txt` | `node:16` findings with an available fix |
