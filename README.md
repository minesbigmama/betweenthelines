# betweenthelines
where what is said becomes what's heard
import React, { useEffect, useMemo, useState } from "react";

const APP_NAME = "BetweenTheLines";
const TAGLINE = "Where what’s said becomes what’s understood.";

const LS_PLAN = "betweenthelines_plan";
const LS_USAGE = "betweenthelines_usage_count";
const LS_DATE  = "betweenthelines_usage_date";

const PRESETS = [
  { id: "task", label: "Task-focused", description: "Brief, direct, low emotional language, solution/next-steps oriented.", directness: 85, validation: 25, detail: 35, solution: 80 },
  { id: "connect", label: "Connection-focused", description: "Relational, validating, collaborative framing with feelings/context.", directness: 45, validation: 85, detail: 70, solution: 55 },
  { id: "pro", label: "Neutral Professional", description: "Concise, clear expectations, neutral tone, action items & timelines.", directness: 75, validation: 40, detail: 55, solution: 75 },
  { id: "support", label: "Supportive Listener", description: "High validation & reflective listening; few directives.", directness: 35, validation: 95, detail: 60, solution: 35 },
  { id: "blunt", label: "Blunt / No-frills", description: "Very direct, minimal padding, strong ask/boundary statements.", directness: 95, validation: 15, detail: 25, solution: 70 },
];

const HEDGES = ["maybe","perhaps","sort of","kind of","a bit","just","I think","I feel like","it seems","possibly","might","could","I guess"];
const SOFTENERS = ["maybe","could we","would you be open to","would it help if","I wonder if","when you have a moment","if it's okay","I feel","I'm thinking"];
const POLITE_PREFIXES = ["please","could you","would you","can you","let's"];

