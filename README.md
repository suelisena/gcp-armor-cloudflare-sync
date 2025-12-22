# 🛡️ GCP Armor Cloudflare Sync (Budget Edition)

> "Because we don't have $3,000/month for Cloud Armor Enterprise."

### 🧐 What is this?
If you’re anything like me, seeing the **$3,000/month** price tag for Google Cloud Armor Enterprise is enough to make your heart skip a beat. You’re in the right place.

This project is a **Cloud Run Job** deployed in a **Tier-1 region (us-central1)**. It periodically fetches the latest Cloudflare IP ranges and neatly (and **for free**) updates your Cloud Armor firewall rules.

### 💰 Why this approach?
- **Cost-effective**: Deployed in `us-central1`, using minimal resources ~~(**128MiB RAM**)~~, running once a week — the cost is practically **$0.00**.
- **Low maintenance**: No more manually pasting 100+ IP ranges until your hands hurt.
- **Secure**: Strictly enforces Cloudflare-only access to your Load Balancer, effectively ghosting direct-to-IP attackers.

> [!IMPORTANT]
> **2025-12-22 Update:** As of now, the minimum memory allocation for Cloud Run Jobs has been raised to **512MiB**. But we’ve updated our deployment specs accordingly to comply with Google’s new floor while still keeping costs at a literal $0.00. ~~Fuck~~Nice try, Google

---

## 🌐 The IPv6 Manifesto (Why only IPv6?)
Our company is strictly **IPv6-only** for this project. This isn't just a technical choice; it's a statement driven by three cold, hard facts:

1. **Vibe Check**: Honestly? IPv6 just looks cooler. It's the "cyberpunk" of IP addresses.
2. **Financial Reality**: IPv6 is the future, and more importantly, it's the "free" part of the future. We are budget ninjas.
3. **Cloud Armor Constraints**: Cloudflare's IPv4 list is massive (way more than 10 ranges). **Google Cloud Armor has a hard limit of 10 IP ranges per rule.** To avoid creating multiple rules (which complicates management and adds unnecessary overhead), we sync the compact IPv6 list into a single, elegant rule. One rule to rule them all.

---

### 🛠️ Tech Stack
- **Bash & Gcloud CLI**: Why compile Go when a shell script does the job? (Switched to Bash for maximum simplicity).
- **Docker (Slim)**: Using the official Google Cloud CLI slim image to keep things light.
- **Cloud Run Jobs**: Executes and vanishes. No idle costs, no mercy for Google's billing department.
- **us-central1**: The world’s cheapest digital tax haven.

---

## ⚙️ Configuration (Environment Variables)

> This step in the document may not necessarily be completed in one go. If you have any questions, please ask **Gemini** to help you solve it.

This container is generalized. You don't need to hardcode anything; just pass these variables to your Cloud Run Job:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `PROJECT_ID` | Your GCP Project ID | `glass-gasket-482010-v8` |
| `POLICY_NAME` | The name of your Cloud Armor Security Policy | `allow-cloudflare-ipv6-only` |
| `RULE_PRIORITY` | The priority of the rule to update (must exist) | `100` |

---

## 🔐 Security & IAM Setup

To make this work securely, you need a dedicated **Service Account**. Don't use your Owner account!

1. **Create the Service Account:**
   ```bash
   gcloud iam service-accounts create armor-updater-sa \
       --display-name="Cloud Armor Auto Updater"
   ```

2. **Assign the "Compute Security Admin" Role:**
   ```bash
   gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
       --member="serviceAccount:armor-updater-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
       --role="roles/compute.securityAdmin"
   ```

## 🤖 Continuous Integration (Cloud Build)
We use Cloud Build to automatically build the image on every git push.

⚠️ **Critical Note**: To avoid "Logs Bucket" permission errors, we use a `cloudbuild.yaml` with `logging: CLOUD_LOGGING_ONLY`.

1. **The cloudbuild.yaml**
   ```yaml
   options:
     logging: CLOUD_LOGGING_ONLY
   steps:
     - name: 'gcr.io/cloud-builders/docker'
       args: ['build', '-t', 'gcr.io/$PROJECT_ID/armor-updater:$COMMIT_SHA', '.']
   images:
     - 'gcr.io/$PROJECT_ID/armor-updater:$COMMIT_SHA'
   ```

2. **Setup the Trigger**
   In GCP Console, create a Trigger pointing to this repository and select Cloud Build configuration file (yaml) as the configuration type.

## 🚀 Deployment & Automation

1. **Deploy the Cloud Run Job**
   ```bash
   gcloud run jobs deploy cloudflare-ip-sync \
       --image gcr.io/YOUR_PROJECT_ID/armor-updater:latest \
       --region us-central1 \
       --service-account armor-updater-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com \
       --set-env-vars PROJECT_ID=YOUR_PROJECT_ID,POLICY_NAME=allow-cloudflare-ipv6-only,RULE_PRIORITY=100
   ```

2. **Schedule it (The "Set and Forget" Step)**
   Run this to sync every day at midnight:
   ```bash
   gcloud scheduler jobs create http trigger-armor-update \
       --location us-central1 \
       --schedule="0 0 * * *" \
       --uri="https://us-central1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/YOUR_PROJECT_ID/jobs/cloudflare-ip-sync:run" \
       --http-method POST \
       --oauth-service-account-email="armor-updater-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com"
   ```

## 📝 License
MIT. Go forth and save that $3,000.

