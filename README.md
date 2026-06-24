<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>HealthSphere AI Platform</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.development.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.development.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.2/babel.min.js"></script>
<style>
  html, body, #root { margin: 0; padding: 0; height: 100%; }
</style>
</head>
<body>
<div id="root"></div>
<script type="text/babel">
const { useState, useEffect, useRef, useCallback } = React;


// ─── CONSTANTS ───────────────────────────────────────────────────────────────
const DISEASES_DB = [
  { id: "viral_fever", name: "Viral Fever", symptoms: ["fever", "headache", "fatigue", "body aches", "chills"], severity: "moderate" },
  { id: "dengue", name: "Dengue", symptoms: ["high fever", "rash", "joint pain", "eye pain", "nausea"], severity: "high" },
  { id: "influenza", name: "Influenza", symptoms: ["fever", "cough", "sore throat", "runny nose", "fatigue"], severity: "moderate" },
  { id: "typhoid", name: "Typhoid", symptoms: ["sustained fever", "abdominal pain", "weakness", "headache", "diarrhea"], severity: "high" },
  { id: "malaria", name: "Malaria", symptoms: ["cyclical fever", "chills", "sweating", "headache", "nausea"], severity: "high" },
  { id: "diabetes", name: "Type 2 Diabetes", symptoms: ["frequent urination", "excessive thirst", "fatigue", "blurred vision", "slow healing"], severity: "chronic" },
  { id: "hypertension", name: "Hypertension", symptoms: ["headache", "dizziness", "chest pain", "shortness of breath", "nosebleed"], severity: "moderate" },
  { id: "covid19", name: "COVID-19", symptoms: ["fever", "cough", "loss of smell", "loss of taste", "fatigue", "shortness of breath"], severity: "moderate-high" },
];

const MOCK_USERS = [
  { id: 1, name: "Arjun Sharma", email: "arjun@example.com", age: 28, gender: "Male", bloodGroup: "B+", role: "user", predictions: 12, healthScore: 78 },
  { id: 2, name: "Priya Patel", email: "priya@example.com", age: 34, gender: "Female", bloodGroup: "A+", role: "user", predictions: 8, healthScore: 85 },
  { id: 3, name: "Rahul Verma", email: "rahul@example.com", age: 45, gender: "Male", bloodGroup: "O+", role: "user", predictions: 23, healthScore: 62 },
  { id: 4, name: "Sneha Gupta", email: "sneha@example.com", age: 22, gender: "Female", bloodGroup: "AB+", role: "user", predictions: 5, healthScore: 91 },
];

const MOCK_HOSPITALS = [
  { id: 1, name: "Victoria Hospital (NSCB Medical College)", type: "Government", distance: "3.2 km", rating: 4.3, emergency: true, phone: "0761-2622580", area: "Medical College Road, Garha", lat: 23.1728, lng: 79.9358 },
  { id: 2, name: "Madan Mahal Government Hospital", type: "Government", distance: "2.1 km", rating: 4.0, emergency: true, phone: "0761-2490330", area: "Madan Mahal, Jabalpur", lat: 23.1673, lng: 79.9378 },
  { id: 3, name: "Balaji Hospital", type: "Private", distance: "1.6 km", rating: 4.6, emergency: true, phone: "0761-4050000", area: "Napier Town, Jabalpur", lat: 23.1693, lng: 79.9331 },
  { id: 4, name: "Nazareth Hospital", type: "Private", distance: "2.8 km", rating: 4.5, emergency: true, phone: "0761-2622121", area: "Russel Chowk, Jabalpur", lat: 23.1601, lng: 79.9322 },
  { id: 5, name: "Rani Durgavati Hospital", type: "Government", distance: "4.1 km", rating: 4.1, emergency: true, phone: "0761-2490400", area: "Vijay Nagar, Jabalpur", lat: 23.1750, lng: 79.9545 },
  { id: 6, name: "Shri Ram Hospital", type: "Private", distance: "3.5 km", rating: 4.4, emergency: true, phone: "0761-4033333", area: "Gorakhpur, Jabalpur", lat: 23.1812, lng: 79.9402 },
];

const COMMUNITY_ALERTS = [
  { id: 1, disease: "Dengue", area: "Gorakhpur", cases: 31, severity: "high", date: "2026-06-05", trend: "rising" },
  { id: 2, disease: "Viral Fever", area: "Napier Town", cases: 58, severity: "moderate", date: "2026-06-06", trend: "stable" },
  { id: 3, disease: "Cholera", area: "Garha", cases: 11, severity: "high", date: "2026-06-07", trend: "rising" },
  { id: 4, disease: "Typhoid", area: "Adhartal", cases: 19, severity: "moderate", date: "2026-06-04", trend: "declining" },
];

// ─── CLAUDE API ───────────────────────────────────────────────────────────────
async function callClaude(messages, systemPrompt, maxTokens = 1000) {
  const response = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "claude-sonnet-4-6",
      max_tokens: maxTokens,
      system: systemPrompt,
      messages,
    }),
  });
  const data = await response.json();
  if (data.error) throw new Error(data.error.message);
  return data.content.map(b => b.text || "").join("\n").trim();
}

async function callClaudeJSON(messages, systemPrompt) {
  const text = await callClaude(messages, systemPrompt + "\n\nRespond ONLY with valid JSON. No markdown, no backticks, no explanation before or after the JSON.");
  try {
    // Try to extract JSON object from the response even if there's extra text
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (jsonMatch) return JSON.parse(jsonMatch[0]);
    return JSON.parse(text.trim());
  } catch {
    return null;
  }
}

// ─── ICONS ───────────────────────────────────────────────────────────────────
const icons = {
  dashboard: "⊞",
  users: "👥",
  disease: "🧬",
  reports: "📋",
  hospitals: "🏥",
  community: "📡",
  analytics: "📊",
  settings: "⚙️",
  chat: "🤖",
  prediction: "🔬",
  prevention: "🛡",
  health: "❤️",
  logout: "⎋",
  alert: "🚨",
  admin: "👑",
  upload: "📤",
  search: "🔍",
  refresh: "↻",
  check: "✓",
  cross: "✕",
  arrow: "→",
  star: "★",
  bell: "🔔",
  lock: "🔒",
  eye: "👁",
  send: "➤",
  download: "⬇",
  plus: "＋",
  edit: "✏",
  delete: "🗑",
  map: "📍",
  phone: "📞",
  calendar: "📅",
};

