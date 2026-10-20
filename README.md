# Enterprise Security Architecture Overlay for Microsoft Foundry

## Executive Summary
This repository outlines a Zero Trust security architecture overlay for enterprise AI deployments using Microsoft Foundry and Azure OpenAI. Moving beyond basic platform adoption, this design strictly enforces network isolation, cryptographic identity verification, and data loss prevention to mitigate the OWASP Top 10 vulnerabilities for Large Language Models (LLMs).

The architecture assumes a baseline RAG (Retrieval-Augmented Generation) deployment and applies a strict separation of the Control Plane (infrastructure management) and Data Plane (AI agent execution and tool access).

---

## View 1: Network Perimeter & Routing Architecture
To prevent data exfiltration and enforce strict access controls, the AI application and its underlying PaaS dependencies (Azure OpenAI, Azure AI Search, Cosmos DB) are isolated from the public internet. 

*   **Ingress Zero Trust:** Client traffic is routed via a Split-Brain DNS architecture through an Application Gateway (WAF) and API Management (APIM) before reaching the App Service via Private Endpoint.
*   **Egress Inspection:** Outbound agent compute traffic is forced via VNet Integration and a `0.0.0.0/0` User Defined Route (UDR) to Azure Firewall, terminating at Zscaler Internet Access (ZIA) for Layer 7 DLP and URL filtering.

![Azure AI Network Security Architecture](./Azure_AI_Network_Security_Architecture_-_Gabriel_Thomas.jpg)
*(Note: Ensure your image file is in the repository and the path matches)*

---

## View 2: Identity & Authorization Sequence
This architecture mandates a **Dual-Identity Model**, strictly separating the infrastructure authentication from user data access to prevent privilege escalation via prompt injection.

1.  **System-Assigned Identity:** The AI Agent authenticates to the Azure OpenAI Private Endpoint using its own Managed Identity (Control Plane).
2.  **OAuth 2.0 On-Behalf-Of (OBO) Flow:** To access downstream tools (e.g., Sales Database), the Agent exchanges the user's session token for a scoped OBO token. The downstream API verifies the Entra ID cryptographic signature, ensuring access control is evaluated against the *human user*, not the AI application.

![Identity & Authorization Sequence Diagram](./Identity_&_Authorization_Sequence_Diagram_-_Gabriel_Thomas_2.jpg)

### IAM Least Privilege Matrix (Managed Identity)
The AI Application's Managed Identity is restricted to the following Azure built-in roles:
*   **`Foundry User`**: Baseline interaction with project data and agents; denies project settings management.
*   **`Cognitive Services OpenAI User`**: Permits prompt execution; denies model deployment/alteration.
*   **`Search Index Data Reader`**: Read-only queries for RAG execution; prevents index deletion or poisoning.

---

## View 3: Security Guardrails & OWASP Mitigation Matrix

| OWASP LLM Vulnerability | Architectural Threat | Azure Native Mitigation Control |
| :--- | :--- | :--- |
| **LLM01: Prompt Injection** | Attacker bypasses system instructions via malicious RAG context. | **Azure AI Content Safety (Prompt Shields):** RAG context is routed via the `documents` API field and Base-64 encoded to neutralize instructional weight. |
| **LLM02: Insecure Output Handling** | AI generates a SQL injection payload passed to downstream tools. | **API Management (APIM):** Outbound responses are validated against strict OpenAPI JSON schemas; malformed payloads are dropped. |
| **LLM03: Training Data Poisoning** | Attacker alters the underlying vector embeddings. | **Azure RBAC:** AI Managed Identity is restricted to `Search Index Data Reader`; cannot write to the vector database. |
| **LLM04: Model Denial of Service** | User floods context window to exhaust API quota or compute. | **API Management (APIM):** Enforces `max-token` limits and rate-limiting policies at the VNet ingress gateway. |
| **LLM05: Supply Chain Vulnerabilities** | Developers deploy compromised HuggingFace models. | **Azure Policy:** Enforces `Only Allow Direct From Azure` and restricts deployments to an approved whitelist (e.g., `gpt-4o`). |
| **LLM06: Sensitive Information Disclosure** | AI attempts to exfiltrate data to an unauthorized external IP. | **VNet Integration & UDR:** Outbound traffic is forced to Azure Firewall and Zscaler for destination URL filtering. |
| **LLM06: Sensitive Information Disclosure** | AI summarizes confidential internal documents for an external user. | **Purview DSPM for AI:** Inline API evaluation blocks outputs containing unauthorized Sensitivity Labels or custom SITs before egress. |
| **LLM09: Overreliance (Misinformation)** | AI hallucinates unverified claims not found in RAG sources. | **Azure AI Content Safety (Groundedness):** Evaluates final response against source text; ungrounded outputs are blocked prior to user delivery. |