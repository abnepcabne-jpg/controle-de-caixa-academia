/* Caixa Academia — LocalStorage SPA (pronto pra rodar)
   - Login / Logout
   - Entradas / Gastos
   - Fechamento diário
   - Histórico (período)
   - Histórico por aluno
   - PDF (dia, período, mês, aluno)
*/

const DB_KEY = "caixa_academia_db_v1";
const SESSION_KEY = "caixa_academia_session_v1";

/* -------------------- Helpers -------------------- */
const $ = (sel) => document.querySelector(sel);

function nowISODate() {
  const d = new Date();
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, "0");
  const day = String(d.getDate()).padStart(2, "0");
  return `${y}-${m}-${day}`;
}
function nowTime() {
  const d = new Date();
  const hh = String(d.getHours()).padStart(2, "0");
  const mm = String(d.getMinutes()).padStart(2, "0");
  return `${hh}:${mm}`;
}
function uid() {
  return Math.random().toString(16).slice(2) + Date.now().toString(16);
}
function money(n) {
  const v = Number(n || 0);
  return v.toLocaleString("pt-BR", { style: "currency", currency: "BRL" });
}
function safeLower(s){ return String(s||"").trim().toLowerCase(); }

function toast(msg) {
  const el = $("#toast");
  el.textContent = msg;
  el.style.display = "block";
  clearTimeout(toast._t);
  toast._t = setTimeout(() => (el.style.display = "none"), 2600);
}

function loadDB() {
  const raw = localStorage.getItem(DB_KEY);
  if (!raw) return null;
  try { return JSON.parse(raw); } catch { return null; }
}
function saveDB(db) {
  localStorage.setItem(DB_KEY, JSON.stringify(db));
}

function hashPass(p) {
  // Simples (não-criptográfico). Para algo sério: backend + hash real.
  let h = 0;
  const s = String(p || "");
  for (let i = 0; i < s.length; i++) h = (h * 31 + s.charCodeAt(i)) >>> 0;
  return `h${h.toString(16)}`;
}

/* -------------------- DB Model -------------------- */
function seedDB() {
  const db = {
    config: { name: "Caixa Academia", logo: "CA" },
    users: [
      { username: "admin", pass: hashPass("1234"), role: "Admin" }
    ],
    movements: {
      // date: { in:[], out:[], closed:false, closedAt:null, closedBy:null }
    },
    closures: [
      // {date, totals:{...}, closedAt, closedBy}
    ]
  };
  saveDB(db);
  return db;
}

function getDB() {
  return loadDB() || seedDB();
}

function getSession() {
  const raw = localStorage.getItem(SESSION_KEY);
  if (!raw) return null;
  try { return JSON.parse(raw); } catch { return null; }
}
function setSession(s) {
  localStorage.setItem(SESSION_KEY, JSON.stringify(s));
}
function clearSession() {
  localStorage.removeItem(SESSION_KEY);
}

function ensureDay(db, date) {
  if (!db.movements[date]) {
    db.movements[date] = { in: [], out: [], closed: false, closedAt: null, closedBy: null };
  }
  return db.movements[date];
}

function isAdmin(session) {
  return session?.role === "Admin";
}

/* -------------------- Views / Tabs -------------------- */
function show(el, yes) {
  el.classList.toggle("hidden", !yes);
}
function setTab(tabId) {
  const tabs = document.querySelectorAll(".tab");
  tabs.forEach(t => t.classList.toggle("active", t.dataset.tab === tabId));

  ["tabMov","tabHistory","tabStudents","tabSettings"].forEach(id => {
    show($("#"+id), id === tabId);
  });

  if (tabId === "tabStudents") refreshStudents();
}