// ─── STYLES ──────────────────────────────────────────────────────────────────
const styles = `
  @import url('https://fonts.googleapis.com/css2?family=Sora:wght@300;400;500;600;700;800&family=DM+Mono:wght@400;500&display=swap');
  
  :root {
    --bg: #070b12;
    --bg2: #0c1220;
    --bg3: #111827;
    --card: #0f1624;
    --card2: #141e2e;
    --border: rgba(99,140,255,0.1);
    --border2: rgba(99,140,255,0.22);
    --accent: #4f7cff;
    --accent2: #8b5cf6;
    --accent3: #06b6d4;
    --accent4: #10b981;
    --accent5: #f59e0b;
    --accent6: #ef4444;
    --accent7: #ec4899;
    --text1: #e8eeff;
    --text2: #7888aa;
    --text3: #3a4a66;
    --glow: rgba(79,124,255,0.2);
    --sans: 'Sora', sans-serif;
    --mono: 'DM Mono', monospace;
    --r: 12px;
    --r2: 8px;
  }
  
  * { box-sizing: border-box; margin: 0; padding: 0; }
  
  body { font-family: var(--sans); background: var(--bg); color: var(--text1); height: 100vh; overflow: hidden; }
  
  .app { display: flex; height: 100vh; overflow: hidden; }
  
  /* SIDEBAR */
  .sidebar {
    width: 220px; flex-shrink: 0; background: var(--bg2);
    border-right: 1px solid var(--border); display: flex; flex-direction: column;
    overflow-y: auto; overflow-x: hidden;
  }
  .sidebar::-webkit-scrollbar { width: 3px; }
  .sidebar::-webkit-scrollbar-track { background: transparent; }
  .sidebar::-webkit-scrollbar-thumb { background: var(--border2); border-radius: 4px; }
  
  .logo { padding: 20px 16px; border-bottom: 1px solid var(--border); display: flex; align-items: center; gap: 10px; }
  .logo-icon { width: 34px; height: 34px; background: linear-gradient(135deg, var(--accent), var(--accent2)); border-radius: 9px; display: flex; align-items: center; justify-content: center; font-size: 16px; flex-shrink: 0; }
  .logo-text { font-size: 14px; font-weight: 700; line-height: 1.1; }
  .logo-sub { font-size: 9px; color: var(--text3); letter-spacing: 0.1em; text-transform: uppercase; }
  
  .nav-section { padding: 12px 0; }
  .nav-label { font-size: 9px; color: var(--text3); letter-spacing: 0.12em; text-transform: uppercase; padding: 0 16px; margin-bottom: 4px; }
  .nav-item { display: flex; align-items: center; gap: 9px; padding: 8px 16px; cursor: pointer; font-size: 12.5px; color: var(--text2); transition: all 0.15s; border-left: 2px solid transparent; }
  .nav-item:hover { color: var(--text1); background: rgba(79,124,255,0.06); }
  .nav-item.active { color: var(--accent); background: rgba(79,124,255,0.1); border-left-color: var(--accent); }
  .nav-item span.icon { font-size: 14px; }
  
  .sidebar-bottom { margin-top: auto; padding: 12px; border-top: 1px solid var(--border); }
  .user-pill { display: flex; align-items: center; gap: 9px; padding: 8px 10px; background: var(--card); border-radius: var(--r2); cursor: pointer; }
  .user-avatar { width: 28px; height: 28px; border-radius: 50%; background: linear-gradient(135deg, var(--accent), var(--accent2)); display: flex; align-items: center; justify-content: center; font-size: 11px; font-weight: 700; flex-shrink: 0; }
  .user-name { font-size: 12px; font-weight: 600; color: var(--text1); }
  .user-role { font-size: 10px; color: var(--text3); }
  
  /* MAIN */
  .main { flex: 1; overflow-y: auto; overflow-x: hidden; background: var(--bg); }
  .main::-webkit-scrollbar { width: 4px; }
  .main::-webkit-scrollbar-track { background: transparent; }
  .main::-webkit-scrollbar-thumb { background: var(--border); border-radius: 4px; }
  
  .topbar { position: sticky; top: 0; z-index: 100; background: rgba(7,11,18,0.9); backdrop-filter: blur(12px); border-bottom: 1px solid var(--border); padding: 14px 24px; display: flex; align-items: center; gap: 12px; }
  .page-title { font-size: 16px; font-weight: 700; flex: 1; }
  .page-sub { font-size: 11px; color: var(--text3); }
  
  .content { padding: 24px; }
  
  /* CARDS */
  .card { background: var(--card); border: 1px solid var(--border); border-radius: var(--r); padding: 20px; margin-bottom: 16px; }
  .card-sm { background: var(--card); border: 1px solid var(--border); border-radius: var(--r); padding: 14px 16px; }
  .card-title { font-size: 13.5px; font-weight: 600; color: var(--text1); }
  .card-sub { font-size: 11px; color: var(--text2); margin-top: 2px; }
  
  .grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
  .grid-3 { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 16px; }
  .grid-4 { display: grid; grid-template-columns: 1fr 1fr 1fr 1fr; gap: 16px; }
  
  /* STAT CARDS */
  .stat-card { background: var(--card); border: 1px solid var(--border); border-radius: var(--r); padding: 18px; }
  .stat-icon { font-size: 22px; margin-bottom: 10px; }
  .stat-val { font-size: 28px; font-weight: 800; line-height: 1; }
  .stat-label { font-size: 11px; color: var(--text2); margin-top: 4px; }
  .stat-delta { font-size: 10px; margin-top: 6px; }
  .stat-delta.up { color: var(--accent4); }
  .stat-delta.down { color: var(--accent6); }
  
  /* BUTTONS */
  .btn { display: inline-flex; align-items: center; gap: 6px; border: none; border-radius: var(--r2); cursor: pointer; font-family: var(--sans); font-weight: 600; transition: all 0.15s; }
  .btn-primary { background: var(--accent); color: #fff; padding: 9px 18px; font-size: 12.5px; }
  .btn-primary:hover { background: #6b8fff; transform: translateY(-1px); }
  .btn-secondary { background: var(--card2); color: var(--text1); border: 1px solid var(--border); padding: 8px 16px; font-size: 12px; }
  .btn-secondary:hover { border-color: var(--accent); color: var(--accent); }
  .btn-danger { background: rgba(239,68,68,0.15); color: var(--accent6); border: 1px solid rgba(239,68,68,0.3); padding: 7px 14px; font-size: 11.5px; }
  .btn-danger:hover { background: rgba(239,68,68,0.25); }
  .btn-sm { padding: 6px 12px; font-size: 11px; }
  .btn-icon { padding: 7px; background: var(--card2); border: 1px solid var(--border); color: var(--text2); border-radius: var(--r2); cursor: pointer; font-size: 14px; transition: all 0.15s; }
  .btn-icon:hover { color: var(--accent); border-color: var(--accent); }
  .btn-loading { opacity: 0.7; pointer-events: none; }
  
  /* BADGES */
  .badge { display: inline-flex; align-items: center; gap: 4px; padding: 3px 8px; border-radius: 20px; font-size: 10.5px; font-weight: 600; }
  .badge-blue { background: rgba(79,124,255,0.15); color: var(--accent); border: 1px solid rgba(79,124,255,0.25); }
  .badge-green { background: rgba(16,185,129,0.15); color: var(--accent4); border: 1px solid rgba(16,185,129,0.25); }
  .badge-yellow { background: rgba(245,158,11,0.15); color: var(--accent5); border: 1px solid rgba(245,158,11,0.25); }
  .badge-red { background: rgba(239,68,68,0.15); color: var(--accent6); border: 1px solid rgba(239,68,68,0.25); }
  .badge-purple { background: rgba(139,92,246,0.15); color: var(--accent2); border: 1px solid rgba(139,92,246,0.25); }
  
  /* FORMS */
  input, select, textarea {
    background: var(--card2); border: 1px solid var(--border); border-radius: var(--r2);
    color: var(--text1); font-family: var(--sans); font-size: 12.5px; padding: 9px 12px;
    outline: none; transition: border 0.15s; width: 100%;
  }
  input:focus, select:focus, textarea:focus { border-color: var(--accent); }
  label { font-size: 11px; color: var(--text2); margin-bottom: 4px; display: block; font-weight: 500; }
  .form-group { margin-bottom: 14px; }
  
  /* TABLE */
  .table-wrap { overflow-x: auto; }
  table { width: 100%; border-collapse: collapse; font-size: 12.5px; }
  thead th { padding: 10px 14px; text-align: left; font-size: 10.5px; color: var(--text3); text-transform: uppercase; letter-spacing: 0.08em; border-bottom: 1px solid var(--border); font-weight: 600; }
  tbody td { padding: 12px 14px; border-bottom: 1px solid rgba(99,140,255,0.05); color: var(--text2); }
  tbody tr:hover td { background: rgba(79,124,255,0.04); color: var(--text1); }
  tbody tr:last-child td { border-bottom: none; }
  
  /* SEARCH BAR */
  .search-bar { position: relative; }
  .search-bar input { padding-left: 34px; }
  .search-icon { position: absolute; left: 11px; top: 50%; transform: translateY(-50%); font-size: 14px; color: var(--text3); pointer-events: none; }
  
  /* PROGRESS BAR */
  .progress-wrap { background: rgba(99,140,255,0.1); border-radius: 4px; height: 6px; overflow: hidden; }
  .progress-bar { height: 100%; border-radius: 4px; transition: width 0.5s ease; }
  
  /* CHAT */
  .chat-container { height: 380px; overflow-y: auto; padding: 16px; display: flex; flex-direction: column; gap: 12px; }
  .chat-container::-webkit-scrollbar { width: 3px; }
  .chat-container::-webkit-scrollbar-thumb { background: var(--border); border-radius: 4px; }
  .msg-row { display: flex; gap: 8px; align-items: flex-start; }
  .msg-row.user { flex-direction: row-reverse; }
  .msg-avatar { width: 28px; height: 28px; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-size: 12px; flex-shrink: 0; }
  .msg-avatar.ai { background: linear-gradient(135deg, var(--accent), var(--accent2)); }
  .msg-avatar.user { background: linear-gradient(135deg, var(--accent4), var(--accent3)); }
  .msg-bubble { max-width: 70%; padding: 10px 13px; border-radius: 10px; font-size: 12.5px; line-height: 1.5; }
  .msg-bubble.ai { background: var(--card2); border: 1px solid var(--border); color: var(--text1); }
  .msg-bubble.user { background: linear-gradient(135deg, var(--accent), var(--accent2)); color: #fff; }
  .chat-input-row { display: flex; gap: 8px; padding: 12px 16px; border-top: 1px solid var(--border); }
  .chat-input-row input { flex: 1; }
  
  /* SYMPTOM TAGS */
  .tag-area { display: flex; flex-wrap: wrap; gap: 6px; min-height: 36px; padding: 8px; background: var(--card2); border: 1px solid var(--border); border-radius: var(--r2); }
  .sym-tag { display: inline-flex; align-items: center; gap: 4px; padding: 4px 10px; background: rgba(79,124,255,0.15); border: 1px solid rgba(79,124,255,0.25); border-radius: 20px; font-size: 11.5px; color: var(--accent); cursor: default; }
  .sym-tag span { cursor: pointer; opacity: 0.7; font-size: 12px; }
  .sym-tag span:hover { opacity: 1; }
  
  /* PREDICTION RESULT */
  .pred-item { display: flex; align-items: center; gap: 10px; margin-bottom: 10px; }
  .pred-name { font-size: 12px; color: var(--text1); width: 130px; flex-shrink: 0; }
  .pred-bar-wrap { flex: 1; background: rgba(99,140,255,0.08); border-radius: 4px; height: 7px; overflow: hidden; }
  .pred-pct { font-size: 11.5px; color: var(--text2); font-family: var(--mono); width: 36px; text-align: right; flex-shrink: 0; }
  
  /* TABS */
  .tabs { display: flex; gap: 4px; background: var(--card2); padding: 4px; border-radius: var(--r2); }
  .tab { padding: 6px 14px; font-size: 11.5px; cursor: pointer; border-radius: 6px; color: var(--text2); font-weight: 500; transition: all 0.15s; }
  .tab.active { background: var(--card); color: var(--text1); box-shadow: 0 1px 4px rgba(0,0,0,0.3); }
  
  /* LOADING */
  .spinner { width: 18px; height: 18px; border: 2px solid rgba(79,124,255,0.2); border-top-color: var(--accent); border-radius: 50%; animation: spin 0.7s linear infinite; display: inline-block; }
  @keyframes spin { to { transform: rotate(360deg); } }
  
  .ai-loading { display: flex; align-items: center; gap: 8px; padding: 10px 13px; background: var(--card2); border: 1px solid var(--border); border-radius: 10px; font-size: 12px; color: var(--text2); width: fit-content; }
  
  /* LOGIN */
  .login-screen { position: fixed; inset: 0; background: var(--bg); display: flex; align-items: center; justify-content: center; z-index: 1000; }
  .login-card { background: var(--card); border: 1px solid var(--border); border-radius: 18px; padding: 40px; width: 380px; }
  
  /* UPLOAD ZONE */
  .upload-zone { border: 2px dashed var(--border2); border-radius: var(--r); padding: 32px; text-align: center; cursor: pointer; transition: all 0.2s; }
  .upload-zone:hover { border-color: var(--accent); background: rgba(79,124,255,0.04); }
  
  /* SEVERITY PILLS */
  .sev-pill { display: inline-flex; align-items: center; gap: 5px; padding: 5px 12px; border-radius: 20px; font-size: 11.5px; font-weight: 600; }
  .sev-low { background: rgba(16,185,129,0.15); color: var(--accent4); border: 1px solid rgba(16,185,129,0.25); }
  .sev-mod { background: rgba(245,158,11,0.15); color: var(--accent5); border: 1px solid rgba(245,158,11,0.25); }
  .sev-high { background: rgba(239,68,68,0.15); color: var(--accent6); border: 1px solid rgba(239,68,68,0.25); }
  
  /* DIVIDER */
  .divider { height: 1px; background: var(--border); margin: 16px 0; }
  
  /* SCROLL FIX */
  .overflow-y { overflow-y: auto; }
  .overflow-y::-webkit-scrollbar { width: 3px; }
  .overflow-y::-webkit-scrollbar-thumb { background: var(--border); border-radius: 4px; }
  
  /* HEATMAP */
  .heatmap { display: grid; grid-template-columns: repeat(7, 1fr); gap: 4px; }
  .hm-cell { aspect-ratio: 1; border-radius: 4px; cursor: pointer; transition: opacity 0.2s; }
  .hm-cell:hover { opacity: 0.7; }
  
  /* TIMELINE */
  .timeline { position: relative; padding-left: 24px; }
  .timeline::before { content: ''; position: absolute; left: 6px; top: 0; bottom: 0; width: 1px; background: var(--border); }
  .tl-item { position: relative; margin-bottom: 20px; }
  .tl-dot { position: absolute; left: -22px; top: 3px; width: 10px; height: 10px; border-radius: 50%; border: 2px solid var(--accent); background: var(--bg); }
  .tl-date { font-size: 10px; color: var(--text3); margin-bottom: 3px; font-family: var(--mono); }
  .tl-text { font-size: 12.5px; color: var(--text1); font-weight: 500; }
  .tl-sub { font-size: 11px; color: var(--text2); margin-top: 2px; }
  
  /* METRIC RING */
  .ring-wrap { position: relative; width: 80px; height: 80px; }
  .ring-score { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; font-size: 18px; font-weight: 800; }
  .ring-label { font-size: 9px; color: var(--text3); }
  
  /* ALERT ROW */
  .alert-row { display: flex; align-items: center; gap: 12px; padding: 12px; background: var(--card2); border: 1px solid var(--border); border-radius: var(--r2); margin-bottom: 8px; }
  .alert-icon { width: 34px; height: 34px; border-radius: 8px; display: flex; align-items: center; justify-content: center; font-size: 16px; flex-shrink: 0; }
  
  /* EXPLAINABILITY */
  .explain-bar { margin-bottom: 8px; }
  .explain-label { display: flex; justify-content: space-between; font-size: 11.5px; color: var(--text2); margin-bottom: 4px; }
  
  /* SCROLL TO TOP FIX */  
  .page-view { animation: fadeIn 0.2s ease; }
  @keyframes fadeIn { from { opacity: 0; transform: translateY(6px); } to { opacity: 1; transform: translateY(0); } }
  
  select option { background: var(--card2); }
  
  pre { font-family: var(--mono); font-size: 12px; color: var(--text2); white-space: pre-wrap; }
  
  .report-result { background: var(--card2); border: 1px solid var(--border); border-radius: var(--r2); padding: 14px; margin-top: 14px; font-size: 12.5px; line-height: 1.7; color: var(--text1); }
  
  strong { color: var(--text1); }

  .chip { display: inline-flex; align-items: center; gap: 4px; padding: 3px 9px; background: var(--card2); border: 1px solid var(--border); border-radius: 20px; font-size: 11px; color: var(--text2); cursor: pointer; transition: all 0.15s; }
  .chip:hover, .chip.sel { background: rgba(79,124,255,0.15); border-color: var(--accent); color: var(--accent); }
  
  .num-ring { width: 8px; height: 8px; border-radius: 50%; display: inline-block; flex-shrink: 0; }

  /* ── MOBILE BOTTOM NAV (hidden on desktop) ─────────────────────── */
  .mobile-nav { display: none; }

  /* ── TABLET & MOBILE (max 768px) ───────────────────────────────── */
  @media (max-width: 768px) {

    /* Layout */
    .app { flex-direction: column; height: 100vh; overflow: hidden; }
    .sidebar { display: none !important; }
    .main { flex: 1; overflow-y: auto; overflow-x: hidden; padding-bottom: 68px; }

    /* Topbar */
    .topbar { padding: 10px 14px; gap: 8px; flex-wrap: nowrap; }
    .page-title { font-size: 12.5px; }
    .page-sub { display: none; }

    /* Content */
    .content { padding: 12px; }

    /* Cards */
    .card { padding: 14px; margin-bottom: 12px; }
    .card-sm { padding: 11px 13px; }
    .card-title { font-size: 12.5px; }

    /* Grids — single column */
    .grid-2 { grid-template-columns: 1fr !important; gap: 12px; }
    .grid-3 { grid-template-columns: 1fr !important; gap: 12px; }
    .grid-4 { grid-template-columns: 1fr 1fr !important; gap: 10px; }

    /* Stat cards */
    .stat-card { padding: 14px; }
    .stat-val { font-size: 22px; }
    .stat-icon { font-size: 17px; margin-bottom: 5px; }
    .stat-label { font-size: 10px; }

    /* Login */
    .login-card { width: 92vw; padding: 24px 18px; }
    .login-screen { padding: 16px; justify-content: center; }

    /* Chat */
    .chat-container { height: 290px; padding: 10px; }
    .msg-bubble { max-width: 86%; font-size: 12px; padding: 9px 11px; }
    .chat-input-row { padding: 8px 10px; gap: 6px; }

    /* Prediction bar */
    .pred-name { width: 90px; font-size: 11px; }

    /* Tabs */
    .tabs { overflow-x: auto; flex-wrap: nowrap; -webkit-overflow-scrolling: touch; }
    .tab { white-space: nowrap; padding: 6px 10px; font-size: 11px; }

    /* Inputs — 16px prevents iOS zoom */
    input, select, textarea { font-size: 16px !important; }

    /* Buttons */
    .btn-primary { padding: 10px 14px; font-size: 12px; }
    .btn-secondary { padding: 9px 12px; font-size: 11.5px; }
    .btn-sm { padding: 7px 11px; font-size: 11px; }

    /* Tables */
    .table-wrap { overflow-x: auto; -webkit-overflow-scrolling: touch; }
    table { min-width: 480px; }

    /* Heatmap */
    .heatmap { grid-template-columns: repeat(7, 1fr); }

    /* Upload zone */
    .upload-zone { padding: 20px 12px; }

    /* Bottom Navigation */
    .mobile-nav {
      display: flex !important;
      position: fixed;
      bottom: 0; left: 0; right: 0;
      height: 62px;
      background: var(--bg2);
      border-top: 1px solid var(--border);
      z-index: 999;
      overflow-x: auto;
      overflow-y: hidden;
      scrollbar-width: none;
    }
    .mobile-nav::-webkit-scrollbar { display: none; }
    .mobile-nav-item {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-width: 54px;
      flex: 1;
      padding: 5px 2px;
      cursor: pointer;
      color: var(--text3);
      border-top: 2px solid transparent;
      transition: all 0.15s;
    }
    .mobile-nav-item.active {
      color: var(--accent);
      border-top-color: var(--accent);
      background: rgba(79,124,255,0.07);
    }
    .mobile-nav-item .mn-icon { font-size: 18px; line-height: 1; }
    .mobile-nav-item .mn-label { font-size: 8px; font-weight: 600; margin-top: 2px; white-space: nowrap; }
  }

  /* ── SMALL PHONES (max 420px) ───────────────────────────────────── */
  @media (max-width: 420px) {
    .content { padding: 10px; }
    .card { padding: 12px; }
    .topbar { padding: 8px 10px; }
    .page-title { font-size: 12px; }
    .grid-4 { grid-template-columns: 1fr 1fr !important; }
    .stat-val { font-size: 20px; }
    .chat-container { height: 260px; }
  }
`;

