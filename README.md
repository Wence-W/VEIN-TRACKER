# Vein Clinic Tracker

The Vein Pre-Authorization & Treatment Tracker as a standalone web app. Staff open it at its own URL in any browser. Records are stored in one shared database, so every computer sees the same data. Sign-in uses one-time email links, and only authorized emails get in.

The tabs, forms, dropdowns, vein schema, PO milestones, search, dashboard, chart and Excel export all match the tracker. It includes every change made on the GitHub Pages version (BBA ID as the record identifier, Patient name on the form, authorization status, the AASV vein), plus:

- **Shared records.** Everything any user adds or changes appears for everyone, within about 15 seconds.
- **Deleting records.** Each record has a Delete button, in its row and inside the record form. Treatment entries can be deleted one at a time. Every delete asks for confirmation first, and deleted records disappear for everyone. A deleted record can't come back through syncing or by re-importing an old backup. The person who deleted it can undo the delete for a few seconds.
- **No demo data.** The app never creates sample records. When it starts, it removes any sample records that were brought in from older versions: records marked as demo, or named like "Demo Patient B". Real records are never touched.
- **Multiple treatments per patient.** Each treatment session is its own entry, with a treatment date, a post-treatment U/S date, and the procedure performed on each vein. Units stay in the treatment plan. The spreadsheet export adds a Treatments sheet with one row per session.
- **Initial U/S.** A status (Not received, Requested, Received, Reviewed, Not needed) and a date on every tab except Promo. Promo keeps its US Date field.
- **Shared notifications.** When another user adds, changes, moves, approves or deletes a record, you get notified. Each notification shows the BBA ID, what changed and when. Notifications appear in these places:
  - **Bell icon:** in the header, with the number of unread notifications.
  - **Sidebar:** a badge on the tab the record is in. It's green when it includes a new approval.
  - **Dashboard:** a "new" badge on the summary cards, and a What's new panel.
  - **Table:** a label next to the record's BBA ID.

  Clicking a notification opens the record and marks it read. "Mark all as read" clears everything. Read status is saved per user on the server, so it's the same on every computer you sign in from. Your own changes don't notify you.

The rest is unchanged from the first Google Cloud version:

- A shared Google Cloud database replaces each browser's local storage.
- Real sign-in replaces the in-page email check. The quick-login buttons are gone.
- Records sync between computers about every 15 seconds, and immediately when you switch back to the tab.
- If two people edit the same record at once, the second person is warned instead of silently overwriting the first.
- Every change is recorded in an activity log, visible to administrators.
- Users are signed out automatically after 30 minutes without activity.
- There are two access levels. Staff work with patient records. Administrators can also manage users, settings, imports and backups.
- Records from the original browser-only tracker can be imported.

This is a new project. It doesn't touch the original Beltline tracker or any other project.

## How it fits together

```
GitHub (private repo) ──push──▶ Cloud Build ──deploys──▶ Cloud Run (this app)
                                                            │
                         staff browsers ◀── HTTPS ──────────┤
                                                            ├──▶ Firestore (patient records, users, audit log)
                         sign-in links ◀── Identity Platform┘
```

| Piece | What it does | Under the Google Cloud BAA |
|---|---|---|
| Cloud Run | Hosts the app and its API | Yes |
| Firestore | Stores records, users, settings, activity log | Yes |
| Identity Platform | Email sign-in links | Yes |
| Cloud Build | Builds the app from GitHub (code only, no patient data) | — |
| GitHub | Stores the code only, never patient data | — |

Browsers never talk to the database directly. Every request goes through the server, which checks the sign-in token, checks the email against the authorized list, and records the action.

> Use the Google Cloud product names above. Firebase-branded services (Firebase Authentication, Firebase Hosting) are **not** covered by Google's BAA. Identity Platform uses the same sign-in technology and **is** covered.

## Costs

Google bills by usage. For about 10 staff and a few thousand records, expect only a few dollars a month, often close to $0. Most of the usage comes from each open tracker checking for updates every 15 seconds; with 10 people working full days, that may add a dollar or two a month in database reads. Check Google's current pricing pages before relying on this. The usage-based parts are Cloud Run, Firestore, Identity Platform, Cloud Build and Artifact Registry. Accepting the BAA is free.

Set a budget alert (step 1) so you're emailed if anything unexpected happens.

---

## Deployment, step by step

Allow about an hour. You need a Google account that can create a project with the practice's billing account, and a GitHub account. The person who accepts the BAA in step 2 must be authorized to sign agreements for the practice.

