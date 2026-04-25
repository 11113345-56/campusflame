# CampusFlame — Complete Setup Guide
## Two-Site Architecture + Supabase + Daily.co Video Calls

---

## 🗂️ WHAT YOU HAVE

You received two zip files:
- **campusflame-user.zip** → The main dating app (users sign up and log in here)
- **campusflame-admin.zip** → The admin control panel (only you access this)

Both sites connect to the **same Supabase project** — that's how the admin site can see and control everything the users do.

---

## STEP 1 — SUPABASE: CREATE YOUR DATABASE TABLES

1. Go to https://supabase.com → open your project
2. Click **SQL Editor** in the left sidebar
3. Copy and paste ALL of this SQL, then click **Run**:

```sql
-- PROFILES TABLE (main user data)
create table if not exists profiles (
  id uuid references auth.users on delete cascade primary key,
  name text,
  phone text unique,
  age integer,
  gender text,
  university text,
  location text,
  role text default 'user',
  avatar_url text,
  bio text,
  interests text,
  looking_for text,
  relationship_type text,
  profile_complete boolean default false,
  premium_until timestamptz,
  status text default 'active',
  created_at timestamptz default now()
);

-- MESSAGES TABLE (chat between users)
create table if not exists messages (
  id uuid default gen_random_uuid() primary key,
  sender_id uuid references profiles(id) on delete cascade,
  receiver_id uuid references profiles(id) on delete cascade,
  content text,
  type text default 'text',
  created_at timestamptz default now()
);

-- FLAGGED CONTENT TABLE (auto-moderation)
create table if not exists flagged_content (
  id uuid default gen_random_uuid() primary key,
  message_id uuid,
  reason text,
  flagged_at timestamptz default now(),
  reviewed boolean default false
);

-- LOVE DOCTOR MESSAGES TABLE
create table if not exists love_doctor_messages (
  id uuid default gen_random_uuid() primary key,
  user_id uuid references profiles(id) on delete cascade,
  user_name text,
  message text,
  sender text,
  is_priority boolean default false,
  read_by_admin boolean default false,
  created_at timestamptz default now()
);
```

---

## STEP 2 — SUPABASE: SET ROW LEVEL SECURITY (RLS) POLICIES

Still in the SQL Editor, paste and run this:

```sql
-- Enable RLS on all tables
alter table profiles enable row level security;
alter table messages enable row level security;
alter table flagged_content enable row level security;
alter table love_doctor_messages enable row level security;

-- PROFILES policies
create policy "Users can read all profiles"
  on profiles for select using (true);

create policy "Users can update own profile"
  on profiles for update using (auth.uid() = id);

create policy "Users can insert own profile"
  on profiles for insert with check (auth.uid() = id);

-- MESSAGES policies
create policy "Users can read their own messages"
  on messages for select
  using (auth.uid() = sender_id or auth.uid() = receiver_id);

create policy "Users can send messages"
  on messages for insert with check (auth.uid() = sender_id);

create policy "Users can delete own messages"
  on messages for delete using (auth.uid() = sender_id);

-- FLAGGED CONTENT (admin needs full access via service role, users can insert)
create policy "Anyone can flag content"
  on flagged_content for insert with check (true);

create policy "Admins can view flagged content"
  on flagged_content for select
  using (
    exists (select 1 from profiles where id = auth.uid() and role = 'admin')
  );

create policy "Admins can update flagged content"
  on flagged_content for update
  using (
    exists (select 1 from profiles where id = auth.uid() and role = 'admin')
  );

-- LOVE DOCTOR policies
create policy "Users can insert love doctor messages"
  on love_doctor_messages for insert with check (true);

create policy "Users can read own love doctor messages"
  on love_doctor_messages for select
  using (
    auth.uid() = user_id
    or exists (select 1 from profiles where id = auth.uid() and role = 'admin')
  );

create policy "Admins can update love doctor messages"
  on love_doctor_messages for update
  using (
    exists (select 1 from profiles where id = auth.uid() and role = 'admin')
  );
```

