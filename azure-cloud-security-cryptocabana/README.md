# Azure Cloud Security Lab: CryptoCabana

Hands-on Microsoft Azure cloud security lab completed through TryHackMe.

[TryHackMe | CryptoCabana](https://tryhackme.com/room/hh-cryptocabana-f81cac95?utm_campaign=social_share&utm_medium=social&utm_content=share-completed-room&utm_source=copy&share_date=2026-11-18&sharerId=6aeb8235127e36e4ec8f6c56)

**Completed** · Microsoft Azure · Cloud Security · Medium

---

## Overview

This repository documents my approach to the **TryHackMe CryptoCabana** cloud security challenge.

The exercise provided an intentionally vulnerable Microsoft Azure environment and demonstrated how weaknesses in cloud access controls, identity permissions, and credential management can create a chain of unintended access across multiple Azure services.

Rather than focusing only on obtaining the challenge flag, I used the exercise to understand the relationship between **Azure Storage, Shared Access Signatures (SAS), service principals, Azure Key Vault, secret versioning, and least-privilege access**.

> [!NOTE]
> All credentials, tokens, tenant IDs, subscription IDs, resource names, secret values, version identifiers, usernames, and challenge flags have been removed or replaced with placeholders.

---

## Lab Environment

| Attribute | Details |
|---|---|
| **Platform** | TryHackMe |
| **Lab** | CryptoCabana |
| **Environment** | Microsoft Azure |
| **Category** | Cloud Security |
| **Difficulty** | Medium |
| **Primary Interface** | Azure CLI / Cloud Shell |
| **Status** | Completed |

### Technologies & Concepts

- Microsoft Azure
- Azure CLI
- Azure Cloud Shell
- Bash
- Azure Blob Storage
- Shared Access Signatures (SAS)
- Azure Service Principals
- Azure Key Vault
- Secret versioning
- Cloud Identity and Access Management (IAM)
- Least privilege
- Credential management

---

## Investigation Path

The central lesson of the lab was that a relatively small exposure can become significantly more serious when it provides a path to additional cloud resources.

My investigation followed this general sequence:

```text
Public Web Application
        │
        ▼
Client-side Configuration
        │
        ▼
Azure Storage Access
        │
        ▼
Storage Enumeration
        │
        ▼
Sensitive Artifact
        │
        ▼
Application Identity
        │
        ▼
Service Principal
        │
        ▼
Azure Key Vault
        │
        ▼
Secret Version Investigation
        │
        ▼
Challenge Completed
```

The important part was not any individual command. It was understanding **why access at one stage enabled the next stage**.

---

# Investigation

## 1. Client-Side Application Investigation

I began by examining resources exposed by the public-facing web application.

A JavaScript resource used by the application could be downloaded for inspection:

```bash
curl https://<LAB-APPLICATION>/app.js -o app.js
```

### Evidence

<img width="704" alt="Client-side application investigation" src="https://github.com/user-attachments/assets/6b97ce45-394c-4ab7-bab7-f89c058aef16">

*Figure 1. Initial inspection of the CryptoCabana application's client-side source. Lab-specific information has been redacted.*

The purpose was to determine what configuration or cloud-access information was exposed client-side and whether it pointed toward additional Azure resources.

### Security Lesson

Information delivered to the client should effectively be considered visible to the user.

If an application exposes cloud access mechanisms client-side, their permissions should therefore be designed under the assumption that they may be discovered.

This reinforces the importance of:

- limiting credential scope;
- limiting credential lifetime;
- separating public application functionality from sensitive resources; and
- applying least privilege to cloud access.

---

## 2. Azure Storage Enumeration

After identifying an Azure Storage access mechanism in the lab, I investigated the permissions associated with it.

I used Azure CLI to determine which storage containers were accessible:

```bash
az storage container list \
  --account-name <STORAGE-ACCOUNT> \
  --sas-token "<REDACTED-SAS-TOKEN>" \
  --query "[].name" \
  -o tsv
```

The command returned multiple accessible containers, demonstrating that the available storage permissions extended beyond a single application resource.

### Security Lesson

A **Shared Access Signature (SAS)** should be scoped according to the principle of least privilege.

The potential impact of an exposed SAS token depends on factors such as:

- permitted services;
- permitted resource types;
- allowed operations;
- expiration time;
- protocol restrictions; and
- sensitivity of accessible data.

If an application only requires access to a narrowly defined resource, the associated access mechanism should not unnecessarily expose additional storage resources.

---

## 3. Investigating Accessible Storage

I next examined the storage areas available through the lab credential.

A relevant artifact could be retrieved using Azure CLI:

```bash
az storage blob download \
  --container-name <CONTAINER> \
  --name <FILE> \
  --file /tmp/<FILE> \
  --account-name <STORAGE-ACCOUNT> \
  --sas-token "<REDACTED-SAS-TOKEN>"
```

The retrieved information revealed that the initial storage exposure could be used to continue the investigation into the Azure environment.

This was an important point in the exercise because the original storage access was no longer an isolated issue.

Instead, it became part of a broader **credential and trust chain**.

### Security Lesson

Sensitive authentication material should not be stored in locations accessible using credentials intended for unrelated application functionality.

A relatively minor storage exposure can become substantially more significant when the accessible data includes credentials associated with another security boundary.

---

## 4. Application Identity & Service Principal

The investigation subsequently exposed credentials associated with an Azure application identity.

Within the authorized lab environment, I authenticated using the discovered service principal:

```bash
az login --service-principal \
  -u "<CLIENT-ID>" \
  -p "<REDACTED-CLIENT-SECRET>" \
  --tenant "<TENANT-ID>"
```

I then verified the security context and investigated what resources this identity was permitted to access.

### Security Lesson

Service principals are **machine identities** and should be treated as security-sensitive identities.

A compromised service principal can become a pivot point when:

1. its credentials are accessible from another resource;
2. those credentials remain valid; and
3. the identity has permissions across additional Azure resources.

The exercise therefore demonstrated that cloud security cannot be evaluated purely at the individual-resource level.

The permissions and trust relationships connecting those resources also matter.

---

## 5. Azure Key Vault Investigation

The application identity provided access to secrets stored within Azure Key Vault.

I queried the relevant lab secrets using Azure CLI:

```bash
az keyvault secret show \
  --vault-name "<KEY-VAULT>" \
  --name "<SECRET-NAME>" \
  --query value \
  -o tsv
```

The challenge distributed the required information across multiple secrets.

However, one of the current secret values indicated that it had previously been **rotated**.

That meant retrieving only the current values was not sufficient to complete the investigation.

---

## 6. Secret Version Investigation

I therefore examined the available version history associated with the relevant secret:

```bash
az keyvault secret list-versions \
  --vault-name "<KEY-VAULT>" \
  --name "<SECRET-NAME>" \
  -o json
```

This revealed that multiple versions of the secret existed.

The relevant historical version could then be inspected within the authorized lab environment:

```bash
az keyvault secret show \
  --vault-name "<KEY-VAULT>" \
  --name "<SECRET-NAME>" \
  --version "<VERSION-ID>" \
  --query value \
  -o tsv
```

### Evidence

<img width="484" height="346" alt="image" src="https://github.com/user-attachments/assets/aa949b2f-69ac-4618-82f6-d335f30cce00" />

*Figure 2. Azure Cloud Shell session showing the investigation of a historical Key Vault secret version. Lab-specific identifiers, usernames, version IDs, and secret values have been redacted.*

The historical version contained the information required to complete the final stage of the investigation.

### Security Lesson

**Secret rotation and complete removal of historical secret material are not necessarily the same thing.**

Secret lifecycle management should consider:

- access to historical versions;
- identity permissions;
- credential rotation;
- credential revocation;
- retention requirements;
- unused or legacy credentials; and
- whether an application identity actually requires access to previous versions.

---

# Understanding the Security Chain

The most useful part of the exercise was understanding how several individual weaknesses could combine into a wider cloud-security issue.

```text
Client-side exposure
        ↓
Cloud access mechanism discovered
        ↓
Broader-than-required storage access
        ↓
Sensitive authentication material accessible
        ↓
Application identity compromised
        ↓
Additional Azure permissions inherited
        ↓
Key Vault accessible
        ↓
Historical secret accessible
```

None of these stages should be considered completely independently.

A cloud architecture can contain several controls that appear acceptable individually while still creating a problematic **combined trust path**.

---

# Key Security Takeaways

## 1. Least privilege needs to exist across the architecture

Restricting individual identities and credentials reduces the possibility of chaining several weaknesses together.

Least privilege should apply not only to users but also to:

- SAS tokens;
- application identities;
- service principals;
- storage permissions; and
- Key Vault permissions.

---

## 2. SAS permissions should be narrowly scoped

The impact of an exposed SAS token depends heavily on its permissions, scope, and lifetime.

Applications should receive only the storage permissions required for their intended functionality.

---

## 3. Application identities are security boundaries

Service principals should only have access to resources required by their workload.

Compromise of one application should not automatically provide a path toward unrelated cloud resources.

---

## 4. Credential storage matters as much as credential strength

A strong credential offers little protection if it is stored somewhere accessible through a weaker security boundary.

Secrets should therefore be protected according to the sensitivity of the systems they authorize access to.

---

## 5. Secret rotation is only part of secret lifecycle management

Replacing a secret does not necessarily remove every security consideration associated with the previous value.

Historical versions, existing permissions, credential revocation, and retention should also be evaluated.

---

## 6. Small weaknesses can create larger attack paths

The lab reinforced an important cloud-security principle:

> **Security weaknesses should be evaluated not only individually, but also in terms of how they can be chained across identities, permissions, and services.**

---

# Skills Practiced

| Area | Practical Experience |
|---|---|
| **Azure CLI** | Querying and interacting with Azure resources through Cloud Shell |
| **Azure Storage** | Investigating accessible containers and blobs |
| **SAS** | Understanding token-based storage access and permission scope |
| **Azure IAM** | Working with application identities and service-principal authentication |
| **Azure Key Vault** | Investigating secret access |
| **Secret Management** | Understanding rotation and historical secret versions |
| **Cloud Security** | Following trust relationships across Azure resources |
| **Security Analysis** | Connecting individual weaknesses into a broader access path |
| **Least Privilege** | Understanding why access should be limited at every stage |

---

# What I Learned

Before completing this exercise, it would have been easy to view Azure Storage, service principals, and Key Vault as largely separate security topics.

The lab helped me understand how they interact.

The key lesson was that **cloud security is heavily dependent on relationships between identities, credentials, permissions, and resources**.

A storage permission issue becomes much more serious if the accessible data contains credentials. Those credentials become more significant if the associated identity has access to another sensitive Azure service.

Understanding those relationships was more valuable than simply obtaining the challenge flag.

---

# Ethical & Authorization Statement

This repository documents work performed exclusively within the **authorized and intentionally vulnerable TryHackMe CryptoCabana training environment**.

No real-world third-party infrastructure was targeted or tested.

This repository intentionally does **not** disclose:

- SAS tokens;
- passwords;
- client secrets;
- temporary access credentials;
- usernames;
- tenant or subscription identifiers;
- lab-specific Azure resource identifiers;
- Key Vault secret values;
- secret version identifiers; or
- the TryHackMe challenge flag.

All sensitive and environment-specific values have been replaced with descriptive placeholders or visually redacted from screenshots.

The purpose of this repository is to document my practical development in **Azure cloud security, identity management, access control, credential security, and secure cloud architecture**.

---

# Further Development

This exercise forms part of my broader development in cloud and information security.

Areas I intend to explore further include:

- Microsoft Entra ID security
- Azure RBAC
- Managed identities
- Azure Key Vault security
- Identity attack paths
- Cloud logging and monitoring
- Microsoft Defender for Cloud
- Microsoft Sentinel
- Cloud incident detection
- Secure Azure architecture

---

## Disclaimer

This repository is an independent learning write-up based on an authorized TryHackMe lab.

TryHackMe and CryptoCabana are referenced solely to identify the training environment in which the exercise was completed. This repository is not affiliated with or endorsed by TryHackMe.

Challenge answers and sensitive lab-specific values have intentionally been omitted.
