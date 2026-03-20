
Yes. For this demo, I’d build a **single-screen Expo React Native app** with **Supabase anonymous auth**, **one `plan_generations` table**, and **one Supabase Edge Function** called `generate-plan` that calls **Gemini 2.5 Flash** and returns **strict JSON**. That is the cleanest setup for your requirement because Supabase already gives you a browser/mobile-friendly data API on top of Postgres, while Edge Functions are specifically meant for low-latency authenticated endpoints and small AI orchestration. Gemini 2.5 Flash is a stable model with structured outputs support, function calling, and a 1M-token input window, and Google still offers free-tier pricing for it. ([Supabase](https://supabase.com/docs/guides/api "https://supabase.com/docs/guides/api"))

I would also **not** call Gemini directly from the app. Gemini requests require an API key, and Supabase recommends storing secrets in project secrets / environment variables for Edge Functions. The Gemini REST API works anywhere you can make HTTP requests, so it fits naturally inside a Deno-based Supabase Edge Function. ([Google AI for Developers](https://ai.google.dev/api "https://ai.google.dev/api"))

## What this demo should do

The app is one page and has one job:

1. User pastes or writes a task/problem into a large text box.
    
2. User taps **Generate Plan**.
    
3. The app sends the text to a Supabase Edge Function.
    
4. The function calls Gemini and asks for a **structured plan**.
    
5. The function saves the result in Supabase and returns it.
    
6. The app renders the same AI result in three synchronized views:
    
    - **Flowchart**
        
    - **Steps**
        
    - **Checklist**
        

This should feel like an **AI planning canvas**, not like a chat app.

## Scope for the demo

Keep the demo intentionally tight:

**In scope**

- one screen
    
- anonymous session
    
- text input
    
- choose preferred output format: `Auto`, `Flowchart`, `Steps`, `Checklist`
    
- AI-generated plan
    
- store the result in Supabase
    
- show loading / success / error states
    
- optional recent-history strip at the bottom
    

**Out of scope**

- speech-to-text
    
- reminders
    
- calendar sync
    
- collaboration
    
- drag-and-drop flow editing
    
- long-term memory across notes
    

## Overall feel of the app

The screen should feel:

- calm
    
- focused
    
- mobile-native
    
- productivity-oriented
    

A good UI mood is:

- large rounded input card at the top
    
- one strong accent color for the primary button
    
- soft neutral background
    
- result card under the input
    
- segmented control for `Flowchart / Steps / Checklist`
    
- friendly loading copy such as “Turning this into a plan…”
    

This should feel like: **“describe a messy task, get back a clean plan.”**

## Preferred architecture

For this demo, use a **feature-first, thin-client + one-function backend** architecture.

### On the client

Use the app for:

- anonymous sign-in
    
- text entry
    
- format selection
    
- calling the Edge Function
    
- rendering the result
    
- optionally reading saved plan history
    

### In Supabase

Use:

- **Auth** for anonymous sessions
    
- **Postgres** for saved plans
    
- **Edge Functions** for the Gemini call
    
- **RLS** so each anonymous or permanent user only sees their own plans
    

### In Gemini

Use:

- **Gemini 2.5 Flash**
    
- **structured output / JSON schema**
    
- no chat state
    
- one request in, one typed plan out
    

That architecture is ideal here because Supabase’s REST/Data API is auto-generated from your database schema and can be called directly from the client when RLS is enabled, while Edge Functions are the right place for secrets and AI orchestration. Google’s structured outputs let Gemini return schema-constrained JSON instead of loose prose. ([Supabase](https://supabase.com/docs/guides/api "https://supabase.com/docs/guides/api"))

## Why I’d use Gemini 2.5 Flash here

Use **`gemini-2.5-flash`** as the default model.

Why:

- it is **stable**
    
- it supports **structured outputs**
    
- it supports **function calling**
    
- it is designed for **low-latency / large-scale** use
    
- Google lists a **free tier** for it
    
- preview models have more restrictive rate limits
    
- older `gemini-2.0-flash` is deprecated and scheduled to shut down on **June 1, 2026**. ([Google AI for Developers](https://ai.google.dev/gemini-api/docs/models/gemini-2.5-flash "https://ai.google.dev/gemini-api/docs/models/gemini-2.5-flash"))
    

For a tiny demo, this is the right balance of quality and simplicity. If you want a cheaper fallback later, add `gemini-2.5-flash-lite`. Google’s pricing page shows a free tier for that model too. ([Google AI for Developers](https://ai.google.dev/gemini-api/docs/pricing "https://ai.google.dev/gemini-api/docs/pricing"))

## The data contract: make AI output one canonical shape

The most important design decision is this:

**Do not ask Gemini for UI coordinates.**  
Ask it for a **canonical plan object**, then let the app derive the flowchart layout.

That keeps the AI job simple and the UI deterministic.

Use one output shape like this:

```ts
export type PlanOutput = {
  title: string
  summary: string
  preferredView: 'flowchart' | 'steps' | 'checklist'
  steps: Array<{
    id: string
    title: string
    description: string
    kind: 'start' | 'action' | 'decision' | 'end'
    dependsOn: string[]
  }>
  edges: Array<{
    from: string
    to: string
    label?: string | null
  }>
  checklist: string[]
  assumptions: string[]
}
```

Why this shape works:

- **Flowchart** uses `steps + edges`
    
- **Steps view** uses `steps`
    
- **Checklist view** uses `checklist`
    
- **Summary card** uses `title + summary`
    
- **Assumptions** explains ambiguity
    

Gemini’s structured output support is exactly for this kind of typed extraction: you provide a JSON Schema and get predictable JSON back. The docs also note that the JS SDK works nicely with Zod. ([Google AI for Developers](https://ai.google.dev/gemini-api/docs/structured-output "https://ai.google.dev/gemini-api/docs/structured-output"))

## Complete technical flow

Here’s the full request lifecycle.

### 1. App launch

- initialize Supabase client
    
- check for an existing session
    
- if no session exists, call `supabase.auth.signInAnonymously()`
    

Anonymous sign-ins are a good match for full-feature demos that should work without asking for email/password. Anonymous users still use the `authenticated` database role, so you can protect rows with standard RLS policies. ([Supabase](https://supabase.com/docs/guides/auth/auth-anonymous "https://supabase.com/docs/guides/auth/auth-anonymous"))

### 2. User enters text

Example:

> “I need to launch a small online clothing store in 30 days. I don’t know where to start. Give me a plan.”

The client keeps local form state:

- `inputText`
    
- `requestedFormat`
    
- `uiState = idle | submitting | success | error`
    

### 3. User taps Generate

Client invokes:

```ts
await supabase.functions.invoke('generate-plan', {
  body: {
    inputText,
    requestedFormat,
  },
})
```

Supabase’s JS client has a built-in `functions.invoke()` call for Edge Functions. ([Supabase](https://supabase.com/docs/reference/javascript/functions-invoke "https://supabase.com/docs/reference/javascript/functions-invoke"))

### 4. Edge Function validates request

The function:

- reads the auth header
    
- checks the user session
    
- validates `inputText`
    
- inserts a `pending` row into `plan_generations`
    

### 5. Edge Function calls Gemini

The function sends a `generateContent` request to Gemini using REST with:

- model: `gemini-2.5-flash`
    
- prompt text
    
- `responseMimeType: "application/json"`
    
- `responseJsonSchema: ...`
    

Gemini’s REST API supports `generateContent`, works in any environment that can make HTTP requests, and requires the `x-goog-api-key` header. Structured outputs use `responseMimeType` plus `responseJsonSchema`. ([Google AI for Developers](https://ai.google.dev/api "https://ai.google.dev/api"))

### 6. Function validates AI output

Even though structured outputs help, still validate the response on the server with Zod before saving it.

### 7. Function updates the row

Set:

- `status = 'completed'`
    
- `title`
    
- `summary`
    
- `output_json`
    
- `model = 'gemini-2.5-flash'`
    

If Gemini fails or schema parsing fails:

- set `status = 'failed'`
    
- store `error_message`
    

### 8. Client receives result

The client:

- switches to success state
    
- shows the title/summary
    
- renders result tabs
    

### 9. Client renders three views from the same output

- `FlowchartView`
    
- `StepListView`
    
- `ChecklistView`
    

The flowchart is **derived from the plan**, not generated as raw UI. That keeps rendering reliable.

## Component structure

Use this screen structure:

```text
PlannerScreen
  ├─ AuthBootstrap
  ├─ PromptCard
  │   ├─ TaskTextArea
  │   ├─ FormatPicker
  │   ├─ ExamplePromptRow
  │   └─ GenerateButton
  ├─ StatusBanner
  ├─ ResultCard
  │   ├─ ResultHeader
  │   ├─ ViewModeTabs
  │   ├─ FlowchartView
  │   ├─ StepListView
  │   └─ ChecklistView
  └─ RecentPlansStrip (optional)
```

### Responsibilities

`AuthBootstrap`

- signs the user in anonymously if needed
    

`PromptCard`

- owns input text and format choice
    

`GenerateButton`

- disabled when empty or loading
    

`StatusBanner`

- loading / error / success messaging
    

`ResultHeader`

- AI-generated title + summary
    

`FlowchartView`

- read-only SVG canvas
    
- converts `steps + edges` into simple positioned nodes
    

`StepListView`

- expandable ordered list
    

`ChecklistView`

- clean action list
    

`RecentPlansStrip`

- fetches last 5 saved plans from Supabase
    

## Preferred file structure

For this demo, keep one app repo with a `src` app folder and a `supabase` backend folder.

```text
planner-demo/
  App.tsx
  package.json
  .env
  src/
    lib/
      supabase.ts
      database.types.ts
      env.ts
    shared/
      planSchema.ts
      prompts.ts
    theme/
      colors.ts
      spacing.ts
      typography.ts
    features/
      planner/
        components/
          PromptCard.tsx
          TaskTextArea.tsx
          FormatPicker.tsx
          GenerateButton.tsx
          ResultCard.tsx
          ResultHeader.tsx
          ViewModeTabs.tsx
          FlowchartView.tsx
          StepListView.tsx
          ChecklistView.tsx
          StatusBanner.tsx
          RecentPlansStrip.tsx
        hooks/
          useAnonymousSession.ts
          useGeneratePlan.ts
          usePlanHistory.ts
        services/
          plannerApi.ts
        utils/
          layoutGraph.ts
          topologicalSort.ts
          normalizePlan.ts
        types/
          planner.ts
    screens/
      PlannerScreen.tsx
  supabase/
    config.toml
    migrations/
      20260320_init_planner_demo.sql
    functions/
      generate-plan/
        index.ts
      .env
```

## Flowchart rendering strategy

Do **not** use a heavy node editor for this demo.  
Use **`react-native-svg`** and keep the graph read-only.

Expo’s docs show `react-native-svg` is supported in Expo and installed with `npx expo install react-native-svg`. It gives you SVG primitives like `Rect`, `Path`, `Line`, and `Text`, which is enough for a clean read-only flowchart. ([Expo Documentation](https://docs.expo.dev/versions/latest/sdk/svg/ "https://docs.expo.dev/versions/latest/sdk/svg/"))

### Layout rule

Use a simple deterministic layout:

- y-position = step depth
    
- x-position = branch lane
    
- decision nodes wider than action nodes
    
- edges drawn as straight or curved SVG paths
    

That is much easier than asking the AI for pixel-perfect positions.

## Database design for the demo

For the demo, **one table is enough**:

```sql
create extension if not exists pgcrypto;

create table public.plan_generations (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  input_text text not null check (char_length(trim(input_text)) > 0 and char_length(input_text) <= 5000),
  requested_format text not null default 'auto'
    check (requested_format in ('auto', 'flowchart', 'steps', 'checklist')),
  status text not null default 'pending'
    check (status in ('pending', 'completed', 'failed')),
  title text,
  summary text,
  output_json jsonb,
  model text not null default 'gemini-2.5-flash',
  error_message text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create or replace function public.set_updated_at()
returns trigger
language plpgsql
as $$
begin
  new.updated_at = now();
  return new;
end;
$$;

create trigger trg_plan_generations_updated_at
before update on public.plan_generations
for each row
execute function public.set_updated_at();

alter table public.plan_generations enable row level security;

create policy "users can read own plans"
on public.plan_generations
for select
to authenticated
using ((select auth.uid()) = user_id);

create policy "users can insert own plans"
on public.plan_generations
for insert
to authenticated
with check ((select auth.uid()) = user_id);

create policy "users can update own plans"
on public.plan_generations
for update
to authenticated
using ((select auth.uid()) = user_id)
with check ((select auth.uid()) = user_id);

create policy "users can delete own plans"
on public.plan_generations
for delete
to authenticated
using ((select auth.uid()) = user_id);
```

This works well because Supabase exposes tables in the `public` schema through the Data API, and those tables should have RLS enabled. ([Supabase](https://supabase.com/docs/guides/api "https://supabase.com/docs/guides/api"))

## Gemini prompt design

Use a fixed system-style prompt inside the Edge Function:

```ts
const prompt = `
You are a task planning engine.

Turn the user's text into a practical execution plan.

Rules:
- Output only valid JSON matching the provided schema.
- Keep the plan realistic and concise.
- Prefer 4 to 12 steps.
- Use "decision" nodes only when branching is necessary.
- Include assumptions when the request is ambiguous.
- preferredView should be:
  - "flowchart" when branching or decision points exist
  - "steps" for linear plans
  - "checklist" for very simple action lists

User request:
${inputText}

Requested display preference:
${requestedFormat}
`
```

## Edge Function shape

Because Google’s JS quickstart targets Node.js 18+, while the Gemini REST API works in any HTTP environment, I’d use plain `fetch` inside the Deno-based Supabase Edge Function instead of adding the JS SDK there. ([Google AI for Developers](https://ai.google.dev/gemini-api/docs/quickstart "https://ai.google.dev/gemini-api/docs/quickstart"))

A good `generate-plan` function looks like this:

```ts
// supabase/functions/generate-plan/index.ts
import { createClient } from 'npm:@supabase/supabase-js@2'
import { z } from 'npm:zod@3.23.8'

const PlanSchema = z.object({
  title: z.string(),
  summary: z.string(),
  preferredView: z.enum(['flowchart', 'steps', 'checklist']),
  steps: z.array(
    z.object({
      id: z.string(),
      title: z.string(),
      description: z.string(),
      kind: z.enum(['start', 'action', 'decision', 'end']),
      dependsOn: z.array(z.string()),
    })
  ),
  edges: z.array(
    z.object({
      from: z.string(),
      to: z.string(),
      label: z.string().nullable().optional(),
    })
  ),
  checklist: z.array(z.string()),
  assumptions: z.array(z.string()),
})

const corsHeaders = {
  'Content-Type': 'application/json',
}

Deno.serve(async (req) => {
  try {
    const authHeader = req.headers.get('Authorization')
    if (!authHeader) {
      return new Response(JSON.stringify({ error: 'Missing auth header' }), {
        status: 401,
        headers: corsHeaders,
      })
    }

    const supabase = createClient(
      Deno.env.get('SUPABASE_URL')!,
      Deno.env.get('SUPABASE_ANON_KEY')!,
      { global: { headers: { Authorization: authHeader } } }
    )

    const { data: authData } = await supabase.auth.getUser()
    const user = authData.user
    if (!user) {
      return new Response(JSON.stringify({ error: 'Unauthorized' }), {
        status: 401,
        headers: corsHeaders,
      })
    }

    const { inputText, requestedFormat } = await req.json()

    if (!inputText || typeof inputText !== 'string' || inputText.trim().length < 10) {
      return new Response(JSON.stringify({ error: 'Please enter a longer task description.' }), {
        status: 400,
        headers: corsHeaders,
      })
    }

    const { data: row, error: insertError } = await supabase
      .from('plan_generations')
      .insert({
        user_id: user.id,
        input_text: inputText,
        requested_format: requestedFormat ?? 'auto',
        status: 'pending',
      })
      .select()
      .single()

    if (insertError) throw insertError

    const responseJsonSchema = {
      type: 'object',
      required: ['title', 'summary', 'preferredView', 'steps', 'edges', 'checklist', 'assumptions'],
      properties: {
        title: { type: 'string' },
        summary: { type: 'string' },
        preferredView: { type: 'string', enum: ['flowchart', 'steps', 'checklist'] },
        steps: {
          type: 'array',
          items: {
            type: 'object',
            required: ['id', 'title', 'description', 'kind', 'dependsOn'],
            properties: {
              id: { type: 'string' },
              title: { type: 'string' },
              description: { type: 'string' },
              kind: { type: 'string', enum: ['start', 'action', 'decision', 'end'] },
              dependsOn: {
                type: 'array',
                items: { type: 'string' },
              },
            },
          },
        },
        edges: {
          type: 'array',
          items: {
            type: 'object',
            required: ['from', 'to'],
            properties: {
              from: { type: 'string' },
              to: { type: 'string' },
              label: { type: ['string', 'null'] },
            },
          },
        },
        checklist: {
          type: 'array',
          items: { type: 'string' },
        },
        assumptions: {
          type: 'array',
          items: { type: 'string' },
        },
      },
    }

    const prompt = `
Turn the user's request into a clear execution plan.
Return only valid JSON matching the schema.
Use flowchart view only when branching exists.
Keep plans practical and concise.

Requested format: ${requestedFormat ?? 'auto'}

User request:
${inputText}
`

    const geminiRes = await fetch(
      'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent',
      {
        method: 'POST',
        headers: {
          'x-goog-api-key': Deno.env.get('GEMINI_API_KEY')!,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          contents: [{ role: 'user', parts: [{ text: prompt }] }],
          generationConfig: {
            responseMimeType: 'application/json',
            responseJsonSchema,
          },
        }),
      }
    )

    if (!geminiRes.ok) {
      const errorText = await geminiRes.text()

      await supabase
        .from('plan_generations')
        .update({
          status: 'failed',
          error_message: errorText,
        })
        .eq('id', row.id)

      return new Response(JSON.stringify({ error: 'Gemini request failed' }), {
        status: 502,
        headers: corsHeaders,
      })
    }

    const geminiJson = await geminiRes.json()
    const rawText = geminiJson?.candidates?.[0]?.content?.parts?.[0]?.text ?? '{}'
    const parsed = PlanSchema.parse(JSON.parse(rawText))

    await supabase
      .from('plan_generations')
      .update({
        status: 'completed',
        title: parsed.title,
        summary: parsed.summary,
        output_json: parsed,
        model: 'gemini-2.5-flash',
      })
      .eq('id', row.id)

    return new Response(JSON.stringify({ id: row.id, plan: parsed }), {
      status: 200,
      headers: corsHeaders,
    })
  } catch (error) {
    return new Response(
      JSON.stringify({
        error: error instanceof Error ? error.message : 'Unknown error',
      }),
      { status: 500, headers: corsHeaders }
    )
  }
})
```

## Client-side Supabase setup

Supabase’s React Native docs use `createClient` with persisted session storage and session auto-refresh handling. Use that pattern. ([Supabase](https://supabase.com/docs/guides/auth/quickstarts/react-native "https://supabase.com/docs/guides/auth/quickstarts/react-native"))

A minimal client file:

```ts
// src/lib/supabase.ts
import { AppState, Platform } from 'react-native'
import AsyncStorage from '@react-native-async-storage/async-storage'
import 'react-native-url-polyfill/auto'
import { createClient, processLock } from '@supabase/supabase-js'
import type { Database } from './database.types'

export const supabase = createClient<Database>(
  process.env.EXPO_PUBLIC_SUPABASE_URL!,
  process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY!,
  {
    auth: {
      ...(Platform.OS !== 'web' ? { storage: AsyncStorage } : {}),
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: false,
      lock: processLock,
    },
  }
)

if (Platform.OS !== 'web') {
  AppState.addEventListener('change', (state) => {
    if (state === 'active') supabase.auth.startAutoRefresh()
    else supabase.auth.stopAutoRefresh()
  })
}
```

Anonymous bootstrap:

```ts
// src/features/planner/hooks/useAnonymousSession.ts
import { useEffect } from 'react'
import { supabase } from '../../../lib/supabase'

export function useAnonymousSession() {
  useEffect(() => {
    let cancelled = false

    async function ensureSession() {
      const { data } = await supabase.auth.getSession()
      if (cancelled || data.session) return
      await supabase.auth.signInAnonymously()
    }

    ensureSession()
    return () => {
      cancelled = true
    }
  }, [])
}
```

## Client-side generation hook

```ts
// src/features/planner/hooks/useGeneratePlan.ts
import { useState } from 'react'
import { supabase } from '../../../lib/supabase'

export function useGeneratePlan() {
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)
  const [plan, setPlan] = useState<any>(null)

  async function generate(inputText: string, requestedFormat: string) {
    setLoading(true)
    setError(null)

    const { data, error } = await supabase.functions.invoke('generate-plan', {
      body: { inputText, requestedFormat },
    })

    if (error) {
      setLoading(false)
      setError(error.message)
      return
    }

    setPlan(data?.plan ?? null)
    setLoading(false)
  }

  return { loading, error, plan, generate }
}
```

## Supabase setup manual

This is the cleanest hosted setup.

### 1. Create the app

Expo currently documents `create-expo-app@latest --template default@sdk-55` for a fresh project, while Supabase’s Expo quickstart also documents creating a TypeScript Expo app and then installing `@supabase/supabase-js` plus the required React Native dependencies. ([Expo Documentation](https://docs.expo.dev/get-started/create-a-project/ "https://docs.expo.dev/get-started/create-a-project/"))

```bash
npx create-expo-app@latest planner-demo --template default@sdk-55
cd planner-demo

npx expo install @supabase/supabase-js @react-native-async-storage/async-storage react-native-url-polyfill react-native-svg
npm install zod zod-to-json-schema
npm i supabase@">=1.8.1" --save-dev
```

### 2. Create the Supabase project

Create a project in the Supabase dashboard, then grab the project URL and anon key from the dashboard / connect dialog. Supabase’s React Native quickstart points you to those exact values for the client setup. ([Supabase](https://supabase.com/docs/guides/getting-started/quickstarts/expo-react-native "https://supabase.com/docs/guides/getting-started/quickstarts/expo-react-native"))

Add this to `.env`:

```bash
EXPO_PUBLIC_SUPABASE_URL=your-project-url
EXPO_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

### 3. Log in and initialize the CLI

Supabase’s CLI docs show `supabase init`, `supabase login`, and `supabase link` as the normal flow for local development, migrations, and function deployment. ([Supabase](https://supabase.com/docs/guides/functions/quickstart "https://supabase.com/docs/guides/functions/quickstart"))

```bash
npx supabase login
npx supabase init
npx supabase link --project-ref YOUR_PROJECT_REF
```

### 4. Enable anonymous sign-ins

Turn on anonymous auth in your Supabase project. Supabase documents anonymous users as a good fit for full-feature demos without collecting personal info, and the user still gets the `authenticated` role for RLS-protected access. ([Supabase](https://supabase.com/docs/guides/auth/auth-anonymous "https://supabase.com/docs/guides/auth/auth-anonymous"))

Then in the app, call:

```ts
await supabase.auth.signInAnonymously()
```

If you expose the demo publicly, note that Supabase recommends CAPTCHA / Turnstile for anonymous sign-ins to reduce abuse, and it documents an IP-based default limit of 30 requests per hour for anonymous sign-ins. ([Supabase](https://supabase.com/docs/guides/auth/auth-anonymous "https://supabase.com/docs/guides/auth/auth-anonymous"))

### 5. Create the migration

Supabase documents schema changes through SQL migrations and uses `supabase migration new` plus `supabase db push` to apply them. ([Supabase](https://supabase.com/docs/guides/deployment/database-migrations "https://supabase.com/docs/guides/deployment/database-migrations"))

```bash
npx supabase migration new init_planner_demo
```

Paste the SQL schema from the previous section into the new migration file, then push:

```bash
npx supabase db push
```

### 6. Create the Edge Function

Supabase’s function quickstart uses:

```bash
npx supabase functions new generate-plan
```

and then deploys with `supabase functions deploy`. ([Supabase](https://supabase.com/docs/guides/functions/quickstart "https://supabase.com/docs/guides/functions/quickstart"))

### 7. Add the Gemini API key

Google’s quickstart says you can create a Gemini API key for free in AI Studio. Supabase secrets docs say production function secrets should be stored via dashboard secrets or `supabase secrets set`. ([Google AI for Developers](https://ai.google.dev/gemini-api/docs/quickstart "https://ai.google.dev/gemini-api/docs/quickstart"))

Create `supabase/functions/.env` locally:

```bash
GEMINI_API_KEY=your-gemini-api-key
```

Local function testing:

```bash
npx supabase start
npx supabase functions serve generate-plan --env-file supabase/functions/.env
```

Deploy the secret to hosted Supabase:

```bash
npx supabase secrets set --env-file supabase/functions/.env
```

Deploy the function:

```bash
npx supabase functions deploy generate-plan
```

Supabase notes that after setting secrets, you do not need to redeploy just to make the secrets available. ([Supabase](https://supabase.com/docs/guides/functions/quickstart "https://supabase.com/docs/guides/functions/quickstart"))

### 8. Generate DB types

Supabase can generate TypeScript types directly from your schema, which is worth doing even for a demo. ([Supabase](https://supabase.com/docs/guides/api/rest/generating-types "https://supabase.com/docs/guides/api/rest/generating-types"))

```bash
npx supabase gen types typescript --project-id "$PROJECT_REF" --schema public > src/lib/database.types.ts
```

## Suggested one-screen layout

```text
[ Header ]
Task Planner

[ Prompt Card ]
Describe the task...
[ multiline input area ]

[ Auto ] [ Flowchart ] [ Steps ] [ Checklist ]

[ Generate Plan ]

[ Status Banner ]
Turning this into a plan...

[ Result Card ]
AI Title
Short summary

[ Flowchart | Steps | Checklist ]

...rendered result...
```

## Recommended rendering behavior

### Flowchart tab

- show nodes as rounded cards
    
- arrows between nodes
    
- decision nodes visually distinct
    
- scroll vertically
    
- zoom is optional; do not build drag-and-drop yet
    

### Steps tab

- numbered steps
    
- each step card shows title + description
    
- dependency badges like “after Step 2”
    

### Checklist tab

- plain clean checkboxes
    
- copy/share friendly
    
- derived from the same plan JSON
    

## Important implementation notes

Keep these rules:

- **Canonical source = `output_json`**
    
- **Flowchart is derived**
    
- **One request at a time**
    
- **Disable Generate while loading**
    
- **Cap input length** around 3,000–5,000 characters for predictable UX on the free tier
    
- **Save failed runs too** so you can debug model failures
    
- **Validate AI JSON on the server**
    
- **Never put the Gemini key in the app**
    

Gemini rate limits are applied per project, not per API key, and rate-limit ceilings can change by tier; Google says to view your active limits in AI Studio. That’s another reason to keep the demo conservative and serial rather than allowing spammy parallel requests. ([Google AI for Developers](https://ai.google.dev/gemini-api/docs/rate-limits "https://ai.google.dev/gemini-api/docs/rate-limits"))

## What I would ship as v1

If I were implementing this right now, the exact v1 would be:

- anonymous sign-in on launch
    
- one multiline input
    
- format toggle
    
- one `generate-plan` Edge Function
    
- one `plan_generations` table
    
- one flowchart renderer using `react-native-svg`
    
- steps + checklist tabs
    
- save each generation to Supabase
    

That gets you a real, demoable product without extra backend complexity.

The next sensible build order is:

1. Supabase project + schema
    
2. anonymous auth
    
3. `generate-plan` Edge Function
    
4. client submit flow
    
5. steps/checklist rendering
    
6. flowchart rendering last
    

If you want the strongest next artifact, the right one is the actual migration SQL plus the full `generate-plan` Edge Function and the `PlannerScreen.tsx` scaffold.