/* -------------------- Calculations -------------------- */
function totalsForDay(day) {
  const sumByPay = (arr) => arr.reduce((acc, it) => {
    const k = it.pay || "Espécie";
    acc[k] = (acc[k] || 0) + Number(it.value || 0);
    return acc;
  }, { Espécie:0, Pix:0, Cartão:0 });

  const inBy = sumByPay(day.in);
  const outBy = sumByPay(day.out);

  const inTotal = inBy["Espécie"] + inBy["Pix"] + inBy["Cartão"];
  const outTotal = outBy["Espécie"] + outBy["Pix"] + outBy["Cartão"];
  const balance = inTotal - outTotal;

  return { inBy, outBy, inTotal, outTotal, balance };
}

/* -------------------- Rendering -------------------- */
function refreshHeader(db, session) {
  $("#brandTitle").textContent = db.config?.name || "Caixa Academia";
  $("#logoBox").textContent = (db.config?.logo || "CA").toUpperCase();

  if (session) {
    $("#userName").textContent = session.username;
    $("#userRole").textContent = session.role;
    show($("#userPill"), true);
    show($("#btnLogout"), true);
    show($("#btnChangePass"), true);
  } else {
    show($("#userPill"), false);
    show($("#btnLogout"), false);
    show($("#btnChangePass"), false);
  }
}

function refreshKPIs(db, date) {
  const day = ensureDay(db, date);
  const t = totalsForDay(day);

  $("#kpiIn").textContent = money(t.inTotal);
  $("#kpiInSub").textContent = `${day.in.length} lançamentos`;

  $("#kpiOut").textContent = money(t.outTotal);
  $("#kpiOutSub").textContent = `${day.out.length} lançamentos`;

  $("#kpiBalance").textContent = money(t.balance);
  $("#kpiBalanceSub").textContent = `Espécie ${money(t.inBy["Espécie"]-t.outBy["Espécie"])} • Pix ${money(t.inBy["Pix"]-t.outBy["Pix"])} • Cartão ${money(t.inBy["Cartão"]-t.outBy["Cartão"])}`;

  $("#kpiStatus").textContent = day.closed ? "Fechado" : "Aberto";
  $("#kpiStatusSub").textContent = day.closed
    ? `Fechado às ${day.closedAt} por ${day.closedBy}`
    : "Fechamento diário";
}

function renderDayTables(db, date) {
  const day = ensureDay(db, date);

  const tbIn = $("#tbodyIn");
  tbIn.innerHTML = "";
  day.in
    .slice()
    .sort((a,b)=> (a.time>b.time? -1:1))
    .forEach(it => {
      const tr = document.createElement("tr");
      tr.innerHTML = `
        <td>${it.time}</td>
        <td>${it.student || "-"}</td>
        <td>${it.type}</td>
        <td>${it.pay}</td>
        <td><b>${money(it.value)}</b></td>
        <td>${it.receiver}</td>
        <td>${day.closed ? "" : `<span class="pill-del" data-del-in="${it.id}">Excluir</span>`}</td>
      `;
      tbIn.appendChild(tr);
    });

  const tbOut = $("#tbodyOut");
  tbOut.innerHTML = "";
  day.out
    .slice()
    .sort((a,b)=> (a.time>b.time? -1:1))
    .forEach(it => {
      const tr = document.createElement("tr");
      tr.innerHTML = `
        <td>${it.time}</td>
        <td>${it.cat}</td>
        <td>${it.desc}</td>
        <td>${it.pay}</td>
        <td><b>${money(it.value)}</b></td>
        <td>${day.closed ? "" : `<span class="pill-del" data-del-out="${it.id}">Excluir</span>`}</td>
      `;
      tbOut.appendChild(tr);
    });

  // binds delete
  document.querySelectorAll("[data-del-in]").forEach(btn=>{
    btn.onclick = () => deleteMovement("in", btn.getAttribute("data-del-in"));
  });
  document.querySelectorAll("[data-del-out]").forEach(btn=>{
    btn.onclick = () => deleteMovement("out", btn.getAttribute("data-del-out"));
  });

  // disable forms if closed
  const disabled = day.closed;
  ["#btnAddIn","#btnQuickDaily","#btnAddOut"].forEach(id => {
    const el = $(id);
    el.disabled = disabled;
    el.style.opacity = disabled ? .5 : 1;
    el.style.cursor = disabled ? "not-allowed" : "pointer";
  });
}