// ─── AUTH PAGE ───────────────────────────────────────────────────────────────
function LoginPage({ onLogin }) {
  const [form, setForm] = useState({ email: "", password: "", name: "", age: "", gender: "Male" });
  const [mode, setMode] = useState("login"); // login | register
  const [loading, setLoading] = useState(false);
  const [err, setErr] = useState("");

  const handleSubmit = async () => {
    setLoading(true); setErr("");
    await new Promise(r => setTimeout(r, 600));
    if (mode === "login") {
      const u = MOCK_USERS.find(u => u.email === form.email) || MOCK_USERS[0];
      onLogin(u);
    } else {
      const newUser = { id: Date.now(), name: form.name, email: form.email, age: parseInt(form.age)||25, gender: form.gender, bloodGroup: "O+", role: "user", predictions: 0, healthScore: 75 };
      onLogin(newUser);
    }
    setLoading(false);
  };

  return (
    <div className="login-screen">
      <div style={{ textAlign: "center", marginBottom: 24 }}>
        <div className="logo-icon" style={{ width: 52, height: 52, margin: "0 auto 12px", fontSize: 24, background: "linear-gradient(135deg, #4f7cff, #8b5cf6)", borderRadius: 14 }}>🏥</div>
        <div style={{ fontFamily: "var(--sans)", fontSize: 22, fontWeight: 800, color: "var(--text1)" }}>HealthSphere AI</div>
        <div style={{ fontSize: 11, color: "var(--text3)", marginTop: 2, letterSpacing: "0.1em" }}>INTELLIGENT HEALTH PLATFORM</div>
      </div>
      <div className="login-card">
        <div className="tabs" style={{ marginBottom: 20 }}>
          <div className={`tab ${mode === "login" ? "active" : ""}`} style={{ flex: 1, textAlign: "center" }} onClick={() => setMode("login")}>Sign In</div>
          <div className={`tab ${mode === "register" ? "active" : ""}`} style={{ flex: 1, textAlign: "center" }} onClick={() => setMode("register")}>Register</div>
        </div>

        {mode === "register" && (
          <>
            <div className="form-group">
              <label>Full Name</label>
              <input placeholder="Your full name" value={form.name} onChange={e => setForm({ ...form, name: e.target.value })} />
            </div>
            <div className="grid-2" style={{ gap: 10 }}>
              <div className="form-group"><label>Age</label><input type="number" placeholder="25" value={form.age} onChange={e => setForm({ ...form, age: e.target.value })} /></div>
              <div className="form-group"><label>Gender</label>
                <select value={form.gender} onChange={e => setForm({ ...form, gender: e.target.value })}>
                  <option>Male</option><option>Female</option><option>Other</option>
                </select>
              </div>
            </div>
          </>
        )}

        <div className="form-group">
          <label>Email</label>
          <input type="email" placeholder="you@example.com" value={form.email} onChange={e => setForm({ ...form, email: e.target.value })} />
        </div>
        <div className="form-group">
          <label>Password</label>
          <input type="password" placeholder="••••••••" value={form.password} onChange={e => setForm({ ...form, password: e.target.value })} />
        </div>

        {err && <div style={{ color: "var(--accent6)", fontSize: 12, marginBottom: 12 }}>{err}</div>}

        <button className={`btn btn-primary ${loading ? "btn-loading" : ""}`} style={{ width: "100%", justifyContent: "center", padding: "11px", marginBottom: 12 }} onClick={handleSubmit}>
          {loading ? <><div className="spinner" /> Processing…</> : (mode === "login" ? "Sign In" : "Create Account")}
        </button>

        <div style={{ textAlign: "center", fontSize: 11, color: "var(--text3)" }}>
          Demo: use any email • password: any
        </div>
        <div style={{ textAlign: "center", marginTop: 10 }}>
          <span className="badge badge-purple" style={{ cursor: "pointer" }} onClick={() => onLogin({ ...MOCK_USERS[0], role: "admin" })}>👑 Enter as Admin</span>
        </div>
      </div>
    </div>
  );
}

// ─── DASHBOARD ───────────────────────────────────────────────────────────────
function Dashboard({ user }) {
  const stats = [
    { icon: "🧬", val: "2,847", label: "Total Predictions", delta: "+124 this week", dir: "up" },
    { icon: "👥", val: "1,203", label: "Registered Users", delta: "+38 this week", dir: "up" },
    { icon: "📋", val: "847", label: "Reports Analyzed", delta: "+61 this week", dir: "up" },
    { icon: "🚨", val: "4", label: "Active Alerts", delta: "Dengue rising", dir: "down" },
  ];

  return (
    <div className="page-view">
      <div className="grid-4" style={{ marginBottom: 20 }}>
        {stats.map((s, i) => (
          <div key={i} className="stat-card">
            <div className="stat-icon">{s.icon}</div>
            <div className="stat-val">{s.val}</div>
            <div className="stat-label">{s.label}</div>
            <div className={`stat-delta ${s.dir}`}>{s.dir === "up" ? "↑" : "↓"} {s.delta}</div>
          </div>
        ))}
      </div>

      <div className="grid-2">
        <div className="card">
          <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 16 }}>
            <div className="card-title">📊 Disease Trend — Last 6 Months</div>
          </div>
          <svg viewBox="0 0 480 140" style={{ width: "100%", height: 140 }}>
            <defs>
              <linearGradient id="dg1" x1="0" y1="0" x2="0" y2="1">
                <stop offset="0%" stopColor="#4f7cff" stopOpacity="0.4"/>
                <stop offset="100%" stopColor="#4f7cff" stopOpacity="0"/>
              </linearGradient>
            </defs>
            {[0,40,80,120].map(y => <line key={y} x1="0" y1={y} x2="480" y2={y} stroke="rgba(255,255,255,0.04)" strokeWidth="1"/>)}
            <path d="M0,110 C60,90 120,70 180,80 C240,90 300,50 360,40 C420,30 450,35 480,25" fill="none" stroke="#4f7cff" strokeWidth="2.5"/>
            <path d="M0,110 C60,90 120,70 180,80 C240,90 300,50 360,40 C420,30 450,35 480,25 L480,140 L0,140Z" fill="url(#dg1)"/>
            <path d="M0,125 C60,118 120,112 180,105 C240,98 300,110 360,100 C420,90 450,105 480,95" fill="none" stroke="#10b981" strokeWidth="1.5"/>
            <path d="M0,130 C60,128 120,124 180,127 C240,130 300,120 360,116 C420,112 450,117 480,113" fill="none" stroke="#f59e0b" strokeWidth="1.5" strokeDasharray="5,3"/>
            {["Jan","Feb","Mar","Apr","May","Jun"].map((m, i) => (
              <text key={m} x={i * 96} y={138} fill="#3a4a66" fontSize="9" fontFamily="Sora, sans-serif">{m}</text>
            ))}
          </svg>
          <div style={{ display: "flex", gap: 14, marginTop: 8 }}>
            {[["#4f7cff","Viral Fever"], ["#10b981","Dengue"], ["#f59e0b","Typhoid"]].map(([c, l]) => (
              <div key={l} style={{ display: "flex", alignItems: "center", gap: 5, fontSize: 11, color: "var(--text2)" }}>
                <span style={{ width: 16, height: 2, background: c, display: "inline-block", borderRadius: 1 }}></span>{l}
              </div>
            ))}
          </div>
        </div>

        <div className="card">
          <div className="card-title" style={{ marginBottom: 16 }}>🚨 Community Alerts</div>
          {COMMUNITY_ALERTS.map(a => (
            <div key={a.id} className="alert-row">
              <div className="alert-icon" style={{ background: a.severity === "high" ? "rgba(239,68,68,0.15)" : "rgba(245,158,11,0.12)" }}>
                {a.severity === "high" ? "🔴" : "🟡"}
              </div>
              <div style={{ flex: 1 }}>
                <div style={{ fontWeight: 600, fontSize: 12.5, color: "var(--text1)" }}>{a.disease} — {a.area}</div>
                <div style={{ fontSize: 11, color: "var(--text3)" }}>{a.cases} cases • {a.date}</div>
              </div>
              <span className={`badge ${a.trend === "rising" ? "badge-red" : a.trend === "declining" ? "badge-green" : "badge-yellow"}`}>
                {a.trend === "rising" ? "↑" : a.trend === "declining" ? "↓" : "→"} {a.trend}
              </span>
            </div>
          ))}
        </div>
      </div>

      <div className="grid-4" style={{ marginTop: 0 }}>
        {[["Age 21–35","42%","Highest risk group","var(--accent)"], ["Viral Fever","32%","Most predicted","var(--accent3)"], ["Avg Confidence","84%","Model accuracy","var(--accent4)"], ["Avg Health Score","74/100","Platform average","var(--accent2)"]].map(([l, v, s, c]) => (
          <div key={l} className="card-sm" style={{ textAlign: "center" }}>
            <div style={{ fontSize: 24, fontWeight: 800, color: c, marginBottom: 4 }}>{v}</div>
            <div style={{ fontSize: 12, fontWeight: 600, color: "var(--text1)" }}>{l}</div>
            <div style={{ fontSize: 10.5, color: "var(--text3)", marginTop: 2 }}>{s}</div>
          </div>
        ))}
      </div>
    </div>
  );
}

