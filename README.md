# CampusKart

A campus second-hand marketplace. Sellers list items, an admin approves them before they go public, and buyers contact sellers directly to arrange the handoff.

## Stack

- **Backend:** Node.js, Express, MongoDB (Mongoose), JWT auth, Google Sign-In, Twilio Verify (SMS OTP)
- **Frontend:** React (Vite), Tailwind CSS, React Router, @react-oauth/google

## Project structure

```
campuskart/
  server/     Express API
  client/     React app
```

## 1. Run it locally

### Backend

```bash
cd server
npm install
copy .env.example .env      # Windows (or `cp` on Mac/Linux)
```

Edit `.env` — at minimum, set these to get the app running:
```
PORT=5000
MONGO_URI=mongodb://localhost:27017/campuskart
JWT_SECRET=some_long_random_string
CLIENT_URL=http://localhost:5173
```

`GOOGLE_CLIENT_ID`, `ALLOWED_GOOGLE_DOMAIN`, and the `TWILIO_*` variables can stay blank for now — see sections 3 and 4 below for how to fill them in. The app runs fine without them; you just won't be able to use Google Sign-In or phone verification until they're set.

```bash
npm run dev
```

### Frontend

```bash
cd client
npm install
copy .env.example .env
```

`.env`:
```
VITE_API_URL=http://localhost:5000/api
VITE_GOOGLE_CLIENT_ID=            # same Client ID as the server, see section 3
```

```bash
npm run dev
```

The app runs on `http://localhost:5173`.

## 2. Make yourself an admin

```bash
cd server
node seedAdmin.js your-email@example.com YourChosenPassword
```

This either creates a brand-new admin account with that email/password, or promotes an existing account with that email to admin (password left unchanged in that case). Log in with that email and password at `/login` using the "Log in with email and password" option.

## 3. Setting up Google Sign-In

1. Go to [console.cloud.google.com](https://console.cloud.google.com), create a project (or use an existing one)
2. Go to **APIs & Services → Credentials**
3. Click **Create Credentials → OAuth client ID**
4. If prompted, configure the OAuth consent screen first (External, fill in app name, your email — no need to submit for verification for local/small-scale use)
5. Application type: **Web application**
6. Under **Authorized JavaScript origins**, add:
   - `http://localhost:5173` (for local dev)
   - your production frontend URL once deployed (e.g. `https://campuskart.vercel.app`)
7. Copy the **Client ID** it gives you (looks like `123456789-abc...apps.googleusercontent.com`)
8. Paste it into **both**:
   - `server/.env` → `GOOGLE_CLIENT_ID`
   - `client/.env` → `VITE_GOOGLE_CLIENT_ID`
9. Restart both `npm run dev` processes

### Restricting to your college domain (optional)

If your college issues Google Workspace accounts on a specific domain (e.g. `stu.pes.edu`), you can set:
```
ALLOWED_GOOGLE_DOMAIN=stu.pes.edu
```
in `server/.env`. Anyone signing in with a Google account outside that domain will be rejected. **Only set this if you've actually confirmed your college's student emails are Google Workspace accounts on that exact domain** — if they're not, this will incorrectly block everyone. Leave it blank if unsure; Google Sign-In will still work for anyone with any Google account, just without the domain restriction.

## 4. Setting up phone verification (Twilio)

Real SMS verification costs money per message sent — there's no free way around this. Twilio gives new accounts some free trial credit, enough to test with.

1. Sign up at [twilio.com/try-twilio](https://www.twilio.com/try-twilio)
2. From the [Twilio Console](https://console.twilio.com), copy your **Account SID** and **Auth Token**
3. Go to **Verify → Services** → create a new Verify Service (any friendly name, e.g. "CampusKart")
4. Copy that service's **SID** (starts with `VA...`)
5. Add all three to `server/.env`:
   ```
   TWILIO_ACCOUNT_SID=AC...
   TWILIO_AUTH_TOKEN=...
   TWILIO_VERIFY_SERVICE_SID=VA...
   ```
6. Restart the server

**Trial account limitation:** Twilio trial accounts can only send SMS to phone numbers you've manually verified in the Twilio console first (Console → Phone Numbers → Verified Caller IDs). To send to *any* number, you need to upgrade to a paid account and add billing — plan for this before a real public launch.

Users verify their phone from the **Profile** page after logging in.

## 5. How the moderation flow works

1. A logged-in user creates a listing → saved as `status: "pending"`, not visible publicly
2. Shows up in `/admin` under "pending"
3. Admin approves (→ `approved`, now public) or rejects with a reason (→ `rejected`, shown to the seller)
4. Seller can edit a pending/rejected listing — saving resets it to `pending`
5. Seller marks it `sold` once gone — drops out of the public feed

## 6. Contact reveal

Buyers see a seller's phone/email only after tapping "Contact seller" on an approved listing. If the seller has verified their phone, buyers see a "✓ Verified" badge next to the number. Each reveal is logged in `ContactLog` (buyer, seller, item, timestamp).

## 7. Deploying

| Piece | Where |
|---|---|
| Database | [MongoDB Atlas](https://www.mongodb.com/atlas) free tier |
| API (`server/`) | [Render](https://render.com) or [Railway](https://railway.app) |
| Frontend (`client/`) | [Vercel](https://vercel.com) or [Netlify](https://netlify.com) |

Same as local setup, but:
- Add your production frontend URL to Google's **Authorized JavaScript origins** (step 3.6 above)
- Set `CLIENT_URL` on the server to your production frontend URL (for CORS)
- Set `VITE_API_URL` on the frontend to your production API URL
- All the same env vars (`MONGO_URI`, `JWT_SECRET`, `GOOGLE_CLIENT_ID`, `TWILIO_*`) need to be set on your hosting provider's dashboard, not just in a local `.env` file (which never gets deployed)

## 8. Known gaps / good next steps

- No password reset flow for password-based accounts (Google accounts don't need one)
- No in-app chat — contact is direct (phone/email)
- No rate limiting on listing creation or phone OTP requests — worth adding before public launch to control spam and Twilio costs
- No report/flag button on listings yet
- Twilio trial accounts require pre-verified numbers — real public use needs a paid Twilio account with billing enabled
