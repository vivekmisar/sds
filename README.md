## Portfolio Contact & Vercel Deployment Guide

This markdown is **just for you** to remember how your contact form works and how to deploy the project to Vercel with everything configured.

---

## 1. How the Contact Form Works

- **Frontend component**: `src/components/contact.tsx`
  - Shows the **Get In Touch** section.
  - On submit it calls:

    ```ts
    fetch("/api/contact", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ name, email, message }),
    });
    ```

  - If the response is `200 OK` it:
    - Opens the **“Message Sent!”** popup.
    - Resets the form.
  - If the response is **not OK**, it shows the alert:
    - `"Failed to send message. Please try again."`

- **Backend handler**: `api/contact.ts`
  - This is a **Vercel Serverless Function** (uses `@vercel/node`).
  - Accepts `POST` requests with JSON body: `{ name, email, message }`.
  - Validates that all fields are present.
  - Calls `sendMail` from `lib/mail.ts` to send the email.
  - Behaviour:
    - If `sendMail` succeeds → returns `200` with `"Message sent successfully"`.
    - If `sendMail` fails **and** `EMAIL_USER` is not set:
      - Returns `200` with `"Message received (Email not configured)"`  
        (frontend still shows success, but no real email is sent).
    - For other failures → returns `500` with `"Failed to send email"` or `"Internal server error"`.

- **Mail utility**: `lib/mail.ts`
  - Uses `nodemailer` with Gmail (or any SMTP) to send emails.
  - Reads credentials from environment variables:

    ```env
    EMAIL_USER=your-email@gmail.com
    EMAIL_PASS=your-app-password
    ```

  - If these are **missing**, it logs a warning and returns `false` so you know mail is not really being sent.

---

## 2. Why the Form Fails on `npm run dev`

- `npm run dev` runs the **Vite dev server** at `http://localhost:5173`.
- Vite only serves the **frontend** – it does **not** serve your `api/contact.ts` serverless function.
- When the browser calls `POST /api/contact` on `localhost:5173`:
  - That route does not exist → it returns `404` or a network error.
  - `res.ok` is `false` → the code shows: **“Failed to send message. Please try again.”**

The code is correct for Vercel; the issue is that the dev server you are using doesn’t know about `/api/contact`.

To test the contact form correctly **locally**, you must use **`vercel dev`** instead of `npm run dev`.

---

## 3. Local Development with Working Contact Form

### 3.1 Install and log in to Vercel CLI (one time)

```bash
npm install -g vercel
vercel login
```

### 3.2 Optional: Local `.env` for email

Create a file named `.env.local` (or `.env`) in the project root:

```env
EMAIL_USER=your-email@gmail.com
EMAIL_PASS=your-app-password
```

> For Gmail, create an **App Password** (requires 2‑step verification) and use that as `EMAIL_PASS`. Do **not** use your regular Gmail password.

### 3.3 Start the dev environment with API support

From the project root:

```bash
vercel dev
```

- This command:
  - Serves your **Vite frontend**.
  - Serves your **serverless functions** in the `api/` folder, including `/api/contact`.
- Usually it runs on `http://localhost:3000`.

Now when you submit the contact form:

- Request goes to `POST http://localhost:3000/api/contact`.
- The serverless function runs exactly like it will on Vercel.
- If `EMAIL_USER` and `EMAIL_PASS` are set, you will actually receive an email.

> If you still run `npm run dev`, expect the contact form to fail because `/api/contact` does not exist there.

---

## 4. Preparing Email Credentials (Gmail Example)

1. Turn on **2‑Step Verification** on your Google account.
2. Open **Security → App passwords**.
3. Create a new app password, e.g. name it `Portfolio`.
4. Google gives you a 16‑character password (no spaces).
5. Use:
   - `EMAIL_USER = your-email@gmail.com`
   - `EMAIL_PASS =` the 16‑character app password

You will use the **same values** both locally (in `.env.local`) and on Vercel (in Environment Variables).

