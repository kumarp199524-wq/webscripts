// ---------- DOM helpers ----------
const $ = sel => document.querySelector(sel);
const editor = $('#editor');
const preview = $('#preview');

// ---------- State ----------
const state = {
  ignoreStopwords: true,
  topN: 20,
  minLen: 3,
  readingWPM: 220,
  speakingWPM: 130,
  goal: 1000,
  fontSize: 16,
  highlightShort: true,
  highlightLong: true,
};

// Common English stopwords (lowercase)
const STOPWORDS = new Set(
  "a,about,above,after,again,against,all,am,an,and,any,are,aren't,as,at,be,because,been,before,being,below,between,both,but,by,could,couldn't,did,didn't,do,does,doesn't,doing,don't,down,during,each,few,for,from,further,had,hadn't,has,hasn't,have,haven't,having,he,he'd,he'll,he's,her,here,here's,hers,herself,him,himself,his,how,how's,i,i'd,i'll,i'm,i've,if,in,into,is,isn't,it,it's,its,itself,let's,me,more,most,mustn't,my,myself,no,nor,not,of,off,on,once,only,or,other,ought,our,ours,ourselves,out,over,own,same,shan't,she,she'd,she'll,she's,should,shouldn't,so,some,such,than,that,that's,the,their,theirs,them,themselves,then,there,there's,these,they,they'd,they'll,they're,they've,this,those,through,to,too,under,until,up,very,was,wasn't,we,we'd,we'll,we're,we've,were,weren't,what,what's,when,when's,where,where's,which,while,who,who's,whom,why,why's,with,won't,would,wouldn't,you,you'd,you'll,you're,you've,your,yours,yourself,yourselves"
    .split(',').map(s => s.trim())
);

// ---------- Local storage ----------
function saveLocal() {
  const payload = {
    text: editor.value,
    seoTitle: $('#seoTitle')?.value || '',
    seoMeta: $('#seoMeta')?.value || '',
    seoImage: $('#seoImage')?.value || '',
    serpUrl: $('#serpUrl')?.value || '',
    serpSlug: $('#serpSlug')?.value || '',
    serpDevice: $('#serpDevice')?.value || 'desktop',
    serpShowDate: $('#serpShowDate')?.checked ?? true,
    settings: state
  };
  localStorage.setItem('wc_data_v1', JSON.stringify(payload));
}
function loadLocal() {
  const raw = localStorage.getItem('wc_data_v1');
  if (!raw) return;
  try {
    const d = JSON.parse(raw);
    if (typeof d.text === 'string') editor.value = d.text;
    if (d.settings) Object.assign(state, d.settings);
    ['seoTitle','seoMeta','seoImage','serpUrl','serpSlug','serpDevice'].forEach(id=>{
      if ($('#'+id) && typeof d[id] === 'string') $('#'+id).value = d[id];
    });
    if ($('#serpShowDate') && typeof d.serpShowDate === 'boolean') $('#serpShowDate').checked = d.serpShowDate;
  } catch(e) {}
}