/* -------------------- CRUD -------------------- */
function deleteMovement(kind, id) {
  const db = getDB();
  const date = nowISODate();
  const day = ensureDay(db, date);
  if (day.closed) return toast("Caixa do dia está fechado. Não é possível excluir.");

  const list = kind === "in" ? day.in : day.out;
  const idx = list.findIndex(x => x.id === id);
  if (idx >= 0) {
    list.splice(idx,1);
    saveDB(db);
    refreshAll();
    toast("Lançamento excluído.");
  }
}

function addIncome(payload) {
  const db = getDB();
  const session = getSession();
  const date = nowISODate();
  const day = ensureDay(db, date);
  if (day.closed) return toast("Caixa do dia está fechado. Não é possível lançar.");

  day.in.push({
    id: uid(),
    date,
    time: nowTime(),
    student: payload.student || "",
    type: payload.type,
    pay: payload.pay,
    value: Number(payload.value),
    receiver: payload.receiver,
    note: payload.note || "",
    createdBy: session.username
  });

  saveDB(db);
  refreshAll();
  toast("Entrada registrada.");
}

function addExpense(payload) {
  const db = getDB();
  const session = getSession();
  const date = nowISODate();
  const day = ensureDay(db, date);
  if (day.closed) return toast("Caixa do dia está fechado. Não é possível lançar.");

  day.out.push({
    id: uid(),
    date,
    time: nowTime(),
    cat: payload.cat,
    desc: payload.desc,
    pay: payload.pay,
    value: Number(payload.value),
    createdBy: session.username
  });

  saveDB(db);
  refreshAll();
  toast("Gasto registrado.");
}

function closeDay() {
  const db = getDB();
  const session = getSession();
  const date = nowISODate();
  const day = ensureDay(db, date);

  if (day.closed) return toast("O caixa de hoje já está fechado.");

  const t = totalsForDay(day);

  day.closed = true;
  day.closedAt = `${nowTime()}`;
  day.closedBy = session.username;

  // salvar snapshot em closures (para histórico e auditoria)
  const existing = db.closures.find(c=>c.date===date);
  const snapshot = {
    date,
    totals: {
      inTotal: t.inTotal,
      outTotal: t.outTotal,
      balance: t.balance,
      inBy: t.inBy,
      outBy: t.outBy
    },
    closedAt: day.closedAt,
    closedBy: day.closedBy
  };
  if (!existing) db.closures.push(snapshot);
  else Object.assign(existing, snapshot);

  saveDB(db);
  refreshAll();
  toast("Caixa do dia fechado.");
}

/* -------------------- History -------------------- */
function movementsInRange(db, from, to) {
  const out = [];
  const start = new Date(from);
  const end = new Date(to);
  if (isNaN(start) || isNaN(end)) return out;

  // inclusive
  for (let d = new Date(start); d <= end; d.setDate(d.getDate()+1)) {
    const dt = d.toISOString().slice(0,10);
    const day = db.movements[dt];
    if (!day) continue;

    day.in.forEach(x => out.push({ kind:"Entrada", date:dt, detail: `${x.type}${x.student?` • ${x.student}`:""}`, pay:x.pay, value:x.value, operator:x.receiver }));
    day.out.forEach(x => out.push({ kind:"Gasto", date:dt, detail: `${x.cat} • ${x.desc}`, pay:x.pay, value:-Math.abs(x.value), operator:x.createdBy }));
  }
  // newest first
  out.sort((a,b)=> (a.date<b.date? 1 : a.date>b.date? -1 : 0));
  return out;
}

