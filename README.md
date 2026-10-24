# STI Alaminos Research Library

This web application runs on the computer hosting its server. By default, it stores accounts and paper records in a local SQLite database; it can optionally use MongoDB Atlas for those records. Uploaded research documents remain in the server computer's local `data\uploads\` folder. The library and keyword-based research assistant do not require an AI provider or paid API.

## Start the application

1. Install Node.js 22.13 or newer. The local SQLite fallback requires a recent Node.js version. If `node` is not recognized in a terminal after installation, restart VS Code so its terminal reloads the updated PATH.
2. Open PowerShell in the project folder:

   ```powershell
   cd "C:\Users\Marjorie\Documents\htmlprojj"
   ```

3. Install the free server dependencies once:

   ```powershell
   npm.cmd install
   ```

4. Start the website:

   ```powershell
   npm.cmd start
   ```

5. Visit <http://localhost:3000/research.html>. Leave the terminal open while using the app. Stop the server with `Ctrl+C`.

Do not open `research.html` directly as a `file://` page; use the local URL above so accounts, documents, and the database work.

## Accounts and permissions

- The library, search, assistant, paper previews, downloads, and submissions require a signed-in account. Visitors see only the account screen until they sign in.
- Create an account from the Log in window.
- The first account created on a new local database becomes Admin.
- Later self-registered accounts are Students by default.
- Students can submit research papers for Admin review. Pending papers are not visible in the library or downloadable until approved.
- Students can check pending, approved, and rejected submissions from **My submissions**.
- Admins can approve or reject submissions from **Review submissions**. Rejected documents are removed from local storage.
- Admins can create accounts, change roles, reset a user's password, or remove accounts.
- Faculty and Admin accounts can publish and manage papers directly.
- Signed-in users can download documents.

Passwords are hashed on the local server. Sessions are held in an HTTP-only cookie and expire after inactivity. This local application listens only on this computer; it is not configured for school-network or public access.

## Private access for groupmates with Tailscale

The app can be shared privately with trusted Tailscale users while it runs on this computer. Everyone needs to be invited to the same Tailscale network and have Tailscale connected. The host computer must stay on, connected to the internet, and running the app. For HTTPS access, Tailscale Serve proxies to the app on localhost; the app still binds only to `127.0.0.1`.

1. In the Tailscale admin console, invite each trusted groupmate to your tailnet. Have each person install Tailscale on their own computer and sign in with their own invited account. Do not share your account password.
2. Enable MagicDNS if needed. In the Tailscale admin console, find your computer's MagicDNS name. It looks like `computer-name.your-tailnet.ts.net`; use its HTTPS origin, for example `https://computer-name.your-tailnet.ts.net`.
3. Create a local environment file in the project folder, which is excluded from source control:

   ```powershell
   Copy-Item .env.example .env
   ```

   Open `.env` in VS Code and set `PUBLIC_ORIGIN` to your exact HTTPS MagicDNS origin. Keep `HOST=127.0.0.1`. Use `ALLOW_PASSWORD_SIGNUP=true` for username/password accounts; set it to `false` only if Google sign-in is fully configured and you want to restrict account creation to invited Google test users.
4. Start the app in the VS Code terminal:

   ```powershell
   $env:HOST = "127.0.0.1"
   npm.cmd start
   ```

5. In the same computer's terminal, enable the private HTTPS proxy:

   ```powershell
   tailscale serve --bg 3000
   tailscale serve status
   ```

   Confirm Tailscale shows the same `https://...ts.net` address you entered as `PUBLIC_ORIGIN`. This is private to your tailnet; do not enable Funnel.
6. In Google Cloud Console, create a project and configure its OAuth consent screen for **External** users in **Testing** mode. Add each groupmate's Google email under **Test users**. Google sign-in is limited to the test users you add until you complete Google's review and publish the app.
7. Create an OAuth client with application type **Web application**. Add this exact **Authorized redirect URI**, replacing the example host with your MagicDNS name:

   ```text
   https://computer-name.your-tailnet.ts.net/auth/google/callback
   ```

   Copy the client ID and secret into `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` in `.env`. Never share the client secret or commit `.env`.
8. Restart the app after editing `.env`. Groupmates connect Tailscale, then visit `https://computer-name.your-tailnet.ts.net/research.html` and choose **Continue with Google**. Only the Google accounts added as test users can sign in while the consent screen is in Testing mode.

Google sign-in creates new accounts as Students after the existing Admin account is set up. Existing password accounts aren't automatically linked to a Google identity. Keep the server terminal open and Tailscale connected while the group uses the app. Do not enable Tailscale Funnel, router port forwarding, or Windows firewall access from public networks. Invite only people you trust; anyone in the tailnet who knows the app address can reach its login page. Tailscale encrypts the connection between tailnet devices, but this is still a small private-group setup, not school-approved public hosting. Ask the school before storing real student data.

To switch back to computer-only access, stop the server and clear `PUBLIC_ORIGIN`, `GOOGLE_CLIENT_ID`, and `GOOGLE_CLIENT_SECRET` in `.env`, then run `npm.cmd start`. To move to another host later, back up the database and uploads together.

## Documents and backup

Upload PDF, DOCX, or TXT files up to 10 MB. File contents and paper metadata persist on this computer:

- SQLite database: `data\research.sqlite`
- Uploaded files: `data\uploads\`

When MongoDB Atlas is enabled, account and paper metadata are stored in Atlas instead of SQLite, but uploaded documents stay in `data\uploads\` on the server computer. Keep that folder available and back it up. The `data` folder is intentionally excluded from source control.

## Optional MongoDB Atlas database

Atlas stores account and paper metadata in the cloud; it does not host this website or the uploaded documents. Your computer must still be on, connected to Tailscale, and running the app for groupmates to use it. Before switching, make a backup of the entire `data` folder and check that cloud storage is allowed for your group's research data.

1. Create a free Atlas cluster and a database user with a strong, unique password.
2. In **Network Access**, allow the server computer's current public IP address. Do not use `0.0.0.0/0` for this private group setup. If your internet provider changes your public IP, update the Atlas access list.
3. Copy the cluster's Node.js connection string. Keep its password private and URL-encode special characters in it as Atlas instructs.
4. Put the connection string in the local `.env` file as `MONGODB_URI=...`. Leave `MONGODB_DATABASE=sti_research_hub`, save the file, and restart the server.
5. On the first connection, the app automatically imports users, sessions, and paper metadata from `data\research.sqlite` only if the selected Atlas database is empty. The import does not copy uploaded files; keep `data\uploads\` on the server. The local SQLite database is left unchanged as a pre-migration snapshot, but changes made after switching to Atlas are not written back to it.

The app refuses to import into an Atlas database that already contains app data without its import marker, rather than overwriting that data. Back up both the local database and uploads before switching. Do not share or commit `.env`, which contains the Atlas connection string. A free cluster should not be treated as the only backup of important work.

## Research assistant

The assistant runs free, local keyword and phrase matching across saved paper titles, authors, topics, courses, and abstracts. It can surface papers, show abstracts, and provide basic catalog comparisons. It is not a generative AI model and does not send paper information to an external service.

## Limits

This is a small private-group application when accessed over Tailscale. It is not hardened for public or school-network hosting. Those deployments require approval and additional security review, HTTPS, backups, stronger account recovery, and deployment-specific changes.
