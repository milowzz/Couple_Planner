# Couple Planner

A shared calendar and goals planner for two authorized users. The site is a single-page app in `index.html`; Firebase Authentication handles sign-in and Firestore syncs the shared planner between devices.

## Features

- Month calendar for plans, optional times, and partner ownership
- Goals timeline with start date, target date, owner, and progress
- Filters for all items, either partner, or shared items
- Personalization for names, colors, appearance, heading font, and paper texture
- Google sign-in and email/password accounts with email verification
- Shared, real-time Firestore updates for the two authorized accounts
- Export of plans and goals to an `.ics` calendar file
- URL state for the selected view, filters, and month

## Project Structure

- `index.html` contains the markup, styles, app logic, and Firebase client configuration.
- `README.md` contains setup, development, and deployment instructions.

There is no build step or package manager required. Firebase's compat SDK is loaded from Google's CDN, so a network connection is required for authentication and cloud sync.

## Requirements

- Git for Windows
- Python 3 for local HTTP testing (or an equivalent local web server)
- A Firebase project with Authentication and Cloud Firestore enabled
- The GitHub repository connected to the Netlify site

## Get the Project

Clone the repository once:

```powershell
git clone YOUR_REPOSITORY_URL
cd YOUR_REPOSITORY_FOLDER
code .
```

Replace the repository URL and folder placeholders with your repository's details.

If `git` is not recognized after installing Git for Windows, fully close and reopen VS Code so its terminal refreshes PATH.

## Firebase Setup

### 1. Create or select the Firebase project

Create a project at [Firebase Console](https://console.firebase.google.com/). Analytics is optional. Use the same Firebase project whose web configuration is in `index.html`.

### 2. Enable sign-in providers

In **Build → Authentication → Sign-in method**, enable:

- Google
- Email/Password

Email/password accounts must verify their email before the planner opens. Google accounts can enter after Google sign-in succeeds.

### 3. Authorize the site domains

In **Authentication → Settings → Authorized domains**, add every origin used to open the app, including:

- `localhost` for local development
- your Netlify site's hostname for the deployed site

Add any custom domain here too. A missing domain causes `auth/unauthorized-domain` during sign-in.

### 4. Register the Firebase web app

In **Project settings → General → Your apps**, register a Web app if one is not already registered. Copy its Firebase web configuration into the `firebaseConfig` object in `index.html`.

The Firebase web API key is client configuration and is visible in the page source. Restrict it in Google Cloud Console to the app's HTTP referrers and the required APIs. Never put a service account key or other private server credential in `index.html` or GitHub.

### 5. Create the Firestore database

In **Build → Firestore Database**, create the database. Do not leave test-mode rules deployed. While authentication and the final rules are being prepared, a deny-all rule is a safe temporary state; the app will continue using browser-local storage, but cloud sync will be unavailable.

### 6. Create both accounts and copy their UIDs

Use the app's **Create an account** form for Email/Password or **Continue with Google**. Verify any email/password account from its verification email. After each person has signed in, copy that account's UID from **Authentication → Users**.

The app allows account creation, but Firestore access is restricted to the two UIDs in the rules below. New accounts do not automatically get access to the shared planner.

### 7. Publish Firestore security rules

Replace all three placeholders with the planner document ID and the two authorized users' exact UIDs from the same Firebase project. The document ID must match `PLANNER_ID` in `index.html`. Paste the complete rules into **Firestore Database → Rules**, then publish:

```text
rules_version = '2';

service cloud.firestore {
	match /databases/{database}/documents {
		match /planners/REPLACE_WITH_PLANNER_ID {
			allow read, write: if request.auth != null
				&& request.auth.uid in [
					'REPLACE_WITH_FIRST_AUTHORIZED_UID',
					'REPLACE_WITH_SECOND_AUTHORIZED_UID'
				];
		}

		match /{document=**} {
			allow read, write: if false;
		}
	}
}
```

Do not deploy with the placeholders still in place. The anonymous sign-in flow has been removed; the app must sign in with one of the two listed accounts. A rule that only checks `request.auth != null` is not sufficient because anyone can create an account, including an anonymous account if that provider is enabled.

## Run Locally

Firebase sign-in needs an HTTP origin, so do not test sign-in by opening `index.html` directly as a `file://` URL. In PowerShell, from the repository folder, run:

```powershell
py -m http.server 8000 --bind 127.0.0.1
```

Then open [http://localhost:8000](http://localhost:8000). Stop the server with Ctrl+C. Make sure `localhost` is listed under Firebase Authentication's authorized domains.

## Sign-In Workflow

- **Google:** Select **Continue with Google**. Desktop uses a popup; narrow/mobile layouts use a redirect.
- **Email/password:** Select **Create an account**, submit an email and password, then follow the verification email. Return to the app and choose **I verified my email**.
- **Returning account:** Enter the same email and password, or use Google with the same Google account.
- **Password recovery:** Enter the account email and select **Reset password**.
- **Sign out:** Use the **Sign out** button in the planner header.

## Data and Sync Behavior

- Every change is saved to that browser's local storage.
- Once an authorized user signs in, the app listens to the shared Firestore document and writes changes to it with a short debounce.
- When the shared document already exists, its state is loaded into the browser. If it does not exist, the first authorized sign-in seeds it from that browser's local state.
- The planner stores its whole state in one Firestore document. Simultaneous edits are last-write-wins; this version does not merge concurrent changes. Avoid editing from both devices at the exact same time.
- If Firestore access is denied or unavailable, the app reports a sync issue and local browser storage remains available.
- Browser-local data is separate on each device. Before first sync, decide which device's existing local data should seed the shared document. Export an `.ics` backup from the planner before testing if those plans matter.

## Test Before Relying on Sync

1. Sign in with the first authorized account on one browser/device and confirm the planner shows **Shared planner synced**.
2. Add a disposable test plan and wait for sync to complete.
3. Sign in with the second authorized account in a separate browser/device and confirm the test plan appears.
4. Add a second disposable plan as the second account and confirm it appears for the first account.
5. In Firestore Rules Playground, verify an unrelated UID is denied read and write access to your planner document.
6. Delete the test plans from the app after testing.

## Deploy to Netlify

The Netlify site is connected to the GitHub repository. Pushing to `main` triggers a deploy; no local build command is required.

```powershell
git status
git add index.html README.md
git commit -m "Update planner setup guide"
git push origin main
```

Check the **Deploys** page in the Netlify dashboard and confirm the latest deploy is **Published**. Test sign-in on your Netlify site after deployment.

## Troubleshooting

- **`auth/unauthorized-domain`:** Add the current hostname to Firebase Authentication → Settings → Authorized domains.
- **Signed in but Firestore access is denied:** Confirm the signed-in user's UID is in the published rules, the document ID matches `PLANNER_ID`, and the rules are published to the correct Firebase project/database.
- **Email account cannot open the planner:** Complete email verification, return to the page, and select **I verified my email**.
- **No cloud sync:** Check the visible sync status and browser console, then verify Firebase project configuration, network access to Firebase, and Firestore rules.
- **Netlify page not found:** Confirm the deployed repository has a lowercase `index.html` at its root and inspect the latest Netlify deploy log.
