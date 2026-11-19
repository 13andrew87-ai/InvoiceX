{
  "name": "trade-invoice-app",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "@stripe/react-stripe-js": "^2.7.3",
    "@stripe/stripe-js": "^4.1.0",
    "jspdf": "^2.5.1",
    "next": "14.2.5",
    "react": "18.3.1",
    "react-dom": "18.3.1",
    "stripe": "^16.6.0"
  },
  "devDependencies": {
    "@types/node": "20.14.10",
    "@types/react": "18.3.3",
    "@types/react-dom": "18.3.0",
    "autoprefixer": "10.4.19",
    "postcss": "8.4.41",
    "tailwindcss": "3.4.10",
    "typescript": "5.5.4"
  }
}/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./src/**/*.{js,ts,jsx,tsx}"],
  theme: { extend: {} },
  plugins: [],
};@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --bg-app: #0b0f19;
  --bg-card: rgba(17, 24, 39, 0.7);
  --text-main: #f3f4f6;
  --text-muted: #9ca3af;
  --border-color: rgba(56, 189, 248, 0.15);
}

body {
  background-color: var(--bg-app);
  color: var(--text-main);
  background-image: 
    radial-gradient(circle at 10% 20%, rgba(56, 189, 248, 0.04) 0%, transparent 40%),
    radial-gradient(circle at 90% 80%, rgba(45, 212, 191, 0.04) 0%, transparent 40%);
  background-attachment: fixed;
  font-family: system-ui, -apple-system, sans-serif;
}

.cyber-sooth-card {
  background: var(--bg-card);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border: 1px solid var(--border-color);
  box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.37);
  transition: all 0.3s ease;
}

.cyber-sooth-card:hover {
  border-color: rgba(56, 189, 248, 0.35);
}
4. src/lib/types.ts


export interface Company {
  id: string;
  name: string;
  address: string;
  licenseNumber?: string;
  taxId?: string;
  paymentTerms: "Net 15" | "Net 30";
  defaultHourlyRate: number;
  taxRate: number;
  taxMaterials: boolean;
  taxLabor: boolean;
  stripeApiKey?: string;
  reminderEmailEnabled: boolean;
}

export interface Client {
  id: string;
  companyName: string;
  primaryContact: string;
  billingEmail: string;
  phone: string;
  taxId?: string;
  poRequired: boolean;
  addresses: string[];
}

export interface LineItem {
  id: string;
  type: "labor" | "material" | "fee";
  description: string;
  quantity: number;
  unitPrice: number;
  markup?: number;
}

export interface Quote {
  id: string;
  number: string;
  status: "draft" | "sent" | "approved" | "invoiced";
  issueDate: string;
  dueDate: string;
  clientId: string;
  siteAddress: string;
  poNumber?: string;
  lineItems: LineItem[];
  subtotalLabor: number;
  subtotalMaterials: number;
  discount?: number;
  tax: number;
  total: number;
  signature?: string;
}

export interface Invoice extends Quote {
  status: "invoiced" | "partially_paid" | "paid" | "overdue";
  payments: { amount: number; date: string }[];
}
5. src/lib/store.ts


import { Company, Client, Quote, Invoice } from "./types";

const LS_KEY = "trade-invoice-app-data";

interface Store {
  company: Company;
  clients: Client[];
  quotes: Quote[];
  invoices: Invoice[];
}

