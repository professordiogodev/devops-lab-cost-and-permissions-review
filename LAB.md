# LAB | AWS: The Surprise Bill and the Permission Audit

## The situation

Thursday. The CFO forwards you an email from AWS: *"Your free tier usage is at 85%."* Underneath she wrote: *"I thought AWS was free for a year. Why am I getting emails about money? And while I have you: the investors asked whether the intern can do more than we think. Can you prove he cannot?"*

Two jobs, both console work, both about **trust**: trust in the bill and trust in the permissions. You will set the guardrails up before anyone is surprised, and then check, with AWS's own tools, that the access you handed out is the access that is needed.

> **Your mission**
> - Create a **budget** that warns you before the money is spent
> - Explore where money goes in **Cost Explorer**, and make costs attributable with **tags**
> - Test permissions with the **Policy Simulator** instead of guessing
> - Find unused permissions with **Access Advisor**
> - Understand how an organisation limits every account with **SCPs**

**Time:** 60-75 min | **Cost:** free | **You need:** an AWS account with an admin user, and the `sam-intern` user and `Clippy-Developers` group from the IAM lab (create them again quickly if you cleaned up; the policy is in that lab)

> Billing pages are available to the account root user and to IAM users **only if** "IAM user and role access to Billing information" is activated. If you cannot see **Billing**, open **Account** (top right) and enable it for IAM users.

---

## Part 1 - Recon: where do I look?

1. Open **Billing and Cost Management**. Find the **Free Tier** page. Which services are closest to their limit? What is the unit of each (hours, GB, requests)?
2. Open **Cost Explorer**. If it has never been used, enable it (data takes up to 24 hours to appear). What does the default chart show?
3. Find the **Bills** page. What does the current month's total say?
4. In **Budgets**, are there any budgets already? Many accounts have none until the first surprise.

> **Predict** What do you think is the most expensive thing a beginner can leave running on AWS by accident? Write a guess of three items, you will compare with the Cost Explorer ranking in a real account later.

---

## Part 2 - Build the guardrail: a budget

Create a **Cost budget** (Budgets -> Create budget -> Customize -> Cost budget):

- Name: `clippy-monthly`
- Period: monthly, recurring, **fixed** amount of **$10**
- Alert 1: **Actual** cost at **50%** of the budget
- Alert 2: **Forecasted** cost at **100%** of the budget
- Email recipients: your own address

Open the budget after creating it and read the **budget details** page.

> **Think** Alert 2 is a *forecast*. Why is that more useful than an alert on the actual spend? What does AWS need before it can forecast (look at the notes in the console)? What does a budget **not** do: does it stop spending?

> **Break it** A forgotten NAT Gateway costs about $32 a month plus data. If you created one on the 1st of the month and forgot it, on which day would each of your two alerts fire? Do the arithmetic.

---

## Part 3 - See and attribute the spend

1. In **Cost Explorer**, group by **Service**, choose a daily granularity and the last 30 days. Then group by **Region** and by **Usage type**. Take note of the three ways to slice the same bill.
2. Tags decide whether you can answer *"Which project spent this?"*. In Billing -> **Cost allocation tags**, find the tag `Project` (it only appears if some resource has it, tag any S3 bucket or IAM user with `Project=clippy` first if needed). Activate it.
3. Tags take up to 24 hours to appear in reports. For now, open **Resource Groups & Tag Editor -> Tag Editor**, search for resources across Regions, and list the ones **without** a `Project` tag.

> **Think** Which part of your account would be impossible to attribute if you had not tagged anything? What is a simple team rule you would put in a pull-request template to keep tags consistent?

---

## Part 4 - Prove the permissions: Policy Simulator

Open the **IAM Policy Simulator** (search for "Policy Simulator" or `policysim.aws.amazon.com`).

1. Choose **Users, groups and roles -> sam-intern**.
2. Select the services EC2 and S3, then run `ec2:RunInstances`, `s3:GetObject` and `s3:PutObject` **with a resource**: type the ARN of the uploads bucket objects (`arn:aws:s3:::clippy-uploads-XXXX/*`). Repeat with the finance bucket.
3. For each row, read the result and click **Show statement** or the matched policy. Which statement allowed it, or what denied it?

> **Predict** Before you run each row, write allowed or denied. How many did you get right?

