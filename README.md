# Digital Career Center CRM

A mobile-first internal CRM for Digital Career Center. It is a single React/Vite application connected directly to Supabase for authentication and persistent CRM data. Supabase Row Level Security limits employee access to currently assigned leads. There is no Express/API server, public employee signup, demo data, or service-role secret in the browser.

## Features

- Admin and employee dashboards with real Supabase counts and team sales.
- Admin-created employee accounts, generated `DCC001` IDs, activation, profile edits, and password resets.
- Lead search by name, phone, and WhatsApp; filters for status, employee, follow-up, source, and course.
- Lead assignment/reassignment, call outcomes, `tel:` calling, notes, and audit history.
- Follow-up today, upcoming, and overdue lists with completion/cancellation.
- Enrollment sales with the salesperson fixed from the authenticated account.
- Responsive desktop/mobile navigation and lead cards.
- PostgreSQL constraints, audit triggers, indexes, RLS, and Supabase Realtime.

## Stack

React, JavaScript, Vite, Tailwind CSS, Supabase Auth, Supabase Postgres/RLS, Supabase Edge Functions, Lucide icons.

## Requirements

- Node.js 20.19+ and npm.
- A Supabase project.

## 1. Create a Supabase project

1. Create a project in the Supabase Dashboard and choose a strong database password.
2. In **Project Settings → API**, copy the Project URL and the publishable/anon key. These two values are safe for a browser when RLS is enabled.
3. Do not copy, paste, or deploy the `service_role` key into this project or Vercel. Supabase makes it available only inside Edge Functions.
4. In **Authentication → Settings**, disable public signups. The app has no signup UI; disabling provider signups also prevents direct public account creation.

## 2. Create database tables and security policies

Open **SQL Editor → New query**, paste all of [`supabase/schema.sql`](supabase/schema.sql), and run it. It creates `profiles`, `leads`, `follow_ups`, `lead_history`, and `sales`, plus indexes, audit/follow-up/sale triggers, dashboard functions, Realtime publication membership, and RLS policies.

The RLS rules are the security boundary, not the UI. An active employee can select/update only leads assigned to their auth user ID; admins can manage all CRM records. Employee inserts cannot change assignment/contact ownership, follow-ups remain limited to their assigned leads, sales attribution is checked against `auth.uid()`, and lead history has no client insert/update/delete policy.

## 3. Configure the first administrator

There is no public signup. Create exactly one initial admin in the Supabase Dashboard:

1. In **Authentication → Users → Add user**, create a confirmed user with email `dccadmin@dcc.internal` and a strong admin password. Keep that password private.
2. In SQL Editor, run the profile bootstrap below, replacing the name if needed:

```sql
insert into public.profiles (id, user_id, full_name, role, active)
select id, 'DCCADMIN', 'DCC Administrator', 'ADMIN', true
from auth.users
where email = 'dccadmin@dcc.internal'
on conflict (id) do update
set user_id = excluded.user_id,
    full_name = excluded.full_name,
    role = 'ADMIN',
    active = true;
```

Sign in to the CRM with User ID `DCCADMIN` and the password used when you created the Supabase Auth user. The login Edge Function maps the User ID to the private Auth email; employees never enter or see that internal email.

## 4. Deploy Supabase Edge Functions

Employee account creation/reset needs the Supabase Auth Admin API. Its service-role access stays server-side in Edge Functions; it is never exposed to React.

Install/use the Supabase CLI, authenticate, link the project, and deploy from this repository root:

```powershell
npx supabase login
npx supabase link --project-ref <your-project-ref>
npx supabase functions deploy login --no-verify-jwt
npx supabase functions deploy manage-employee
```

The login function verifies User ID/password and returns a Supabase Auth session. `manage-employee` verifies the caller's active `ADMIN` profile before creating accounts or changing employee access. Supabase provides the function runtime credentials; do not add service-role credentials to `.env`, Vercel, or GitHub.

## 5. Configure and run locally

From this directory:

```powershell
npm install
Copy-Item .env.example .env
```

Set the two public values in `.env`:

```dotenv
VITE_SUPABASE_URL=https://<project-ref>.supabase.co
VITE_SUPABASE_ANON_KEY=<publishable-or-anon-key>
```

Then start Vite:

```powershell
npm run dev
```

Open the local URL Vite prints. If the public values are not configured, the app shows a setup screen instead of fabricated CRM data. Client-side values are public by design; database access is restricted by RLS.

## 6. Employee accounts

Sign in as `DCCADMIN`, open **Team → Add employee**, and set the employee's name and a temporary password. The database generates the next unique `DCC001`, `DCC002`, … ID. Share that ID/password through an approved secure channel. Admins can edit profile details, activate/deactivate accounts, and reset passwords; there is no employee self-registration.

Employees sign in with the generated ID and password, see only their assigned leads, and can call, update outcomes, add notes, schedule/complete follow-ups, and record an enrollment. The salesperson is derived from the signed-in profile and cannot be chosen in the UI.

## 7. Build and preview

```powershell
npm run build
npm run preview
```

## 8. Deploy from GitHub to Vercel

1. Push this project folder as the GitHub repository root. Confirm `.env` is not staged; `.gitignore` excludes it.
2. Import the GitHub repository into Vercel. Use the project root, build command `npm run build`, and output directory `dist`.
3. Add `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` in Vercel's Production environment variables, then deploy.
4. In the Supabase Dashboard **Authentication → URL Configuration**, add the Vercel URL to allowed redirect URLs. Password login does not redirect, but this keeps Auth project settings aligned.
5. Test the admin login, create an employee, verify employee lead isolation, and check mobile sign-in before sharing the URL.

The live URL is created by your Vercel account during deployment; it cannot be claimed or published without access to your Supabase and Vercel projects.

## Project structure

```text
.
  src/                 React UI and Supabase client
  supabase/schema.sql  Tables, RLS, triggers, dashboard RPCs
  supabase/functions/  Secure User ID login and admin account management
  .env.example
  package.json
```
