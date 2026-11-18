# Azure Cloud Security Lab: CryptoCabana

Hands-on Microsoft Azure cloud security lab completed through TryHackMe.

[ [TryHackMe | CryptoCabana](https://tryhackme.com/room/hh-cryptocabana-f81cac95?utm_campaign=social_share&utm_medium=social&utm_content=share-completed-room&utm_source=copy&share_date=2026-11-18&sharerId=6aeb8235127e36e4ec8f6c56) ]

Completed · Microsoft Azure · Cloud Security · Medium

## Overview

This repository documents my approach to the **TryHackMe CryptoCabana** cloud security challenge.

The exercise provided an intentionally vulnerable Microsoft Azure environment and demonstrated how weaknesses in cloud access controls, identity permissions, and credential management can create a chain of unintended access across multiple Azure services.

Rather than focusing only on obtaining the challenge flag, I used the exercise to understand the relationship between **Azure Storage, Shared Access Signatures (SAS), service principals, Azure Key Vault, secret versioning, and least-privilege access**.

> [!NOTE]
> All credentials, tokens, tenant IDs, subscription IDs, resource names, secret values, version identifiers, and challenge flags have been removed or replaced with placeholders.

---

## Lab Environment

| | |
|---|---|
| **Platform** | TryHackMe |
| **Lab** | CryptoCabana |
| **Environment** | Microsoft Azure |
| **Category** | Cloud Security |
| **Difficulty** | Medium |
| **Primary Interface** | Azure CLI / Cloud Shell |

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
<img width="704" height="331" alt="image" src="https://github.com/user-attachments/assets/6b97ce45-394c-4ab7-bab7-f89c058aef16" />

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

I used Azure CLI to determine which storage containers were accessible.

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
  --container-name <CONTAINER>