const defaultStore: Store = {
  company: {
    id: "c1",
    name: "Apex Trade Services",
    address: "123 Industrial Way, Phoenix, AZ",
    licenseNumber: "ROC-123456",
    taxId: "12-3456789",
    paymentTerms: "Net 30",
    defaultHourlyRate: 95,
    taxRate: 0.08,
    taxMaterials: true,
    taxLabor: false,
    reminderEmailEnabled: true,
  },
  clients: [
    {
      id: "cl1",
      companyName: "Metro Property Management",
      primaryContact: "Jane Doe",
      billingEmail: "jane@metro.com",
      phone: "555-123-4567",
      poRequired: true,
      addresses: ["100 Main St, Suite 200"],
    },
  ],
  quotes: [],
  invoices: [
    {
      id: "inv1",
      number: "I-2001",
      status: "invoiced",
      issueDate: "2025-09-15",
      dueDate: "2025-10-15",
      clientId: "cl1",
      siteAddress: "100 Main St, Suite 200",
      poNumber: "PO-112",
      lineItems: [
        { id: "li1", type: "labor", description: "Panel upgrade", quantity: 6, unitPrice: 95 },
      ],
      subtotalLabor: 570,
      subtotalMaterials: 0,
      tax: 0,
      total: 570,
      payments: [],
    },
  ],
};

export function loadStore(): Store {
  if (typeof window === "undefined") return defaultStore;
  const raw = localStorage.getItem(LS_KEY);
  if (!raw) {
    localStorage.setItem(LS_KEY, JSON.stringify(defaultStore));
    return defaultStore;
  }
  return JSON.parse(raw);
}

export function saveStore(store: Store) {
  if (typeof window !== "undefined") {
    localStorage.setItem(LS_KEY, JSON.stringify(store));
  }
}

export function updateCompany(partial: Partial<Company>) {
  const store = loadStore();
  store.company = { ...store.company, ...partial };
  saveStore(store);
  return store.company;
}

export function addInvoice(invoice: Omit<Invoice, "id">) {
  const store = loadStore();
  const newInv: Invoice = { ...invoice, id: "inv_" + Date.now() };
  store.invoices.push(newInv);
  saveStore(store);
  return newInv;
}
6. src/lib/utils.ts


export function formatCurrency(amount: number): string {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
  }).format(amount);
}

export function formatDate(dateStr: string): string {
  return new Date(dateStr).toLocaleDateString("en-US", {
    month: "short",
    day: "numeric",
    year: "numeric",
  });
}
7. src/components/Navbar.tsx


import Link from "next/link";

export default function Navbar() {
  return (
    <nav className="sticky top-0 z-50 backdrop-blur-xl bg-[#0b0f19]/80 border-b border-cyan-500/10 px-6 py-4 flex justify-between items-center shadow-lg">
      <div className="flex items-center gap-8">
        <span className="font-bold tracking-wider text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-teal-300">
          ⚡ FIELDCORE
        </span>
        <div className="hidden md:flex gap-6 text-xs font-medium text-gray-400">
          <Link href="/" className="hover:text-cyan-400 transition">Dashboard</Link>
          <Link href="/invoices" className="hover:text-cyan-400 transition">Invoices</Link>
          <Link href="/settings" className="hover:text-cyan-400 transition">Settings</Link>
        </div>
      </div>
      <div className="flex items-center gap-2 px-3 py-1.5 rounded-full bg-cyan-950/40 border border-cyan-500/20 text-cyan-400 text-xs">
        <span className="w-2 h-2 rounded-full bg-teal-400 animate-pulse"></span>
        <span>Stripe Secure</span>
      </div>
    </nav>
  );
}
8. src/components/Layout.tsx


import Navbar from "./Navbar";

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-h-screen bg-[#0b0f19] text-gray-100">
      <Navbar />
      <main className="p-4 md:p-8 max-w-7xl mx-auto">{children}</main>
    </div>
  );
}
9. src/components/KPICard.tsx


export default function KPICard({ 
  title, 
  value, 
  subtitle 
}: { 
  title: string; 
  value: string; 
  subtitle?: string 
}) {
  return (
    <div className="cyber-sooth-card p-6 rounded-2xl relative overflow-hidden">
      <div className="text-xs font-semibold uppercase tracking-wider text-gray-400 mb-2">
        {title}
      </div>
      <div className="text-3xl font-extrabold tracking-tight text-white mb-2">
        {value}
      </div>
      {subtitle && (
        <div className="text-xs text-teal-400 font-medium">
          {subtitle}
        </div>
      )}
    </div>
  );
}
10. src/components/PaymentModal.tsx


