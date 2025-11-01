# AMIN-TOUCH-DESHBOARD
# AMIN TOUCH DASHBOARD SYSTEM — Full Project Scaffold (Next.js + Supabase)

This document contains a ready-to-use project scaffold for **AMIN TOUCH DASHBOARD SYSTEM** using **Next.js (pages)**, **TailwindCSS**, and **Supabase** as the backend (Auth, Database, Storage). It includes login system (admin + staff), admin and staff dashboards, invoice download (CSV/PDF), notifications via email (Supabase SMTP or SendGrid), Bengali UI, dark/light theme, and instructions to run & deploy.

---

## Project structure (files to create)

```
amin-touch-dashboard/
├─ README.md
├─ package.json
├─ next.config.js
├─ tailwind.config.js
├─ postcss.config.js
├─ .env.local (gitignored)
├─ public/
│  ├─ logo.png        # place your uploaded logo here
│  └─ bg-login.jpg    # attractive background for login
├─ pages/
│  ├─ _app.js
│  ├─ index.js        # landing / choose admin or staff
│  ├─ admin-login.js
│  ├─ staff-login.js
│  ├─ admin/
│  │  └─ index.js     # admin dashboard
│  ├─ staff/
│  │  └─ index.js     # staff dashboard
│  ├─ api/
│  │  ├─ supabase-auth.js
│  │  ├─ invoice.js   # generate PDF/CSV
│  │  └─ notify-admin.js
├─ lib/
│  └─ supabaseClient.js
├─ components/
│  ├─ Header.js
│  ├─ Footer.js
│  ├─ Layout.js
│  ├─ LoginForm.js
│  ├─ DataTable.js
│  ├─ SummaryCards.js
│  └─ ThemeToggle.js
└─ utils/
   ├─ csv.js
   └─ pdf.js
```

---

## KEY FILES (paste these into the corresponding files)

> NOTE: Below are the most important files. Create them under your project and adjust as needed.

### package.json

```json
{
  "name": "amin-touch-dashboard",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3000",
    "build": "next build",
    "start": "next start -p 3000",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "13.4.7",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "@supabase/supabase-js": "^2.0.0",
    "tailwindcss": "^3.0.0",
    "autoprefixer": "^10.0.0",
    "postcss": "^8.0.0",
    "jsPDF": "^2.5.1",
    "papaparse": "^5.3.2",
    "clsx": "^1.2.1"
  }
}
```

---

### lib/supabaseClient.js

```js
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY

export const supabase = createClient(supabaseUrl, supabaseAnonKey)
```

---

### pages/_app.js (wrap with Tailwind + Layout)

```js
import '../styles/globals.css'
import { useEffect } from 'react'
import Layout from '../components/Layout'

function MyApp({ Component, pageProps }) {
  return (
    <Layout>
      <Component {...pageProps} />
    </Layout>
  )
}

export default MyApp
```

---

### pages/index.js (landing)

```js
import Link from 'next/link'
export default function Home(){
  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50 dark:bg-gray-900">
      <div className="p-8 bg-white dark:bg-gray-800 rounded-lg shadow-lg text-center">
        <img src="/logo.png" alt="logo" className="mx-auto h-20" />
        <h1 className="text-2xl font-bold mt-4">AMIN TOUCH DASHBOARD SYSTEM</h1>
        <p className="mt-2 text-gray-600">AMIN TOUCH TRADING CONTRACTING & HOSPITALITY SERVICES</p>
        <div className="mt-6 space-x-4">
          <Link href="/admin-login"><a className="px-4 py-2 bg-indigo-600 text-white rounded">Admin Login</a></Link>
          <Link href="/staff-login"><a className="px-4 py-2 border rounded">Staff Login</a></Link>
        </div>
      </div>
    </div>
  )
}
```

---

### pages/admin-login.js and pages/staff-login.js

Both pages use the same `LoginForm` component but point to different auth flows. For simplicity we will use Supabase row-level users stored in `staff` table and custom admin check.

```js
// pages/admin-login.js
import LoginForm from '../components/LoginForm'

export default function AdminLogin(){
  return <LoginForm type="admin" />
}
```

```js
// pages/staff-login.js
import LoginForm from '../components/LoginForm'

export default function StaffLogin(){
  return <LoginForm type="staff" />
}
```

---

### components/LoginForm.js (core auth UI + local credential fallback)