function applyHistory() {
  const db = getDB();
  const from = $("#hisFrom").value;
  const to = $("#hisTo").value;
  const filter = $("#hisFilter").value;

  if (!from || !to) return toast("Selecione data inicial e final.");

  let list = movementsInRange(db, from, to);
  if (filter === "in") list = list.filter(x=>x.kind==="Entrada");
  if (filter === "out") list = list.filter(x=>x.kind==="Gasto");

  // summary
  const totalIn = list.filter(x=>x.kind==="Entrada").reduce((s,x)=>s + Math.abs(Number(x.value)), 0);
  const totalOut = list.filter(x=>x.kind==="Gasto").reduce((s,x)=>s + Math.abs(Number(x.value)), 0);
  const balance = totalIn - totalOut;

  $("#historySummary").innerHTML = `
    <div><b>Período:</b> ${from} → ${to}</div>
    <div style="margin-top:6px"><b>Entradas:</b> ${money(totalIn)} • <b>Gastos:</b> ${money(totalOut)} • <b>Saldo:</b> ${money(balance)}</div>
    <div class="muted mini" style="margin-top:6px">Total de registros: ${list.length}</div>
  `;

  // table
  const tb = $("#tbodyHistory");
  tb.innerHTML = "";
  list.forEach(x=>{
    const tr = document.createElement("tr");
    tr.innerHTML = `
      <td>${x.date}</td>
      <td>${x.kind}</td>
      <td>${x.detail}</td>
      <td>${x.pay}</td>
      <td><b>${money(x.value)}</b></td>
      <td>${x.operator || "-"}</td>
    `;
    tb.appendChild(tr);
  });
}

/* -------------------- Students -------------------- */
let selectedStudent = "";

function getAllStudents(db) {
  const set = new Set();
  Object.values(db.movements).forEach(day=>{
    day?.in?.forEach(x=>{
      const s = String(x.student||"").trim();
      if (s) set.add(s);
    });
  });
  return Array.from(set).sort((a,b)=>a.localeCompare(b,"pt-BR"));
}

function refreshStudents() {
  const db = getDB();
  const q = safeLower($("#studentSearch").value);
  const list = getAllStudents(db).filter(n => safeLower(n).includes(q));

  const box = $("#studentList");
  box.innerHTML = "";
  if (!list.length) {
    box.innerHTML = `<div class="muted mini">Nenhum aluno encontrado (ou ainda não há lançamentos com nome).</div>`;
  } else {
    list.forEach(name=>{
      const el = document.createElement("div");
      el.className = "list-item";
      el.innerHTML = `<b>${name}</b><span class="muted mini">ver</span>`;
      el.onclick = ()=> selectStudent(name);
      box.appendChild(el);
    });
  }

  if (selectedStudent && !getAllStudents(db).includes(selectedStudent)) {
    selectedStudent = "";
    $("#studentSelected").value = "";
    $("#tbodyStudent").innerHTML = "";
  }
}

function selectStudent(name) {
  selectedStudent = name;
  $("#studentSelected").value = name;

  const db = getDB();
  const rows = [];

  Object.values(db.movements).forEach(day=>{
    day?.in?.forEach(x=>{
      if (String(x.student||"").trim() === name) {
        rows.push({
          date: x.date,
          type: x.type,
          pay: x.pay,
          value: x.value,
          receiver: x.receiver
        });
      }
    });
  });

  rows.sort((a,b)=> (a.date<b.date? 1 : a.date>b.date? -1 : 0));

  const tb = $("#tbodyStudent");
  tb.innerHTML = "";
  rows.forEach(r=>{
    const tr = document.createElement("tr");
    tr.innerHTML = `
      <td>${r.date}</td>
      <td>${r.type}</td>
      <td>${r.pay}</td>
      <td><b>${money(r.value)}</b></td>
      <td>${r.receiver}</td>
    `;
    tb.appendChild(tr);
  });
}

