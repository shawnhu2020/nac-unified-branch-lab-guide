# Part A — CI/CD Workflows with GitHub Actions

In this lab, you will use [GitHub Actions](https://docs.github.com/en/actions) to deploy and validate two Unified Branch networks — entirely from the cloud, with no software installed on your local machine. You will trigger a full CI/CD pipeline, explore each automated stage, and deliberately introduce configuration errors to see the pipeline catch them.

!!! info "What you need"
    - A **GitHub account** (free tier is fine)
    - Your **Meraki Dashboard account** and lab organization access
    - A **browser** — that's it

---

## Learning Objectives

By the end of Part A, you will be able to:

- Fork a Network as Code repository and configure it for your lab pod
- Store API key securely in GitHub and understand how it flow into CI/CD pipelines
- Understand the NaC YAML data model — pods variables, templates, and the `!env` tag
- Trigger and observe a multi-stage GitHub Actions pipeline (Prepare → Validate → Plan → Deploy → Test)
- Read pipeline artifacts including the merged NaC configuration and Terraform plan
- Run syntax and semantic validation workflows and understand pass/fail behavior
- Understand how scheduled integration tests enable continuous Day-2 drift detection

---

## Pipeline Overview

The full CI/CD pipeline runs these stages in sequence:

```
┌─────────┐   ┌──────────┐   ┌──────┐   ┌────────┐
│ Prepare │ → │ Validate │ → │ Plan │ → │ Deploy │
└─────────┘   └──────────┘   └──────┘   └────────┘
                                              │
                              ┌───────────────┴───────────────┐
                              ▼                               ▼
                    ┌──────────────────┐     ┌───────────────────────┐
                    │ Integration Test │     │   Idempotency Test    │
                    └──────────────────┘     └───────────────────────┘
```

| Stage | What it does |
|-------|-------------|
| **Prepare** | Merges all YAML data model files into a single `merged_configuration.nac.yaml` and uploads it as an artifact |
| **Validate** | Checks Terraform formatting and runs `nac-validate` to validate the merged config against the NaC schema |
| **Plan** | Runs `terraform plan` and saves the plan — shows exactly what will be created/changed |
| **Deploy** | Runs `terraform apply` using the saved plan — creates the branch networks and claims the hardware |
| **Integration Test** | Runs Robot Framework tests against the live Meraki Dashboard to verify the deployed config matches the data model |
| **Idempotency Test** | Runs `terraform plan` again after deploy — expects zero changes, confirming the pipeline is idempotent |

---

## Step 1 — Log In and Identify Your Lab Organization

1. Open [https://dashboard.meraki.com](https://dashboard.meraki.com) and sign in with the email address from your lab reservation.

2. If your account has access to multiple organizations, click the organization name in the top-left corner and select the lab organization — it follows the naming pattern **Org-[org_id]**.

    !!! danger "Important"
        To confirm you are in the correct lab org, navigate to **Organization > Administrators**. You should see an admin named **ENLS** with the email **launchpad-labs-ops@cisco.com**.

<!-- 3. Note down your full organization name (e.g. `Org-3665367146726165048`) — you will need it in Step 4. -->

---

## Step 2 — Read Serial Numbers from Network Notes

The Launchpad Labs chatbot has already loaded your branch hardware serial numbers into the **Network notes** field of the Datacenter network.

1. In the Dashboard, navigate to the **Datacenter** network.

2. Navigate to **Network-wide > General** and scroll to the **Network notes** field.

3. You should see your lab organization name and serial numbers for branches in this format:

    ``` { .bash .no-copy }

        #Copy and paste the following into .env of your git repository.
        org_name=Org-XXXXXXXXXXXXXXXXXXX

        # ── Device serials (per-branch)
        #
        branch1_mx_serial=XXXX-XXXX-XXXX
        branch1_ms_serial=XXXX-XXXX-XXXX
        branch1_cw_serial=XXXX-XXXX-XXXX
        branch2_mx_serial=XXXX-XXXX-XXXX
        branch2_ms_serial=XXXX-XXXX-XXXX
        branch2_cw_serial=XXXX-XXXX-XXXX
    ```

4. Copy the entire note — you will paste them into the `.env` file in Step 4.

    !!! danger "Important"
        Devices can only be claimed into one organization at a time. Use only the serial numbers assigned to your lab pod and only claim them to your lab organization.

---

## Step 4 — Fork the Repository

Forking creates your own copy of the lab repo under your GitHub account. This is where you will store your secrets, configure your lab pod details, and trigger workflows.

1. Go to the lab repository:

    ```
    https://FINAL-REPO-TO-BE-UPDATED
    ```

2. Click **Fork** (top-right corner of the page).

3. On the fork screen, leave all defaults and click **Create fork**.

    !!! info "Information"
        Your fork will be at `https://github.com/YOUR_GITHUB_USERNAME/bac-lab`. All subsequent steps happen in your fork — not the original repo.

---

## Step 5 — Generate Your Meraki API Key

The pipeline authenticates to Meraki using a personal API key tied to your dashboard account. Follow these steps to generate one. **If you already have an API key for your lab account, skip this step.**

1. In the Meraki Dashboard, click your **profile icon** in the top-right corner and select **My profile**.

    ![Meraki Dashboard profile menu](assets/media/image11.png){ width="60%" }

2. Scroll down to the **API access** section and click **Generate new API key**. 

3. Copy the key immediately and save it somewhere safe (e.g. a password manager). **It will only be displayed once** — if you navigate away without copying it, you will need to revoke it and generate a new one.

    ![Meraki Dashboard profile menu](assets/media/image12.png){ width="60%" }

    !!! warning "Keep your API key private"
        Your API key has the same access level as your dashboard login. Never share it, commit it to a repository, or paste it into chat. You will store it securely as a GitHub Secret in the next step.

---

## Step 6 — Add Your Meraki API Key as a GitHub Secret

The pipeline reads your API key from a GitHub Secret — an encrypted value that GitHub injects into workflow runs at runtime. It is never stored in the repository files.

1. In your fork, click **Settings** (top navigation tab).

2. In the left sidebar, go to **Secrets and variables > Actions**.

3. Click **New repository secret** and set:

    | Field | Value |
    |-------|-------|
    | **Name** | `MERAKI_API_KEY` |
    | **Secret** | Your Meraki API key |

4. Click **Add secret**.

    !!! info "How the pipeline uses it"
        The workflow file references `${{ secrets.MERAKI_API_KEY }}` and injects it as `MERAKI_API_KEY`, `MERAKI_DASHBOARD_API_KEY`, and `TF_VAR_meraki_api_key` — all three environment variables that the Terraform provider and NaC module require.

---

## Step 7 — Configure Your Lab Pod Details in .env

The `.env` file at the root of the repo stores non-secret variables that the pipeline loads at runtime: your org name, serial numbers, and hub network name. Unlike the API key, these are committed to the repo.

1. In your fork on GitHub, click on the `.env` file to open it, then click the **pencil icon** (Edit) to edit it in the browser.

2. Fill in your values and replace those lines for the same variables at the top of the file:

    ``` { .bash .no-copy }
    #Copy and paste the following into .env of your git repository.
    org_name=Org-XXXXXXXXXXXXXXXXXXX

    # ── Device serials (per-branch)
    #
    branch1_mx_serial=XXXX-XXXX-XXXX
    branch1_ms_serial=XXXX-XXXX-XXXX
    branch1_cw_serial=XXXX-XXXX-XXXX
    branch2_mx_serial=XXXX-XXXX-XXXX
    branch2_ms_serial=XXXX-XXXX-XXXX
    branch2_cw_serial=XXXX-XXXX-XXXX
    ```

3. Click **Commit changes**. Use the default commit message and commit directly to `main`.

    !!! warning "Note"
        Notice that the API key is **not** in this file. The `.env` file only holds non-sensitive variables. The API key lives exclusively in GitHub Secrets.

---

## Step 8 — Create a GitHub Environment

The **Plan** and **Deploy** jobs reference a GitHub Environment for scoped secrets and optional approval gates.

1. In your fork, go to **Settings > Environments**.

2. Click **New environment** and name it. Then click **Configure environment**.

    ```
    unified-branch-network-as-code
    ```

    !!! warning "Note"
        This name must match exactly. The pipeline workflow file (`pipeline.yml`) hardcodes `environment: unified-branch-network-as-code` in the Plan, Deploy, Integration Test, and Idempotency Test jobs. If the environment does not exist in your repo settings under this name, those jobs will fail waiting for an environment that GitHub cannot find.

3. Leave Deployment branches and tags with **No restriction**.

    !!! info "Information"
        Notice the **Deployment branches and tags** setting on this page. By default, all branches and tags are allowed to trigger a deployment to this environment. In a real-world pipeline you would restrict this — for example, allowing only the `main` branch to deploy to production. This prevents feature branches from accidentally deploying infrastructure changes.


---

## Step 9 — Run the Deploy Pipeline

You are now ready to trigger the full CI/CD pipeline.

1. In your fork, click the **Actions** tab.

2. In the left sidebar, click **Deploy Small Branch as Code**.

3. Click **Run workflow** (top-right dropdown), keep `main` branch selected, and click **Run workflow**.

4. Monitor the workflow status changes for each stage. 
<!-- Refresh the page and click on the new workflow run to open it. -->

### Follow Each Stage

As the pipeline runs, expand each job to observe what is happening:

**Prepare:**

- Terraform downloads the NaC model module and merges all YAML data files into `merged_configuration.nac.yaml`
- When complete, click **Artifacts** and download `merged-config` — open the file to see the fully rendered configuration with all template variables substituted

**Validate:**

- `terraform fmt -check` verifies all Terraform files are correctly formatted
- `nac-validate` validates the merged configuration against the NaC schema
- Look for `✅ nac-validate passed` in the output

**Plan:**

- `terraform plan` shows exactly what will be created: two networks (`Unified Branch 1` and `Unified Branch 2`), device claim resources, VLANs, firewall rules, SSIDs, switch profiles, and more
- Download the `plan-output` artifact to review the full plan

**Deploy:**

- `terraform apply` executes the plan — this is where the Meraki API calls happen
- The two branch networks are created and your physical MX85, MS250, and CW9172H devices are claimed

    !!! warning "Note"
        On your **first run** you will see a yellow warning step inside the Deploy job: `Unable to download artifact(s): Artifact not found for name: terraform-state-pre-deploy`. This is expected — no prior Terraform state exists yet, so there is nothing to restore. The step is configured with `continue-on-error: true` and the Deploy job will still show a green ✅. This warning disappears on all subsequent runs once a state file exists.

**Integration Test:**

- Robot Framework (`nac-test`) runs test cases against the live Meraki Dashboard
- Look at the **Job Summary** for a pass/fail count table
- Download `integration-test-results` and open the HTML report in your browser for the full test output

**Idempotency Test:**

- `terraform plan` runs again — it should show `0 to add, 0 to change, 0 to destroy`
- This confirms that running the pipeline twice does not cause unintended changes

!!! warning "Note"
    The full pipeline typically takes **5–10 minutes**. Do not close the tab — watch each job complete in sequence.

---

## Step 10 — Verify in the Meraki Dashboard

1. Return to [https://dashboard.meraki.com](https://dashboard.meraki.com) and confirm you are in the lab organization.

2. In the network selector, you should now see:
    - **Unified Branch 1**
    - **Unified Branch 2**

3. Select **Unified Branch 1** and verify:
    - **Network-wide > General** — time zone and address match the data model
    - **Security & SD-WAN > Addressing & VLANs** — Data, Voice, IoT, Guest, and Infra VLANs present
    - **Wireless > SSIDs** — Data and Guest SSIDs configured
    - **Security & SD-WAN > VPN Status**  - Status of remote VPN peer Datacenter should be in green 
    - **Security & SD-WAN > Appliance Status**, then navigate to **Tools > Ping**, configure the following and click `Ping`. You should see successful ping proving the VPN tunnel is up between branch and datacenter.
        - Source IP Address: `VLAN 10`
        - Destination IP Address: `10.255.20.1` (Gateway IP for Voice VLAN on Datacenter side)
    !!! note
        The VPN connectivity between branch networks and Datacenter might take a couple minutes.

4. Check **Organization > Inventory** to confirm MX85, MS250, and CW9172H for both branches are claimed and assigned to the correct networks.

    !!! tip "Hint"
        Navigate to **Organization > Change Log** to see the full API call log. You will see `POST api/v1/networks/{id}/devices/claim` alongside all the configuration calls — exactly what Terraform executed on your behalf.

---

## Step 11 — Explore Syntax Validation

The **Syntax Validation** workflow validates your YAML data files against `schema.yaml` using `iac-validate`. In this step you will break a schema rule and watch the pipeline catch it.

### Run the Baseline Check

1. Go to **Actions > Run Syntax Validation** and click **Run workflow**. Run the workflow from **Branch: main**.
2. Confirm it passes: look for **Success** status and `✅ Syntax Validation` .

### Introduce a Violation

1. In your fork, edit file `schema.yaml` and find:

    ```yaml
    organizations:
      name: str(min=1, max=128, required=False)
    ```

2. Change `max=128` to `max=10` and commit directly to `main`.

3. Go to **Actions > Run Syntax Validation** and run it again.

4. The job should **fail** — your org name (e.g. `Org-3665367146726165048`) is longer than 10 characters. Look for `ERROR` annotations in the job log and download the `syntax-validate-output` artifact for details.

!!! note
    If the **Artifacts** section is not visible, refresh the browser page.

### Restore

Change `max=10` back to `max=128` and commit.

!!! info "Key Takeaway"
    Syntax validation catches structural violations — wrong types, out-of-range values, required fields missing. It runs before any infrastructure changes are attempted, acting as the first automated guard rail.

---

## Step 12 — Explore Semantic (Business Rule) Validation

The **Semantic Validation** workflow uses custom Python rules in the `rules/` folder to enforce business policies that go beyond what a schema can express.

### Run the Baseline Check

1. Go to **Actions > Run Semantic / Business Rule** and click **Run workflow**.
2. Confirm it passes: `✅ Semantics Validate`.

### Examine the Rule

Edit `rules/101_admin_name.py`. This rule rejects any administrator named `root` — a common policy to prevent use of generic privileged account names.

### Introduce a Violation

1. Open `data/org_global.nac.yaml` and change the administrator name to `root`:

    ```yaml
    administrators:
      - name: root    # <-- changed from your actual name
    ```

2. Commit to `main` branch.

3. Go to **Actions > Run Semantic / Business Rule** and run it again.

4. The job should **fail**. Look for the `ERROR` line citing the admin name violation. In the Artifacts
 section, download `semantics-validate-output` for full details.

!!! note
    If the **Artifacts** section is not visible, refresh the browser page.


### Restore

Revert the admin name back to its original value `admin` and commit.

!!! info "Key Takeaway"
    Semantic validation is the second layer of protection. It catches violations that are syntactically valid YAML but violate your organization's policies. A file with `name: root` passes the schema but fails the business rule — these two layers work together.

---

## Step 13 — Scheduled Integration Tests

The **Scheduled Integration Test** workflow runs automatically every 6 hours via cron. It re-runs all Robot Framework tests against the live Dashboard to detect configuration drift — without deploying anything.

### Trigger Manually

1. Go to **Actions > Scheduled Integration Test** and click **Run workflow**.
2. Review the Job Summary and download the test results artifact.
3. On your computer, unzip the downloaded artifact and open `report.html` from the `tests/results` folder in your browser to review each test item and its result statistics.

!!! note
    If the **Artifacts** section is not visible, refresh the browser page.



### Look at the Schedule

Open `.github/workflows/scheduled-integration-test.yml` and find:

```yaml
on:
  schedule:
    - cron: '0 */6 * * *'   # every 6 hours
  workflow_dispatch:          # allow manual trigger
```

!!! info "Key Takeaway — Day-2 Operations"
    Scheduled tests answer a critical question: **"Has anyone manually changed the network since we last deployed?"**

    If a team member makes a change directly in the Meraki Dashboard (bypassing the pipeline), the next scheduled test will catch the drift and fail. Your team is alerted, and you can either restore the configuration (re-run the deploy pipeline) or update the data model to make the change official.

    This is the essence of **GitOps** — the Git repository is the single source of truth, and any deviation from it is treated as an anomaly.

---

## Part A Complete!

You have successfully:

- Forked the NaC repository and configured it for your lab pod
- Stored your Meraki API key securely in GitHub Secrets
- Triggered and observed a full 6-stage CI/CD pipeline
- Reviewed pipeline artifacts: merged NaC config, Terraform plan, and integration test HTML report
- Deployed two fully-configured Unified Branch networks via automation — without installing anything locally
- Verified the deployed networks in the Meraki Dashboard
- Introduced and observed a **syntax violation** (schema max length) caught by the Validate stage
- Introduced and observed a **semantic violation** (banned admin name) caught by the business rules engine
- Triggered a **scheduled integration test** and understood its role in Day-2 drift detection

---

!!! tip "Continue to Part B (Optional)"
    If you want hands-on experience running Terraform from your own laptop — including understanding the data model in depth and running `terraform plan` and `terraform apply` directly — continue to **[Part B — Network as Code with Terraform](PART-B-TERRAFORM.md)**.

    **Before starting Part B**, you must remove the two branch networks created in this lab so that Part B can deploy them from a clean state using local Terraform. Run the **Cleanup – Delete Branch Networks** workflow to do this:

    1. In your GitHub repository, go to **Actions > Cleanup – Delete Branch Networks** and click **Run workflow**.
    2. In the confirmation field, type `destroy` and click **Run workflow**.
    3. The **Plan Destroy** job runs first — review its Job Summary to confirm only **Unified Branch 1** and **Unified Branch 2** are listed for deletion. The Datacenter (hub) network will not be touched.
    4. The **Destroy Branch Networks** job starts automatically after the plan — wait for it to complete (green ✅).
    5. Verify in the Meraki Dashboard that both branch networks are gone.

    You are now ready to start **[Part B — Network as Code with Terraform](PART-B-TERRAFORM.md)**.
