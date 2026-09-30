# FashilHack Multi-Account IAM Governance on AWS

This project is a complete multi-account AWS governance environment, built end-to-end around a real Organization spanning a management account and two workload accounts. It covers the full governance lifecycle in one continuous build: department-scoped identity through IAM Identity Center, cross-account role assumption between environments, permission boundaries preventing self-escalation, organization-wide guardrails that survive even an account's own administrator, centralized logging no member account can silence, and an access-scanning service that catches sharing nobody intended.

It demonstrates the core discipline multi-account governance actually requires: designing access deliberately, layering enforcement so no single account can override it, and proving  not assuming  that every control genuinely works.

- **Region:** `us-east-1`, held constant throughout
- **Organization:** management account `xarbi`, two member accounts, `FashilHack-Dev` and `FashilHack-Prod`
- **Identity source:** AWS IAM Identity Center's built-in directory

Every control described here was tested with a real allow/deny pair, not just configured and screenshotted. Where something didn't work on the first attempt, that's documented too, gathered into its own section at the end rather than scattered through the build  the troubleshooting is as much a part of this project as the parts that worked cleanly.

## Table of Contents

1. Architecture Overview
2. Foundation  the Organization and Identity Design
3. Proving the Model  Access by Department
4. Cross-Account Role Assumption
5. Permission Boundaries
6. Service Control Policies  the Guardrails
7. Centralized Logging and Automated Detection
8. Full Proof Run and Logging Verification
9. Real Issues Hit and Resolved
10. Conclusion and Next Steps

---

## Architecture Overview

The full environment as it exists today: one Organization, one management account holding identity and the guardrails, and two workload accounts inheriting every rule automatically.

```
AWS Organization
├── Management account  xarbi
│     IAM Identity Center · 3 Service Control Policies
│     IAM Access Analyzer · Organization CloudTrail
└── OU: Workloads  (the guardrails apply to everything inside)
      ├── FashilHack-Dev    developer builds freely
      └── FashilHack-Prod   developer: read-only, admin: full
```

The build happened in the order below  each stage assumes everything before it already exists.

### What's demonstrated where

| Area | Where it's demonstrated |
|---|---|
| Organizations, OUs, one rule applied to many accounts | Foundation |
| Group-based identity, one login, different role per account | Foundation |
| Implicit-deny access proof | Proving the Model |
| Cross-account trust, two independent documents | Cross-Account Role Assumption |
| Privilege-escalation prevention on self-created roles | Permission Boundaries |
| Guardrails that survive an account's own admin | Service Control Policies |
| Tamper-resistant central logging, automated sharing detection | Centralized Logging |
| Every control matched to an actual record | Full Proof Run |

---

## Foundation  the Organization and Identity Design

**Before any guardrail or automation could be built, the Organization itself had to exist, along with an identity model that made "who can do what, in which account" a deliberate design rather than an accident.**

Two accounts were created underneath one management account, and one login-per-person model was built to reach both of them differently depending on where someone actually stood.

| Account | Role |
|---|---|
| `xarbi` (management) | Runs nothing itself. Holds Identity Center, the guardrails, logging, and the scanner. SCPs never apply here  the deliberate exemption everything else is built around. |
| `FashilHack-Dev` | Developers build freely here |
| `FashilHack-Prod` | The real thing runs here  developers get read-only |

Identity lives in exactly one place: IAM Identity Center, inside the management account. Three users, three matching groups, three permission-set templates, and one assignment table connecting them to real accounts:

| Group | In FashilHack-Dev | In FashilHack-Prod |
|---|---|---|
| developers | DeveloperAccess | ReadOnlyAccess |
| admins | AdministratorAccess | AdministratorAccess |
| auditors | ReadOnlyAccess | ReadOnlyAccess |

That table is the actual point of the whole Foundation: the same person, `developer`, ends up with full build access in one account and almost nothing in the other  one login, two outcomes, decided entirely by which account they choose. The moment that table is written, AWS reaches into each target account on its own and creates a real IAM role there, something like `AWSReservedSSO_DeveloperAccess_e3c221776121d854`. Everything above that point is instructions; that generated role is the object actually checked on every request from then on.

### Key decisions and why