/* -------------------- Users -------------------- */
function renderUsersTable() {
  const db = getDB();
  const session = getSession();
  const tb = $("#tbodyUsers");
  tb.innerHTML = "";

  db.users.forEach(u=>{
    const tr = document.createElement("tr");
    const canDel = isAdmin(session) && u.username !== "admin";
    tr.innerHTML = `
      <td><b>${u.username}</b></td>
      <td>${u.role}</td>
      <td>${canDel ? `<span class="pill-del" data-del-user="${u.username}">Remover</span>` : ""}</td>
    `;
    tb.appendChild(tr);
  });

  document.querySelectorAll("[data-del-user]").forEach(btn=>{
    btn.onclick = () => {
      const user = btn.getAttribute("data-del-user");
      const db2 = getDB();
      db2.users = db2.users.filter(x=>x.username !== user);
      saveDB(db2);
      renderUsersTable();
      toast("Usuário removido.");
    };
  });

  // disable create if not admin
  const disable = !isAdmin(session);
  ["#newUser","#newPass","#newRole","#btnAddUser"].forEach(id=>{
    const el = $(id);
    el.disabled = disable;
    el.style.opacity = disable ? .5 : 1;
  });
}

/* -------------------- PDFs -------------------- */
function requirePDFReady() {
  return (window.jspdf && window.jspdf.jsPDF);
}

function pdfHeader(doc, db, title, subtitle) {
  doc.setFontSize(14);
  doc.text(db.config?.name || "Caixa Academia", 14, 14);
  doc.setFontSize(11);
  doc.text(title, 14, 22);
  if (subtitle) {
    doc.setFontSize(10);
    doc.text(subtitle, 14, 28);
  }
}

function exportDayPDF() {
  if (!requirePDFReady()) return toast("Biblioteca PDF ainda carregando. Tente novamente em 2s.");

  const db = getDB();
  const date = nowISODate();
  const day = ensureDay(db, date);
  const t = totalsForDay(day);

  const { jsPDF } = window.jspdf;
  const doc = new jsPDF();

  pdfHeader(doc, db, `Relatório do Dia: ${date}`, `Status: ${day.closed ? "Fechado" : "Aberto"}`);

  doc.setFontSize(10);
  doc.text(`Entradas: ${money(t.inTotal)}  |  Gastos: ${money(t.outTotal)}  |  Saldo: ${money(t.balance)}`, 14, 36);

  doc.text(`Por forma (Entradas): Espécie ${money(t.inBy["Espécie"])} • Pix ${money(t.inBy["Pix"])} • Cartão ${money(t.inBy["Cartão"])}`, 14, 44);
  doc.text(`Por forma (Gastos): Espécie ${money(t.outBy["Espécie"])} • Pix ${money(t.outBy["Pix"])} • Cartão ${money(t.outBy["Cartão"])}`, 14, 50);

  doc.autoTable({
    startY: 58,
    head: [["Hora","Aluno","Tipo","Forma","Valor","Recebido por","Obs"]],
    body: day.in.map(x=>[x.time, x.student||"-", x.type, x.pay, money(x.value), x.receiver, x.note||""]),
    styles: { fontSize: 9 }
  });

  const y2 = doc.lastAutoTable.finalY + 8;
  doc.autoTable({
    startY: y2,
    head: [["Hora","Categoria","Descrição","Pago por","Valor"]],
    body: day.out.map(x=>[x.time, x.cat, x.desc, x.pay, money(x.value)]),
    styles: { fontSize: 9 }
  });

  doc.save(`caixa_${date}.pdf`);
}

function exportMonthPDF() {
  if (!requirePDFReady()) return toast("Biblioteca PDF ainda carregando. Tente novamente em 2s.");

  const db = getDB();
  const today = nowISODate();
  const ym = today.slice(0,7); // YYYY-MM
  const from = `${ym}-01`;
  const to = today;

  exportPeriodPDF(from, to, `Relatório do Mês (${ym})`);
}

