# Deep Dive: Authorization Models

## Executive Summary

Authorization models form the foundation of modern access control systems, determining **who can access what resources under what conditions**. This research provides a comprehensive analysis of authorization paradigms ranging from traditional models like ACLs and RBAC to modern approaches like ReBAC, ABAC, and policy engines. The landscape has evolved significantly, with organizations increasingly adopting hybrid approaches and fine-grained authorization (FGA) to meet complex security requirements in distributed, cloud-native environments.

---

## 1. Introduction to Authorization

### Authentication vs. Authorization

| Aspect | Authentication | Authorization |
|--------|---------------|---------------|
| **Purpose** | Verifies identity | Determines access rights |
| **Question Answered** | "Who are you?" | "What can you do?" |
| **Mechanism** | Passwords, biometrics, MFA | Policies, roles, attributes |
| **Timing** | Once per session | Continuous, per-request |

### The CIA Triad and Authorization

Authorization directly supports all three pillars of information security:

- **Confidentiality**: Restricting data access to authorized users
- **Integrity**: Controlling who can modify data
- **Availability**: Preventing DoS attacks through access control

---

## 2. Core Authorization Models

### 2.1 Access Control Lists (ACL)

**Definition**: ACLs are object-centric lists specifying which subjects (users/groups) can access a resource and what operations they can perform.

**How It Works**:
```
Resource: document.txt
ACL:
  - user:alice → read, write
  - user:bob → read
  - group:editors → read, write, delete
```

**Advantages**:
- Simple to understand and implement
- Granular control at the resource level
- Easy to review who has access to specific resources
- Direct mapping between users and permissions

**Disadvantages**:
- **Scalability issues**: As users and resources grow, ACLs become unwieldy
- **Management complexity**: Permissions scattered across multiple resources
- **Performance overhead**: Requires searching entire lists for access decisions
- **Storage inefficiency**: Same subject may appear in multiple ACLs
- **No inherent grouping**: Difficult to manage users with similar access needs

**Best Use Cases**:
- Small organizations with limited users and resources
- File system permissions in operating systems
- Network device access control
- Simple web applications

---

### 2.2 Discretionary Access Control (DAC)

**Definition**: DAC allows resource owners to decide who can access their resources and what permissions they grant.

**Key Characteristics**:
- Resource owners control access decisions
- Access can be shared or transferred by users
- Based on user identity and ownership
- Commonly implemented using ACLs

**Real-World Examples**:
- Unix/Linux file permissions
- Windows file sharing
- Google Docs sharing settings
- Social media privacy controls

**Advantages**:
- Highly flexible and user-friendly
- Easy to implement and manage
- Empowers resource owners
- Quick permission changes without administrative overhead

**Disadvantages**:
- **Security vulnerabilities**: Relies on user discretion, prone to errors
- **Privilege creep**: Users may accumulate excessive permissions
- **Limited visibility**: Decentralized control makes auditing difficult
- **Malware propagation**: Compromised accounts can spread access
- **Inconsistent policies**: No centralized enforcement

---

### 2.3 Mandatory Access Control (MAC)

**Definition**: MAC enforces access through strict, system-defined policies based on security classifications and clearances. Users cannot modify permissions.

**Key Characteristics**:
- Centralized policy enforcement
- Based on security labels and classifications
- Users cannot override or share access
- Kernel-level enforcement

**Security Labels Example**:
```
User Clearance: TOP SECRET / SCI
Resource Classification: CONFIDENTIAL
Action: ALLOW (user clearance ≥ resource classification)
```

**Advantages**:
- **Highest security level**: Prevents unauthorized data leakage
- **Strict compliance**: Enforces regulatory requirements automatically
- **No user override**: Eliminates human error in access decisions
- **Centralized governance**: Single point of policy control

**Disadvantages**:
- **Very rigid**: No flexibility for business needs
- **Complex implementation**: Requires significant expertise
- **High administrative overhead**: All changes require administrator action
- **Limited usability**: Can impede legitimate workflows
- **Expensive to maintain**: Specialized systems and personnel

**Best Use Cases**:
- Military and government systems
- Intelligence agencies
- Critical infrastructure
- Highly regulated environments (nuclear, defense)

---

### 2.4 Role-Based Access Control (RBAC)

**Definition**: RBAC assigns permissions to roles rather than individual users. Users inherit permissions through role assignments.

