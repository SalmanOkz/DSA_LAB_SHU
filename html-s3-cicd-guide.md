# Deploy index.html to AWS S3 with GitHub Actions

Step-by-step setup and troubleshooting guide for Muhammad Salman.

Repository: https://github.com/SalmanOkz/DSA_LAB_SHU

## 1. Understand the pipeline

1. Edit `index.html` on your computer.
2. Push or commit the change to GitHub's `main` branch.
3. GitHub Actions starts a temporary Ubuntu runner.
4. The runner checks out the repository and checks that `index.html` exists and is not empty.
5. GitHub obtains an OIDC token and exchanges it for temporary AWS role credentials.
6. The runner uploads `index.html` to S3.
7. Visitors open the S3 website endpoint to see the page.

This follows the supplied diagram. Since the application is plain HTML, Node.js, npm installation, and a compilation step are unnecessary. Docker and EC2 are also unnecessary for this pipeline. The Ubuntu runner is provided by GitHub, not an EC2 server you create.

## 2. Values used in this project

| Setting | Value |
| --- | --- |
| GitHub repository | `SalmanOkz/DSA_LAB_SHU` |
| Deployment branch | `main` |
| AWS account ID | `459640517588` |
| IAM role | `GitHubHtmlDeployRole` |
| Role ARN | `arn:aws:iam::459640517588:role/GitHubHtmlDeployRole` |
| Upload permission policy | `GitHubDeployHtmlPolicy` |
| OIDC provider | `token.actions.githubusercontent.com` |
| OIDC audience | `sts.amazonaws.com` |
| S3 bucket | Replace `YOUR_BUCKET_NAME` throughout this guide |
| AWS region | Use the actual region of your bucket |

The bucket name and region were not confirmed when this guide was written. Do not paste bucket placeholders into AWS unchanged. Examples such as `us-east-1` are not confirmation of your actual region.

## 3. Prepare the repository

Place `index.html` in the repository root. The completed repository needs these files:

- `index.html`
- `.github/workflows/deploy.yml`

The workflow below deploys only `index.html`. Inline CSS and JavaScript work normally. Separately stored CSS, JavaScript, images, and other local assets require additional upload commands and corresponding IAM object permissions. Relative assets will otherwise be missing on the website.

## 4. Create the S3 bucket

1. Sign in to AWS and open **S3**.
2. Choose **Create bucket** and select a general purpose bucket.
3. Enter a globally unique name. Record it as `YOUR_BUCKET_NAME`.
4. Select a region and record its code, such as `us-east-1`.
5. Keep Object Ownership set to **Bucket owner enforced**, with ACLs disabled.
6. Use **SSE-S3** default encryption for this direct public website setup.
7. Create the bucket.

AWS hosting and transfer may incur charges depending on your account and usage. This guide does not assume free-tier eligibility.

## 5. Enable static website hosting

1. Open the bucket and choose **Properties**.
2. Scroll to **Static website hosting** and choose **Edit**.
3. Enable hosting and choose **Host a static website**.
4. Set the index document to `index.html`, matching capitalization exactly.
5. Leave the error document unset unless you have created and deployed one.
6. Save the settings.
7. Copy the **Bucket website endpoint** shown in this section.

Use the website endpoint to visit the page, rather than assuming the normal S3 API URL is the website URL. Direct S3 website endpoints support HTTP. For HTTPS, a later extension can use CloudFront and a private S3 origin; that is a different access setup.

## 6. Configure public website reads: S3 bucket policy

Use a dedicated bucket containing only material intended for public viewing.

1. Open **Permissions > Block public access > Edit**.
2. Clear **Block all public access** for this website bucket and save the change.
3. Account-level Block Public Access can still prevent a public bucket policy. If blocked, review that setting and its effects on other buckets before changing it. CloudFront with a private bucket is an alternative.
4. Open **Permissions > Bucket policy > Edit**.
5. Replace `YOUR_BUCKET_NAME` below with the exact bucket name and save:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicWebsiteRead",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
    }
  ]
}
```

This policy allows public object reads. It does not grant public uploads, deletion, or bucket listing. Keep ACLs disabled; no `public-read` ACL is needed.

## 7. Create the GitHub OIDC identity provider

1. Open **IAM > Identity providers**.
2. If the provider already exists, reuse it.
3. Otherwise choose **Add provider > OpenID Connect**.
4. Set the provider URL to `https://token.actions.githubusercontent.com`.
5. Set the audience to `sts.amazonaws.com`.
6. Add the provider.

