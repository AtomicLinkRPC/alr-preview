# AtomicLinkRPC (ALR)

Welcome! This repository contains the AtomicLinkRPC (ALR) project. ALR is a high-performance C++ framework for building asynchronous, scalable, and easy-to-use RPC applications.

![Atom by Atom](assets/images/atom_by_atom.png){ style="display:block; margin:auto; width:75%;" }

---

The documentation is structured to provide a comprehensive understanding of ALR, from a high-level overview to deep technical details.

## Getting Started

-   **[Overview](./overview.md)**: A high-level introduction to ALR, its goals, and key features. Start here to get a feel for what ALR is all about.

-   **[Building From Source](./building-from-source.md)**: Instructions to prepare and build ALR from sources. Afterwards, run the examples and performance tests. Use this as a starting point for building your own ALR applications.

-   **[User Guide](./user-guide.md)**: A practical guide with step-by-step instructions and examples to help you understand how to build your own ALR applications.

## Core Reference

-   **[API Reference - Core](./api-reference-core.md)**: A reference for essential ALR classes, types, and the ambient context system. Start here for foundational API information.

-   **[API Reference - Advanced](./api-reference-advanced.md)**: Advanced features including service registry, discovery, thread control, performance utilities, and schema introspection.

-   **[Examples](./examples.md)**: A collection of code examples demonstrating various features of ALR, from basic "Hello World" to advanced concepts like service discovery and peer-to-peer communication.

## Deep Dives

-   **[Technical Documentation](./technical.md)**: A deep dive into the architecture and design principles of ALR, including ALR codegen, schema negotiation, serialization, and threading model.

-   **[Evolve Policies](./evolve-policies.md)**: A detailed description of the API Evolve Policy feature with policy modes, enum modifiers, helper functions, and violation handling to safely evolve your API.

-   **[Migration Guide](./migration-guide.md)**: A guide for transitioning from other RPC frameworks (gRPC, Cap'n Proto, REST) to ALR with concept mapping and code examples.

-   **[Troubleshooting Guide](./troubleshooting.md)**: Solutions for build issues, runtime errors, performance tuning, debugging, and API evolution troubleshooting.

## Performance & Promotion

-   **[Benchmarks](./benchmarks.md)**: A performance comparison between ALR and gRPC, showcasing ALR's significant speed advantages with detailed analysis and charts.

-   **[Promotional Brief](./promotional.md)**: A summary of ALR's value proposition, targeting executives and stakeholders who may not be interested in the technical details.

## Comprehensive Reference

-   **[Comprehensive Guide](./atomic-link-rpc.md)**: A single document that combines the most important information from all other documents into a comprehensive resource.

## Raw Data & Examples

-   **[Raw Results (localhost)](./assets/results/raw_results_localhost.md)**: Raw performance results from a subset of tests used during ALR development, using loopback.
 
-   **[Raw Results (remote)](./assets/results/raw_results_remote.md)**: Raw performance results from a subset of tests used during ALR development, using a physical 2.5Gbps Ethernet connection between two physical hosts.

-   **Source Examples**: Source code from the CityGuide example:
    - [client_main.cpp](./assets/city_guide_example_src/client_main.cpp.md)
    - [city_guide_common.h](./assets/city_guide_example_src/city_guide_common.h.md)
    - [client.h](./assets/city_guide_example_src/client.h.md)
    - [client.cpp](./assets/city_guide_example_src/client.cpp.md)
    - [service.h](./assets/city_guide_example_src/service.h.md)
    - [service.cpp](./assets/city_guide_example_src/service.cpp.md)
    - [service_main.cpp](./assets/city_guide_example_src/service_main.cpp.md)
    - [alr.cfg](./assets/city_guide_example_src/alr.cfg.md)

---
Notes

- APIs and performance may change.
- If a link is broken, please file feedback using the template in the repository.
- For additional information, email <alr-project@alienworks.com>.
