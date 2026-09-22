---
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>School Management System</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.5/babel.min.js"></script>
<script src="https://cdn.tailwindcss.com"></script>
<style>
  html, body { margin: 0; padding: 0; height: 100%; background: #F6F1E4; }
  #root { min-height: 100%; }
</style>
</head>
<body>
<div id="root"></div>
<script type="text/babel" data-presets="env,react">
const { useState, useEffect, useCallback, useMemo } = React;

/* ---------------------------------------------------------------------- */
/* Storage helpers                                                         */
/* ---------------------------------------------------------------------- */

const K = {
  school: "sms-school",
  core: "sms-core",
  academics: "sms-academics",
  finance: "sms-finance",
  messages: "sms-messages",
  timetable: "sms-timetable",
};

const API_BASE = "/api/storage";
async function loadKey(key, fallback) {
  try {
    const res = await fetch(`${API_BASE}/${encodeURIComponent(key)}`);
    if (res.ok) {
      const data = await res.json();
      return data && typeof data.value === "string" ? JSON.parse(data.value) : fallback;
    }
    if (res.status === 404) return fallback;
  } catch {
    /* backend not reachable — fall through */
  }
  try {
    if (window.storage && window.storage.get) {
      const res = await window.storage.get(key, true);
      return res ? JSON.parse(res.value) : fallback;
    }
  } catch {
    /* fall through to localStorage */
  }
  try {
    const raw = localStorage.getItem(key);
    return raw ? JSON.parse(raw) : fallback;
  } catch {
    return fallback;
  }
}
async function saveKey(key, value) {
  try {
    const res = await fetch(`${API_BASE}/${encodeURIComponent(key)}`, {
      method: "PUT",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ value: JSON.stringify(value) }),
    });
    if (res.ok) return;
  } catch {
    /* backend not reachable — fall through */
  }
  try {
    if (window.storage && window.storage.set) {
      await window.storage.set(key, JSON.stringify(value), true);
      return;
    }
  } catch {
    /* fall through to localStorage */
  }
  try {
    localStorage.setItem(key, JSON.stringify(value));
  } catch {
    /* best effort */
  }
}

/* ---------------------------------------------------------------------- */
/* Icon shim (replaces lucide-react for standalone HTML use)               */
/* ---------------------------------------------------------------------- */

const ICON_GLYPHS = {
  LayoutDashboard: "\u25A6", Users: "\u{1F465}", BookOpen: "\u{1F4D6}",
  ClipboardList: "\u{1F4CB}", FileText: "\u{1F4C4}", Wallet: "\u{1F4B0}",
  MessageSquare: "\u{1F4AC}", GraduationCap: "\u{1F393}", Bell: "\u{1F514}",
  ChevronRight: "\u203A", Plus: "+", X: "\u2715", Search: "\u{1F50D}",
  Send: "\u27A4", School: "\u{1F3EB}", Trash2: "\u{1F5D1}", Pencil: "\u270E",
  CheckCircle2: "\u2713", AlertTriangle: "\u26A0", Menu: "\u2630",
  Award: "\u{1F3C6}", Printer: "\u{1F5A8}", Calendar: "\u{1F4C5}", Clock: "\u{1F551}",
  HelpCircle: "\u2753", Upload: "\u2B06",
};

function Icon({ name, size = 16, className = "" }) {
  return (
    <span
      className={className}
      style={{ fontSize: size, lineHeight: 1, display: "inline-flex", alignItems: "center", justifyContent: "center", width: size, height: size }}
      aria-hidden="true"
    >
      {ICON_GLYPHS[name] || "\u2022"}
    </span>
  );
}



const uid = (p) => `${p}_${Math.random().toString(36).slice(2, 9)}`;

function makeUsername(name) {
  const slug = name.trim().toLowerCase().replace(/[^a-z0-9]+/g, ".").replace(/^\.+|\.+$/g, "");
  return `${slug}${Math.floor(10 + Math.random() * 90)}`;
}
function generateTempPassword(len = 8) {
  const chars = "ABCDEFGHJKLMNPQRSTUVWXYZabcdefghjkmnpqrstuvwxyz23456789";
  let out = "";
  for (let i = 0; i < len; i++) out += chars[Math.floor(Math.random() * chars.length)];
  return out;
}
function waLink(phone, text) {
  const digits = (phone || "").replace(/[^0-9]/g, "");
  return `https://wa.me/${digits}?text=${encodeURIComponent(text)}`;
}
function toWhatsAppNumber(phone) {
  // Converts a local Kenyan-style number (leading 0) into the international
  // format wa.me requires (country code, no leading 0 or +).
  const digits = (phone || "").replace(/[^0-9]/g, "");
  return digits.startsWith("0") ? `254${digits.slice(1)}` : digits;
}
function credentialsMessage(schoolName, teacherName, username, tempPassword) {
  return `Hello ${teacherName}, welcome to ${schoolName || "the school"}'s management system! Your sign-in credentials:\nUsername: ${username}\nTemporary password: ${tempPassword}\nPlease sign in and change your password on first login.`;
}

const HELP_MANAGER_PHONE = "0707744407";
function helpMessage(schoolName) {
  return `Hello, I need help with ${schoolName || "our school"}'s management system.`;
}