Google moves menu items now and then. If a menu path below doesn't match, type the page name into the search bar at the top of the Google Cloud console.

### 1. Create the Google Cloud project

1. Go to <https://console.cloud.google.com>. Create a new project named `vein-clinic-tracker`.
2. Link the practice's billing account. Identity Platform requires billing.
3. Go to **Billing → Budgets & alerts** and create a budget of $25/month with email alerts.
4. Write down the **Project ID** shown on the dashboard, for example `vein-clinic-tracker-123456`.

### 2. Accept the Google Cloud BAA (do this before any real patient data goes in)

1. Search the console for **BAA**, or go to **IAM & Admin → Privacy & Security**.
2. Review the Google Cloud Business Associate Amendment and accept it for the practice.
3. Save a copy of the acceptance for your HIPAA records.

### 3. Turn on the services

Go to **APIs & Services → Enable APIs and services** and enable each of these:
- Cloud Run Admin API
- Cloud Build API
- Artifact Registry API
- Cloud Firestore API
- Identity Toolkit API (this is Identity Platform)

### 4. Create the database

1. Go to **Firestore → Create database**.
2. Choose **Native mode**, database ID `(default)`, and location `us-east1` (South Carolina).
3. Open the **Rules** tab, paste in the contents of `firestore.rules` from this project, and publish. This blocks all direct browser access; only the app's server can reach the data.
4. Recommended: under **Disaster recovery**, turn on **point-in-time recovery** and a **daily backup schedule**.

### 5. Set up sign-in (Identity Platform)

1. Go to **Identity Platform** and enable it if prompted.
2. Open **Providers → Add a provider → Email / Password**. Turn it on, tick **Passwordless login (email link)**, and save.
3. Select **Application setup details** at the top right and copy two values:
   - `apiKey`, which becomes `IDP_API_KEY`
   - `authDomain`, which becomes `IDP_AUTH_DOMAIN` (looks like `your-project.firebaseapp.com`)

   Neither value is a secret. They identify the project to the sign-in service.
4. Leave this page open. You'll add the app's URL in step 9.

### 6. Create the app's service account

1. Go to **IAM & Admin → Service accounts → Create service account**.
2. Name it `vein-tracker-run`.
3. Grant it the role **Cloud Datastore User**. That's the only role it needs.

### 7. Put the code on GitHub

1. Create a **new, private** repository named `vein-clinic-tracker`. Don't reuse an existing repository.
2. Upload every file in this folder, including the hidden `.github` folder and `.gitignore`. With git:
   ```bash
   cd vein-clinic-tracker
   git init && git add . && git commit -m "Vein Clinic Tracker"
   git branch -M main
   git remote add origin https://github.com/YOUR-ACCOUNT/vein-clinic-tracker.git
   git push -u origin main
   ```
   Without git, choose **Add file → Upload files** on GitHub and drag in the folder's contents.
3. The **Actions** tab will run a check (tests plus a build). It should show a green tick.

**Never commit patient data.** That includes backups, exported spreadsheets and screenshots of records. `.gitignore` blocks the common file types, but stay careful.

### 8. Deploy to Cloud Run from GitHub

1. Go to **Cloud Run → Deploy container → Service**.
2. Choose **Continuously deploy from a repository (source or function)** and select **Set up with Cloud Build**.
   - Connect your GitHub account, then choose the `vein-clinic-tracker` repository.
   - Branch: `^main$`. Build type: **Dockerfile**, at path `/Dockerfile`.
3. Fill in the service settings:
   - **Service name:** `vein-clinic-tracker`
   - **Region:** `us-east1`
   - **Authentication:** **Allow public access** (also called "allow unauthenticated invocations"). The app has its own sign-in; this setting only lets browsers reach the sign-in page.
   - **Billing:** request-based. **Minimum instances:** 0. **Maximum instances:** 3.
4. Under **Containers, Volumes, Networking, Security**:
   - On the **Security** tab, set **Service account** to `vein-tracker-run`.
   - On the **Variables & Secrets** tab, add these variables:

     | Name | Value |
     |---|---|
     | `PROJECT_ID` | your Project ID from step 1 |
     | `IDP_API_KEY` | from step 5 |
     | `IDP_AUTH_DOMAIN` | from step 5 |
     | `ADMIN_EMAILS` | comma-separated administrators, e.g. `Dianaw@beltlinehealth.com,Corbin@smb-sc.com` |
     | `IDLE_MINUTES` | `30` (optional; minutes of inactivity before sign-out) |

5. Select **Create**. The first build takes about 3–5 minutes. When it finishes, the page shows the app's URL, something like `https://vein-clinic-tracker-xxxxxxxxxx.us-east1.run.app`.