**Core Components**:
- **Users**: Individuals or systems requiring access
- **Roles**: Job functions or responsibilities
- **Permissions**: Operations allowed on resources
- **Sessions**: Context where users activate roles

**RBAC Hierarchy**:
```
Level 1: Flat RBAC (basic role-permission mapping)
Level 2: Hierarchical RBAC (role inheritance)
Level 3: Constrained RBAC (separation of duties)
Level 4: Symmetric RBAC (role-permission review)
```

**Example Implementation**:
```
Roles:
  - Admin: [create, read, update, delete]
  - Editor: [create, read, update]
  - Viewer: [read]

User Assignments:
  - alice → Admin
  - bob → Editor
  - carol → Viewer
```

**Advantages**:
- **Simplified administration**: Manage permissions by role, not individual
- **Alignment with business structure**: Roles mirror organizational hierarchy
- **Reduced complexity**: Fewer permission assignments to manage
- **Easy onboarding**: New users assigned to appropriate roles
- **Strong auditability**: Clear mapping of who has what access
- **Industry adoption**: 94.7% of companies have implemented RBAC

**Disadvantages**:
- **Role explosion**: Too many roles as organizations grow complex
- **Static nature**: Roles don't adapt to context or conditions
- **Coarse-grained**: May grant more access than needed
- **Role design challenges**: Poorly designed roles create security gaps
- **Maintenance overhead**: Roles require regular review and cleanup

**The Role Explosion Problem**:

| Organization Size | Typical Role Count | Risk Level |
|-------------------|-------------------|------------|
| Small (<100 users) | 10-20 | Low |
| Medium (100-1000) | 50-150 | Medium |
| Large (1000-10000) | 200-1000 | High |
| Enterprise (>10000) | 1000+ | Critical |

**Solutions for Role Explosion**:
1. **Move context out of roles**: Use attributes for dynamic conditions
2. **Data classification**: Tag resources and apply policies based on classification
3. **Business entitlements**: Use external entitlement data instead of encoding in roles
4. **Role governance**: Regular recertification and cleanup processes
5. **Hybrid approaches**: Combine RBAC with ABAC for dynamic scenarios

---

### 2.5 Attribute-Based Access Control (ABAC)

**Definition**: ABAC makes access decisions based on attributes of users, resources, actions, and environment, evaluated against policies.

**Attribute Categories**:

| Category | Examples |
|----------|----------|
| **Subject Attributes** | department, role, clearance, location |
| **Resource Attributes** | classification, owner, sensitivity, type |
| **Action Attributes** | read, write, delete, execute |
| **Environment Attributes** | time, IP address, device type, risk score |

**Policy Example**:
```
PERMIT IF:
  user.department == resource.owner_department
  AND user.clearance >= resource.classification
  AND time.hour BETWEEN 9 AND 17
  AND device.trusted == true
```

**Advantages**:
- **High flexibility**: Adapts to complex, dynamic environments
- **Fine-grained control**: Context-aware decisions
- **Dynamic authorization**: Real-time policy evaluation
- **Scalability**: Attributes can be reused across policies
- **Regulatory compliance**: Supports complex compliance requirements
- **No role explosion**: Attributes replace many role combinations

**Disadvantages**:
- **Complexity**: Requires expertise to design and implement
- **Policy management**: Policies can become complex and hard to audit
- **Performance**: Real-time evaluation can introduce latency
- **Attribute management**: Requires comprehensive attribute infrastructure
- **Testing challenges**: Complex policies are harder to verify

**ABAC vs RBAC**:

| Aspect | RBAC | ABAC |
|--------|------|------|
| Granularity | Coarse | Fine |
| Flexibility | Low | High |
| Complexity | Simple | Complex |
| Context awareness | No | Yes |
| Scalability | Limited | High |
| Best for | Stable structures | Dynamic environments |

---

### 2.6 Relationship-Based Access Control (ReBAC)

**Definition**: ReBAC grants permissions based on relationships between entities (users, resources, groups), typically modeled as a graph.

**Core Concept**:
Instead of asking "What role does this user have?", ReBAC asks "How is this user related to this resource?"

**Relationship Tuples**:
```
user:alice owner document:report2024
user:bob member group:engineering
group:engineering viewer folder:projects
document:report2024 parent folder:projects
```

**Inheritance Example**:
```
If: user:carol member group:engineering
And: group:engineering viewer folder:projects
And: document:report2024 parent folder:projects
Then: user:carol can view document:report2024
```

