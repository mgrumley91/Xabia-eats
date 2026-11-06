The `gemini` Supabase Edge Function (deployed with verify_jwt off) reads bill photos for the app.
The Gemini key lives in the `public.secrets` table (RLS on, no policies, so phones can't read it);
set or replace it from the app's Settings → Reading bill photos.
