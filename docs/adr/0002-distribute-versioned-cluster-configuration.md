---
status: proposed
date: 2026-11-11
---

# ADR-0002: Distribute versioned cluster configuration

## Context and Problem Statement

Users with several T3 environments need a deliberate way to keep selected configuration, skills,
and tool definitions consistent without treating those machines as one server. Today, linking
an environment allows agents to address another environment; it does not establish common
configuration ownership or turn the environments into interchangeable execution resources.

In this proposal, a **cluster** is an explicitly enrolled set of independently durable environments
that accept selected desired configuration from one authority. Membership is a configuration
relationship, not a filesystem boundary, an execution grant, or a user account. A machine can
remain linked without joining a cluster. Initially an environment can belong to at most one
active cluster, avoiding overlapping writers for the same configuration.

An environment retains its identity, database, projects, credentials, and authoritative thread
history. A **thread** remains a durable conversation; a **run** is one provider execution. The
user-facing **execution session** defined by
[ADR-0001](0001-use-session-scoped-execution-targets.md) binds a thread/workspace lineage to a
runner and an immutable effective configuration snapshot; it is not an OAuth login or a provider's
native session. Clustering is optional input to that snapshot. Neither execution targets nor
clustering depends on the other, and neither requires a cloud sandbox.

### Existing behavior and evidence

These are current constraints, not claims that this proposal is implemented:

- [Remote architecture](../internals/remote.md#peer-links) and
  [PeerLinks](../../apps/server/src/peer/PeerLinks.ts) describe OAuth MCP peer links. The linking
  environment stores the receiving environment's token locally; unlinking forgets it, whereas
  revocation is performed at the receiver. Forwarding checks caller modes and destination access.
  [RemoteDelegation](../../apps/server/src/peer/RemoteDelegation.ts) creates an ordinary thread
  owned by the remote environment. This is routing, not isolation or home-owned remote execution.
- [Settings ownership](../internals/overview.md#settings-ownership) and
  [settings guidance](../user/project-settings.md) distinguish device preferences, environment
  defaults, and project overrides. **All environments** bulk-edits connected, loaded servers;
  offline and future environments do not inherit a synchronized global default.
  [ServerSettingsService](../../apps/server/src/serverSettings.ts) reads and atomically writes
  the owning server's settings file and vets changes against local managed policy.
- [Managed policy](../user/managed-policy.md) is deployed per machine and loaded at server startup;
  it is not cluster configuration. Its defaults, locks, and limits must continue to apply.
  [Environment authentication](../internals/environment-auth.md) keeps sessions, signing keys,
  grants, and revocations environment-local, including when transport or endpoint changes.
- Skill inventory is provider/workspace-specific. For example,
  [Claude discovery](../../apps/server/src/provider/Drivers/ClaudeSkills.ts) reads local user,
  project, additional-directory, and enabled-plugin roots with native precedence;
  [Codex discovery](../../apps/server/src/provider/CodexProvider.ts) requests `skills/list` for
  the workspace. [ProviderRegistry](../../apps/server/src/provider/ProviderRegistry.ts) maintains
  machine and workspace snapshots. These are discovery paths, not cross-machine installers.
- T3's built-in MCP tools are registered by
  [McpHttpServer](../../apps/server/src/mcp/McpHttpServer.ts), with authorization/routing declared
  through [McpToolAccess](../../apps/server/src/mcp/McpToolAccess.ts). Provider adapters inject
  thread-scoped T3 access; for example, the
  [Codex adapter](../../apps/server/src/orchestration-v2/Adapters/CodexAdapterV2.ts) overrides
  T3's MCP registrations without replacing other launch-configured servers. Discovery,
  registration, configuration distribution, and permission to execute are different operations.
- [Secret settings resolution](../../apps/server/src/secrets/settingsSecretValue.ts) resolves
  references at the owning environment and rejects unresolved required secrets rather than
  silently changing credentials. [ScheduledTaskService](../../apps/server/src/scheduledTasks/ScheduledTaskService.ts)
  launches or messages threads through server services. A schedule is not currently a cluster job.

The reviewed peer, settings, provider discovery, and tool registration paths do not establish
versioned cluster propagation. This ADR proposes that missing relationship; it does not describe
an existing sync feature.

## Decision Drivers

- Make configuration ownership and convergence explainable when machines are offline or disagree.
- Preserve independent environments, existing clients/routes/peers, and home-owned conversation
  history without copying databases or pretending execution can transparently fail over.
- Distribute reviewable configuration and artifacts without spreading credentials, executable
  trust, filesystem assumptions, or ambient permissions.
- Admit work against an actual compatible, acknowledged snapshot, not an optimistic sync label.
- Prefer one writer and explicit recovery over automatic elections or multiwriter merge rules.

## Considered Options

- Full environment state/database replication.
- Ad hoc bilateral or multiwriter synchronization of settings and directories.
- Single-authority, versioned desired-configuration distribution with local application.

## Decision Outcome

Chosen option: **Single-authority, versioned desired-configuration distribution with local
application**, because it makes the shared intent durable while leaving state, trust decisions,
and actual execution with their existing owners.

Everything below is proposed behavior. Accepting this ADR would not itself ship these capabilities.

### Independent authority boundaries

The cluster authority is one nominated environment. It owns membership records and the ordered
history of desired-configuration publications. Authorized administrators write through that
server; connecting several clients does not create several authorities. Members pull authenticated
publications and return application reports. Notifications may prompt a pull but are not evidence
that a member applied anything. There is no dependency on an always-open client.

| Relationship                   | What it permits                                                           | What it does not permit                                                                    |
| ------------------------------ | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Transport/pairing or peer link | Reach an environment using its existing approved access                   | Join a cluster, publish configuration, or gain additional scopes                           |
| Cluster enrollment             | Receive the selected profile and publish that member's application status | Execute workloads, administer another member, or read its secrets                          |
| Configuration-author grant     | Propose/publish allowlisted desired configuration at the authority        | Approve executable trust or change another environment's managed policy                    |
| Execution grant                | Admit a particular caller/session to an explicitly capable target         | Move the home thread or enable transitive use of that target's peers                       |
| Secret grant                   | Resolve named references for approved consumers at an approved target     | Export credential values, grant access to all referenced secrets, or share provider logins |

New cluster operations require purpose-specific authorization; an existing MCP link token or
ordinary settings-write permission must not silently acquire cluster administration. Publication
and membership changes have authenticated actors and durable receipts. Artifact activation also
requires the receiving environment's approval. A full-access agent or OS administrator can already
act as the server's user; cluster authorization is not a sandbox against a compromised member.

### Enrollment, removal, and authority loss

- **Create/add:** a local administrator creates the cluster at the authority and chooses the
  distributable profile. An administrator of each joining environment explicitly approves the
  authority's identity/signing-key fingerprint, the selected scopes of configuration, and the
  initial revision. The authority separately accepts that stable environment identity and member
  authentication key. Invitations are short-lived, one-use, and bound to that target and cluster;
  knowing a URL, repository identity, or peer-link token is insufficient. A descriptor's claimed
  environment ID is not cryptographic proof. Existing authenticated encrypted routes may carry
  the protocol, but enrollment pins the authority key independently of route discovery.
- **Member identity:** membership binds the environment, not the browser, desktop installation,
  or network address. A membership generation and per-member enrollment generation distinguish
  fresh enrollment from an old grant. A member authenticates as itself, can report only its own
  status, and cannot publish revisions or enroll other members. Duplicate/cloned member keys
  must be resolved by explicit reenrollment, not automatic identity merging.
- **Leave/remove/revoke:** either the member's local administrator can leave or the authority can
  revoke membership. The authority persists a tombstone, advances membership generation, stops
  delivery and membership-lease renewal for that member, and rejects its old grants. Reachable members stop
  cluster admission and withdraw cluster-managed projections. Rejoining is a new explicit
  approval with a new enrollment generation; restoring an old link or settings backup cannot
  resurrect membership. Removing a cluster membership does not unlink peers, delete threads,
  remove unmanaged files, revoke unrelated client sessions, or revoke unrelated execution grants.
  Those are separate operations, offered explicitly where also wanted.
- **Offline/rejoin:** a member retains its last verified configuration and reports no new success
  while unreachable. On reconnection it first obtains current membership, then reconciles to the
  current revision; it does not replay every intermediate installation. A revoked member cannot
  reconnect with its previous enrollment. Tombstones and revision high-water marks survive restart
  and are not ordinary cache entries. Loss of these records requires reenrollment.
- **Admission freshness:** the authority issues finite, signed membership leases bound to cluster,
  member, enrollment generation, authority epoch, observed profile head, and revocation frontier.
  Members renew them in the background, not through a per-session execution-routing hop. The
  authority fixes and exposes the maximum duration at enrollment; members/targets may shorten,
  never extend it. A home/executor may admit sessions and runs with a still-fresh cached lease,
  verified profile, and independent execution/secret grants. This permits bounded stale operation
  during a partition. Once newer desired state is known, new sessions require it unless an explicit
  still-authorized pin was selected. Renewal checks current membership and revocations. Expiry
  blocks further cluster-bound session/run/tool admission and requests affected runner cancellation.
  An expired lease cannot be renewed from cached data; clock uncertainty or rollback fails closed,
  and restart requires a fresh check. An explicitly standalone session can still use local
  configuration and independently installed artifacts, but cannot silently reuse the cluster
  profile or cluster artifact mounts.
- **Revocation limits:** an online member enforces revocation when received: deny affected new
  session/run/tool admissions and request runner cancellation. An offline member can enforce it
  only at lease expiry. Cancellation of an already-running process is best effort and cannot
  undo filesystem writes or external effects. A hostile or compromised machine can retain copied
  configuration/artifact bytes and ignore cancellation. Therefore neither offline deletion nor
  immediate remote credential revocation is promised. Secret issuers and execution targets
  enforce their own grants/expiry independently, potentially sooner than the cluster lease.
- **Authority unavailable:** members retain last-applied status and may admit/continue cluster work
  only while their membership leases and independent grants remain valid. There are no new
  publications, enrollments, or lease renewals; after expiry cluster-bound work is blocked. There
  is no automatic promotion. Standalone environments and unrelated peer links remain usable.
  Thread/history authority is unaffected.
- **Transfer/recovery:** planned transfer fences the old publisher, exports only the cluster's
  configuration/membership history, and publishes a signed successor key/authority epoch that
  members explicitly accept. It does not export the environment database or signing private key.
  Without the old authority, recovery requires local administrator repinning at every retained
  member to a new authority epoch/key after fencing the old instance; an unreachable member is
  not assumed recovered. A returned old authority cannot supersede a member's accepted epoch.
  Divergent restored authorities trigger a conflict, not an election or last-writer-wins repair.

### Common configuration and local resolution

Distribute a typed, allowlisted profile, not `settings.json`, provider homes, shell environments,
project databases, or arbitrary dotfile trees. The publication declares schema version, required
capabilities, common values, allowed override scopes, and optional member-specific desired values.
A stable logical provider/tool/skill ID is distinct from a machine's installed instance or path.
An explicit local mapping binds them; matching a display name must not select an account or binary.

Each enrolled setting has exactly one cluster owner. Common values supply the cluster-owned
virtual environment-default layer; authority-authored per-member values replace common values,
then explicit local member overrides replace those only where the profile permits. A member edit
either creates a visible allowed local override or is refused as a cluster-owned conflict. It
never secretly republishes the cluster value. Member overrides are local records with their own
version/digest and reported provenance, not competing cluster publications. Their writes also
require an expected local version and reject stale edits instead of silently losing changes.

For cluster-owned keys, retained pre-enrollment environment values are dormant fallback rather
than being overwritten. For unmanaged keys, existing local resolution is unchanged. Repository
`t3.json` and built-in defaults remain fallback; eligible project overrides remain above the
virtual environment defaults. Existing managed-policy defaults fill unset values and its locks
and limits remain authoritative over every layer. Cluster configuration cannot install, edit, or
relax managed policy. A locally forbidden or differently overridden value is reported as such;
matching publication revisions alone does not mean effective configurations match. Work requiring
an exact common value must reject a member whose effective snapshot differs.

Machine paths, provider binary/home locations, workspace roots, browser profiles, device access,
client connection catalogs, and UI preferences stay local. Tool definitions use logical workspace,
artifact, and executable references; each member explicitly resolves them with its OS/architecture
adapter and local path rules. No Linux path or shell fragment is mechanically translated into a
Windows command. Unsupported bindings produce a blocked status, not a guessed fallback. Project
configuration can use an explicitly mapped logical project selector, never another environment's
raw project ID or an assumption that equal repository identities share filesystem permissions.

### Revisions, conflicts, application, and rollback

The authority serializes publications with compare-and-swap against the expected head. Each
immutable publication contains the cluster ID, authority epoch, increasing revision, parent digest,
schema/capability requirements, membership generation, typed profile, and artifact digests. It is
signed by the pinned authority key. A canonical encoding and named cryptographic hash algorithm
produce its content digest; a hash detects corruption, while the signature and pinned authority
establish origin. Revisions contain secret references, never resolved credentials.

Publishing commands carry idempotency keys: retrying the same actor/key/payload returns its
recorded revision; reusing the key for another payload or writing against a stale head returns a
conflict with the current revision. There is no timestamp merge. A member verifies signature,
epoch, membership, schema, and the signed parent chain from its accepted head or enrollment
anchor. Missing chain metadata is fetched before applying a later full snapshot. Older revisions
cannot replace its high-water mark. The same epoch/revision with another digest freezes cluster
application/admission and reports a security conflict for explicit administrator recovery.

Members stage the complete selected profile and immutable artifacts, validate mappings and local
policy, obtain necessary trust approvals, and then atomically switch one local active-profile
pointer. An application key includes publication digest, local override digest, policy fingerprint,
and adapter/projection versions. Repeated delivery is idempotent. A crash before the switch keeps
the previous active profile; a crash afterwards recovers the switched profile and can resend its
acknowledgement without reinstalling. There is no transaction across members and no success claim
for partially staged configuration. Providers without an isolated projection mechanism are
ineligible for that profile rather than having live native configuration rewritten mid-session.

The authority exposes per-member **desired**, **applied**, and **error** state: desired revision
and digest; last applied revision plus effective snapshot digest; pending/staging/approval/blocked
or failed reason; last contact time; and local masks/overrides. Acknowledgement means locally
activated, not merely downloaded. Failure retains the prior applied revision and reports the
failed desired one. Offline status is last-known information, never a synthetic current success.
Status and diagnostics exclude secret values and credential-bearing command output.

Rollback publishes a new, increasing revision referencing earlier reviewed content; it never
rewinds revision or membership high-water marks. Revalidation and current trust/secret/execution
grants still apply. Rollback is not deletion of history or reversal of a tool's external effects.
A local emergency suspension withdraws the profile and reports blocked status; it does not create
an unannounced cluster fork.

### Skill and tool artifacts

A profile can reference reviewed, content-addressed skill bundles and declarative external-tool
recipes. It cannot distribute replacement T3 MCP implementations; built-in T3 tool availability
continues to depend on the installed server and authorization. Provider discovery still reads what
that member/provider actually supports. Applying a profile does not manufacture native support.

- **Publish/install:** select artifacts explicitly and publish a manifest with exact content hash,
  provenance, file inventory, provider/OS/architecture requirements, and declared execution/network
  requirements. Pin dependencies and immutable package versions; mutable branches or `latest`
  are not reproducible artifacts. Never export credential files, native provider homes, caches,
  shell environments, or an entire skill parent directory by implication. Publication review must
  inspect the selected content for accidental secrets; a signature is not proof that text is safe.
- **Verify/trust:** download into a quarantine area without executing installers, hooks, or tools.
  Verify every file against the manifest and reject absolute/traversal paths, escaping symlinks,
  device files, and OS name/case collisions. Installation/execution hooks are executable content
  requiring explicit review and local authorization. A receiving administrator approves the exact
  artifact digest, provider projection, requested capabilities, and executable recipe before
  activation. Skill instructions also require trust approval: text can induce unsafe actions even
  without an executable bit. Membership or a trusted transport is not that approval. A changed
  digest or broader capability request requires new approval; trust is never inferred from a name.
- **Project safely:** use a cluster-owned namespace and immutable directories, with provider-specific
  session overlays/isolated homes only where supported. Never overwrite an unrelated native skill,
  `.mcp.json`, plugin install record, or user tool definition. Name collisions with existing
  discovered resources block the binding until the administrator explicitly selects an alias or
  resolves precedence. Local unmanaged edits stay local. An edited managed projection is drift:
  stop advertising it as applied, preserve/quarantine the edited copy, and require explicit
  restore or adoption as a newly reviewed artifact; do not upload it or overwrite it silently.
- **Tools:** a recipe names a pinned artifact/executable, structured arguments, logical directories,
  typed secret-reference slots, and an allowed remote endpoint where applicable. It does not copy
  raw environment/launch-argument strings, auth headers, embedded URL credentials, or secret-bearing
  query strings. Local resolution produces command/arguments without shell interpolation. Fetching
  dependencies or running approved install scripts occurs only after approval and within the
  separately allowed installation boundary. An unavailable runtime or unapproved network/tool
  capability blocks installation/use; it does not trigger a wider global install.
- **Update/delete:** ordinary changes stage a new digest and affect new sessions. Removal is an
  explicit tombstone in a new revision, withdrawing that artifact from new snapshots; it deletes
  only cluster-owned projections, never user directories or provider-native installations.
  Content pinned by active sessions is retained until release, then garbage-collected with its
  cluster provenance. Leaving withdraws projections under the same ownership rule. Security
  revocation separately denies new admissions/uses and requests affected sessions stop under the
  lease rules, even when old bytes remain pinned or an old revision is requested. Routine removal
  and rollback do not bypass that denylist.

### Secrets and workload placement

Profiles contain logical secret references and required consumers, not credentials, provider OAuth
state, pairing/session tokens, private keys, browser cookies, or secret-provider bootstrap material.
Reference names are still potentially sensitive metadata and are disclosed only to enrolled
recipients. Each environment explicitly maps a reference to its own secret provider and approves
the consumer, target, and lifetime. Identical reference names do not imply identical accounts.
A missing/denied reference blocks the consumer, with no ambient environment-variable, home-machine,
or alternate-account fallback. A receiver does not return resolved values in application reports.
If a separate secret service grants short-lived access to a runner, that is an independent,
audited grant from that service, never a consequence of configuration publication or membership.
Revocation must happen at the relevant issuer; deleting a reference cannot unlearn a credential
already disclosed to a process.

Members may advertise versioned, time-bounded capabilities: provider/native protocol support,
platform, compatible artifact/runtime availability, resource availability, and applied snapshot.
Advertisements contain no credentials and are hints, not authorization. Workload placement must
intersect requested capabilities, fresh target capabilities, caller/link permission ceilings,
explicit execution grants, local managed policy, artifact approvals, and required secret grants.
The target rechecks this intersection and acknowledges the exact snapshot before admission;
a stale advertisement cannot authorize a run. The configuration authority is neither the workload
owner nor a required execution-routing hop. Multiple home environments may schedule independently;
their executors validate fresh membership evidence where relevant plus their own grants.
Membership alone never adds a machine to the set of executable targets.

[ADR-0001](0001-use-session-scoped-execution-targets.md) owns session-scoped placement and runner
failure semantics. The home environment remains the only authoritative writer of thread/history,
including when an explicitly capable peer supplies the runner. Existing remote delegation remains
remote-owned work; it is not silently reinterpreted as that execution capability. No member
proxies permission through another peer merely because both are in the cluster.

At admission, persist the execution session's immutable effective snapshot reference as revision,
digest, and schema version, including resolved nonsecret configuration, local binding/projection
versions, artifact hashes, and policy provenance. The runner verifies and acknowledges it. It
contains no secret values; secret grants are checked separately. Ordinary updates do not mutate
that session's view. To adopt another snapshot, explicitly stop/rebind at a safe boundary under
ADR-0001. Revocation is a live authorization check, not a mutable configuration snapshot. A tighter
local policy can suspend execution immediately without rewriting historical snapshot evidence.

The snapshot pins defaults, artifacts, bindings and policy, not every interactive run choice.
Explicit provider/model/mode commands remain available within its allowed envelope and current
grants as defined in ADR-0001; they do not mutate the published profile or allocate new compute.

### Entry points and compatibility

Web, hosted web, desktop, and mobile must use the same server-side membership/publication/status
services and scoped contracts. The selected environment is always explicit; losing it must not
redirect an edit or workload to a different machine. Clients show cluster-common values, local
provenance, pending approvals, stale/offline state, conflicts, and removal status. Desktop hosting
a server does not make the desktop client the authority. SSH, direct, relay, and local routes
remain transport choices, and clients/peers that never opt in retain their current behavior.

Agent MCP operations use those services with purpose-specific grants and expected revisions;
agents cannot self-enroll machines, approve their own artifacts, or convert configuration-write
access into execution/secret permission. Existing read-only/mode ceilings and link-origin fences
remain applicable. Agent-generated content is not an administrator's trust approval.

Scheduled and webhook tasks remain stored and dispatched by their home environment. Recurrence,
webhook secrets, pending deliveries, and task records are not replicated, preventing duplicate
triggers on every member. At dispatch they resolve an eligible, acknowledged snapshot and target
through the same admission service as an interactive client or agent. Unavailability blocks or
fails according to the task's explicit policy; it does not clone the task or silently select
another account, target, or older snapshot. A continuing session retains its pinned snapshot;
a fresh session resolves the then-current desired revision unless an explicit still-authorized
pin was chosen.

Negotiate cluster protocol, document schema, artifact format, and individual capabilities
independently of app version. Members reject unknown required fields/features; explicitly optional
extensions can be ignored only when the publication declares that omission safe. The authority
must not claim convergence after dropping a required tool or constraint. Unsupported members
remain linked and retain their prior local/applied configuration, but are blocked for workloads
requiring the rejected revision. A downlevel client can still use existing APIs; it must not edit
or round-trip cluster-owned fields it cannot represent. Existing client and peer credentials gain
no new permissions during upgrade.

### Consequences

- **Positive:** a single durable desired revision gives installs, drift, rollback, and disagreement
  an explicit owner without coupling configuration to conversation storage.
- **Positive:** independently verified application and snapshots prevent an offline or incompatible
  machine being presented as synchronized or ready merely because it is linked.
- **Positive:** enrollment, executable trust, workload admission, and secret access can be revoked
  independently without deleting the user's conversations or breaking unrelated connections.
- **Negative:** authority outages eventually block new cluster-bound sessions and stop leased
  sessions. Before lease expiry, disconnected machines may still act on stale membership. This
  consciously trades immediate revocation for bounded offline operation without routing every
  execution through the configuration publisher.
- **Negative:** installation approvals, platform mappings, revision history, artifact storage,
  reconciliation/status, and provider-specific isolated projections add operational work.
  Some providers will be ineligible until their projection boundary can preserve snapshots.
- **Negative:** one writer and explicit per-member disaster recovery do not support convenient
  offline editing, automatic failover, or instant recovery of unreachable machines.
- **Negative:** configuration or artifact bytes already received cannot be remotely recalled.
  Hashes and signatures do not make executable code or skill instructions harmless.
- **Neutral:** members may intentionally have different effective snapshots through allowed
  overrides and policy. Convergence means the same declared intent is accounted for, not that
  every machine has identical providers, paths, credentials, or permissions.

## Pros and Cons of the Options

### Full environment state/database replication

- Good, because it could present one inventory and simplify reading the same durable data from
  several nodes if a distributed persistence and consistency model were built first.
- Bad, because it mixes conversation ownership, execution recovery, authentication, credentials,
  and machine-local paths into a configuration problem. Replicating an event log alone is not a
  valid database backup/rebuild model; the current
  [retention model](../internals/overview.md#durable-intent-and-side-effects) prunes older events.
- Bad, because competing schedulers, command receipts, providers, and filesystem effects require
  coordinated ownership and fencing. A copied SQLite file does not supply these properties.
- Bad, because it expands disclosure and revocation boundaries and requires a separately designed
  distributed database/HA product. That is neither necessary nor promised here.

### Ad hoc bilateral or multiwriter synchronization

- Good, because two machines could exchange small settings or directory changes without an
  always-available nominated authority; local editing feels familiar.
- Bad, because offline concurrent edits, deletion, rename, provider-native precedence, and mutable
  files require merge semantics that are unsafe for permissions and executable configuration.
- Bad, because pairwise trust can become transitive configuration authority, with no clear answer
  to who may publish, resolve conflicts, revoke an artifact, or stop a removed member reappearing.
- Bad, because last-writer-wins clocks hide intent, while general multiwriter conflict resolution
  imposes complexity without a requirement for several independent configuration owners.

### Single-authority, versioned desired-configuration distribution with local application

- Good, because ordered immutable intent, explicit local overrides, ownership-safe projections,
  and independent applied status make partial failure explainable and rollback reviewable.
- Good, because existing environments can adopt only the configuration relationship they want;
  peers and session-scoped targets remain useful without enrolling.
- Bad, because authority availability and administration become visible constraints, and desired
  state can remain blocked by local policy, missing dependencies, approval, or version skew.
- Bad, because no protocol can revoke already-copied bytes or give a machine capabilities its
  provider/OS does not support. Those limitations remain user-visible rather than hidden.

## More Information

### Migration strategy

Start with standalone behavior unchanged and advertise cluster support as absent until implemented.
Enrollment previews the selected nonsecret common profile, differing member values, unmanaged
resources, project masks, and local policy; it imports nothing automatically from existing links
or bulk settings targets. Preserve pre-enrollment settings for detach, and place the cluster layer
and artifact projections separately so removal does not require reconstructing a copied settings
file. Existing native skills/tool configurations stay unmanaged unless explicitly adopted as
reviewed artifacts. No database, environment identity, auth grant, or thread migration is needed.

Roll out negotiated read/status and enrollment support before allowing profile activation.
Enable a profile only for members/adapters that can validate, stage, atomically apply, and report
it; an older member remains a valid ordinary peer. Every relevant client and agent entry point
must represent pending/conflicting/revoked state before exposing mutations. Activation for
execution requires the snapshot/admission contract, not a UI-only sync flag. Disabling clustering
withdraws only cluster-owned projections, restores normal local resolution, and leaves existing
thread/history ownership and noncluster connections intact.

### Acceptance scenarios for a future implementation

These are design acceptance criteria, not tests exercised by this documentation change.

1. **Opt-in:** linking laptop A to workstation B grants no membership, config publication, artifact
   installation, runner access, or secrets. Enrolling B still grants none of the latter three.
2. **Offline install/rejoin:** B misses several revisions. A shows B's old applied digest and last
   contact. B validates membership and the signed chain on return, stages only the current profile,
   and reports success only after activation; a removed member must reenroll instead.
3. **Conflict/replay:** two clients publish against the same head. One succeeds; the other receives
   a stale-head conflict. Retrying the successful command returns its original revision. A forged
   same-revision/different-hash publication freezes application, and an old signed revision cannot
   rewind state. Rebooting cannot discard tombstones or high-water marks.
4. **Crash/rollback:** B loses power during staging and retains its previous active profile. Losing
   the acknowledgement after activation does not reinstall or execute hooks on retry. Republishing
   an earlier payload as a new revision rolls future sessions back without mutating running ones
   or restoring a revoked artifact.
5. **Platform/policy skew:** a Windows member cannot resolve a Linux-only tool or is restricted by
   managed policy. It reports the exact blocking binding/constraint and prior applied state; it
   neither runs a translated shell command nor drops the requirement while claiming convergence.
6. **Artifact trust/drift/deletion:** a signed skill with executable helpers does not run before
   local approval. A hash mismatch, escaping path, native-name collision, or local projection edit
   blocks activation. A removal withdraws only cluster-owned projections; active snapshots retain
   their bytes until release unless security revocation suspends their use. Local native skills
   and tools are untouched.
7. **Secret isolation:** a reference valid on A but unavailable/denied on B blocks that tool on B.
   It does not copy A's provider login, inherit B's ambient credentials, or include the value in
   snapshots/status. A secret issuer's revocation wins over a still-valid configuration revision.
8. **Revocation under partition:** A revokes B while B is disconnected. A immediately refuses
   membership-lease renewal and reachable targets reject the received revocation. A disconnected
   B may still admit cluster work only within its existing lease and independent grants; expiry
   blocks further session/run/tool admission. Reconnection cannot revive the old enrollment.
   The UI does not claim that B's files were erased or a running external operation was undone.
9. **Authority loss/recovery:** with A down, unrelated local/direct/SSH/relay clients and ordinary
   peers still work. Home/executor admission needs no per-session call to A while its evidence
   remains fresh; expiry blocks cluster-profile work. No member elects itself publisher. Explicit
   successor repinning restores renewal; an old A with an older epoch cannot override that choice.
10. **Every entry point:** web/desktop/mobile, an authorized agent, and a home-owned scheduled task
    receive the same conflict, missing-approval, revoked-member, or unavailable-authority outcome
    for the same operation. No surface bypasses the target's policy or silently selects another
    environment. A timer firing during an outage creates no duplicate schedule on another member.
11. **Independent execution:** a noncluster environment can use a session-scoped runner, and a
    cluster can distribute configuration without offering any runner. When a capable peer is
    explicitly selected, the home retains thread/history and the peer acknowledges the session's
    revision/digest/schema before admission. A runner failure follows ADR-0001, not database
    failover or transparent provider handoff.

This decision intentionally does **not** promise a distributed database, shared provider accounts,
HA control plane, replicated schedules/workspaces, universal package management, or transparent
provider/session continuation across machines. Those require separate decisions rather than an
expanded meaning of “linked” or “cluster.”