function exportPeriodPDF(from, to, titleOverride) {
  if (!requirePDFReady()) return toast("Biblioteca PDF ainda carregando. Tente novamente em 2s.");

  const db = getDB();
  const list = movementsInRange(db, from, to);

  const totalIn = list.filter(x=>x.kind==="Entrada").reduce((s,x)=>s + Math.abs(Number(x.value)), 0);
  const totalOut = list.filter(x=>x.kind==="Gasto").reduce((s,x)=>s + Math.abs(Number(x.value)), 0);
  const balance = totalIn - totalOut;

  const { jsPDF } = window.jspdf;
  const doc = new jsPDF();

  const title = titleOverride || `Relatório do Período`;
  pdfHeader(doc, db, title, `${from} → ${to}`);

  doc.setFontSize(10);
  doc.text(`Entradas: ${money(totalIn)}  |  Gastos: ${money(totalOut)}  |  Saldo: ${money(balance)}`, 14, 36);

  doc.autoTable({
    startY: 44,
    head: [["Data","Tipo","Detalhe","Forma","Valor","Operador"]],
    body: list.map(x=>[x.date, x.kind, x.detail, x.pay, money(x.value), x.operator||"-"]),
    styles: { fontSize: 9 }
  });

  doc.save(`relatorio_${from}_a_${to}.pdf`);
}

function exportStudentPDF() {
  if (!requirePDFReady()) return toast("Biblioteca PDF ainda carregando. Tente novamente em 2s.");
  if (!selectedStudent) return toast("Selecione um aluno primeiro.");

  const db = getDB();
  const rows = [];

  Object.values(db.movements).forEach(day=>{
    day?.in?.forEach(x=>{
      if (String(x.student||"").trim() === selectedStudent) {
        rows.push([x.date, x.time, x.type, x.pay, money(x.value), x.receiver, x.note||""]);
      }
    });
  });

  rows.sort((a,b)=> (a[0]<b[0]? 1 : a[0]>b[0]? -1 : 0));

  const total = rows.reduce((s,r)=>{
    const val = Number(String(r[4]).replace(/[^\d,-]/g,"").replace(".","").replace(",",".") || 0);
    return s + val;
  }, 0);

  const { jsPDF } = window.jspdf;
  const doc = new jsPDF();

  pdfHeader(doc, db, `Histórico do Aluno: ${selectedStudent}`, `Total recebido: ${money(total)}`);

  doc.autoTable({
    startY: 34,
    head: [["Data","Hora","Tipo","Forma","Valor","Recebido por","Obs"]],
    body: rows,
    styles: { fontSize: 9 }
  });

  doc.save(`aluno_${selectedStudent.replace(/\s+/g,"_")}.pdf`);
}

/* -------------------- Auth -------------------- */
function login() {
  const db = getDB();
  const u = safeLower($("#loginUser").value);
  const p = $("#loginPass").value;

  if (!u || !p) return toast("Informe usuário e senha.");

  const user = db.users.find(x => safeLower(x.username) === u);
  if (!user) return toast("Usuário não encontrado.");

  if (user.pass !== hashPass(p)) return toast("Senha incorreta.");

  setSession({ username: user.username, role: user.role });
  boot();
  toast("Bem-vindo!");
}

function logout() {
  clearSession();
  boot();
  toast("Você saiu.");
}

function openPassModal() {
  $("#oldPass").value = "";
  $("#newPass1").value = "";
  $("#newPass2").value = "";
  show($("#modalPass"), true);
}
function closePassModal() {
  show($("#modalPass"), false);
}

