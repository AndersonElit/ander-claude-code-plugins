# IEEE 830 Compliance Checklist

Use this checklist to validate the quality of an SRS document. Not every item applies to every project — focus on what's relevant.

## Document Completeness

- [ ] Purpose and scope are clearly defined
- [ ] All stakeholders and user classes are identified
- [ ] Functional requirements cover all features mentioned in the scope
- [ ] Non-functional requirements have measurable acceptance criteria
- [ ] External interfaces (user, hardware, software, communication) are documented
- [ ] Assumptions and dependencies are listed
- [ ] Glossary covers all domain-specific terms

## Requirement Quality Criteria

Each individual requirement should satisfy these properties:

### Correct
The requirement accurately reflects a genuine need of the stakeholder or system. Verify by asking: "If we implement this exactly as written, will it solve the user's actual problem?"

### Unambiguous
The requirement has only one possible interpretation. Red flags:
- Words like "appropriate", "intuitive", "user-friendly", "fast", "robust"
- Pronouns with unclear antecedents ("it should process it and return it")
- Compound requirements joined by "and/or"

**Fix:** Replace vague terms with measurable criteria. Split compound requirements.

### Complete
The requirement contains all information needed for implementation. A developer reading it should not need to guess at edge cases, error handling, or boundary conditions.

Check for:
- What happens with empty/null inputs?
- What happens at boundary values (0, max, negative)?
- What error messages should be displayed?
- What's the expected behavior for concurrent access?

### Consistent
No requirement contradicts another requirement in the document. Common conflicts:
- Two requirements specifying different behaviors for the same input
- Performance requirements that contradict security requirements (e.g., "respond in 50ms" vs "encrypt all data at rest with AES-256 and log every access")
- Priority conflicts (both features marked "must have" but they're mutually exclusive)

### Ranked for Importance
Every requirement has an explicit priority. Use one system consistently:
- **High / Medium / Low** — simple, works for most projects
- **MoSCoW** (Must / Should / Could / Won't) — more nuanced, good for agile
- **Numerical** (1-5) — fine-grained, useful for large requirement sets

### Verifiable
The requirement can be checked through testing, inspection, or demonstration. If you can't write a test case for it, rewrite it.

| Bad (not verifiable) | Good (verifiable) |
|---------------------|-------------------|
| "The system shall be fast" | "API responses shall return in < 200ms at p95 with 500 concurrent users" |
| "The system shall be secure" | "All passwords shall be hashed using bcrypt with a cost factor of 12" |
| "The system shall be easy to use" | "A new user shall complete the registration flow in under 2 minutes without external help" |
| "The system shall handle errors gracefully" | "When a downstream service returns 5xx, the system shall retry 3 times with exponential backoff (1s, 2s, 4s) and return a 503 with a user-friendly message if all retries fail" |

### Modifiable
The document structure allows requirements to be changed without cascading updates. This is achieved through:
- Unique IDs for every requirement
- Cross-references by ID (not by page number or section position)
- Non-redundant content (each requirement stated once, referenced elsewhere)

### Traceable
Every requirement can be traced:
- **Backward**: to its origin (stakeholder request, business rule, regulation)
- **Forward**: to design, implementation, and test artifacts

The traceability matrix in Section 6 of the SRS provides this.

## Non-Functional Requirements Checklist

### Performance
- [ ] Response time targets for key operations (with percentile and load)
- [ ] Throughput requirements (requests/second, transactions/minute)
- [ ] Maximum concurrent users
- [ ] Data volume expectations (current and projected growth)
- [ ] Batch processing time windows (if applicable)

### Security
- [ ] Authentication method specified
- [ ] Authorization model defined (RBAC, ABAC, etc.)
- [ ] Data encryption requirements (at rest, in transit)
- [ ] Session management rules
- [ ] Audit logging requirements
- [ ] Compliance standards referenced (OWASP, PCI-DSS, GDPR, HIPAA)
- [ ] Input validation requirements
- [ ] Rate limiting / abuse prevention

### Availability
- [ ] Uptime SLA defined (e.g., 99.9%)
- [ ] Planned maintenance windows
- [ ] Recovery Time Objective (RTO)
- [ ] Recovery Point Objective (RPO)
- [ ] Failover strategy
- [ ] Backup frequency and retention

### Scalability
- [ ] Expected growth trajectory
- [ ] Horizontal vs vertical scaling strategy
- [ ] Database scaling approach (sharding, read replicas)
- [ ] Caching strategy requirements
- [ ] Load balancing requirements

### Usability
- [ ] Accessibility standards (WCAG 2.1 level)
- [ ] Supported devices and screen sizes
- [ ] Language/localization requirements
- [ ] Maximum acceptable page load time for users
- [ ] Offline capability requirements (if applicable)

### Maintainability
- [ ] Code coverage targets
- [ ] Documentation requirements
- [ ] Deployment frequency expectations
- [ ] Monitoring and alerting requirements
- [ ] Log retention and format

## Common Pitfalls

1. **Gold plating** — Adding requirements nobody asked for. Every requirement should trace back to a stakeholder need.
2. **Implementation masquerading as requirements** — "Use React with Redux" is a design decision, not a requirement. The requirement is "The UI shall support modern browsers and render responsively across desktop and mobile."
3. **Missing error scenarios** — The happy path is always documented. The sad paths (timeouts, invalid input, system failures) are where real requirements live.
4. **Aspirational non-functionals** — "99.999% uptime" sounds great but costs a fortune. Make sure non-functional requirements match the project's budget and actual needs.
5. **Orphan requirements** — Requirements that exist in the document but aren't connected to any use case or feature. They usually indicate scope creep or leftover ideas from brainstorming.
