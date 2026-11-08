#!/usr/bin/env bash
set -euo pipefail
mkdir -p app/api/ai app/api/github app/\(app\)/dashboard app/\(app\)/chat app/\(app\)/repos app/\(app\)/ai app/\(auth\)/login app/\(auth\)/register app/auth/callback components lib/supabase supabase

# ---------- package.json ----------
cat > package.json <<'EOF'
{
  "name": "workhub",
  "version": "1.0.0",
  "private": true,
  "scripts": { "dev": "next dev", "build": "next build", "start": "next start" },
  "dependencies": {
    "@supabase/ssr": "^0.5.2",
    "@supabase/supabase-js": "^2.45.4",
    "lucide-react": "^0.453.0",
    "next": "14.2.15",
    "react": "18.3.1",
    "react-dom": "18.3.1"
  },
  "devDependencies": {
    "@types/node": "^20.16.11",
    "@types/react": "^18.3.11",
    "@types/react-dom": "^18.3.0",
    "autoprefixer": "^10.4.20",
    "postcss": "^8.4.47",
    "tailwindcss": "^3.4.13",
    "typescript": "^5.6.3"
  }
}
EOF

# ---------- konfiguracja ----------
cat > tsconfig.json <<'EOF'
{
  "compilerOptions": {
    "target": "ES2022", "lib": ["dom","dom.iterable","esnext"],
    "allowJs": true, "skipLibCheck": true, "strict": true, "noEmit": true,
    "esModuleInterop": true, "module": "esnext", "moduleResolution": "bundler",
    "resolveJsonModule": true, "isolatedModules": true, "jsx": "preserve",
    "incremental": true, "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts","**/*.ts","**/*.tsx",".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
EOF

cat > next.config.mjs <<'EOF'
/** @type {import('next').NextConfig} */
export default { reactStrictMode: true }
EOF

cat > postcss.config.mjs <<'EOF'
export default { plugins: { tailwindcss: {}, autoprefixer: {} } }
EOF

cat > tailwind.config.ts <<'EOF'
import type { Config } from 'tailwindcss'
export default {
  content: ['./app/**/*.{ts,tsx}', './components/**/*.{ts,tsx}'],
  theme: { extend: {} },
  plugins: [],
} satisfies Config
EOF

cat > .env.example <<'EOF'
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
# GitHub (opcjonalne – panel pokaże stan konfiguracji)
GITHUB_TOKEN=
GITHUB_REPO=owner/repo
# AI (opcjonalne – OpenAI-compatible)
OPENAI_API_KEY=
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_MODEL=gpt-4o-mini
EOF

# ---------- supabase schema ----------
cat > supabase/schema.sql <<'EOF'
create extension if not exists "pgcrypto";

create table if not exists public.tasks (
  id uuid primary key default gen_random_uuid(),
  title text not null check (char_length(title) between 1 and 500),
  done boolean not null default false,
  author_id uuid not null references auth.users(id) on delete cascade,
  author_email text not null,
  created_at timestamptz not null default now()
);

create table if not exists public.messages (
  id uuid primary key default gen_random_uuid(),
  body text not null check (char_length(body) between 1 and 4000),
  author_id uuid not null references auth.users(id) on delete cascade,
  author_email text not null,
  created_at timestamptz not null default now()
);

alter table public.tasks enable row level security;
alter table public.messages enable row level security;

drop policy if exists tasks_read on public.tasks;
drop policy if exists tasks_insert on public.tasks;
drop policy if exists tasks_update on public.tasks;
drop policy if exists tasks_delete on public.tasks;
create policy tasks_read on public.tasks for select to authenticated using (true);
create policy tasks_insert on public.tasks for insert to authenticated with check (auth.uid() = author_id);
create policy tasks_update on public.tasks for update to authenticated using (true);
create policy tasks_delete on public.tasks for delete to authenticated using (true);

drop policy if exists messages_read on public.messages;
drop policy if exists messages_insert on public.messages;
create policy messages_read on public.messages for select to authenticated using (true);
create policy messages_insert on public.messages for insert to authenticated with check (auth.uid() = author_id);

alter publication supabase_realtime add table public.tasks;
alter publication supabase_realtime add table public.messages;
EOF

# ---------- lib/supabase ----------
cat > lib/supabase/client.ts <<'EOF'
import { createBrowserClient } from '@supabase/ssr'
let _c: ReturnType<typeof createBrowserClient> | null = null
export function getBrowserClient() {
  if (!_c) _c = createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
  return _c
}
EOF

cat > lib/supabase/server.ts <<'EOF'
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
export async function getServerClient() {
  const store = await cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => store.getAll(),
        setAll: (list) => list.forEach((c) => store.set(c.name, c.value, c.options)),
      },
    }
  )
}
EOF

# ---------- middleware ----------
cat > middleware.ts <<'EOF'
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(req: NextRequest) {
  let res = NextResponse.next({ request: req })
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => req.cookies.getAll(),
        setAll: (list) => {
          list.forEach(({ name, value }) => req.cookies.set(name, value))
          res = NextResponse.next({ request: req })
          list.forEach(({ name, value, options }) => res.cookies.set(name, value, options))
        },
      },
    }
  )
  const { data: { user } } = await supabase.auth.getUser()
  const path = req.nextUrl.pathname
  const isAuth = path.startsWith('/login') || path.startsWith('/register') || path.startsWith('/auth')
  if (!user && !isAuth) {
    const u = req.nextUrl.clone(); u.pathname = '/login'; u.searchParams.set('next', path)
    return NextResponse.redirect(u)
  }
  if (user && (path === '/login' || path === '/register')) {
    const u = req.nextUrl.clone(); u.pathname = '/dashboard'; u.searchParams.delete('next')
    return NextResponse.redirect(u)
  }
  return res
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp|ico)$).*)'],
}
EOF