```js
import { useState } from 'react'
import { supabase } from '../lib/supabaseClient'
import Router from 'next/router'

export default function LoginForm({ type }){
  const [username, setUsername] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')

  async function handleSubmit(e){
    e.preventDefault()
    setError('')

    // For demo: allow serverless credential check via Supabase table 'staff' or hardcoded admin
    if(type==='admin'){
      if(username==='admin9197' && password==='Admin9197'){
        // set localStorage and redirect
        localStorage.setItem('user', JSON.stringify({ role:'admin', name:'AL AMIN', username:'admin9197' }))
        Router.push('/admin')
        return
      } else {
        setError('আপনার admin credential সঠিক নয়')
        return
      }
    }

    // staff: check hardcoded credentials OR lookup in supabase
    const staffCreds = [
      { username:'ronytalukder', password:'@jead2016R', name:'RONY TALUKDER' },
      { username:'mahir', password:'Mahir3', name:'MAHIR' },
      { username:'sakiladnan', password:'Sakiladnan', name:'SAKIL ADNAN' }
    ]
    const found = staffCreds.find(s => s.username===username && s.password===password)
    if(found){
      localStorage.setItem('user', JSON.stringify({ role:'staff', ...found }))
      Router.push('/staff')
      return
    }

    // Optionally: check Supabase 'staff' table
    const { data, error } = await supabase.from('staff').select('*').eq('username', username).single()
    if(error){
      // ignore
    } else if(data && data.password === password){
      localStorage.setItem('user', JSON.stringify({ role:'staff', name:data.name, username:data.username }))
      Router.push('/staff')
      return
    }

    setError('ইউজারনেম অথবা পাসওয়ার্ড ভুল')
  }

  return (
    <div className="min-h-screen flex items-center justify-center bg-login bg-cover">
      <form onSubmit={handleSubmit} className="bg-white/90 p-6 rounded shadow max-w-md w-full">
        <h2 className="text-xl font-bold mb-4">{type==='admin' ? 'অ্যাডমিন লগইন' : 'স্টাফ লগইন'}</h2>
        <label className="block mb-2">ইউজারনেম
          <input value={username} onChange={e=>setUsername(e.target.value)} className="w-full border p-2 mt-1" />
        </label>
        <label className="block mb-2">পাসওয়ার্ড
          <input type="password" value={password} onChange={e=>setPassword(e.target.value)} className="w-full border p-2 mt-1" />
        </label>
        {error && <p className="text-red-600">{error}</p>}
        <button className="mt-4 px-4 py-2 bg-indigo-600 text-white rounded">লগইন</button>
      </form>
    </div>
  )
}
```

---

### pages/admin/index.js (Admin Dashboard - simplified)

```js
import { useEffect, useState } from 'react'
import { supabase } from '../../lib/supabaseClient'

export default function AdminDashboard(){
  const [staffList, setStaffList] = useState([])

  useEffect(()=>{
    async function load(){
      const { data } = await supabase.from('staff').select('*')
      setStaffList(data||[])
    }
    load()
  },[])

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold">অ্যাডমিন ড্যাশবোর্ড</h1>
      <div className="mt-4">
        <table className="w-full">
          <thead>
            <tr><th>নাম</th><th>ইউজারনেম</th><th>বর্তমান ক্যাশ</th><th>অ্যাকশন</th></tr>
          </thead>
          <tbody>
            {staffList.map(s=> (
              <tr key={s.id} className="border-t">
                <td>{s.name}</td>
                <td>{s.username}</td>
                <td>{s.current_cash || 0}</td>
                <td><a className="underline" href={`/api/invoice?username=${s.username}`}>Download Invoice</a></td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  )
}
```

---

### pages/staff/index.js (Staff Dashboard - simplified)

```js
import { useEffect, useState } from 'react'
import { supabase } from '../../lib/supabaseClient'

export default function StaffDashboard(){
  const [entries, setEntries] = useState([])
  const user = typeof window !== 'undefined' ? JSON.parse(localStorage.getItem('user')||null) : null

  useEffect(()=>{
    if(!user) return
    async function load(){
      const { data } = await supabase.from('daily_income').select('*').eq('username', user.username).order('date', { ascending:false })
      setEntries(data||[])
    }
    load()
  },[user])

  async function addDummy(){
    // For demo: create an entry with auto-calcs
    const payload = { username: user.username, date: new Date().toISOString().slice(0,10), daily_income: 100, income_minus: 10, total_cash: 90, entry_time: new Date().toISOString() }
    await supabase.from('daily_income').insert([payload])
    setEntries(prev=>[payload,...prev])
    // notify admin
    await fetch('/api/notify-admin', { method:'POST', headers:{'content-type':'application/json'}, body: JSON.stringify({ username: user.username, name:user.name, action:'created entry' }) })
  }

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold">স্টাফ ড্যাশবোর্ড</h1>
      <button onClick={addDummy} className="mt-4 px-3 py-2 bg-green-600 text-white rounded">নতুন এন্ট্রি যোগ করো (ডেমো)</button>

      <div className="mt-6">
        <table className="w-full">
          <thead><tr><th>Date</th><th>Daily</th><th>Minus</th><th>Total Cash</th></tr></thead>
          <tbody>
            {entries.map((e,i)=> (
              <tr key={i} className="border-t"><td>{e.date}</td><td>{e.daily_income}</td><td>{e.income_minus}</td><td>{e.total_cash}</td></tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  )
}
```

---

### API: pages/api/notify-admin.js

