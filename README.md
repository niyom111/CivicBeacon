# CivicBeacon

AI-powered civic issue reporting: upload a photo and location, and the system detects the issue, figures out the right municipal department, drafts a complaint, and sends it automatically.

Live (Production): https://beacon-civic-voice-mr6eyhaf2-niyomdedhia33-5933s-projects.vercel.app

---

## Problem Statement
Citizens find it hard to report local civic issues (potholes, garbage, broken streetlights). Reporting is manual, slow, and requires figuring out which department to contact and what to write.

## Our Approach (How we tackled it)
- Simple web form for image + address + description.
- Edge function processes the request and calls a WorqHat AI workflow.
- WorqHat analyzes the image, summarizes the issue, picks the department and policy reference, and an email is auto-composed and sent to the authority.
- User gets immediate success feedback; no sign-in required.

## Tech Stack
- Frontend: Vite + React + TypeScript, Tailwind, shadcn/ui, lucide-react, react-router.
- Backend: Supabase Edge Function (Deno) acting as a thin API wrapper to WorqHat.
- AI Automation: WorqHat Flow (image analysis, text generation, routing, email).
- Hosting: Vercel (static front-end with SPA rewrites).

---

## Frontend (UI/UX)
Key component: `src/components/ReportForm.tsx`
- Inputs: file (image), address (text), description (textarea).
- Image preview before submission.
- POSTs `multipart/form-data` to the edge function endpoint.
- Toast feedback, loading state, and success card.

Styling notes:
- Built with shadcn/ui primitives; file input styled to align the native “Choose File” button inside the upload container.

### Local run
```
# from beacon-civic-voice/
npm install
npm run dev
```
App runs on http://localhost:5173.

### Required env (frontend)
Create `.env` in `beacon-civic-voice/`:
```
VITE_SUPABASE_URL=<your-supabase-project-url>
```
The frontend calls `${VITE_SUPABASE_URL}/functions/v1/submit-report`.

---

## Backend (Edge Function) + WorqHat Workflow

### WorqHat Workflow (Screenshot)
<img width="1262" height="367" alt="worqhat-workflow" src="https://github.com/user-attachments/assets/2721b7a9-0ac8-4ef1-acf7-d169cab9e03d" />

Implementation: `supabase/functions/submit-report/index.ts` (Deno).
- Accepts `multipart/form-data`: `file` (image), `address`, `description`.
- Forwards these to a WorqHat Flow (file-based flow endpoint) with Bearer `WORQHAT_API_KEY`.
- Returns the WorqHat response as JSON to the frontend.

### Required env (function)
- `WORQHAT_API_KEY` (stored in the function environment).

### API Contract
- Endpoint: `POST /functions/v1/submit-report`
- Body: `multipart/form-data`
  - `file`: image
  - `address`: string
  - `description`: string
- Success: `200 { success: true, data: <worqhat_response> }`
- Error: `4xx/5xx { error: string, details?: any }`

### WorqHat Flow: CivicBeacon (Detailed Workflow)
CivicBeacon is an AI-powered workflow built using WorqHat that allows citizens to easily report local issues (like potholes, garbage, or streetlight failures).
Users upload a photo and type the address — the system automatically detects the issue, identifies the relevant municipal department, generates a prefilled complaint, and emails it to the concerned authority.

⚙️ Workflow Structure

1️⃣ File Upload Trigger
- Purpose: Starting point of the workflow.
- Inputs:
  - `file`: image of the issue (uploaded by user)
  - `Address`: typed address (user input field)
- Output: Passes both inputs to the next node.

2️⃣ Image Analysis
- Node Type: Image Analysis (WorqHat AI Node)
- Description: Analyze uploaded images to detect the type of civic issue.
- Prompt (in “Add your question”):
  - Identify what civic issue is shown in the image.
  - Return only the issue name in short form (e.g., “pothole”, “garbage pile”, “broken streetlight”, etc.)
- Image Input: `Input: file`
- Output Example:
```
{
  "detected_issue": "pothole"
}
```

3️⃣ Text Generation 1
- Purpose: Generate a short issue summary using image output + user address.
- Prompt:
```
Create a one-line description of the civic issue based on this input:
Issue: {{image-analysis.output.detected_issue}}
Location: {{file-upload-trigger-initial.input.Address}}
```
- Example Output: `"A pothole was found near Rander Road, Surat."`

4️⃣ Text Generation 2
- Purpose: Auto-match the detected issue to the correct municipal department and mention related policy or ordinance.
- Prompt:
```
You are a civic automation assistant.
Match the issue to the correct municipal department in Surat and cite an example of the policy section that covers it.

Example output format:
{
  "department_name": "Road Maintenance Department",
  "policy_section": "Surat Municipal Corporation Road Maintenance Act - Section 14"
}

Input:
Issue: {{image-analysis.output.detected_issue}}
Location: {{file-upload-trigger-initial.input.Address}}
```

5️⃣ Email Node
- Purpose: Automatically send a prefilled complaint email to the detected department.
- Fields:
  - Sender Name: CivicBeacon System
  - Recipient Address: test email (e.g. punemuncipality@gmail.com)
  - Subject: `New Civic Issue Report`
  - Body:
```
Dear {{text-generation-2.output.department_name}} Team,

A new civic issue has been reported through CivicBeacon.

📍 Location: {{file-upload-trigger-initial.input.Address}}
📝 Issue Summary: {{text-generation-1.output.detected_issue}}
🏛️ Policy Reference: {{text-generation-2.output.policy_section}}

Please review and take appropriate action.
This complaint has been automatically generated to assist in faster redressal.

Thank you,
CivicBeacon Automated System
```

6️⃣ Return State
- Purpose: End node confirming workflow completion.
- Output Type: OK (200)

---

## Frontend ↔ Backend Integration
- The form posts `multipart/form-data` to the Supabase function.
- The function relays to WorqHat and returns JSON.
- Frontend shows success/failure and clears the form on success.

Sequence:
1. User selects image + enters address/description → clicks Submit.
2. Frontend `fetch` → Supabase Edge Function: `/functions/v1/submit-report`.
3. Edge Function → WorqHat Flow (file upload + inputs).
4. WorqHat analyzes, picks department, sends email, returns JSON.
5. Function returns `{ success: true, data }` → UI shows confirmation.

---

## Deployment (Vercel)
Static frontend is deployed to Vercel using `vercel.json` with SPA rewrites to `index.html`.

Commands (from `beacon-civic-voice/`):
```
# one-time login
npx vercel login

# build locally into .vercel/output
yarn install # or npm install / pnpm install
npx vercel build --prod

# deploy prebuilt output to production
npx vercel deploy --prebuilt --prod --yes
```

Important:
- Ensure `.env` contains `VITE_SUPABASE_URL` in Vercel Project → Settings → Environment Variables.
- Edge Function hosting is on Supabase (deploy functions there); the frontend just calls the URL.

Live URL (Production):
- https://beacon-civic-voice-mr6eyhaf2-niyomdedhia33-5933s-projects.vercel.app

---

## Future Improvements
- Real department email directory and city-wise routing.
- Status tracking dashboard for submitted reports.
- Map pin / reverse geocoding for addresses.
- Accessibility and mobile UX audits.
