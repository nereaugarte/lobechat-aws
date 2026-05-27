# Q2 — AI chat dump

## Session 1 — Claude Sonnet 4.6 via Claude Code, 2026-05-27 15:05 UTC

**Prompt:**

```
I'm writing up the 3-environment architecture section. My constraint is Qdrant must stay
on EC2 in all three envs — no managed vector DB. I want to justify the EBS sizing
per environment, not just pick round numbers. What's the right mental model for
sizing Qdrant storage when you're running a legal corpus that today is ~3M vectors
at 1536 dimensions (text-embedding-3-small output)?

Also, for the snapshot policy, I was going to go with daily EBS snapshots but I'm
wondering whether there's a meaningful argument for also triggering Qdrant's own
snapshot API before bulk re-indexing vs just relying on EBS. Are they redundant or
complementary?
```

**Response:**

```
The storage model for Qdrant on EBS:

1536-dim float32 vectors = 6144 bytes/vector. At 3M vectors, raw vector data is ~18 GB.
But Qdrant's HNSW index (m=16, ef_construction=100 default) adds roughly 1.2–1.5× overhead
for the graph structure, so the index itself is ~22–27 GB. Add payload storage (metadata
fields: doc_type, executed_date, jurisdiction, etc.) at maybe 200 bytes/doc → 600 MB.
Plus WAL and segment housekeeping: budget 30–40% headroom. Call it 50–60 GB realistic
minimum for production. 500 GB gp3 gives you room to 10× the corpus and add collections
without resizing — EBS resize requires a reboot (or live resize with file system extension,
but that adds ops risk mid-production). So 500 GB is defensible as a planning decision,
not over-provisioning.

For dev at <100k vectors: even 20 GB is generous. You want a volume that lets you rebuild
from scratch fast (snapshot restore time scales with volume size, not data size), so
keeping dev small is also about recovery speed.

On EBS snapshot vs Qdrant snapshot API: they are complementary, not redundant.

EBS snapshot is a block-level point-in-time of the volume. It captures Qdrant data in
whatever state the OS had on disk — if Qdrant had an in-flight write or an open WAL
segment, the restored snapshot may need recovery. Qdrant is designed for this (it uses
a write-ahead log and can recover from a mid-write snapshot), but the recovered state is
"last flushed checkpoint" not "exact moment of crash."

Qdrant's `/collections/{name}/snapshots` API produces a collection-level export that is
internally consistent: all segments are flushed, WAL is drained, and the snapshot is a
clean copy. It's what you want as a pre-bulk-reindex checkpoint because it gives you an
exact rollback point if the re-index produces bad embeddings or overwrites production
vectors with incorrect content. If you later detect the issue and restore from EBS, you'd
lose the correct vectors too (because the re-index overwrote them before the next EBS
snapshot). The Qdrant snapshot to S3 is your fine-grained undo.

So the right policy is: EBS daily for disaster recovery, Qdrant API snapshot before any
operation that modifies the collection (re-index, schema change, bulk delete).
```

**What I kept / changed:** Used the vector math (6144 bytes/vector, ~50–60 GB minimum) to back the 500 GB EBS choice in the answer. Kept the EBS vs Qdrant snapshot complementarity argument verbatim in structure; tightened the prose. Added the `r7g.xlarge` (32 GB RAM) justification for memory-mapped segments fitting in RAM — this was my own reasoning from Qdrant's memory-mapping model, not in the AI response.

---

## Session 2 — Claude Sonnet 4.6 via Claude Code, 2026-05-27 15:30 UTC

**Prompt:**

```
For the promotion flow: I want to make the GitOps story tight. The project already
uses Commitizen for tagging. What's the cleanest way to handle the case where
stage and prod use different secrets but the same container image tag? I want to
avoid the antipattern where someone promotes a hotfix directly to main without
going through stage, and also avoid the antipattern where the stage environment
drifts because of an ad-hoc docker compose up on the EC2.

Also: the grader will ask about config promotion. The answer "secrets stay in
Secrets Manager and are not in git" is correct but shallow — what's the non-obvious
detail that shows I actually understand the operational edge case here?
```

**Response:**

```
The cleanest separation is: the container image is the immutable artifact that
crosses environments; Secrets Manager is the environment-specific lookup layer.

For the hotfix antipattern: protect main with a required GitHub Actions environment
("prod") that needs a human approver, and make the CI pipeline fail if the commit
is not reachable from the stage branch tip. That is, before deploying to prod, the
workflow checks `git merge-base --is-ancestor $COMMIT $STAGE_BRANCH_SHA`. If the
hotfix bypassed stage, the check fails and the deploy is blocked. This is a two-line
shell check that prevents the entire class of "emergency hotfix went to prod untested"
incidents.

For stage drift: the real fix is that nobody has SSH access to the stage EC2 to do
ad-hoc docker compose up. The EC2 runs a systemd-timer that hourly checks whether
the running container SHA matches the latest image tag in ECR for the stage branch.
If it diverges, it sends a CloudWatch alarm. Enforcement, not just policy.

On config promotion — the non-obvious edge case is secret version skew:

When you rotate a secret in Secrets Manager (e.g. rotate the Postgres password), the
new version is `AWSCURRENT`. But ECS tasks that started before the rotation are still
running with the old secret baked into their environment variables at launch time. They
don't automatically pick up the new value — ECS does not hot-reload secrets. A new task
deploy is required to pull `AWSCURRENT`. If you rotate but don't redeploy, you end up
with a split-brain: new tasks use the new secret, old tasks use the old one. In a
stateful service like Postgres, if the password rotated, old tasks start getting auth
failures at their next reconnect. The fix is to gate secret rotation on a forced task
replacement — either via a CodePipeline stage or a Lambda that triggers `ecs:UpdateService
--force-new-deployment` after the secret rotation completes. This is the operational edge
case that shows you understand ECS secrets lifecycle vs Lambda (where you can reload at
invocation) vs EC2 instance (where you would use SSM agent to refresh).
```

**What I kept / changed:** Used the ECS secret version skew observation directly in the answer — this is a genuine operational gotcha that differentiates a Level 3 answer. Added it to the Secrets Manager entry in the managed services section. Kept the merge-base hotfix prevention logic; simplified to a description rather than shell code. The "nobody SSOs to stage EC2" enforcement point was reshaped into the branching policy section.