function toCsv(rows) {
  return rows
    .map((row) => row.map((cell) => {
      const s = String(cell ?? "");
      return /[",\n]/.test(s) ? `"${s.replace(/"/g, '""')}"` : s;
    }).join(","))
    .join("\r\n");
}
function downloadCsv(filename, rows) {
  const csv = toCsv(rows);
  const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}

function parseCsv(text) {
  const rows = [];
  let row = [], field = "", inQuotes = false;
  for (let i = 0; i < text.length; i++) {
    const c = text[i];
    if (inQuotes) {
      if (c === '"') {
        if (text[i + 1] === '"') { field += '"'; i++; }
        else inQuotes = false;
      } else field += c;
    } else if (c === '"') {
      inQuotes = true;
    } else if (c === ",") {
      row.push(field); field = "";
    } else if (c === "\n" || c === "\r") {
      if (c === "\r" && text[i + 1] === "\n") i++;
      row.push(field); field = "";
      rows.push(row); row = [];
    } else {
      field += c;
    }
  }
  if (field !== "" || row.length) { row.push(field); rows.push(row); }
  return rows
    .map((r) => r.map((c) => c.trim()))
    .filter((r) => r.length && r.some((c) => c !== ""));
}

/* ---------------------------------------------------------------------- */
/* Seed data                                                               */
/* ---------------------------------------------------------------------- */

const SUBJECTS = ["Mathematics", "English", "Kiswahili", "Science", "History", "Art"];
const TERMS = ["Term 1", "Term 2", "Term 3"];
const DAYS = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"];
const GRADE_LEVELS = [
  "PP1", "PP2",
  "Grade 1", "Grade 2", "Grade 3", "Grade 4", "Grade 5", "Grade 6",
  "Grade 7", "Grade 8", "Grade 9", "Grade 10", "Grade 11", "Grade 12",
];

function seedSchool() {
}

const NON_TEACHING_PERIODS = ["Break", "Lunch"];

function seedTimetable(core) {
  const teachingSlots = [
    { start: "08:00", end: "08:40" },
    { start: "08:40", end: "09:20" },
    { start: "09:20", end: "10:00" },
    { start: "10:20", end: "11:00" },
    { start: "11:00", end: "11:40" },
    { start: "12:40", end: "13:20" },
    { start: "13:20", end: "14:00" },
  ];
  const fixedSlots = [
    { start: "10:00", end: "10:20", subject: "Break" },
    { start: "11:40", end: "12:40", subject: "Lunch" },
  ];
  const periods = [];
  core.classes.forEach((cls) => {
    const pairs = [];
    core.teachers.forEach((t) => {
      if (t.classes.includes(cls.id)) {
        t.subjects.forEach((subject) => pairs.push({ subject, teacherId: t.id }));
      }
    });
    DAYS.forEach((day) => {
      fixedSlots.forEach((slot) => {
        periods.push({ id: uid("pd"), classId: cls.id, day, startTime: slot.start, endTime: slot.end, subject: slot.subject, teacherId: "" });
      });
    });
    if (pairs.length === 0) return;
    let idx = 0;
    DAYS.forEach((day) => {
      teachingSlots.forEach((slot) => {
        const pair = pairs[idx % pairs.length];
        idx++;
        periods.push({ id: uid("pd"), classId: cls.id, day, startTime: slot.start, endTime: slot.end, subject: pair.subject, teacherId: pair.teacherId });
      });
    });
  });
  return { periods };
}

function seedCore() {
  return { students: [], teachers: [], classes: [], subjects: SUBJECTS };
}

function seedAcademics() {
  return { grades: [], assessments: [], assessmentScores: [], subjectComments: [], classTeacherComments: [], headteacherComments: [] };
}

function seedFinance() {
  const feeStructure = [
    { id: "fee_tuition_t1", name: "Tuition — Term 1", amount: 500, term: "Term 1" },
    { id: "fee_activity_t1", name: "Activity Fee — Term 1", amount: 60, term: "Term 1" },
  ];
  return { feeStructure, payments: [] };
}

function seedMessages() {
  return { threads: [] };
}

/* ---------------------------------------------------------------------- */
/* Small UI primitives                                                     */
/* ---------------------------------------------------------------------- */

const Panel = ({ title, action, children, className = "" }) => (
  <div className={`sms-bg-paper border sms-border-sand ${className}`}>
    {title && (
      <div className="flex items-center justify-between px-5 py-3 border-b sms-border-sand">
        <h3 className="font-serif sms-fs-15 tracking-wide sms-text-ink">{title}</h3>
        {action}
      </div>
    )}
    <div className="p-5">{children}</div>
  </div>
);

const Btn = ({ children, onClick, variant = "primary", type = "button", className = "" }) => {
  const base = "inline-flex items-center gap-1.5 px-3.5 py-2 sms-fs-13 font-medium transition-colors border";
  const styles = {
    primary: "sms-bg-ink sms-text-cream sms-border-ink sms-hover-bg-navy2",
    ghost: "bg-transparent sms-text-ink sms-border-ink-30 sms-hover-border-ink",
    danger: "bg-transparent sms-text-rust sms-border-rust-40 sms-hover-bg-rust-10",
    gold: "sms-bg-brass sms-text-ink sms-border-brass sms-hover-bg-brassdark",
  };
  return (
    <button type={type} onClick={onClick} className={`${base} ${styles[variant]} ${className}`}>
      {children}
    </button>
  );
};

const Input = (props) => (
  <input
    {...props}
    className={`w-full border sms-border-sand bg-white px-3 py-2 sms-fs-13 sms-text-ink focus:outline-none sms-focus-border-brass ${props.className || ""}`}
  />
);

const Select = ({ children, ...props }) => (
  <select
    {...props}
    className={`w-full border sms-border-sand bg-white px-3 py-2 sms-fs-13 sms-text-ink focus:outline-none sms-focus-border-brass ${props.className || ""}`}
  >
    {children}
  </select>
);

function Modal({ title, onClose, children }) {
  return (
    <div className="fixed inset-0 sms-bg-ink-40 flex items-center justify-center z-50 p-4">
      <div className="sms-bg-paper border sms-border-sand w-full max-w-md sms-maxh-85vh overflow-y-auto">
        <div className="flex items-center justify-between px-5 py-3 border-b sms-border-sand">
          <h3 className="font-serif sms-fs-15 sms-text-ink">{title}</h3>
          <button onClick={onClose} className="sms-text-ink-60 sms-hover-text-ink">
            <Icon name="X" size={18} />
          </button>
        </div>
        <div className="p-5">{children}</div>
      </div>
    </div>
  );
}

function letterGrade(score) {
  if (score >= 90) return "EE1"; // Exceeding Expectation
  if (score >= 75) return "EE2";
  if (score >= 58) return "ME1"; // Meeting Expectation
  if (score >= 41) return "ME2";
  if (score >= 31) return "AE1"; // Approaching Expectation
  if (score >= 21) return "AE2";
  if (score >= 11) return "BE1"; // Below Expectation
  return "BE2";
}

function findInstructor(core, classId, subject) {
  const t = core.teachers.find((tt) => tt.classes.includes(classId) && tt.subjects.includes(subject));
  return t ? t.name : null;
}

function ratingColor(score) {
  const r = letterGrade(score);
  if (r.startsWith("EE")) return "#2F6F4E";
  if (r.startsWith("ME")) return "#1C2541";
  if (r.startsWith("AE")) return "#B08D57";
  return "#A63A32";
}

function buildSubjectBarChartSvg(grades, width = 560) {
  if (!grades.length) return "";
  const rowH = 14;
  const gap = 5;
  const labelW = Math.min(120, Math.max(65, width * 0.22));
  const scoreW = 34;
  const barMaxW = width - labelW - scoreW - 8;
  const height = grades.length * (rowH + gap) + gap;
  const rows = grades.map((g, i) => {
    const y = gap + i * (rowH + gap);
    const barW = Math.max(2, (g.score / 100) * barMaxW);
    const color = ratingColor(g.score);
    const label = g.subject.length > 16 ? `${g.subject.slice(0, 15)}…` : g.subject;
    return `
      <text x="0" y="${y + rowH / 2 + 3}" font-size="9.5" fill="#1C2541" font-family="-apple-system, Segoe UI, Arial, sans-serif">${label}</text>
      <rect x="${labelW}" y="${y}" width="${barMaxW}" height="${rowH}" rx="2" fill="#F1E9D3" />
      <rect x="${labelW}" y="${y}" width="${barW}" height="${rowH}" rx="2" fill="${color}" />
      <text x="${labelW + barMaxW + 6}" y="${y + rowH / 2 + 3}" font-size="9.5" fill="#1C2541" font-family="-apple-system, Segoe UI, Arial, sans-serif">${g.score}%</text>`;
  }).join("");
  return `<svg viewBox="0 0 ${width} ${height}" width="100%" height="${height}" xmlns="http://www.w3.org/2000/svg">${rows}</svg>`;
}

function buildStampSvg(schoolName, headteacherName, size = 78) {
  const cx = size / 2, cy = size / 2;
  const school = (schoolName || "SCHOOL").toUpperCase();
  const schoolShort = school.length > 22 ? `${school.slice(0, 21)}…` : school;
  const name = headteacherName || "Headteacher";
  const nameShort = name.length > 18 ? `${name.slice(0, 17)}…` : name;
  return `
    <svg width="${size}" height="${size}" viewBox="0 0 ${size} ${size}" xmlns="http://www.w3.org/2000/svg">
      <circle cx="${cx}" cy="${cy}" r="${size / 2 - 2}" fill="none" stroke="#A63A32" stroke-width="1.6" />
      <circle cx="${cx}" cy="${cy}" r="${size / 2 - 6}" fill="none" stroke="#A63A32" stroke-width="0.8" />
      <text x="${cx}" y="${size * 0.27}" text-anchor="middle" font-size="5" fill="#A63A32" font-family="Georgia, serif" letter-spacing="0.4">${schoolShort}</text>
      <text x="${cx}" y="${size * 0.50}" text-anchor="middle" font-size="7" fill="#A63A32" font-family="Georgia, serif" font-weight="bold" letter-spacing="0.5">APPROVED</text>
      <text x="${cx}" y="${size * 0.66}" text-anchor="middle" font-size="5" fill="#A63A32" font-family="Georgia, serif">${nameShort}</text>
      <text x="${cx}" y="${size * 0.81}" text-anchor="middle" font-size="4.5" fill="#A63A32" font-family="Georgia, serif" letter-spacing="0.5">HEADTEACHER</text>
    </svg>`;
}

function polarToCartesian(cx, cy, r, angleDeg) {
  const rad = ((angleDeg - 90) * Math.PI) / 180;
  return { x: cx + r * Math.cos(rad), y: cy + r * Math.sin(rad) };
}
function buildPieChartSvg(segments, size = 150) {
  const total = segments.reduce((sum, s) => sum + s.value, 0);
  const cx = size / 2, cy = size / 2, r = size / 2 - 3;
  const slices = [];
  if (total === 0) {
    slices.push(`<circle cx="${cx}" cy="${cy}" r="${r}" fill="#F1E9D3" stroke="#D9CFB8" stroke-width="1" />`);
  } else {
    let angle = 0;
    segments.forEach((seg) => {
      if (seg.value <= 0) return;
      const sliceAngle = (seg.value / total) * 360;
      if (sliceAngle >= 359.99) {
        slices.push(`<circle cx="${cx}" cy="${cy}" r="${r}" fill="${seg.color}" />`);
      } else {
        const start = polarToCartesian(cx, cy, r, angle);
        const end = polarToCartesian(cx, cy, r, angle + sliceAngle);
        const largeArc = sliceAngle > 180 ? 1 : 0;
        slices.push(`<path d="M ${cx} ${cy} L ${start.x.toFixed(2)} ${start.y.toFixed(2)} A ${r} ${r} 0 ${largeArc} 1 ${end.x.toFixed(2)} ${end.y.toFixed(2)} Z" fill="${seg.color}" stroke="#fff" stroke-width="1.5" />`);
      }
      angle += sliceAngle;
    });
  }
  return `<svg viewBox="0 0 ${size} ${size}" width="${size}" height="${size}" xmlns="http://www.w3.org/2000/svg">${slices.join("")}</svg>`;
}

const RATING_LEGEND = [
  { codes: "EE1/EE2", label: "Exceeding", range: "75–100" },
  { codes: "ME1/ME2", label: "Meeting", range: "41–74" },
  { codes: "AE1/AE2", label: "Approaching", range: "21–40" },
  { codes: "BE1/BE2", label: "Below", range: "0–20" },
];

const COMMENT_BANK = {
  EE1: { en: "Exceptional performance; keep up the excellent work.", sw: "Kiwango cha juu sana; endelea na bidii hiyo." },
  EE2: { en: "Excellent performance this term.", sw: "Umefanya vizuri sana muhula huu." },
  ME1: { en: "Good performance; meeting the expected standard well.", sw: "Umefanya vizuri; unakidhi matarajio kwa kiwango kizuri." },
  ME2: { en: "Meeting the expected standard.", sw: "Unakidhi matarajio ya kiwango kinachohitajika." },
  AE1: { en: "Approaching the expected standard; more practice needed.", sw: "Unakaribia kiwango kinachohitajika; endelea kujitahidi zaidi." },
  AE2: { en: "Approaching the expected standard; needs more support.", sw: "Unakaribia kiwango; unahitaji msaada zaidi." },
  BE1: { en: "Below the expected standard; needs close support.", sw: "Chini ya kiwango kinachohitajika; unahitaji msaada wa karibu." },
  BE2: { en: "Well below the expected standard; needs significant support.", sw: "Chini sana ya kiwango; unahitaji msaada mkubwa." },
};

function suggestedComment(subject, rating) {
  const entry = COMMENT_BANK[rating];
  if (!entry) return "";
  return subject === "Kiswahili" ? entry.sw : entry.en;
}

const REPORT_CARD_PRINT_CSS = `
  * { box-sizing: border-box; }
  html, body { height: 100%; }
  body { font-family: -apple-system, 'Segoe UI', Roboto, Calibri, Helvetica, Arial, sans-serif; font-size: 13px; line-height: 1.3; color: #1C2541; margin: 0; background: #fff; }
  .page { padding: 4mm; page-break-inside: avoid; height: 289mm; overflow: hidden; }
  .page + .page { page-break-before: always; }
  .sheet { border: 1.5px solid #1C2541; padding: 9px 13px 7px; min-height: 100%; box-sizing: border-box; }
  .header { text-align: center; border-bottom: 2px solid #B08D57; padding-bottom: 5px; margin-bottom: 7px; }
  .header img { width: 32px; height: 32px; object-fit: contain; margin-bottom: 2px; }
  .eyebrow { font-size: 10px; letter-spacing: 0.12em; text-transform: uppercase; color: #1C2541aa; }
  h1 { font-size: 17px; margin: 2px 0 0; letter-spacing: 0.01em; font-weight: 700; }
  .meta { display: flex; justify-content: space-between; align-items: center; font-size: 13px; margin-bottom: 6px; gap: 10px; }
  .meta .label { color: #1C2541aa; }
  .meta table td { padding: 0; line-height: 1.25; }
  .photo { width: 36px; height: 36px; border-radius: 50%; object-fit: cover; border: 1px solid #D9CFB8; flex-shrink: 0; }
  .photo-fallback { display: flex; align-items: center; justify-content: center; background: #F1E9D3; color: #1C2541aa; font-size: 13px; }
  table { width: 100%; border-collapse: collapse; font-size: 13px; margin-bottom: 5px; }
  th, td { padding: 2px 4px; border-bottom: 1px solid #D9CFB8; text-align: left; vertical-align: top; line-height: 1.25; }
  th { text-transform: uppercase; font-size: 10px; letter-spacing: 0.02em; color: #1C2541aa; font-weight: 600; }
  th:nth-child(3), th:nth-child(4), th:nth-child(5), th:nth-child(6), td:nth-child(3), td:nth-child(4), td:nth-child(5), td:nth-child(6) { text-align: right; white-space: nowrap; }
  th:nth-child(2), td:nth-child(2) { font-size: 13px; }
  .comment-cell { max-width: 150px; font-size: 13px; color: #1C2541cc; white-space: normal; }
  .chartbox { margin-bottom: 4px; }
  .chartbox .label { font-size: 10px; text-transform: uppercase; letter-spacing: 0.03em; color: #1C2541aa; margin-bottom: 1px; }
  .muted { font-size: 13px; color: #1C2541aa; margin-bottom: 6px; }
  .totalsrow { display: flex; justify-content: space-between; font-size: 11px; color: #1C2541aa; padding: 0 8px; margin-bottom: 3px; }
  .totalsrow strong { color: #1C2541; }
  .avgrow { display: flex; justify-content: space-between; align-items: center; background: #FBF8F1; border: 1px solid #D9CFB8; border-left: 3px solid #B08D57; padding: 3px 8px; margin-bottom: 6px; }
  .avgrow .avgval { font-size: 16px; font-weight: 700; }
  .pathway { border-top: 1px solid #D9CFB8; padding-top: 4px; margin-bottom: 4px; }
  .pathway .label { font-size: 10px; text-transform: uppercase; letter-spacing: 0.03em; color: #1C2541aa; margin-bottom: 1px; }
  .pathway p { font-size: 13px; margin: 0 0 1px; line-height: 1.3; }
  .comments { border-top: 1px solid #D9CFB8; padding-top: 4px; margin-bottom: 4px; }
  .comments .label { font-size: 10px; text-transform: uppercase; letter-spacing: 0.03em; color: #1C2541aa; margin-bottom: 1px; }
  .comments p { font-size: 13px; margin: 0 0 1px; line-height: 1.3; }
  .note { font-style: italic; color: #1C2541b3; }
  .legend { font-size: 9.5px; color: #1C2541aa; display: flex; flex-wrap: wrap; gap: 1px 8px; border-top: 1px solid #D9CFB8; padding-top: 4px; margin-bottom: 6px; }
  .legend strong { color: #1C2541; }
  .signatures { display: flex; justify-content: space-between; gap: 12px; margin-top: 18px; }
  .sig { flex: 1; text-align: center; }
  .sig .sigline { border-top: 1px solid #1C2541; margin-top: 14px; padding-top: 2px; font-size: 10px; color: #1C2541aa; }
  .autosign { font-family: 'Brush Script MT', 'Segoe Script', 'Lucida Handwriting', cursive; font-size: 21px; color: #1C2541; transform: rotate(-2deg); display: inline-block; }
  .sigimg { max-height: 32px; max-width: 100px; object-fit: contain; }
  .stampimg { max-height: 64px; max-width: 64px; object-fit: contain; }
  .stampwrap { display: flex; justify-content: center; }
  .footer { display: flex; justify-content: space-between; font-size: 9px; color: #1C2541aa; margin-top: 4px; padding-top: 3px; border-top: 1px solid #D9CFB8; }
  @page { size: A4; margin: 0; }
`;

function buildReportCardPageHtml(school, student, cls, term, grades, avg, avgMid, avgEnd, positions, pathway, subjectComments, classTeacherComment, headteacherComment, core) {
  const comments = subjectComments || [];
  const classTeacherRecord = core && cls ? core.teachers.find((t) => t.id === cls.teacherId) : null;
  const classTeacherName = classTeacherComment?.teacherName || classTeacherRecord?.name || null;
  const classTeacherSignatureImage = classTeacherRecord?.signatureImage || null;
  const headteacherStampImage = school?.stampImage || null;
  const rowsHtml = grades.map((g) => {
    const comment = comments.find((c) => c.subject === g.subject)?.comment || "";
    return `
    <tr><td>${g.subject}</td><td>${(core && cls ? findInstructor(core, cls.id, g.subject) : null) || "—"}</td><td>${g.midScore !== null ? `${g.midScore}%` : "—"}</td><td>${g.endScore !== null ? `${g.endScore}%` : "—"}</td><td>${g.score}%</td><td>${letterGrade(g.score)}</td><td class="comment-cell">${comment}</td></tr>`;
  }).join("");
  const legendHtml = RATING_LEGEND.map((r) => `<span><strong>${r.codes}</strong> ${r.label} (${r.range})</span>`).join("");
          <div class="eyebrow">${school?.name || "Unregistered School"}</div>
          <h1>Report Card — ${term}</h1>
        </div>
        <div class="meta">
          <div>
            <div><span class="label">Student:</span> <strong>${student.name}</strong></div>
            <div><span class="label">Class:</span> <strong>${cls?.name || ""}</strong></div>
          </div>
          ${student.photo ? `<img class="photo" src="${student.photo}" />` : `<div class="photo photo-fallback">${(student.name || "?").charAt(0)}</div>`}
        </div>
        ${grades.length ? `
          <table>
            <thead><tr><th>Subject</th><th>Instructor</th><th>Mid</th><th>End</th><th>Score</th><th>Rating</th><th>Comment</th></tr></thead>
            <tbody>${rowsHtml}</tbody>
          </table>
          <div class="chartbox">
            <div class="label">Learning area performance</div>
            ${buildSubjectBarChartSvg(grades, 500)}
          </div>` : `<p class="muted">No subject grades recorded for this term yet.</p>`}
        <div class="totalsrow">
          <span>Total Mid-Term average: <strong>${avgMid !== null ? `${avgMid}%` : "—"}</strong></span>
          <span>Total End-Term average: <strong>${avgEnd !== null ? `${avgEnd}%` : "—"}</strong></span>
        </div>
        <div class="avgrow">
          <span class="avglabel">Overall average</span>
          <span class="avgval">${avg !== null ? `${avg}% (${letterGrade(avg)})` : "—"}</span>
        </div>
        <div class="totalsrow">
          <span>Stream position: <strong>${positions && positions.streamPosition !== null ? `${positions.streamPosition} of ${positions.streamTotal}` : "—"}</strong></span>
          <span>Grade position: <strong>${positions && positions.gradePosition !== null ? `${positions.gradePosition} of ${positions.gradeTotal}` : "—"}</strong></span>
        </div>
        <div class="pathway">
          <div class="label">Recommended pathway</div>
          ${pathway ? `<p><strong>${pathway.track}</strong>${pathway.topSubject ? ` — based on strongest performance in ${pathway.topSubject}` : ""}</p>` : `<p>Not enough grade data yet to suggest a pathway.</p>`}
        </div>
        ${classTeacherComment?.comment ? `
          <div class="comments">
            <div class="label">Class teacher's comment${classTeacherComment.teacherName ? ` — ${classTeacherComment.teacherName}` : ""}</div>
            <p>"${classTeacherComment.comment}"</p>
          </div>` : ""}
        ${headteacherComment?.comment ? `
          <div class="comments">
            <div class="label">Headteacher's comment${school?.headteacherName ? ` — ${school.headteacherName}` : ""}</div>
            <p>"${headteacherComment.comment}"</p>
          </div>` : ""}
        <div class="legend">${legendHtml}</div>
        <div class="signatures">
          <div class="sig">
            ${classTeacherSignatureImage
              ? `<img class="sigimg" src="${classTeacherSignatureImage}" />`
              : (classTeacherName ? `<div class="autosign">${classTeacherName}</div>` : "")}
            <div class="sigline">Class Teacher's Signature</div>
          </div>
          <div class="sig">
            <div class="stampwrap">${headteacherStampImage ? `<img class="stampimg" src="${headteacherStampImage}" />` : buildStampSvg(school?.name, school?.headteacherName)}</div>
            <div class="sigline">Headteacher's Signature</div>
          </div>
          <div class="sig"><div class="sigline">Parent/Guardian's Signature</div></div>
        </div>
        <div class="footer">
          <span>${school?.name || "Unregistered School"} — ${term}</span>
          <span>Generated on ${generatedOn}</span>
        </div>
      </div>
    </div>`;
}

function openReportCardPrintWindow(pagesHtml, title) {
  const win = window.open("", "_blank", "width=800,height=900");
  if (!win) return;
  win.document.write(`
    <html>
      <head><title>${title}</title><style>${REPORT_CARD_PRINT_CSS}</style></head>
      <body>${pagesHtml}</body>
    </html>
  `);
  win.document.close();
  win.focus();
  win.print();
}

const PATHWAY_SUBJECT_KEYWORDS = {
  STEM: ["math", "science", "physics", "chemistry", "biology", "computer", "ict", "agriculture", "technology", "engineering"],
  "Social Science": ["english", "kiswahili", "history", "geography", "religious", "cre", "ire", "hre", "social", "business", "economics", "civics", "language", "literature"],
  "Arts and Sports": ["art", "music", "drama", "sport", "physical education", "p.e.", " pe ", "dance", "design", "theatre"],
};

function classifySubjectPathway(subject) {
  const s = ` ${(subject || "").toLowerCase()} `;
  for (const [pathway, keywords] of Object.entries(PATHWAY_SUBJECT_KEYWORDS)) {
    if (keywords.some((k) => s.includes(k))) return pathway;
  }
  return "Social Science"; // general/unclassified learning areas default here
}

function computePathway(avg, grades) {
  if (avg === null || !grades.length) return null;
  const buckets = { STEM: [], "Social Science": [], "Arts and Sports": [] };
  grades.forEach((g) => buckets[classifySubjectPathway(g.subject)].push(g.score));

  const bucketAverages = Object.entries(buckets)
    .filter(([, scores]) => scores.length > 0)
    .map(([pathway, scores]) => ({ pathway, avg: scores.reduce((a, b) => a + b, 0) / scores.length }))
    .sort((a, b) => b.avg - a.avg);

  const top = bucketAverages[0];
  const topSubject = grades.reduce((best, g) => (g.score > best.score ? g : best), grades[0]);
  return { track: `${top.pathway} Pathway`, topSubject: topSubject?.subject || null };
}

function isKiswahili(subject) {
  return (subject || "").trim().toLowerCase() === "kiswahili";
}
function commentColumnLabel(subject) {
  return isKiswahili(subject) ? "Maoni" : "Comment";
}
function commentPlaceholder(subject) {
  return isKiswahili(subject)
    ? "Andika maoni ya mwalimu kwa Kiswahili…"
    : "Add a teacher's comment for this learning area…";
}

const AUTO_COMMENT_TEMPLATES = {
  EE1: {
    en: (s) => `Excellent work in ${s}.`,
    sw: (s) => `Kazi bora katika ${s}.`,
  },
  EE2: {
    en: (s) => `Very strong in ${s}.`,
    sw: (s) => `Utendaji mzuri sana katika ${s}.`,
  },
  ME1: {
    en: (s) => `Good grasp of ${s}.`,
    sw: (s) => `Anaelewa vizuri ${s}.`,
  },
  ME2: {
    en: (s) => `Meets expectations in ${s}.`,
    sw: (s) => `Anatimiza matarajio katika ${s}.`,
  },
  AE1: {
    en: (s) => `Improving steadily in ${s}.`,
    sw: (s) => `Anaendelea vizuri katika ${s}.`,
  },
  AE2: {
    en: (s) => `Early progress in ${s}.`,
    sw: (s) => `Anaanza kuimarika katika ${s}.`,
  },
  BE1: {
    en: (s) => `Needs support in ${s}.`,
    sw: (s) => `Anahitaji msaada katika ${s}.`,
  },
  BE2: {
    en: (s) => `Needs close support in ${s}.`,
    sw: (s) => `Anahitaji msaada wa karibu katika ${s}.`,
  },
};

function generateAutoComment(subject, score) {
  if (score === "" || score === null || score === undefined) return "";
  const rating = letterGrade(score);
  const tmpl = AUTO_COMMENT_TEMPLATES[rating];
  if (!tmpl) return "";
  return isKiswahili(subject) ? tmpl.sw(subject) : tmpl.en(subject);
}

const CLASS_TEACHER_COMMENT_TEMPLATES = {
  EE1: "An outstanding term overall. Keep up the excellent work and keep seeking new challenges.",
  EE2: "A very strong term overall. Consistent effort is paying off well — well done.",
  ME1: "A good term overall, meeting expectations well across most learning areas.",
  ME2: "A satisfactory term overall, meeting basic expectations. More consistency will help going forward.",
  AE1: "Progress is developing steadily this term. More consistent effort will help meet expectations.",
  AE2: "Early signs of progress this term. Needs more consistent engagement and support at home.",
  BE1: "This term has been below expectations overall and needs closer support and follow-up.",
  BE2: "Performance this term is well below expectations. A support plan is strongly recommended.",
};

const HEADTEACHER_COMMENT_TEMPLATES = {
  EE1: "Excellent overall results this term. Congratulations on an exceptional performance.",
  EE2: "Very good overall results this term. Well done.",
  ME1: "A commendable overall performance this term. Keep working hard.",
  ME2: "A fair overall performance this term. Continued effort is encouraged.",
  AE1: "Overall performance is approaching expectations. Increased effort is encouraged next term.",
  AE2: "Overall performance is approaching expectations. Support and encouragement at home are advised.",
  BE1: "Overall performance requires improvement. Please engage additional support at home and school.",
  BE2: "Overall performance is a serious concern this term. An intervention plan is strongly recommended.",
};

function generateOverallComment(kind, avg) {
  if (avg === null || avg === undefined) return "";
  const rating = letterGrade(avg);
  const templates = kind === "head" ? HEADTEACHER_COMMENT_TEMPLATES : CLASS_TEACHER_COMMENT_TEMPLATES;
  return templates[rating] || "";
}

const ASSESSMENT_TYPES = [
  { id: "midterm", label: "Mid-Term" },
  { id: "endterm", label: "End-Term" },
];

/**
 * Combines separately-entered mid-term and end-term marks into one row per
 * subject (per term, if a term is given — otherwise across all terms).
 * The overall score is the average of whichever of the two are present.
 */
function combineGradesFor(allGrades, studentId, term) {
  const filtered = allGrades.filter((g) => g.studentId === studentId && (term === undefined || g.term === term));
  const map = new Map();
  filtered.forEach((g) => {
    const key = `${g.subject}|${g.term}`;
    if (!map.has(key)) map.set(key, { subject: g.subject, term: g.term, id: g.id });
    const rec = map.get(key);
    if (g.type === "midterm") rec.midScore = g.score;
    else rec.endScore = g.score;
    rec.id = g.id; // last-written record's id is fine for a stable-enough React key
  });
  return [...map.values()]
    .map((rec) => {
      const scores = [rec.midScore, rec.endScore].filter((s) => typeof s === "number");
      const score = scores.length ? Math.round(scores.reduce((a, b) => a + b, 0) / scores.length) : null;
      return { id: rec.id, subject: rec.subject, term: rec.term, score, midScore: rec.midScore ?? null, endScore: rec.endScore ?? null };
    })
    .filter((r) => r.score !== null);
}

function averageScoreFor(allGrades, studentId, term) {
  const g = combineGradesFor(allGrades, studentId, term);
  return g.length ? g.reduce((a, b) => a + b.score, 0) / g.length : null;
}

function rankStudentIn(students, allGrades, studentId, term) {
  const scored = students
    .map((s) => ({ id: s.id, avg: averageScoreFor(allGrades, s.id, term) }))
    .filter((x) => x.avg !== null)
    .sort((a, b) => b.avg - a.avg);
  let position = null, rankNum = 0, lastAvg = null;
  scored.forEach((x, i) => {
    if (x.avg !== lastAvg) { rankNum = i + 1; lastAvg = x.avg; }
    if (x.id === studentId) position = rankNum;
  });
  return { position, total: scored.length };
}

/**
 * Returns the learner's rank within their own stream (class) and within the
 * whole grade level (every stream sharing the same grade level combined).
 */
function computeStreamAndGradePositions(core, allGrades, studentId, term) {
  const student = core.students.find((s) => s.id === studentId);
  if (!student) return { streamPosition: null, streamTotal: 0, gradePosition: null, gradeTotal: 0 };
  const cls = core.classes.find((c) => c.id === student.classId);
  const gradeLevel = cls?.gradeLevel || cls?.name || "";

  const streamStudents = core.students.filter((s) => s.classId === student.classId);
  const gradeClassIds = core.classes.filter((c) => (c.gradeLevel || c.name) === gradeLevel).map((c) => c.id);
  const gradeStudents = core.students.filter((s) => gradeClassIds.includes(s.classId));

  const stream = rankStudentIn(streamStudents, allGrades, studentId, term);
  const grade = rankStudentIn(gradeStudents, allGrades, studentId, term);
  return { streamPosition: stream.position, streamTotal: stream.total, gradePosition: grade.position, gradeTotal: grade.total };
}

/* ---------------------------------------------------------------------- */
/* Main app                                                                 */
/* ---------------------------------------------------------------------- */

function LoginScreen({ school, core, updateSchool, onSignIn, backendStatus }) {
  const [tab, setTab] = useState("admin");

  // Admin
  const isAdminSetUp = !!school.adminAccount;
  const [adminName, setAdminName] = useState("");
  const [adminUsername, setAdminUsername] = useState("");
  const [adminPassword, setAdminPassword] = useState("");
  const [adminPassword2, setAdminPassword2] = useState("");
  const [secQuestion, setSecQuestion] = useState("");
  const [secAnswer, setSecAnswer] = useState("");
  const [adminError, setAdminError] = useState("");

  const [recoverStep, setRecoverStep] = useState(1); // 1: answer question, 2: set new password
  const [recoverAnswer, setRecoverAnswer] = useState("");
  const [recoverNewPassword, setRecoverNewPassword] = useState("");
  const [recoverNewPassword2, setRecoverNewPassword2] = useState("");
  const [recoverError, setRecoverError] = useState("");
  const [recoverDone, setRecoverDone] = useState(false);

  const createAdminAccount = () => {
    if (!adminName.trim() || !adminUsername.trim() || !adminPassword) { setAdminError("Please fill in every field, including the security question."); return; }
    if (adminPassword !== adminPassword2) { setAdminError("Passwords don't match."); return; }
    if (!secQuestion.trim() || !secAnswer.trim()) { setAdminError("A security question and answer are needed for account recovery."); return; }
    updateSchool({ ...school, adminAccount: { name: adminName.trim(), username: adminUsername.trim(), password: adminPassword, securityQuestion: secQuestion.trim(), securityAnswer: secAnswer.trim() } });
    onSignIn({ role: "admin", id: "admin" });
  };
  const signInAdmin = () => {
    const acc = school.adminAccount;
    if (acc && acc.username === adminUsername.trim() && acc.password === adminPassword) {
      onSignIn({ role: "admin", id: "admin" });
    } else {
      setAdminError("Incorrect username or password.");
    }
  };

  const openRecover = () => {
    setShowRecover(true);
    setRecoverStep(1);
    setRecoverAnswer("");
    setRecoverNewPassword("");
    setRecoverNewPassword2("");
    setRecoverError("");
    setRecoverDone(false);
  };
  const submitRecoverAnswer = () => {
    const acc = school.adminAccount;
    if (!acc?.securityAnswer || recoverAnswer.trim().toLowerCase() !== acc.securityAnswer.toLowerCase()) {
      setRecoverError("That answer doesn't match.");
      return;
    }
    setRecoverError("");
    setRecoverStep(2);
  };
  const submitNewPassword = () => {
    if (!recoverNewPassword) { setRecoverError("Enter a new password."); return; }
    if (recoverNewPassword !== recoverNewPassword2) { setRecoverError("Passwords don't match."); return; }
    updateSchool({ ...school, adminAccount: { ...school.adminAccount, password: recoverNewPassword } });
    setRecoverError("");
    setRecoverDone(true);
  };

  // Teacher
  const [teacherUsername, setTeacherUsername] = useState("");
  const [teacherPassword, setTeacherPassword] = useState("");
  const [teacherError, setTeacherError] = useState("");
  const signInTeacher = () => {
    const t = core.teachers.find((tt) => tt.username === teacherUsername.trim() && tt.tempPassword === teacherPassword);
    if (t) onSignIn({ role: "teacher", id: t.id });
  };

  // Parent/Guardian
  const [parentUsername, setParentUsername] = useState("");
  const [parentPassword, setParentPassword] = useState("");
  const [parentError, setParentError] = useState("");
  const signInParent = () => {
    const s = core.students.find((ss) => ss.guardianUsername === parentUsername.trim() && ss.guardianTempPassword === parentPassword);
    if (s) onSignIn({ role: "parent", id: s.id });
    else setParentError("Incorrect username or password. Ask the school admin for your sign-in details.");
  };

  const tabs = [
    { id: "admin", label: "Admin" },
    { id: "teacher", label: "Teacher" },
    { id: "parent", label: "Parent/Guardian" },
  ];

  const linkStyle = { color: "#B08D57", cursor: "pointer", textDecoration: "underline", background: "none", border: "none", padding: 0, font: "inherit" };

  return (
    <div className="sms-sans flex items-center justify-center sms-bg-cream" style={{ minHeight: 500 }}>
      <style>{`
        .sms-sans { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif; }
        .sms-bg-cream { background-color: #F6F1E4; }
      `}</style>
      <div className="w-full max-w-sm border" style={{ borderColor: "#D9CFB8", backgroundColor: "#FBF8F1" }}>
        <div className="text-center px-6 pt-6 pb-4 border-b" style={{ borderColor: "#D9CFB8" }}>
          {school.logo ? (
            <img src={school.logo} alt="School logo" className="w-10 h-10 object-contain mx-auto mb-1.5" />
          ) : (
            <Icon name="School" size={26} style={{ color: "#B08D57", margin: "0 auto 6px" }} />
          )}
          <div style={{ fontFamily: "Georgia, serif", fontSize: 17, color: "#1C2541" }}>{school.name || "Unregistered School"}</div>
          <div className="text-xs mt-0.5" style={{ color: "#1C254199" }}>Sign in to continue</div>
          {backendStatus === "connected" && (
            <div className="text-xs mt-2 inline-flex items-center gap-1.5 px-2 py-1" style={{ color: "#2F6F4E", backgroundColor: "#2F6F4E14", border: "1px solid #2F6F4E33" }}>
              <span style={{ width: 6, height: 6, borderRadius: "50%", backgroundColor: "#2F6F4E", display: "inline-block" }}></span>
              Connected to server — accounts work on any device
            </div>
          )}
          {backendStatus === "offline" && (
            <div className="text-xs mt-2 inline-flex items-center gap-1.5 px-2 py-1" style={{ color: "#A63A32", backgroundColor: "#A63A3214", border: "1px solid #A63A3233" }}>
              <span style={{ width: 6, height: 6, borderRadius: "50%", backgroundColor: "#A63A32", display: "inline-block" }}></span>
              No server connected — this account will only work on this device
            </div>
          )}
        </div>
        <div className="flex border-b" style={{ borderColor: "#D9CFB8" }}>
          {tabs.map((t) => (
            <button
              key={t.id}
              onClick={() => { setTab(t.id); setShowRecover(false); }}
              className="flex-1 py-2 text-xs font-medium"
              style={tab === t.id ? { borderBottom: "2px solid #B08D57", color: "#1C2541" } : { color: "#1C254180" }}
            >
              {t.label}
            </button>
          ))}
        </div>

        <div className="p-6 space-y-3">
          {tab === "admin" && (
            isAdminSetUp ? (
              showRecover ? (
                recoverDone ? (
                  <>
                    <p className="text-xs" style={{ color: "#2F6F4E" }}>Password updated. You can sign in with your new password now.</p>
                    <Btn onClick={() => setShowRecover(false)} className="w-full justify-center">Back to sign in</Btn>
                  </>
                ) : recoverStep === 1 ? (
                  <>
                    <p className="text-xs" style={{ color: "#1C254199" }}>Security question:</p>
                    <p className="text-xs font-medium" style={{ color: "#1C2541" }}>{school.adminAccount?.securityQuestion}</p>
                    <Input placeholder="Your answer" value={recoverAnswer} onChange={(e) => setRecoverAnswer(e.target.value)} />
                    {recoverError && <p className="text-xs" style={{ color: "#A63A32" }}>{recoverError}</p>}
                    <Btn onClick={submitRecoverAnswer} className="w-full justify-center">Continue</Btn>
                    <button onClick={() => setShowRecover(false)} style={linkStyle} className="text-xs block">Back to sign in</button>
                  </>
                ) : (
                  <>
                    <Input type="password" placeholder="New password" value={recoverNewPassword} onChange={(e) => setRecoverNewPassword(e.target.value)} />
                    <Input type="password" placeholder="Confirm new password" value={recoverNewPassword2} onChange={(e) => setRecoverNewPassword2(e.target.value)} />
                    {recoverError && <p className="text-xs" style={{ color: "#A63A32" }}>{recoverError}</p>}
                    <Btn onClick={submitNewPassword} className="w-full justify-center">Set new password</Btn>
                  </>
                )
              ) : (
                <>
                  <p className="text-xs" style={{ color: "#1C254199" }}>Sign in with the admin account for this school.</p>
                  <Input placeholder="Username" value={adminUsername} onChange={(e) => setAdminUsername(e.target.value)} />
                  <Input type="password" placeholder="Password" value={adminPassword} onChange={(e) => setAdminPassword(e.target.value)} />
                  {adminError && <p className="text-xs" style={{ color: "#A63A32" }}>{adminError}</p>}
                  <Btn onClick={signInAdmin} className="w-full justify-center">Sign in</Btn>
                  {school.adminAccount?.securityQuestion && (
                    <button onClick={openRecover} style={linkStyle} className="text-xs block">Forgot password?</button>
                  )}
                </>
              )
            ) : (
              <>
                <p className="text-xs" style={{ color: "#1C254199" }}>No admin account yet — create the first one for this school.</p>
                <Input placeholder="Your full name" value={adminName} onChange={(e) => setAdminName(e.target.value)} />
                <Input placeholder="Choose a username" value={adminUsername} onChange={(e) => setAdminUsername(e.target.value)} />
                <Input type="password" placeholder="Choose a password" value={adminPassword} onChange={(e) => setAdminPassword(e.target.value)} />
                <Input type="password" placeholder="Confirm password" value={adminPassword2} onChange={(e) => setAdminPassword2(e.target.value)} />
                <div className="pt-1 text-xs" style={{ color: "#1C254199" }}>For account recovery:</div>
                <Input placeholder="Security question (e.g. your first pet's name)" value={secQuestion} onChange={(e) => setSecQuestion(e.target.value)} />
                <Input placeholder="Answer" value={secAnswer} onChange={(e) => setSecAnswer(e.target.value)} />
                {adminError && <p className="text-xs" style={{ color: "#A63A32" }}>{adminError}</p>}
                <Btn onClick={createAdminAccount} className="w-full justify-center">Create admin account</Btn>
              </>
            )
          )}

          {tab === "teacher" && (
            <>
              <p className="text-xs" style={{ color: "#1C254199" }}>Sign in with the username and temporary password your admin sent you.</p>
              <Input placeholder="Username" value={teacherUsername} onChange={(e) => setTeacherUsername(e.target.value)} />
              <Input type="password" placeholder="Password" value={teacherPassword} onChange={(e) => setTeacherPassword(e.target.value)} />
              {teacherError && <p className="text-xs" style={{ color: "#A63A32" }}>{teacherError}</p>}
              <Btn onClick={signInTeacher} className="w-full justify-center">Sign in</Btn>
              <p className="text-xs" style={{ color: "#1C254199" }}>Forgot your password? Ask your school admin to resend your sign-in details.</p>
            </>
          )}

          {tab === "parent" && (
            <>
              <p className="text-xs" style={{ color: "#1C254199" }}>Sign in with the username and temporary password the school gave you when your child was enrolled.</p>
              <Input placeholder="Username" value={parentUsername} onChange={(e) => setParentUsername(e.target.value)} />
              <Input type="password" placeholder="Password" value={parentPassword} onChange={(e) => setParentPassword(e.target.value)} />
              {parentError && <p className="text-xs" style={{ color: "#A63A32" }}>{parentError}</p>}
              <Btn onClick={signInParent} className="w-full justify-center">Sign in</Btn>
              <p className="text-xs" style={{ color: "#1C254199" }}>Forgot your password? Contact the school admin for help.</p>
            </>
          )}
        </div>
      </div>
    </div>
  );
}

function SchoolManagementSystem() {
  const [school, setSchool] = useState(null);
  const [core, setCore] = useState(null);
  const [academics, setAcademics] = useState(null);
  const [finance, setFinance] = useState(null);
  const [messages, setMessages] = useState(null);
  const [timetable, setTimetable] = useState(null);
  const [loaded, setLoaded] = useState(false);

  const [session, setSession] = useState(null); // { role, id } | null
  const [backendStatus, setBackendStatus] = useState("checking"); // "checking" | "connected" | "offline"
  const [view, setView] = useState("dashboard");
  const [navOpen, setNavOpen] = useState(false);
  const [helpOpen, setHelpOpen] = useState(false);

  useEffect(() => {
    (async () => {
      try {
        const controller = typeof AbortController !== "undefined" ? new AbortController() : null;
        const timer = controller ? setTimeout(() => controller.abort(), 2500) : null;
        const res = await fetch("/api/storage", controller ? { signal: controller.signal } : undefined);
        if (timer) clearTimeout(timer);
        setBackendStatus(res.ok ? "connected" : "offline");
      } catch {
        setBackendStatus("offline");
      }
    })();
    (async () => {
      const sc = await loadKey(K.school, null);
      const c = await loadKey(K.core, null);
      const a = await loadKey(K.academics, null);
      const f = await loadKey(K.finance, null);
      const m = await loadKey(K.messages, null);
      const tt = await loadKey(K.timetable, null);
      const finalSchool = sc || seedSchool();
      const finalCore = c || seedCore();
      const finalAcademics = a || seedAcademics();
      const finalFinance = f || seedFinance();
      const finalMessages = m || seedMessages();
      const finalTimetable = tt || seedTimetable(finalCore);
      setSchool(finalSchool);
      setCore(finalCore);
      setAcademics(finalAcademics);
      setFinance(finalFinance);
      setMessages(finalMessages);
      setTimetable(finalTimetable);
      if (!sc) saveKey(K.school, finalSchool);
      if (!c) saveKey(K.core, finalCore);
      if (!a) saveKey(K.academics, finalAcademics);
      if (!f) saveKey(K.finance, finalFinance);
      if (!m) saveKey(K.messages, finalMessages);
      if (!tt) saveKey(K.timetable, finalTimetable);
      try {
        const raw = localStorage.getItem("sms-session");
        if (raw) setSession(JSON.parse(raw));
      } catch { /* ignore */ }
      setLoaded(true);
    })();
  }, []);

  const signIn = (s) => {
    setSession(s);
    setView("dashboard");
    try { localStorage.setItem("sms-session", JSON.stringify(s)); } catch { /* ignore */ }
  };
  const signOut = () => {
    setSession(null);
    try { localStorage.removeItem("sms-session"); } catch { /* ignore */ }
  };

  const updateSchool = useCallback((next) => { setSchool(next); saveKey(K.school, next); }, []);
  const updateCore = useCallback((next) => { setCore(next); saveKey(K.core, next); }, []);
  const updateAcademics = useCallback((next) => { setAcademics(next); saveKey(K.academics, next); }, []);
  const updateFinance = useCallback((next) => { setFinance(next); saveKey(K.finance, next); }, []);
  const updateMessages = useCallback((next) => { setMessages(next); saveKey(K.messages, next); }, []);
  const updateTimetable = useCallback((next) => { setTimetable(next); saveKey(K.timetable, next); }, []);

  if (!loaded) {
    return (
      <div className="flex items-center justify-center font-serif" style={{ minHeight: 400, backgroundColor: "#F6F1E4", color: "#1C2541" }}>
        Opening the register…
      </div>
    );
  }

  if (!session) {
    return <LoginScreen school={school} core={core} updateSchool={updateSchool} onSignIn={signIn} backendStatus={backendStatus} />;
  }

  const role = session.role;
  const actingAsId = session.id;
  const signedInName =
    role === "admin" ? (school.adminAccount?.name || "Administrator") :
    role === "teacher" ? (core.teachers.find((t) => t.id === actingAsId)?.name || "Teacher") :
    (core.students.find((s) => s.id === actingAsId)?.guardianName || "Parent/Guardian");

  const navByRole = {
    admin: [
      { id: "dashboard", label: "Dashboard", icon: "LayoutDashboard" },
      { id: "schoolsetup", label: "School Setup", icon: "School" },
      { id: "students", label: "Learners", icon: "Users" },
      { id: "teachers", label: "Teachers", icon: "GraduationCap" },
      { id: "gradeentry", label: "Grade Entry", icon: "BookOpen" },
      { id: "timetable", label: "Timetables", icon: "Calendar" },
      { id: "assessment", label: "Assessment Center", icon: "ClipboardList" },
      { id: "reportcards", label: "Report Cards", icon: "FileText" },
      { id: "rankings", label: "Ranking List", icon: "Award" },
      { id: "finance", label: "Finance Control", icon: "Wallet" },
      { id: "messages", label: "Messages", icon: "MessageSquare" },
    ],
    teacher: [
      { id: "dashboard", label: "Study Center", icon: "LayoutDashboard" },
      { id: "gradeentry", label: "Grade Entry", icon: "BookOpen" },
      { id: "timetable", label: "Timetables", icon: "Calendar" },
      { id: "assessment", label: "Assessment Center", icon: "ClipboardList" },
      { id: "reportcards", label: "Report Cards", icon: "FileText" },
      { id: "rankings", label: "Ranking List", icon: "Award" },
      { id: "messages", label: "Messages", icon: "MessageSquare" },
    ],
    parent: [
      { id: "dashboard", label: "Overview", icon: "LayoutDashboard" },
      { id: "timetable", label: "Timetables", icon: "Calendar" },
      { id: "reportcards", label: "Report Cards", icon: "FileText" },
      { id: "finance", label: "Fees & Payments", icon: "Wallet" },
      { id: "messages", label: "Messages", icon: "MessageSquare" },
    ],
  };

  const nav = navByRole[role];

  return (
    <div className="min-sms-h-600 sms-bg-cream sms-text-ink" style={{ fontFamily: "Georgia, 'Iowan Old Style', serif" }}>
      <style>{`
        .sms-sans { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif; }

        .sms-bg-ink { background-color: #1C2541; }
        .sms-bg-ink-40 { background-color: rgba(28,37,65,0.4); }
        .sms-bg-cream { background-color: #F6F1E4; }
        .sms-bg-paper { background-color: #FBF8F1; }
        .sms-bg-wheat { background-color: #F1E9D3; }
        .sms-bg-menu { background-color: #2451D6; }
        .sms-bg-menu-active { background-color: rgba(255,255,255,0.16); }
        .sms-text-menu { color: #DCE6FF; }
        .sms-text-menu-active { color: #FFFFFF; }
        .sms-border-menu-active { border-color: #FFD166; }
        .sms-hover-bg-menu-item:hover { background-color: rgba(255,255,255,0.1); }
        .sms-bg-brass { background-color: #B08D57; }
        .sms-bg-navy2 { background-color: #2A3660; }
        .sms-bg-rust-5 { background-color: rgba(166,58,50,0.05); }

        .sms-border-ink { border-color: #1C2541; }
        .sms-border-ink-30 { border-color: rgba(28,37,65,0.3); }
        .sms-border-sand { border-color: #D9CFB8; }
        .sms-border-sand-60 { border-color: rgba(217,207,184,0.6); }
        .sms-border-brass { border-color: #B08D57; }
        .sms-border-navy3 { border-color: #3d4a80; }
        .sms-border-rust-30 { border-color: rgba(166,58,50,0.3); }
        .sms-border-rust-40 { border-color: rgba(166,58,50,0.4); }

        .sms-text-ink { color: #1C2541; }
        .sms-text-ink-30 { color: rgba(28,37,65,0.3); }
        .sms-text-ink-40 { color: rgba(28,37,65,0.4); }
        .sms-text-ink-50 { color: rgba(28,37,65,0.5); }
        .sms-text-ink-60 { color: rgba(28,37,65,0.6); }
        .sms-text-ink-70 { color: rgba(28,37,65,0.7); }
        .sms-text-cream { color: #F6F1E4; }
        .sms-text-brass { color: #B08D57; }
        .sms-text-rust { color: #A63A32; }
        .sms-text-rust-60 { color: rgba(166,58,50,0.6); }
        .sms-text-rust-70 { color: rgba(166,58,50,0.7); }
        .sms-text-green { color: #2F6F4E; }

        .sms-hover-bg-navy2:hover { background-color: #2A3660; }
        .sms-hover-bg-brassdark:hover { background-color: #9c7c4a; }
        .sms-hover-bg-rust-10:hover { background-color: rgba(166,58,50,0.1); }
        .sms-hover-bg-wheat-50:hover { background-color: rgba(241,233,211,0.5); }
        .sms-hover-bg-wheat-60:hover { background-color: rgba(241,233,211,0.6); }
        .sms-hover-border-ink:hover { border-color: #1C2541; }
        .sms-hover-text-ink:hover { color: #1C2541; }
        .sms-hover-text-rust:hover { color: #A63A32; }
        .sms-focus-border-brass:focus { border-color: #B08D57; }

        .sms-fs-11 { font-size: 11px; line-height: 1.45; }
        .sms-fs-12 { font-size: 12px; line-height: 1.45; }
        .sms-fs-13 { font-size: 13px; line-height: 1.5; }
        .sms-fs-14 { font-size: 14px; line-height: 1.5; }
        .sms-fs-15 { font-size: 15px; line-height: 1.5; }
        .sms-fs-16 { font-size: 16px; line-height: 1.4; }
        .sms-fs-20 { font-size: 20px; line-height: 1.35; }
        .sms-fs-22 { font-size: 22px; line-height: 1.3; }
        .sms-fs-28 { font-size: 28px; line-height: 1.25; }

        .sms-h-400 { height: 400px; }
        .sms-h-480 { height: 480px; }
        .min-sms-h-400 { min-height: 400px; }
        .min-sms-h-600 { min-height: 600px; }

        .sms-maxh-85vh { max-height: 85vh; }
        .sms-maxw-140 { max-width: 140px; }
        .sms-maxw-160 { max-width: 160px; }
        .sms-maxw-180 { max-width: 180px; }
        .sms-maxw-200 { max-width: 200px; }
        .sms-maxw-75pct { max-width: 75%; }

        .sms-divide-sand-60 > :not([hidden]) ~ :not([hidden]) { border-color: rgba(217,207,184,0.6); }
        .sms-tracking-2em { letter-spacing: 0.2em; }
      `}</style>

      {/* Header */}
      <div className="border-b sms-border-sand sms-bg-ink sms-text-cream">
        <div className="flex items-center justify-between px-4 md:px-6 py-3">
          <div className="flex items-center gap-2">
            <button className="md:hidden" onClick={() => setNavOpen((v) => !v)}>
              <Icon name="Menu" size={20} />
            </button>
            {school.logo ? (
              <img src={school.logo} alt="School logo" className="w-7 h-7 object-contain bg-white" />
            ) : (
              <Icon name="School" size={20} className="sms-text-brass" />
            )}
            <span className="font-serif sms-fs-16 tracking-wide">{school.name || "Unregistered School"}</span>
          </div>
          <div className="sms-sans flex items-center gap-2 sms-fs-12">
            <span
              title={backendStatus === "connected" ? "Connected to server — data shared across devices" : backendStatus === "offline" ? "No server connected — saved on this device only" : "Checking connection…"}
              style={{
                width: 8, height: 8, borderRadius: "50%", display: "inline-block", flexShrink: 0,
                backgroundColor: backendStatus === "connected" ? "#4ADE80" : backendStatus === "offline" ? "#F87171" : "#DCE6FF",
              }}
            ></span>
            <button
              onClick={() => setHelpOpen(true)}
              title="Help Center"
              className="flex items-center gap-1 px-2 py-1.5 border sms-border-navy3 sms-hover-bg-navy2"
            >
              <Icon name="HelpCircle" size={15} />
              <span className="hidden md:inline">Help</span>
            </button>
            <div className="text-right hidden sm:block">
              <div className="sms-fs-12 font-medium">{signedInName}</div>
              <div className="sms-fs-11" style={{ color: "#DCE6FF" }}>{role === "admin" ? "Administrator" : role === "teacher" ? "Teacher" : "Parent/Guardian"}</div>
            </div>
            <button
              onClick={signOut}
              title="Sign out"
              className="flex items-center gap-1 px-2 py-1.5 border sms-border-navy3 sms-hover-bg-navy2"
            >
              <span>Sign out</span>
            </button>
          </div>
        </div>
      </div>

      {helpOpen && (
        <Modal title="Help Center" onClose={() => setHelpOpen(false)}>
          <div className="space-y-3">
            <p className="sms-fs-13 sms-text-ink-70">
              Need help using {school.name || "the system"}? Our support manager is available on WhatsApp.
            </p>
            <div className="border sms-border-sand bg-white px-3 py-2 sms-fs-13">
              WhatsApp: <strong>{HELP_MANAGER_PHONE}</strong>
            </div>
            <Btn
              onClick={() => window.open(waLink(toWhatsAppNumber(HELP_MANAGER_PHONE), helpMessage(school.name)), "_blank")}
              className="w-full justify-center"
            >
              <Icon name="Send" size={14} /> Chat on WhatsApp
            </Btn>
          </div>
        </Modal>
      )}

      <div className="flex">
        {/* Sidebar */}
        <div className={`sms-sans w-56 shrink-0 sms-bg-menu ${navOpen ? "block absolute z-40 h-full" : "hidden"} md:block md:relative`}>
          <nav className="py-3">
            {nav.map((n) => {
              const active = view === n.id;
              return (
                <button
                  key={n.id}
                  onClick={() => { setView(n.id); setNavOpen(false); }}
                  className={`w-full flex items-center gap-2.5 px-5 py-2.5 sms-fs-13 text-left border-l-4 ${
                    active ? "sms-border-menu-active sms-bg-menu-active sms-text-menu-active font-semibold" : "border-transparent sms-text-menu sms-hover-bg-menu-item"
                  }`}
                >
                  <Icon name={n.icon} size={16} />
                  {n.label}
                </button>
              );
            })}
          </nav>
        </div>

        {/* Main content */}
        <div className="flex-1 p-4 md:p-6 sms-sans min-w-0">
          {view === "dashboard" && (
            <Dashboard role={role} actingAsId={actingAsId} core={core} academics={academics} finance={finance} messages={messages} school={school} setView={setView} />
          )}
          {view === "schoolsetup" && role === "admin" && (
            <SchoolSetup school={school} updateSchool={updateSchool} core={core} updateCore={updateCore} />
          )}
          {view === "students" && role === "admin" && (
            <StudentsAdmin core={core} updateCore={updateCore} />
          )}
          {view === "teachers" && role === "admin" && (
            <TeachersAdmin core={core} updateCore={updateCore} school={school} />
          )}
          {view === "gradeentry" && (role === "teacher" || role === "admin") && (
            <GradeEntry core={core} academics={academics} updateAcademics={updateAcademics} role={role} teacherId={actingAsId} />
          )}
          {view === "assessment" && (
            <AssessmentCenter
              core={core} academics={academics} updateAcademics={updateAcademics}
              role={role} teacherId={actingAsId}
            />
          )}
          {view === "reportcards" && (
            <ReportCards core={core} academics={academics} updateAcademics={updateAcademics} role={role} actingAsId={actingAsId} school={school} />
          )}
              academics={academics} updateAcademics={updateAcademics}
              role={role} actingAsId={actingAsId} school={school}
            />
          )}
          {view === "rankings" && (role === "admin" || role === "teacher") && (
            <RankingList core={core} academics={academics} school={school} role={role} teacherId={actingAsId} />
          )}
          {view === "finance" && (
            <Finance core={core} finance={finance} updateFinance={updateFinance} role={role} studentId={actingAsId} />
          )}
          {view === "messages" && (
            <MessagesView core={core} messages={messages} updateMessages={updateMessages} role={role} actingAsId={actingAsId} />
          )}
        </div>
      </div>
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/* Dashboard                                                                */
/* ---------------------------------------------------------------------- */

function Stat({ label, value, tone = "default" }) {
  const tones = { default: "sms-text-ink", good: "sms-text-green", bad: "sms-text-rust" };
  return (
    <div className="border sms-border-sand sms-bg-paper px-5 py-4">
      <div className="sms-fs-11 uppercase tracking-widest sms-text-ink-50">{label}</div>
      <div className={`font-serif sms-fs-28 ${tones[tone]}`}>{value}</div>
    </div>
  );
}

function SchoolSetup({ school, updateSchool, core, updateCore }) {
  const [form, setForm] = useState({ ...school, headteacherName: school.headteacherName || "" });
  const [saved, setSaved] = useState(false);
  const [newArea, setNewArea] = useState("");
  const [gradeLevel, setGradeLevel] = useState(GRADE_LEVELS[0]);
  const [stream, setStream] = useState("");
  const [newClassTeacherId, setNewClassTeacherId] = useState("");

  const onLogoChange = (e) => {
    const file = e.target.files?.[0];
    if (!file) return;
    if (!file.type.startsWith("image/")) return;
    const reader = new FileReader();
    reader.onload = () => setForm((f) => ({ ...f, logo: reader.result }));
    reader.readAsDataURL(file);
  };

  const onStampChange = (e) => {
    const file = e.target.files?.[0];
    if (!file) return;
    if (!file.type.startsWith("image/")) return;
    const reader = new FileReader();
    reader.onload = () => setForm((f) => ({ ...f, stampImage: reader.result }));
    reader.readAsDataURL(file);
  };

  const save = () => {
    if (!form.name.trim()) return;
    updateSchool({ ...form, registered: true });
    setSaved(true);
    setTimeout(() => setSaved(false), 2000);
  };

  const addArea = () => {
    const name = newArea.trim();
    if (!name || core.subjects.some((s) => s.toLowerCase() === name.toLowerCase())) return;
    updateCore({ ...core, subjects: [...core.subjects, name] });
    setNewArea("");
  };

  const removeArea = (name) => {
    updateCore({ ...core, subjects: core.subjects.filter((s) => s !== name) });
  };

  const addClass = () => {
    const name = stream.trim() ? `${gradeLevel} ${stream.trim()}` : gradeLevel;
    if (core.classes.some((c) => c.name.toLowerCase() === name.toLowerCase())) return;
    updateCore({ ...core, classes: [...core.classes, { id: uid("cls"), name, gradeLevel, stream: stream.trim(), teacherId: newClassTeacherId || "" }] });
    setStream("");
    setNewClassTeacherId("");
  };

  const setClassTeacher = (classId, teacherId) => {
    updateCore({ ...core, classes: core.classes.map((c) => (c.id === classId ? { ...c, teacherId } : c)) });
  };

  const removeClass = (classId) => {
    if (core.students.some((s) => s.classId === classId)) return;
    updateCore({ ...core, classes: core.classes.filter((c) => c.id !== classId) });
  };

  return (
    <div className="space-y-4 max-w-lg">
      <h2 className="font-serif sms-fs-20">School registration</h2>
      {!school.registered && (
        <div className="flex items-center gap-2 sms-fs-13 sms-text-rust border sms-border-rust-30 sms-bg-rust-5 px-3 py-2">
          <Icon name="AlertTriangle" size={14} /> Your school isn't registered yet. Complete this form to set it up.
        </div>
      )}
      <Panel title="School profile">
        <div className="space-y-3">
          <div className="flex items-center gap-4">
            <div className="w-16 h-16 border sms-border-sand bg-white flex items-center justify-center overflow-hidden shrink-0">
              {form.logo ? <img src={form.logo} alt="Logo preview" className="w-full h-full object-contain" /> : <Icon name="School" size={22} className="sms-text-ink-30" />}
            </div>
            <label className="sms-fs-13">
              <span className="inline-block px-3 py-2 border sms-border-ink-30 cursor-pointer sms-hover-border-ink">Upload logo</span>
              <input type="file" accept="image/*" onChange={onLogoChange} className="hidden" />
            </label>
          </div>
          <Input placeholder="School name" value={form.name} onChange={(e) => setForm({ ...form, name: e.target.value })} />
          <Input placeholder="Motto (optional)" value={form.motto} onChange={(e) => setForm({ ...form, motto: e.target.value })} />
          <Input placeholder="Headteacher's name" value={form.headteacherName} onChange={(e) => setForm({ ...form, headteacherName: e.target.value })} />
          <div className="flex items-center gap-4">
            <div className="w-16 h-16 rounded-full border sms-border-sand bg-white flex items-center justify-center overflow-hidden shrink-0">
              {form.stampImage ? <img src={form.stampImage} alt="Stamp preview" className="w-full h-full object-contain" /> : <Icon name="Award" size={20} className="sms-text-ink-30" />}
            </div>
            <label className="sms-fs-13">
              <span className="inline-block px-3 py-2 border sms-border-ink-30 cursor-pointer sms-hover-border-ink">Upload headteacher's stamp</span>
              <input type="file" accept="image/*" onChange={onStampChange} className="hidden" />
              <div className="sms-fs-11 sms-text-ink-50 mt-1">Used on printed report cards. Falls back to an auto-generated seal if none is uploaded.</div>
            </label>
          </div>
          <Input placeholder="Address" value={form.address} onChange={(e) => setForm({ ...form, address: e.target.value })} />
          <Input placeholder="Contact email" value={form.email} onChange={(e) => setForm({ ...form, email: e.target.value })} />
          <Input placeholder="Contact phone" value={form.phone} onChange={(e) => setForm({ ...form, phone: e.target.value })} />
          <Btn onClick={save} className="w-full justify-center">
            {saved ? <><Icon name="CheckCircle2" size={14} /> Saved</> : school.registered ? "Update school profile" : "Register school"}
          </Btn>
        </div>
      </Panel>

      <Panel title="Grades & class teachers">
        <div className="space-y-2 mb-3">
          {core.classes.length === 0 && <p className="sms-fs-13 sms-text-ink-50">No grades added yet.</p>}
          {core.classes.map((c) => {
            const learnerCount = core.students.filter((s) => s.classId === c.id).length;
            return (
              <div key={c.id} className="flex items-center justify-between gap-2 border-b sms-border-sand-60 py-2">
                <div className="min-w-0">
                  <div className="sms-fs-13 font-medium">{c.name}</div>
                  <div className="sms-fs-12 sms-text-ink-60">{learnerCount} learner{learnerCount === 1 ? "" : "s"}</div>
                </div>
                <div className="flex items-center gap-2 shrink-0">
                  <Select value={c.teacherId || ""} onChange={(e) => setClassTeacher(c.id, e.target.value)} className="sms-maxw-180">
                    <option value="">Unassigned</option>
                    {core.teachers.map((t) => <option key={t.id} value={t.id}>{t.name}</option>)}
                  </Select>
                  <button
                    onClick={() => removeClass(c.id)}
                    disabled={learnerCount > 0}
                    title={learnerCount > 0 ? "Move or remove learners before deleting this grade" : "Delete grade"}
                    className={`sms-text-rust-70 ${learnerCount > 0 ? "opacity-30" : "sms-hover-text-rust"}`}
                  >
                    <Icon name="Trash2" size={14} />
                  </button>
                </div>
              </div>
            );
          })}
        </div>
        <div className="flex flex-wrap gap-2">
          <Select value={gradeLevel} onChange={(e) => setGradeLevel(e.target.value)} className="sms-maxw-160">
            {GRADE_LEVELS.map((g) => <option key={g} value={g}>{g}</option>)}
          </Select>
          <Input placeholder="Stream (optional, e.g. A)" value={stream} onChange={(e) => setStream(e.target.value)} onKeyDown={(e) => e.key === "Enter" && addClass()} className="sms-maxw-160" />
          <Select value={newClassTeacherId} onChange={(e) => setNewClassTeacherId(e.target.value)} className="sms-maxw-160">
            <option value="">Unassigned</option>
            {core.teachers.map((t) => <option key={t.id} value={t.id}>{t.name}</option>)}
          </Select>
          <Btn onClick={addClass}><Icon name="Plus" size={14} /> Add grade</Btn>
        </div>
      </Panel>

      <Panel title="Learning areas">
        <div className="flex flex-wrap gap-1.5 mb-3">
          {core.subjects.map((s) => (
            <span key={s} className="inline-flex items-center gap-1.5 px-2.5 py-1 sms-fs-12 border sms-border-sand bg-white">
              {s}
              <button onClick={() => removeArea(s)} className="sms-text-rust-60 sms-hover-text-rust"><Icon name="X" size={12} /></button>
            </span>
          ))}
          {core.subjects.length === 0 && <p className="sms-fs-13 sms-text-ink-50">No learning areas yet.</p>}
        </div>
        <div className="flex gap-2">
          <Input placeholder="New learning area (e.g. Geography)" value={newArea} onChange={(e) => setNewArea(e.target.value)} onKeyDown={(e) => e.key === "Enter" && addArea()} />
          <Btn onClick={addArea}><Icon name="Plus" size={14} /> Add</Btn>
        </div>
      </Panel>
    </div>
  );
}

function Dashboard({ role, actingAsId, core, academics, finance, messages, school, setView }) {
  if (role === "admin") {
    const totalCollected = finance.payments.reduce((s, p) => s + p.amount, 0);
    const totalBilled = finance.feeStructure.reduce((s, f) => s + f.amount, 0) * core.students.length;
    return (
      <div className="space-y-5">
        <h2 className="font-serif sms-fs-20">Administrator dashboard</h2>
        {!school.registered && (
          <button onClick={() => setView("schoolsetup")} className="w-full text-left flex items-center gap-2 sms-fs-13 sms-text-rust border sms-border-rust-30 sms-bg-rust-5 px-3 py-2 sms-hover-bg-rust-10">
            <Icon name="AlertTriangle" size={14} /> Your school isn't registered yet — click to add its name and logo.
          </button>
        )}
        <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
          <Stat label="Students" value={core.students.length} />
          <Stat label="Teachers" value={core.teachers.length} />
          <Stat label="Fees collected" value={`$${totalCollected}`} tone="good" />
          <Stat label="Outstanding" value={`$${Math.max(totalBilled - totalCollected, 0)}`} tone="bad" />
        </div>
        <Panel title="Classes">
          <div className="grid md:grid-cols-2 gap-3">
            {core.classes.map((c) => {
              const teacher = core.teachers.find((t) => t.id === c.teacherId);
              const count = core.students.filter((s) => s.classId === c.id).length;
              return (
                <div key={c.id} className="border sms-border-sand px-4 py-3 flex justify-between items-center">
                  <div>
                    <div className="font-medium sms-fs-14">{c.name}</div>
                    <div className="sms-fs-12 sms-text-ink-60">Class teacher: {teacher?.name}</div>
                  </div>
                  <div className="sms-fs-12 sms-text-ink-60">{count} students</div>
                </div>
              );
            })}
          </div>
        </Panel>
        <Panel title="Learners by gender">
          {(() => {
            const male = core.students.filter((s) => s.gender === "Male").length;
            const female = core.students.filter((s) => s.gender === "Female").length;
            const unspecified = core.students.length - male - female;
            const total = core.students.length;
            const segments = [
              { label: "Male", value: male, color: "#1C2541" },
              { label: "Female", value: female, color: "#B08D57" },
              { label: "Unspecified", value: unspecified, color: "#2F6F4E" },
            ];
            return (
              <div className="flex items-center gap-6 flex-wrap">
                <div dangerouslySetInnerHTML={{ __html: buildPieChartSvg(segments.filter((s) => s.value > 0), 140) }} />
                <div className="sms-fs-13 space-y-1.5">
                  {segments.map((s) => (
                    <div key={s.label} className="flex items-center gap-2">
                      <span className="inline-block w-3 h-3 rounded-full" style={{ backgroundColor: s.color }}></span>
                      <span>{s.label}: <strong>{s.value}</strong>{total > 0 && <span className="sms-text-ink-50"> ({Math.round((s.value / total) * 100)}%)</span>}</span>
                    </div>
                  ))}
                </div>
              </div>
            );
          })()}
        </Panel>
      </div>
    );
  }

  if (role === "teacher") {
    const teacher = core.teachers.find((t) => t.id === actingAsId);
    const myClasses = core.classes.filter((c) => teacher.classes.includes(c.id));
    const myStudents = core.students.filter((s) => myClasses.some((c) => c.id === s.classId));
    return (
      <div className="space-y-5">
        <h2 className="font-serif sms-fs-20">Welcome, {teacher.name}</h2>
        <div className="grid grid-cols-2 md:grid-cols-3 gap-4">
          <Stat label="Classes" value={myClasses.length} />
          <Stat label="Students" value={myStudents.length} />
          <Stat label="Subjects" value={teacher.subjects.join(", ")} />
        </div>
        <Panel title="My students">
          <div className="overflow-x-auto">
          <table className="w-full sms-fs-13">
            <thead>
              <tr className="text-left sms-text-ink-50 border-b sms-border-sand">
                <th className="py-1.5 font-normal">Name</th>
                <th className="py-1.5 font-normal">Class</th>
                <th className="py-1.5 font-normal">Guardian</th>
              </tr>
            </thead>
            <tbody>
              {myStudents.map((s) => (
                <tr key={s.id} className="border-b sms-border-sand-60">
                  <td className="py-1.5">{s.name}</td>
                  <td className="py-1.5">{myClasses.find((c) => c.id === s.classId)?.name}</td>
                  <td className="py-1.5">{s.guardianName}</td>
                </tr>
              ))}
            </tbody>
          </table>
          </div>
        </Panel>
      </div>
    );
  }

  // parent
  const student = core.students.find((s) => s.id === actingAsId);
  const cls = core.classes.find((c) => c.id === student.classId);
  const myGrades = combineGradesFor(academics.grades, student.id);
  const avg = myGrades.length ? Math.round(myGrades.reduce((s, g) => s + g.score, 0) / myGrades.length) : null;
  const balanceInfo = computeBalance(finance, student.id);
  const thread = messages.threads.find((t) => t.studentId === student.id);
  const unread = thread ? thread.messages.filter((m) => m.role === "teacher").length : 0;

  return (
    <div className="space-y-5">
      <h2 className="font-serif sms-fs-20">{student.name} — {cls?.name}</h2>
      <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
        <Stat label="Average score" value={avg !== null ? `${avg}%` : "—"} />
        <Stat label="Rating" value={avg !== null ? letterGrade(avg) : "—"} />
        <Stat label="Fee balance" value={`$${balanceInfo.balance}`} tone={balanceInfo.balance > 0 ? "bad" : "good"} />
        <Stat label="Messages from teachers" value={unread} />
      </div>
      <Panel title="Recent subject scores">
        {myGrades.length === 0 ? (
          <p className="sms-fs-13 sms-text-ink-60">No grades recorded yet.</p>
        ) : (
          <ul className="space-y-1.5">
            {myGrades.map((g) => (
              <li key={g.id} className="flex justify-between sms-fs-13 border-b sms-border-sand-60 py-1.5">
                <span>{g.subject} · {g.term}</span>
                <span className="font-medium">{g.score}% ({letterGrade(g.score)})</span>
              </li>
            ))}
          </ul>
        )}
      </Panel>
    </div>
  );
}

function computeBalance(finance, studentId) {
  const applicable = finance.feeStructure; // simplistic: all fee items apply to all students
  const billed = applicable.reduce((s, f) => s + f.amount, 0);
  const paid = finance.payments.filter((p) => p.studentId === studentId).reduce((s, p) => s + p.amount, 0);
  return { billed, paid, balance: Math.max(billed - paid, 0) };
}

/* ---------------------------------------------------------------------- */
/* Admin: Students                                                         */
/* ---------------------------------------------------------------------- */

function StudentsAdmin({ core, updateCore }) {
  const emptyForm = { name: "", classId: core.classes[0]?.id || "", guardianName: "", guardianContact: "", photo: null, gender: "" };
  const [modal, setModal] = useState(false);
  const [editingId, setEditingId] = useState(null);
  const [form, setForm] = useState(emptyForm);
  const [query, setQuery] = useState("");
  const [importSummary, setImportSummary] = useState("");
  const [credModal, setCredModal] = useState(null); // student record just issued guardian credentials

  const openAdd = () => { setEditingId(null); setForm(emptyForm); setModal(true); };
  const openEdit = (s) => { setEditingId(s.id); setForm({ name: s.name, classId: s.classId, guardianName: s.guardianName, guardianContact: s.guardianContact, photo: s.photo || null, gender: s.gender || "" }); setModal(true); };

  const onPhotoChange = (e) => {
    const file = e.target.files?.[0];
    if (!file || !file.type.startsWith("image/")) return;
    const reader = new FileReader();
    reader.onload = () => setForm((f) => ({ ...f, photo: reader.result }));
    reader.readAsDataURL(file);
  };

  const saveStudent = () => {
    if (!form.name.trim()) return;
    if (editingId) {
      const existing = core.students.find((s) => s.id === editingId);
      const needsCreds = !existing?.guardianUsername;
      const updated = {
        ...existing, ...form,
        guardianUsername: existing?.guardianUsername || makeUsername(form.guardianName || form.name),
        guardianTempPassword: existing?.guardianTempPassword || generateTempPassword(),
      };
      updateCore({ ...core, students: core.students.map((s) => (s.id === editingId ? updated : s)) });
      if (needsCreds) setCredModal(updated);
    } else {
      const newStudent = {
        id: uid("s"), ...form,
        guardianUsername: makeUsername(form.guardianName || form.name),
        guardianTempPassword: generateTempPassword(),
      };
      updateCore({ ...core, students: [...core.students, newStudent] });
      setCredModal(newStudent);
    }
    setModal(false);
    setEditingId(null);
    setForm(emptyForm);
  };
  const removeStudent = (id) => updateCore({ ...core, students: core.students.filter((s) => s.id !== id) });

  const downloadTemplate = () => {
    const exampleClass = core.classes[0]?.name || "Grade 7A";
    downloadCsv("learners_template.csv", [
      ["Full Name", "Class", "Gender", "Guardian Name", "Guardian Email"],
      ["e.g. Jane Doe", exampleClass, "Female", "e.g. Mary Doe", "e.g. mary.doe@example.com"],
    ]);
  };

  const onUploadTemplate = (e) => {
    const file = e.target.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      const rows = parseCsv(String(reader.result));
      const dataRows = rows.slice(1).filter((r) => !(r[0] || "").toLowerCase().startsWith("e.g."));
      let added = 0, skipped = 0;
      const newStudents = [];
      dataRows.forEach((r) => {
        const [name, className, genderRaw, guardianName, guardianContact] = r;
        const cls = core.classes.find((c) => c.name.toLowerCase() === (className || "").toLowerCase());
        if (!name || !cls) { skipped++; return; }
        const g = (genderRaw || "").trim().toLowerCase();
        const gender = g === "male" || g === "m" ? "Male" : g === "female" || g === "f" ? "Female" : "";
        newStudents.push({ id: uid("s"), name, classId: cls.id, guardianName: guardianName || "", guardianContact: guardianContact || "", photo: null, gender });
        added++;
      });
      if (newStudents.length) updateCore({ ...core, students: [...core.students, ...newStudents] });
      setImportSummary(`Imported ${added} learner${added === 1 ? "" : "s"}.${skipped ? ` ${skipped} row${skipped === 1 ? "" : "s"} skipped (missing name or class not found).` : ""}`);
    };
    reader.readAsText(file);
    e.target.value = "";
  };

  const filtered = core.students.filter((s) => s.name.toLowerCase().includes(query.toLowerCase()));

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between flex-wrap gap-2">
        <h2 className="font-serif sms-fs-20">Learners</h2>
        <div className="flex gap-2">
          <Btn variant="ghost" onClick={downloadTemplate}><Icon name="FileText" size={14} /> Download template</Btn>
          <label>
            <span className="inline-flex items-center gap-1.5 px-3.5 py-2 sms-fs-13 font-medium border sms-border-ink-30 sms-hover-border-ink cursor-pointer sms-text-ink"><Icon name="Upload" size={14} /> Upload filled template</span>
            <input type="file" accept=".csv" onChange={onUploadTemplate} className="hidden" />
          </label>
          <Btn onClick={openAdd}><Icon name="Plus" size={14} /> Enroll learner</Btn>
        </div>
      </div>
      {importSummary && (
        <div className="flex items-center justify-between sms-fs-12 border sms-border-sand sms-bg-wheat px-3 py-1.5">
          <span>{importSummary}</span>
          <button onClick={() => setImportSummary("")} className="sms-text-ink-50 sms-hover-text-ink"><Icon name="X" size={12} /></button>
        </div>
      )}
      <div className="relative max-w-xs">
        <Icon name="Search" size={14} className="absolute left-2.5 top-2.5 sms-text-ink-40" />
        <Input placeholder="Search learners…" value={query} onChange={(e) => setQuery(e.target.value)} className="pl-8" />
      </div>
      <Panel>
        <div className="overflow-x-auto">
        <table className="w-full sms-fs-13">
          <thead>
            <tr className="text-left sms-text-ink-50 border-b sms-border-sand">
              <th className="py-1.5 font-normal"></th>
              <th className="py-1.5 font-normal">Name</th>
              <th className="py-1.5 font-normal">Class</th>
              <th className="py-1.5 font-normal">Guardian</th>
              <th className="py-1.5 font-normal">Contact</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            {filtered.map((s) => (
              <tr key={s.id} className="border-b sms-border-sand-60">
                <td className="py-2">
                  {s.photo ? (
                    <img src={s.photo} alt={s.name} className="w-8 h-8 rounded-full object-cover border sms-border-sand" />
                  ) : (
                    <div className="w-8 h-8 rounded-full sms-bg-wheat flex items-center justify-center sms-fs-11 sms-text-ink-50">{s.name.charAt(0)}</div>
                  )}
                </td>
                <td className="py-2">{s.name}</td>
                <td className="py-2">{core.classes.find((c) => c.id === s.classId)?.name}</td>
                <td className="py-2">{s.guardianName}</td>
                <td className="py-2 sms-text-ink-60">{s.guardianContact}</td>
                <td className="py-2 text-right">
                  <button onClick={() => openEdit(s)} className="sms-text-ink-50 sms-hover-text-ink mr-2"><Icon name="Pencil" size={14} /></button>
                  <button onClick={() => removeStudent(s.id)} className="sms-text-rust-70 sms-hover-text-rust"><Icon name="Trash2" size={14} /></button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
        </div>
      </Panel>

      {modal && (
        <Modal title={editingId ? "Edit learner" : "Enroll learner"} onClose={() => setModal(false)}>
          <div className="space-y-3">
            <div className="flex items-center gap-4">
              <div className="w-16 h-16 rounded-full border sms-border-sand bg-white flex items-center justify-center overflow-hidden shrink-0">
                {form.photo ? <img src={form.photo} alt="Learner preview" className="w-full h-full object-cover" /> : <Icon name="Users" size={20} className="sms-text-ink-30" />}
              </div>
              <label className="sms-fs-13">
                <span className="inline-block px-3 py-2 border sms-border-ink-30 cursor-pointer sms-hover-border-ink">Upload photo</span>
                <input type="file" accept="image/*" onChange={onPhotoChange} className="hidden" />
              </label>
            </div>
            <Input placeholder="Full name" value={form.name} onChange={(e) => setForm({ ...form, name: e.target.value })} />
            <Select value={form.classId} onChange={(e) => setForm({ ...form, classId: e.target.value })}>
              {core.classes.map((c) => <option key={c.id} value={c.id}>{c.name}</option>)}
            </Select>
            <Select value={form.gender} onChange={(e) => setForm({ ...form, gender: e.target.value })}>
              <option value="">Gender — unspecified</option>
              <option value="Male">Male</option>
              <option value="Female">Female</option>
            </Select>
            <Input placeholder="Guardian name" value={form.guardianName} onChange={(e) => setForm({ ...form, guardianName: e.target.value })} />
            <Input placeholder="Guardian email" value={form.guardianContact} onChange={(e) => setForm({ ...form, guardianContact: e.target.value })} />
            <Btn onClick={saveStudent} className="w-full justify-center">{editingId ? "Save changes" : "Enroll learner"}</Btn>
          </div>
        </Modal>
      )}

      {credModal && (
        <Modal title="Guardian sign-in credentials ready" onClose={() => setCredModal(null)}>
          <div className="space-y-3">
            <p className="sms-fs-13 sms-text-ink-60">{credModal.name} is enrolled. Share these sign-in details with {credModal.guardianName || "the parent/guardian"} so they can access the Parent/Guardian portal:</p>
            <div className="border sms-border-sand bg-white px-3 py-2 sms-fs-13 space-y-1">
              <div>Username: <strong>{credModal.guardianUsername}</strong></div>
              <div>Temporary password: <strong>{credModal.guardianTempPassword}</strong></div>
            </div>
          </div>
        </Modal>
      )}
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/* Admin: Teachers                                                         */
/* ---------------------------------------------------------------------- */

function TeachersAdmin({ core, updateCore, school }) {
  const emptyForm = { name: "", phone: "", subjects: [], classes: [], signatureImage: null };
  const [modal, setModal] = useState(false);
  const [editingId, setEditingId] = useState(null);
  const [form, setForm] = useState(emptyForm);
  const [credModal, setCredModal] = useState(null); // teacher record just issued credentials

  const toggle = (arr, val) => (arr.includes(val) ? arr.filter((v) => v !== val) : [...arr, val]);

  const openAdd = () => { setEditingId(null); setForm(emptyForm); setModal(true); };
  const openEdit = (t) => { setEditingId(t.id); setForm({ name: t.name, phone: t.phone || "", subjects: [...t.subjects], classes: [...t.classes], signatureImage: t.signatureImage || null }); setModal(true); };

  const onSignatureChange = (e) => {
    const file = e.target.files?.[0];
    if (!file || !file.type.startsWith("image/")) return;
    const reader = new FileReader();
    reader.onload = () => setForm((f) => ({ ...f, signatureImage: reader.result }));
    reader.readAsDataURL(file);
  };

  const saveTeacher = () => {
    if (!form.name.trim()) return;
    if (editingId) {
      const existing = core.teachers.find((t) => t.id === editingId);
      const needsCreds = !existing.username;
      const updated = {
        ...existing, ...form,
        username: existing.username || makeUsername(form.name),
        tempPassword: existing.tempPassword || generateTempPassword(),
      };
      updateCore({ ...core, teachers: core.teachers.map((t) => (t.id === editingId ? updated : t)) });
      setModal(false); setEditingId(null); setForm(emptyForm);
      if (needsCreds) setCredModal(updated);
    } else {
      const newTeacher = { id: uid("t"), ...form, username: makeUsername(form.name), tempPassword: generateTempPassword() };
      updateCore({ ...core, teachers: [...core.teachers, newTeacher] });
      setModal(false); setForm(emptyForm);
      setCredModal(newTeacher);
    }
  };
  const removeTeacher = (id) => updateCore({ ...core, teachers: core.teachers.filter((t) => t.id !== id) });

  const sendCreds = (t) => {
    if (!t.phone) return;
    const msg = credentialsMessage(school?.name, t.name, t.username, t.tempPassword);
    window.open(waLink(t.phone, msg), "_blank");
  };

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between">
        <h2 className="font-serif sms-fs-20">Teachers</h2>
        <Btn onClick={openAdd}><Icon name="Plus" size={14} /> Enroll learning area teacher</Btn>
      </div>
      <div className="grid md:grid-cols-2 gap-3">
        {core.teachers.map((t) => (
          <Panel key={t.id}>
            <div className="flex justify-between items-start gap-2">
              <div className="min-w-0">
                <div className="font-medium sms-fs-14">{t.name}</div>
                <div className="sms-fs-12 sms-text-ink-60 mt-1">Learning area(s): {t.subjects.join(", ") || "—"}</div>
                <div className="sms-fs-12 sms-text-ink-60">Classes: {t.classes.map((cid) => core.classes.find((c) => c.id === cid)?.name).join(", ") || "—"}</div>
                <div className="sms-fs-12 sms-text-ink-60">WhatsApp: {t.phone || "—"}</div>
                {t.username && <div className="sms-fs-12 sms-text-ink-50">Username: {t.username}</div>}
              </div>
              <div className="flex flex-col items-end gap-2 shrink-0">
                <div className="flex gap-2">
                  <button onClick={() => openEdit(t)} className="sms-text-ink-50 sms-hover-text-ink"><Icon name="Pencil" size={14} /></button>
                  <button onClick={() => removeTeacher(t.id)} className="sms-text-rust-70 sms-hover-text-rust"><Icon name="Trash2" size={14} /></button>
                </div>
                {t.username && (
                  <Btn variant="ghost" onClick={() => sendCreds(t)} className={!t.phone ? "opacity-40 pointer-events-none" : ""}>
                    <Icon name="Send" size={13} /> WhatsApp
                  </Btn>
                )}
              </div>
            </div>
          </Panel>
        ))}
      </div>

      {modal && (
        <Modal title={editingId ? "Edit teacher" : "Enroll learning area teacher"} onClose={() => setModal(false)}>
          <div className="space-y-3">
            <Input placeholder="Full name" value={form.name} onChange={(e) => setForm({ ...form, name: e.target.value })} />
            <Input placeholder="WhatsApp number (e.g. +2547XXXXXXXX)" value={form.phone} onChange={(e) => setForm({ ...form, phone: e.target.value })} />
            <div>
              <div className="sms-fs-12 sms-text-ink-60 mb-1">Learning area(s)</div>
              <div className="flex flex-wrap gap-1.5">
                {core.subjects.map((s) => (
                  <button key={s} type="button" onClick={() => setForm({ ...form, subjects: toggle(form.subjects, s) })}
                    className={`px-2.5 py-1 sms-fs-12 border ${form.subjects.includes(s) ? "sms-bg-ink text-white sms-border-ink" : "sms-border-sand"}`}>
                    {s}
                  </button>
                ))}
              </div>
            </div>
            <div>
              <div className="sms-fs-12 sms-text-ink-60 mb-1">Classes</div>
              <div className="flex flex-wrap gap-1.5">
                {core.classes.map((c) => (
                  <button key={c.id} type="button" onClick={() => setForm({ ...form, classes: toggle(form.classes, c.id) })}
                    className={`px-2.5 py-1 sms-fs-12 border ${form.classes.includes(c.id) ? "sms-bg-ink text-white sms-border-ink" : "sms-border-sand"}`}>
                    {c.name}
                  </button>
                ))}
              </div>
            </div>
            <div>
              <div className="sms-fs-12 sms-text-ink-60 mb-1">Signature (used on report cards when this teacher is a class teacher)</div>
              <div className="flex items-center gap-3">
                <div className="w-20 h-10 border sms-border-sand bg-white flex items-center justify-center overflow-hidden shrink-0">
                  {form.signatureImage ? <img src={form.signatureImage} alt="Signature preview" className="w-full h-full object-contain" /> : <span className="sms-fs-11 sms-text-ink-30">None</span>}
                </div>
                <label className="sms-fs-13">
                  <span className="inline-block px-3 py-2 border sms-border-ink-30 cursor-pointer sms-hover-border-ink">Upload signature</span>
                  <input type="file" accept="image/*" onChange={onSignatureChange} className="hidden" />
                </label>
              </div>
            </div>
            <Btn onClick={saveTeacher} className="w-full justify-center">{editingId ? "Save changes" : "Enroll teacher"}</Btn>
          </div>
        </Modal>
      )}

      {credModal && (
        <Modal title="Sign-in credentials ready" onClose={() => setCredModal(null)}>
          <div className="space-y-3">
            <p className="sms-fs-13 sms-text-ink-60">{credModal.name}'s account has been created. Send these credentials via WhatsApp:</p>
            <div className="border sms-border-sand bg-white px-3 py-2 sms-fs-13 space-y-1">
              <div>Username: <strong>{credModal.username}</strong></div>
              <div>Temporary password: <strong>{credModal.tempPassword}</strong></div>
            </div>
            {credModal.phone ? (
              <Btn onClick={() => sendCreds(credModal)} className="w-full justify-center"><Icon name="Send" size={14} /> Send via WhatsApp</Btn>
            ) : (
              <p className="sms-fs-12 sms-text-rust">No WhatsApp number on file — edit this teacher to add one, then send from their card.</p>
            )}
          </div>
        </Modal>
      )}
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/* Teacher: Grade Entry                                                    */
/* ---------------------------------------------------------------------- */

function GradeEntry({ core, academics, updateAcademics, role, teacherId }) {
  const teacher = role === "teacher" ? core.teachers.find((t) => t.id === teacherId) : null;
  const myClasses = teacher ? core.classes.filter((c) => teacher.classes.includes(c.id)) : core.classes;
  const subjectsList = teacher ? teacher.subjects : core.subjects;
  const [classId, setClassId] = useState(myClasses[0]?.id || "");
  const [subject, setSubject] = useState(subjectsList[0] || "");
  const [term, setTerm] = useState(TERMS[0]);
  const [assessmentType, setAssessmentType] = useState("midterm");
  const [totalMarks, setTotalMarks] = useState(100);
  const [commentVersion, setCommentVersion] = useState({});
  const [drafts, setDrafts] = useState({});
  const [savedFlash, setSavedFlash] = useState(false);
  const [importSummary, setImport: Template Updates
description: Learn how template updates work for authors and consumers.
---

Railway supports automatic update notifications for templates, allowing template authors to push changes to users who have deployed their templates.

## For template authors

As a template author, you can push updates to all users who have deployed your template. When you merge changes to the root branch (typically `main` or `master`) of your template's GitHub repository, Railway will automatically detect these changes and notify users who have deployed your template that an update is available.

Users will receive a notification about the update and can choose to apply it to their deployment when they're ready.

<Banner variant="info">
**Best Practice**: Keep your template's changelog up to date and document breaking changes clearly so that consumers understand what's changing when they receive update notifications.
</Banner>

### Requirements

- Your template must be based on a GitHub repository
- Updates are triggered when changes are merged to the root branch (`main` or `master`)
- Docker image-based templates cannot be automatically updated through this mechanism

### Update workflow

1. Make changes to your template's GitHub repository
2. Merge changes to the root branch
3. Railway detects the changes automatically
4. Users who deployed your template receive an update notification
5. Users can choose when to apply the update

## For template consumers

When you deploy any services from a template based on a GitHub repo, Railway will check to see if the template has been updated by its creator.

If an upstream update is available, you will receive a notification. You can then choose to apply the update to your deployment when you're ready.

### How updates work

- Railway monitors the template's source repository for changes
- When updates are detected, you'll see a notification in your project
- Updates are opt-in, you control when to apply them
- Review the template author's changelog before updating to understand what's changing

### Limitations

- This feature only works for services based on GitHub repositories
- At this time, there is no mechanism to check for updates to Docker images from which services may be sourced

<Banner variant="info">
If you're curious, you can read more about how we built updatable templates in this <Link href="https://blog.railway.com/p/updatable-starters" target="_blank">blog post</Link>.
</Banner>
