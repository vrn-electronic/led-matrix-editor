
<!doctype html>

<html lang="km">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>LED MATRIX EDITOR BY VRN ELECTRONIC</title>
<style>
:root{--bg:#0f1720;--card:#0b1220;--accent:#ff5b7f;--muted:#98a0aa;--cell:18px}
*{box-sizing:border-box}
body{margin:0;font-family:Inter, system-ui, 'Noto Sans Khmer', Arial;background:var(--bg);color:#e6eef6}
.header{display:flex;align-items:center;gap:12px;padding:12px 16px;background:linear-gradient(90deg,#071021, #0b1b2b);box-shadow:0 4px 12px rgba(0,0,0,0.6)}
.brand{font-weight:700;font-size:16px}
.sub{font-size:12px;color:var(--muted)}
.container{padding:14px;max-width:1100px;margin:0 auto}
.topbar{display:flex;gap:8px;flex-wrap:wrap;align-items:center}
.controls{display:flex;gap:8px;flex-wrap:wrap;margin-top:12px}
button,select,input{background:linear-gradient(#0f2635,#081422);border:1px solid rgba(255,255,255,0.03);color:#e8f0f8;padding:8px 10px;border-radius:8px}
#canvas{display:grid;grid-template-columns:repeat(32,var(--cell));grid-template-rows:repeat(8,var(--cell));gap:4px;margin-top:14px;touch-action:none}
.cell{width:var(--cell);height:var(--cell);background:#09111a;border-radius:6px;border:1px solid #07111a}
.cell.on{background:var(--accent);box-shadow:0 0 8px rgba(255,91,127,0.6), inset 0 -2px 6px rgba(0,0,0,0.25)}
.panel{display:flex;gap:12px;margin-top:14px}
.left{flex:1}
.right{width:360px}
.frames{display:flex;gap:8px;flex-wrap:wrap;margin-top:10px}
.thumb{width:120px;height:36px;background:#07111a;border-radius:8px;padding:6px;display:grid;grid-template-columns:repeat(16,6px);grid-auto-rows:6px;gap:2px}
.thumb .dot{width:6px;height:6px;background:#031117;border-radius:3px}
.thumb .dot.on{background:var(--accent)}
textarea{width:100%;height:220px;background:#021021;color:#cfe9ff;border-radius:8px;border:1px solid #06202b;padding:10px}
.small{font-size:13px;color:var(--muted)}
.footer{margin-top:12px;color:var(--muted);font-size:13px}
@media (max-width:760px){:root{--cell:12px}.right{width:100%}.panel{flex-direction:column}.thumb{width:80px;height:26px}}
</style>
</head>
<body>
<div class="header">
  <div style="width:56px;height:56px;border-radius:8px;background:linear-gradient(180deg,#11202b,#07121a);display:flex;align-items:center;justify-content:center;font-weight:800;color:var(--accent)">VRN</div>
  <div>
    <div class="brand">LED MATRIX EDITOR <span style="color:var(--accent)">BY VRN ELECTRONIC</span></div>
    <div class="sub">32 × 8 — Create animation for MAX7219 (D1 Mini / ESP8266)</div>
  </div>
</div>
<div class="container">
  <div class="topbar">
    <div class="controls">
      <button id="draw">Draw Mode</button>
      <button id="erase">Erase Mode</button>
      <button id="newFrame">New Frame</button>
      <button id="dupFrame">Duplicate Frame</button>
      <button id="delFrame">Delete Frame</button>
      <button id="play">Play</button>
      <label class="small">Delay(ms) <input id="fps" type="number" value="150" style="width:90px"></label>
      <button id="save">Save / Export C code</button>
      <button id="clear">Clear Frame</button>
    </div>
  </div>  <div class="panel">
    <div class="left">
      <div id="canvas" aria-label="matrix"></div>
      <div class="frames" id="frames"></div>
    </div>
    <div class="right">
      <div style="background:var(--card);padding:10px;border-radius:10px">
        <div style="font-weight:700;margin-bottom:6px">Preview & Controls</div>
        <div class="small">Use touch/click to draw. Tap Play to preview animation. Frames shown below.</div>
        <hr style="border:none;border-top:1px solid rgba(255,255,255,0.03);margin:8px 0">
        <div style="margin-bottom:8px"><strong>Frames:</strong> <span id="frameCount">0</span></div>
        <div style="display:flex;gap:6px;flex-wrap:wrap"><button id="leftBtn">←</button><button id="upBtn">↑</button><button id="downBtn">↓</button><button id="rightBtn">→</button></div>
      </div><div style="height:12px"></div>


<div style="background:var(--card);padding:10px;border-radius:10px">
    <div style="font-weight:700;margin-bottom:6px">Export (Arduino)</div>
    <div class="small">C arrays ready for paste into Arduino IDE (D1 Mini / ESP8266). Columns = 32, bits = rows (bit0 = top).</div>
    <textarea id="code" readonly></textarea>
    <div style="display:flex;gap:8px;margin-top:8px"><button id="copy">Copy</button><button id="download">Download .html</button></div>
  </div>

  <div class="footer">Made with ❤ by VRN Electronic — Open on phone or PC. Share the exported C array to use in MD_MAX72xx projects.</div>
</div>

  </div>
</div><script>
const W=32, H=8; let frames=[]; let cur=0; let mode='draw'; let playing=false; let playInterval=null;
const canvasEl=document.getElementById('canvas'); const framesEl=document.getElementById('frames'); const fpsEl=document.getElementById('fps'); const codeEl=document.getElementById('code');

// build canvas cells
for(let y=0;y<H;y++){ for(let x=0;x<W;x++){ const c=document.createElement('div'); c.className='cell'; c.dataset.x=x; c.dataset.y=y; c.addEventListener('pointerdown',paint); c.addEventListener('pointerenter',paintOver); canvasEl.appendChild(c); }}
function setMode(m){ mode=m; document.getElementById('draw').style.opacity = m==='draw'?1:0.6; document.getElementById('erase').style.opacity = m==='erase'?1:0.6; }
function paint(e){ const el=e.currentTarget; if(mode==='draw') el.classList.add('on'); else el.classList.remove('on'); saveToFrame(); updateThumb(cur); }
function paintOver(e){ if(e.buttons===1) paint(e); }

function newFrame(copy=null){ const g=Array.from({length:H}, ()=>Array(W).fill(0)); if(copy){ for(let y=0;y<H;y++) for(let x=0;x<W;x++) g[y][x]=copy[y][x]; } frames.push(g); cur=frames.length-1; render(); renderList(); }
function dupFrame(){ newFrame(JSON.parse(JSON.stringify(frames[cur]))); }
function delFrame(){ if(frames.length<=1) return; frames.splice(cur,1); cur=Math.max(0,cur-1); render(); renderList(); }
function clearFrame(){ for(let y=0;y<H;y++) for(let x=0;x<W;x++) frames[cur][y][x]=0; render(); updateThumb(cur); }
function saveToFrame(){ const cells=document.querySelectorAll('.cell'); cells.forEach(c=>{ const x=+c.dataset.x, y=+c.dataset.y; frames[cur][y][x] = c.classList.contains('on')?1:0; }); }
function render(){ const cells=document.querySelectorAll('.cell'); const g=frames[cur]; cells.forEach(c=>{ const x=+c.dataset.x, y=+c.dataset.y; c.classList.toggle('on', !!g[y][x]); }); document.getElementById('frameCount').textContent = frames.length; highlightThumb(); }

function renderList(){ framesEl.innerHTML=''; frames.forEach((f,i)=>{ const t=document.createElement('div'); t.className='thumb'; t.dataset.i=i; for(let y=0;y<H;y++){ for(let x=0;x<16;x++){ const d=document.createElement('div'); d.className='dot'; const on = f[y][x] ? true : false; if(on) d.classList.add('on'); t.appendChild(d); }} t.addEventListener('click',()=>{ cur=i; render(); }); framesEl.appendChild(t); }); }
function updateThumb(i){ const thumb = framesEl.children[i]; if(!thumb) return; thumb.innerHTML=''; const f=frames[i]; for(let y=0;y<H;y++){ for(let x=0;x<16;x++){ const d=document.createElement('div'); d.className='dot'; if(f[y][x]) d.classList.add('on'); thumb.appendChild(d); }} }
function highlightThumb(){ [...framesEl.children].forEach((t,idx)=> t.style.outline = idx===cur? '2px solid rgba(255,91,127,0.6)':'none'); }

function frameToCols(frame){ const out=[]; for(let x=0;x<W;x++){ let b=0; for(let y=0;y<H;y++){ if(frame[y][x]) b |= (1<<y); } out.push(b); } return out; }

function exportCode(){ let s='// LED MATRIX FRAMES (32x8) - VRN Electronic
#include <avr/pgmspace.h>

#define FRAME_COUNT '+frames.length+'
#define FRAME_WIDTH '+W+'

'; frames.forEach((f,i)=>{ const cols = frameToCols(f); s += 'const uint8_t frame_'+i+'['+W+'] PROGMEM = {
  '+ cols.map(n=>'0x'+n.toString(16).padStart(2,'0')).join(', ') +'
};

'; }); s += 'const uint8_t * const frames[FRAME_COUNT] PROGMEM = {
  '+ frames.map((_,i)=>'frame_'+i).join(',
  ') +'
};

// Use pgm_read_byte(&frame_x[col]) to read each column
'; codeEl.value = s; }

function copyCode(){ codeEl.select(); document.execCommand('copy'); alert('Copied to clipboard'); }

function downloadHTML(){ const blob = new Blob([document.documentElement.outerHTML], {type:'text/html'}); const url=URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download='vrn-led32x8.html'; a.click(); URL.revokeObjectURL(url); }

function playToggle(){ playing = !playing; const btn=document.getElementById('play'); btn.textContent = playing? 'Stop':'Play'; if(playing) startPlay(); else stopPlay(); }
function startPlay(){ let i=0; const delay = Math.max(20, parseInt(fpsEl.value||150)); playInterval = setInterval(()=>{ cur = (cur+1)%frames.length; render(); }, delay); }
function stopPlay(){ clearInterval(playInterval); playInterval=null; }

// arrow moves
function shift(dx,dy){ const g=frames[cur]; const n=Array.from({length:H}, ()=>Array(W).fill(0)); for(let y=0;y<H;y++) for(let x=0;x<W;x++){ const nx=x-dx, ny=y-dy; if(nx>=0 && nx<W && ny>=0 && ny<H) n[y][x]=g[ny][nx]; } frames[cur]=n; render(); renderList(); }

// events
setMode('draw'); document.getElementById('draw').onclick=()=>setMode('draw'); document.getElementById('erase').onclick=()=>setMode('erase'); document.getElementById('newFrame').onclick=()=>newFrame(); document.getElementById('dupFrame').onclick=()=>dupFrame(); document.getElementById('delFrame').onclick=()=>delFrame(); document.getElementById('clear').onclick=()=>clearFrame(); document.getElementById('play').onclick=()=>playToggle(); document.getElementById('save').onclick=()=>exportCode(); document.getElementById('copy').onclick=copyCode; document.getElementById('download').onclick=downloadHTML; document.getElementById('leftBtn').onclick=()=>shift(1,0); document.getElementById('upBtn').onclick=()=>shift(0,1); document.getElementById('downBtn').onclick=()=>shift(0,-1); document.getElementById('rightBtn').onclick=()=>shift(-1,0);

// init with a few demo frames
(function init(){ const heart1 = [
  '00000000000000000000000000000000',
  '00000001100110000000000000000000',
  '00000011111111100000000000000000',
  '00000111111111110000000000000000',
  '00000111111111110000000000000000',
  '00000011111111100000000000000000',
  '00000001111111000000000000000000',
  '00000000111100000000000000000000'
]; const heart2 = [
  '00000000000000000000000000000000',
  '00000001011001000000000000000000',
  '00000011111111100000000000000000',
  '00000111111111110000000000000000',
  '00000111111111110000000000000000',
  '00000011111111100000000000000000',
  '00000001111111000000000000000000',
  '00000000111100000000000000000000'
];
 function strToGrid(arr){ const g=Array.from({length:H}, ()=>Array(W).fill(0)); for(let y=0;y<H;y++){ for(let x=0;x<W;x++){ g[y][x] = arr[y][x]==='1'?1:0; }} return g; }
 newFrame(strToGrid(heart1)); newFrame(strToGrid(heart2)); renderList(); render(); exportCode(); })();
</script></body>
</html>