```js
import { supabase } from '../../lib/supabaseClient'

export default async function handler(req,res){
  if(req.method!=='POST') return res.status(405).end()
  const { username, name, action } = req.body || {}

  // You can either trigger Supabase Functions or send via SMTP provider. For simplicity, save a notification row in 'notifications' table.
  const { data, error } = await supabase.from('notifications').insert([{ username, name, action, created_at: new Date() }])

  // Optionally: call external email provider from server (SendGrid) using SERVICE_ROLE_KEY in environment and server-side SMTP

  if(error) return res.status(500).json({ error })
  res.status(200).json({ ok:true })
}
```

---

### pages/api/invoice.js (CSV generator)

```js
import { supabase } from '../../lib/supabaseClient'
import { parse } from 'papaparse'

export default async function handler(req,res){
  const { username } = req.query
  if(!username) return res.status(400).json({ error:'username required' })
  const { data } = await supabase.from('daily_income').select('*').eq('username', username).order('date', { ascending:true })

  // convert to CSV
  const header = ['date','daily_income','income_minus','total_cash','entry_time']
  const csv = [header.join(',')].concat((data||[]).map(r => [r.date,r.daily_income,r.income_minus,r.total_cash,r.entry_time].join(','))).join('\n')

  res.setHeader('Content-Type','text/csv')
  res.setHeader('Content-Disposition', `attachment; filename="invoice_${username}.csv"`)
  res.send(csv)
}
```

---

## Supabase DB Schema (recommended)

Run these SQL statements in Supabase SQL editor:

```sql
-- staff table
create table if not exists staff (
  id bigint generated by default as identity primary key,
  username text unique,
  password text,
  name text,
  current_cash numeric default 0
);

-- daily income
create table if not exists daily_income (
  id bigint generated by default as identity primary key,
  username text references staff(username),
  date date,
  daily_income numeric,
  income_minus numeric,
  total_cash numeric,
  minus_regen text,
  entry_time timestamp
);

-- otp system
create table if not exists otp_income (
  id bigint generated by default as identity primary key,
  username text references staff(username),
  date date,
  otp_use numeric,
  otp_use_time time,
  otp_minus numeric,
  total_otp_cash numeric,
  otp_minus_regen text,
  entry_time timestamp
);

-- notifications
create table if not exists notifications (
  id bigint generated by default as identity primary key,
  username text,
  name text,
  action text,
  created_at timestamp default now()
);
```

---

## Environment variables (.env.local)

```
NEXT_PUBLIC_SUPABASE_URL=https://<your>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...   # keep secret, used only server-side
NEXT_PUBLIC_APP_NAME="AMIN TOUCH DASHBOARD SYSTEM"
```

---

## Run locally (Codespaces or locally)

1. create project files as above
2. `npm install`
3. populate `.env.local` with Supabase keys
4. `npm run dev` → open http://localhost:3000

If you are in GitHub Codespaces: open the forwarded port 3000 in the Ports panel to preview.


---

## Deployment (Vercel)

1. Push repo to GitHub.
2. Import to Vercel and connect to the GitHub repo.
3. Add the environment variables in Vercel project settings (use same names as `.env.local`).
4. Deploy — automatic on every push.

---

## Notes & Next steps (what I implemented in scaffold)
- Login pages for Admin & Staff (hardcoded admin credentials included). Staff credentials included as demo and also supports Supabase staff table.
- Admin dashboard lists staff and allows CSV invoice download.
- Staff dashboard allows adding demo entries and sends notification rows to `notifications` table.
- Invoice generation API produces CSV. You can extend to PDF using jsPDF or server-side PDF library.
- Email notifications: you can connect SendGrid or SMTP and call provider from `/api/notify-admin` using the SUPABASE_SERVICE_ROLE_KEY or another server-side secret.
- Bengali UI strings are used in key places.
- Light/Dark theme toggle, footer, and animations to be added in `components/Layout.js` and `components/ThemeToggle.js` (you can copy patterns from many Tailwind templates).

---

## Admin & Staff default credentials (for demo)

- Admin: `admin9197` / `Admin9197` (Name: AL AMIN)
- Staff 1: `ronytalukder` / `@jead2016R` (RONY TALUKDER)
- Staff 2: `mahir` / `Mahir3` (MAHIR)
- Staff 3: `sakiladnan` / `Sakiladnan` (SAKIL ADNAN)

(You should insert these into `staff` table in Supabase for full DB-based login.)

---

## Final: what I can do next for you (choose one)
1. Generate all project files and push to your GitHub repo (I can provide exact git commands and a zip-style file content). 
2. Create full UI components (Header, Footer, ThemeToggle, SummaryCards) and paste code here.
3. Add SendGrid email sending code to `/api/notify-admin` and show how to configure.

বলো কোনটা করতে চাই — আমি তোমার পছন্দ অনুযায়ী পরবর্তী ফাইল/কোড তৈরি করে দেব।