From now on, every push to `main` on GitHub redeploys automatically.

### 9. Allow sign-in from the app's URL

1. Go back to **Identity Platform → Settings → Security → Authorized domains**.
2. Add the Cloud Run host, without `https://`, for example `vein-clinic-tracker-xxxxxxxxxx.us-east1.run.app`.
3. Save.

### 10. First sign-in

1. Open the URL in Chrome and enter an email listed in `ADMIN_EMAILS`.
2. Open the link that arrives by email, on the same computer.
3. You're in, as an administrator. Open **Authorized users** to review the list. The original 10 emails are pre-loaded as Staff. Promote anyone else who should manage users.
4. Share the URL with your team. They don't need Claude accounts or Google accounts; any email address on the authorized list works.

### 11. Bring over records from the original tracker

The original tracker kept records inside each browser, and the published Claude artifact kept a separate copy. Repeat this for **each computer and browser** where records were entered:

1. Open the original tracker there.
2. Go to **Settings → Download backup** and save the `.json` file.

Then, in the new app, as an administrator:

1. Go to **Settings → Import from the original tracker**.
2. Choose each backup file in turn.

Demo records are skipped. Records already imported are never duplicated or overwritten, so importing the same file twice is safe. Records that are missing a name on a tab that requires one are placed on **Procter Pay Attention**, with a note saying which tab they came from.

Delete the backup files from those computers afterwards; they contain patient information.

### 12. Move over from the GitHub Pages version

The GitHub Pages tracker (wence-w.github.io/VEIN-TRACKER) saves records separately in each browser. Nothing is copied automatically, and nothing is deleted.

1. On **each computer** where staff entered records there, open the GitHub Pages tracker and go to **Settings → Download backup**.
2. In the new app, sign in as an administrator. Go to **Settings → Import from the original tracker** and import each backup file.

The import keeps names, approvals (with their original dates) and AASV entries. Records already imported are skipped, so importing overlapping backups is safe. Records without a BBA ID on a tab that requires one are placed on **Procter Pay Attention**, with a note saying where they came from.

After everyone has switched, replace the GitHub Pages `index.html` with a short page that forwards to the new address, so the old link keeps working. Ask for this once you have the Cloud Run URL, and it can be prepared for you.

### 13. Optional: clean up old notifications automatically

Notifications older than 30 days stop showing in the app. To also delete them from the database, go to **Firestore → Time-to-live (TTL)**, choose **Create policy**, and enter collection group `events` and timestamp field `expireAt`. Each notification then deletes itself 90 days after it was created.

### 14. Optional: your own address

To use an address like `tracker.beltlinehealth.com`, use **Cloud Run → Manage custom domains** (or put a load balancer in front, if that region doesn't support domain mapping). After that, add the new domain to Identity Platform's authorized domains.

---

## Running it locally (for developers)

```bash
npm install
npm run dev        # http://localhost:8080
```

`npm run dev` uses an in-memory database and a development sign-in box. Both are refused when `NODE_ENV=production`, so they can't be switched on in the deployed app. Use made-up data only.

`npm test` runs the record-validation tests.

## Security checklist

- [ ] Google Cloud BAA accepted and filed (step 2)
- [ ] Firestore rules deny all browser access (step 4)
- [ ] Firestore backups / point-in-time recovery on (step 4)
- [ ] Only 1–3 administrators in `ADMIN_EMAILS`
- [ ] Authorized users list reviewed; remove people the day they leave
- [ ] Staff told that exported spreadsheets and backups contain PHI and must stay on clinic devices
- [ ] Budget alert set (step 1)
- [ ] GitHub repository is private, and only the people who maintain the app have access

This setup covers the technical side. HIPAA also expects the practice to have a risk assessment, written policies, and staff training. Your compliance officer or HIPAA consultant should review the deployment before real patient data is entered.

## Troubleshooting

- **"Can't reach the tracker server."** In Cloud Run, open the service's **Logs** tab. A missing variable from step 8 shows up there as an error at startup.
- **No sign-in email arrives.** Check spam. Then confirm Email link is enabled (step 5) and the app's domain is authorized (step 9).
- **"This email is not on the authorized list."** An administrator needs to add the email under **Authorized users**.
- **Locked out of admin.** Anyone in `ADMIN_EMAILS` is always made an administrator again when the app restarts. Edit the variable in Cloud Run, which redeploys the app.
- **Cloud Run won't allow public access.** The organization may have a "domain restricted sharing" policy. An org administrator can make an exception for this one project.