# ---------- app/layout ----------
cat > app/globals.css <<'EOF'
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --bg: 8 12 20;
  --panel: 13 18 28;
  --border: 30 41 59;
  --text: 226 232 240;
  --muted: 148 163 184;
  --accent: 34 211 238;
  --accent-fg: 8 12 20;
}
html.light {
  --bg: 248 250 252;
  --panel: 255 255 255;
  --border: 226 232 240;
  --text: 15 23 42;
  --muted: 100 116 139;
  --accent: 8 145 178;
  --accent-fg: 255 255 255;
}
html, body { height: 100%; }
body {
  background-color: rgb(var(--bg));
  color: rgb(var(--text));
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, "Segoe UI", sans-serif;
  font-feature-settings: "cv02","cv03","cv04","cv11";
  -webkit-font-smoothing: antialiased;
  background-image:
    radial-gradient(1000px 500px at 90% -10%, rgba(34,211,238,0.10), transparent 60%),
    linear-gradient(rgba(148,163,184,0.05) 1px, transparent 1px),
    linear-gradient(90deg, rgba(148,163,184,0.05) 1px, transparent 1px);
  background-size: auto, 44px 44px, 44px 44px;
  background-attachment: fixed;
}
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: 0.001ms !important; transition-duration: 0.001ms !important; }
}
.panel {
  background: rgb(var(--panel) / 0.85);
  border: 1px solid rgb(var(--border));
  border-radius: 12px;
  backdrop-filter: blur(8px);
}
.panel-glow { box-shadow: 0 0 0 1px rgb(var(--accent) / 0.10), 0 0 32px -8px rgb(var(--accent) / 0.25); }
.btn {
  display: inline-flex; align-items: center; gap: 8px; justify-content: center;
  padding: 8px 14px; border-radius: 8px; font-size: 14px; font-weight: 500;
  border: 1px solid rgb(var(--border)); background: rgb(var(--panel));
  color: rgb(var(--text)); transition: all .15s ease;
}
.btn:hover { border-color: rgb(var(--accent) / 0.5); color: rgb(var(--accent)); }
.btn:focus-visible { outline: 2px solid rgb(var(--accent)); outline-offset: 2px; }
.btn-primary {
  background: rgb(var(--accent)); color: rgb(var(--accent-fg)); border-color: rgb(var(--accent));
}
.btn-primary:hover { filter: brightness(1.1); color: rgb(var(--accent-fg)); }
.btn:disabled { opacity: 0.5; cursor: not-allowed; }
.input {
  width: 100%; padding: 10px 12px; border-radius: 8px; font-size: 14px;
  background: rgb(var(--bg) / 0.6); border: 1px solid rgb(var(--border));
  color: rgb(var(--text)); transition: border-color .15s, box-shadow .15s;
}
.input:focus { outline: none; border-color: rgb(var(--accent)); box-shadow: 0 0 0 3px rgb(var(--accent) / 0.18); }
.muted { color: rgb(var(--muted)); }
.accent { color: rgb(var(--accent)); }
.divider { height: 1px; background: rgb(var(--border)); }
::-webkit-scrollbar { width: 8px; height: 8px; }
::-webkit-scrollbar-thumb { background: rgb(var(--border)); border-radius: 8px; }
EOF

cat > app/layout.tsx <<'EOF'
import './globals.css'
import type { Metadata } from 'next'
export const metadata: Metadata = { title: 'WorkHub — wspólna przestrzeń zespołu', description: 'Pulpit, zadania, czat, repozytoria i asystent AI w jednym miejscu.' }
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="pl" suppressHydrationWarning>
      <head>
        <script dangerouslySetInnerHTML={{ __html:
          `try{if(localStorage.theme==='light')document.documentElement.classList.add('light')}catch(e){}` }} />
      </head>
      <body>{children}</body>
    </html>
  )
}
EOF

cat > app/page.tsx <<'EOF'
import { redirect } from 'next/navigation'
export default function Home() { redirect('/dashboard') }
EOF

# ---------- komponenty ----------
cat > components/ThemeToggle.tsx <<'EOF'
'use client'
import { useEffect, useState } from 'react'
import { Moon, Sun } from 'lucide-react'
export default function ThemeToggle() {
  const [light, setLight] = useState(false)
  useEffect(() => { setLight(document.documentElement.classList.contains('light')) }, [])
  function toggle() {
    const next = !light
    setLight(next)
    document.documentElement.classList.toggle('light', next)
    try { localStorage.theme = next ? 'light' : 'dark' } catch {}
  }
  return (
    <button onClick={toggle} aria-label="Przełącz motyw" className="btn" title={light ? 'Tryb ciemny' : 'Tryb jasny'}>
      {light ? <Moon size={16} /> : <Sun size={16} />}
      <span className="hidden sm:inline">{light ? 'Ciemny' : 'Jasny'}</span>
    </button>
  )
}
EOF