import { useState } from "react";
import { loadStripe } from "@stripe/stripe-js";
import { Elements, CardElement, useStripe, useElements } from "@stripe/react-stripe-js";

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY || "pk_test_sample");

export default function PaymentModal({ 
  amount, 
  onSuccess, 
  onCancel 
}: { 
  amount: number; 
  onSuccess: () => void; 
  onCancel: () => void 
}) {
  return (
    <Elements stripe={stripePromise}>
      <CheckoutForm amount={amount} onSuccess={onSuccess} onCancel={onCancel} />
    </Elements>
  );
}

function CheckoutForm({ 
  amount, 
  onSuccess, 
  onCancel 
}: { 
  amount: number; 
  onSuccess: () => void; 
  onCancel: () => void 
}) {
  const stripe = useStripe();
  const elements = useElements();
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!stripe || !elements) return;
    
    setLoading(true);
    setTimeout(() => {
      setLoading(false);
      alert("Payment Successful!");
      onSuccess();
    }, 1500);
  };

  return (
    <div className="fixed inset-0 bg-black/70 backdrop-blur-sm flex items-center justify-center p-4 z-50">
      <div className="cyber-sooth-card p-6 rounded-2xl w-full max-w-md">
        <h2 className="text-xl font-bold mb-4 text-white">Secure Card Payment</h2>
        <p className="text-sm text-gray-400 mb-6">
          Amount Due: <span className="text-cyan-400 font-bold">${amount}</span>
        </p>
        
        <form onSubmit={handleSubmit} className="space-y-4">
          <div className="bg-gray-900/80 p-3 rounded-xl border border-cyan-500/20">
            <CardElement 
              options={{ 
                style: { 
                  base: { 
                    color: '#ffffff', 
                    fontSize: '16px',
                    '::placeholder': { color: '#6b7280' }
                  } 
                } 
              }} 
            />
          </div>

          <div className="flex gap-3 pt-4">
            <button 
              type="button" 
              onClick={onCancel} 
              className="flex-1 py-2 bg-gray-800 hover:bg-gray-700 rounded-xl font-medium"
            >
              Cancel
            </button>
            <button 
              type="submit" 
              disabled={!stripe || loading} 
              className="flex-1 py-2 bg-cyan-600 hover:bg-cyan-500 text-white rounded-xl font-bold disabled:opacity-50"
            >
              {loading ? "Processing..." : `Pay $${amount}`}
            </button>
          </div>
        </form>
      </div>
    </div>
  );
}
11. src/pages/api/create-payment-intent.ts


import type { NextApiRequest, NextApiResponse } from 'next';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY || 'sk_test_mock', {
  apiVersion: '2023-10-16',
});

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const { amount } = req.body;

  try {
    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(amount * 100),
      currency: 'usd',
    });

    res.status(200).json({ clientSecret: paymentIntent.client_secret });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
}
12. src/pages/index.tsx


import { useState, useEffect } from "react";
import { loadStore } from "../lib/store";
import Layout from "../components/Layout";
import KPICard from "../components/KPICard";
import { formatCurrency, formatDate } from "../lib/utils";
import PaymentModal from "../components/PaymentModal";