---

## 5. Deploying to Vercel (Step‑by‑Step)

These steps assume the project is in a Git repository (GitHub/GitLab/Bitbucket).

### 5.1 Push code to Git

1. Initialize git if needed:

   ```bash
   git init
   git add .
   git commit -m "Initial portfolio"
   ```

2. Create a GitHub repository and push:

   ```bash
   git remote add origin https://github.com/your-username/your-portfolio.git
   git push -u origin main
   ```

### 5.2 Create the Vercel project

1. Go to `https://vercel.com` and log in.
2. Click **“Add New → Project”**.
3. Choose your portfolio repository and click **Import**.

Vercel will usually auto-detect the settings, but you can verify:

- **Framework Preset**: Vite (or “Other” with custom build).
- **Build Command**: `npm run build`
- **Output Directory**: `dist`
- **Root Directory**: project root (where `package.json` and `vite.config.ts` are).

### 5.3 Configure environment variables on Vercel

In your Vercel project:

1. Go to **Settings → Environment Variables**.
2. Add:
   - `EMAIL_USER` = your Gmail address (same as local).
   - `EMAIL_PASS` = your Gmail app password.
3. Set them for the **Production** environment (and optionally Preview/Development).
4. Click **Save**.

These will be available to `lib/mail.ts` and `api/contact.ts` at build and runtime.

### 5.4 Trigger the deployment

- If this is the first import, Vercel will auto‑deploy right after you click **Deploy**.
- For future changes:
  - Push changes to your main branch.
  - Vercel will automatically build and deploy.

You can also click **Deploy** or **Redeploy** from the Vercel dashboard if needed.

### 5.5 What Vercel does during deploy

1. Installs dependencies with `npm install`.
2. Runs `npm run build` → Vite builds the React app into the `dist/` folder.
3. Detects the `api/` folder:
   - Builds `api/contact.ts` as a **serverless function**.
   - Mounts it at `/api/contact` on your production URL.
4. Serves:
   - Static assets from `dist/`.
   - API requests through the serverless function.

Your production URL will look like:

- `https://your-project-name.vercel.app`

---

## 6. Verifying the Contact Form on Production

1. Open your Vercel URL in the browser.
2. Scroll to the **Get In Touch** section.
3. Fill in **Name**, **Email**, and **Message**.
4. Click **Send Message**.
5. Expected behaviour:
   - The page does **not** show the “Failed to send message” alert.
   - A popup appears: **“Message Sent!”**.
   - You receive an email at `EMAIL_USER` with the form contents.

If the popup appears but you **do not** get an email:

- Check the **Vercel Logs** for your project.
- Look for messages from `lib/mail.ts`:
  - If it says **“Email credentials not found. Skipping email send.”** → env vars are missing or mis‑typed.
  - If there is an SMTP error, double‑check `EMAIL_USER` and `EMAIL_PASS`.

---

## 7. Other Ways People Can Contact You

Even without the form, your contact section already exposes:

- **Direct email**: `aniketjumde55@gmail.com`
- **Phone**: `+91 93569 91161`
- **GitHub**: `https://github.com/aniketjumde`
- **LinkedIn**: `https://www.linkedin.com/in/aniket-jumde-74275a289/`
- **LeetCode**: `https://leetcode.com/u/AniketJumde55/`

The form just gives them a friendlier way to reach out while still landing in your email inbox.

---

## 8. Quick Checklist (When Something Feels Broken)

- [ ] Are you using **`vercel dev`** (not just `npm run dev`) when testing the form locally?
- [ ] Did you create `EMAIL_USER` and `EMAIL_PASS` in your local `.env`?
- [ ] Did you add the same env vars in **Vercel → Settings → Environment Variables**?
- [ ] Does the form work on the Vercel URL even if it fails on `localhost:5173`?

If all of the above are true, the contact form should be fully working both locally (via `vercel dev`) and on Vercel.