For this account, its ARN is:

```text
arn:aws:iam::459640517588:oidc-provider/token.actions.githubusercontent.com
```

Open the provider's **Audiences** tab and verify that `sts.amazonaws.com` appears as the actual audience value.

OIDC supplies temporary credentials. No AWS access key ID or secret access key is needed in GitHub.

## 8. Create the S3 upload permission policy

1. Open **IAM > Policies > Create policy > JSON**.
2. Paste the following after replacing `YOUR_BUCKET_NAME`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UploadWebsite",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/index.html"
    }
  ]
}
```

3. Name it `GitHubDeployHtmlPolicy` and create it.

This grants the single-object upload used by `aws s3 cp`. The workflow does not use `sync`, list objects, delete files, or set ACLs, so those permissions are not included.

## 9. Create the IAM role and trust policy

For a new role, open **IAM > Roles > Create role > Custom trust policy**. For the existing role, open **GitHubHtmlDeployRole > Trust relationships > Edit trust policy**.

Use this policy for this repository:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::459640517588:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:SalmanOkz@188548630/DSA_LAB_SHU@1380848991:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Attach `GitHubDeployHtmlPolicy` and name the role `GitHubHtmlDeployRole` if creating it. Its ARN is:

```text
arn:aws:iam::459640517588:role/GitHubHtmlDeployRole
```

### Why the numeric IDs matter here

The actual diagnostic output from run #3 contained:

```text
iss: https://token.actions.githubusercontent.com
aud: sts.amazonaws.com
sub: repo:SalmanOkz@188548630/DSA_LAB_SHU@1380848991:ref:refs/heads/main
repository: SalmanOkz/DSA_LAB_SHU
ref: refs/heads/main
```

The earlier trust condition `repo:SalmanOkz/DSA_LAB_SHU:ref:refs/heads/main` did not match this token and caused the role-assumption failure. Use the observed subject above, without broad wildcard permissions. Other repositories or subject configurations may emit different subjects; inspect their actual claims instead of reusing these IDs.

### Three policies/settings serve different purposes

| Policy or setting | Purpose | Location |
| --- | --- | --- |
| S3 bucket policy | Allows visitors to read website files | S3 bucket > Permissions |
| IAM permission policy | Allows the deployment role to upload `index.html` | IAM policy attached to role |
| IAM trust policy | Allows this GitHub identity to assume the role | IAM role > Trust relationships |
| OIDC provider audience | Registers the expected token audience | IAM > Identity providers |

An upload permission policy cannot repair a role-assumption error. Authentication must succeed before S3 upload permissions are evaluated.

## 10. Add GitHub repository secrets

Open:

https://github.com/SalmanOkz/DSA_LAB_SHU/settings/secrets/actions

Choose **New repository secret** for each entry:

| Secret name | Value |
| --- | --- |
| `AWS_REGION` | Actual bucket region code |
| `AWS_ROLE_ARN` | `arn:aws:iam::459640517588:role/GitHubHtmlDeployRole` |
| `S3_BUCKET_NAME` | Exact bucket name, without `s3://` or a URL |

The YAML uses `secrets.NAME` because these values were stored under **Secrets**. If storing values under **Variables** instead, the YAML must use `vars.NAME`. Do not mix these contexts.

Updating a secret does not trigger a workflow. Start or re-run a workflow afterward. Updating an AWS policy also does not trigger GitHub Actions.

## 11. Add the complete deployment workflow

Create or replace `.github/workflows/deploy.yml` with:

```yaml
name: Deploy HTML to AWS S3

on:
  push:
    branches:
      - main
    paths:
      - index.html
      - .github/workflows/deploy.yml
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

concurrency:
  group: production-s3-deployment
  cancel-in-progress: true

jobs:
  deploy:
    name: Validate and deploy
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Check deployment settings
        env:
          AWS_REGION: ${{ secrets.AWS_REGION }}
          AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
          S3_BUCKET_NAME: ${{ secrets.S3_BUCKET_NAME }}
        run: |
          for key in AWS_REGION AWS_ROLE_ARN S3_BUCKET_NAME; do
            if [ -z "${!key}" ]; then
              echo "::error::Missing repository secret: $key"
              exit 1
            fi
          done

      - name: Validate index.html
        run: |
          if [ ! -s index.html ]; then
            echo "::error::index.html is missing or empty"
            exit 1
          fi
          echo "index.html exists and is not empty."

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v6.3.0
        timeout-minutes: 2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}
          audience: sts.amazonaws.com
          retry-max-attempts: 2

      - name: Upload index.html to S3
        env:
          S3_BUCKET_NAME: ${{ secrets.S3_BUCKET_NAME }}
        run: |
          aws s3 cp index.html "s3://${S3_BUCKET_NAME}/index.html" \
            --content-type "text/html; charset=utf-8" \
            --cache-control "no-cache"

      - name: Deployment completed
        run: echo "Website uploaded successfully."
```

Notes:

- `contents: read` permits repository checkout.
- `id-token: write` permits requesting an OIDC token; it does not grant AWS permissions by itself.
- The settings check reports missing secret names without printing their values.
- HTML validation checks only presence and nonzero size. It is not a full HTML syntax check or browser test.
- `cancel-in-progress: true` favors the latest deployment when new commits arrive.
- The job timeout starts when execution begins, not while waiting in the queue.
- AWS retries are limited to avoid lengthy retries for a persistent permission mismatch.
- The upload replaces the existing `index.html` object and asks browsers to revalidate cached content.
- The checkout version is retained from the working configuration. Run #3 logged a Node 20 deprecation warning and said GitHub forced it to Node 24; checkout still succeeded. That warning was not the authentication failure.

## 12. Commit and start the pipeline

### Through GitHub's website

1. Open or create `.github/workflows/deploy.yml`.
2. Paste the YAML and commit to `main`.
3. Open **Actions > Deploy HTML to AWS S3**.
4. Select the latest run and click the **Validate and deploy** job.
5. Expand individual steps to see their logs.

### Through local Git

From your already-cloned repository:

```bash
git add index.html .github/workflows/deploy.yml
git commit -m "Configure HTML deployment to S3"
git push origin main
```

### Manual run or retry

- New run: **Actions > Deploy HTML to AWS S3 > Run workflow > main > Run workflow**.
- Retry a completed failure: open the failed run and choose **Re-run all jobs**.
- A re-run uses that run's original workflow revision. If you changed YAML, use the new commit's run or manually start a run on current `main`.

## 13. Verify the website and automatic updates

1. Confirm the credentials and upload steps succeed.
2. Open S3 and verify `index.html` is present in the bucket root.
3. Open the website endpoint from **Properties > Static website hosting**.
4. Edit a visible heading in `index.html`.
5. Commit and push the change to `main`.
6. Wait for the new run to succeed, then refresh the website.

Changes to files other than `index.html` and the named workflow do not trigger this workflow's push filter. A README-only commit will not deploy.

## 14. Optional OIDC diagnostic step

If AWS rejects the role assumption, insert this step immediately before **Configure AWS credentials**. It prints only selected identity fields, not the authentication token:

```yaml
      - name: Diagnose GitHub OIDC identity
        shell: python
        run: |
          import os
          import json
          import base64
          import urllib.parse
          import urllib.request

          url = os.environ["ACTIONS_ID_TOKEN_REQUEST_URL"]
          separator = "&" if "?" in url else "?"
          url += separator + urllib.parse.urlencode({
              "audience": "sts.amazonaws.com"
          })

          request = urllib.request.Request(
              url,
              headers={
                  "Authorization": (
                      "Bearer "
                      + os.environ["ACTIONS_ID_TOKEN_REQUEST_TOKEN"]
                  )
              }
          )

          with urllib.request.urlopen(request, timeout=30) as response:
              token = json.load(response)["value"]

          payload = token.split(".")[1]
          payload += "=" * (-len(payload) % 4)
          claims = json.loads(base64.urlsafe_b64decode(payload))

          for key in ["iss", "aud", "sub", "repository", "ref"]:
              print(f"{key}: {claims.get(key)}")
```

