# Claude Code Usage Disclosure

This prototype was developed with extensive use of Claude Code (Anthropic's AI coding assistant).

## Design Phase (Jan 22-25)

Claude generated the design documents in `docs/`, following this evolution:

1. **Requirements Analysis** — Started with clarifying questions to understand constraints (5-dev team, 6-month timeline, 20-50k DAU, Canada + Europe)

2. **Initial Architecture** — First draft assumed AWS stack (Node.js, S3, MediaConvert, SQS). This reflected industry-standard choices but didn't match my existing expertise.

3. **Pivot to Rails** — Shifted to Rails 8 API given my production experience. Claude generated implementation guides for both Mux and Coconut transcoding services.

4. **Vendor Decision** — Chose Coconut over Mux based on:
   - Lower cost (~$0.015/min vs Mux's $0.02/min + per-viewer fees)
   - More control over output formats and storage
   - S3-compatible storage integration

5. **Infrastructure Pivot** — Switched from AWS to DigitalOcean (Spaces + App Platform) for simpler deployment and cost predictability.

## Implementation Phase

Claude Code assisted with:
- Initial project scaffolding (controllers, models, services, jobs)
- Coconut API integration and webhook handling
- StorageService for DigitalOcean Spaces with presigned URLs
- Test suite setup with FactoryBot
- Terraform configuration (partially abandoned)
- API documentation and README

## Human Decisions

Key decisions I made independently:
- Choosing Rails over Node.js (familiarity)
- Selecting DigitalOcean over AWS (simpler, cost-effective for prototype)
- Keeping architecture simple (no Kubernetes, no multi-region)
- Abandoning Terraform in favor of manual App Platform setup
- Adding HTTP Basic Auth for demo security

## Debugging

Significant debugging sessions involved Claude Code for:
- S3 endpoint configuration issues (CDN vs API endpoint)
- Coconut API authentication (EU region endpoint)
- CORS configuration for browser uploads
- Test suite fixes after model refactoring

---

*This disclosure is provided in the interest of transparency regarding AI-assisted development.*