- **One Organization instead of separate, unrelated AWS accounts.** Only an Organization lets one place set rules that automatically apply to every account underneath it  the entire premise the rest of this project depends on.
- **Groups, never individual users, in every assignment.** Access changes become membership changes, not policy rewrites  the same reasoning behind every well-designed identity system, cloud or on-prem.
- **The management account deliberately left empty of workloads.** It's the one account guardrails can't reach, so nothing real is allowed to run there  see Service Control Policies below for why that exemption exists at all.

### Evidence

Foundation has no screenshots of its own  its proof is what the next section demonstrates directly: the assignment table actually producing different, correct outcomes per account.

---

## Proving the Model  Access by Department

**With the identity design in place, the next step was proving it actually behaves the way the assignment table says it should  not trusting the console's confirmation screen.**

Logged in as each of the three users and attempted the same action, creating an S3 bucket, in both accounts.

**developer**  full access in Dev, denied in Prod:

![Bucket created in Dev as developer](<screenshots/developer access/created bucket in dev account using developer user.png>)
*Write access succeeding exactly where the assignment table says it should.*

![Denied creating a bucket in Prod as developer](<screenshots/developer access/denied create s3 bucket in fashilhack-prod account since developer has only read access there.png>)
*The same user, same action, a different account  denied, because Prod's developer group was only ever assigned ReadOnlyAccess. Nobody wrote a rule blocking this specifically; write access was simply never granted, which is implicit deny doing its job silently and correctly.*

**admin**  full control in both accounts:

![Bucket created in Dev as admin](<screenshots/admin access/created bucket in dev account using admin.png>)
![Bucket created in Prod as admin](<screenshots/admin access/created bucket in prod account using admin user.png>)
*Both succeeding, matching the assignment table's AdministratorAccess row in both accounts.*

**auditor**  can see, cannot change, in either account:

