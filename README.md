# AtomicLinkRPC (ALR) — Document Preview

This repository is a documentation-only preview of AtomicLinkRPC, a C++ framework for high-performance, asynchronous, and scalable RPC.

<p align="center">
<img src="docs/assets/images/atom_by_atom.png" alt="Atom by Atom" width="75%"/>
</p>

---

**What’s inside**
- Overview & design goals
- Type system & threading model
- Built-in Registry: discovery, registration, and load-balancing
- CommonEndpointClass: simplified endpoint ergonomics
- API & usage patterns (C++-first, low boilerplate)
- Benchmarks and methodology
- Roadmap and feedback channels

**What’s not inside (yet)**
- Framework source code. This preview is docs-only.

**Quick links**
- Start here: [docs/index.md](./docs/index.md)
- Give feedback: open an issue using the [Feedback template](./.github/ISSUE_TEMPLATE/feedback.yml)
- License for docs: [LICENSE-DOCS.md](./LICENSE-DOCS.md)
- Conduct: [CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md)

---

## Why ALR?
- **C++-native ergonomics:** your classes *are* the IDL.
- **Flexible sync/async:** call patterns in any order; static and instance methods supported.
- **Tiny messages:** schema handshake enables compact, tag-free binary layout.
- **Throughput at scale:** opportunistic batching + lock-free hot paths.
- **Operational:** built-in Registry (discovery + load-balancing), ambient vars, async callbacks.

---

## Roadmap (high-level, non-binding)
- v0.1 Docs Preview (this repo)
- v0.2 Public demos & perf harness
- v0.3 Early-access SDK (select partners)
- v1.0 Initial code release

---

## Questions & Feedback
- Use the [issue template](./.github/ISSUE_TEMPLATE/feedback.yml) and include the doc page, section, and suggestions.
- Prefer actionable feedback (missing details, unclear steps, incorrect claims).
- For additional information, email <alr-project@alienworks.com>.

---

## Trademark and Legal
- See [TRADEMARKS.md](./TRADEMARKS.md) and [DISCLAIMER.md](./DISCLAIMER.md).

AtomicLinkRPC™ and ALR™ are trademarks of the AtomicLinkRPC project. All other marks belong to their respective owners.