**Google Zanzibar**:

Google's Zanzibar is the most influential ReBAC implementation, powering authorization for:
- Google Drive
- YouTube
- Gmail
- Google Photos
- Google Calendar

**Zanzibar Performance**:
- 95% of checks complete in <10ms
- 99.999% availability
- Trillions of ACLs
- Millions of checks per second

**OpenFGA** (Open Source Zanzibar Implementation):

```python
# OpenFGA Authorization Model
type user
type document
  relations
    define owner: [user]
    define editor: [user] or owner
    define viewer: [user] or editor
```

**Advantages**:
- **Natural modeling**: Mirrors real-world relationships
- **Permission inheritance**: Hierarchical access through relationships
- **Graph-based queries**: Efficient traversal for complex scenarios
- **Scalability**: Handles billions of relationships
- **Flexibility**: Can express RBAC and ABAC patterns
- **Collaboration-friendly**: Perfect for sharing scenarios

**Disadvantages**:
- **Complex implementation**: Requires graph database expertise
- **Limited context**: Static relationships don't capture dynamic conditions
- **Query complexity**: Complex relationship queries can be expensive
- **Learning curve**: Different mental model from traditional approaches

**ReBAC vs ABAC**:

| Aspect | ReBAC | ABAC |
|--------|-------|------|
| Primary data | Relationships | Attributes |
| Data model | Graph | Key-value |
| Best for | Hierarchies, sharing | Context-aware policies |
| Dynamic conditions | Limited | Excellent |
| Implementation | OpenFGA, Zanzibar | OPA, XACML |

---

### 2.7 Policy-Based Access Control (PBAC)

**Definition**: PBAC centralizes authorization logic into explicit policies that define access conditions, combining elements of RBAC, ABAC, and ReBAC.

**Key Characteristics**:
- Policies are the central artifact
- Externalized from application code
- Can combine multiple access control models
- Centralized policy management

**PBAC Architecture**:
```
┌─────────────────┐
│  Policy Engine  │
│  (PDP)          │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌─────────┐
│ PAP   │ │  PIP    │
│(Policy│ │(Policy  │
│Admin)  │ │Info)    │
└───────┘ └─────────┘
    │         │
    ▼         ▼
┌─────────────────┐
│  PEP            │
│ (Policy Enf.)   │
└─────────────────┘
```

**Policy Example** (Cedar):
```
permit (
    principal == User::"alice",
    action == Action::"viewDocument",
    resource == Document::"report2024"
)
when {
    resource.classification == "public" ||
    (resource.owner == principal && context.time.hour >= 9 && context.time.hour <= 17)
};
```

**Advantages**:
- **Centralized governance**: Single source of truth for policies
- **Flexibility**: Can express complex, multi-factor policies
- **Auditability**: Clear, explicit policy definitions
- **Version control**: Policies can be versioned like code
- **Separation of concerns**: Auth logic separate from business logic

**Disadvantages**:
- **Policy complexity**: Complex policies can be hard to manage
- **Performance overhead**: Centralized evaluation can introduce latency
- **Learning curve**: Requires understanding policy language
- **Dependency**: Policy engine becomes critical infrastructure

---

## 3. Capability-Based Security

**Definition**: Capability-based security uses unforgeable tokens (capabilities) that grant specific rights to access resources. Unlike ACLs where the resource tracks who can access it, capabilities are held by the subject.

**Capability Structure**:
```
Capability = {
  resource_id: "document:123",
  rights: ["read", "write"],
  issuer: "auth-server",
  expiry: "2024-12-31T23:59:59Z",
  signature: "..."
}
```

**ACL vs Capability Lists**:

| Aspect | ACL | Capability |
|--------|-----|------------|
| Orientation | Object-centric | Subject-centric |
| Storage | With resource | With subject |
| Delegation | Complex (update resource) | Simple (pass token) |
| Lookup | Search required | Direct access |
| Revocation | Remove from resource | Invalidate token |

**Advantages**:
- **Enhanced modularity**: Supports protected procedures and type managers
- **Improved security**: Encapsulation reduces trusted computing base
- **Easy delegation**: Transfer rights by passing capability
- **Least privilege**: Subjects only hold explicit authorities
- **No ambient authority**: Authority cannot be assumed from context