---

## STEP 3 — SUPABASE: SET UP STORAGE (for profile photos)

1. Go to **Storage** in the Supabase sidebar
2. Click **New Bucket** → name it: `avatars` → make it **Public** → Save
3. Go to **Policies** tab inside the avatars bucket and add:

```sql
-- Allow anyone to read avatars
create policy "Public avatars"
  on storage.objects for select using (bucket_id = 'avatars');

-- Allow authenticated users to upload their own avatar
create policy "Users upload own avatar"
  on storage.objects for insert
  with check (bucket_id = 'avatars' and auth.role() = 'authenticated');

-- Allow users to update their own avatar
create policy "Users update own avatar"
  on storage.objects for update
  using (bucket_id = 'avatars' and auth.uid()::text = (storage.foldername(name))[2]);
```

---

## STEP 4 — CREATE YOUR ADMIN ACCOUNT

⚠️ IMPORTANT: Do this BEFORE you share the user site with anyone.

1. Open the admin site on your computer
2. Go to: `campusflame-admin/pages/setup.html`
3. Open it in your browser (just double-click the file or use a local server)
4. Fill in your name, phone number, PIN
5. For **Admin Secret Key**, type: `campusflame-admin-2025`
   (You can change this secret inside setup.html before using it — look for `ADMIN_SECRET`)
6. Click **Create Admin Account**
7. It will redirect you to login.html — log in with your phone + PIN
8. **After logging in successfully**, DELETE or rename `setup.html` so no one else can use it

---

## STEP 5 — FIX THE SUPABASE KEYS (if you use a different Supabase project)

Both config.js files already have your project's keys. If you ever create a new Supabase project:

**In `campusflame-user/js/config.js`:**
```js
const SUPABASE_URL = 'YOUR_NEW_URL';
const SUPABASE_ANON_KEY = 'YOUR_NEW_ANON_KEY';
```

**In `campusflame-admin/js/config.js`:**
```js
const SUPABASE_URL = 'YOUR_NEW_URL';
const SUPABASE_ANON_KEY = 'YOUR_NEW_ANON_KEY';
```

Find these values in: Supabase Dashboard → Settings → API

---

## STEP 6 — DEPLOYING THE TWO SITES