cat > components/Sidebar.tsx <<'EOF'
'use client'
import Link from 'next/link'
import { usePathname, useRouter } from 'next/navigation'
import { LayoutDashboard, MessagesSquare, Github, Sparkles, LogOut } from 'lucide-react'
import { getBrowserClient } from '@/lib/supabase/client'
const NAV = [
  { href: '/dashboard', label: 'Pulpit', Icon: LayoutDashboard },
  { href: '/chat', label: 'Czat', Icon: MessagesSquare },
  { href: '/repos', label: 'Repozytoria', Icon: Github },
  { href: '/ai', label: 'AI', Icon: Sparkles },
]
export default function Sidebar({ email }: { email: string }) {
  const path = usePathname()
  const router = useRouter()
  async function logout() {
    await getBrowserClient().auth.signOut()
    router.push('/login'); router.refresh()
  }
  return (
    <aside className="w-full lg:w-60 shrink-0 lg:border-r lg:min-h-screen flex flex-col gap-1 p-4 lg:p-5" style={{ borderColor: 'rgb(var(--border))' }}>
      <div className="flex items-center justify-between lg:block mb-3">
        <div className="flex items-center gap-2">
          <div className="w-7 h-7 rounded-md grid place-items-center" style={{ background: 'rgb(var(--accent))', color: 'rgb(var(--accent-fg))' }}>
            <span className="text-xs font-bold">W</span>
          </div>
          <div>
            <div className="text-sm font-semibold tracking-tight">WorkHub</div>
            <div className="text-[10px] muted uppercase tracking-widest">Workspace</div>
          </div>
        </div>
      </div>
      <nav className="flex lg:flex-col gap-1 overflow-x-auto lg:overflow-visible">
        {NAV.map(({ href, label, Icon }) => {
          const active = path.startsWith(href)
          return (
            <Link key={href} href={href}
              className="btn justify-start shrink-0"
              style={active ? { borderColor: 'rgb(var(--accent))', color: 'rgb(var(--accent))', background: 'rgb(var(--accent) / 0.08)' } : {}}>
              <Icon size={16} /><span>{label}</span>
            </Link>
          )
        })}
      </nav>
      <div className="hidden lg:block mt-auto pt-4 border-t" style={{ borderColor: 'rgb(var(--border))' }}>
        <div className="text-xs muted mb-2 truncate" title={email}>{email}</div>
        <button onClick={logout} className="btn w-full justify-start"><LogOut size={16} /> Wyloguj</button>
      </div>
    </aside>
  )
}
EOF