**Disadvantages**:
- **Memory management**: Difficult to deallocate unused capabilities
- **Revocation challenges**: Distributed capabilities hard to revoke
- **Complex propagation**: Tracking capability flow is difficult
- **Limited review**: Hard to audit who has access to what
- **Confused deputy problem**: Risk of capability misuse

**Modern Implementations**:
- CHERI processor extensions
- seL4 microkernel
- Web capabilities (CapTP)
- Cloud native capabilities (SPIFFE/SPIRE)

---

## 4. Modern Authorization Frameworks

### 4.1 Open Policy Agent (OPA)

**Overview**: OPA is a general-purpose policy engine that uses the Rego language for policy definition.

**Key Features**:
- General-purpose policy engine
- Declarative policy language (Rego)
- Multiple deployment options (sidecar, library, service)
- Rich ecosystem and integrations
- Kubernetes-native (Gatekeeper)

**Rego Example**:
```rego
package example.authz

import future.keywords.if
import future.keywords.in

default allow := false

allow if {
    input.user.role == "admin"
}

allow if {
    input.user.department == input.resource.department
    input.action == "read"
}
```

**Strengths**:
- Extremely flexible and expressive
- Mature ecosystem and tooling
- Wide range of integrations
- Production-ready

**Challenges**:
- Steep learning curve (Rego)
- Can be error-prone (runtime exceptions, non-determinism)
- Requires careful policy testing

---

### 4.2 AWS Cedar

**Overview**: Cedar is a policy language and evaluation engine designed specifically for application authorization.

**Key Features**:
- Purpose-built for authorization
- Human-readable syntax
- Static type checking
- Formal verification (Cedar Analysis)
- Deterministic evaluation

**Cedar Example**:
```cedar
permit (
    principal == User::"alice",
    action == Action::"view",
    resource == Photo::"vacation.jpg"
)
when {
    resource in Album::"personal"
};
```

**Strengths**:
- Security-first design
- Mathematical correctness guarantees
- Sub-millisecond performance
- Easy to read and write

**Limitations**:
- AWS ecosystem alignment
- Smaller community than OPA
- Limited external data integration

**OPA vs Cedar**:

| Aspect | OPA | Cedar |
|--------|-----|-------|
| Language | Rego | Cedar DSL |
| Focus | General policy | Authorization only |
| Learning curve | Steep | Moderate |
| Verification | Testing | Formal proof |
| Performance | Good | Excellent |
| Ecosystem | Large | Growing |

---

### 4.3 OpenFGA (Zanzibar-based)

**Overview**: OpenFGA is an open-source authorization system based on Google's Zanzibar paper.

**Key Features**:
- Relationship-based access control
- Graph-based permission model
- Horizontal scalability
- Consistent, low-latency reads

**Authorization Model**:
```yaml
model:
  schema_version: "1.1"
  type_definitions:
    - type: user
    - type: document
      relations:
        owner:
          directly:
            - type: user
        editor:
          directly:
            - type: user
          via:
            - relation: owner
        viewer:
          directly:
            - type: user
          via:
            - relation: editor
```

**Use Cases**:
- Multi-tenant SaaS applications
- Collaboration tools
- Social networks
- Content management systems

---

## 5. Authorization Standards

### 5.1 XACML (eXtensible Access Control Markup Language)

**Overview**: XACML is an OASIS standard for attribute-based access control using XML.

**Architecture**:
- **PDP** (Policy Decision Point): Evaluates policies
- **PEP** (Policy Enforcement Point): Enforces decisions
- **PAP** (Policy Administration Point): Manages policies
- **PIP** (Policy Information Point): Provides attributes

**XACML Policy Example**:
```xml
<Policy PolicyId="example" RuleCombiningAlgId="deny-overrides">
  <Target>
    <Subjects><Subject>user</Subject></Subjects>
    <Resources><Resource>document</Resource></Resources>
    <Actions><Action>read</Action></Actions>
  </Target>
  <Rule RuleId="1" Effect="Permit">
    <Condition>
      <Apply FunctionId="string-equal">
        <SubjectAttributeDesignator AttributeId="department"/>
        <ResourceAttributeDesignator AttributeId="owner-department"/>
      </Apply>
    </Condition>
  </Rule>
</Policy>
```

**Challenges**:
- Verbose XML syntax
- Complex implementation
- Limited adoption in modern applications

---

### 5.2 ALFA (Abbreviated Language for Authorization)

**Overview**: ALFA is a concise, human-readable DSL for authorization policies that compiles to XACML.