function normalizeWhitespace(s){ return s.replace(/\s+/g," ").replace(/\s([?.!,;:])/g,(_m,p1)=>p1).trim(); }
function removeHedges(s){ let out=s; HEDGES.forEach(h=>{ const re=new RegExp("\\b"+h.replace(/[-/\\^$*+?.()|[\]{}]/g,"\\$&")+"\\b","gi"); out=out.replace(re,""); }); return normalizeWhitespace(out); }
function addSofteners(s){ const lower=s.toLowerCase(); const has=POLITE_PREFIXES.some(p=>lower.includes(p)); if(has) return s; const soft=SOFTENERS[Math.floor(Math.random()*SOFTENERS.length)]; const first=s.charAt(0); const adjusted= first===first.toUpperCase()? first.toLowerCase()+s.slice(1): s; return soft+" "+adjusted; }
function toBullets(text){ return text.replace(/([.!?])\s+/g,(_m,p1)=>p1+"|").split("|").map(p=>p.trim()).filter(Boolean); }
function ensurePleaseForRequests(s){ const ask=/\?$/.test(s)||/\b(can|could|would|please|let's)\b/i.test(s); if(!ask) return s; if(/\bplease\b/i.test(s)) return s; return s.replace(/^(.*?)([a-z])/i,(_m,pre,first)=>pre+"please "+first); }
function clarifyLine(intent){ switch(intent){ case "vent": return "Would you like empathy right now, solutions, or both?"; case "request": return "What outcome and timeframe would work best for you?"; case "boundary": return "What boundary do you want to set, and what consequence if it's crossed?"; case "apology": return "Is there a specific repair you'd like to propose?"; default: return "Is there anything you'd like me to clarify or expand?"; } }
function classifyIntent(s){ const t=s.toLowerCase(); if (/(sorry|apologize|my fault)/.test(t)) return "apology"; if (/(thank|appreciate|grateful)/.test(t)) return "appreciation"; if (/(i can't|i cannot|i won't|not okay|stop\s+that|this needs to stop)/.test(t)) return "boundary"; if (/(can you|could you|would you|please|\?)/.test(t)) return "request"; if (/(always|never|so tired|i'm upset|i'm mad|this sucks|ugh|!)/.test(t)) return "vent"; return "inform"; }

function transform(input, from, to){
  const notes=[]; const extras=[]; let text=normalizeWhitespace(input); const intent=classifyIntent(text);
  if (to.directness > from.directness + 15){ const before=text; text=removeHedges(text); if(before!==text) notes.push("Removed hedging for more direct phrasing."); }
  else if (to.directness + 15 < from.directness){ const before=text; text=addSofteners(text); if(before!==text) notes.push("Added softeners to reduce perceived bluntness."); }
  if (to.validation > from.validation + 20){ let preface="I hear you."; if (intent==="vent") preface="I can see this is really frustrating."; if (intent==="boundary") preface="I want to be clear and respectful here."; if (intent==="request") preface="I appreciate your help with this."; text=preface+" "+text; notes.push("Added a brief validation preface."); }
  if (to.solution > from.solution + 20){ if (intent==="vent"){ extras.push("Possible next step: Would you like to brainstorm a couple options together?"); } else if (intent==="request"){ if (!/\b(today|tomorrow|tonight|\d\d?:\d\d|\bby\b)/i.test(text)){ text=text+" (what timeline works?)"; notes.push("Prompted for a timeline to make the request actionable."); } } }
  if (to.detail > from.detail + 20){ if(!/\bbecause\b/i.test(text)){ text=text+" — because it would help me/us avoid stress."; notes.push("Added a light rationale to match a more contextual style."); } }
  else if (to.detail + 20 < from.detail){ text=text.replace(/\b(in order to|basically|actually|literally|really)\b/gi,""); notes.push("Trimmed filler words for brevity."); }
  if (intent==="request" && to.validation>=40){ const before=text; text=ensurePleaseForRequests(text); if(before!==text) notes.push("Added polite marker to the request."); }
  if (to.detail>=75 && /[.!?]/.test(text)){ const bullets=toBullets(text); if (bullets.length>=2){ text=bullets.map(b=>"• "+b).join("\n"); notes.push("Structured into bullet points for readability."); } }
  const intentPrompt=clarifyLine(intent); if(!extras.includes(intentPrompt)) extras.push(intentPrompt);
  return { translation:text, notes, extras };
}
function applyTone(text, tone){ if(tone==="warm") return "😊 "+text+" Thanks for understanding."; if(tone==="cool") return "[Neutral/Detached] "+text; return text; }
function clamp(n,min=0,max=100){ return Math.max(min, Math.min(max, Math.round(n))); }
function transformWithVariants(input, from, to, tone){
  const base=transform(input,from,to);
  const soft=transform(input,from,{...to, directness:clamp(to.directness-20), validation:clamp(to.validation+20)}).translation;
  const neutral=transform(input,from,{...to}).translation;
  const direct=transform(input,from,{...to, directness:clamp(to.directness+20), validation:clamp(to.validation-15)}).translation;
  return { translation: applyTone(base.translation,tone), variants:[applyTone(soft,tone), applyTone(neutral,tone), applyTone(direct,tone)], notes: base.notes };
}

function todayISO(){ return new Date().toISOString().slice(0,10); }
function loadPlan(){ return localStorage.getItem(LS_PLAN) || "free"; }
function savePlan(p){ localStorage.setItem(LS_PLAN, p); }
function loadUsage(){ const date=localStorage.getItem(LS_DATE)||todayISO(); const raw=localStorage.getItem(LS_USAGE); const count= raw? parseInt(raw,10): 0; return {count,date}; }
function incUsage(){ const d=todayISO(); const {count,date}=loadUsage(); const next= date===d? count+1: 1; localStorage.setItem(LS_DATE,d); localStorage.setItem(LS_USAGE,String(next)); return next; }
function resetUsageIfNewDay(){ const d=todayISO(); const {date}=loadUsage(); if (date!==d){ localStorage.setItem(LS_DATE,d); localStorage.setItem(LS_USAGE,"0"); } }

function Card({title,children}){ return (<div className="bg-white rounded-2xl shadow p-4"><h2 className="text-lg font-semibold mb-2">{title}</h2>{children}</div>); }
function UpgradeNotice({onOpenPricing}){
  return (<div className="bg-amber-50 border border-amber-200 text-amber-800 p-3 rounded-xl text-sm flex items-center justify-between">
    <span>You’ve hit the Free plan limit (3/day). Upgrade for unlimited and extra features.</span>
    <button onClick={onOpenPricing} className="ml-3 px-3 py-1 rounded-lg bg-amber-200 hover:bg-amber-300">View Pricing</button>
  </div>);
}

function PricingTable({startCheckout}){
  return (
    <div className="grid md:grid-cols-3 gap-4">
      <PriceCol name="Free" price="$0" cta="Stay on Free" features={["3 translations/day","Basic presets","Copy to clipboard"]} onClick={()=>{}} />
      <PriceCol name="Pro" price="$4.99/mo" sub="or $19.99/yr" cta="Upgrade to Pro" features={["Unlimited translations","All presets","Tone toggles","Coaching mode","Save history"]} onClick={()=>{
        const choice = window.prompt("Type 'm' for $4.99/mo or 'y' for $19.99/yr","m");
        startCheckout(choice==="y" ? "pro_yearly" : "pro_monthly");
      }} highlight />
      <PriceCol name="Lifetime" price="$29.99 one-time" cta="Buy Lifetime" features={["Everything in Pro","Lifetime access","Founders badge"]} onClick={()=>startCheckout("lifetime")} />
    </div>
  );
}
function PriceCol({name,price,sub,cta,features,onClick,highlight=false}){
  return (<div className={`rounded-2xl border ${highlight? "border-indigo-500 shadow-lg":"border-slate-200 shadow"} bg-white p-5 flex flex-col`}>
    <div className="text-xl font-semibold">{name}</div>
    <div className="text-3xl font-bold mt-2">{price}</div>
    {sub && <div className="text-slate-500 text-sm">{sub}</div>}
    <ul className="mt-4 space-y-2 text-sm text-slate-700">{features.map((f,i)=>(<li key={i}>• {f}</li>))}</ul>
    <button onClick={onClick} className={`mt-4 px-4 py-2 rounded-xl ${highlight? "bg-indigo-600 text-white":"bg-slate-100 text-slate-800"}`}>{cta}</button>
  </div>);
}

export default function App(){
  const [input,setInput]=useState("We need to leave now.");
  const [fromId,setFromId]=useState(PRESETS[0].id);
  const [toId,setToId]=useState(PRESETS[1].id);
  const [tone,setTone]=useState("neutral");
  const [custom,setCustom]=useState(null);
  const [result,setResult]=useState(null);

  const [plan,setPlan]=useState(loadPlan());
  const [usageToday,setUsageToday]=useState(0);
  const [showPricing,setShowPricing]=useState(false);
  const [showLimitNotice,setShowLimitNotice]=useState(false);

  useEffect(()=>{
    const params=new URLSearchParams(window.location.search);
    const planParam=params.get("plan");
    if (planParam==="pro" || planParam==="lifetime"){
      savePlan(planParam);
      setPlan(planParam);
      window.history.replaceState({}, document.title, window.location.pathname);
    }
  },[]);

  useEffect(()=>{
    resetUsageIfNewDay();
    setUsageToday(loadUsage().count);
  },[]);

  const from=useMemo(()=> PRESETS.find(p=>p.id===fromId), [fromId]);
  const baseTo=useMemo(()=> PRESETS.find(p=>p.id===toId), [toId]);
  const to= custom || baseTo;
  const canTranslate=()=> plan!=="free" || usageToday<3;

  function handleTranslate(){
    if(!input.trim()) return;
    if(!canTranslate()){ setShowLimitNotice(true); setShowPricing(true); return; }
    const out=transformWithVariants(input, from, to, tone);
    setResult(out);
    if(plan==="free") setUsageToday(incUsage());
  }

  async function startCheckout(kind){
    try{
      const res = await fetch("/api/create-checkout-session", { method:"POST", headers:{ "Content-Type":"application/json" }, body: JSON.stringify({ kind }) });
      const data = await res.json();
      if (data?.url){ window.location.href = data.url; } else { alert("Could not start checkout: " + (data?.error || "")); }
    }catch(e){ alert("Checkout error: " + e.message); }
  }

  function copyText(text){ navigator.clipboard.writeText(text); }
  function downloadText(text,filename="betweenthelines.txt"){
    const blob=new Blob([text],{type:"text/plain;charset=utf-8"});
    const url=URL.createObjectURL(blob);
    const a=document.createElement("a"); a.href=url; a.download=filename; a.click(); URL.revokeObjectURL(url);
  }

  return (
    <div className="min-h-screen text-slate-900 p-6">
      <div className="max-w-5xl mx-auto">
        <header className="mb-6 flex flex-col gap-2">
          <h1 className="text-3xl font-bold">{APP_NAME}</h1>
          <p className="text-slate-600">{TAGLINE}</p>
          <p className="text-slate-600">Translate messages between communication styles while preserving meaning and adding the right level of validation, clarity, and structure. Not a mind reader—always double-check and ask questions.</p>
          <div className="text-xs text-slate-500">Plan: <span className="font-medium uppercase">{plan}</span> • Usage today: {usageToday}/3</div>
          {showLimitNotice && <UpgradeNotice onOpenPricing={()=>setShowPricing(true)} />}
        </header>

        <section className="grid md:grid-cols-2 gap-4 mb-4">
          <div className="bg-white rounded-2xl shadow p-4">
            <label className="block text-sm font-medium mb-1">What was said</label>
            <textarea value={input} onChange={e=>setInput(e.target.value)} className="w-full h-40 rounded-xl border border-slate-200 p-3 focus:outline-none focus:ring-2 focus:ring-indigo-400" placeholder="Type or paste the exact message here" />
          </div>
          <div className="bg-white rounded-2xl shadow p-4 flex flex-col gap-3">
            <div className="grid grid-cols-2 gap-3">
              <div>
                <label className="block text-sm font-medium mb-1">From style</label>
                <select value={fromId} onChange={e=>setFromId(e.target.value)} className="w-full rounded-xl border border-slate-200 p-2">
                  {PRESETS.map(p=>(<option key={p.id} value={p.id}>{p.label}</option>))}
                </select>
                <p className="text-xs text-slate-500 mt-1">{PRESETS.find(p=>p.id===fromId)?.description}</p>
              </div>
              <div>
                <label className="block text-sm font-medium mb-1">To style</label>
                <select value={toId} onChange={e=>setToId(e.target.value)} className="w-full rounded-xl border border-slate-200 p-2">
                  {PRESETS.map(p=>(<option key={p.id} value={p.id}>{p.label}</option>))}
                </select>
                <p className="text-xs text-slate-500 mt-1">{PRESETS.find(p=>p.id===toId)?.description}</p>
              </div>
            </div>

            <div className="mt-2">
              <label className="block text-sm font-medium mb-1">Tone</label>
              <div className="flex gap-4 text-sm">
                {["warm","neutral","cool"].map(t=>(
                  <label key={t} className="inline-flex items-center gap-2">
                    <input type="radio" value={t} checked={tone===t} onChange={()=>setTone(t)} />
                    {t}
                  </label>
                ))}
              </div>
            </div>

            <div className="mt-2">
              <label className="block text-sm font-medium mb-1">Fine-tune target</label>
              {["directness","validation","detail","solution"].map(f=>(
                <div key={f} className="mb-2">
                  <label className="block text-xs text-slate-500">{f}</label>
                  <input type="range" min={0} max={100} onChange={e=>{
                    const val=parseInt(e.target.value,10);
                    const baseTo=PRESETS.find(p=>p.id===toId);
                    setCustom({
                      ...(custom || {...baseTo, id:"custom", label:"Custom", description:"User tuned"}),
                      [f]: val,
                    });
                  }} />
                </div>
              ))}
            </div>

            <div className="flex items-center gap-2 mt-2">
              <button onClick={handleTranslate} className="px-4 py-2 rounded-xl bg-indigo-600 text-white shadow hover:bg-indigo-700">Translate</button>
              <button onClick={()=>setShowPricing(true)} className="px-4 py-2 rounded-xl bg-slate-100 text-slate-800 border border-slate-200 hover:bg-slate-200">View Pricing</button>
              <button onClick={()=>startCheckout("addon_coaching")} className="px-4 py-2 rounded-xl bg-emerald-600 text-white shadow hover:bg-emerald-700">Add-on: Coaching+</button>
            </div>
          </div>
        </section>

        {result && (
          <section className="grid md:grid-cols-2 gap-4">
            <Card title="Primary Translation">
              <pre className="whitespace-pre-wrap bg-slate-50 rounded-xl p-3 border border-slate-200 min-h-[120px]">{result.translation}</pre>
              <div className="mt-2 flex gap-2">
                <button onClick={()=>copyText(result.translation)} className="px-3 py-2 rounded-xl bg-slate-100 border border-slate-200 hover:bg-slate-200">Copy</button>
                <button onClick={()=>downloadText(result.translation)} className="px-3 py-2 rounded-xl bg-slate-100 border border-slate-200 hover:bg-slate-200">Download</button>
              </div>
            </Card>
            <Card title="Coaching Variants (Soft / Neutral / Direct)">
              {result.variants.map((v,i)=>(
                <div key={i} className="mb-3">
                  <div className="font-medium text-sm text-slate-600">Option {i+1}</div>
                  <pre className="whitespace-pre-wrap bg-slate-50 rounded-xl p-3 border border-slate-200">{v}</pre>
                  <div className="mt-2 flex gap-2">
                    <button onClick={()=>copyText(v)} className="px-3 py-2 rounded-xl bg-slate-100 border border-slate-200 hover:bg-slate-200">Copy</button>
                    <button onClick={()=>downloadText(v, `betweenthelines-variant-${i+1}.txt`)} className="px-3 py-2 rounded-xl bg-slate-100 border border-slate-200 hover:bg-slate-200">Download</button>
                  </div>
                </div>
              ))}
              {result.notes?.length>0 && (
                <div className="pt-2 border-t border-slate-200 mt-2">
                  <h3 className="font-medium text-sm mb-1">Why it changed</h3>
                  <ul className="list-disc pl-5 text-sm">{result.notes.map((n,i)=>(<li key={i}>{n}</li>))}</ul>
                </div>
              )}
            </Card>
          </section>
        )}

        {showPricing && (
          <section className="mt-6 bg-white rounded-2xl shadow p-4">
            <h2 className="text-lg font-semibold mb-2">Pricing</h2>
            <p className="text-slate-600 mb-3">{APP_NAME} — {TAGLINE}</p>
            <PricingTable startCheckout={startCheckout} />
            <p className="text-xs text-slate-500 mt-3">Payments powered by Stripe Checkout. After payment you’ll return here and unlock automatically on this device.</p>
          </section>
        )}

        <footer className="mt-8 text-xs text-slate-500 space-y-1">
          <p>{APP_NAME} — {TAGLINE}</p>
          <p>Built as a heuristic MVP. Communication is complex—use translations as a starting point, not a verdict.</p>
          <p>Deploy free on Vercel/Netlify. Add Firebase/Supabase later for account sync.</p>
        </footer>
      </div>
    </div>
  );
}