Each site needs to be hosted separately (they're just static HTML files).

### Option A: Netlify (Free, Recommended)

**User site:**
1. Go to https://netlify.com → Sign up free
2. Drag and drop the `campusflame-user` folder onto the Netlify dashboard
3. It gives you a URL like: `campusflame-user.netlify.app`
4. You can add a custom domain like `campusflame.ug`

**Admin site:**
1. Do the same for `campusflame-admin` folder
2. You get a different URL like: `cf-admin-2025.netlify.app`
3. Keep this URL private — only share with yourself

### Option B: GitHub Pages (Free)

1. Create two GitHub repos (one private for admin, one public for user site)
2. Upload the files to each repo
3. Go to Settings → Pages → Deploy from main branch

### Option C: cPanel Hosting (if you have a web host)

1. Open your cPanel File Manager
2. Upload `campusflame-user` contents to `public_html/`
3. Upload `campusflame-admin` contents to `public_html/admin/` or a subdomain

---

## HOW DOES THE ADMIN SITE SEE USER DATA?

Both sites connect to the same Supabase database using the same URL and API key. Think of it like two shops connected to the same warehouse — the user shop lets customers buy, the admin shop lets you manage inventory.

When a user signs up on the user site → their profile is saved in Supabase. When you open the admin site → it reads those same profiles from Supabase. When you ban a user in admin → it updates their `status` to `banned` in Supabase, and the user site checks that status on login and blocks them.

---

## STEP 7 — VIDEO CALLS WITH DAILY.CO

Daily.co gives you free video calls up to 1,000 minutes/month.

### Setup:

1. Go to https://www.daily.co → Sign up free
2. Go to **Dashboard → Developers → API Keys** → copy your API key
3. In the admin dashboard, go to your Netlify/hosting **Environment Variables** and add:
   ```
   DAILY_API_KEY = your_daily_api_key_here
   ```

### How to add video calls to the user site:

In `campusflame-user/pages/chats.html` (inside the chat window), add a video call button. When pressed it:
1. Calls Daily's API to create a temporary room
2. Opens that room URL in a new window (or an iframe)

Here's the code to add inside chats.html where you want the video call button:

```html
<button onclick="startVideoCall(peerId)" id="vid-btn" style="...">📹 Video Call</button>

<script>
async function startVideoCall(peerId) {
  // Create a Daily.co room via your backend or Supabase Edge Function
  // Room name: use a combo of both user IDs so they get the same room
  const roomName = [myProfile.id, peerId].sort().join('-').slice(0, 40);
  
  // Option 1 (simple): Open Daily prebuilt UI
  const roomUrl = `https://your-domain.daily.co/${roomName}`;
  window.open(roomUrl, '_blank', 'width=900,height=700');
  
  // Option 2 (custom): Use Daily.co JS SDK
  // Add to your HTML: <script src="https://unpkg.com/@daily-co/daily-js"></script>
  // Then:
  // const callFrame = DailyIframe.createFrame();
  // callFrame.join({ url: roomUrl });
}
</script>
```

### Creating rooms via Daily API (Supabase Edge Function):

1. Go to Supabase → **Edge Functions** → New Function → name it `create-video-room`
2. Paste this code:

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';

serve(async (req) => {
  const { roomName } = await req.json();
  const response = await fetch('https://api.daily.co/v1/rooms', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${Deno.env.get('DAILY_API_KEY')}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      name: roomName,
      properties: { exp: Math.floor(Date.now() / 1000) + 3600 } // expires in 1hr
    })
  });
  const data = await response.json();
  return new Response(JSON.stringify({ url: data.url }), {
    headers: { 'Content-Type': 'application/json' }
  });
});
```

3. Deploy it: `supabase functions deploy create-video-room`
4. Call it from your app:

```js
const { data } = await sb.functions.invoke('create-video-room', {
  body: { roomName: [userId1, userId2].sort().join('-').slice(0,40) }
});
window.open(data.url, '_blank');
```

---

## TROUBLESHOOTING

**"Account not found" on login:**
The user signed up but their profile wasn't saved in the `profiles` table. Go to Supabase → Table Editor → profiles and check if they are there. If not, check that the signup code ran without errors.

**Admin login says "Access denied":**
The profile with your phone number doesn't have `role = 'admin'`. Go to Supabase → Table Editor → profiles → find your record → change `role` from `user` to `admin` manually.

**Images not uploading:**
Make sure you created the `avatars` storage bucket and set it to **Public**.

**RLS blocking everything:**
If you see "permission denied" errors, make sure you ran all the RLS policies in Step 2.

**Users can't see each other:**
Make sure `profile_complete = true` for those users. Only complete profiles appear in the discover page.

---

## PESAPAL PAYMENT INTEGRATION

Your Pesapal keys are already saved in config.js:
- Consumer Key: `dXDHd51IwbQOjDtnQ5xCbEoFH6HjquL8`
- Consumer Secret: `E5OsD+bWTYxbxpacxE7rMUM13gU=`

To activate real payments:
1. Go to https://pay.pesapal.com → Developer Portal → Register your website URL
2. Pesapal uses OAuth 1.0 — you'll need a backend (Supabase Edge Function) to generate the payment URL server-side (because the secret key must not be in client-side JS)
3. Ask the next step of this when ready — the payment flow needs its own guide

---

*CampusFlame — Where Campus Love Ignites 🔥*