// ─── SYMPTOM CHECKER / DISEASE PREDICTION ─────────────────────────────────────
function DiseasePrediction({ user }) {
  const [symptoms, setSymptoms] = useState([]);
  const [input, setInput] = useState("");
  const [age, setAge] = useState(user.age || 28);
  const [gender, setGender] = useState(user.gender || "Male");
  const [result, setResult] = useState(null);
  const [loading, setLoading] = useState(false);
  const [followUps, setFollowUps] = useState([]);
  const [fuLoading, setFuLoading] = useState(false);

  const addSymptom = () => {
    const v = input.trim();
    if (!v || symptoms.includes(v)) return;
    setSymptoms([...symptoms, v]);
    setInput("");
  };

  const suggestSymptoms = ["Fever", "Headache", "Fatigue", "Cough", "Nausea", "Chest pain", "Rash", "Joint pain", "Dizziness", "Diarrhea"];

  const loadExample = () => {
    setAge(28);
    setGender("Male");
    setSymptoms(["Fever", "Headache", "Body Pain", "Fatigue"]);
  };

  const predict = async () => {
    if (!symptoms.length) return;
    setLoading(true); setResult(null); setFollowUps([]);
    try {
      const data = await callClaudeJSON([{ role: "user", content: `Patient: ${age}yr ${gender}. Symptoms: ${symptoms.join(", ")}. Analyze and return JSON.` }],
        `You are a medical AI. Return JSON with:
{
  "predictions": [{"disease":"name","confidence":85,"description":"brief"},...] (top 4),
  "severity": "low|moderate|high",
  "risk": "low|moderate|high",
  "recommendation": "brief advice",
  "explainability": {"symptom_pattern":40,"seasonal_trends":30,"history":30},
  "followUpQuestions": ["q1","q2","q3"]
}
Base confidence on symptom match. severity/risk independently assessed. Be medically accurate.`);
      setResult(data);
      setFollowUps(data?.followUpQuestions || []);
    } catch { setResult({ error: true }); }
    setLoading(false);
  };

  const getFollowUpAnswer = async (q) => {
    setFuLoading(true);
    const answer = await callClaude([{ role: "user", content: `Patient question: "${q}". Context: symptoms are ${symptoms.join(", ")}, age ${age}, gender ${gender}. Top prediction: ${result?.predictions?.[0]?.disease}` }],
      "You are a concise medical AI assistant. Answer in 1-2 sentences with practical advice.");
    setFollowUps(prev => prev.map(fq => fq === q ? { q, a: answer } : fq));
    setFuLoading(false);
  };

  const sevColor = { low: "sev-low", moderate: "sev-mod", high: "sev-high" };

  return (
    <div className="page-view">
      <div className="grid-2">
        <div className="card">
          <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 14 }}>
            <div className="card-title">🔬 Symptom Input</div>
            <button className="btn btn-secondary btn-sm" onClick={loadExample} style={{ fontSize: 11, background: "rgba(245,158,11,0.1)", border: "1px solid rgba(245,158,11,0.35)", color: "#f59e0b" }}>
              ⚡ Try Example
            </button>
          </div>
          <div style={{ background: "rgba(245,158,11,0.07)", border: "1px solid rgba(245,158,11,0.2)", borderRadius: "var(--r2)", padding: "10px 13px", marginBottom: 14, fontSize: 11.5, color: "var(--text2)", lineHeight: 1.6 }}>
            <span style={{ color: "#f59e0b", fontWeight: 600 }}>📋 How it works:</span> Enter patient age &amp; gender → Add symptoms one by one (or click quick-add chips) → Click <strong style={{ color: "var(--text1)" }}>Run AI Prediction</strong> → AI will show disease name, confidence %, risk level, and why it predicted that disease.
          </div>

          <div className="grid-2" style={{ gap: 10, marginBottom: 14 }}>
            <div className="form-group" style={{ marginBottom: 0 }}>
              <label>Age</label>
              <input type="number" value={age} onChange={e => setAge(e.target.value)} />
            </div>
            <div className="form-group" style={{ marginBottom: 0 }}>
              <label>Gender</label>
              <select value={gender} onChange={e => setGender(e.target.value)}>
                <option>Male</option><option>Female</option><option>Other</option>
              </select>
            </div>
          </div>

          <label>Add Symptoms</label>
          <div style={{ display: "flex", gap: 8, marginBottom: 10 }}>
            <input value={input} onChange={e => setInput(e.target.value)} placeholder="e.g. headache, fever…" onKeyDown={e => e.key === "Enter" && addSymptom()} />
            <button className="btn btn-primary btn-sm" onClick={addSymptom}>Add</button>
          </div>

          <div className="tag-area" style={{ marginBottom: 12 }}>
            {symptoms.map(s => (
              <div key={s} className="sym-tag">{s} <span onClick={() => setSymptoms(symptoms.filter(x => x !== s))}>×</span></div>
            ))}
            {!symptoms.length && <span style={{ fontSize: 11.5, color: "var(--text3)" }}>No symptoms added yet</span>}
          </div>

          <div style={{ marginBottom: 14 }}>
            <div style={{ fontSize: 11, color: "var(--text3)", marginBottom: 7 }}>Quick add:</div>
            <div style={{ display: "flex", flexWrap: "wrap", gap: 5 }}>
              {suggestSymptoms.map(s => (
                <span key={s} className={`chip ${symptoms.includes(s) ? "sel" : ""}`} onClick={() => !symptoms.includes(s) && setSymptoms([...symptoms, s])}>{s}</span>
              ))}
            </div>
          </div>

          <button className={`btn btn-primary ${loading ? "btn-loading" : ""}`} style={{ width: "100%", justifyContent: "center", padding: "11px" }} onClick={predict} disabled={!symptoms.length}>
            {loading ? <><div className="spinner" /> Analyzing…</> : "🧠 Run AI Prediction"}
          </button>
        </div>

        <div className="card">
          <div className="card-title" style={{ marginBottom: 14 }}>🧠 AI Prediction Results</div>

          {!result && !loading && (
            <div style={{ textAlign: "center", padding: "30px 20px", color: "var(--text3)" }}>
              <div style={{ fontSize: 32, marginBottom: 10 }}>🔬</div>
              <div style={{ fontSize: 13, marginBottom: 12 }}>Add symptoms and click Analyze to get AI predictions</div>
              <div style={{ background: "rgba(79,124,255,0.07)", border: "1px solid rgba(79,124,255,0.18)", borderRadius: "var(--r2)", padding: "12px 14px", textAlign: "left", fontSize: 12, color: "var(--text2)", lineHeight: 1.7 }}>
                <div style={{ color: "var(--accent)", fontWeight: 600, marginBottom: 6 }}>👁 Example Output Preview:</div>
                <div>🏆 <strong style={{ color: "var(--text1)" }}>Viral Fever</strong> — 85% confidence</div>
                <div style={{ marginTop: 4 }}>⚡ Risk: <span style={{ color: "#f59e0b" }}>Moderate</span> &nbsp;|&nbsp; 🩺 Severity: <span style={{ color: "#f59e0b" }}>Moderate</span></div>
                <div style={{ marginTop: 4 }}>🧩 Prediction factors: Fever 35% · Body Pain 25% · Headache 20% · Fatigue 20%</div>
                <div style={{ marginTop: 4 }}>💊 Recommendation: Rest, stay hydrated, take paracetamol for fever.</div>
              </div>
            </div>
          )}

          {loading && (
            <div style={{ padding: "40px 20px", textAlign: "center" }}>
              <div className="spinner" style={{ width: 32, height: 32, margin: "0 auto 12px", borderWidth: 3 }} />
              <div style={{ color: "var(--text2)", fontSize: 13 }}>Running diagnostic analysis…</div>
            </div>
          )}

          {result && !result.error && (
            <>
              <div style={{ marginBottom: 14 }}>
                {result.predictions?.map((p, i) => (
                  <div key={i} className="pred-item">
                    <div className="pred-name">{p.disease}</div>
                    <div className="pred-bar-wrap">
                      <div className="progress-bar" style={{ width: `${p.confidence}%`, background: i === 0 ? "linear-gradient(90deg, #4f7cff, #8b5cf6)" : i === 1 ? "linear-gradient(90deg, #06b6d4, #4f7cff)" : "linear-gradient(90deg, #f59e0b, #ef4444)" }} />
                    </div>
                    <div className="pred-pct">{p.confidence}%</div>
                  </div>
                ))}
              </div>

              <div style={{ display: "flex", gap: 8, marginBottom: 14, flexWrap: "wrap" }}>
                <span className={`sev-pill ${sevColor[result.risk] || "sev-mod"}`}>⚡ Risk: {result.risk}</span>
                <span className={`sev-pill ${sevColor[result.severity] || "sev-mod"}`}>🩺 Severity: {result.severity}</span>
              </div>

              <div style={{ background: "rgba(139,92,246,0.07)", border: "1px solid rgba(139,92,246,0.18)", borderRadius: "var(--r2)", padding: "12px 14px", marginBottom: 14 }}>
                <div style={{ fontSize: 11, color: "#a78bfa", fontWeight: 600, marginBottom: 8 }}>🧩 AI Explainability</div>
                {result.explainability && Object.entries(result.explainability).map(([k, v]) => (
                  <div key={k} className="explain-bar">
                    <div className="explain-label"><span>{k.replace(/_/g, " ")}</span><span>{v}%</span></div>
                    <div className="progress-wrap"><div className="progress-bar" style={{ width: `${v}%`, background: "var(--accent2)" }} /></div>
                  </div>
                ))}
              </div>

              <div style={{ background: "rgba(16,185,129,0.07)", border: "1px solid rgba(16,185,129,0.18)", borderRadius: "var(--r2)", padding: "12px 14px" }}>
                <div style={{ fontSize: 11, color: "var(--accent4)", fontWeight: 600, marginBottom: 4 }}>💊 Recommendation</div>
                <div style={{ fontSize: 12.5, color: "var(--text2)", lineHeight: 1.6 }}>{result.recommendation}</div>
              </div>
            </>
          )}
        </div>
      </div>

      {followUps.length > 0 && (
        <div className="card">
          <div className="card-title" style={{ marginBottom: 14 }}>💬 Dynamic Follow-up Questions</div>
          <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
            {followUps.map((fq, i) => (
              <div key={i} style={{ background: "var(--card2)", border: "1px solid var(--border)", borderRadius: "var(--r2)", padding: 12 }}>
                <div style={{ fontSize: 12.5, color: "var(--text1)", fontWeight: 500, marginBottom: typeof fq === "object" ? 8 : 0 }}>
                  {typeof fq === "object" ? fq.q : fq}
                </div>
                {typeof fq === "object" && fq.a && (
                  <div style={{ fontSize: 12, color: "var(--accent4)", lineHeight: 1.6 }}>→ {fq.a}</div>
                )}
                {typeof fq !== "object" && (
                  <button className="btn btn-secondary btn-sm" style={{ marginTop: 8 }} onClick={() => getFollowUpAnswer(fq)} disabled={fuLoading}>
                    {fuLoading ? <><div className="spinner" style={{ width: 12, height: 12 }} /> Asking…</> : "Get AI Answer"}
                  </button>
                )}
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}

// ─── AI HEALTH ASSISTANT ──────────────────────────────────────────────────────
function AIAssistant({ user }) {
  const [messages, setMessages] = useState([
    { role: "assistant", content: `Hello ${user.name}! I'm your AI Health Assistant. I can help with symptom analysis, health advice, medication information, nutrition guidance, and more. What's on your mind today?` }
  ]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);
  const scrollRef = useRef(null);

  useEffect(() => {
    if (scrollRef.current) scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
  }, [messages]);

  const send = async () => {
    if (!input.trim() || loading) return;
    const userMsg = { role: "user", content: input };
    const newMsgs = [...messages, userMsg];
    setMessages(newMsgs);
    setInput("");
    setLoading(true);

    try {
      const apiMsgs = newMsgs.map(m => ({ role: m.role === "assistant" ? "assistant" : "user", content: m.content }));
      const reply = await callClaude(apiMsgs, `You are HealthSphere AI, a compassionate, knowledgeable AI health assistant. Patient: ${user.name}, ${user.age}yr, ${user.gender}. Always be helpful, accurate, and recommend consulting a doctor for serious concerns. Keep responses concise (2-3 paragraphs max).`);
      setMessages(prev => [...prev, { role: "assistant", content: reply }]);
    } catch {
      setMessages(prev => [...prev, { role: "assistant", content: "I'm sorry, I encountered an issue. Please try again." }]);
    }
    setLoading(false);
  };

  const quickPrompts = ["What are symptoms of dengue?", "Tips to lower blood pressure naturally", "Best diet for diabetes management", "When should I see a doctor for fever?"];

  const loadExample = () => {
    const exampleConvo = [
      { role: "user", content: "I have had fever since yesterday and also have headache and body pain." },
      { role: "assistant", content: "I understand — that sounds uncomfortable. Based on what you've described (fever + headache + body pain), this could be Viral Fever or possibly early Dengue.\n\n🌡️ Immediate steps:\n• Take Paracetamol (500mg) every 6 hours for fever\n• Drink at least 2–3 litres of water or ORS daily\n• Complete bed rest\n• Monitor your temperature every 6 hours\n\nDo you also have any rash, joint pain, or eye pain? Those would suggest Dengue, which needs a doctor visit promptly. ⚠️" }
    ];
    setMessages(prev => [...prev, ...exampleConvo]);
  };

  return (
    <div className="page-view">
      <div className="card" style={{ padding: 0, overflow: "hidden" }}>
        <div style={{ display: "flex", alignItems: "center", gap: 10, padding: "14px 18px", borderBottom: "1px solid var(--border)" }}>
          <div style={{ width: 34, height: 34, borderRadius: "50%", background: "linear-gradient(135deg, var(--accent), var(--accent2))", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 16 }}>🤖</div>
          <div>
            <div className="card-title">AI Health Assistant</div>
            <div className="card-sub">Powered by Gen AI</div>
          </div>
          <span className="badge badge-green" style={{ marginLeft: "auto" }}>● Online</span>
          <button className="btn btn-secondary btn-sm" onClick={loadExample} style={{ fontSize: 11, background: "rgba(245,158,11,0.1)", border: "1px solid rgba(245,158,11,0.35)", color: "#f59e0b" }}>
            ⚡ Try Example
          </button>
        </div>

        <div className="chat-container" ref={scrollRef}>
          {messages.map((m, i) => (
            <div key={i} className={`msg-row ${m.role === "user" ? "user" : ""}`}>
              <div className={`msg-avatar ${m.role === "assistant" ? "ai" : "user"}`}>
                {m.role === "assistant" ? "🤖" : user.name.slice(0, 2).toUpperCase()}
              </div>
              <div className={`msg-bubble ${m.role === "assistant" ? "ai" : "user"}`}>
                {m.content}
              </div>
            </div>
          ))}
          {loading && (
            <div className="msg-row">
              <div className="msg-avatar ai">🤖</div>
              <div className="ai-loading"><div className="spinner" style={{ width: 14, height: 14 }} /> Thinking…</div>
            </div>
          )}
        </div>

        <div style={{ padding: "10px 16px", borderTop: "1px solid rgba(99,140,255,0.06)" }}>
          <div style={{ background: "rgba(79,124,255,0.06)", border: "1px solid rgba(79,124,255,0.15)", borderRadius: "var(--r2)", padding: "9px 13px", marginBottom: 10, fontSize: 11.5, color: "var(--text2)", lineHeight: 1.6 }}>
            <span style={{ color: "var(--accent)", fontWeight: 600 }}>💬 How it works:</span> Type any health question or describe your symptoms → AI will ask follow-up questions → Then give diagnosis, advice &amp; recommendations. Click <strong style={{ color: "var(--text1)" }}>⚡ Try Example</strong> above to see a sample conversation instantly.
          </div>
          <div style={{ display: "flex", flexWrap: "wrap", gap: 5, marginBottom: 10 }}>
            {quickPrompts.map(q => (
              <span key={q} className="chip" onClick={() => { setInput(q); }}>{q}</span>
            ))}
          </div>
        </div>

        <div className="chat-input-row">
          <input value={input} onChange={e => setInput(e.target.value)} placeholder="Ask me anything about your health…" onKeyDown={e => e.key === "Enter" && send()} />
          <button className="btn btn-primary" onClick={send} disabled={loading || !input.trim()}>
            {loading ? <div className="spinner" style={{ width: 14, height: 14 }} /> : "➤"}
          </button>
        </div>
      </div>
    </div>
  );
}

// ─── MEDICAL REPORT ANALYZER ──────────────────────────────────────────────────
function ReportAnalyzer({ user }) {
  const [file, setFile] = useState(null);
  const [manualText, setManualText] = useState("");
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState({
    overallStatus: "abnormal",
    urgency: "soon",
    summary: "This report for Rahul Verma (45M) shows multiple concerning findings. Hemoglobin, RBC, and Platelets are all below normal range suggesting possible anemia or hematological issue. WBC is elevated indicating active infection or inflammation. Blood sugar and HbA1c values point to uncontrolled Type 2 Diabetes. Elevated Creatinine and CRP suggest early kidney stress and systemic inflammation. Lipid profile shows high cardiovascular risk with elevated LDL and low HDL.",
    abnormalValues: [
      { test: "Hemoglobin", value: "10.2 g/dL", normal: "13.5–17.5 g/dL", status: "low", concern: "Low hemoglobin indicates anemia — may cause fatigue and weakness" },
      { test: "WBC Count", value: "13,400 /µL", normal: "4,500–11,000 /µL", status: "high", concern: "Elevated WBC suggests active infection or inflammatory response" },
      { test: "Platelets", value: "92,000 /µL", normal: "150,000–400,000 /µL", status: "low", concern: "Thrombocytopenia — increased bleeding risk, needs investigation" },
      { test: "Fasting Blood Sugar", value: "148 mg/dL", normal: "70–100 mg/dL", status: "high", concern: "Significantly elevated — consistent with poorly controlled diabetes" },
      { test: "HbA1c", value: "7.8%", normal: "<5.7%", status: "high", concern: "Indicates average blood glucose has been high for past 2–3 months" },
      { test: "Serum Creatinine", value: "1.4 mg/dL", normal: "0.7–1.2 mg/dL", status: "high", concern: "Mildly elevated — early kidney stress, monitor closely" },
      { test: "CRP", value: "18 mg/L", normal: "<5 mg/L", status: "high", concern: "High inflammation marker — infection or autoimmune cause possible" },
      { test: "LDL Cholesterol", value: "158 mg/dL", normal: "<100 mg/dL", status: "high", concern: "Very high LDL significantly increases cardiovascular risk" },
      { test: "HDL Cholesterol", value: "38 mg/dL", normal: ">40 mg/dL", status: "low", concern: "Low HDL reduces protection against heart disease" },
    ],
    normalValues: ["RBC: 3.8 million/µL (borderline)", "Total Cholesterol: 234 mg/dL (borderline high)"],
    possibleConditions: [
      { condition: "Type 2 Diabetes (Uncontrolled)", likelihood: "high", evidence: "HbA1c 7.8%, fasting sugar 148 mg/dL — both well above normal thresholds" },
      { condition: "Anemia (possible Iron-deficiency or Chronic Disease)", likelihood: "high", evidence: "Low Hemoglobin 10.2, low RBC 3.8, low platelets 92,000 — multi-lineage involvement" },
      { condition: "Active Bacterial Infection / Sepsis risk", likelihood: "moderate", evidence: "WBC 13,400 and CRP 18 indicate significant inflammatory or infectious process" },
      { condition: "Early Chronic Kidney Disease (CKD Stage 1–2)", likelihood: "moderate", evidence: "Creatinine 1.4 in context of diabetes is a red flag for diabetic nephropathy" },
      { condition: "Dyslipidemia / Cardiovascular Risk", likelihood: "high", evidence: "LDL 158, HDL 38, Total Cholesterol 234 — classic atherogenic lipid profile" },
    ],
    recommendations: [
      "Consult an endocrinologist urgently for diabetes management — consider medication adjustment",
      "Repeat CBC and peripheral blood smear to investigate cause of anemia and thrombocytopenia",
      "Start a statin medication for LDL cholesterol under cardiologist guidance",
      "Monitor kidney function every 3 months — avoid NSAIDs and nephrotoxic drugs",
      "Strict diabetic diet: low carb, no refined sugar, portion control",
      "Follow up with doctor within 1–2 weeks given multiple abnormal findings",
      "Check iron studies, B12, and folate to determine anemia type",
    ]
  });
  const [tab, setTab] = useState("upload");
  const fileRef = useRef(null);

  const analyze = async (text) => {
    setLoading(true); setResult(null);
    try {
      const res = await callClaudeJSON(
        [{ role: "user", content: "Analyze this medical report and return JSON:\n\n" + text }],
        "You are a medical report analyzer. Given lab report data, return a JSON object with these exact keys: summary (string overview), overallStatus (normal/borderline/abnormal), urgency (routine/soon/urgent), abnormalValues (array of {test,value,normal,status,concern}), normalValues (array of strings), possibleConditions (array of {condition,likelihood,evidence}), recommendations (array of strings)."
      );
      setResult(res || { error: "Could not parse AI response. Please try again." });
    } catch(e) {
      setResult({ error: "Analysis failed: " + e.message });
    }
    setLoading(false);
  };

  const handleFile = (e) => {
    const f = e.target.files[0];
    if (!f) return;
    setFile(f);
    const reader = new FileReader();
    reader.onload = (ev) => {
      const text = ev.target.result;
      analyze(text.slice(0, 3000));
    };
    reader.readAsText(f);
  };

  const EXAMPLE_REPORT = `Patient: Rahul Verma | Age: 45 | Male | Date: 10-Jun-2026

COMPLETE BLOOD COUNT (CBC)
Hemoglobin: 10.2 g/dL         [Normal: 13.5-17.5]   ← LOW
WBC Count: 13,400 /µL          [Normal: 4500-11000]   ← HIGH
Platelets: 92,000 /µL          [Normal: 150000-400000] ← LOW
RBC: 3.8 million/µL            [Normal: 4.5-5.9]      ← LOW

BIOCHEMISTRY
Fasting Blood Sugar: 148 mg/dL [Normal: 70-100]       ← HIGH
HbA1c: 7.8%                    [Normal: <5.7%]        ← HIGH
Serum Creatinine: 1.4 mg/dL    [Normal: 0.7-1.2]     ← HIGH
CRP: 18 mg/L                   [Normal: <5]           ← HIGH

LIPID PROFILE
Total Cholesterol: 234 mg/dL   [Normal: <200]         ← HIGH
LDL: 158 mg/dL                 [Normal: <100]         ← HIGH
HDL: 38 mg/dL                  [Normal: >40]          ← LOW`;

  const loadExample = () => {
    setTab("manual");
    setManualText(EXAMPLE_REPORT);
    analyze(EXAMPLE_REPORT);
  };
  const statusColors = { normal: "badge-green", borderline: "badge-yellow", abnormal: "badge-red" };
  const urgColors = { routine: "badge-green", soon: "badge-yellow", urgent: "badge-red" };

  return (
    <div className="page-view">
      <div className="grid-2">
        <div className="card">
          <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 14 }}>
            <div className="card-title">📤 Upload Medical Report</div>
            <button className="btn btn-secondary btn-sm" onClick={loadExample} style={{ fontSize: 11, background: "rgba(245,158,11,0.1)", border: "1px solid rgba(245,158,11,0.35)", color: "#f59e0b" }}>
              ⚡ Try Example
            </button>
          </div>
          <div style={{ background: "rgba(245,158,11,0.07)", border: "1px solid rgba(245,158,11,0.2)", borderRadius: "var(--r2)", padding: "10px 13px", marginBottom: 14, fontSize: 11.5, color: "var(--text2)", lineHeight: 1.6 }}>
            <span style={{ color: "#f59e0b", fontWeight: 600 }}>📋 How it works:</span> Upload a blood report file <strong style={{ color: "var(--text1)" }}>OR</strong> switch to <strong style={{ color: "var(--text1)" }}>Manual Entry</strong> and paste report values → Click <strong style={{ color: "var(--text1)" }}>Analyze Report</strong> → AI detects abnormal values, possible conditions, and gives recommendations. Click <strong style={{ color: "var(--text1)" }}>⚡ Try Example</strong> to load a sample report instantly.
          </div>

          <div className="tabs" style={{ marginBottom: 16 }}>
            <div className={`tab ${tab === "upload" ? "active" : ""}`} onClick={() => setTab("upload")}>File Upload</div>
            <div className={`tab ${tab === "manual" ? "active" : ""}`} onClick={() => setTab("manual")}>Manual Entry</div>
          </div>

          {tab === "upload" ? (
            <>
              <div className="upload-zone" onClick={() => fileRef.current?.click()}>
                <div style={{ fontSize: 36, marginBottom: 10 }}>📄</div>
                <div style={{ fontWeight: 600, marginBottom: 6, fontSize: 13 }}>{file ? file.name : "Drop PDF, TXT, or image here"}</div>
                <div style={{ fontSize: 11.5, color: "var(--text3)" }}>Supports blood reports, scan reports, lab results</div>
                <input ref={fileRef} type="file" accept=".txt,.pdf,.csv" style={{ display: "none" }} onChange={handleFile} />
              </div>
              <div style={{ marginTop: 12, fontSize: 11, color: "var(--text3)" }}>
                📌 For demo: upload a .txt file with lab values like "Hemoglobin: 10.2 g/dL, WBC: 12000"
              </div>
            </>
          ) : (
            <>
              <label>Paste Report Values</label>
              <textarea
                rows={8}
                value={manualText}
                onChange={e => setManualText(e.target.value)}
                placeholder={"Paste report text here...\n\nExample:\nHemoglobin: 10.2 g/dL (Normal: 13.5-17.5)\nWBC: 12,000 /µL (Normal: 4,500-11,000)\nPlatelets: 95,000 /µL (Normal: 150,000-400,000)\nBlood Sugar (Fasting): 126 mg/dL\nCreatinine: 1.4 mg/dL"}
                style={{ marginBottom: 12, resize: "vertical" }}
              />
              <button className={`btn btn-primary ${loading ? "btn-loading" : ""}`} style={{ width: "100%", justifyContent: "center" }} onClick={() => manualText && analyze(manualText)}>
                {loading ? <><div className="spinner" /> Analyzing…</> : "🔍 Analyze Report"}
              </button>
            </>
          )}
        </div>

        <div className="card">
          <div className="card-title" style={{ marginBottom: 14 }}>📊 Analysis Results</div>

          {!result && !loading && (
            <div style={{ textAlign: "center", padding: "40px 20px", color: "var(--text3)" }}>
              <div style={{ fontSize: 36, marginBottom: 10 }}>🔬</div>
              <div style={{ fontSize: 13 }}>Upload or paste a report to see AI analysis</div>
            </div>
          )}

          {loading && (
            <div style={{ padding: "40px 20px", textAlign: "center" }}>
              <div className="spinner" style={{ width: 32, height: 32, margin: "0 auto 12px", borderWidth: 3 }} />
              <div style={{ color: "var(--text2)", fontSize: 13 }}>Analyzing your report with AI…</div>
            </div>
          )}

          {result && !result.error && (
            <div style={{ maxHeight: 460, overflowY: "auto" }} className="overflow-y">
              <div style={{ display: "flex", gap: 8, marginBottom: 14, flexWrap: "wrap" }}>
                <span className={`badge ${statusColors[result.overallStatus]}`}>Status: {result.overallStatus}</span>
                <span className={`badge ${urgColors[result.urgency]}`}>Urgency: {result.urgency}</span>
              </div>

              <div style={{ background: "var(--card2)", border: "1px solid var(--border)", borderRadius: "var(--r2)", padding: 12, marginBottom: 12, fontSize: 12.5, lineHeight: 1.6, color: "var(--text2)" }}>
                {result.summary}
              </div>

              {result.abnormalValues?.length > 0 && (
                <>
                  <div style={{ fontSize: 11, color: "var(--accent6)", fontWeight: 600, marginBottom: 8 }}>⚠️ Abnormal Values</div>
                  {result.abnormalValues.map((v, i) => (
                    <div key={i} style={{ background: "rgba(239,68,68,0.06)", border: "1px solid rgba(239,68,68,0.15)", borderRadius: "var(--r2)", padding: "10px 12px", marginBottom: 8 }}>
                      <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 4 }}>
                        <span style={{ fontWeight: 600, fontSize: 12.5 }}>{v.test}</span>
                        <span className={`badge ${v.status === "high" ? "badge-red" : "badge-yellow"}`}>{v.value} ({v.status})</span>
                      </div>
                      <div style={{ fontSize: 11, color: "var(--text3)" }}>Normal: {v.normal} • {v.concern}</div>
                    </div>
                  ))}
                </>
              )}

              {result.possibleConditions?.length > 0 && (
                <>
                  <div style={{ fontSize: 11, color: "var(--accent5)", fontWeight: 600, marginBottom: 8 }}>🔍 Possible Conditions</div>
                  {result.possibleConditions.map((c, i) => (
                    <div key={i} style={{ display: "flex", justifyContent: "space-between", alignItems: "center", padding: "8px 0", borderBottom: "1px solid var(--border)" }}>
                      <div>
                        <div style={{ fontSize: 12.5, fontWeight: 500 }}>{c.condition}</div>
                        <div style={{ fontSize: 11, color: "var(--text3)" }}>{c.evidence}</div>
                      </div>
                      <span className={`badge ${c.likelihood === "high" ? "badge-red" : c.likelihood === "moderate" ? "badge-yellow" : "badge-green"}`}>{c.likelihood}</span>
                    </div>
                  ))}
                </>
              )}

              {result.recommendations?.length > 0 && (
                <div style={{ marginTop: 12 }}>
                  <div style={{ fontSize: 11, color: "var(--accent4)", fontWeight: 600, marginBottom: 8 }}>✅ Recommendations</div>
                  {result.recommendations.map((r, i) => (
                    <div key={i} style={{ display: "flex", gap: 8, fontSize: 12, color: "var(--text2)", marginBottom: 6 }}>
                      <span style={{ color: "var(--accent4)" }}>→</span> {r}
                    </div>
                  ))}
                </div>
              )}
            </div>
          )}
          {result?.error && <div style={{ color: "var(--accent6)", fontSize: 13 }}>{result.error}</div>}
        </div>
      </div>
    </div>
  );
}