// ---------- Text utilities ----------
function getParagraphs(text){ return text.replace(/\r/g,'').split(/\n{2,}|\n\s*\n/).filter(Boolean); }
function getSentences(text){
  const cleaned = text.replace(/\n/g,' ').replace(/\s+/g,' ').trim();
  if (!cleaned) return [];
  const raw = cleaned.split(/(?<=[.!?])\s+(?=[A-Z0-9"'])/);
  return raw.filter(s => s.trim().length > 0);
}
function getWords(text){ return (text.toLowerCase().match(/[a-zA-ZÀ-ÖØ-öø-ÿ']+/g) || []); }

// ---------- Syllables & Readability ----------
function syllableCount(word){
  word = word.toLowerCase().replace(/[^a-z]/g,'');
  if (!word) return 0;
  if (word.length <= 3) return 1;
  const vowels = 'aeiouy';
  let syl = 0, prevV = false;
  for (let i=0;i<word.length;i++){
    const isV = vowels.includes(word[i]);
    if (isV && !prevV) syl++;
    prevV = isV;
  }
  if (word.endsWith('e')) syl--;
  if (/[^aeiou]le$/.test(word)) syl++;
  return syl < 1 ? 1 : syl;
}
function readabilityMetrics(text){
  const sentences = getSentences(text);
  const words = getWords(text);
  const wordCount = words.length || 1;
  const sentenceCount = Math.max(sentences.length, 1);
  const syllables = words.reduce((a,w)=>a+syllableCount(w),0);

  const asl = wordCount / sentenceCount;
  const asw = syllables / wordCount;

  const fleschEase = Math.max(0, Math.min(100, 206.835 - 1.015*asl - 84.6*asw));
  const fkGrade = 0.39*asl + 11.8*asw - 15.59;
  const complex = words.filter(w=>syllableCount(w)>=3 && !/^(?:mr|mrs|dr|ms)$/i.test(w)).length;
  const pctComplex = (complex / wordCount) * 100;
  const gunningFog = 0.4 * (asl + pctComplex);

  return {fleschEase, fkGrade, gunningFog, asl, asw, syllables, wordCount, sentenceCount};
}

// ---------- SEO Helpers ----------
function seoStatus(val, minGood, maxGood, minWarn, maxWarn){
  if (val>=minGood && val<=maxGood) return {text:'Good', cls:'good'};
  if (val>=minWarn && val<=maxWarn) return {text:'Close', cls:'warn'};
  return {text:'Needs work', cls:'bad'};
}
function ensureMeta(name){
  let el = document.querySelector(`meta[name="${name}"]`);
  if(!el){ el = document.createElement('meta'); el.setAttribute('name', name); document.head.appendChild(el); }
  return el;
}
function ensureOg(property){
  let el = document.querySelector(`meta[property="${property}"]`);
  if(!el){ el = document.createElement('meta'); el.setAttribute('property', property); document.head.appendChild(el); }
  return el;
}
function ensureCanonical(url){
  let link = document.querySelector('link[rel="canonical"]');
  if(!link){ link = document.createElement('link'); link.setAttribute('rel','canonical'); document.head.appendChild(link); }
  link.setAttribute('href', url || location.href);
}
function buildFullUrl(){
  const baseUrl = ($('#serpUrl')?.value || '').trim();
  const slugRaw = ($('#serpSlug')?.value || '').trim();
  try{
    const base = baseUrl || (location.origin || 'https://example.com');
    const slug = slugRaw ? (slugRaw.startsWith('/') ? slugRaw : '/' + slugRaw) : '';
    const u = new URL(base.replace(/\/$/, '') + slug);
    return { full: u.href, domain: u.host, path: slugRaw||'' };
  }catch(e){
    return { full: (baseUrl + (slugRaw||'')) || location.href, domain: (baseUrl.replace(/^https?:\/\//,'')||'example.com'), path: slugRaw||'' };
  }
}
function updateHeadSeo(){
  const title = ($('#seoTitle')?.value || '').trim() || document.title;
  const desc  = ($('#seoMeta')?.value || '').trim() || '';
  const img   = ($('#seoImage')?.value || '').trim() || '';
  const urlObj = buildFullUrl();

  document.title = title;
  ensureMeta('description').setAttribute('content', desc);
  ensureMeta('robots').setAttribute('content', 'index,follow');

  ensureOg('og:title').setAttribute('content', title);
  ensureOg('og:description').setAttribute('content', desc);
  ensureOg('og:type').setAttribute('content', 'website');
  ensureOg('og:url').setAttribute('content', urlObj.full);
  if (img) ensureOg('og:image').setAttribute('content', img);

  ensureMeta('twitter:card').setAttribute('content', img ? 'summary_large_image' : 'summary');
  ensureMeta('twitter:title').setAttribute('content', title);
  ensureMeta('twitter:description').setAttribute('content', desc);
  if (img) ensureMeta('twitter:image').setAttribute('content', img);

  ensureCanonical(urlObj.full);

  // JSON-LD
  const ld = {
    "@context": "https://schema.org",
    "@type": "SoftwareApplication",
    "name": title,
    "description": desc,
    "applicationCategory": "UtilitiesApplication",
    "operatingSystem": "Web",
    "url": urlObj.full
  };
  const script = document.querySelector('#ldjson-app');
  if (script) script.textContent = JSON.stringify(ld, null, 2);
}
function updateSeoCounters(){
  const t = $('#seoTitle')?.value || '';
  const m = $('#seoMeta')?.value || '';
  $('#seoTitleCount').textContent = String(t.length);
  $('#seoMetaCount').textContent  = String(m.length);
  const s1 = seoStatus(t.length, 50, 60, 30, 70);
  const s2 = seoStatus(m.length, 150, 160, 120, 180);
  $('#seoTitleStatus').textContent = s1.text; $('#seoTitleStatus').className = s1.cls;
  $('#seoMetaStatus').textContent  = s2.text; $('#seoMetaStatus').className  = s2.cls;
  updateHeadSeo();
  updateSerpPreview();
}

// ---------- SERP Preview ----------
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');
function measurePx(text, font){ ctx.font = font; return ctx.measureText(text).width; }
function ellipsizeByWidth(text, maxPx, font){
  ctx.font = font;
  if (measurePx(text, font) <= maxPx) return text;
  let lo=0, hi=text.length, ans='';
  while (lo<=hi){
    const mid = Math.floor((lo+hi)/2);
    const slice = text.slice(0, mid) + '…';
    if (measurePx(slice, font) <= maxPx){ ans = slice; lo = mid + 1; } else { hi = mid - 1; }
  }
  return ans || text.slice(0,1) + '…';
}
function wrapToLines(text, maxPx, font, maxLines){
  ctx.font = font;
  const words = text.split(/\s+/);
  const lines = [];
  let cur = '';
  for (const w of words){
    const test = cur ? cur + ' ' + w : w;
    if (measurePx(test, font) <= maxPx){ cur = test; }
    else { lines.push(cur); cur = w; if (lines.length === maxLines-1) break; }
  }
  if (cur) lines.push(cur);
  if (lines.length > maxLines) lines.length = maxLines;
  if (lines.length === maxLines && words.join(' ') !== lines.join(' ')){
    lines[maxLines-1] = ellipsizeByWidth(lines[maxLines-1], maxPx, font);
  }
  return lines;
}
function updateSerpPreview(){
  const title = ($('#seoTitle')?.value || '').trim() || 'Your title will appear here';
  const meta  = ($('#seoMeta')?.value  || '').trim() || 'Your meta description preview will appear here';
  const device = ($('#serpDevice')?.value || 'desktop');
  const showDate = $('#serpShowDate')?.checked !== false;

  const urlObj = buildFullUrl();

  const titleMaxPx = device==='mobile' ? 550 : 600;
  const metaLinePx = device==='mobile' ? 540 : 460;
  const t = ellipsizeByWidth(title, titleMaxPx, 'bold 18px Arial');
  const metaLines = wrapToLines(meta, metaLinePx, '13px Arial', 2);

  $('#serpTitle').textContent = t;
  $('#serpMeta').textContent  = metaLines.join(' ');
  $('#serpDomain').textContent = urlObj.domain + (urlObj.path || '');
  $('#serpDate').textContent   = showDate ? new Date().toLocaleDateString(undefined,{month:'short', day:'2-digit', year:'numeric'}) : '';

  const note = $('#fullUrlNote');
  if (note) note.textContent = urlObj.full ? 'Previewing: ' + urlObj.full : '';
}

// ---------- Keyword density ----------
function frequency(text){
  const words = getWords(text);
  const total = words.length;
  const minLen = Number($('#minLen')?.value ?? 3);
  const ignore = $('#ignoreStopwords')?.checked ?? true;
  const map = new Map();
  for (const w of words){
    if (ignore && STOPWORDS.has(w)) continue;
    if (w.length < minLen) continue;
    map.set(w, (map.get(w)||0)+1);
  }
  const arr = [...map.entries()].sort((a,b)=> b[1]-a[1]);
  return {arr, total};
}

// ---------- UI Updates ----------
function formatTimeFromWords(totalWords, wpm){
  if (wpm <= 0) return '0:00';
  const minutes = totalWords / wpm;
  const mm = Math.floor(minutes);
  const ss = Math.round((minutes - mm) * 60);
  return mm + ':' + String(ss).padStart(2,'0');
}
function esc(s){ return s.replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;','\'':'&#39;'}[c])); }
function updatePreview(text){
  const showShort = $('#toggleShort')?.checked ?? true;
  const showLong  = $('#toggleLong')?.checked ?? true;
  const sentences = getSentences(text);
  const html = sentences.map(s=>{
    const wc = getWords(s).length;
    let cls = '';
    if (showLong && wc >= 25) cls = 'hl-long';
    else if (showShort && wc <= 7) cls = 'hl-short';
    return '<span class="'+cls+'">'+esc(s)+'</span>';
  }).join(' ');
  preview.innerHTML = html || '<span class="muted">Nothing to highlight yet.</span>';
}
function updateAll(){
  const text = editor.value;

  const words = getWords(text);
  const wordCount = words.length;
  const chars = text.length;
  const charsNo = text.replace(/\s/g,'').length;
  const sents = getSentences(text);
  const paras = getParagraphs(text);

  $('#statWords').textContent = wordCount;
  $('#statChars').textContent = chars;
  $('#statCharsNo') && ($('#statCharsNo').textContent = charsNo);
  $('#statSentences').textContent = sents.length;
  $('#statParagraphs') && ($('#statParagraphs').textContent = paras.length);
  $('#statAvgWs').textContent = (wordCount && sents.length) ? (wordCount/sents.length).toFixed(2) : '0';

  $('#statReadTime').textContent = formatTimeFromWords(wordCount, Number($('#readingSpeed').value));
  $('#statSpeakTime').textContent = formatTimeFromWords(wordCount, Number($('#speakingSpeed').value));

  const r = readabilityMetrics(text);
  $('#statSylPerWord').textContent = isFinite(r.asw) ? r.asw.toFixed(2) : '0';
  $('#fleschEase').textContent = isFinite(r.fleschEase) ? r.fleschEase.toFixed(1) : '–';
  $('#fkGrade').textContent = isFinite(r.fkGrade) ? r.fkGrade.toFixed(1) : '–';
  $('#gunningFog').textContent = isFinite(r.gunningFog) ? r.gunningFog.toFixed(1) : '–';

  const goal = Number($('#goalInput').value) || 0;
  const pct = goal>0 ? Math.min(100, Math.round((wordCount/goal)*100)) : 0;
  $('#goalProgress').style.width = pct + '%';
  $('#goalBadge').textContent = pct + '% complete';

  const topN = Number($('#topN')?.value ?? 20);
  const res = frequency(text);
  const arr = res.arr; const total = res.total;

  const densityRows = arr.slice(0, topN).map(([w,c])=>{
    const pct = total ? ((c/total)*100).toFixed(2) : '0.00';
    return `<tr><td>${w}</td><td>${c}</td><td>${pct}%</td></tr>`;
  }).join('');
  $('#kwTable').innerHTML = densityRows || '<tr><td colspan="3" class="muted">No terms yet.</td></tr>';

  // No Word Frequency table (removed)

  updatePreview(text);
  throttledSave();
  updateSeoCounters();
}

// ---------- Throttle save ----------
let saveTimer = null;
function throttledSave(){ clearTimeout(saveTimer); saveTimer = setTimeout(saveLocal, 400); }

// ---------- Actions (buttons that exist before script tag executes) ----------
$('#btnCopy').addEventListener('click', async ()=>{
  try{ await navigator.clipboard.writeText(editor.value); toast('Copied to clipboard'); }
  catch(e){ alert('Copy failed. Select and copy manually.'); }
});
$('#btnDownload').addEventListener('click', ()=>{
  const blob = new Blob([editor.value], {type:'text/plain;charset=utf-8'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href=url; a.download='text.txt'; a.click();
  URL.revokeObjectURL(url);
});
$('#btnPdf').addEventListener('click', ()=>{
  try{
    const jsPDF = window.jspdf.jsPDF;
    const doc = new jsPDF({unit:'pt', format:'a4'});
    const margin = 48; const width = doc.internal.pageSize.getWidth() - margin*2; let y = margin;
    const title = ($('#seoTitle')?.value || '').trim();
    const meta  = ($('#seoMeta')?.value || '').trim();
    const body  = editor.value || '';
    doc.setFont('times','normal');
    if(title){ doc.setFontSize(16); const t = doc.splitTextToSize(title, width); doc.text(t, margin, y); y += t.length*18 + 8; }
    if(meta){  doc.setFontSize(11); const m = doc.splitTextToSize(meta, width);  doc.text(m, margin, y); y += m.length*14 + 12; }
    doc.setFontSize(12);
    const lines = doc.splitTextToSize(body, width); const lh = 16;
    lines.forEach(ln=>{ if(y > doc.internal.pageSize.getHeight()-margin){ doc.addPage(); y = margin; } doc.text(ln, margin, y); y += lh; });
    doc.save('document.pdf');
  }catch(err){ console.error(err); alert('PDF export failed. Please try again.'); }
});
$('#btnShare').addEventListener('click', async ()=>{
  const text = editor.value.trim();
  if(navigator.share && text){
    try{ await navigator.share({ text }); }catch(e){}
  } else {
    try{ await navigator.clipboard.writeText(text); toast('Copied text (sharing not supported)'); }
    catch(e){ alert('Share not supported. Text copy failed.'); }
  }
});
$('#btnClear').addEventListener('click', ()=>{
  if(!editor.value) return;
  if(confirm('Clear the editor? This cannot be undone.')){ editor.value=''; updateAll(); }
});
$('#btnResetSettings').addEventListener('click', ()=>{
  localStorage.removeItem('wc_data_v1');
  location.reload();
});

// Settings bindings (defensive for optional elements)
$('#fontSize').addEventListener('change', e=>{ state.fontSize = Number(e.target.value); editor.style.fontSize = state.fontSize + 'px'; updateAll(); });
$('#readingSpeed').addEventListener('change', e=>{ state.readingWPM = Number(e.target.value); updateAll(); });
$('#speakingSpeed').addEventListener('change', e=>{ state.speakingWPM = Number(e.target.value); updateAll(); });
$('#ignoreStopwords').addEventListener('change', e=>{ state.ignoreStopwords = e.target.checked; updateAll(); });

const topNEl = $('#topN');
const minLenEl = $('#minLen');
topNEl && topNEl.addEventListener('change', e=>{ state.topN = Number(e.target.value); updateAll(); });
minLenEl && minLenEl.addEventListener('change', e=>{ state.minLen = Number(e.target.value); updateAll(); });

$('#goalInput').addEventListener('input', e=>{ state.goal = Number(e.target.value)||0; updateAll(); });
$('#toggleShort') && $('#toggleShort').addEventListener('change', e=>{ state.highlightShort = e.target.checked; updateAll(); });
$('#toggleLong') && $('#toggleLong').addEventListener('change', e=>{ state.highlightLong = e.target.checked; updateAll(); });

// Inputs affecting SEO/SERP
document.addEventListener('input', e=>{
  const id = e.target && e.target.id || '';
  if (['seoTitle','seoMeta','seoImage'].includes(id)){ updateSeoCounters(); throttledSave(); }
  if (['serpUrl','serpSlug','serpShowDate','serpDevice'].includes(id)){ updateSerpPreview(); updateHeadSeo(); throttledSave(); }
});

// ---------- Toast ----------
function toast(msg){
  const el = document.createElement('div');
  el.textContent = msg;
  el.style.position='fixed'; el.style.bottom='20px'; el.style.left='50%';
  el.style.transform='translateX(-50%)'; el.style.background='#111827'; el.style.color='#fff';
  el.style.padding='8px 12px'; el.style.borderRadius='10px'; el.style.fontSize='14px'; el.style.zIndex='9999'; el.style.opacity='0.95';
  document.body.appendChild(el);
  setTimeout(()=>el.remove(), 1400);
}

// ---------- Modals ----------
function openModal(id){ document.getElementById(id).style.display = 'block'; }
function closeModal(id){ document.getElementById(id).style.display = 'none'; }

// Bind modal events AFTER DOM is fully parsed (modals are below script tag)
document.addEventListener('DOMContentLoaded', () => {
  document.querySelectorAll('[data-modal]').forEach(a=>{
    a.addEventListener('click', e=>{
      e.preventDefault();
      openModal(a.getAttribute('data-modal'));
    });
  });
  document.querySelectorAll('.close').forEach(btn=>{
    btn.addEventListener('click', ()=> closeModal(btn.getAttribute('data-close')));
  });
  window.addEventListener('click', e=>{
    document.querySelectorAll('.modal').forEach(m=>{
      if (e.target === m) m.style.display = 'none';
    });
  });
  window.addEventListener('keydown', e=>{
    if (e.key === 'Escape') document.querySelectorAll('.modal').forEach(m=> m.style.display='none');
  });
});

// ---------- Self-tests (console) ----------
(function runTests(){
  const results = []; const assert = (n,c)=>results.push({n,pass:!!c});
  const lines = 'a\nb\nc'.split(/\n/); assert('Regex split on newlines', lines.length===3);
  const g = readabilityMetrics('This is simple. Another sentence.'); assert('Readability finite', isFinite(g.fleschEase));
  const canvas2 = document.createElement('canvas').getContext('2d'); assert('Canvas ctx', !!canvas2);
  if(console.groupCollapsed){
    console.groupCollapsed(`Self-tests: ${results.filter(x=>x.pass).length}/${results.length} passed`);
    results.forEach(t=> console[t.pass?'log':'error'](`${t.pass?'✔':'✖'} ${t.n}`));
    console.groupEnd();
  }else{
    console.log(results);
  }
})();

// ---------- Hidden SEO file generators (console-only URLs) ----------
(function(){
  const base = location.origin;
  const robots = `User-agent: *
Allow: /
Disallow: /private/
Disallow: /tmp/
Disallow: /backup/
Sitemap: ${base}/sitemap.xml`;
  window.__robotsUrl = URL.createObjectURL(new Blob([robots],{type:'text/plain'}));

  const today = new Date().toISOString().split('T')[0];
  const pages = ['/']; // SPA, so main is root
  let xml = `<?xml version="1.0" encoding="UTF-8"?>\n<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">\n`;
  pages.forEach(p=>{
    xml += `  <url>\n    <loc>${base}${p}</loc>\n    <lastmod>${today}</lastmod>\n    <changefreq>daily</changefreq>\n    <priority>1.0</priority>\n  </url>\n`;
  });
  xml += `</urlset>`;
  window.__sitemapUrl = URL.createObjectURL(new Blob([xml],{type:'application/xml'}));

  console.log('robots.txt →', window.__robotsUrl);
  console.log('sitemap.xml →', window.__sitemapUrl);
})();

// ---------- Init ----------
function updateSerpAndSEO(){ updateSeoCounters(); updateSerpPreview(); }
function init(){
  loadLocal();
  $('#fontSize').value = String(state.fontSize);
  editor.style.fontSize = state.fontSize + 'px';
  $('#readingSpeed').value = String(state.readingWPM);
  $('#speakingSpeed').value = String(state.speakingWPM);
  $('#goalInput').value = String(state.goal);
  $('#ignoreStopwords').checked = state.ignoreStopwords;
  $('#toggleShort') && ($('#toggleShort').checked = state.highlightShort);
  $('#toggleLong') && ($('#toggleLong').checked = state.highlightLong);
  editor.addEventListener('input', updateAll);
  updateAll();
  updateSerpAndSEO();
}
document.addEventListener('DOMContentLoaded', init);
