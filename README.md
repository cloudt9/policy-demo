AMI Cleanup Logic — Overview & Decision Workflow
1. Introduction

Brief outline of the purpose of this document:

Describe the existing AMI cleanup capabilities.

Show the considerations informing cleanup behaviour.

Document supported use cases and scenarios.

Present the final end-to-end decision workflow for AMI deletion.

2. Existing Features

Summarise what the AMI cleanup process already supports:

2.1 Branch-Aware Cleanup Modes

Dev Mode: Evaluates AMIs built from feature branches.

Gold Mode: Evaluates AMIs built from main branch builds.

2.2 Launch-Permission Approval Detection

Determines whether an AMI is:

Approved (both approver accounts added)

Unapproved

Partially approved (misconfiguration → skipped)

2.3 Cross-Account Usage Detection

Identifies whether an AMI is used by:

EC2 instances

Launch templates

Auto Scaling groups

Other accounts

2.4 Tag-Based Behaviour Controls

Branch tags

image_family tags

deletion_protection flag (absolute skip)

Age metadata

Optional retention overrides

2.5 Retention Policies

Dev mode: short age-based retention

Gold mode:

unapproved: age-based cleanup

approved: keep N most recent + optional age constraints

3. Key Considerations

Document the logic drivers:

3.1 Safety & Guard Rails

Never delete AMIs in use

Always skip AMIs with deletion_protection=enabled

Partial approvals treated as misconfigurations, never deleted

3.2 Build Cadence & Capacity

Feature branches generate many temporary builds → require aggressive cleanup

Main branch builds are slower, more stable → require conservative retention

3.3 Disaster Build Leak Handling

Scenario where feature branch pipelines fail to clean up after themselves:

Dev mode is designed to automatically clean leaked builds based on age

Ensures runaway storage growth is controlled without manual intervention

3.4 Auditability

Every decision logged to DynamoDB

Includes: branch, mode, approval state, age, usage status, skip reason or delete decision

4. Supported Use Cases

Outline what situations the system is designed to handle:

4.1 Routine Feature Branch Builds

Many short-lived builds

Auto-cleaned after threshold (e.g., 3 days)

4.2 Main Branch Promotion Workflow

Builds sit awaiting approval

Approved images retained for deployments

Unapproved images cleaned up if not validated within timeframe

4.3 Disaster / Leak Cleanup

Builds left behind due to pipeline failure

Dev mode cleans them automatically after max age

4.4 Manual Protection of Specific AMIs

By applying deletion_protection=enabled

Ensures the system will not remove them, regardless of other conditions

5. Scenarios Covered

Summarise examples:

Scenario A — Old Feature Build Not in Use

Mode: Dev

Branch=feature, age > dev_threshold

Decision: Delete

Scenario B — Approved Main Build, Outside N Retained

Mode: Gold

Branch=main, approval=approved, older than top N

Decision: Delete (subject to min-age)

Scenario C — Unapproved Main Build Stale

Approval=unapproved, age > gold_threshold

Decision: Delete

Scenario D — Protected Image

deletion_protection=enabled

Decision: Always Skip

Scenario E — Image in Use Anywhere

Any branch, any mode, any age

Decision: Skip

6. Final Decision Workflow

A concise flow that incorporates all rules:

Step 1 — Mode / Branch Filter

Dev mode → consider only Branch=feature

Gold mode → consider only Branch=main

Else → skip

Step 2 — Deletion Protection

If deletion_protection=enabled → Skip

Step 3 — Usage Check

If AMI used by any instance / LT / ASG / external account → Skip

Step 4 — Approval State

Approved (two approvers)

Unapproved

Partial approval → Skip + flag

Step 5 — Apply Mode-Specific Retention
Dev Mode

If age ≥ DEV_AGE_THRESHOLD → Delete

Else → Skip

Gold Mode — Unapproved

If age ≥ GOLD_UNAPPROVED_AGE_THRESHOLD → Delete

Else → Skip

Gold Mode — Approved

