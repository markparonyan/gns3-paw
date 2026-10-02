# Spec: Python client for the GNS3 v3 controller API

## Problem Statement

People who automate GNS3 labs from Python (network engineers, trainers, CI pipelines, Ansible/Nornir users, AI tooling) have no maintained, official way to talk to a GNS3 3.x server. The request has been open since 2015 (GNS3/gns3-server#353), and every attempt so far has fallen short:

- **gns3fy** is pleasant to use but only speaks the v2 API, cannot authenticate against v3 (JWT), is pinned to Pydantic v1 and is effectively unmaintained.
- **The earlier gns3-paw / gns3client MVP** targets v3 but wraps ~183k lines of output from an old OpenAPI generator. It swallows errors into dictionaries and has argument-mismatch bugs in project creation and snapshots. It is awkward to use (string method names like `get_projects_v3_projects_get`) and to maintain.
- **The gns3-copilot client inside gns3-server 3.1** works against v3, but it is plain dictionaries and handler functions bound to server internals and LangChain. It cannot be used outside the server.

As a result, users hand-roll HTTP calls. They deal with token handling, error parsing and the controller's quirks (port numbering, default names, asynchronous node start-up) on their own. Every new server release can silently break their scripts.

## Solution

Build a new Python package with two layers:

1. **Generated low-level layer.** It is produced from the server's OpenAPI document (`openapi.json`): typed models for every schema, and one typed method per API operation. It is regenerated automatically for each GNS3 release, so it always matches the server. This is the direction the GNS3 maintainers asked for in #353.
2. **Hand-written convenience layer.** It sits on top of the low-level layer and is "simple and clean for users", in the spirit of Cisco's virl2-client:
   - work with projects, nodes and links by name
   - connect nodes by port name or by next free port
   - wait for real state instead of sleeping
   - batch operations
   - clear exceptions

Both layers are available as a synchronous and an asynchronous client with the same surface. The package starts under the author's GitHub account and is intended to move into the GNS3 organization later.

## User Stories

### Connection and authentication

1. As a lab automator, I want to create a client with a server URL, username and password, so that I can start scripting without handling tokens myself.
2. As a lab automator, I want to create a client from an existing access token, so that I can reuse a token obtained elsewhere (for example, from the web UI or a CI secret).
3. As a CI pipeline author, I want the client to read the server URL and credentials from environment variables when they are not passed, so that secrets stay out of scripts.
4. As a long-running service author, I want the client to refresh its access token with the refresh token before or when it expires, so that sessions longer than the token lifetime keep working.
5. As a user, I want to log out explicitly, so that my tokens are revoked when a script finishes.
6. As a security-conscious user, I want TLS verification on by default, and a visible warning when I turn it off, so that I don't accidentally talk to a spoofed server.
7. As a user of a self-signed lab server, I want to pass a custom CA bundle, so that I can keep verification on without a public certificate.
8. As a user, I want to set request timeouts, so that a hung server doesn't block my script forever.
9. As a user, I want the client to check the server version when it connects and warn me if it is not a supported 3.x version, so that I learn about incompatibilities early.
10. As a user, I want to use the client as a context manager, so that connections are closed reliably.

### Sync and async

11. As a script writer, I want a synchronous client, so that I can write simple top-to-bottom automation.
12. As an asyncio application developer, I want an asynchronous client with the same methods and return types, so that I can embed GNS3 control in async services without threads.
13. As a maintainer, I want the sync and async clients generated from one definition, so that they never drift apart.

### Low-level API coverage

14. As a power user, I want typed methods for every non-excluded operation in the v3 API (projects, nodes, links, drawings, snapshots, templates, images, appliances, computes, symbols, users, groups, roles, privileges, ACL, resource pools, API keys, server settings, controller), so that I never have to drop down to raw HTTP.
15. As a power user, I want method names and parameters that read naturally (for example, "get node", "start node"), so that the API is guessable.
16. As a power user, I want request bodies and responses as typed model objects, so that my editor autocompletes fields and type checkers catch mistakes.
17. As a power user, I want path identifiers typed as UUIDs, with strings accepted too, so that I can pass whatever I have.
18. As a power user, I want query parameters such as tag filters, export options and recursive file listing as keyword arguments, so that every option the API offers is reachable.
19. As a user, I want responses the API leaves untyped (stats, markers, statistics, available filters) returned as plain dictionaries, so that I still get the data even when no schema exists.
20. As a user, I want fields that a newer server adds to be ignored instead of crashing my older client, so that upgrading the server doesn't break my scripts.
21. As a user, I want template-specific properties (RAM, adapters, images and other emulator settings) preserved when I read and update templates, so that I don't lose data the schema doesn't spell out.
22. As a user, I want partial updates to send only the fields I set, so that updating a node's name doesn't reset its other properties.
23. As a user, I want to be able to set a field to null on purpose (for example, to restore a server setting's default), as distinct from leaving it out, so that I can express both intents.
24. As a user, I want appliances returned as the correct model for their registry version, so that I can read v8 and older appliances alike.

### Files, streams and binary data

25. As a user, I want to export a project as a binary archive streamed to a file, with the export options available, so that I can back up labs without loading everything into memory.
26. As a user, I want to import a project from an archive file or stream, so that I can restore or share labs.
27. As a user, I want to upload images (QEMU, IOS, IOU) from a file path or stream, so that I can provision templates from scripts.
28. As a user, I want to read, write, list and delete files in a node's directory, so that I can push startup configs and pull logs.
29. As a user, I want to read and write project files and fetch the project's topology file, so that I can inspect or version lab definitions.
30. As a user, I want to download a link's packet capture file, or stream it while capturing, so that I can analyse traffic with my own tools.
31. As a user, I want to upload and download custom symbols, so that I can brand my labs.
32. As a user, I want to consume the controller and project notification streams as an iterator (sync) or async iterator, so that I can react to server events.

### Errors

33. As a user, I want every failed request to raise an exception, never to come back as an error dictionary, so that failures can't be mistaken for data.
34. As a user, I want a small exception hierarchy that maps statuses to meaningful types (authentication, permission, not found, conflict, validation, server error), so that I can catch exactly what I care about.
35. As a user, I want the exception to carry the HTTP status, the server's message and the raw response, so that I can debug and log precisely.
36. As a user, I want validation errors (422) to expose the server's per-field details, so that I can see which input was wrong.
37. As a user, I want connection errors and timeouts raised as distinct client exceptions, so that I can retry them separately from API errors.
38. As a library consumer, I want the package to log through the standard logging module and never print, so that it fits into my application's logging.

### Convenience layer: projects

39. As a lab automator, I want to get a project by name or ID, so that scripts can refer to labs the way people do.
40. As a lab automator, I want "get or create" for projects by name, so that scripts can be re-run safely.
41. As a lab automator, I want to open, close, duplicate, lock/unlock and delete a project from a project handle, so that common lifecycle actions are one call.
42. As a lab automator, I want to start, stop, suspend and reload all nodes in a project, and optionally wait until they reach the target state, so that I know when the lab is ready.
43. As a lab automator, I want to create, list, restore and delete snapshots by name, so that I can reset labs between exercises.
44. As a trainer, I want a project inventory summary (nodes with type, status, console host/port and credentials, plus links described by node and port names), so that I can hand students or tools a readable view of the lab.

### Convenience layer: nodes

45. As a lab automator, I want to get a node by name within a project, with a clear error when the name is missing or ambiguous, so that I don't have to track UUIDs.
46. As a lab automator, I want to create a node from a template given by name or ID, with position and optional name, so that building a topology is short and readable.
47. As a lab automator, I want to start, stop, suspend, reload, isolate and un-isolate a node, so that I can simulate device failures (#353 asked for start/stop/reload).
48. As a lab automator, I want to wait for a node to reach a given status with a timeout, polling rather than fixed sleeps, so that scripts are both fast and reliable.
49. As a lab automator, I want to list a node's ports with their names, adapter/port numbers and whether each is in use, so that I can choose interfaces.
50. As a lab automator, I want to read a node's console connection details, so that I can hand them to telnet/SSH tooling.
51. As a lab automator, I want to duplicate a node and update its properties, so that I can scale out similar devices.

### Convenience layer: links and link quality

52. As a lab automator, I want to connect two nodes by port name (for example, "R1 Gi0/0" to "R2 Gi0/0"), so that I can build topologies the way I draw them.
53. As a lab automator, I want to connect two nodes on their next free compatible ports when I don't care which, so that quick labs need no port bookkeeping.
54. As a lab automator, I want a clear error when a port doesn't exist or is already in use, before the request is sent, so that mistakes are easy to diagnose.
55. As a lab automator, I want to find and delete the link between two named node ports, so that I can rewire a topology.
56. As a network tester, I want to suspend and resume a link (bring it down and up), so that I can test failover (#353: shut down / bring up interfaces).
57. As a network tester, I want to set link-quality impairments (delay, jitter, packet loss, corruption, drop frequency, BPF filters) on a running link and clear them again, checked against what the link supports, so that I can emulate degraded links (#353, Mininet-style).
58. As a network tester, I want to start and stop packet capture on a link and get the capture file, so that I can verify behaviour under test.

### Convenience layer: batch operations

59. As a lab automator, I want to create many nodes in one call, with results returned in the order I submitted them, so that I can build large topologies quickly.
60. As a lab automator, I want batches that rely on server-assigned default names created one at a time, and fully named batches created concurrently, so that names and console ports come out in a predictable order.
61. As a lab automator, I want to create many links, or start/stop many nodes, in one call, with a per-item result showing success or failure, so that one bad item doesn't hide the others.
62. As a user, I want a cap on how many batch requests run at once, so that I don't overload the server.

### Templates, images, appliances and computes

63. As an administrator, I want to list templates (with tag filtering), get a template by name, and create, update, duplicate and delete templates, so that I can manage the device catalogue from code.
64. As an administrator, I want to list, inspect, upload, delete and prune images and trigger automatic template installation, so that I can manage disk usage and onboarding.
65. As an administrator, I want to list appliances and install one as a template, so that I can provision vendor images programmatically.
66. As an administrator, I want to list, add, update, connect and remove computes and query their Docker images and VMs, so that I can manage multi-host deployments (a concern raised in #353).

### Access control and administration

67. As an administrator, I want to manage users, groups, roles, privileges, ACL entries and resource pools, so that I can run a shared multi-user GNS3 server from code.
68. As a user, I want to create, list, revoke, restore and delete my API keys, so that I can give automation its own credentials.
69. As an administrator, I want to read and update server settings and the IOU license, and reload or shut down the controller, so that I can manage the server itself.

### Maintenance and release

70. As a maintainer, I want the OpenAPI document for each supported GNS3 release committed in the repository, so that every generated line can be traced to its source.
71. As a maintainer, I want one command that regenerates models and endpoints from the committed document, so that updating for a new release is mechanical.
72. As a maintainer, I want CI to fail if the committed generated code differs from fresh generator output, so that nobody hand-edits generated files.
73. As a maintainer, I want an automated job that, for a new GNS3 release tag, exports that release's OpenAPI document, regenerates, and opens a pull request, so that the client keeps up with the server (as requested by the GNS3 maintainers).
74. As a maintainer, I want an explicit, reviewed list of excluded and hand-written operations, so that spec quirks are handled on purpose rather than by accident.
75. As a maintainer, I want CI to fail when a new operation appears in the spec that is neither generated nor listed as hand-written or excluded, so that coverage gaps are caught.
76. As a user, I want the package version to state which GNS3 server version it was generated against, so that I can choose a compatible release.
77. As a contributor, I want linting, formatting, type checking and tests to run in CI, so that contributions stay consistent.
78. As a user, I want reference docs generated from docstrings plus a tutorial covering the convenience layer, so that I can learn the library quickly.
79. As a type-checker user, I want the package to ship type information, so that strict type checking works in my code.

## Implementation Decisions

### Scope and targets

- **Server API:** GNS3 controller API v3 only. v2 is not supported; gns3fy remains the option for v2.
- **Spec source:** The OpenAPI document (OpenAPI 3.1) exported from the gns3-server application for a given release. It is committed in the repository and is the single source for generation.
- **Ownership:** Developed under the author's GitHub account first, with the GNS3 organization as the intended long-term home.

### Architecture: four modules

**1. Transport and session core** (hand-written):
- Owns the httpx client (sync) and async client.
- Handles base URL, timeouts, TLS configuration, authentication headers and token lifecycle: password login through the JSON authenticate endpoint, refresh through the refresh endpoint, and logout.
- Maps errors to exceptions.
- The generated layer only builds request descriptions (method, path, query, body, expected response type). The transport executes them, which lets one generated definition serve both sync and async clients.

**2. Generated models:**
- Produced with datamodel-code-generator using the `msgspec.Struct` output type.
- Settings:
  - unknown fields are ignored on decode (forward compatibility)
  - an unset-vs-null distinction is used for update/partial models
  - a name map turns server-internal names (such as the module-path-prefixed `LinkType` and `Category` duplicates) into clean public names
- Open-ended schemas (notably `Template`, and node `properties`) must keep extra fields rather than dropping them.
- The appliance union, discriminated by the integer `registry_version`, must decode to the right model.

**3. Generated endpoints:**
- Produced by a small in-house generator (a template over the spec's `paths`). Off-the-shelf client generators are not used.
- One namespace per API tag (projects, nodes, links, …), exposed on both `Client` and `AsyncClient`.
- Methods get readable names derived from the operation summary and tag, not the raw `operationId`. Path parameters become positional arguments; query parameters become keyword arguments.
- Responses that the spec leaves untyped are returned as dictionaries.

**4. Convenience layer** (hand-written):
- Project, node and link **handles**: lightweight objects holding the client, their IDs and the last fetched data.
- Explicit `refresh` instead of an automatic sync cache. Neither gns3fy-style self-mutating records nor virl2-client-style auto-sync.
- Provides:
  - name lookup
  - get-or-create
  - port-name and next-free-port linking with an in-use check
  - link suspend/resume and link-quality helpers on top of the link `filters` field, validated against the link's available filters
  - polling `wait_for` helpers with timeout and interval
  - snapshot-by-name helpers
  - project inventory summary
  - batch helpers
- Built only on the public generated layer, so it doesn't change when the server spec changes unless a new feature is wanted.

### Hand-written low-level operations

The spec describes these incorrectly or incompletely for code generation:
- **Mislabelled as JSON** but actually binary or streaming: project export, project/node file reads, capture stream and capture file download, controller and project notifications.
- **Missing a request body** in the spec: image upload, symbol upload, project/node file writes, project import.

All of these get hand-written sync and async methods in the same namespaces, using streaming request and response bodies.

### Excluded operations

Not generated or exposed:
- the AI/copilot and MCP placeholder routes (which also reuse one `operationId` across methods)
- deprecated compute-forwarding routes
- deprecated GNS3 VM routes

The exclusion list is explicit and reviewed.

### Batch semantics

- Results come back in submission order, one result object per item with a success or failure status.
- Concurrency is capped and configurable.
- Node creation falls back to one-at-a-time whenever any item lacks an explicit name, because the controller assigns default names and console ports in arrival order. (Lesson from the gns3-copilot implementation.)

### Error model

- A base package exception, with:
  - an API error carrying the HTTP status, the server `message` (from the `ErrorMessage` schema) and the raw response
  - subclasses for 401, 403, 404, 409, 422 (exposing `HTTPValidationError` details) and 5xx
- Separate transport errors for connection failures and timeouts.
- Nothing is swallowed or printed.

### Security defaults

- TLS verification on by default, with a logged warning when it is disabled; plain HTTP allowed explicitly.
- Credentials can come from arguments or environment variables.
- Tokens are never logged.

### Regeneration pipeline

- A single regenerate command (export spec → generate models → generate endpoints → format).
- A CI drift check.
- A release-triggered automation that opens a pull request with the new spec and regenerated code.
- The supported server version is recorded in package metadata.

### Tooling

- Modern packaging with uv.
- ruff for linting and formatting, mypy or pyright for type checks, pytest.
- Ships type information.
- mkdocs documentation.

### Prior-art lessons adopted

- From gns3fy: name- and port-name-based ergonomics.
- From gns3-paw: v3-first layering and the CI/doc setup.
- From the gns3-copilot client: JWT handling, error extraction, timeouts, batch ordering.
- From virl2-client: handle-based objects and convergence waiting.

### Explicitly rejected

- The old openapi-generator Python output.
- Error dictionaries as return values.
- `print()` in library code.
- Global suppression of TLS warnings.
- Sleep-based waits.
- A new HTTP session per call.
- Pydantic v1.

## Testing Decisions

### What makes a good test

- It exercises public behaviour: what a user calls, what goes over the wire, and what comes back or is raised.
- It does not assert on internal helpers, generated-code structure or private state, so refactors and regenerations don't break tests.
- Every behaviour test runs against **both** the sync and async clients from a single test definition.

### Seam 1 (primary): the public clients over a mocked HTTP transport

- httpx's mock transport replaces the network.
- Tests assert:
  - the method, path, query and body sent
  - typed results decoded from realistic JSON fixtures
  - exceptions raised for each error status
  - token login, refresh and logout flows
  - TLS and timeout configuration being applied
- The convenience layer is tested through the same seam, using fixture topologies:
  - name lookup and ambiguity errors
  - port-name linking and in-use detection
  - next-free-port selection
  - `wait_for` with simulated status changes and timeouts
  - batch ordering and per-item failure reporting
  - sequential fallback for unnamed node batches
  - link-quality helpers
  - snapshot-by-name
  - inventory summary
- Streaming and binary methods are tested here too (export/import, uploads, file read/write, capture download, notification iteration).

### Seam 2: the code generator

Given the committed spec:
- generated code imports and type-checks
- every non-excluded, non-hand-written operation has a method
- the exclusion and hand-written lists cover the rest
- decoding representative payloads for tricky schemas works:
  - open-ended `Template` keeps extra fields
  - the appliance union picks the right variant
  - partial updates omit unset fields
  - unknown fields are ignored

The CI drift check (regenerate and compare) is part of this seam.

### Seam 3 (optional, marked and skippable): integration against a real GNS3 3.x server in Docker

- Exercises a full lab lifecycle: create project → create nodes from built-in templates → link → start → wait → capture → snapshot → restore → export → delete.
- Runs on demand or on a schedule, not on every push.

### Prior art

- gns3fy's requests-mock fixtures with JSON topology data: the model for fixture-driven tests.
- gns3-paw's split of tests by layer: kept conceptually, but MagicMock-based tests are replaced by transport-level mocks.

## Out of Scope

- A command-line interface. Planned for later; not a priority now.
- GNS3 API v2 support.
- Calling compute nodes' own API directly; only the controller API is targeted.
- Interacting with devices through their consoles (telnet/SSH/WebSocket sessions, sending commands, pyATS-style integrations) and per-guest-OS config import/export plugins (the 2015 suggestion in #353). The client exposes console connection details and node files, which such tools can build on.
- The AI/copilot and MCP endpoints, and LLM-specific conveniences such as response field filtering or compact link formats.
- An automatically synchronised local object cache, or event-driven state updates in the convenience layer (waits use polling).
- Web Wireshark and traffic-marker replay features beyond exposing their generated low-level methods.

## Further Notes

### Background

- **Maintainer direction from GNS3/gns3-server#353:**
  - generate the client from the OpenAPI spec
  - regenerate on each GNS3 release, not at install time
  - keep it simple and clean
  - virl2-client was suggested as inspiration
- **Size of the v3 spec** (checked against the 3.1 export): about 187 operations across about 20 tags, about 150 schemas, plus about 15 placeholder AI/MCP routes to exclude.
- **Expected generated size:** a few thousand lines. Models are plain declarations, and validation, serialization and HTTP live in msgspec and httpx. That compares with ~183k lines in the previous attempt. The real figure is to be confirmed in the first prototype.

### Open items to settle early

- **Package name** (gns3fy, gns3client and gns3-paw are taken or confusing).
- **Minimum Python version.**
- **Licence.** gns3fy is MIT; gns3-server and the earlier MVP are GPL-3.0.
- **Exact token-expiry handling** (proactive refresh vs refresh on 401).
- **Whether API keys can be sent directly as bearer credentials** or must be exchanged for a JWT first.

### Prototype first

The first piece of work should be a prototype that runs the model generator on the committed spec. It should confirm:
- the open-ended model behaviour
- the unset-vs-null handling
- the appliance union
- the name mapping
- the actual generated size

These come before any convenience-layer work.

### Upstream fixes

Several spec defects should also be reported upstream so the hand-written list can shrink over time:
- duplicate `operationId`s
- binary endpoints declared as JSON
- missing request bodies
- a list endpoint that returns 201
