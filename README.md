<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>듣고 누르는 사람 확인 실험</title>
<style>
:root{--paper:#F2F4F1;--ink:#23303B;--grid:#C9D3D6;--blue:#2B59C3;--red:#C7362F;--muted:#5E6B73}
*{box-sizing:border-box}
html{background:var(--paper);color:var(--ink)}
body{margin:0;font-family:"Apple SD Gothic Neo","Malgun Gothic",system-ui,sans-serif;font-size:18px;line-height:1.65}
main{max-width:680px;margin:0 auto;padding:44px 22px 80px}
h1{font-size:28px;line-height:1.3;margin:0 0 6px}
h2{font-size:21px;margin:32px 0 8px}
p{margin:0 0 12px}
.sub,.note{color:var(--muted)}
.note{font-size:15px}
fieldset{border:0;padding:0;margin:0 0 18px}
legend{font-weight:700;margin-bottom:6px;padding:0}
label.opt{display:flex;gap:10px;align-items:flex-start;padding:4px 0;cursor:pointer}
input[type=radio],input[type=checkbox]{width:20px;height:20px;margin-top:4px;accent-color:var(--blue)}
input[type=text]{font:inherit;font-size:20px;padding:6px 10px;border:1.5px solid var(--ink);border-radius:3px;width:9ch}
button{font:inherit;font-size:17px;padding:10px 20px;border:1.5px solid var(--ink);background:var(--ink);color:var(--paper);border-radius:3px;cursor:pointer}
button.quiet{background:transparent;color:var(--ink)}
:focus-visible{outline:3px solid var(--blue);outline-offset:3px}
.row{display:flex;gap:10px;flex-wrap:wrap;align-items:center;margin-top:14px}
.err{color:var(--red);font-weight:600}
table{border-collapse:collapse;width:100%;font-size:15px;margin:8px 0 18px}
td,th{border-bottom:1px solid var(--grid);padding:6px 4px;text-align:left;vertical-align:top}
th{font-weight:600;color:var(--muted)}
body.dark{background:#000;min-height:100vh}
html:has(body.dark){background:#000}
body.dark main{display:none}
.sr{position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0 0 0 0)}
</style>
</head>
<body>
<main id="app"></main>
<p id="say" class="sr" role="status" aria-live="assertive"></p>
<script>
/* =====================================================================
   듣고 누르는 사람 확인 실험 (실험 4, 재설계판 v5-protocol)
   대전대신고 ACT(KE)_5

   설계 근거
   - 리듬 문제 생성·판정은 AI 백테스트(lab/rhythm/rc/core.py)와 같은 규칙이다.
     생성 규칙을 바꾸면 AI 공격 결과를 사람 결과와 비교할 수 없게 되므로 바꾸지 않는다.
   - 섭동 없는 '기준선' 문항 1개를 추가했다. 개인의 평균 비동기와 흔들림(SD)을 재서
     탈락이 사람 때문인지 장비 때문인지 가린다 (Repp & Su 2013; Anglada-Tort 외 2022).
   - 판정 기준은 '사전 고정 v4'를 그대로 두고, 고친 기준은 RULESETS에 '추가'만 한다.
     결과는 모든 기준을 나란히 보고한다 (지도교사 피드백 7).
   - 통과율은 문항 기준(윌슨 95% 신뢰구간)과 참가자 기준을 따로 낸다 (피드백 5).
   - 숫자 듣기(비교 기준)는 브라우저 음성 + 분홍 잡음으로 만든다. AI 백테스트의 espeak 문항과
     음성이 달라 사람 쪽 '기존 음성 CAPTCHA 대표값'으로만 쓴다 (한계로 보고).
   ===================================================================== */
(() => {
const RULESETS = {
  "v4_사전고정": {alpha: [0.1, 1.2], beta: [-1.6, -0.4], noise: [5, 45], residual: [0, 55]}
  // 파일럿 분석으로 기준을 고칠 경우 여기에 새 이름으로 '추가'한다. v4는 지우지 않는다.
};
const PRIMARY = "v4_사전고정";
const N = {rhythmReal: 3, digitsReal: 2};
const NAME = {rhythm: "박자 따라 치기", digits: "숫자 듣기"};
const $app = document.getElementById("app"), $say = document.getElementById("say");
let P = null, ctx = null, aborted = false, koVoice = null;
const ADMIN = new URLSearchParams(location.search).has("admin");  // 진행자 전용 화면: 주소 끝에 ?admin

/* ---------- 저장 ---------- */
const store = {get(k){try{return JSON.parse(localStorage.getItem(k));}catch(e){return null;}},
               set(k,v){try{localStorage.setItem(k,JSON.stringify(v));}catch(e){}}};
const allRecs = () => store.get("act5_recs") || [];
const addRec = r => { const a = allRecs().filter(x => x.pid !== r.pid); a.push(r); store.set("act5_recs", a); };
const setup = () => store.get("act5_setup") || {};

/* ---------- 도구 ---------- */
const sleep = ms => new Promise(r => setTimeout(r, ms));
const render = h => { $app.innerHTML = h; const f = $app.querySelector("[autofocus],button,input"); if (f) f.focus(); };
const pick = a => a[Math.floor(Math.random() * a.length)];
const ri = (a, b) => a + Math.floor(Math.random() * (b - a + 1));
function pickVoice(){ try { koVoice = speechSynthesis.getVoices().find(v => /^ko/i.test(v.lang)) || null; } catch(e){} }
try { pickVoice(); speechSynthesis.onvoiceschanged = pickVoice; } catch(e){}
function speak(text, rate = 1.05){
  $say.textContent = text;
  return new Promise(res => {
    try {
      speechSynthesis.cancel();
      const u = new SpeechSynthesisUtterance(text); u.lang = "ko-KR"; if (koVoice) u.voice = koVoice; u.rate = rate;
      const to = setTimeout(res, 1500 + text.length * 120); u.onend = () => { clearTimeout(to); res(); }; u.onerror = u.onend;
      speechSynthesis.speak(u);
    } catch(e){ setTimeout(res, 300); }
  });
}
function waitKey(test){
  return new Promise(res => { const h = ev => { if (ev.key === "Escape") aborted = true;
    if (aborted || test(ev)){ ev.preventDefault(); document.removeEventListener("keydown", h, true); res(ev); } };
    document.addEventListener("keydown", h, true); });
}
function collector(onKey){ const h = ev => { if (ev.key === "Escape") aborted = true; onKey(ev); };
  document.addEventListener("keydown", h, true); return () => document.removeEventListener("keydown", h, true); }
async function audio(){ if (!ctx) ctx = new (window.AudioContext || window.webkitAudioContext)(); if (ctx.state !== "running") await ctx.resume(); return ctx; }
function atime(ev){  // 키 입력 시각 → 오디오 시계(초). 출력 지연 보정
  try { const ts = ctx.getOutputTimestamp && ctx.getOutputTimestamp();
    if (ts && ts.performanceTime) return ts.contextTime + (ev.timeStamp - ts.performanceTime) / 1000; } catch(e){}
  return ctx.currentTime - (ctx.outputLatency || ctx.baseLatency || 0);
}
function beep(t, f, dur = 0.06, gain = 0.7){
  const o = ctx.createOscillator(), g = ctx.createGain(); o.frequency.value = f; o.connect(g); g.connect(ctx.destination);
  g.gain.setValueAtTime(0.0001, t); g.gain.exponentialRampToValueAtTime(gain, t + 0.003);
  g.gain.exponentialRampToValueAtTime(0.0001, t + dur); o.start(t); o.stop(t + dur + 0.02);
}
function pinkNoise(sec, level){
  const n = Math.floor(ctx.sampleRate * sec), b = ctx.createBuffer(1, n, ctx.sampleRate), d = b.getChannelData(0);
  let b0=0,b1=0,b2=0,b3=0,b4=0,b5=0,b6=0;
  for (let i = 0; i < n; i++){ const w = Math.random()*2-1;
    b0=.99886*b0+w*.0555179; b1=.99332*b1+w*.0750759; b2=.969*b2+w*.153852; b3=.8665*b3+w*.3104856;
    b4=.55*b4+w*.5329522; b5=-.7616*b5-w*.016898; d[i]=(b0+b1+b2+b3+b4+b5+b6+w*.5362)*.11*level; b6=w*.115926; }
  const s = ctx.createBufferSource(); s.buffer = b; s.connect(ctx.destination); return s;
}

/* ---------- 리듬: 문제 생성 (core.py make_challenge 와 동일) ---------- */
function makeChallenge(baseline){
  const ioi = pick([500, 550, 600, 650, 700]);
  const ks = [ri(3, 5), ri(8, 10), ri(13, 14)];
  const ds = baseline ? [0, 0, 0] : ks.map(() => pick([-1, 1]) * pick([70, 80, 90]));
  const beats = [];
  for (let i = 0; i < 20; i++){ let b = (4 + i) * ioi; ks.forEach((k, j) => { if (i >= k) b += ds[j]; }); beats.push(b); }
  return {ioi, ks, ds, beats};
}
/* ---------- 리듬: 특징 추출 (core.py features 와 동일) ---------- */
const nanmean = a => { const z = a.filter(x => !Number.isNaN(x)); return z.length ? z.reduce((p, q) => p + q, 0) / z.length : NaN; };
function features(c, taps){
  const B = c.beats, ioi = c.ioi, T = taps.slice().sort((a, b) => a - b), used = new Set();
  const A = B.map(b => { if (!T.length) return NaN; let j = 0, d = Infinity;
    T.forEach((t, i) => { if (Math.abs(t - b) < d){ d = Math.abs(t - b); j = i; } });
    if (!used.has(j) && d <= ioi / 2){ used.add(j); return T[j] - b; } return NaN; });
  const missed = A.filter(Number.isNaN).length, f = {missed, valid: false};
  if (missed > 3) return f;
  const mA = nanmean(A), X = [], y = [];
  for (let i = 0; i < A.length - 1; i++){ const dA = A[i + 1] - A[i];
    if (!Number.isNaN(dA) && !Number.isNaN(A[i])){ X.push([A[i] - mA, B[i + 1] - B[i] - ioi, 1]); y.push(dA); } }
  if (X.length < 12) return f;
  const M = [[0,0,0],[0,0,0],[0,0,0]], v = [0,0,0];
  X.forEach((r, n) => { for (let i = 0; i < 3; i++){ v[i] += r[i] * y[n]; for (let j = 0; j < 3; j++) M[i][j] += r[i] * r[j]; } });
  const det = m => m[0][0]*(m[1][1]*m[2][2]-m[1][2]*m[2][1]) - m[0][1]*(m[1][0]*m[2][2]-m[1][2]*m[2][0]) + m[0][2]*(m[1][0]*m[2][1]-m[1][1]*m[2][0]);
  let D = det(M);
  if (Math.abs(D) < 1e-9){ for (let i = 0; i < 3; i++) M[i][i] += 1e-6; D = det(M); }  // 오차가 전부 0인 봇: 파이썬 lstsq처럼 계산을 이어감
  const co = [0,1,2].map(k => det(M.map((row, i) => row.map((x, j) => j === k ? v[i] : x))) / D);
  const res = X.map((r, n) => y[n] - (r[0]*co[0] + r[1]*co[1] + r[2]*co[2]));
  const rm = res.reduce((a, b) => a + b, 0) / res.length;
  return {missed, valid: true, mean_async: mA,
    sd_async: Math.sqrt(nanmean(A.map(a => (a - mA) ** 2))),
    alpha: -co[0], beta: co[1], noise: Math.sqrt(res.reduce((a, b) => a + (b - rm) ** 2, 0) / res.length),
    residual: Math.abs(nanmean(A.slice(-3)) - nanmean(A.slice(0, c.ks[0])))};
}
function verdict(f, rules){
  if (!f.valid) return {pass: false, fails: ["탭 누락"]};
  const fails = Object.entries(rules).filter(([k, [lo, hi]]) => !(f[k] >= lo && f[k] <= hi)).map(([k]) => k);
  return {pass: fails.length === 0, fails};
}

/* ---------- 과제 ---------- */
async function rhythmTrial(baseline){
  const c = makeChallenge(baseline), t0 = ctx.currentTime + 0.6, taps = [];
  for (let i = 0; i < 4; i++) beep(t0 + i * c.ioi / 1000, 1500);
  c.beats.forEach(b => beep(t0 + b / 1000, 880));
  const stop = collector(e => { if (e.code === "Space"){ e.preventDefault(); if (!e.repeat) taps.push(+((atime(e) - t0) * 1000).toFixed(1)); } });
  const dur = 600 + c.beats[19] + c.ioi * 1.5; await sleep(dur); stop();
  const f = features(c, taps), v = {};
  Object.entries(RULESETS).forEach(([name, r]) => { v[name] = verdict(f, r); });
  return {challenge: c, taps, features: f, verdicts: v, pass: v[PRIMARY].pass, rt_s: dur / 1000};
}
const KO = ["공", "일", "이", "삼", "사", "오", "육", "칠", "팔", "구"];
async function playDigits(code){
  const noise = pinkNoise(14, 0.6); noise.start();
  await sleep(700);
  for (const c of code){ if (aborted) break; await speak(KO[+c], 0.9 + Math.random() * 0.3); await sleep(350 + Math.random() * 450); }
  try { noise.stop(); } catch(e){}
}
async function digitsTrial(){
  const code = Array.from({length: 5}, () => ri(0, 9)).join("");
  let typed = "", replays = 0, playing = null; const t0 = performance.now();
  const play = () => { playing = playDigits(code); };
  play();
  await new Promise(res => { const stop = collector(e => {
    if (/^[0-9]$/.test(e.key)){ typed += e.key; e.preventDefault(); }
    else if (e.key === "Backspace"){ typed = typed.slice(0, -1); e.preventDefault(); }
    else if ((e.key === "r" || e.key === "R" || e.key === "ㄱ")){ e.preventDefault(); replays++; try{speechSynthesis.cancel();}catch(x){} play(); }
    else if (e.key === "Enter" && typed){ e.preventDefault(); stop(); res(); }
    if (aborted){ stop(); res(); } }); });
  try { speechSynthesis.cancel(); } catch(e){}
  return {code, response: typed, pass: typed === code, replays, rt_s: (performance.now() - t0) / 1000};
}

/* ---------- 진행 ---------- */
async function runBlock(kind){
  const intro = kind === "rhythm"
    ? "박자 따라 치기. 높은 소리 네 번 뒤, 낮은 소리에 맞춰 스페이스바를 누르세요. 박자가 바뀌어도 계속 따라 치세요."
    : "숫자 듣기. 잡음 속 숫자 다섯 개를 듣고 입력한 뒤 엔터. 다시 듣기는 R.";
  await speak(intro);
  const plan = kind === "rhythm"
    ? [["practice", "연습"], ["baseline", "다음"], ...Array.from({length: N.rhythmReal}, (_, i) => ["real", `실전 ${i + 1}번`])]
    : [["practice", "연습"], ...Array.from({length: N.digitsReal}, (_, i) => ["real", `실전 ${i + 1}번`])];
  for (const [phase, label] of plan){
    if (aborted) return;
    await speak(`${label}. 스페이스바로 시작.`);
    await waitKey(e => e.code === "Space"); if (aborted) return;
    await sleep(250);
    const r = kind === "rhythm" ? await rhythmTrial(phase === "baseline") : await digitsTrial();
    if (aborted) return;
    P.trials.push({kind, phase, ...r, at: Date.now()}); store.set("act5_current", P);
  }
  await speak("쉬운 정도를 1부터 7 중에 누르세요. 7이 가장 쉬움.");
  const ev = await waitKey(e => /^[1-7]$/.test(e.key)); if (!aborted) P.seq[kind] = +ev.key;
}

/* ---------- 통계 ---------- */
function wilson(k, n){ if (!n) return [NaN, NaN]; const z = 1.96, p = k / n, d = 1 + z*z/n,
  c = (p + z*z/(2*n)) / d, h = z * Math.sqrt(p*(1-p)/n + z*z/(4*n*n)) / d; return [Math.max(0, c - h), Math.min(1, c + h)]; }
const pct = x => Number.isNaN(x) ? "-" : (x * 100).toFixed(0) + "%";
function summary(){
  const recs = allRecs().filter(r => r.cond === "blindfold");
  const rows = [];
  const block = (kind, passOf) => {
    const per = recs.map(r => r.trials.filter(t => t.kind === kind && t.phase === "real")).filter(ts => ts.length);
    const all = per.flat(), k = all.filter(passOf).length, [lo, hi] = wilson(k, all.length);
    const pm = per.length ? per.reduce((s, ts) => s + ts.filter(passOf).length / ts.length, 0) / per.length : NaN;
    const maj = per.filter(ts => ts.filter(passOf).length * 2 > ts.length).length;
    return {n: all.length, people: per.length, rate: all.length ? k / all.length : NaN, lo, hi, pm, maj};
  };
  Object.keys(RULESETS).forEach(name => rows.push([`${NAME.rhythm} (${name})`, block("rhythm", t => t.verdicts && t.verdicts[name] && t.verdicts[name].pass)]));
  rows.push([NAME.digits, block("digits", t => t.pass)]);
  return {people: recs.length, rows};
}
function diagnosis(){  // 탈락 원인: 실전 리듬 문항에서 기준별로 벗어난 비율
  const ts = allRecs().filter(r => r.cond === "blindfold").flatMap(r => r.trials.filter(t => t.kind === "rhythm" && t.phase === "real"));
  if (!ts.length) return "";
  const r = RULESETS[PRIMARY], keys = ["탭 누락", ...Object.keys(r)];
  const cnt = k => ts.filter(t => t.verdicts[PRIMARY].fails.includes(k)).length;
  const med = k => { const v = ts.map(t => t.features[k]).filter(x => typeof x === "number").sort((a, b) => a - b); return v.length ? v[Math.floor(v.length / 2)].toFixed(2) : "-"; };
  const base = allRecs().filter(r => r.cond === "blindfold").flatMap(r => r.trials.filter(t => t.kind === "rhythm" && t.phase === "baseline" && t.features.valid));
  const bm = base.length ? nanmean(base.map(t => t.features.mean_async)).toFixed(0) : "-", bs = base.length ? nanmean(base.map(t => t.features.sd_async)).toFixed(0) : "-";
  return `<h2>리듬 탈락 원인 (${PRIMARY}, 실전 ${ts.length}문항)</h2><table><tr><th>기준</th><th>허용 범위</th><th>벗어난 문항</th><th>중앙값</th></tr>${
    keys.map(k => `<tr><td>${k}</td><td>${r[k] ? r[k].join(" ~ ") : "누락 3개 이하"}</td><td>${cnt(k)}</td><td>${r[k] ? med(k) : "-"}</td></tr>`).join("")}</table>
    <p class="note">기준선(섭동 없음) 평균 비동기 ${bm} ms, 비동기 표준편차 ${bs} ms. 보통 사람은 평균 비동기가 음수(소리보다 먼저 누름)다. 크게 양수면 장비 지연을 의심한다.</p>`;
}
function statsHtml(){
  if (!ADMIN) return "";
  const s = summary();
  return `<h2>누적 결과 (본실험, ${s.people}명)</h2>
  <table><tr><th>과제</th><th>실전 문항</th><th>문항 통과율 (95% CI)</th><th>참가자 평균 통과율</th><th>과반 통과 참가자</th></tr>
  ${s.rows.map(([name, b]) => `<tr><td>${name}</td><td>${b.n}</td><td>${pct(b.rate)} (${pct(b.lo)}~${pct(b.hi)})</td><td>${pct(b.pm)}</td><td>${b.maj}/${b.people}</td></tr>`).join("")}</table>
  ${diagnosis()}
  <div class="row"><button class="quiet" id="json">전체 기록(JSON) 내보내기</button><button class="quiet" id="csv">CSV 내보내기</button><button class="quiet" id="wipe">기록 모두 지우기</button></div>`;
}
function download(name, text, type){ const a = document.createElement("a"); a.href = URL.createObjectURL(new Blob([text], {type})); a.download = name; a.click(); }
function wire(){
  const j = document.getElementById("json"), c = document.getElementById("csv"), w = document.getElementById("wipe");
  if (j) j.onclick = () => download("act5_results.json", JSON.stringify(allRecs()), "application/json");
  if (c) c.onclick = () => {
    const cols = ["pid","cond","laptop","device","order","kind","phase","pass_primary","fails_primary","alpha","beta","noise","residual","mean_async","sd_async","missed","code","response","replays","rt_s","seq"];
    const rows = [cols.join(",")];
    allRecs().forEach(r => r.trials.forEach(t => { const f = t.features || {};
      rows.push([r.pid, r.cond, r.setup.laptop, r.setup.device, r.order.join("-"), t.kind, t.phase, t.pass,
        t.verdicts ? t.verdicts[PRIMARY].fails.join("|") : "", f.alpha, f.beta, f.noise, f.residual, f.mean_async, f.sd_async, f.missed,
        t.code || "", t.response || "", t.replays || 0, (t.rt_s || 0).toFixed(2), (r.seq || {})[t.kind] || ""].join(",")); }));
    download("act5_results.csv", "﻿" + rows.join("\n"), "text/csv"); };
  if (w) w.onclick = () => { if (confirm("기록을 모두 지웁니다. 먼저 JSON으로 내보냈나요?")){ store.set("act5_recs", []); home(); } };
}

/* ---------- 화면 ---------- */
function home(){
  aborted = false; const s = setup();
  render(`<h1>듣고 누르는 사람 확인 실험</h1>
  <p class="sub">대전대신고 ACT(KE)_5 · 1인 약 3분 · 이름과 연락처는 저장하지 않습니다</p>
  <h2>진행자 설정 (실험 시작 전 한 번)</h2>
  <p class="note">모든 참가자를 같은 장비로 진행합니다. 장비 차이가 결과에 섞이지 않게 하기 위해서입니다.</p>
  <fieldset><legend>장비 고정 확인</legend>
    <label class="opt"><input type="checkbox" class="chk"> 같은 노트북, 같은 유선 이어폰을 끝까지 씁니다</label>
    <label class="opt"><input type="checkbox" class="chk"> 음량을 정하고 끝까지 바꾸지 않습니다</label>
    <label class="opt"><input type="checkbox" class="chk"> 알림 소리와 다른 탭 소리를 껐고, 충전기를 꽂았습니다</label></fieldset>
  <fieldset><legend>노트북 이름</legend><input type="text" id="laptop" value="${s.laptop || "A"}" aria-label="노트북 이름"></fieldset>
  <fieldset><legend>소리 장치</legend>
    <label class="opt"><input type="radio" name="dev" value="wired" ${s.device !== "bt" ? "checked" : ""}> 유선 이어폰·헤드폰 (권장)</label>
    <label class="opt"><input type="radio" name="dev" value="bt" ${s.device === "bt" ? "checked" : ""}> 블루투스 (지연 때문에 권장하지 않음)</label></fieldset>
  <h2>참가자</h2>
  <fieldset><legend>동의</legend><label class="opt"><input type="checkbox" id="consent"> 참가자가 동의했습니다 (미성년자는 보호자 동의서 제출 완료)</label></fieldset>
  <fieldset><legend>참가 번호</legend><input type="text" id="pnum" inputmode="numeric" maxlength="3" value="${(allRecs().length + 1)}"></fieldset>
  <fieldset><legend>조건</legend>
    <label class="opt"><input type="radio" name="cond" value="blindfold" checked> 본실험 (화면이 꺼진 채 진행)</label>
    <label class="opt"><input type="radio" name="cond" value="practice"> 화면 보며 연습 (통계에서 빠짐)</label></fieldset>
  <div class="row"><button id="go">소리 확인 후 시작</button></div><p id="msg"></p>
  ${ADMIN && allRecs().length ? statsHtml() : ""}`);
  document.getElementById("go").onclick = start; wire();
}
async function start(){
  const msg = t => document.getElementById("msg").innerHTML = `<span class="err">${t}</span>`;
  if ([...document.querySelectorAll(".chk")].some(c => !c.checked)) return msg("장비 고정 항목 3개를 모두 확인해 주세요.");
  if (!document.getElementById("consent").checked) return msg("동의를 확인해 주세요.");
  const n = parseInt(document.getElementById("pnum").value, 10); if (!Number.isFinite(n)) return msg("참가 번호를 숫자로 적어 주세요.");
  const s = {laptop: document.getElementById("laptop").value.trim() || "A", device: document.querySelector("input[name=dev]:checked").value};
  store.set("act5_setup", s);
  await audio();
  P = {v: "v5-protocol", pid: `P${n}`, cond: document.querySelector("input[name=cond]:checked").value, setup: s,
       order: n % 2 ? ["rhythm", "digits"] : ["digits", "rhythm"], trials: [], seq: {},
       env: {outputLatency: ctx.outputLatency || null, baseLatency: ctx.baseLatency || null, sampleRate: ctx.sampleRate,
             voice: koVoice ? koVoice.name : null, ua: navigator.userAgent}, started: new Date().toISOString()};
  render(`<h2>소리 확인</h2><p>버튼을 누르면 소리 3번과 음성이 나옵니다. 편하게 들리는지 참가자에게 확인합니다.</p>
    <div class="row"><button id="t">소리 듣기</button><button class="quiet" id="ok">잘 들림, 다음</button></div>`);
  document.getElementById("t").onclick = async () => { const t = ctx.currentTime + 0.2; [0, .6, 1.2].forEach(d => beep(t + d, 880)); await sleep(1800); speak("소리가 잘 들리면 진행자에게 알려 주세요."); };
  document.getElementById("ok").onclick = ready;
}
async function ready(){
  render(`<h2>준비</h2><p>참가자에게 스페이스바, 엔터, 숫자 줄, R 키 위치를 손으로 익히게 합니다.</p>
    <p>참가자가 스페이스바를 누르면 화면이 꺼지고 음성 안내로 진행됩니다. 진행자는 돕지 않습니다. 중단은 Esc.</p>
    <p class="note">순서: ${P.order.map(k => NAME[k]).join(" → ")} · 리듬은 연습 1, 기준선 1, 실전 ${N.rhythmReal} · 숫자는 연습 1, 실전 ${N.digitsReal}</p>`);
  await waitKey(e => e.code === "Space"); if (aborted) return finish();
  if (P.cond === "blindfold") document.body.classList.add("dark");
  for (const k of P.order){ if (aborted) break; await runBlock(k); }
  finish();
}
function finish(){
  document.body.classList.remove("dark"); try { speechSynthesis.cancel(); } catch(e){}
  if (!P) return home();
  P.ended = new Date().toISOString(); P.aborted = aborted;
  if (P.trials.length) addRec(P); store.set("act5_current", null);
  const mine = P.trials.filter(t => t.phase === "real");
  render(`<h2>${aborted ? "중단했습니다" : "끝났습니다. 고맙습니다"}</h2>
    ${ADMIN ? `<p>이 참가자: 리듬 실전 ${mine.filter(t => t.kind === "rhythm" && t.pass).length}/${mine.filter(t => t.kind === "rhythm").length} 통과,
    숫자 실전 ${mine.filter(t => t.kind === "digits" && t.pass).length}/${mine.filter(t => t.kind === "digits").length} 통과</p>` : "<p>결과가 저장되었습니다.</p>"}
    <div class="row"><button id="next">다음 참가자</button></div>${statsHtml()}`);
  document.getElementById("next").onclick = () => { P = null; home(); }; wire();
  if (!aborted) speak("끝났습니다. 고맙습니다.");
}
home();
})();
</script>
</body>
</html>