// ─── HEALTH MONITORING ────────────────────────────────────────────────────────
function HealthMonitoring({ user }) {
  const [bmi, setBmi] = useState({ weight: 70, height: 175 });
  const [lifestyle, setLifestyle] = useState({ sleep: 7, exercise: 3, smoking: "No", alcohol: "Occasionally", stress: "Moderate" });
  const [score, setScore] = useState(null);
  const [loading, setLoading] = useState(false);

  const bmiVal = (bmi.weight / ((bmi.height / 100) ** 2)).toFixed(1);
  const bmiStatus = bmiVal < 18.5 ? "Underweight" : bmiVal < 25 ? "Normal" : bmiVal < 30 ? "Overweight" : "Obese";
  const bmiColor = bmiVal < 18.5 ? "var(--accent3)" : bmiVal < 25 ? "var(--accent4)" : bmiVal < 30 ? "var(--accent5)" : "var(--accent6)";

  const loadExample = () => {
    const exBmi = { weight: 72, height: 175 };
    const exLifestyle = { sleep: 8, exercise: 5, smoking: "No", alcohol: "Occasionally", stress: "Low" };
    setBmi(exBmi);
    setLifestyle(exLifestyle);
    const exBmiVal = (exBmi.weight / ((exBmi.height / 100) ** 2)).toFixed(1);
    setLoading(true);
    callClaudeJSON([{ role: "user", content: JSON.stringify({ ...exLifestyle, bmi: exBmiVal, age: user.age, gender: user.gender }) }],
      `You are a health scoring AI. Return JSON:
{"score":75,"grade":"B","breakdown":{"diet":70,"exercise":80,"sleep":75,"stress":60},"insights":["insight1","insight2","insight3"],"riskFactors":["risk1","risk2"],"improvements":["tip1","tip2","tip3"]}`)
    .then(data => { setScore(data); setLoading(false); }).catch(() => setLoading(false));
  };

  const calculateScore = async () => {
    setLoading(true);
    const data = await callClaudeJSON([{ role: "user", content: JSON.stringify({ ...lifestyle, bmi: bmiVal, age: user.age, gender: user.gender }) }],
      `You are a health scoring AI. Return JSON:
{"score":75,"grade":"B","breakdown":{"diet":70,"exercise":80,"sleep":75,"stress":60},"insights":["insight1","insight2","insight3"],"riskFactors":["risk1","risk2"],"improvements":["tip1","tip2","tip3"]}`);
    setScore(data);
    setLoading(false);
  };

  const timeline = [
    { date: "2026-06-01", event: "Blood test results analyzed", sub: "Hemoglobin slightly low", color: "var(--accent5)" },
    { date: "2026-05-20", event: "BMI reassessed", sub: "Moved from Overweight → Normal", color: "var(--accent4)" },
    { date: "2026-05-10", event: "Symptom prediction run", sub: "Viral Fever, 87% confidence", color: "var(--accent)" },
    { date: "2026-04-28", event: "Health score calculated", sub: "Score: 78/100 — Good", color: "var(--accent2)" },
  ];

  return (
    <div className="page-view">
      <div className="grid-2">
        <div className="card">
          <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 14 }}>
            <div className="card-title">⚖️ BMI Calculator</div>
            <button className="btn btn-secondary btn-sm" onClick={loadExample} style={{ fontSize: 11, background: "rgba(245,158,11,0.1)", border: "1px solid rgba(245,158,11,0.35)", color: "#f59e0b" }}>
              ⚡ Try Example
            </button>
          </div>
          <div style={{ background: "rgba(245,158,11,0.07)", border: "1px solid rgba(245,158,11,0.2)", borderRadius: "var(--r2)", padding: "10px 13px", marginBottom: 14, fontSize: 11.5, color: "var(--text2)", lineHeight: 1.6 }}>
            <span style={{ color: "#f59e0b", fontWeight: 600 }}>📋 How it works:</span> Enter your weight &amp; height (BMI auto-calculates) → Fill lifestyle details like sleep, exercise, stress → Click <strong style={{ color: "var(--text1)" }}>Calculate Health Score</strong> → AI gives a score out of 100 with breakdown and improvement tips. Click <strong style={{ color: "var(--text1)" }}>⚡ Try Example</strong> to pre-fill with a healthy person's data.
          </div>
          <div className="grid-2" style={{ gap: 10, marginBottom: 16 }}>
            <div className="form-group" style={{ marginBottom: 0 }}>
              <label>Weight (kg)</label>
              <input type="number" value={bmi.weight} onChange={e => setBmi({ ...bmi, weight: +e.target.value })} />
            </div>
            <div className="form-group" style={{ marginBottom: 0 }}>
              <label>Height (cm)</label>
              <input type="number" value={bmi.height} onChange={e => setBmi({ ...bmi, height: +e.target.value })} />
            </div>
          </div>
          <div style={{ background: "var(--card2)", border: "1px solid var(--border)", borderRadius: "var(--r2)", padding: 16, textAlign: "center" }}>
            <div style={{ fontSize: 40, fontWeight: 800, color: bmiColor }}>{bmiVal}</div>
            <div style={{ fontSize: 13, color: "var(--text2)", marginTop: 4 }}>BMI — <span style={{ color: bmiColor, fontWeight: 600 }}>{bmiStatus}</span></div>
            <div style={{ display: "flex", justifyContent: "center", gap: 8, marginTop: 10, flexWrap: "wrap" }}>
              {[["<18.5","Underweight","var(--accent3)"], ["18.5–24.9","Normal","var(--accent4)"], ["25–29.9","Overweight","var(--accent5)"], ["≥30","Obese","var(--accent6)"]].map(([r, l, c]) => (
                <span key={l} className="chip" style={{ color: c, borderColor: c, background: "transparent" }}>{r} {l}</span>
              ))}
            </div>
          </div>

          <div className="divider" />
          <div className="card-title" style={{ marginBottom: 12 }}>🏃 Lifestyle Assessment</div>
          <div className="grid-2" style={{ gap: 10 }}>
            {[["sleep","Sleep (hrs/night)","number"],["exercise","Exercise (days/week)","number"]].map(([k, l, t]) => (
              <div key={k} className="form-group" style={{ marginBottom: 0 }}>
                <label>{l}</label>
                <input type={t} value={lifestyle[k]} onChange={e => setLifestyle({ ...lifestyle, [k]: +e.target.value })} />
              </div>
            ))}
            {[["smoking","Smoking","No,Occasionally,Yes"],["alcohol","Alcohol","No,Occasionally,Regularly"],["stress","Stress","Low,Moderate,High"]].map(([k, l, opts]) => (
              <div key={k} className="form-group" style={{ marginBottom: 0 }}>
                <label>{l}</label>
                <select value={lifestyle[k]} onChange={e => setLifestyle({ ...lifestyle, [k]: e.target.value })}>
                  {opts.split(",").map(o => <option key={o}>{o}</option>)}
                </select>
              </div>
            ))}
          </div>
          <button className={`btn btn-primary ${loading ? "btn-loading" : ""}`} style={{ width: "100%", justifyContent: "center", marginTop: 14 }} onClick={calculateScore}>
            {loading ? <><div className="spinner" /> Calculating…</> : "❤️ Calculate Health Score"}
          </button>
        </div>

        <div>
          {score && (
            <div className="card" style={{ marginBottom: 16 }}>
              <div className="card-title" style={{ marginBottom: 14 }}>❤️ Personal Health Score</div>
              <div style={{ display: "flex", gap: 20, alignItems: "center", marginBottom: 16 }}>
                <div style={{ position: "relative", width: 90, height: 90, flexShrink: 0 }}>
                  <svg viewBox="0 0 90 90" width="90" height="90">
                    <circle cx="45" cy="45" r="38" fill="none" stroke="rgba(79,124,255,0.1)" strokeWidth="10" />
                    <circle cx="45" cy="45" r="38" fill="none" stroke="var(--accent)" strokeWidth="10"
                      strokeDasharray={`${(score.score / 100) * 239} 239`} strokeDashoffset="60" strokeLinecap="round" />
                  </svg>
                  <div style={{ position: "absolute", inset: 0, display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center" }}>
                    <div style={{ fontSize: 20, fontWeight: 800 }}>{score.score}</div>
                    <div style={{ fontSize: 9, color: "var(--text3)" }}>GRADE {score.grade}</div>
                  </div>
                </div>
                <div>
                  {score.breakdown && Object.entries(score.breakdown).map(([k, v]) => (
                    <div key={k} style={{ marginBottom: 8 }}>
                      <div style={{ display: "flex", justifyContent: "space-between", fontSize: 11.5, marginBottom: 3 }}>
                        <span style={{ color: "var(--text2)", textTransform: "capitalize" }}>{k}</span>
                        <span style={{ color: "var(--text1)", fontFamily: "var(--mono)" }}>{v}</span>
                      </div>
                      <div className="progress-wrap">
                        <div className="progress-bar" style={{ width: `${v}%`, background: v >= 70 ? "var(--accent4)" : v >= 50 ? "var(--accent5)" : "var(--accent6)" }} />
                      </div>
                    </div>
                  ))}
                </div>
              </div>

              {score.improvements?.length > 0 && (
                <div style={{ background: "rgba(79,124,255,0.06)", border: "1px solid var(--border)", borderRadius: "var(--r2)", padding: 12 }}>
                  <div style={{ fontSize: 11, color: "var(--accent)", fontWeight: 600, marginBottom: 8 }}>💡 Improvement Tips</div>
                  {score.improvements.map((t, i) => (
                    <div key={i} style={{ fontSize: 12, color: "var(--text2)", marginBottom: 5 }}>→ {t}</div>
                  ))}
                </div>
              )}
            </div>
          )}

          <div className="card">
            <div className="card-title" style={{ marginBottom: 14 }}>📈 Health Journey Timeline</div>
            <div className="timeline">
              {timeline.map((t, i) => (
                <div key={i} className="tl-item">
                  <div className="tl-dot" style={{ borderColor: t.color }} />
                  <div className="tl-date">{t.date}</div>
                  <div className="tl-text">{t.event}</div>
                  <div className="tl-sub">{t.sub}</div>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

// ─── PREVENTION SIMULATOR ─────────────────────────────────────────────────────
function Prevention({ user }) {
  const [form, setForm] = useState({ age: user.age || 30, gender: user.gender || "Male", smoking: "No", alcohol: "No", exercise: "Moderate", diet: "Balanced", stress: "Moderate", bmi: "Normal", familyHistory: "" });
  const [result, setResult] = useState(null);
  const [loading, setLoading] = useState(false);

  const PREVENTION_PROMPT = `You are a preventive health AI. Return ONLY valid JSON, no extra text:
{"riskScore":35,"riskLevel":"moderate","futureRisks":[{"disease":"Type 2 Diabetes","5yr":18,"10yr":34,"description":"High sugar diet and low exercise increase risk"},{"disease":"Hypertension","5yr":22,"10yr":40,"description":"Stress and weight contribute significantly"},{"disease":"Heart Disease","5yr":12,"10yr":25,"description":"Linked to cholesterol and sedentary lifestyle"}],"dietPlan":["Reduce refined sugar and processed foods","Increase vegetables, fruits, and whole grains","Drink 8-10 glasses of water daily","Limit sodium intake to reduce BP risk"],"fitnessPlan":["30 min brisk walk 5 days/week","Add strength training 2 days/week","Yoga or stretching for stress relief"],"lifestyleChanges":["Quit or reduce smoking","Limit alcohol to weekends only","Practice 10 min meditation daily"],"preventionSteps":["Annual blood sugar and BP checkup","Monitor BMI monthly","Sleep 7-8 hours consistently"]}`;

  const loadExample = () => {
    const exampleForm = { age: 45, gender: "Male", smoking: "Occasionally", alcohol: "Occasionally", exercise: "Low", diet: "Average", stress: "High", bmi: "Overweight", familyHistory: "Diabetes, Hypertension" };
    setForm(exampleForm);
    setLoading(true); setResult(null);
    callClaudeJSON([{ role: "user", content: `Analyze prevention risk for: ${JSON.stringify(exampleForm)}` }], PREVENTION_PROMPT)
      .then(data => { setResult(data || {}); setLoading(false); })
      .catch(() => setLoading(false));
  };

  const simulate = async () => {
    setLoading(true); setResult(null);
    try {
      const data = await callClaudeJSON([{ role: "user", content: `Analyze prevention risk for: ${JSON.stringify(form)}` }], PREVENTION_PROMPT);
      setResult(data || {});
    } catch { setResult({}); }
    setLoading(false);
  };

  return (
    <div className="page-view">
      <div className="grid-2">
        <div className="card">
          <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 14 }}>
            <div className="card-title">🛡 Prevention Simulator</div>
            <button className="btn btn-secondary btn-sm" onClick={loadExample} style={{ fontSize: 11, background: "rgba(245,158,11,0.1)", border: "1px solid rgba(245,158,11,0.35)", color: "#f59e0b" }}>
              ⚡ Try Example
            </button>
          </div>
          <div style={{ background: "rgba(245,158,11,0.07)", border: "1px solid rgba(245,158,11,0.2)", borderRadius: "var(--r2)", padding: "10px 13px", marginBottom: 14, fontSize: 11.5, color: "var(--text2)", lineHeight: 1.6 }}>
            <span style={{ color: "#f59e0b", fontWeight: 600 }}>📋 How it works:</span> Fill your current lifestyle (smoking, exercise, diet, stress etc.) → Click <strong style={{ color: "var(--text1)" }}>Simulate Future Health Risks</strong> → AI shows your current risk %, future disease risks for 5 &amp; 10 years, and a personalised diet + fitness prevention plan. Click <strong style={{ color: "var(--text1)" }}>⚡ Try Example</strong> to load a high-risk profile demo.
          </div>
          <div className="grid-2" style={{ gap: 10 }}>
            {[["age","Age","number"],["gender","Gender","select:Male,Female,Other"],["bmi","BMI Status","select:Underweight,Normal,Overweight,Obese"],["smoking","Smoking","select:No,Occasionally,Daily"],["alcohol","Alcohol","select:No,Occasionally,Regularly"],["exercise","Exercise Level","select:Sedentary,Low,Moderate,High"],["diet","Diet Quality","select:Poor,Average,Balanced,Excellent"],["stress","Stress Level","select:Low,Moderate,High,Severe"]].map(([k, l, t]) => (
              <div key={k} className="form-group" style={{ marginBottom: 0 }}>
                <label>{l}</label>
                {t.startsWith("select:") ? (
                  <select value={form[k]} onChange={e => setForm({ ...form, [k]: e.target.value })}>
                    {t.slice(7).split(",").map(o => <option key={o}>{o}</option>)}
                  </select>
                ) : (
                  <input type={t} value={form[k]} onChange={e => setForm({ ...form, [k]: e.target.value })} />
                )}
              </div>
            ))}
          </div>
          <div className="form-group" style={{ marginTop: 10 }}>
            <label>Family Medical History (optional)</label>
            <input placeholder="e.g. diabetes, hypertension" value={form.familyHistory} onChange={e => setForm({ ...form, familyHistory: e.target.value })} />
          </div>
          <button className={`btn btn-primary ${loading ? "btn-loading" : ""}`} style={{ width: "100%", justifyContent: "center", marginTop: 10 }} onClick={simulate}>
            {loading ? <><div className="spinner" /> Running simulation…</> : "🔮 Simulate Future Health Risks"}
          </button>
        </div>

        <div className="card" style={{ minHeight: 300 }}>
          <div className="card-title" style={{ marginBottom: 14 }}>🔮 Simulation Results</div>
          {!result && !loading && (
            <div style={{ textAlign: "center", padding: "30px 20px", color: "var(--text3)" }}>
              <div style={{ fontSize: 36, marginBottom: 10 }}>🔮</div>
              <div style={{ fontSize: 13, marginBottom: 12 }}>Fill your profile and click Simulate, or click ⚡ Try Example</div>
              <div style={{ background: "rgba(79,124,255,0.07)", border: "1px solid rgba(79,124,255,0.18)", borderRadius: "var(--r2)", padding: "12px 14px", textAlign: "left", fontSize: 12, color: "var(--text2)", lineHeight: 1.7 }}>
                <div style={{ color: "var(--accent)", fontWeight: 600, marginBottom: 6 }}>👁 Example Output Preview:</div>
                <div>📊 <strong style={{ color: "var(--text1)" }}>Overall Risk Score:</strong> 42% — Moderate</div>
                <div style={{ marginTop: 4 }}>⚠️ <strong style={{ color: "var(--text1)" }}>Diabetes</strong> — 5yr: 18% | 10yr: 34%</div>
                <div style={{ marginTop: 4 }}>🥗 Personalised Diet Plan + 🏋️ Fitness Plan</div>
                <div style={{ marginTop: 4 }}>🛡 Prevention steps to reduce future risk</div>
              </div>
            </div>
          )}
          {loading && (
            <div style={{ padding: "40px 20px", textAlign: "center" }}>
              <div className="spinner" style={{ width: 32, height: 32, margin: "0 auto 12px", borderWidth: 3 }} />
              <div style={{ color: "var(--text2)", fontSize: 13 }}>Running prevention simulation…</div>
            </div>
          )}
          {result && (
            <div style={{ display: "flex", flexDirection: "column", gap: 14 }}>
              <div>
                <div style={{ display: "flex", alignItems: "center", gap: 14, marginBottom: 14 }}>
                  <div style={{ textAlign: "center" }}>
                    <div style={{ fontSize: 32, fontWeight: 800, color: result.riskLevel === "high" ? "var(--accent6)" : result.riskLevel === "moderate" ? "var(--accent5)" : "var(--accent4)" }}>{result.riskScore}%</div>
                    <div style={{ fontSize: 10, color: "var(--text3)" }}>OVERALL RISK</div>
                  </div>
                  <div style={{ flex: 1 }}>
                    <div style={{ fontWeight: 600, fontSize: 13 }}>Risk Level: <span style={{ color: result.riskLevel === "high" ? "var(--accent6)" : result.riskLevel === "moderate" ? "var(--accent5)" : "var(--accent4)" }}>{result.riskLevel}</span></div>
                    <div style={{ fontSize: 12, color: "var(--text3)", marginTop: 4 }}>Based on your current lifestyle profile</div>
                  </div>
                </div>
                {result.futureRisks?.map((r, i) => (
                  <div key={i} style={{ background: "var(--card2)", border: "1px solid var(--border)", borderRadius: "var(--r2)", padding: "10px 12px", marginBottom: 8 }}>
                    <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 6, alignItems: "center" }}>
                      <span style={{ fontWeight: 600, fontSize: 12.5 }}>{r.disease}</span>
                      <div style={{ display: "flex", gap: 6 }}>
                        <span className="badge badge-yellow">5yr: {r["5yr"]}%</span>
                        <span className="badge badge-red">10yr: {r["10yr"]}%</span>
                      </div>
                    </div>
                    <div style={{ fontSize: 11, color: "var(--text3)" }}>{r.description}</div>
                  </div>
                ))}
              </div>

              <div className="grid-2" style={{ gap: 14 }}>
                <div className="card-sm">
                  <div style={{ fontSize: 11, color: "var(--accent4)", fontWeight: 600, marginBottom: 10 }}>🥗 Diet Plan</div>
                  {result.dietPlan?.map((t, i) => <div key={i} style={{ fontSize: 12, color: "var(--text2)", marginBottom: 6 }}>→ {t}</div>)}
                </div>
                <div className="card-sm">
                  <div style={{ fontSize: 11, color: "var(--accent3)", fontWeight: 600, marginBottom: 10 }}>🏋️ Fitness Plan</div>
                  {result.fitnessPlan?.map((t, i) => <div key={i} style={{ fontSize: 12, color: "var(--text2)", marginBottom: 6 }}>→ {t}</div>)}
                </div>
              </div>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}

// ─── HOSPITALS ────────────────────────────────────────────────────────────────
function Hospitals() {
  const [search, setSearch] = useState("");
  const filtered = MOCK_HOSPITALS.filter(h => h.name.toLowerCase().includes(search.toLowerCase()));

  return (
    <div className="page-view">
      <div className="card" style={{ marginBottom: 16 }}>
        <div style={{ display: "flex", gap: 10, alignItems: "center" }}>
          <div className="search-bar" style={{ flex: 1 }}>
            <span className="search-icon">🔍</span>
            <input placeholder="Search hospitals…" value={search} onChange={e => setSearch(e.target.value)} />
          </div>
          <span className="badge badge-red">🚨 Emergency: 112</span>
          <span className="badge badge-blue">🚑 Ambulance: 108</span>
        </div>
      </div>

      <div className="grid-2">
        {filtered.map(h => (
          <div key={h.id} className="card">
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-start", marginBottom: 12 }}>
              <div>
                <div style={{ fontWeight: 700, fontSize: 14, color: "var(--text1)", marginBottom: 4 }}>{h.name}</div>
                <div style={{ fontSize: 11.5, color: "var(--text2)" }}>📍 {h.distance} away</div>
              </div>
              <span className={`badge ${h.type === "Government" ? "badge-green" : "badge-blue"}`}>{h.type}</span>
            </div>
            <div style={{ display: "flex", gap: 8, marginBottom: 12, flexWrap: "wrap" }}>
              {h.emergency && <span className="badge badge-red">🚨 Emergency</span>}
              <span className="badge badge-yellow">★ {h.rating}</span>
            </div>
            <div style={{ display: "flex", gap: 8 }}>
              <button className="btn btn-primary btn-sm" style={{ flex: 1, justifyContent: "center" }}>📞 {h.phone}</button>
              <button className="btn btn-secondary btn-sm" onClick={() => window.open(`https://www.google.com/maps/search/${encodeURIComponent(h.name + ' ' + h.area)}/@${h.lat},${h.lng},15z`, '_blank')}>📍 Directions</button>
            </div>
          </div>
        ))}
      </div>

      <div className="card">
        <div className="card-title" style={{ marginBottom: 14 }}>🚨 Emergency Contacts</div>
        <div className="grid-4">
          {[["108","Ambulance","var(--accent6)","🚑"], ["112","Emergency","var(--accent6)","🆘"], ["1091","Women Safety","var(--accent7)","🛡"], ["104","Health Helpline","var(--accent4)","📞"]].map(([num, label, color, icon]) => (
            <div key={num} style={{ background: "var(--card2)", border: `1px solid ${color}33`, borderRadius: "var(--r2)", padding: 16, textAlign: "center" }}>
              <div style={{ fontSize: 22, marginBottom: 6 }}>{icon}</div>
              <div style={{ fontSize: 22, fontWeight: 800, color }}>{num}</div>
              <div style={{ fontSize: 11, color: "var(--text3)", marginTop: 2 }}>{label}</div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

// ─── COMMUNITY ALERTS ─────────────────────────────────────────────────────────
function Community() {
  const [aiAnalysis, setAiAnalysis] = useState(null);
  const [loading, setLoading] = useState(false);

  const heatCells = Array.from({ length: 35 }, () => Math.random());

  const runAnalysis = async () => {
    setLoading(true);
    const data = await callClaude(
      [{ role: "user", content: `Analyze these community disease alerts for Jabalpur: ${JSON.stringify(COMMUNITY_ALERTS)}. Provide outbreak risk assessment and public health recommendations.` }],
      "You are a public health AI analyst. Provide a brief, actionable community health briefing in 3-4 sentences covering outbreak risks, affected areas, and recommendations. Be direct and informative."
    );
    setAiAnalysis(data);
    setLoading(false);
  };

  return (
    <div className="page-view">
      <div className="grid-2">
        <div className="card">
          <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 14 }}>
            <div className="card-title">🚨 Community Disease Alerts</div>
            <span className="badge badge-red">● Live</span>
          </div>
          {COMMUNITY_ALERTS.map(a => (
            <div key={a.id} className="alert-row">
              <div className="alert-icon" style={{ background: a.severity === "high" ? "rgba(239,68,68,0.12)" : "rgba(245,158,11,0.1)" }}>
                {a.severity === "high" ? "🔴" : "🟡"}
              </div>
              <div style={{ flex: 1 }}>
                <div style={{ fontWeight: 600, fontSize: 12.5 }}>{a.disease}</div>
                <div style={{ fontSize: 11, color: "var(--text3)" }}>{a.area} • {a.cases} cases • {a.date}</div>
              </div>
              <div style={{ display: "flex", flexDirection: "column", gap: 4, alignItems: "flex-end" }}>
                <span className={`badge ${a.severity === "high" ? "badge-red" : "badge-yellow"}`}>{a.severity}</span>
                <span className={`badge ${a.trend === "rising" ? "badge-red" : a.trend === "declining" ? "badge-green" : "badge-yellow"}`}>{a.trend}</span>
              </div>
            </div>
          ))}

          <button className={`btn btn-secondary ${loading ? "btn-loading" : ""}`} style={{ width: "100%", justifyContent: "center", marginTop: 12 }} onClick={runAnalysis}>
            {loading ? <><div className="spinner" style={{ width: 12, height: 12 }} /> Analyzing…</> : "🤖 AI Outbreak Analysis"}
          </button>

          {aiAnalysis && (
            <div style={{ background: "rgba(139,92,246,0.07)", border: "1px solid rgba(139,92,246,0.2)", borderRadius: "var(--r2)", padding: 12, marginTop: 12, fontSize: 12.5, lineHeight: 1.7, color: "var(--text1)" }}>
              <div style={{ fontSize: 10, color: "var(--accent2)", fontWeight: 600, marginBottom: 6 }}>🤖 AI ANALYSIS</div>
              {aiAnalysis}
            </div>
          )}
        </div>

        <div className="card">
          <div className="card-title" style={{ marginBottom: 14 }}>🗺 Disease Heatmap — Jabalpur</div>
          <div className="heatmap" style={{ marginBottom: 12 }}>
            {heatCells.map((v, i) => {
              const color = v < 0.2 ? "rgba(79,124,255,0.15)" : v < 0.4 ? "rgba(79,124,255,0.3)" : v < 0.6 ? "rgba(245,158,11,0.4)" : v < 0.8 ? "rgba(239,68,68,0.5)" : "rgba(239,68,68,0.85)";
              return <div key={i} className="hm-cell" style={{ background: color }} title={`${Math.round(v * 20)} cases`} />;
            })}
          </div>
          <div style={{ display: "flex", gap: 10, flexWrap: "wrap", marginTop: 8 }}>
            {[["rgba(79,124,255,0.3)","Low"], ["rgba(245,158,11,0.4)","Moderate"], ["rgba(239,68,68,0.6)","High"], ["rgba(239,68,68,0.9)","Critical"]].map(([c, l]) => (
              <div key={l} style={{ display: "flex", alignItems: "center", gap: 5, fontSize: 11, color: "var(--text2)" }}>
                <span style={{ width: 12, height: 12, borderRadius: 3, background: c, display: "inline-block" }} /> {l}
              </div>
            ))}
          </div>

          <div className="divider" />
          <div className="card-title" style={{ marginBottom: 12 }}>📊 Local Health Trends</div>
          {[["Dengue cases this week", "+34%", "var(--accent6)"], ["Vaccination coverage", "67%", "var(--accent4)"], ["Hospital bed occupancy", "78%", "var(--accent5)"], ["Outbreak risk index", "Medium", "var(--accent5)"]].map(([l, v, c]) => (
            <div key={l} style={{ display: "flex", justifyContent: "space-between", padding: "8px 0", borderBottom: "1px solid rgba(99,140,255,0.05)", fontSize: 12.5 }}>
              <span style={{ color: "var(--text2)" }}>{l}</span>
              <span style={{ color: c, fontWeight: 600 }}>{v}</span>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

// ─── ANALYTICS DASHBOARD ──────────────────────────────────────────────────────
function Analytics() {
  const diseases = [["Viral Fever", 32, "var(--accent)"], ["Dengue", 18, "var(--accent6)"], ["Diabetes", 14, "var(--accent5)"], ["Hypertension", 11, "var(--accent2)"], ["Typhoid", 9, "var(--accent3)"], ["Influenza", 8, "var(--accent4)"], ["Malaria", 5, "var(--accent7)"], ["COVID-19", 3, "var(--accent)"]];
  const months = ["Jan","Feb","Mar","Apr","May","Jun"];
  const predictions = [124, 189, 213, 198, 267, 301];

  return (
    <div className="page-view">
      <div className="grid-4" style={{ marginBottom: 20 }}>
        {[["🎯", "Avg Accuracy", "84.2%", "+2.1%"], ["📊", "Predictions", "2,847", "+14%"], ["✅", "Correct", "2,394", "84.1%"], ["❌", "Reviewed", "453", "15.9%"]].map(([i, l, v, d]) => (
          <div key={l} className="stat-card">
            <div className="stat-icon">{i}</div>
            <div className="stat-val">{v}</div>
            <div className="stat-label">{l}</div>
            <div className="stat-delta up">↑ {d}</div>
          </div>
        ))}
      </div>

      <div className="grid-2">
        <div className="card">
          <div className="card-title" style={{ marginBottom: 14 }}>📊 Monthly Predictions</div>
          <svg viewBox="0 0 400 120" style={{ width: "100%", height: 120 }}>
            {predictions.map((v, i) => {
              const h = (v / 350) * 90;
              const x = i * 68 + 14;
              return (
                <g key={i}>
                  <rect x={x} y={100 - h} width={40} height={h} rx="4" fill={`rgba(79,124,255,${0.4 + i * 0.1})`} />
                  <text x={x + 20} y={115} fill="#3a4a66" fontSize="9" textAnchor="middle" fontFamily="Sora">{months[i]}</text>
                  <text x={x + 20} y={100 - h - 4} fill="#7888aa" fontSize="9" textAnchor="middle" fontFamily="Sora">{v}</text>
                </g>
              );
            })}
          </svg>
        </div>

        <div className="card">
          <div className="card-title" style={{ marginBottom: 14 }}>🎯 Top Conditions Distribution</div>
          {diseases.map(([name, pct, color]) => (
            <div key={name} style={{ marginBottom: 10 }}>
              <div style={{ display: "flex", justifyContent: "space-between", fontSize: 12, marginBottom: 4 }}>
                <span style={{ color: "var(--text2)" }}>{name}</span>
                <span style={{ color, fontWeight: 700, fontFamily: "var(--mono)" }}>{pct}%</span>
              </div>
              <div className="progress-wrap">
                <div className="progress-bar" style={{ width: `${pct * 3}%`, background: color }} />
              </div>
            </div>
          ))}
        </div>
      </div>

      <div className="grid-2">
        <div className="card">
          <div className="card-title" style={{ marginBottom: 14 }}>👥 Age-wise Distribution</div>
          <svg viewBox="0 0 400 120" style={{ width: "100%", height: 120 }}>
            {[["0–10",20],["11–20",52],["21–35",100],["36–50",80],["51–65",55],["65+",25]].map(([label, h], i) => {
              const x = i * 66 + 8;
              const pct = (h / 100) * 90;
              return (
                <g key={i}>
                  <rect x={x} y={100 - pct} width={50} height={pct} rx="4" fill={`rgba(79,124,255,${0.3 + (h / 140)})`} />
                  <text x={x + 25} y={115} fill="#3a4a66" fontSize="8.5" textAnchor="middle" fontFamily="Sora">{label}</text>
                </g>
              );
            })}
          </svg>
        </div>

        <div className="card">
          <div className="card-title" style={{ marginBottom: 16 }}>⚧ Gender Distribution</div>
          <div style={{ display: "flex", alignItems: "center", gap: 24, justifyContent: "center" }}>
            <svg viewBox="0 0 100 100" width={110} height={110}>
              <circle cx={50} cy={50} r={38} fill="none" stroke="rgba(79,124,255,0.1)" strokeWidth={14} />
              {/* Male 62% = 62% of circumference = 0.62 * 2π * 38 ≈ 148.1 */}
              <circle cx={50} cy={50} r={38} fill="none" stroke="var(--accent)" strokeWidth={14}
                strokeDasharray="148 239" strokeDashoffset="60" strokeLinecap="butt" />
              {/* Female 38% = 38% of circumference ≈ 90.7, offset = -(239-148) + 60 = -31 */}
              <circle cx={50} cy={50} r={38} fill="none" stroke="var(--accent7)" strokeWidth={14}
                strokeDasharray="90 239" strokeDashoffset="-88" strokeLinecap="butt" />
            </svg>
            <div>
              {[["var(--accent)","Male","62%"],["var(--accent7)","Female","38%"]].map(([c,l,v]) => (
                <div key={l} style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 10 }}>
                  <span style={{ width: 12, height: 12, borderRadius: "50%", background: c, display: "inline-block" }} />
                  <span style={{ fontSize: 13, color: "var(--text2)" }}>{l}</span>
                  <span style={{ fontSize: 16, fontWeight: 800, color: c, marginLeft: 4 }}>{v}</span>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

// ─── USER MANAGEMENT (ADMIN) ──────────────────────────────────────────────────
function UserManagement() {
  const [search, setSearch] = useState("");
  const filtered = MOCK_USERS.filter(u => u.name.toLowerCase().includes(search.toLowerCase()) || u.email.toLowerCase().includes(search.toLowerCase()));

  return (
    <div className="page-view">
      <div className="card" style={{ marginBottom: 16 }}>
        <div style={{ display: "flex", gap: 10, alignItems: "center" }}>
          <div className="search-bar" style={{ flex: 1 }}>
            <span className="search-icon">🔍</span>
            <input placeholder="Search users…" value={search} onChange={e => setSearch(e.target.value)} />
          </div>
          <button className="btn btn-primary btn-sm">＋ Add User</button>
          <button className="btn btn-secondary btn-sm">⬇ Export</button>
        </div>
      </div>

      <div className="card">
        <div className="table-wrap">
          <table>
            <thead>
              <tr>
                <th>User</th><th>Age / Gender</th><th>Blood Group</th><th>Health Score</th><th>Predictions</th><th>Status</th><th>Actions</th>
              </tr>
            </thead>
            <tbody>
              {filtered.map(u => (
                <tr key={u.id}>
                  <td>
                    <div style={{ display: "flex", alignItems: "center", gap: 9 }}>
                      <div style={{ width: 30, height: 30, borderRadius: "50%", background: "linear-gradient(135deg, var(--accent), var(--accent2))", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 11, fontWeight: 700, flexShrink: 0 }}>
                        {u.name.slice(0, 2).toUpperCase()}
                      </div>
                      <div>
                        <div style={{ fontSize: 12.5, fontWeight: 600, color: "var(--text1)" }}>{u.name}</div>
                        <div style={{ fontSize: 11, color: "var(--text3)" }}>{u.email}</div>
                      </div>
                    </div>
                  </td>
                  <td>{u.age}yr / {u.gender}</td>
                  <td><span className="badge badge-blue">{u.bloodGroup}</span></td>
                  <td>
                    <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                      <div className="progress-wrap" style={{ width: 60 }}><div className="progress-bar" style={{ width: `${u.healthScore}%`, background: u.healthScore >= 80 ? "var(--accent4)" : u.healthScore >= 60 ? "var(--accent5)" : "var(--accent6)" }} /></div>
                      <span style={{ fontSize: 12, fontFamily: "var(--mono)" }}>{u.healthScore}</span>
                    </div>
                  </td>
                  <td><span style={{ fontFamily: "var(--mono)", fontSize: 12 }}>{u.predictions}</span></td>
                  <td><span className="badge badge-green">● Active</span></td>
                  <td>
                    <div style={{ display: "flex", gap: 6 }}>
                      <button className="btn-icon" title="Edit">✏</button>
                      <button className="btn-icon" title="View">👁</button>
                      <button className="btn-icon" style={{ color: "var(--accent6)" }} title="Delete">🗑</button>
                    </div>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </div>
    </div>
  );
}

// ─── PROFILE ──────────────────────────────────────────────────────────────────
function Profile({ user, onUpdate }) {
  const [form, setForm] = useState({ ...user });
  const [saved, setSaved] = useState(false);

  const save = async () => {
    await new Promise(r => setTimeout(r, 400));
    onUpdate(form);
    setSaved(true);
    setTimeout(() => setSaved(false), 2000);
  };

  return (
    <div className="page-view">
      <div className="grid-2">
        <div className="card">
          <div style={{ textAlign: "center", marginBottom: 20 }}>
            <div style={{ width: 70, height: 70, borderRadius: "50%", background: "linear-gradient(135deg, var(--accent), var(--accent2))", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 26, fontWeight: 700, margin: "0 auto 12px" }}>
              {user.name.slice(0, 2).toUpperCase()}
            </div>
            <div style={{ fontWeight: 700, fontSize: 16 }}>{user.name}</div>
            <div style={{ fontSize: 12, color: "var(--text3)" }}>{user.email}</div>
            <span className={`badge ${user.role === "admin" ? "badge-purple" : "badge-blue"}`} style={{ marginTop: 8 }}>{user.role === "admin" ? "👑 Admin" : "👤 User"}</span>
          </div>
          <div className="divider" />
          <div className="card-title" style={{ marginBottom: 14 }}>Edit Profile</div>
          {[["name","Full Name","text"],["email","Email","email"],["age","Age","number"]].map(([k, l, t]) => (
            <div key={k} className="form-group">
              <label>{l}</label>
              <input type={t} value={form[k] || ""} onChange={e => setForm({ ...form, [k]: e.target.value })} />
            </div>
          ))}
          <div className="form-group">
            <label>Gender</label>
            <select value={form.gender || "Male"} onChange={e => setForm({ ...form, gender: e.target.value })}>
              <option>Male</option><option>Female</option><option>Other</option>
            </select>
          </div>
          <div className="form-group">
            <label>Blood Group</label>
            <select value={form.bloodGroup || "O+"} onChange={e => setForm({ ...form, bloodGroup: e.target.value })}>
              {["A+","A-","B+","B-","O+","O-","AB+","AB-"].map(b => <option key={b}>{b}</option>)}
            </select>
          </div>
          <button className="btn btn-primary" style={{ width: "100%", justifyContent: "center" }} onClick={save}>
            {saved ? "✓ Saved!" : "Save Changes"}
          </button>
        </div>

        <div>
          <div className="card" style={{ marginBottom: 16 }}>
            <div className="card-title" style={{ marginBottom: 14 }}>📋 Medical History</div>
            <div className="form-group">
              <label>Known Conditions</label>
              <textarea rows={3} placeholder="e.g. Type 2 Diabetes (2018), Hypertension (2021)" style={{ resize: "vertical" }} />
            </div>
            <div className="form-group">
              <label>Current Medications</label>
              <textarea rows={3} placeholder="e.g. Metformin 500mg daily, Amlodipine 5mg" style={{ resize: "vertical" }} />
            </div>
            <div className="form-group">
              <label>Allergies</label>
              <input placeholder="e.g. Penicillin, Sulfa drugs" />
            </div>
            <button className="btn btn-secondary" style={{ width: "100%", justifyContent: "center" }}>Save Medical History</button>
          </div>

          <div className="card">
            <div className="card-title" style={{ marginBottom: 14 }}>📊 Quick Stats</div>
            <div className="grid-2" style={{ gap: 10 }}>
              {[["🔬", "Predictions Run", user.predictions || 0, "var(--accent)"], ["❤️", "Health Score", `${user.healthScore || 75}/100`, "var(--accent4)"], ["📋", "Reports Analyzed", "3", "var(--accent3)"], ["📅", "Member Since", "Jun 2026", "var(--accent2)"]].map(([ic, l, v, c]) => (
                <div key={l} style={{ background: "var(--card2)", borderRadius: "var(--r2)", padding: 12, border: "1px solid var(--border)" }}>
                  <div style={{ fontSize: 18, marginBottom: 4 }}>{ic}</div>
                  <div style={{ fontSize: 18, fontWeight: 800, color: c }}>{v}</div>
                  <div style={{ fontSize: 10.5, color: "var(--text3)", marginTop: 2 }}>{l}</div>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

// ─── MAIN APP ─────────────────────────────────────────────────────────────────
const NAV = [
  { id: "dashboard", icon: "⊞", label: "Dashboard", section: "MAIN" },
  { id: "predict", icon: "🔬", label: "Disease Prediction", section: "MAIN" },
  { id: "assistant", icon: "🤖", label: "AI Assistant", section: "AI" },
  { id: "reports", icon: "📋", label: "Report Analyzer", section: "AI" },
  { id: "health", icon: "❤️", label: "Health Monitor", section: "HEALTH" },
  { id: "prevention", icon: "🛡", label: "Prevention", section: "HEALTH" },
  { id: "hospitals", icon: "🏥", label: "Hospitals", section: "SERVICES" },
  { id: "community", icon: "📡", label: "Community Alerts", section: "SERVICES" },
  { id: "analytics", icon: "📊", label: "Analytics", section: "ADMIN" },
  { id: "users", icon: "👥", label: "User Management", section: "ADMIN" },
  { id: "profile", icon: "👤", label: "Profile", section: "ACCOUNT" },
];

const PAGE_TITLES = {
  dashboard: ["⊞ Dashboard", "Platform overview & key metrics"],
  predict: ["🔬 Disease Prediction", "AI-powered symptom analysis"],
  assistant: ["🤖 AI Health Assistant", "Your personal health AI"],
  reports: ["📋 Report Analyzer", "OCR-powered medical report analysis"],
  health: ["❤️ Health Monitoring", "BMI, lifestyle & health scoring"],
  prevention: ["🛡 Prevention Simulator", "Future risk prediction & prevention"],
  hospitals: ["🏥 Healthcare Services", "Hospitals, clinics & emergency contacts"],
  community: ["📡 Community Alerts", "Outbreak detection & disease heatmaps"],
  analytics: ["📊 Analytics Dashboard", "Disease trends & prediction statistics"],
  users: ["👥 User Management", "Admin panel — manage users"],
  profile: ["👤 My Profile", "Account settings & medical history"],
};

function App() {
  const [user, setUser] = useState(null);
  const [page, setPage] = useState("dashboard");

  const MOBILE_NAV = [
    { id: "dashboard", icon: "⊞", label: "Home" },
    { id: "predict",   icon: "🔬", label: "Predict" },
    { id: "assistant", icon: "🤖", label: "AI Chat" },
    { id: "reports",   icon: "📋", label: "Reports" },
    { id: "health",    icon: "❤️",  label: "Health" },
    { id: "prevention",icon: "🛡",  label: "Prevent" },
    { id: "hospitals", icon: "🏥", label: "Hospitals" },
    { id: "community", icon: "📡", label: "Alerts" },
    { id: "analytics", icon: "📊", label: "Stats" },
    { id: "profile",   icon: "👤", label: "Profile" },
  ];

  if (!user) return (
    <>
      <style>{styles}</style>
      <LoginPage onLogin={setUser} />
    </>
  );

  const sections = [...new Set(NAV.map(n => n.section))];
  const [title, sub] = PAGE_TITLES[page] || ["Page", ""];

  const renderPage = () => {
    switch (page) {
      case "dashboard": return <Dashboard user={user} />;
      case "predict": return <DiseasePrediction user={user} />;
      case "assistant": return <AIAssistant user={user} />;
      case "reports": return <ReportAnalyzer user={user} />;
      case "health": return <HealthMonitoring user={user} />;
      case "prevention": return <Prevention user={user} />;
      case "hospitals": return <Hospitals />;
      case "community": return <Community />;
      case "analytics": return <Analytics />;
      case "users": return user.role === "admin" ? <UserManagement /> : <div className="card" style={{ textAlign: "center", padding: 40, color: "var(--text3)" }}>🔒 Admin access required</div>;
      case "profile": return <Profile user={user} onUpdate={setUser} />;
      default: return <Dashboard user={user} />;
    }
  };

  return (
    <>
      <style>{styles}</style>
      <div className="app">
        <div className="sidebar">
          <div className="logo">
            <div className="logo-icon">🏥</div>
            <div>
              <div className="logo-text">HealthSphere</div>
              <div className="logo-sub">AI Platform</div>
            </div>
          </div>

          {sections.map(sec => (
            <div key={sec} className="nav-section">
              <div className="nav-label">{sec}</div>
              {NAV.filter(n => n.section === sec).map(n => (
                <div key={n.id} className={`nav-item ${page === n.id ? "active" : ""}`} onClick={() => setPage(n.id)}>
                  <span className="icon">{n.icon}</span>
                  {n.label}
                </div>
              ))}
            </div>
          ))}

          <div className="sidebar-bottom">
            <div className="user-pill">
              <div className="user-avatar">{user.name.slice(0, 2).toUpperCase()}</div>
              <div>
                <div className="user-name">{user.name.split(" ")[0]}</div>
                <div className="user-role">{user.role === "admin" ? "👑 Admin" : "Patient"}</div>
              </div>
              <span style={{ marginLeft: "auto", cursor: "pointer", color: "var(--text3)", fontSize: 14 }} onClick={() => setUser(null)} title="Sign out">⎋</span>
            </div>
          </div>
        </div>

        <div className="main">
          <div className="topbar">
            <div style={{ flex: 1 }}>
              <div className="page-title">{title}</div>
              <div className="page-sub">{sub}</div>
            </div>
            <span className="badge badge-green">● Gen AI Active</span>
            <button className="btn-icon" title="Notifications">🔔</button>
          </div>
          <div className="content">
            {renderPage()}
          </div>
        </div>

        {/* Mobile Bottom Navigation */}
        <div className="mobile-nav">
          {MOBILE_NAV.map(n => (
            <div key={n.id} className={`mobile-nav-item ${page === n.id ? "active" : ""}`} onClick={() => setPage(n.id)}>
              <span className="mn-icon">{n.icon}</span>
              <span className="mn-label">{n.label}</span>
            </div>
          ))}
        </div>

      </div>
    </>
  );
}


const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);
</script>
</body>
</html>