Identify N most recent approved for the image family

If outside N and older than GOLD_APPROVED_MIN_AGE → Delete

Else → Skip

Step 6 — Final Verification (Optional but Recommended)

Re-check usage

Ensure deletion_protection still off

Execute delete / deregister

Step 7 — Logging

Record full decision into DynamoDB.

5. BEST PRACTICE: AGE RETENTION THRESHOLDS (updated)

Absolute skip: If tag deletion_protection=enabled is present, the AMI is always skipped from deletion (regardless of mode, approval state, in-use status, age, or retention bucket). Log reason skip: deletion_protection.

Recommended age thresholds (apply only when candidate is not protected and not in use and mode-filter matches):

Dev mode (feature branch)

Keep if in use OR younger than 3 days.

Delete if not in use AND age ≥ 3 days.

Rationale: aggressive cleanup to avoid build flood, short grace window for quick test iterations.

Gold mode (main branch) — unapproved main images

Keep if in use OR younger than 7–14 days (configurable; start at 7).

Delete if not in use AND age ≥ 7 (or 14) days.

Rationale: main branch builds may take longer to review; give teams breathing room.

Gold mode (main branch) — approved images

Primary retention is N (e.g., 3) most recent approved images.

For approved images outside the N window, apply a min-age safeguard (e.g., do not delete approved images younger than 3 days).

Optionally also delete approved images older than a maximum age if they exceed regulatory/company limits — configurable.

Note: age = Now - Image.CreationDate (UTC). Always calculate in a deterministic timezone and log the computed age in DynamoDB record for auditing.

DECISION FLOW (concise, updated — deletion_protection explicit)

Branch filter (mode):

If runner is dev → evaluate only AMIs with tag Branch=feature.

If runner is gold → evaluate only AMIs with tag Branch=main.

Else → skip.

Deletion protection check (absolute early exit):

If tag deletion_protection=enabled → SKIP (log skip_reason=deletion_protection) — stop processing this AMI.

Cross-account usage check:

Query AMI Usage API (or estate scan fallback).

If in use anywhere (instance / LT / ASG) → SKIP (log skip_reason=in_use).

Approval detection (launch permissions):

If launch permissions include both approver accounts → approval=approved.

If none → approval=unapproved.

If exactly one → approval=partial (treat as misconfigured → SKIP + flag for ops).

Apply mode-specific retention logic:

If mode = dev (Branch=feature)

If in use → SKIP.

Else if age >= DEV_AGE_THRESHOLD (default 3 days) → DELETE (unless deletion_protection, but that was already checked).

Else → SKIP (still within grace).

If mode = gold (Branch=main)

If approval = approved:

Retain the N most recent approved images for the family (N configurable, default 3).

If this AMI is outside the N set AND age >= GOLD_APPROVED_MIN_AGE (e.g., 3 days) → DELETE.

Otherwise → SKIP.

If approval = unapproved:

If age >= GOLD_UNAPPROVED_AGE (e.g., 7 days) → DELETE.

Else → SKIP.

If approval = partial: SKIP + flag.

Final pre-delete verification (just before performing deregister):

Re-run AMI Usage check to avoid races (if your workflow cares) — if now in-use → abort and log. (You said ignore race conditions generally, but this check is cheap and useful).

Confirm deletion_protection still not set.

Proceed to delete (or log dry-run).

Logging / auditing: always write final decision to DynamoDB: {ami, branch, mode, approval_state, in_use, deletion_protection, age_days, decision, timestamp, run_id}.

Short examples (for clarity)

AMI: Branch=feature, age = 2 days, not in use → dev mode → SKIP (grace).

AMI: Branch=feature, age = 5 days, not in use → dev mode → DELETE.

AMI: Branch=main, approved, rank = 5th newest (N=3), age = 4 days → gold mode → DELETE (if > min-age).

AMI: Branch=main, unapproved, age = 8 days → gold mode → DELETE (age threshold passed).

AMI: any branch, deletion_protection=enabled → SKIP always.