Save the results for your submission. Then use the simulator on a policy you write: switch to **New Simulation**, paste a small policy that allows `s3:*` on `*`, and run `s3:DeleteBucket`. What does the simulator say?

> **Break it** Create a policy named `clippy-deny-delete` with a `Deny` on `s3:DeleteObject` for the uploads bucket and attach it to Sam's group. Run the simulator again for `s3:DeleteObject`. Allow and Deny are both present; which wins, and what is this rule called?

---

## Part 5 - Find unused permissions: Access Advisor

1. Open **IAM -> Users -> your admin user -> Access Advisor** (also called *Last accessed*). Which services show a recent access time? Which have never been used?
2. Do the same for `sam-intern`. If he has barely used anything, what would you do with a permission that nobody has used for 90 days?
3. Open **IAM -> Credential report** (Account settings or Access reports). Download it. Find: users with a password but no MFA, access keys never used, keys older than 90 days.

> **Think** Access Advisor shows *service-level* last access, with a delay. Why is this a good tool for shrinking permissions but not for proving a user *cannot* do something?

---

## Part 6 - The organisation view (read and think)

Read the AWS documentation page *"Service control policies (SCPs)"*, then answer in your submission:

1. Where does an SCP attach, and what does it do to the account root user?
2. What is the difference between an SCP and an IAM policy? Which one grants permissions?
3. Write in plain English three guardrails you would apply to **every** Clippy account (for example: deny leaving the company's approved Regions, deny disabling CloudTrail, deny creating IAM users with access keys).

---

## Level up (optional)

- **Budget action.** Add a budget action that attaches a deny policy to a role when the budget is exceeded. What are the risks of automatic enforcement?
- **Cost anomaly detection.** Create a monitor in Cost Anomaly Detection for AWS services and an alert subscription. What kind of event would it catch that a fixed budget would not?
- **CLI.** `aws ce get-cost-and-usage` returns the same data as Cost Explorer. Write the call that gives you the last 7 days grouped by service (`--granularity DAILY --group-by Type=DIMENSION,Key=SERVICE`).

<!-- teardown:start (generated by labs.py, do not edit) -->
## Tear down (mandatory)

Resources left behind cost money, keep doors open and pollute the next lab. Removing what you created is part of the job, so it is part of the submission: **you must prove it is gone**.

Do this **last**, after you have pasted your `verify` output into `SUBMISSION.md` (once you tear down, the lab can no longer be verified).

1. Delete the budget `clippy-monthly` (Billing -> Budgets) and any cost anomaly monitor you named `clippy-*`.
2. Delete the deny policy you created in Part 4 (name it `clippy-deny-delete`), then `sam-intern`, the group `Clippy-Developers` and `ClippyDeveloperPolicy` if you recreated them for this lab, and any `clippy-*` bucket.
3. Run the check with your **admin** credentials (CLI or CloudShell).

Then run the proof:

```bash
./teardown-check.sh
```

It prints a list of checks and must end with **TEARDOWN VERIFIED**. If it reports leftovers, remove them and run it again. Paste the full output in your submission.
<!-- teardown:end -->

<!-- submission:start (generated by labs.py, do not edit) -->
## Submission

Answer the questions in `SUBMISSION.md` in **your fork** (paste output in code blocks where asked), push, then submit the **URL of your fork**.

**Questions to answer**

1. Describe the budget you created: amount, scope, thresholds, and who gets the alert. Why alert on the *forecast* as well as the actual spend?
2. In Cost Explorer, grouping by Service, what is your most expensive service this month (even if it is $0.00)? What would you expect to be expensive in a real Clippy deployment, and why?
3. What did you have to do before the tag `Project` showed up as a grouping option in Cost Explorer, and why is it not automatic? Why does consistent tagging matter on day one?
4. Paste the Policy Simulator results for `sam-intern` on `ec2:RunInstances`, `s3:GetObject` on the uploads bucket and on the finance bucket. Which result confirms least privilege and which would be a finding?
5. Access Advisor: which services has your admin user used, and which of its permissions has it never used? What would you do with that list?
6. What is a Service Control Policy and how does it differ from an IAM policy? Give an example of a guardrail you would apply to every account in an organisation.
7. **Tear down.** Paste the output of `./teardown-check.sh`. It must end with **TEARDOWN VERIFIED**.
<!-- submission:end -->