export default function Dashboard() {
  const [store, setStore] = useState(loadStore());
  const [payModal, setPayModal] = useState(false);
  const [activeInv, setActiveInv] = useState<any>(null);

  useEffect(() => {
    setStore(loadStore());
  }, []);

  const outstanding = store.invoices.reduce((sum, i) => sum + i.total, 0);

  return (
    <Layout>
      {payModal && activeInv && (
        <PaymentModal 
          amount={activeInv.total} 
          onSuccess={() => { setPayModal(false); window.location.reload(); }} 
          onCancel={() => setPayModal(false)} 
        />
      )}
      <div className="mb-8">
        <h1 className="text-3xl font-bold tracking-tight text-white">Dashboard</h1>
        <p className="text-sm text-gray-400 mt-1">Job site operations & secure payment tracking.</p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6 mb-8">
        <KPICard title="Total Outstanding" value={formatCurrency(outstanding)} subtitle="Awaiting remittance" />
        <KPICard title="Active Invoices" value={store.invoices.length.toString()} subtitle="Pending collection" />
        <KPICard title="Stripe Status" value="Connected" subtitle="Routing to secure holdings" />
      </div>

      <div className="cyber-sooth-card p-6 rounded-2xl">
        <h2 className="text-lg font-bold mb-4 text-white">Invoices</h2>
        <table className="w-full text-left">
          <thead>
            <tr className="border-b border-gray-800 text-gray-400 text-sm">
              <th className="pb-3">Invoice #</th>
              <th className="pb-3">Due Date</th>
              <th className="pb-3">Total</th>
              <th className="pb-3">Actions</th>
            </tr>
          </thead>
          <tbody>
            {store.invoices.map((inv) => (
              <tr key={inv.id} className="border-b border-gray-800/50">
                <td className="py-4 font-bold text-cyan-400">{inv.number}</td>
                <td className="py-4">{formatDate(inv.dueDate)}</td>
                <td className="py-4">{formatCurrency(inv.total)}</td>
                <td className="py-4">
                  <button 
                    onClick={() => { setActiveInv(inv); setPayModal(true); }} 
                    className="px-3 py-1 bg-cyan-600 hover:bg-cyan-500 text-white rounded-lg text-xs font-bold"
                  >
                    Pay Now
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </Layout>
  );
}
cohere/north-mini-code:free
· 13275 ms

export interface Company {
  id: string;
  name: string;
  address: string;
  licenseNumber?: string;
  taxId?: string;
  paymentTerms: "Net 15" | "Net 30";
  defaultHourlyRate: number;
  taxRate: number;
  taxMaterials: boolean;
  taxLabor: boolean;
  stripeApiKey?: string;
  reminderEmailEnabled: boolean;
}

export interface Client {
  id: string;
  companyName: string;
  primaryContact: string;
  billingEmail: string;
  phone: string;
  taxId?: string;
  poRequired: boolean;
  addresses: string[];
}

export interface LineItem {
  id: string;
  type: "labor" | "material" | "fee";
  description: string;
  quantity: number;
  unitPrice: number;
  markup?: number;
  export interface Quote {
  id: string;
  number: string;
  status: "draft" | "sent" | "approved" | "invoiced";
  issueDate: string;
  dueDate: string;
  clientId: string;
  siteAddress: string;
  poNumber?: string;
  lineItems: LineItem[];
  subtotalLabor: number;
  subtotalMaterials: number;
  discount?: number;
  tax: number;
  total: number;
  signature?: string;
}

export interface Invoice extends Quote {
  status: "invoiced" | "partially_paid" | "paid" | "overdue";
  payments: { amount: number; date: string }[];
}import { Company, Client, Quote, Invoice } from "./types";

const LS_KEY = "trade-invoice-app-data";

interface Store {
  company: Company;
  clients: Client[];
  quotes: Quote[];
  invoices: Invoice[];
}

const defaultStore: Store = {
  company: {
    id: "c1",
    name: "Apex Trade Services",
    address: "123 Industrial Way, Phoenix, AZ",
    licenseNumber: "ROC-123456",
    taxId: "12-3456789",
    paymentTerms: "Net 30",
    defaultHourlyRate: 95,
    taxRate: 0.08,
    taxMaterials: true,
    taxLabor: false,
    reminderEmailEnabled: true,
  },
  clients: [
    {id: "inv1",
      number: "I-2001",
      status: "invoiced",
      issueDate: "2025-09-15",
      dueDate: "2025-10-15",
      clientId: "cl1",
      siteAddress: "100 Main St, Suite 200",
      poNumber: "PO-112",
      lineItems: [
        { id: "li1", type: "labor", description: "Panel upgrade", quantity: 6, unitPrice: 95 },
      ],
      subtotalLabor: 570,
      subtotalMaterials: 0,
      tax: 0,
      total: 570,
      payments: [],
    },
  ],
};

export function loadStore(): Store {
  if (typeof window === "undefined") return defaultStore;
  const raw = localStorage.getItem(LS_KEY);
  if (!raw) {
    localStorage.setItem(LS_KEY, JSON.stringify(defaultStore));
    return defaultStore;
  }
  return JSON.parse(raw);export function saveStore(store: Store) {
  if (typeof window !== "undefined") {
    localStorage.setItem(LS_KEY, JSON.stringify(store));
  }
}

export function updateCompany(partial: Partial<Company>) {
  const store = loadStore();
  store.company = { ...store.company, ...partial };
  saveStore(store);
  return store.company;
}

export function addInvoice(invoice: Omit<Invoice, "id">) {
  const store = loadStore();
  const newInv: Invoice = { ...invoice, id: "inv_" + Date.now() };
  store.invoices.push(newInv);
  saveStore(store);
  return newInv;
export function formatCurrency(amount: number): string {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
  }).format(amount);
}

export function formatDate(dateStr: string): string {
  return new Date(dateStr).toLocaleDateString("en-US", {
    month: "short",
    day: "numeric",
    year: "numeric",
  });
}import Link from "next/link";

export default function Navbar() {
  return (
    <nav className="sticky top-0 z-50 backdrop-blur-xl bg-[#0b0f19]/80 border-b border-cyan-500/10 px-6 py-4 flex justify-between items-center shadow-lg">
      <div className="flex items-center gap-8">
        <span className="font-bold tracking-wider text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-teal-300">
          ⚡ FIELDCORE
        </span>
        <div className="hidden md:flex gap-6 text-xs font-medium text-gray-400">
          <Link href="/" className="hover:text-cyan-400 transition">Dashboard</Link>
          <Link href="/invoices" className="hover:text-cyan-400 transition">Invoices</Link>
          <Link href="/settings" className="hover:text-cyan-400 transition">Settings</Link>
        </div>
      </div>
      <div className="flex items-center gap-2 px-3 py-1.5 rounded-full bg-cyan-950/40 border border-cyan-500/20 text-cyan-400 text-xs">
        <span className="w-2 h-2 rounded-full bg-teal-400 animate-pulse"></span>
        <span>Stripe Secure</span>
      </div>
    </nav>
  );
}import Navbar from "./Navbar";

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-h-screen bg-[#0b0f19] text-gray-100">
      <Navbar />
      <main className="p-4 md:p-8 max-w-7xl mx-auto">{children}</main>
    </div>
  );
}export default function KPICard({ 
  title, 
  value, 
  subtitle 
}: { 
  title: string; 
  value: string; 
  subtitle?: string 
}) {
  return (
    <div className="cyber-sooth-card p-6 rounded-2xl relative overflow-hidden">
      <div className="text-xs font-semibold uppercase tracking-wider text-gray-400 mb-2">
        {title}
      </div>
      <div className="text-3xl font-extrabold tracking-tight text-white mb-2">
        {value}
      </div>
      {subtitle && (
        <div className="text-xs text-teal-400 font-medium">
          {subtitle}
        </div>
      )}
    </div>
  );
}import { useState } from "react";
import { loadStripe } from "@stripe/stripe-js";
import { Elements, CardElement, useStripe, useElements } from "@stripe/react-stripe-js";

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY || "pk_test_sample");

export default function PaymentModal({ 
  amount, 
  onSuccess, 
  onCancel 
}: { 
  amount: number; 
  onSuccess: () => void; 
  onCancel: () => void 
}) {
  return (
    <Elements stripe={stripePromise}>
      <CheckoutForm amount={amount} onSuccess={onSuccess} onCancel={onCancel} />
    </Elements>
  );
}
}function CheckoutForm({ 
  amount, 
  onSuccess, 
  onCancel 
}: { 
  amount: number; 
  onSuccess: () => void; 
  onCancel: () => void 
}) {
  const stripe = useStripe();
  const elements = useElements();
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!stripe || !elements) return;
    
    setLoading(true);
    setTimeout(() => {
      setLoading(false);
      alert("Payment Successful!");
      onSuccess();
    }, 1500);
  };return (
    <div className="fixed inset-0 bg-black/70 backdrop-blur-sm flex items-center justify-center p-4 z-50">
      <div className="cyber-sooth-card p-6 rounded-2xl w-full max-w-md">
        <h2 className="text-xl font-bold mb-4 text-white">Secure Card Payment</h2>
        <p className="text-sm text-gray-400 mb-6">
          Amount Due: <span className="text-cyan-400 font-bold">${amount}</span>
        </p>
        
        <form onSubmit={handleSubmit} className="space-y-4">
          <div className="bg-gray-900/80 p-3 rounded-xl border border-cyan-500/20">
            <CardElement 
              options={{ 
                style: { 
                  base: { 
                    color: '#ffffff', 
                    fontSize: '16px',
                    '::placeholder': { color: '#6b7280' }
                  } 
                } 
              }} 
            />
          </div>

          <div className="flex gap-3 pt-4">
            <button 
              type="button" 
              onClick={onCancel} 
              className="flex-1 py-2 bg-gray-800 hover:bg-gray-700 rounded-xl font-medium"
            >
              Cancel
            </button>
            <button 
              type="submit" 
              disabled={!stripe || loading} 
              className="flex-1 py-2 bg-cyan-600 hover:bg-cyan-500 text-white rounded-xl font-bold disabled:opacity-50"
            >
              {loading ? "Processing..." : `Pay $${amount}`}
            </button>
          </div>
        </form>
      </div>
      </div>
      import type { NextApiRequest, NextApiResponse } from 'next';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY || 'sk_test_mock', {
  apiVersion: '2023-10-16',
});

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const { amount } = req.body;

  try {
    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(amount * 100),
      currency: 'usd',
    });

    res.status(200).json({ clientSecret: paymentIntent.client_secret });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
}
12. src/pages/index.tsx


