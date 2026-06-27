<div align="center">

# 🚗 Cert-V2X

### IEEE 1609.2-compliant PKI backend for Vehicle-to-Everything (V2X) certificate issuance and lifecycle management — built on Spring Boot, designed for real-time vehicle-to-CA communication.

[![Java](https://img.shields.io/badge/Java_21-ED8B00?style=flat-square&logo=openjdk&logoColor=white)](https://openjdk.org)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=flat-square&logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![PKI](https://img.shields.io/badge/PKI-IEEE_1609.2-003087?style=flat-square&logo=letsencrypt&logoColor=white)](https://standards.ieee.org/ieee/1609.2/6085/)
[![c2c-common](https://img.shields.io/badge/Library-c2c--common-6A0DAD?style=flat-square&logo=github&logoColor=white)](https://github.com/CGI-SE-Trusted-Services/c2c-common)
[![Hackathon](https://img.shields.io/badge/AppViewX_Hackathon-Top_Finish_%7C_40%2B_Teams-FFD700?style=flat-square)]()

</div>

---

## ⚡ Hook

> Modern V2X safety systems — emergency braking, intersection management, platooning — depend on vehicles being able to authenticate each other in milliseconds. Cert-V2X is a Spring Boot PKI backend that issues, manages, and revokes IEEE 1609.2-compliant certificates for vehicles via REST APIs, enabling a complete V2X trust infrastructure without a monolithic CA setup.

---

## 🔭 The Problem

Vehicle-to-Everything (V2X) communication is the backbone of intelligent transport systems: vehicles broadcasting safety messages to each other, to infrastructure, and to pedestrian devices at sub-100ms latency. For these messages to be trusted, every vehicle needs a valid, short-lived cryptographic credential — an **Authorization Ticket** — issued by a compliant PKI hierarchy.

The problem with existing PKI tooling for V2X:
- Enterprise CA systems are not designed for high-volume, short-validity certificate issuance at vehicle scale
- The enrollment and authorization flows defined in **IEEE 1609.2** are complex, stateful, and poorly documented outside of standards bodies
- There is no lightweight, developer-friendly backend that exposes V2X certificate lifecycle operations over a clean REST API

**Cert-V2X solves this** by wrapping the [c2c-common](https://github.com/CGI-SE-Trusted-Services/c2c-common) IEEE 1609.2 library in a production-shaped Spring Boot service — giving vehicles a simple REST interface to enroll, get authorized, and have credentials revoked.

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          VEHICLE LAYER                              │
│   Vehicle OBU  ──REST──▶  /api/v1/enroll                           │
│   (On-Board Unit)  ──REST──▶  /api/v1/authorize                    │
│                    ──REST──▶  /api/v1/revoke/{certId}              │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTPS / REST
┌────────────────────────────▼────────────────────────────────────────┐
│                     CERT-V2X SERVICE (Spring Boot)                  │
│                                                                     │
│  ┌─────────────────┐   ┌──────────────────┐   ┌─────────────────┐  │
│  │  REST Controllers│   │  Cert Service    │   │  Revocation Svc │  │
│  │  (API Gateway)   │──▶│  (Issuance       │   │  (CRL Manager)  │  │
│  │                  │   │   Orchestrator)  │   │                 │  │
│  └─────────────────┘   └────────┬─────────┘   └────────┬────────┘  │
│                                 │                       │           │
│  ┌──────────────────────────────▼───────────────────────▼────────┐  │
│  │              c2c-common Library (IEEE 1609.2 / ETSI TS 103097)│  │
│  │   CryptoManager · AuthorityCertGenerator · SecuredDataGen     │  │
│  │   ETSIEnrollmentCredentialGenerator · AuthTicketCertGenerator │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                 │                                   │
│  ┌──────────────────────────────▼────────────────────────────────┐  │
│  │                     PKI Hierarchy (In-Memory / Configurable)   │  │
│  │   Root CA  ──▶  Enrollment CA  ──▶  Authorization CA         │  │
│  └────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Component breakdown

| Component | Responsibility |
|---|---|
| **REST Controllers** | Expose `/enroll`, `/authorize`, `/revoke`, `/status` endpoints; validate request payloads; return certificate bytes or error codes |
| **Certificate Service** | Orchestrates the IEEE 1609.2 enrollment and authorization flows using c2c-common generators; manages key pairs per vehicle identity |
| **Revocation Service** | Maintains CRL state; triggers c2c-common CRL generation; exposes revocation status endpoint |
| **PKI Hierarchy** | In-memory Root CA → Enrollment CA → Authorization CA chain initialized at startup; configurable via `application.yml` |
| **c2c-common** | Upstream library providing all IEEE 1609.2 / ETSI TS 103 097 cryptographic data structures, generators, and COER encoding |

---

## 🔑 REST API Design

### Enrollment — `POST /api/v1/enroll`

Initiates the enrollment flow for a vehicle OBU. Accepts vehicle identity metadata, generates a signed Enrollment Credential using the Enrollment CA, and returns the certificate bytes COER-encoded.

```
POST /api/v1/enroll
Content-Type: application/json

{
  "vehicleId": "OBU-20240601-001",
  "publicKey": "<base64-encoded-EC-public-key>",
  "validityYears": 5
}

Response 200:
{
  "certificateId": "a3f9...hashedId8",
  "enrollmentCert": "<base64-COER-encoded-EtsiTs103097Certificate>",
  "issuedAt": "2024-06-01T10:00:00Z",
  "expiresAt": "2029-06-01T10:00:00Z"
}
```

### Authorization — `POST /api/v1/authorize`

Issues a short-lived Authorization Ticket to a previously enrolled vehicle. Authorization Tickets are the credentials vehicles attach to V2X safety messages (CAM, DENM) to prove authenticity.

```
POST /api/v1/authorize
Content-Type: application/json

{
  "certificateId": "a3f9...hashedId8",
  "enrollmentCert": "<base64-COER-encoded-enrollment-cert>",
  "appPermissions": [{ "psid": 36, "ssp": null }]
}

Response 200:
{
  "authTicketId": "b7c2...hashedId8",
  "authorizationTicket": "<base64-COER-encoded-EtsiTs103097Certificate>",
  "validFor": "PT1H",
  "psid": 36
}
```

### Revocation — `DELETE /api/v1/revoke/{certId}`

Marks a certificate as revoked and updates the CRL. The CRL can be fetched by infrastructure nodes to validate vehicle messages in real time.

```
DELETE /api/v1/revoke/a3f9...hashedId8

Response 200:
{
  "revokedCertId": "a3f9...hashedId8",
  "crlUpdatedAt": "2024-06-01T11:00:00Z"
}
```

### CRL — `GET /api/v1/crl`

Returns the current signed Certificate Revocation List as a COER-encoded `EtsiTs103097DataSigned` structure, signed by the Root CA.

---

## 🛠 Tech Stack — and Why

| Tool | Why this over the alternative |
|---|---|
| **Spring Boot 3** | Production-grade REST layer with autoconfiguration, exception handling, and embedded server — the right choice for a PKI service that may grow into a real deployment. Micronaut or Quarkus are valid but Spring's ecosystem depth matters for cert management tooling. |
| **c2c-common (CGI)** | The only open-source Java library with a correct, tested implementation of both IEEE 1609.2 (US) and ETSI TS 103 097 (EU) V2X PKI structures. Implementing COER-encoded ASN.1 V2X data structures from scratch would have taken months. |
| **Bouncy Castle** | Required by c2c-common as the underlying JCE provider for ECDSA (P-256 / Brainpool P-256r1) operations. Industry standard for Java crypto. |
| **IEEE 1609.2** | The US standard (with 1609.2a 2017 amendment) defines the exact certificate formats, PSID-based app permissions, and authorization ticket structures used in deployed V2X systems (DSRC, C-V2X). Compliance is non-negotiable for interoperability. |
| **In-memory CA hierarchy** | For a hackathon prototype, an in-memory Root → Enrollment CA → Authorization CA chain avoids HSM/database dependencies while fully exercising the cert issuance flows. A production deployment would persist CA keys in an HSM. |

---

## 🏆 The Engineering Triumph — Wiring IEEE 1609.2 into a REST Service

### Situation
The c2c-common library is a correct, complete implementation of IEEE 1609.2 — but it is a low-level cryptographic library, not a service. It has no REST layer, no state management, no concept of a vehicle identity, and no enrollment flow orchestration. The hackathon challenge was to build a meaningful V2X PKI service in a time-boxed environment, starting from this foundation.

### Task
Design and implement the service layer between a vehicle's REST API call and the complex, multi-step IEEE 1609.2 enrollment and authorization flows — handling key pair generation, CA hierarchy initialization, certificate chaining, and CRL management — in a way that could plausibly run in a real V2X deployment.

### Action
Three non-trivial engineering decisions were made:

**1. CA hierarchy initialization at startup** — Rather than requiring pre-generated CA certificates, the service initializes a complete Root CA → Enrollment CA → Authorization CA chain on Spring Boot startup using c2c-common's `ETSIAuthorityCertGenerator`. This means a fresh instance is immediately operational with a valid PKI hierarchy, which is critical for a demo environment.

**2. Vehicle identity mapped to key pair lifecycle** — Each `vehicleId` is mapped to its own generated ECDSA P-256 key pair, stored in a transient `ConcurrentHashMap` keyed by the `HashedId8` of its enrollment certificate. This gives the authorization flow a way to look up the correct signing key without a database, while correctly modelling the per-vehicle identity isolation that IEEE 1609.2 mandates.

**3. Authorization Ticket expiry scoped to PSID** — Rather than issuing a single authorization credential, the service issues per-PSID Authorization Tickets with configurable short validity windows (default: 1 hour). This matches the IEEE 1609.2 design intent: vehicles hold multiple short-lived credentials scoped to specific application permissions (CAM = PSID 36, DENM = PSID 37), not a single long-lived certificate.

### Result
- A working REST API that exercises the complete IEEE 1609.2 enrollment → authorization → revocation lifecycle
- Recognized among **40+ teams** at the AppViewX internal hackathon for concept novelty and implementation quality
- Demonstrated that a V2X PKI backend could be built on top of existing open-source ITS crypto tooling rather than requiring a purpose-built CA system

---

## 🗺 Certificate Flow Diagram

```
Vehicle OBU                   Cert-V2X Service              PKI Hierarchy
    │                               │                              │
    │── POST /enroll ──────────────▶│                              │
    │   { vehicleId, pubKey }       │── genKeyPair(ECDSA P-256) ──▶│
    │                               │── genEnrollCert() ──────────▶│ (Enrollment CA signs)
    │◀── EnrollmentCredential ──────│◀── EtsiTs103097Certificate ──│
    │   (COER-encoded, 5yr)         │                              │
    │                               │                              │
    │── POST /authorize ───────────▶│                              │
    │   { certId, enrollCert }      │── genAuthTicket() ──────────▶│ (Authorization CA signs)
    │◀── AuthorizationTicket ───────│◀── EtsiTs103097Certificate ──│
    │   (COER-encoded, 1hr, PSID)   │                              │
    │                               │                              │
    │   [Vehicle broadcasts V2X     │                              │
    │    safety message signed      │                              │
    │    with Authorization Ticket] │                              │
    │                               │                              │
    │── DELETE /revoke/{certId} ───▶│                              │
    │                               │── updateCRL() ─────────────▶│
    │◀── 200 OK ────────────────────│                              │
```

---

## 🚀 Getting Started

> **Note:** This repository contains architecture documentation only. Source code is not published due to company IP constraints — this was built as part of an AppViewX internal hackathon. The architecture, API design, and flow documentation here represent the full system as built.

To build a similar system yourself:

**Step 1 — Add c2c-common as a Gradle dependency**
```groovy
// build.gradle
repositories {
    maven { url 'https://jitpack.io' }
}

dependencies {
    implementation 'com.github.CGI-SE-Trusted-Services:c2c-common:2.0.0-Beta5'
    implementation 'org.springframework.boot:spring-boot-starter-web:3.2.0'
    implementation 'org.bouncycastle:bcprov-jdk18on:1.77'
}
```

**Step 2 — Initialize the crypto manager and CA hierarchy**
```java
// On application startup
CryptoManager cryptoManager = new DefaultCryptoManager();
cryptoManager.setupAndConnect(new DefaultCryptoManagerParams("BC"));

ETSIAuthorityCertGenerator authorityCertGenerator =
    new ETSIAuthorityCertGenerator(cryptoManager);

// Generate Root CA → Enrollment CA → Authorization CA chain
// See c2c-common README for full example
```

**Step 3 — Wire enrollment into a REST controller**
```java
@PostMapping("/api/v1/enroll")
public ResponseEntity<EnrollmentResponse> enroll(@RequestBody EnrollmentRequest req) {
    // 1. Generate key pair for vehicle
    // 2. Call ETSIEnrollmentCredentialGenerator.genEnrollCredential()
    // 3. Store HashedId8 → KeyPair mapping
    // 4. Return COER-encoded certificate
}
```

---

## 📁 Repository Structure

```
cert-v2x/
├── docs/
│   ├── architecture.md          # This document
│   ├── api-design.md            # Full REST API specification
│   └── diagrams/
│       ├── system-architecture.png
│       └── cert-flow-sequence.png
├── README.md
└── REFERENCES.md
```

---

## 📚 Standards & References

| Standard | Scope |
|---|---|
| [IEEE 1609.2-2016 + 1609.2a-2017](https://standards.ieee.org/ieee/1609.2/6085/) | V2X certificate formats, COER encoding, PSID-based permissions, Authorization Ticket structure |
| [ETSI TS 103 097 V1.3.1](https://www.etsi.org/deliver/etsi_ts/103000_103099/103097/01.03.01_60/ts_103097v010301p.pdf) | EU ITS security headers and certificate profiles |
| [ETSI TS 102 941 V1.3.1](https://www.etsi.org/deliver/etsi_ts/102900_102999/102941/01.03.01_60/ts_102941v010301p.pdf) | ITS PKI enrollment and authorization message flows |
| [c2c-common](https://github.com/CGI-SE-Trusted-Services/c2c-common) | Open-source Java implementation of the above standards (AGPL-3.0) |

---

## 🏆 Recognition

Built as part of the **AppViewX Internal Hackathon** (2024) — recognized among **40+ competing teams** for concept novelty and implementation quality.

> AppViewX is an enterprise PKI and certificate lifecycle management platform serving Fortune 500 clients in banking, healthcare, and government. The hackathon explored next-generation use cases for certificate automation.

---

## 👤 Author

**Sarvesh Ramani** — SDE-II @ AppViewX | M.Tech AI/ML @ BITS Pilani

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/sarvesh-ramani)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/Sarvesh-Ramani)

---

<div align="center">
  <sub>IEEE 1609.2-compliant · Built on c2c-common (AGPL-3.0) · AppViewX Hackathon 2024</sub>
</div>