cat > components/PresenceBar.tsx <<'EOF'
'use client'
import { useEffect, useState } from 'react'
import { getBrowserClient } from '@/lib/supabase/client'
export default function PresenceBar({ userId, email }: { userId: string; email: string }) {
  const [online, setOnline] = useState<{ id: string; email: string }[]>([])
  useEffect(() => {
    const sb = getBrowserClient()
    const ch = sb.channel('presence-room', { config: { presence: { key: userId } } })
    ch.on('presence', { event: 'sync' }, () => {
      const state = ch.presenceState() as Record<string, Array<{ email: string }>>
      setOnline(Object.entries(state).map(([id, v]) => ({ id, email: v[0]?.email ?? '?' })))
    })
    ch.subscribe(async (s) => { if (s === 'SUBSCRIBED') await ch.track({ email }) })
    return () => { sb.removeChannel(ch) }
  }, [userId, email])
  return (
    <div className="panel p-4">
      <div className="flex items-center justify-between mb-3">
        <h3 className="text-sm font-semibold">Zespół online</h3>
        <span className="text-xs accent">{online.length}</span>
      </div>
      {online.length === 0 ? <p className="text-xs muted">Brak aktywnych sesji.</p> : (
        <ul className="space-y-2">
          {online.map(o => (
            <li key={o.id} className="flex items-center gap-2 text-sm">
              <span className="w-2 h-2 rounded-full" style={{ background: 'rgb(var(--accent))', boxShadow: '0 0 8px rgb(var(--accent))' }} />
              <span className="truncate">{o.email}</span>
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
EOF

# ---------- auth ----------
cat > app/\(auth\)/login/page.tsx <<'EOF'
'use client'
import { useState } from 'react'
import Link from 'next/link'
import { useRouter, useSearchParams } from 'next/navigation'
import { getBrowserClient } from '@/lib/supabase/client'
export default function LoginPage() {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [busy, setBusy] = useState(false)
  const [err, setErr] = useState<string | null>(null)
  const router = useRouter()
  const sp = useSearchParams()
  const next = sp.get('next') || '/dashboard'
  async function submit(e: React.FormEvent) {
    e.preventDefault(); setBusy(true); setErr(null)
    const { error } = await getBrowserClient().auth.signInWithPassword({ email, password })
    if (error) { setErr('Nieprawidłowy e-mail lub hasło.'); setBusy(false); return }
    router.push(next); router.refresh()
  }
  return (
    <main className="min-h-screen grid place-items-center p-4">
      <form onSubmit={submit} className="panel panel-glow p-6 w-full max-w-sm space-y-3">
        <div className="mb-2">
          <div className="text-xs accent uppercase tracking-widest">WorkHub</div>
          <h1 className="text-2xl font-semibold tracking-tight mt-1">Zaloguj się</h1>
          <p className="text-sm muted mt-1">Wspólna przestrzeń Twojego zespołu.</p>
        </div>
        {err && <p role="alert" className="text-sm text-red-400">{err}</p>}
        <label className="block text-sm">
          <span className="muted mb-1 block">E-mail</span>
          <input type="email" required autoComplete="email" value={email} onChange={e => setEmail(e.target.value)} className="input" />
        </label>
        <label className="block text-sm">
          <span className="muted mb-1 block">Hasło</span>
          <input type="password" required autoComplete="current-password" minLength={6} value={password} onChange={e => setPassword(e.target.value)} className="input" />
        </label>
        <button disabled={busy} className="btn btn-primary w-full">{busy ? 'Logowanie…' : 'Zaloguj'}</button>
        <p className="text-xs muted text-center">Nie masz konta? <Link href="/register" className="accent underline">Utwórz konto</Link></p>
      </form>
    </main>
  )
}
EOF

cat > app/\(auth\)/register/page.tsx <<'EOF'
'use client'
import { useState } from 'react'
import Link from 'next/link'
import { useRouter } from 'next/navigation'
import { getBrowserClient } from '@/lib/supabase/client'
export default function RegisterPage() {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [busy, setBusy] = useState(false)
  const [err, setErr] = useState<string | null>(null)
  const router = useRouter()
  async function submit(e: React.FormEvent) {
    e.preventDefault(); setBusy(true); setErr(null)
    const { data, error } = await getBrowserClient().auth.signUp({ email, password })
    if (error) { setErr(error.message); setBusy(false); return }
    if (!data.session) { setErr('Potwierdź e-mail z wiadomości, którą wysłał Supabase, a następnie się zaloguj.'); setBusy(false); return }
    router.push('/dashboard'); router.refresh()
  }
  return (
    <main className="min-h-screen grid place-items-center p-4">
      <form onSubmit={submit} className="panel panel-glow p-6 w-full max-w-sm space-y-3">
        <div className="mb-2">
          <div className="text-xs accent uppercase tracking-widest">WorkHub</div>
          <h1 className="text-2xl font-semibold tracking-tight mt-1">Utwórz konto</h1>
          <p className="text-sm muted mt-1">Dołącz do tej samej przestrzeni co zespół.</p>
        </div>
        {err && <p role="alert" className="text-sm text-red-400">{err}</p>}
        <label className="block text-sm">
          <span className="muted mb-1 block">E-mail</span>
          <input type="email" required autoComplete="email" value={email} onChange={e => setEmail(e.target.value)} className="input" />
        </label>
        <label className="block text-sm">
          <span className="muted mb-1 block">Hasło (min. 6 znaków)</span>
          <input type="password" required minLength={6} autoComplete="new-password" value={password} onChange={e => setPassword(e.target.value)} className="input" />
        </label>
        <button disabled={busy} className="btn btn-primary w-full">{busy ? 'Tworzenie…' : 'Zarejestruj się'}</button>
        <p className="text-xs muted text-center">Masz już konto? <Link href="/login" className="accent underline">Zaloguj się</Link></p>
      </form>
    </main>
  )
}
EOF

# ---------- app/(app)/layout ----------
cat > app/\(app\)/layout.tsx <<'EOF'
import { redirect } from 'next/navigation'
import { getServerClient } from '@/lib/supabase/server'
import Sidebar from '@/components/Sidebar'
import ThemeToggle from '@/components/ThemeToggle'
export default async function AppLayout({ children }: { children: React.ReactNode }) {
  const sb = await getServerClient()
  const { data: { user } } = await sb.auth.getUser()
  if (!user) redirect('/login')
  return (
    <div className="min-h-screen flex flex-col lg:flex-row">
      <Sidebar email={user.email ?? ''} />
      <div className="flex-1 flex flex-col min-w-0">
        <header className="flex items-center justify-between gap-3 px-4 lg:px-6 py-3 border-b" style={{ borderColor: 'rgb(var(--border))' }}>
          <div className="text-xs muted uppercase tracking-widest truncate">WorkHub · workspace</div>
          <ThemeToggle />
        </header>
        <main className="flex-1 p-4 lg:p-6 min-w-0">{children}</main>
      </div>
    </div>
  )
}
EOF

# ---------- dashboard ----------
cat > app/\(app\)/dashboard/page.tsx <<'EOF'
import { getServerClient } from '@/lib/supabase/server'
import TasksPanel from './TasksPanel'
import ActivityPanel from './ActivityPanel'
import PresenceBar from '@/components/PresenceBar'
import IntegrationsPanel from './IntegrationsPanel'
export const dynamic = 'force-dynamic'
export default async function DashboardPage() {
  const sb = await getServerClient()
  const { data: { user } } = await sb.auth.getUser()
  const [tasksRes, msgsRes] = await Promise.all([
    sb.from('tasks').select('*').order('created_at', { ascending: false }).limit(50),
    sb.from('messages').select('*').order('created_at', { ascending: false }).limit(20),
  ])
  return (
    <div className="space-y-4">
      <div>
        <h1 className="text-2xl font-semibold tracking-tight">Pulpit</h1>
        <p className="text-sm muted">Wspólny stan pracy zespołu — aktualizuje się w czasie rzeczywistym.</p>
      </div>
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-4">
        <div className="lg:col-span-2 space-y-4">
          <TasksPanel
            userId={user!.id}
            userEmail={user!.email ?? ''}
            initial={tasksRes.data ?? []}
          />
          <IntegrationsPanel />
        </div>
        <div className="space-y-4">
          <PresenceBar userId={user!.id} email={user!.email ?? ''} />
          <ActivityPanel initial={msgsRes.data ?? []} />
        </div>
      </div>
    </div>
  )
}
EOF

cat > app/\(app\)/dashboard/TasksPanel.tsx <<'EOF'
'use client'
import { useEffect, useState } from 'react'
import { getBrowserClient } from '@/lib/supabase/client'
import { Check, Plus, Trash2, Loader2 } from 'lucide-react'
type Task = { id: string; title: string; done: boolean; author_id: string; author_email: string; created_at: string }
export default function TasksPanel({ userId, userEmail, initial }: { userId: string; userEmail: string; initial: Task[] }) {
  const sb = getBrowserClient()
  const [tasks, setTasks] = useState<Task[]>(initial)
  const [title, setTitle] = useState('')
  const [busy, setBusy] = useState(false)
  const [err, setErr] = useState<string | null>(null)
  useEffect(() => {
    const ch = sb.channel('tasks-rt')
      .on('postgres_changes', { event: '*', schema: 'public', table: 'tasks' }, (p: any) => {
        if (p.eventType === 'INSERT') setTasks(cur => cur.some(t => t.id === p.new.id) ? cur : [p.new, ...cur])
        if (p.eventType === 'UPDATE') setTasks(cur => cur.map(t => t.id === p.new.id ? p.new : t))
        if (p.eventType === 'DELETE') setTasks(cur => cur.filter(t => t.id !== p.old.id))
      }).subscribe()
    return () => { sb.removeChannel(ch) }
  }, [sb])
  async function add(e: React.FormEvent) {
    e.preventDefault(); const t = title.trim(); if (!t) return
    setBusy(true); setErr(null); setTitle('')
    const { error } = await sb.from('tasks').insert({ title: t, author_id: userId, author_email: userEmail })
    if (error) setErr(error.message)
    setBusy(false)
  }
  async function toggle(t: Task) {
    setErr(null)
    const { error } = await sb.from('tasks').update({ done: !t.done }).eq('id', t.id)
    if (error) setErr(error.message)
  }
  async function remove(t: Task) {
    setErr(null)
    const { error } = await sb.from('tasks').delete().eq('id', t.id)
    if (error) setErr(error.message)
  }
  const open = tasks.filter(t => !t.done).length
  return (
    <section className="panel p-4">
      <header className="flex items-center justify-between mb-3">
        <h2 className="text-sm font-semibold">Zadania zespołu</h2>
        <span className="text-xs muted">{open} otwarte · {tasks.length} łącznie</span>
      </header>
      <form onSubmit={add} className="flex gap-2 mb-3">
        <input value={title} onChange={e => setTitle(e.target.value)} placeholder="Dodaj zadanie…" maxLength={500} className="input" aria-label="Nowe zadanie" />
        <button disabled={busy || !title.trim()} className="btn btn-primary shrink-0">{busy ? <Loader2 size={16} className="animate-spin" /> : <Plus size={16} />} Dodaj</button>
      </form>
      {err && <p role="alert" className="text-xs text-red-400 mb-2">{err}</p>}
      {tasks.length === 0 ? <p className="text-sm muted py-6 text-center">Brak zadań. Dodaj pierwsze.</p> : (
        <ul className="space-y-1.5 max-h-[420px] overflow-auto pr-1">
          {tasks.map(t => (
            <li key={t.id} className="flex items-start gap-2 p-2 rounded-md hover:bg-white/5 transition-colors">
              <button onClick={() => toggle(t)} aria-label={t.done ? 'Cofnij wykonanie' : 'Oznacz jako wykonane'} className="mt-0.5 w-5 h-5 rounded border grid place-items-center shrink-0"
                style={{ borderColor: t.done ? 'rgb(var(--accent))' : 'rgb(var(--border))', background: t.done ? 'rgb(var(--accent))' : 'transparent', color: 'rgb(var(--accent-fg))' }}>
                {t.done && <Check size={12} />}
              </button>
              <div className="flex-1 min-w-0">
                <div className={"text-sm break-words " + (t.done ? 'line-through muted' : '')}>{t.title}</div>
                <div className="text-[11px] muted mt-0.5">{t.author_email} · {new Date(t.created_at).toLocaleString('pl-PL')}</div>
              </div>
              <button onClick={() => remove(t)} aria-label="Usuń zadanie" className="p-1 rounded opacity-60 hover:opacity-100 hover:text-red-400"><Trash2 size={14} /></button>
            </li>
          ))}
        </ul>
      )}
    </section>
  )
}
EOF

cat > app/\(app\)/dashboard/ActivityPanel.tsx <<'EOF'
'use client'
import { useEffect, useState } from 'react'
import { getBrowserClient } from '@/lib/supabase/client'
type Msg = { id: string; body: string; author_email: string; created_at: string }
export default function ActivityPanel({ initial }: { initial: Msg[] }) {
  const sb = getBrowserClient()
  const [msgs, setMsgs] = useState<Msg[]>(initial)
  useEffect(() => {
    const ch = sb.channel('activity-rt')
      .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'messages' }, (p: any) => {
        setMsgs(cur => cur.some(m => m.id === p.new.id) ? cur : [p.new, ...cur].slice(0, 20))
      }).subscribe()
    return () => { sb.removeChannel(ch) }
  }, [sb])
  return (
    <section className="panel p-4">
      <h2 className="text-sm font-semibold mb-3">Ostatnia aktywność</h2>
      {msgs.length === 0 ? <p className="text-xs muted">Cisza. Zacznij rozmowę w Czacie.</p> : (
        <ul className="space-y-2.5">
          {msgs.slice(0, 8).map(m => (
            <li key={m.id} className="text-xs">
              <div className="muted">{m.author_email} · {new Date(m.created_at).toLocaleTimeString('pl-PL')}</div>
              <div className="truncate">{m.body}</div>
            </li>
          ))}
        </ul>
      )}
    </section>
  )
}
EOF

cat > app/\(app\)/dashboard/IntegrationsPanel.tsx <<'EOF'
import { Github, Sparkles } from 'lucide-react'
export default function IntegrationsPanel() {
  const gh = Boolean(process.env.GITHUB_TOKEN && process.env.GITHUB_REPO)
  const ai = Boolean(process.env.OPENAI_API_KEY)
  return (
    <section className="panel p-4">
      <h2 className="text-sm font-semibold mb-3">Integracje</h2>
      <div className="grid grid-cols-1 sm:grid-cols-2 gap-2">
        <div className="p-3 rounded-md border flex items-center gap-3" style={{ borderColor: 'rgb(var(--border))' }}>
          <Github size={18} className="accent" />
          <div className="flex-1">
            <div className="text-sm">GitHub</div>
            <div className="text-xs muted">{gh ? 'Gotowe' : 'Wymaga GITHUB_TOKEN i GITHUB_REPO'}</div>
          </div>
          <span className={"text-[10px] px-1.5 py-0.5 rounded " + (gh ? 'bg-emerald-500/15 text-emerald-400' : 'bg-amber-500/15 text-amber-400')}>{gh ? 'OK' : 'Brak'}</span>
        </div>
        <div className="p-3 rounded-md border flex items-center gap-3" style={{ borderColor: 'rgb(var(--border))' }}>
          <Sparkles size={18} className="accent" />
          <div className="flex-1">
            <div className="text-sm">Asystent AI</div>
            <div className="text-xs muted">{ai ? 'Gotowe' : 'Wymaga OPENAI_API_KEY'}</div>
          </div>
          <span className={"text-[10px] px-1.5 py-0.5 rounded " + (ai ? 'bg-emerald-500/15 text-emerald-400' : 'bg-amber-500/15 text-amber-400')}>{ai ? 'OK' : 'Brak'}</span>
        </div>
      </div>
    </section>
  )
}
EOF

# ---------- chat ----------
cat > app/\(app\)/chat/page.tsx <<'EOF'
import ChatPanel from './ChatPanel'
export default function ChatPage() { return <ChatPanel /> }
EOF

cat > app/\(app\)/chat/ChatPanel.tsx <<'EOF'
'use client'
import { useEffect, useRef, useState } from 'react'
import { getBrowserClient } from '@/lib/supabase/client'
import { Loader2, Send } from 'lucide-react'
type Msg = { id: string; body: string; author_id: string; author_email: string; created_at: string }
export default function ChatPanel() {
  const sb = getBrowserClient()
  const [me, setMe] = useState<{ id: string; email: string } | null>(null)
  const [msgs, setMsgs] = useState<Msg[]>([])
  const [text, setText] = useState('')
  const [busy, setBusy] = useState(false)
  const [loading, setLoading] = useState(true)
  const [err, setErr] = useState<string | null>(null)
  const endRef = useRef<HTMLDivElement>(null)
  useEffect(() => {
    ;(async () => {
      const { data: { user } } = await sb.auth.getUser()
      if (user) setMe({ id: user.id, email: user.email ?? '' })
      const { data, error } = await sb.from('messages').select('*').order('created_at', { ascending: true }).limit(200)
      if (error) setErr(error.message); else setMsgs(data ?? [])
      setLoading(false)
    })()
  }, [sb])
  useEffect(() => {
    const ch = sb.channel('chat-rt')
      .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'messages' }, (p: any) => {
        setMsgs(cur => cur.some(m => m.id === p.new.id) ? cur : [...cur, p.new])
      }).subscribe()
    return () => { sb.removeChannel(ch) }
  }, [sb])
  useEffect(() => { endRef.current?.scrollIntoView({ behavior: 'smooth' }) }, [msgs.length])
  async function send(e: React.FormEvent) {
    e.preventDefault()
    const body = text.trim(); if (!body || !me) return
    setBusy(true); setErr(null); setText('')
    const { error } = await sb.from('messages').insert({ body, author_id: me.id, author_email: me.email })
    if (error) setErr(error.message)
    setBusy(false)
  }
  return (
    <div className="flex flex-col h-[calc(100vh-8rem)] max-h-[800px]">
      <div className="mb-3 flex items-center justify-between">
        <div>
          <h1 className="text-2xl font-semibold tracking-tight">Czat</h1>
          <p className="text-sm muted">Wspólny wątek całego zespołu.</p>
        </div>
      </div>
      <div className="panel flex-1 flex flex-col min-h-0">
        <div className="flex-1 overflow-auto p-4 space-y-3">
          {loading ? <p className="text-sm muted text-center">Ładowanie…</p>
          : msgs.length === 0 ? <p className="text-sm muted text-center py-8">Brak wiadomości. Napisz pierwszą.</p>
          : msgs.map(m => {
              const mine = m.author_id === me?.id
              return (
                <div key={m.id} className={"flex " + (mine ? 'justify-end' : 'justify-start')}>
                  <div className="max-w-[80%]">
                    <div className={"text-[11px] muted mb-0.5 " + (mine ? 'text-right' : '')}>{m.author_email} · {new Date(m.created_at).toLocaleTimeString('pl-PL')}</div>
                    <div className={"px-3 py-2 rounded-lg text-sm whitespace-pre-wrap break-words border " + (mine ? '' : '')}
                      style={mine ? { background: 'rgb(var(--accent) / 0.14)', borderColor: 'rgb(var(--accent) / 0.3)' } : { background: 'rgb(var(--bg) / 0.5)', borderColor: 'rgb(var(--border))' }}>
                      {m.body}
                    </div>
                  </div>
                </div>
              )
            })}
          <div ref={endRef} />
        </div>
        {err && <p role="alert" className="text-xs text-red-400 px-4 pb-2">{err}</p>}
        <form onSubmit={send} className="p-3 border-t flex gap-2" style={{ borderColor: 'rgb(var(--border))' }}>
          <input value={text} onChange={e => setText(e.target.value)} maxLength={4000} placeholder="Napisz wiadomość…" className="input" aria-label="Wiadomość" />
          <button disabled={busy || !text.trim()} className="btn btn-primary shrink-0">{busy ? <Loader2 size={16} className="animate-spin" /> : <Send size={16} />} Wyślij</button>
        </form>
      </div>
    </div>
  )
}
EOF

# ---------- repos ----------
cat > app/\(app\)/repos/page.tsx <<'EOF'
import ReposClient from './ReposClient'
export default function ReposPage() { return <ReposClient /> }
EOF

cat > app/\(app\)/repos/ReposClient.tsx <<'EOF'
'use client'
import { useEffect, useState } from 'react'
import { GitBranch, GitCommit, FileText, Loader2, Github } from 'lucide-react'
type Branch = { name: string }
type Commit = { sha: string; commit: { message: string; author: { name: string; date: string } } }
type Tree = { path: string; type: string }
export default function ReposClient() {
  const [tab, setTab] = useState<'branches'|'commits'|'tree'>('commits')
  const [data, setData] = useState<any>(null)
  const [loading, setLoading] = useState(true)
  const [err, setErr] = useState<string | null>(null)
  const [missing, setMissing] = useState<string[] | null>(null)
  async function load(which: string) {
    setLoading(true); setErr(null)
    const r = await fetch(`/api/github?what=${which}`)
    const j = await r.json()
    if (j.configured === false) { setMissing(j.missing ?? ['GITHUB_TOKEN']); setData(null) }
    else if (!r.ok) setErr(j.error ?? 'Błąd GitHub API')
    else setData(j.data)
    setLoading(false)
  }
  useEffect(() => { load(tab) }, [tab])
  return (
    <div className="space-y-4">
      <div>
        <h1 className="text-2xl font-semibold tracking-tight">Repozytoria</h1>
        <p className="text-sm muted">Gałęzie, commity i pliki Twojego repozytorium GitHub.</p>
      </div>
      {missing && (
        <div className="panel p-5">
          <div className="flex items-center gap-3 mb-3"><Github size={20} className="accent" /><h2 className="font-semibold">Konfiguracja GitHuba</h2></div>
          <p className="text-sm muted mb-3">Ustaw w zmiennych środowiskowych hostingu:</p>
          <ul className="text-sm space-y-1">
            {missing.map(m => <li key={m}><code className="px-1.5 py-0.5 rounded bg-black/30">{m}</code></li>)}
          </ul>
          <p className="text-xs muted mt-3">GITHUB_REPO ma format <code>owner/repo</code>. Token potrzebuje uprawnień do odczytu repozytorium.</p>
        </div>
      )}
      {!missing && (
        <>
          <div className="flex gap-1">
            {(['commits','branches','tree'] as const).map(t => (
              <button key={t} onClick={() => setTab(t)} className={"btn " + (tab === t ? 'text-[rgb(var(--accent))] border-[rgb(var(--accent))]' : '')}>
                {t === 'commits' ? <><GitCommit size={14}/> Commity</> : t === 'branches' ? <><GitBranch size={14}/> Gałęzie</> : <><FileText size={14}/> Pliki</>}
              </button>
            ))}
          </div>
          {err && <p role="alert" className="text-sm text-red-400">{err}</p>}
          {loading ? <p className="text-sm muted flex items-center gap-2"><Loader2 size={14} className="animate-spin" /> Ładowanie…</p>
          : <div className="panel p-4">
              {tab === 'commits' && (data as Commit[] | undefined)?.map(c => (
                <div key={c.sha} className="py-2 border-b last:border-0" style={{ borderColor: 'rgb(var(--border))' }}>
                  <div className="text-sm">{c.commit.message.split('\n')[0]}</div>
                  <div className="text-[11px] muted mt-0.5"><code>{c.sha.slice(0,7)}</code> · {c.commit.author.name} · {new Date(c.commit.author.date).toLocaleString('pl-PL')}</div>
                </div>
              ))}
              {tab === 'branches' && (data as Branch[] | undefined)?.map(b => (
                <div key={b.name} className="text-sm py-1.5 flex items-center gap-2"><GitBranch size={14} className="accent" /> {b.name}</div>
              ))}
              {tab === 'tree' && (data as { tree?: Tree[] } | undefined)?.tree?.slice(0, 300).map(f => (
                <div key={f.path} className="text-xs py-1 flex items-center gap-2 muted"><FileText size={12} /> <span className="truncate">{f.path}</span></div>
              ))}
            </div>}
        </>
      )}
    </div>
  )
}
EOF

cat > app/api/github/route.ts <<'EOF'
export const runtime = 'nodejs'
export async function GET(req: Request) {
  const token = process.env.GITHUB_TOKEN
  const repo = process.env.GITHUB_REPO
  const missing: string[] = []
  if (!token) missing.push('GITHUB_TOKEN')
  if (!repo) missing.push('GITHUB_REPO')
  if (missing.length) return Response.json({ configured: false, missing })
  const what = new URL(req.url).searchParams.get('what') ?? 'commits'
  const headers = { Authorization: `Bearer ${token}`, Accept: 'application/vnd.github+json', 'User-Agent': 'workhub' }
  let api = ''
  if (what === 'commits') api = `https://api.github.com/repos/${repo}/commits?per_page=15`
  else if (what === 'branches') api = `https://api.github.com/repos/${repo}/branches?per_page=30`
  else if (what === 'tree') api = `https://api.github.com/repos/${repo}/git/trees/HEAD?recursive=1`
  else return Response.json({ error: 'Nieznany zasób' }, { status: 400 })
  const r = await fetch(api, { headers })
  if (!r.ok) return Response.json({ error: `GitHub: ${r.status} ${r.statusText}` }, { status: r.status })
  return Response.json({ configured: true, data: await r.json() })
}
EOF

# ---------- AI ----------
cat > app/\(app\)/ai/page.tsx <<'EOF'
import AiClient from './AiClient'
export default function AiPage() { return <AiClient /> }
EOF

cat > app/\(app\)/ai/AiClient.tsx <<'EOF'
'use client'
import { useState } from 'react'
import { Loader2, Sparkles, Send } from 'lucide-react'
export default function AiClient() {
  const [prompt, setPrompt] = useState('')
  const [answer, setAnswer] = useState('')
  const [busy, setBusy] = useState(false)
  const [err, setErr] = useState<string | null>(null)
  const [needConfig, setNeedConfig] = useState<string[] | null>(null)
  async function send(e: React.FormEvent) {
    e.preventDefault()
    const p = prompt.trim(); if (!p) return
    setBusy(true); setErr(null); setNeedConfig(null); setAnswer('')
    try {
      const r = await fetch('/api/ai', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ prompt: p }) })
      if (r.status === 503) {
        const j = await r.json().catch(() => ({}))
        setNeedConfig([j.missing ?? 'OPENAI_API_KEY'])
        setBusy(false); return
      }
      if (!r.ok) { const t = await r.text(); setErr(`Błąd: ${t || r.status}`); setBusy(false); return }
      if (!r.body) { setErr('Pusta odpowiedź'); setBusy(false); return }
      const reader = r.body.getReader()
      const dec = new TextDecoder()
      let buf = ''
      while (true) {
        const { value, done } = await reader.read()
        if (done) break
        buf += dec.decode(value, { stream: true })
        const parts = buf.split('\n\n'); buf = parts.pop() ?? ''
        for (const part of parts) {
          for (const line of part.split('\n')) {
            if (!line.startsWith('data:')) continue
            const data = line.slice(5).trim()
            if (data === '[DONE]') continue
            try {
              const j = JSON.parse(data)
              const delta = j.choices?.[0]?.delta?.content
              if (delta) setAnswer(a => a + delta)
            } catch {}
          }
        }
      }
    } catch (e: any) { setErr(e?.message ?? 'Błąd połączenia') }
    setBusy(false)
  }
  return (
    <div className="space-y-4 max-w-3xl">
      <div>
        <h1 className="text-2xl font-semibold tracking-tight">Asystent AI</h1>
        <p className="text-sm muted">Wywołanie po stronie serwera. Klucz API nigdy nie trafia do przeglądarki.</p>
      </div>
      {needConfig && (
        <div className="panel p-4">
          <h2 className="font-semibold mb-2 flex items-center gap-2"><Sparkles size={16} className="accent" /> Brak konfiguracji</h2>
          <p className="text-sm muted mb-2">Ustaw na hostingu:</p>
          <ul className="text-sm space-y-1">{needConfig.map(k => <li key={k}><code className="px-1.5 py-0.5 rounded bg-black/30">{k}</code></li>)}</ul>
        </div>
      )}
      <form onSubmit={send} className="panel p-4 space-y-3">
        <label className="block text-sm">
          <span className="muted mb-1 block">Prompt</span>
          <textarea value={prompt} onChange={e => setPrompt(e.target.value)} rows={4} maxLength={4000} className="input resize-none" placeholder="O co chcesz zapytać?" />
        </label>
        <button disabled={busy || !prompt.trim()} className="btn btn-primary">{busy ? <><Loader2 size={16} className="animate-spin" /> Generowanie…</> : <><Send size={16} /> Wyślij</>}</button>
        {err && <p role="alert" className="text-sm text-red-400">{err}</p>}
      </form>
      {(answer || busy) && (
        <div className="panel p-4">
          <div className="text-xs muted mb-2">Odpowiedź</div>
          <div className="text-sm whitespace-pre-wrap break-words">{answer || '…'}</div>
        </div>
      )}
    </div>
  )
}
EOF