import { useState, useEffect } from "react";
import { loadStore } from "../lib/store";
import Layout from "../components/Layout";
import KPICard from "../components/KPICard";
import { formatCurrency, formatDate } from "../lib/utils";
import PaymentModal from "../components/PaymentModal";

export default function Dashboard() {
  const [store, setStore] = useState(loadStore());
  const [payModal, setPayModal] = useState(false);
  const [activeInv, setActiveInv] = useState<any>(null);

  useEffect(() => {
    setStore(loadStore());
  }, []);

  const outstanding = store.invoices.reduce((sum, i) => sum + i.total, 0);

  return (
    <Layout>
      {payModal && activeInv && (
        <PaymentModal 
          amount={activeInv.total} 
          onSuccess={() => { setPayModal(false); window.location.reload(); }} 
          onCancel={() => setPayModal(false)} 
        />
      )}
      <div className="mb-8">
        <h1 className="text-3xl font-bold tracking-tight text-white">Dashboard</h1>
        <p className="text-sm text-gray-400 mt-1">Job site operations & secure payment tracking.</p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6 mb-8">
        <KPICard title="Total Outstanding" value={formatCurrency(outstanding)} subtitle="Awaiting remittance" />
        <KPICard title="Active Invoices" value={store.invoices.length.toString()} subtitle="Pending collection" />
        <KPICard title="Stripe Status" value="Connected" subtitle="Routing to secure holdings" />
      </div>

      <div className="cyber-sooth-card p-6 rounded-2xl">
        <h2 className="text-lg font-bold mb-4 text-white">Invoices</h2>
        <table className="w-full text-left">
          <thead>
            <tr className="border-b border-gray-800 text-gray-400 text-sm">
              <th className="pb-3">Invoice #</th>
              <th className="pb-3">Due Date</th>
              <th className="pb-3">Total</th>
              <th className="pb-3">Actions</th>
            </tr>
          </thead>
          <tbody>
            {store.invoices.map((inv) => (
              <tr key={inv.id} className="border-b border-gray-800/50">
                <td className="py-4 font-bold text-cyan-400">{inv.number}</td>
                <td className="py-4">{formatDate(inv.dueDate)}</td>
                <td className="py-4">{formatCurrency(inv.total)}</td>
                <td className="py-4">
                  <button 
                    onClick={() => { setActiveInv(inv); setPayModal(true); }} 
                    className="px-3 py-1 bg-cyan-600 hover:bg-cyan-500 text-white rounded-lg text-xs font-bold"
                  >
                    Pay Now
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </Layout>
  );
}
cohere/north-mini-code:free
· 13275 ms

Type a message… (⏎ to send, ⇧⏎ for newline)
import { useState, useEffect } from "react";
import { loadStore } from "../lib/store";
import Layout from "../components/Layout";
import KPICard from "../components/KPICard";
import { formatCurrency, formatDate } from "../lib/utils";
import PaymentModal from "../components/PaymentModal";

export default function Dashboard() {
  const [store, setStore] = useState(loadStore());
  const [payModal, setPayModal] = useState(false);
  const [activeInv, setActiveInv] = useState<any>(null);

  useEffect(() => {
    setStore(loadStore());
  }, []);

  const outstanding = store.invoices.reduce((sum, i) => sum + i.total, 0);

  return (
    <Layout>
      {payModal && activeInv && (
        <PaymentModal 
          amount={activeInv.total} 
          onSuccess={() => { setPayModal(false); window.location.reload(); }} 
          onCancel={() => setPayModal(false)} 
        />
      )}<div className="mb-8">
        <h1 className="text-3xl font-bold tracking-tight text-white">Dashboard</h1>
        <p className="text-sm text-gray-400 mt-1">Job site operations & secure payment tracking.</p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6 mb-8">
        <KPICard title="Total Outstanding" value={formatCurrency(outstanding)} subtitle="Awaiting remittance" />
        <KPICard title="Active Invoices" value={store.invoices.length.toString()} subtitle="Pending collection" />
        <KPICard title="Stripe Status" value="Connected" subtitle="Routing to secure holdings" />
      </div>

      <div className="cyber-sooth-card p-6 rounded-2xl">
        <h2 className="text-lg font-bold mb-4 text-white">Invoices</h2>
        <table className="w-full text-left">
          <thead>
            <tr className="border-b border-gray-800 text-gray-400 text-sm">
              <th className="pb-3">Invoice #</th>
              <th className="pb-3">Due Date</th>
              <th className="pb-3">Total</th>
              <th className="pb-3">Actions</th>
            </tr>
          </thead>
          <tbody>
            {store.invoices.map((inv) => (
              <tr key={inv.id} className="border-b border-gray-800/50">
                <td className="py-4 font-bold text-cyan-400">{inv.number}</td>
                <td className="py-4">{formatDate(inv.dueDate)}</td>
                <td className="py-4">{formatCurrency(inv.total)}</td>
                <td className="py-4">
                  <button 
                    onClick={() => { setActiveInv(inv); setPayModal(true); }} 
                    className="px-3 py-1 bg-cyan-600 hover:bg-cyan-500 text-white rounded-lg text-xs font-bold"
                  >
                  Pay Now
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </Layout>
  );
}
