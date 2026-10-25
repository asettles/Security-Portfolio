# Investigating an Azure Deployment That Bypassed Naming Controls

## Scenario

In a live multi-user Azure training tenant, an intern received temporary Contributor access to deploy a test environment, failed to follow naming standards, and left for the weekend. Taking the role of the on-call engineer on Monday, I used Reader access to identify the affected resources, examine their tags, and trace the deployment history. My objective was to reconstruct what happened and determine why the existing naming policy had not blocked the noncompliant deployment.

## Environment

- **Platform:** Live multi-user Azure training tenant.
- **Services:** Azure Resource Manager and Azure Policy.
- **Tools:** Azure portal and Obsidian for investigation notes.
- **Access:** Reader; investigation only, with no configuration changes or remediation performed.

## Investigation

1. **Find the naming outlier.** I reviewed the resource groups in the subscription and looked for a name that did not follow the expected pattern. One group stood out because its name lacked the usual resource-group prefix and environment and region information. This gave me a starting point for further investigation; a naming violation alone would not prove malicious activity.
  <img width="583" height="687" alt="01-resource-groups" src="https://github.com/user-attachments/assets/1a9614e2-5fce-4f6d-93c1-30cdf093fc37" />


2. **Inspect the payload.** I opened the resource group and found one storage account. I examined its tags and found that the cost center was listed as “unspecified” and the environment as “unknown.” Although the tags existed, those values provided little useful information for cost allocation or identifying the resource’s intended environment.
<img width="1904" height="710" alt="02-resource-tags" src="https://github.com/user-attachments/assets/b97ca6d2-97fd-4c23-9641-189abc26de1b" />

3. **Trace the deployment.** I returned to the resource group’s Deployments view and located its deployment record. I used the record to connect the resource to the deployment history. The deployment name supplied a clue about its purpose and timing, but a name alone does not establish the authenticated identity that performed an action.
<img width="1319" height="855" alt="03-deployments-intern" src="https://github.com/user-attachments/assets/8142e3ba-bf6b-4e7e-8b77-b0c6b4de4b8d" />

4. **Identify the governance gap.** I opened the resource group’s Policies view and found the naming policy reporting noncompliance. I then opened **Assignments**, selected the naming policy assignment, and checked its **Parameters** tab. Its Effect was set to **Audit**, which reports violations without blocking the request. The policy detected the naming violation but allowed the deployment to proceed.
<img width="1888" height="648" alt="04-policy-audit" src="https://github.com/user-attachments/assets/608fb7bd-9cc9-42e2-b3e7-2dc492086e18" />



## What broke / what surprised me

I initially followed a deployment from an earlier lesson instead of the one associated with the resource I was investigating. That sent me down the wrong path until I returned to the correct resource group. I learned to verify the scope before drawing conclusions from a deployment record.

What surprised me most was that the naming policy was active and reporting noncompliance, yet the deployment still succeeded. Its effect was set to Audit, so it detected the violation without blocking it. I needed to check how the policy was configured to understand why it allowed the request.

## Findings and recommendations

The naming policy used **Audit**, which reported the violation but allowed the deployment. The resource’s cost-center and environment tags also contained unhelpful values, making cost allocation and resource classification harder.

Recommendations:
- If naming compliance must be enforced, validate the policy’s scope and conditions, test its impact, and change the effect to **Deny**. Document the reason if Audit is intentional.
- Limit temporary Contributor access to the required scope and duration.
- Require meaningful ownership, environment, and cost-center tags at deployment time.

This investigation used Reader access. I did not change the configuration or test remediation.

## What I learned

1. Resource hierarchy helps establish where a change occurred and which controls apply.
2. Tags support investigations only when their values provide useful information.
3. Deployment history helps reconstruct provisioning, but a deployment name alone does not verify who performed it.
4. An active policy can report violations without preventing them; its effect matters.
5. Next time, I will confirm the resource group first and capture redacted screenshots throughout the investigation.