cat > app/api/ai/route.ts <<'EOF'
export const runtime = 'nodejs'
export async function POST(req: Request) {
  const apiKey = process.env.OPENAI_API_KEY
  if (!apiKey) return Response.json({ missing: 'OPENAI_API_KEY' }, { status: 503 })
  const base = (process.env.OPENAI_BASE_URL || 'https://api.openai.com/v1').replace(/\/$/, '')
  const model = process.env.OPENAI_MODEL || 'gpt-4o-mini'
  let body: any
  try { body = await req.json() } catch { return Response.json({ error: 'Nieprawidłowe dane' }, { status: 400 }) }
  const prompt = typeof body?.prompt === 'string' ? body.prompt.trim() : ''
  if (!prompt) return Response.json({ error: 'Pusty prompt' }, { status: 400 })
  if (prompt.length > 4000) return Response.json({ error: 'Prompt zbyt długi' }, { status: 400 })
  const upstream = await fetch(`${base}/chat/completions`, {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${apiKey}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({ model, stream: true, messages: [{ role: 'user', content: prompt }] }),
  })
  if (!upstream.ok) {
    const t = await upstream.text().catch(() => '')
    return new Response(t || `Upstream ${upstream.status}`, { status: upstream.status })
  }
  return new Response(upstream.body, { headers: { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache', 'X-Accel-Buffering': 'no' } })
}
EOF

cat > app/auth/callback/route.ts <<'EOF'
import { NextResponse } from 'next/server'
import { getServerClient } from '@/lib/supabase/server'
export async function GET(req: Request) {
  const url = new URL(req.url)
  const code = url.searchParams.get('code')
  if (code) { const sb = await getServerClient(); await sb.auth.exchangeCodeForSession(code) }
  return NextResponse.redirect(new URL('/dashboard', url.origin))
}
EOF

# ---------- README ----------
cat > README.md <<'EOF'
# WorkHub

Wspólna przestrzeń zespołu: pulpit, zadania, czat, repozytoria GitHub i asystent AI.
Stack: Next.js 14 (App Router) · Supabase (Auth, Postgres, Realtime) · Tailwind.

## Wymagania
- Konto Supabase (darmowe)
- Konto Vercel (darmowe)

## Konfiguracja Supabase
1. Utwórz projekt na supabase.com.
2. SQL Editor → wklej `supabase/schema.sql` → Run.
3. Authentication → Providers → Email: **odznacz** „Confirm email” (dla MVP, żeby rejestracja logowała od razu).
4. Settings → API: skopiuj `Project URL` i `anon public` key.

## Wdrożenie Vercel
1. Push do GitHuba.
2. Vercel → New Project → Import repo.
3. Environment Variables: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`.
4. (Opcjonalnie) `GITHUB_TOKEN`, `GITHUB_REPO`, `OPENAI_API_KEY`, `OPENAI_MODEL`, `OPENAI_BASE_URL`.
5. Deploy → dostajesz publiczny URL HTTPS.

## Test
- Otwórz URL → Zarejestruj się → Zaloguj.
- W drugiej przeglądarce / trybie incognito → Zarejestruj drugie konto.
- Dodaj zadanie → pojawia się w drugiej sesji w ciągu sekundy.
- Napisz na czacie → to samo.
- Sprawdź obecność online (dwa avatary).
EOF


chmod +x workhub-setup.sh
echo "✅ WorkHub scaffold gotowy. Teraz: npm install && npm run dev"
