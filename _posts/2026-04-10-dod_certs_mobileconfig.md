---
layout: post
title: "Deploying DoD PKI Certificates to Managed Macs"
categories: [SCRIPTS]
tags: [pki, certificates, mobileconfig, mdm]
---

# Deploying DoD PKI Certificates to Managed Macs

If you manage Macs in a DoD or DoD-adjacent environment, deploying the Department of Defense PKI certificate chain is a common requirement. Without the DoD root and intermediate certificates installed, users will encounter certificate errors when accessing DoD websites and services — CAC-authenticated portals, internal apps, and anything signed with a DoD-issued certificate.

The traditional approach is to manually download the certificate bundle from the [DoD PKE library](https://public.cyber.mil/pki-pke/), extract the individual certs, and import them. This works but doesn't scale well and doesn't stay current as the DoD periodically updates its certificate chain.

A better approach for managed environments is to deploy the certificates as a `.mobileconfig` profile via MDM. The profile can be scoped, enforced, and updated without any user interaction. The challenge has been generating that profile — until now.

## The Script

[`dod_certs_to_mobileconfig.py`](https://github.com/brodjieski/macos/tree/main/dod_certs_mobileconfig) is a Python script that automates the entire process:

1. Connects to the DoD PKE library and downloads the latest PKI certificate bundle
2. Extracts and converts the P7B certificate files from DER to PEM format using `openssl`
3. Splits the bundle into individual certificates
4. Determines if each certificate is a root CA or intermediate CA
5. Generates a fully formed `.mobileconfig` profile containing all certificates, ready for MDM deployment

## Prerequisites

The script requires:

- **Python** — available via Xcode Command Line Tools or [python.org](https://www.python.org)

You can verify that python is present:

```shell
python3 --version
```
{: .nolineno }

## Basic Usage

Clone or download the script from [GitHub](https://github.com/brodjieski/macos/tree/main/dod_certs_mobileconfig), then run it from your terminal:

```shell
python3 dod_certs_to_mobileconfig.py
```
{: .nolineno }

With no options, the script downloads the latest DoD certificate bundle, processes it, and writes a `.mobileconfig` file to the current working directory. The output filename is derived from the DoD certificate bundle name.

## Options

```
-h, --help                  Show help and exit
-r, --removal-allowed       Allow the profile to be removed by the user
--organization=NAME         Set a display name for the deploying organization
-o PATH, --output=PATH      Write the profile to a specific path
-e, --export-certs          Also save individual certificate files to a local folder
```

### Specifying an organization name

Setting an organization name gives the profile a meaningful display name in System Settings and in your MDM console:

```shell
python3 dod_certs_to_mobileconfig.py --organization "ACME Corp IT"
```
{: .nolineno }

### Specifying an output path

By default the profile lands in the current directory. Use `--output` to send it somewhere specific:

```shell
python3 dod_certs_to_mobileconfig.py --output ~/Desktop/dod_certs.mobileconfig
```
{: .nolineno }

### Exporting individual certificates

If you also need the individual certificate files (for other tooling or manual imports), pass the `--export-certs` flag:

```shell
python3 dod_certs_to_mobileconfig.py --export-certs
```
{: .nolineno }

This saves each certificate as a separate file alongside the profile.

### Allowing profile removal

By default the generated profile is marked as non-removable. If you want users or admins to be able to remove the profile manually (outside of MDM), add `--removal-allowed`:

```shell
python3 dod_certs_to_mobileconfig.py --removal-allowed
```
{: .nolineno }

>Profiles deployed via MDM can always be removed by the MDM admin regardless of this setting. This flag only affects whether a user can remove the profile themselves in System Settings.
{: .prompt-info }

## Deploying via MDM

Once you have the `.mobileconfig` file, deploy it like any other profile through your MDM. Since the script always pulls the latest bundle from the DoD PKE library, you can incorporate it into a regular workflow — for example, running it periodically and uploading the refreshed profile to your MDM when the DoD updates its certificate chain.

```shell
python3 dod_certs_to_mobileconfig.py \
  --organization "ACME Corp IT" \
  --output /path/to/mdm/profiles/dod_certs.mobileconfig
```
{: .nolineno }

## In Conclusion

Manually managing DoD certificate deployments is tedious and error-prone. This script takes a task that would otherwise require downloading, extracting, and manually building a profile into a single command that produces a deployment-ready `.mobileconfig`. Pair it with your MDM and you have a repeatable, current, and enforced certificate deployment.
