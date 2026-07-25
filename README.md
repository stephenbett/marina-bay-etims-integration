# 🏢 Marina Bay eTIMS Integration Engine (KRA OSCU)

![.NET Core](https://img.shields.io/badge/.NET-8.0-512BD4?style=for-the-badge&logo=dotnet)
![Angular](https://img.shields.io/badge/Angular-17-DD0031?style=for-the-badge&logo=angular)
![C#](https://img.shields.io/badge/C%23-12.0-239120?style=for-the-badge&logo=csharp)
![SQL Server](https://img.shields.io/badge/SQL_Server-2022-CC292B?style=for-the-badge&logo=microsoftsqlserver)
![KRA eTIMS](https://img.shields.io/badge/Integration-KRA_eTIMS_OSCU-008080?style=for-the-badge)
![License](https://img.shields.io/badge/Code_Status-Private_Core_/_Public_Architecture-blue?style=for-the-badge)

An enterprise-grade, cloud-native **eTIMS (Electronic Tax Invoice Management System)** integration engine built for the **Marina Bay Tenant Management System (TMS)**. 

This repository documents the technical architecture, data model extensions, cryptographic communication workflow, and integration strategy used to connect a multi-tenant commercial real estate platform with the **Kenya Revenue Authority (KRA)** tax compliance REST APIs via the **Online Sales Control Unit (OSCU)** protocol.

---

## 📑 Table of Contents

- [Architectural Overview](#-architectural-overview)
- [Why OSCU over VSCU?](#-why-oscu-over-vscu)
- [Technology Stack](#-technology-stack)
- [System Architecture & Data Flow](#-system-architecture--data-flow)
- [Database Schema & ERD](#-database-schema--erd)
- [Core Integration Workflows](#-core-integration-workflows)
  - [1. Device Initialization (`cmcKey` Handshake)](#1-device-initialization-cmckey-handshake)
  - [2. Tax Class & Item Code Sync (`saveItem`)](#2-tax-class--item-code-sync-saveitem)
  - [3. Real-Time Transaction Transmission (`saveTrnsSalesOsdc`)](#3-real-time-transaction-transmission-savetrnssalesosdc)
- [Resilience & Emergency Mitigation Engine](#-resilience--emergency-mitigation-engine)
- [C# Integration Engine Blueprint](#-c-integration-engine-blueprint)
- [Key Features & Business Impact](#-key-features--business-impact)
- [Security & Compliance](#-security--compliance)

---

## 📐 Architectural Overview

The **Marina Bay eTIMS Integration Engine** serves as the compliance bridge between the Marina Bay Tenant Management System backend and the KRA eTIMS remote endpoints. It automates the generation of compliant fiscal receipts, tax calculation (split between 16% Standard VAT on Service Charges and Exempt Rent), digital signature verification (`rcptSign`), and fiscal QR code data extraction.


---

## ⚡ Why OSCU over VSCU?

During architectural planning, two KRA integration paths were evaluated: **Virtual Sales Control Unit (VSCU)** and **Online Sales Control Unit (OSCU)**.

| Criteria | VSCU (Virtual Unit) | OSCU (Online Unit) - *Selected* |
| :--- | :--- | :--- |
| **Hosting Model** | Requires self-hosted Java middleware / Docker proxy container. | **Pure Cloud-Native REST/JSON integration directly from API.** |
| **Maintenance** | Host OS updates, Java runtime management, proxy uptime monitoring. | **Zero local proxy footprint; direct HTTPS requests to KRA endpoints.** |
| **Deployment** | Server-bound containerized proxy setup. | **Fully serverless / cloud-ready (Azure App Service / AWS ECS).** |
| **Performance** | Extra network hop (API ➔ Local VSCU ➔ KRA API). | **Direct endpoint-to-endpoint communication.** |

---

## 🛠️ Technology Stack

### Backend Architecture
* **Framework:** C# / ASP.NET Core 8 Web API
* **ORM & Database:** Entity Framework Core 8, Microsoft SQL Server
* **HTTP Resilience:** `HttpClientFactory` with Polly policy handlers (Exponential Backoff & Circuit Breaker)
* **Serialization:** `System.Text.Json` custom snake_case / camelCase converters

### Frontend Architecture
* **Framework:** Angular 17 (TypeScript)
* **Styling:** Tailwind CSS + Reactive Forms
* **State Management:** RxJS Signals & Reactive Services

### Integration Infrastructure
* **Target Protocol:** KRA eTIMS OSCU API (v2.0 Specification)
* **Environments:** Sandbox (`etims-api-sbx.kra.go.ke`) & Production (`etims-api.kra.go.ke`)

---

## 🔄 System Architecture & Data Flow

```mermaid
sequenceDiagram
    autonumber
    actor Tenant / Admin
    participant Frontend as Angular Web App
    participant Backend as C# ASP.NET Core API
    participant DB as SQL Server
    participant KRA as KRA eTIMS OSCU API

    Note over Backend, KRA: Phase 1: Device Initialization (One-Time Setup)
    Backend->>KRA: POST /selectInitOsdcInfo (tin, bhfId, dvcSrlNo)
    KRA-->>Backend: Return HTTP 200 (cmcKey token)
    Backend->>DB: Persist cmcKey in EtimsConfig table

    Note over Tenant, KRA: Phase 2: Monthly Tenant Billing & Fiscalization
    Admin->>Frontend: Execute Monthly Billing Run
    Frontend->>Backend: POST /api/billing/generate-statements
    Backend->>DB: Save Invoices (Pending KRA Sync)
    
    loop For each Invoice / Statement
        Backend->>Backend: Map Line Items to KRA Codes (e.g. Service Charge = 16% VAT)
        Backend->>KRA: POST /saveTrnsSalesOsdc (Headers: tin, bhfId, cmcKey)
        
        alt KRA Success
            KRA-->>Backend: 200 OK (kraInvoiceNo, rcptSign, intrlData)
            Backend->>DB: Update Invoice record with KRA Signature & Timestamp
        else KRA API Offline / Timeout
            Backend->>DB: Queue Invoice in Background Retry Queue
        end
    end

    Backend-->>Frontend: Return Billing Results + Digital Receipts
    Frontend-->>Tenant: Render Statement with KRA Fiscal Verification Details