![Auditor can view existing buckets](<screenshots/auditor access/you can view bucket that exist.png>)
![Auditor denied creating a bucket](<screenshots/auditor access/as auditor you can't create bucket.png>)
*Read access working, write access refused  ReadOnlyAccess doing exactly what its name says, in both accounts identically.*
### Key decisions and why
- **Testing every user in both accounts, not just their "expected" one.** developer's success in Dev proves nothing about Prod on its own  the denial in Prod is the half of the proof that actually matters.
- **Cleaning up test buckets as admin immediately after.** Test resources left behind are how real environments quietly accumulate clutter nobody remembers creating.

---

## Cross-Account Role Assumption

**Developers in Dev normally have almost nothing in Prod. But a developer sometimes legitimately needs to look at Prod's logs  and permanent Prod access is the wrong way to grant that.**

The correct mechanism is a role, in Prod, that can be temporarily borrowed by one specific role in Dev, for one narrow purpose. Two separate documents govern this, and both have to agree:

- **The trust policy**  who is allowed to borrow this role: only Dev's `developer` role, nobody else.
- **The permission policy**  what you can do once borrowed: `CloudWatchLogsReadOnlyAccess`, nothing else.

![Creating the custom role with its trust and permission policy](<screenshots/Build ProdLogReader/we created a role give it custom trust policy and permisson policy created in fashilhack-prod.png>)
*`ProdLogReader`, built inside Prod, with both documents visible together  the actual proof this was a deliberate two-part design, not a single broad grant.*

**Proof it works, from the correct source:**

![Successfully assumed the role](<screenshots/Build ProdLogReader/and here we are login as that role.png>)
![Reading CloudWatch log groups successfully](<screenshots/Build ProdLogReader/we can access cloudwatch log group.png>)
*developer, assuming ProdLogReader, reading Prod's logs  access that Foundation's assignment table alone would never have granted.*

![Denied creating an S3 bucket, correctly outside the granted permission](<screenshots/Build ProdLogReader/and you can't create a bucket denied.png>)
*The same borrowed identity, denied on S3  proving the permission policy is scoped exactly as narrow as intended, not a backdoor to broader Prod access.*

**Proof it refuses the wrong source:**

![Admin's Dev role refused when trying to assume ProdLogReader](<screenshots/Build ProdLogReader/admin in fashilhack-dev account as admin can switch to role ProdLogReader to access production access denied.png>)
*admin's Dev role, denied outright when attempting the same assumption. admin was never named in the trust policy  only developer's role was. This is the proof that "trusted" here means one exact role, not "anyone from Dev."*

### Key decisions and why

- **A purpose-built role instead of widening developer's existing access.** A narrow, revocable, auditable borrowing mechanism is a fundamentally different  and safer  design than simply granting more standing permission.
- **Testing the refusal deliberately, not just the success.** A trust policy that's never been shown to refuse anyone hasn't actually been proven to work; it's only been shown to not-yet-fail.

---

## Permission Boundaries

**Developers need to create their own IAM roles, for the apps they build. But a developer who can create a role with any power at all can create one stronger than themselves and use it to escalate  which defeats every access decision made in Foundation.**

The fix is a second, independent ceiling on any role a developer creates, one that can't be removed by the developer themselves:

- **`DeveloperRoleBoundary`**  caps any role wearing it at S3 read-only, permanently, no matter what else gets attached later.
- **`AllowRoleCreationWithBoundary`**  lets developer create roles at all, but only if this exact boundary is attached to the role being created.

**Proof 1  create with the boundary attached: works.**

![Role created successfully with the boundary attached](<screenshots/boundary/role created with the boundary attached, succeeds.png>)

**Proof 2  create without any boundary: denied outright.**

![Denied, no boundary means the condition is never met, falls back to implicit deny](<screenshots/boundary/no boundary specified at all, the condition isn't met, so the allow never applies, and it falls back to implicit deny.png>)
*The permission to create roles at all was written with a condition requiring this exact boundary  omit it, and the condition simply never matches, and the default answer returns to no.*

**Proof 3  attach something outside the boundary anyway, and it still doesn't work.**

![AmazonEC2FullAccess attached directly to the boundary-capped role](<screenshots/boundary/attached policy to role boundary we created TestRoleWithBoundary.png>)
*AWS allows this attachment without complaint  attaching a policy is never what the boundary blocks.*

![Policy Simulator confirms EC2 stays denied, the boundary never moved](<screenshots/boundary/simulated and denied even though AmazonEC2FullAccess is checked under Identity-based policies, the DeveloperRoleBoundary permissions boundary caps it out, since that boundary only allows S3 read..png>)
*The actual proof: EC2FullAccess is fully attached, and EC2 is still denied. Effective access is always whatever the identity policy and the boundary agree on together  never more, no matter what gets bolted on afterward.*

### Key decisions and why

- **Testing the attach-anyway scenario specifically**, rather than stopping at Proof 2. Proving a boundary blocks role *creation* without a boundary is only half the claim  proving it still holds after something gets attached anyway is the half that actually demonstrates the ceiling can't be worked around.

---

## Service Control Policies  the Guardrails

**Permission Boundaries cap one role. Service Control Policies do the identical thing over an entire account  and, critically, apply even to that account's own administrator and root.**

Three guardrails were written, each aimed at one specific risk:

1. `DenyOutsideUsEast1`  deny nearly everything outside `us-east-1`, with a short exemption list for genuinely global services
2. `DenyStopCloudTrail`  deny stopping, deleting, or editing any CloudTrail trail
3. `DenyLeaveOrg`  deny `organizations:LeaveOrganization`

Rather than attaching untested guardrails straight onto the real `Workloads` OU, a temporary **Test** OU was created first, and Dev was moved into it alone  a mistake there only costs a throwaway test account, never Prod.

**Proving the region lock, as admin, inside Dev:**

![S3 creation denied by the SCP in the wrong region](<screenshots/guardrail SCP/region/s3 creation denied by scp.png>)
*A clean, explicit denial, the SCP named as the reason  even though this is admin, holding full AdministratorAccess.*

**Proving the CloudTrail guardrail, on a throwaway trail:**

![Stopping it denied, SCP named in the error](<screenshots/guardrail SCP/cloudtrail audit/try to stop the trail logging error scp shows up.png>)
*admin, again  unable to stop logging of their own actions, which is exactly the point: an admin who could silence their own audit trail would make every other guardrail unverifiable.*

**Proving the management account is exempt  the entire reason nothing else runs there:**

![Root, in the management account, creating a bucket in a different region with no SCP interference](<screenshots/guardrail SCP/created a s3 bucket in management account choosing diff region showing scp does not apply to root evidence.png>)
*The exact same action that was just denied inside Dev succeeds with zero friction here, because SCPs never reach the management account at all. This single screenshot is the reason Foundation kept that account empty  it's the one place with no ceiling above it, so nothing sensitive belongs there.*

Once proven on the Test OU, all three SCPs were attached to `Workloads` directly, and Dev was moved back  both Dev and Prod now inherit every guardrail automatically, just by living inside that OU.

### Key decisions and why

- **A Test OU before Workloads, every time.** A region-lock guardrail written carelessly can lock an admin out of their own account. Proving it on an account nobody depends on yet is strictly cheaper than proving it on Prod.
- **Testing as admin specifically, not developer.** A guardrail that only stops developer proves nothing about whether it actually reaches the account's most privileged identity  which is the entire point of an SCP existing at all.

---

## Centralized Logging and Automated Detection

**A guardrail nobody can prove was enforced isn't provably a guardrail. This section builds the two systems that make every claim above independently checkable.**

**Organization CloudTrail**  one trail, created from the management account, logging every account in the Organization into a single S3 bucket also in the management account.

![The S3 bucket, showing log data organized by account ID](<screenshots/cloudtrail/s3 buket showing id of both prod and dev logs.png>)
![An actual raw CloudTrail log file, not just claimed, opened and inspected](<screenshots/cloudtrail/one of json gz  data.png>)
*Not just a claim that logging exists  an actual log file, opened and read, proving events are genuinely being captured rather than assumed to be.*

![Confirming member-account admins in Dev cannot stop, delete, or edit the org trail](<screenshots/cloudtrail/as admin in dev if you go to trail you can stop,delete or edit it evidence.png>)
*Two separate protections stack on this one object: the org trail structurally protects itself from member accounts by design, and the region-wide SCP from the previous section also covers any CloudTrail action taken outside `us-east-1`.*

**IAM Access Analyzer**  an external access analyzer at the Organization level, zone of trust set to the entire Organization: sharing to another in-org account is fine, sharing to anything outside it gets flagged automatically.

![The finding, a role in Prod trusted outside the org](<screenshots/analyzer/finding found created a role in production account seen on analyser evidence.png>)
*A role in Prod was deliberately trusted to an account outside the Organization  and the analyzer raised it as an active finding within minutes, with no manual review needed to notice it.*

![The finding resolves once the trust is back inside the organization](<screenshots/analyzer/now externalRole creation the scanner is saying resolved indicating hey this is inside your own organazation.png>)
*Fixing the trust back to an in-org account, and the finding resolving on its own  proof this is a live scanner, not a one-time snapshot.*

![The validation warning at creation time](<screenshots/analyzer/impassrole policy at creation gives this warning.png>)
*A separate, free companion feature  policy validation  catching an `iam:PassRole` grant without a scoped resource at write-time. `PassRole` without a tight resource constraint is a well-known privilege-escalation pattern; catching it before it becomes a live permission is strictly better than catching it in an audit afterward.*

### Key decisions and why

- **Manufacturing a real finding instead of trusting the analyzer's description.** A scanner nobody has ever watched actually catch something is a claim, not a proof.
- **Organization-level logging instead of per-account trails.** A per-account trail that account's own admin controls can be edited or stopped by that same admin  the entire threat model Service Control Policies exist to close.

---

## Full Proof Run and Logging Verification

Every result from every section above was re-run as one continuous pass, then matched to an actual logged record rather than only the console's on-screen response:

![The SwitchRole event, both the mechanism and the record of it happening](<screenshots/logging proof/switchrole event log.png>)
![An SCP denial, logged with the explicit-deny reason](<screenshots/logging proof/SCP denial.png>)
![The CreateRole event for TestRoleWithBoundary](<screenshots/logging proof/TestRoleWithBoundary event log.png>)
*Three separate mechanisms  cross-account assumption, a guardrail denial, and a boundary-constrained creation  each confirmed not just as an on-screen outcome, but as a permanent record in CloudTrail.*

A control that isn't logged can't be proven to have worked after the fact. This is the difference between "it worked" and "here is the record that it worked"  and it's the reason logging was built before the guardrails and access model were even fully proven, not added afterward as an afterthought.

---

## Real Issues Hit and Resolved

Two genuine failures came up during this build, both documented in full because they taught something real about how AWS actually behaves  not just how it's described in documentation.

**1. Root cannot use Switch Role in the console at all.**

Attempting to Switch Role directly as the root user, from the management account, failed outright with "Invalid information in one or more fields"  not a typo, a hard AWS platform limitation. Root has no identity-based policy for STS to evaluate against, so the request is refused before it ever reaches the target role's trust policy.

![The failure, as root](<screenshots/switch-role/switch does not work in root user in management account we created iam role.png>)

**The fix:** a real IAM user, `mgmt-admin`, created in the management account specifically so Switch Role would even appear as a menu option.

![Confirming AdministratorAccess is attached to the new IAM user](<screenshots/switch-role/permession admin access given to iam role.png>)
![Successfully landed inside Dev, wearing OrganizationAccountAccessRole](<screenshots/switch-role/role switched to fashilhack-dev account.png>)

**The actual lesson:** this is the concrete, practical reason real teams create a named admin user immediately and stop using root for anything routine  not as an abstract best practice, but because root is structurally incapable of doing this one specific, common thing.

**2. EC2 showed no visible error outside the region lock, when S3 clearly did.**

Same SCP, same wrong region, same account, same user  S3 was denied cleanly, but attempting to view or launch EC2 outside `us-east-1` produced no visible error.

![EC2 create attempt in the wrong region, no error shown](<screenshots/guardrail SCP/region/if  i chnage the region as you can see no error for ec2 creation.png>)

**Why, most likely:** the EC2 console pages are backed by a large number of read-only Describe calls just to render the screen  listing regions, AMIs, instance types, key pairs  before a single create action is ever attempted. Many of those describe-style calls are broad, low-risk reads that can still succeed under a deny written around create and modify verbs, so a page loading without error is not the same claim as EC2 resources actually being creatable there. The SCP's exemption list names only a short set of genuinely global services, and EC2 isn't one of them  so the actual launch action at the final step should still be denied. What this screenshot captures is the console browsing experience, not a completed launch.

**Why this stays in the write-up instead of being quietly fixed:** it's an honest reminder that "no error on screen" and "the action is actually allowed" are not the same claim, especially on console pages built from many small API calls rather than one. Closing this out fully would mean attempting the actual launch action to completion, or checking CloudTrail for a denied `RunInstances` event specifically  noted here as an open item rather than glossed over.

---

## Conclusion

This build is one continuous governance story, not a series of disconnected labs. Foundation established who should exist and what they should be able to do, per account, using groups and a single assignment table as the source of truth. Proving the Model confirmed that table actually produced the right outcomes, including the implicit-deny case nobody had to write a rule for. Cross-Account Role Assumption showed the correct, narrow way to grant a deliberate exception without widening standing access. Permission Boundaries proved a ceiling can be placed on one role that survives even an attempt to override it. Service Control Policies proved the same idea at the scale of a whole account  reaching even that account's own admin  and proved, just as importantly, exactly where that ceiling stops: the management account, on purpose. Centralized Logging and Detection made every one of those claims independently checkable instead of merely asserted. And the Full Proof Run tied every result to an actual record, closing the loop between "it worked" and "here's the evidence it worked."

Two genuine platform surprises came up along the way  root's inability to use Switch Role, and the distinction between a console page loading without error and an actual denied API call  and both are documented here rather than omitted, because that's the same standard the rest of this build holds itself to: prove it, don't assume it.

### Suggested next steps

- Confirm the EC2 region-lock nuance fully by checking CloudTrail for a denied `RunInstances` event specifically.
- Extend this pattern to more accounts by attaching the same three SCPs to additional OUs  the design scales without new guardrail logic.
- Swap Identity Center's built-in directory for an external identity provider, mirroring how a real organization typically federates identity rather than managing it natively in one cloud.
- Automate the SCP JSON and Identity Center permission sets in Terraform or CloudFormation, now that the console-built version has proven the mechanics work.
