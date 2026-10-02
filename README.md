# Azure Governance Incident investigation

## Scenario

An intern given temporary contributor access deployed a resource in azure that failed to meet compliance with the companies governance policy. As a engineer on-call, I was asked to reconstruct what happened and why the deployment was allowed amidst violating policy. 

## Environment

- platform: Microsoft Azure
- subcription: MHL
- Access-Level: reader (for investigation)
- Services/Tools: Azure Portal, Azure Policy, Resource Groups, Activity Log

## Investigation

### 1. Identifying the offender
While inside the subscription that holds the resource group responsible for the deployment, I discovered a resource group that did not comply with the proper naming convention according to the governance policy.

The resource group identified was labeled testdeploy123.
Conclusion: This identified the non-compliant resource group that I needed to investigate further.

<img width="393" height="235" alt="image" src="https://github.com/user-attachments/assets/6197454b-5b14-4f7e-bcc9-58b77f51f575" />


---

### 2. Inspecting the payload
The resource group contained a resource labeled stinternazwdmranjdpy4, which is a storage account. I reviewed the resource's tags and found that it contained the string that was instructed to be used when documenting the resource, with the key labeled intern-flag.

Conclusion: The storage account contained the expected intern-flag tag, so I continued investigating the deployment that created the resource.

<img width="541" height="386" alt="image" src="https://github.com/user-attachments/assets/56ee13e1-de5a-4085-bf47-ecf5223f0b10" />

---

### 3. Tracing the deployment
I then checked the resource group's deployment settings and retrieved the deployment metadata. The deployment was named interntestdeploy. I reviewed the deployment date and status and confirmed that the deployment successfully created the storage resource.

Conclusion: The deployment associated with the non-compliant resource was successfully completed, so I next investigated how it was able to bypass the naming requirement.

<img width="695" height="467" alt="image" src="https://github.com/user-attachments/assets/fcc08bc0-f783-4d3f-84ad-ed809aaeec9c" />

---

### 4. Checking the policy effect
A policy assignment requiring standard naming conventions for resource groups is supposed to enforce that rule. The policy was applied to the subscription where the resource group resides, and the deployment successfully created the misnamed resource group.
I checked the policy effect to determine whether the policy was malfunctioning or whether its configuration allowed non-compliant resources to exist while still flagging them as violations.
The policy effect was set to Audit. This allows non-compliant resources to exist while recording the violation rather than preventing deployment.

Conclusion: The policy was not configured to block the deployment. Its Audit effect allowed the non-compliant resource group to be created while still identifying it as a policy violation.

<img width="490" height="327" alt="image" src="https://github.com/user-attachments/assets/6f441687-3a9d-4ddf-9715-747f8383bde9" />

<img width="917" height="198" alt="image" src="https://github.com/user-attachments/assets/e31bc61d-6339-40c9-862e-ea8361e10268" />

---

## What broke / what surprised me

The first assumption I had was that the governance policy was not working because a non-compliant resource group had been successfully deployed.
I initially approached the issue as if I needed to determine why Azure had failed to enforce the naming requirement. Rather than assuming the policy was malfunctioning, I traced the resource back through the deployment and then reviewed the policy assignment and its configured effect.
The important finding was that the policy was set to Audit, not Deny. That changed the interpretation of the incident: Azure had not failed to enforce the policy. The policy was behaving according to its configuration by allowing the deployment and recording the resource as non-compliant.
What surprised me was how easy it would be to look at a policy that says a naming standard is required and assume that the policy automatically prevents violations. In practice, I need to verify scope, effect, and deployment context before concluding that a governance control has failed.

## Findings and recommendations

### Findings
- The resource group testdeploy123 violated the organization's naming convention.
- The associated storage account contained the expected intern-flag tag.
- The interntestdeploy deployment successfully created the resource.
- The naming policy was assigned at the subscription level.
- The policy effect was configured as Audit, allowing the non-compliant resource to exist while recording the violation.

### Recommendations

- Use a Deny effect for naming requirements that must be enforced.
  This would prevent future deployments from creating resources that violate the required naming convention.
- Review existing policy assignments and their effects.
  Governance policies should be checked to ensure their configured effect matches the organization's intended enforcement level.
- Continue using Audit where visibility is the objective.
  Audit can be useful when introducing a new policy because it identifies existing violations before moving to enforcement.

## What I learned

- Azure Policy does not automatically block non-compliant resources. The policy effect determines whether Azure audits, denies, or otherwise handles a violation.
- A successful deployment does not necessarily mean it complied with governance requirements. I need to check the applicable policy and its effect rather than assuming a successful deployment passed compliance.
- Investigation should follow the evidence. I started with the non-compliant resource group, traced the resource and deployment, and then examined the policy configuration to explain why the deployment was allowed.
- What I'd do differently: I would check the policy effect earlier in the investigation. Once I know the requirement is enforced through Azure Policy, verifying whether the effect is Audit or Deny can quickly narrow down why a non-compliant deployment succeeded.