**ALFA Example**:
```alfa
namespace example {
  policyset documents {
    apply firstApplicable
    policy viewDocument {
      target clause actionId == "view" and resourceType == "document"
      apply denyUnlessPermit
      rule allowSameDepartment {
        permit
        condition user.department == resource.department
      }
    }
  }
}
```

**Benefits**:
- Concise syntax compared to XACML
- Human-readable policies
- Compiles to standard XACML
- Structured hierarchical organization

---

### 5.3 NGAC (Next Generation Access Control)

**Overview**: NGAC is an ANSI/INCITS standard (526-2016) that provides a fundamental, reusable set of relations and functions for access control.

**Key Features**:
- Expresses wide variety of policies
- Can enforce multiple policies simultaneously
- Logical object abstraction (location-independent)
- Personal Object System (POS) for unified resource view

**NGAC vs ABAC**:
- Both use attributes
- NGAC represents access control data as fundamental relations
- NGAC can identify objects accessible under each policy
- More amenable to diverse operational environments

**Use Cases**:
- Multi-cloud environments
- Complex policy combinations
- High-security environments
- Research and government systems

---

## 6. Granularity in Authorization

### 6.1 Coarse-Grained Authorization (CGA)

**Characteristics**:
- Based on single attributes (typically roles)
- Application-level or module-level access
- Simple to implement and manage
- Lower administrative overhead

**Example**:
```
Role "Admin" → Full application access
Role "User" → Limited application access
```

**Best For**:
- Small organizations
- Simple applications
- Rapid development
- Low-sensitivity data

---

### 6.2 Fine-Grained Authorization (FGA)

**Characteristics**:
- Multiple attributes and conditions
- Resource, record, or field-level access
- Context-aware decisions
- Higher complexity but greater precision

**Example**:
```
User can edit document IF:
  - User is owner OR
  - User has "editor" role AND
  - Document is not published AND
  - Time is within business hours AND
  - User is on corporate network
```

**FGA Implementation Approaches**:

| Approach | Technology | Use Case |
|----------|------------|----------|
| ABAC | OPA, XACML | Context-aware policies |
| ReBAC | OpenFGA, Zanzibar | Relationship-based access |
| PBAC | Cedar, Oso | Centralized policy management |

**Best For**:
- Large enterprises
- Multi-tenant applications
- Regulated industries
- Complex collaboration scenarios
- AI/LLM access control

---

### 6.3 Hybrid Approaches

Most modern systems use a combination:

```
┌─────────────────────────────────────┐
│         Authorization Stack         │
├─────────────────────────────────────┤
│  Layer 3: FGA (ABAC/ReBAC)          │
│           ↓ Context-aware decisions │
│  Layer 2: RBAC                      │
│           ↓ Role-based grouping     │
│  Layer 1: ACL/DAC                   │
│           ↓ Resource ownership      │
└─────────────────────────────────────┘
```

**Recommended Strategy**:
1. Start with RBAC for base access control
2. Layer ABAC for dynamic, context-aware decisions
3. Add ReBAC when relationships become important
4. Use FGA for the most sensitive operations

---

## 7. Authorization in Distributed Systems

### 7.1 Microservices Authorization Patterns

**Pattern 1: Decentralized Authorization**
- Each service implements its own authorization
- Pros: Low latency, service autonomy
- Cons: Code duplication, inconsistency, maintenance burden

**Pattern 2: Centralized with External PDP**
- Services call central policy decision point
- Pros: Consistency, centralized management
- Cons: Network latency, single point of failure

**Pattern 3: Centralized with Embedded PDP**
- Policy engine embedded in each service
- Policies distributed from central source
- Pros: Low latency, consistency
- Cons: Complexity, resource usage

**Pattern 4: Gateway-Based**
- Authorization at API gateway
- Pros: Single enforcement point
- Cons: Limited granularity, can't handle complex domain logic

**Pattern 5: Sidecar Pattern**
- Policy engine as sidecar container
- Pros: Language-agnostic, isolated from app
- Cons: Infrastructure complexity

---

### 7.2 Identity Propagation Patterns

**Clear/Self-Signed Data Structures**:
- Simple JSON or JWT
- Fast but requires trust between services
- Suitable for trusted environments

**Signed by Trusted Issuer**:
- "Passport" pattern with HMAC/signature
- External token agnostic
- Recommended for production