function saveNewPassword() {
  const db = getDB();
  const session = getSession();
  const oldP = $("#oldPass").value;
  const n1 = $("#newPass1").value;
  const n2 = $("#newPass2").value;

  if (!oldP || !n1 || !n2) return toast("Preencha todos os campos.");
  if (n1.length < 4) return toast("Nova senha deve ter pelo menos 4 caracteres.");
  if (n1 !== n2) return toast("Confirmação não confere.");

  const user = db.users.find(x => x.username === session.username);
  if (!user) return toast("Sessão inválida.");
  if (user.pass !== hashPass(oldP)) return toast("Senha atual incorreta.");

  user.pass = hashPass(n1);
  saveDB(db);
  closePassModal();
  toast("Senha alterada com sucesso.");
}

/* -------------------- Config -------------------- */
function loadConfigUI(db) {
  $("#cfgName").value = db.config?.name || "";
  $("#cfgLogo").value = db.config?.logo || "";
}
function saveConfig() {
  const db = getDB();
  db.config.name = String($("#cfgName").value || "Caixa Academia").trim();
  db.config.logo = String($("#cfgLogo").value || "CA").trim().slice(0,2).toUpperCase();
  saveDB(db);
  refreshHeader(db, getSession());
  toast("Configurações salvas.");
}

function addUser() {
  const db = getDB();
  const session = getSession();
  if (!isAdmin(session)) return toast("Apenas admin pode criar usuário.");

  const username = String($("#newUser").value||"").trim();
  const pass = $("#newPass").value;
  const role = $("#newRole").value;

  if (!username || !pass) return toast("Preencha usuário e senha.");
  if (username.length < 3) return toast("Usuário muito curto (mín. 3).");
  if (pass.length < 4) return toast("Senha muito curta (mín. 4).");
  if (db.users.some(u=>safeLower(u.username)===safeLower(username))) return toast("Usuário já existe.");

  db.users.push({ username, pass: hashPass(pass), role });
  saveDB(db);
  $("#newUser").value = "";
  $("#newPass").value = "";
  renderUsersTable();
  toast("Usuário criado.");
}