Decoding these claims is for diagnosis; AWS performs signature and trust validation. Remove this optional step after resolving authentication.

## 15. Troubleshooting

### Input required and not supplied: aws-region

- Create the `AWS_REGION` repository secret with a nonempty actual region code.
- Ensure the workflow uses `${{ secrets.AWS_REGION }}`.
- If the value is a repository variable, use `${{ vars.AWS_REGION }}` instead.
- Re-run after saving; saving the value does not start a run.

### Not authorized to perform sts:AssumeRoleWithWebIdentity

- Confirm `AWS_ROLE_ARN` is the full role ARN, not only the account ID or the permission-policy ARN.
- Edit the trust policy on that exact role.
- Confirm the OIDC provider exists in account `459640517588`.
- Confirm provider audience and trust-policy audience both equal `sts.amazonaws.com`.
- Compare the diagnostic `sub` with the trust policy character for character.
- For this repository, use the numeric-ID subject from section 9.
- Keep `id-token: write` in the workflow.
- A different branch, environment, or customized subject may need a different narrowly scoped trust condition.

### AccessDenied during S3 upload

- Replace `YOUR_BUCKET_NAME` in `GitHubDeployHtmlPolicy` with the real bucket name.
- Confirm the policy is attached to `GitHubHtmlDeployRole`.
- Confirm `S3_BUCKET_NAME` matches that policy's bucket.
- The resource must include `/index.html` for this single-object policy.
- Do not add public write access or administrator permissions to solve this error.

### Website returns 403

- Confirm the upload succeeded and `index.html` exists.
- Check the public-read bucket policy and effective Block Public Access settings.
- Confirm the index document is exactly `index.html`.
- Use the S3 website endpoint.

### Workflow queued or displaying an old status

- Queued means execution has not started; changing AWS permissions cannot make a queued job start.
- Open the job details to distinguish queueing from a running step.
- Avoid repeatedly pushing new commits while diagnosing the same queue.
- Cancel obsolete runs when possible and retain the latest intended run.
- Check https://www.githubstatus.com/ for Actions incidents.
- If cancellation says the run is not in progress, refresh and compare job logs with the displayed run status.
- An old run retains its original concurrency settings even if a later commit changes the YAML.

On October 19, 2026, GitHub reported an Actions and Pages incident at 15:49 UTC (8:49 PM Pakistan time). During this session, runs #4 and #5 were queued, and run #3's logs had completed despite its status still showing in progress. Treat this as historical context, not a statement about GitHub's status when reading this guide later.

## 16. Completion checklist

- [ ] `index.html` is committed at the repository root.
- [ ] S3 bucket name and region are recorded.
- [ ] Static website hosting is enabled with `index.html` as the index document.
- [ ] Public website reads are configured for this dedicated bucket.
- [ ] OIDC provider audience includes `sts.amazonaws.com`.
- [ ] Role trust policy matches the actual numeric-ID subject.
- [ ] Upload policy contains the real bucket name.
- [ ] Upload policy is attached to `GitHubHtmlDeployRole`.
- [ ] All three GitHub repository secrets are present.
- [ ] Workflow on `main` uses the `secrets` context.
- [ ] AWS credential configuration succeeds.
- [ ] S3 upload succeeds.
- [ ] Website opens through the website endpoint.
- [ ] A second HTML edit automatically deploys successfully.

At the time this guide was prepared, the trust-policy correction had been reported saved, but a successful deployment after that correction had not yet been confirmed.

## References

- Repository: https://github.com/SalmanOkz/DSA_LAB_SHU
- Workflow: https://github.com/SalmanOkz/DSA_LAB_SHU/actions/workflows/deploy.yml
- AWS S3 static website tutorial: https://docs.aws.amazon.com/AmazonS3/latest/userguide/HostingWebsiteOnS3Setup.html
- S3 website permissions: https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteAccessPermissionsReqd.html
- GitHub OIDC with AWS: https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws
- AWS credentials action: https://github.com/aws-actions/configure-aws-credentials
- GitHub service status: https://www.githubstatus.com/