**Example (Passport Pattern)**:
```
1. Edge service validates external token
2. Creates internal "Passport" with user context
3. Signs Passport with shared secret
4. Propagates to downstream services
5. Services verify signature and extract context
```

---

### 7.3 Service-to-Service Authentication

**mTLS (Mutual TLS)**:
- Certificates for both client and server
- Platform-level solution recommended
- Widely adopted pattern

**Token-Based**:
- Service accounts with JWT
- Short-lived tokens
- Centralized token issuance

---

## 8. Best Practices and Recommendations

### 8.1 Design Principles

1. **Principle of Least Privilege**
   - Grant minimum necessary access
   - Regular access reviews
   - Just-in-time elevation

2. **Defense in Depth**
   - Multiple authorization layers
   - Don't rely on single control
   - Validate at each boundary

3. **Fail Secure**
   - Default deny
   - Explicit allow required
   - Log denied attempts

4. **Separation of Concerns**
   - Authorization logic separate from business logic
   - Centralized policy management
   - Clear interfaces

---

### 8.2 Implementation Best Practices

1. **Centralize Policies**
   - Single source of truth
   - Version control for policies
   - Code review for policy changes

2. **Use Declarative Policies**
   - Human-readable formats
   - Executable documentation
   - Easier testing and auditing

3. **Implement Policy Testing**
   - Unit tests for policies
   - Integration tests
   - Continuous validation

4. **Monitor and Audit**
   - Log all authorization decisions
   - Alert on anomalies
   - Regular access reviews

5. **Design for Scale**
   - Consider latency requirements
   - Cache authorization decisions
   - Plan for high availability

---

### 8.3 Zero Trust Authorization

**Core Principles**:
- Never trust, always verify
- Continuous validation
- Context-aware decisions
- Least privilege access

**Implementation**:
1. Verify identity (authentication)
2. Verify device health
3. Evaluate risk signals
4. Apply dynamic policies
5. Continuous monitoring
6. Session re-authentication

---

### 8.4 Model Selection Guide

| Scenario | Recommended Model | Notes |
|----------|------------------|-------|
| Small team, simple app | RBAC | Easy to implement |
| Enterprise with clear roles | RBAC + ABAC | Hybrid approach |
| Collaboration/sharing features | ReBAC | Natural relationship modeling |
| Highly regulated industry | ABAC/PBAC | Compliance requirements |
| Multi-tenant SaaS | ReBAC + ABAC | Tenant isolation + context |
| Microservices | PBAC with sidecar | Centralized, distributed enforcement |
| High-security environment | MAC + RBAC | Defense in depth |
| Dynamic, fast-changing | ABAC | Attribute flexibility |

---

## 9. Emerging Trends

### 9.1 Authorization as a Service (AaaS)

- Cloud-hosted authorization services
- Eliminates infrastructure management
- Examples: Oso Cloud, Auth0 FGA, Permit.io

### 9.2 AI/LLM Access Control

- Context-aware authorization for AI agents
- Ensuring AI only accesses permitted data
- Preventing data leakage through LLMs

### 9.3 OpenID AuthZEN

- Emerging standard for authorization
- Similar to OAuth for authentication
- Aims to reduce vendor lock-in

### 9.4 Policy as Code

- Treating policies like application code
- CI/CD integration
- Automated testing and deployment

---

## 10. Conclusion

The authorization landscape has evolved from simple ACLs to sophisticated, context-aware systems. Key takeaways:

1. **No single model fits all**: Choose based on your specific requirements
2. **Hybrid approaches are common**: Combine models for flexibility
3. **Fine-grained is the future**: Modern applications demand precision
4. **Policy as code**: Treat authorization as first-class infrastructure
5. **Centralize and externalize**: Keep auth logic out of application code
6. **Plan for scale**: Design authorization systems that grow with your organization

The most successful implementations typically start with a simple model (RBAC) and progressively enhance it with additional capabilities (ABAC, ReBAC) as requirements evolve. The key is to maintain clear governance, regular audits, and a security-first mindset throughout the authorization lifecycle.

---

## References and Further Reading

1. NIST RBAC Standard (ANSI/INCITS 359-2004)
2. Google Zanzibar Paper (USENIX ATC 2019)
3. NGAC Standard (ANSI/INCITS 526-2016)
4. XACML 3.0 Specification (OASIS)
5. Open Policy Agent Documentation
6. AWS Cedar Documentation
7. OpenFGA Documentation
8. OWASP Authorization Cheat Sheet