/* -------------------- UI Bindings -------------------- */
function bindUI() {
  // login
  $("#btnLogin").onclick = login;
  $("#loginPass").addEventListener("keydown", (e)=>{ if (e.key==="Enter") login(); });
  $("#loginUser").addEventListener("keydown", (e)=>{ if (e.key==="Enter") $("#loginPass").focus(); });

  $("#btnLogout").onclick = logout;

  // pass modal
  $("#btnChangePass").onclick = openPassModal;
  $("#btnCloseModal").onclick = closePassModal;
  $("#modalPass").addEventListener("click", (e)=>{ if (e.target.id==="modalPass") closePassModal(); });
  $("#btnSavePass").onclick = saveNewPassword;

  // tabs
  document.querySelectorAll(".tab").forEach(t=>{
    t.onclick = ()=> setTab(t.dataset.tab);
  });

  // prefill receiver with logged user
  const session = getSession();
  $("#inReceiver").value = session ? session.username : "";

  // add income
  $("#btnAddIn").onclick = () => {
    const student = $("#inStudent").value.trim();
    const type = $("#inType").value;
    const pay = $("#inPay").value;
    const value = Number($("#inValue").value);
    const receiver = $("#inReceiver").value.trim() || (getSession()?.username || "");
    const note = $("#inNote").value.trim();

    if (!receiver) return toast("Preencha 'Recebido por'.");
    if (!value || value <= 0) return toast("Informe um valor válido.");

    addIncome({ student, type, pay, value, receiver, note });

    $("#inStudent").value = "";
    $("#inValue").value = "";
    $("#inNote").value = "";
    $("#inPay").value = "Espécie";
  };

  $("#btnQuickDaily").onclick = () => {
    const v = prompt("Valor da diária (ex: 20):");
    if (v === null) return;
    const value = Number(String(v).replace(",", "."));
    if (!value || value <= 0) return toast("Valor inválido.");
    const receiver = $("#inReceiver").value.trim() || (getSession()?.username || "");
    addIncome({ student:"", type:"Diária", pay:"Espécie", value, receiver, note:"Diária rápida" });
  };

  // add expense
  $("#btnAddOut").onclick = () => {
    const cat = $("#outCat").value;
    const pay = $("#outPay").value;
    const desc = $("#outDesc").value.trim();
    const value = Number($("#outValue").value);

    if (!desc) return toast("Informe a descrição do gasto.");
    if (!value || value <= 0) return toast("Informe um valor válido.");

    addExpense({ cat, pay, desc, value });
    $("#outDesc").value = "";
    $("#outValue").value = "";
    $("#outPay").value = "Espécie";
  };

  // close day
  $("#btnCloseDay").onclick = () => {
    const ok = confirm("Deseja FECHAR o caixa de hoje? Isso desativa lançamentos/exclusões do dia.");
    if (ok) closeDay();
  };

  // PDFs
  $("#btnExportDayPDF").onclick = exportDayPDF;
  $("#btnPDFMonth").onclick = exportMonthPDF;

  $("#btnPDFPeriod").onclick = () => {
    const from = $("#hisFrom").value;
    const to = $("#hisTo").value;
    if (!from || !to) return toast("Selecione data inicial e final.");
    exportPeriodPDF(from, to);
  };

  $("#btnPDFStudent").onclick = exportStudentPDF;

  // history quick ranges
  $("#btnApplyHistory").onclick = applyHistory;
  $("#btnThisMonth").onclick = () => {
    const today = new Date();
    const y = today.getFullYear();
    const m = String(today.getMonth()+1).padStart(2,"0");
    const from = `${y}-${m}-01`;
    const to = nowISODate();
    $("#hisFrom").value = from;
    $("#hisTo").value = to;
    applyHistory();
  };
  $("#btnThisWeek").onclick = () => {
    const d = new Date();
    const day = d.getDay(); // 0 dom
    const diff = (day === 0 ? 6 : day-1); // semana começando segunda
    const start = new Date(d);
    start.setDate(d.getDate() - diff);
    const from = start.toISOString().slice(0,10);
    const to = nowISODate();
    $("#hisFrom").value = from;
    $("#hisTo").value = to;
    applyHistory();
  };

  // students
  $("#studentSearch").addEventListener("input", refreshStudents);

  // config
  $("#btnSaveCfg").onclick = saveConfig;

  // users
  $("#btnAddUser").onclick = addUser;

  // reset demo
  $("#btnResetDemo").onclick = () => {
    const ok = confirm("Isso vai APAGAR TUDO e recriar o usuário admin/1234. Continuar?");
    if (!ok) return;
    localStorage.removeItem(DB_KEY);
    localStorage.removeItem(SESSION_KEY);
    seedDB();
    boot();
    toast("Demo resetada.");
  };
}

/* -------------------- Main Refresh -------------------- */
function refreshAll() {
  const db = getDB();
  const session = getSession();
  const date = nowISODate();

  refreshHeader(db, session);
  refreshKPIs(db, date);
  renderDayTables(db, date);

  // default history dates
  if (!$("#hisFrom").value || !$("#hisTo").value) {
    const y = new Date().getFullYear();
    $("#hisFrom").value = `${y}-01-01`;
    $("#hisTo").value = nowISODate();
  }

  loadConfigUI(db);
  renderUsersTable();

  // auto receiver
  $("#inReceiver").value = session?.username || "";
}

/* -------------------- Boot -------------------- */
function boot() {
  const db = getDB();
  const session = getSession();

  const isLogged = !!session;

  show($("#viewLogin"), !isLogged);
  show($("#viewApp"), isLogged);

  refreshHeader(db, session);

  if (isLogged) {
    // set default tab
    setTab("tabMov");
    refreshAll();
  }
}

document.addEventListener("DOMContentLoaded", () => {
  seedDB(); // se já existir, não sobrescreve (seedDB só salva quando não existe via getDB, mas aqui é safe? Ajuste:)
  // Ajuste: não queremos sobrescrever. Então:
  if (!loadDB()) seedDB();

  bindUI();
  boot();
});
