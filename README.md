<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Binance Spot Tarayıcı</title>
<style>
:root{color-scheme:dark}
*{box-sizing:border-box}
body{margin:0;background:#0b1120;color:#e5e7eb;font-family:Arial,sans-serif}
header{padding:20px 16px;background:#111827;border-bottom:1px solid #263247}
h1{margin:0 0 8px;font-size:23px}
.sub{color:#9ca3af;font-size:13px;line-height:1.6}
main{padding:16px;max-width:1400px;margin:auto}
button{background:#f0b90b;color:#111827;border:0;border-radius:8px;padding:12px 18px;font-weight:bold;font-size:15px;cursor:pointer}
button:disabled{opacity:.5;cursor:wait}
.status{margin:14px 0;padding:12px;background:#111827;border-radius:8px;font-size:14px;line-height:1.6}
.stats{display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:8px;margin:14px 0}
.stat{background:#111827;padding:12px;border-radius:8px}
.stat small{display:block;color:#9ca3af;margin-bottom:7px}
.stat strong{font-size:20px}
.tablewrap{overflow:auto;background:#111827;border-radius:10px}
table{border-collapse:collapse;width:100%;min-width:850px;font-size:13px}
th,td{padding:12px 10px;border-bottom:1px solid #263247;text-align:right;white-space:nowrap}
th{background:#182235;color:#cbd5e1;position:sticky;top:0}
th:first-child,td:first-child{text-align:left}
td:nth-child(2){text-align:left}
.buy{color:#4ade80;font-weight:bold}
.strongbuy{color:#f0b90b;font-weight:bold}
.wait{color:#cbd5e1}
.bad{color:#f87171;font-weight:bold}
.muted{color:#9ca3af}
.note{font-size:12px;color:#9ca3af;line-height:1.6;margin-top:14px}
@media(max-width:600px){h1{font-size:20px}.stats{grid-template-columns:repeat(3,minmax(0,1fr))}.stat{padding:9px}.stat strong{font-size:17px}}
</style>
</head>
<body>
<header>
<h1>Binance Spot Tarayıcı</h1>
<div class="sub">1 saatlik al-sat analizi · 4 saatlik EMA200 trend filtresi</div>
</header>
<main>
<button id="scanBtn" onclick="startScan()">TARAMAYI BAŞLAT</button>
<div id="status" class="status">Başlamak için tarama düğmesine bas.</div>
<div class="stats">
<div class="stat"><small>Taranan coin</small><strong id="total">0</strong></div>
<div class="stat"><small>AL sinyali</small><strong id="buyCount">0</strong></div>
<div class="stat"><small>Güçlü AL</small><strong id="strongCount">0</strong></div>
</div>
<div class="tablewrap">
<table>
<thead><tr>
<th>#</th><th>Coin</th><th>Fiyat (USDT)</th>
<th>24s Hacim (M USDT)</th><th>Skor</th>
<th>RSI</th><th>ADX</th><th>Sinyal</th>
</tr></thead>
<tbody id="results">
<tr><td colspan="8" class="muted" style="text-align:center">Henüz tarama yapılmadı.</td></tr>
</tbody>
</table>
</div>
<div class="note">
Hacim filtresi: 24 saatlik en az 2.000.000 USDT.<br>
Sabit coin sayısı sınırı yoktur. Hacim şartını karşılayan uygun Spot USDT pariteleri taranır. Çok sayıda coin varsa tarama zaman alabilir.<br>
Sinyaller kapanmış mumlardan hesaplanır; yatırım tavsiyesi değildir.
</div>
</main>
<script>
const API="https://api.binance.com";
const MIN_VOLUME=2000000;
const CONCURRENCY=5;
let running=false;

const $=id=>document.getElementById(id);
const fmt=(n,d=4)=>Number.isFinite(n)?n.toLocaleString("en-US",{maximumFractionDigits:d}):"-";
const sleep=ms=>new Promise(r=>setTimeout(r,ms));

async function api(path){
  const res=await fetch(API+path);
  if(!res.ok) throw new Error("Binance API yanıtı: "+res.status);
  return res.json();
}

function ema(values,period){
  if(values.length<period)return NaN;
  const k=2/(period+1);
  let v=values.slice(0,period).reduce((a,b)=>a+b,0)/period;
  for(let i=period;i<values.length;i++)v=values[i]*k+v*(1-k);
  return v;
}

function rma(values,period){
  if(values.length<period)return [];
  const out=Array(values.length).fill(NaN);
  let sum=0;
  for(let i=0;i<period;i++)sum+=values[i];
  out[period-1]=sum/period;
  for(let i=period;i<values.length;i++)
    out[i]=(out[i-1]*(period-1)+values[i])/period;
  return out;
}

function rsi(values,period=14){
  if(values.length<=period)return NaN;
  const gains=[],losses=[];
  for(let i=1;i<values.length;i++){
    const diff=values[i]-values[i-1];
    gains.push(Math.max(diff,0));
    losses.push(Math.max(-diff,0));
  }
  const ag=rma(gains,period),al=rma(losses,period);
  const g=ag[ag.length-1],l=al[al.length-1];
  if(!Number.isFinite(g)||!Number.isFinite(l))return NaN;
  if(l===0)return g===0?50:100;
  return 100-100/(1+g/l);
}

function macd(values){
  if(values.length<35)return {line:NaN,signal:NaN};
  const fast=[],slow=[];
  for(let i=0;i<values.length;i++){
    fast.push(ema(values.slice(0,i+1),12));
    slow.push(ema(values.slice(0,i+1),26));
  }
  const line=fast.map((v,i)=>v-slow[i]).filter(Number.isFinite);
  const sig=ema(line,9);
  return {line:line[line.length-1],signal:sig};
}

function adx(high,low,close,period=14){
  const tr=[],plus=[],minus=[];
  for(let i=1;i<close.length;i++){
    tr.push(Math.max(high[i]-low[i],
      Math.abs(high[i]-close[i-1]),
      Math.abs(low[i]-close[i-1])));
    const up=high[i]-high[i-1];
    const down=low[i-1]-low[i];
    plus.push(up>down&&up>0?up:0);
    minus.push(down>up&&down>0?down:0);
  }
  const atr=rma(tr,period),p=rma(plus,period),m=rma(minus,period);
  const dx=[];
  for(let i=0;i<tr.length;i++){
    if(!Number.isFinite(atr[i])||atr[i]===0||
       !Number.isFinite(p[i])||!Number.isFinite(m[i]))dx.push(NaN);
    else{
      const pdi=100*p[i]/atr[i],mdi=100*m[i]/atr[i];
      dx.push(pdi+mdi===0?0:100*Math.abs(pdi-mdi)/(pdi+mdi));
    }
  }
  const valid=dx.map((v,i)=>({v,i})).filter(x=>Number.isFinite(x.v));
  if(valid.length<period)return NaN;
  const vals=valid.map(x=>x.v);
  return rma(vals,period).slice(-1)[0];
}

function candleData(raw){
  // Son mum henüz kapanmamış olabilir; onu analiz dışında bırak.
  const rows=raw.slice(0,-1);
  return {
    open:rows.map(x=>+x[1]),
    high:rows.map(x=>+x[2]),
    low:rows.map(x=>+x[3]),
    close:rows.map(x=>+x[4]),
    volume:rows.map(x=>+x[5])
  };
}

async function analyze(symbol,ticker){
  const [raw4,raw1]=await Promise.all([
    api("/api/v3/klines?symbol="+symbol+"&interval=4h&limit=220"),
    api("/api/v3/klines?symbol="+symbol+"&interval=1h&limit=100")
  ]);

  const h4=candleData(raw4),h1=candleData(raw1);
  if(h4.close.length<200||h1.close.length<55)return null;

  const c4=h4.close.at(-1);
  const ema200=ema(h4.close,200);
  if(!Number.isFinite(ema200))return null;

  const c=h1.close,high=h1.high,low=h1.low,vol=h1.volume;
  const price=c.at(-1),previous=c.at(-2);
  const e20=ema(c,20),e50=ema(c,50);
  const r=rsi(c,14),m=macd(c);
  const a=adx(high,low,c,14);

  const tr=[];
  for(let i=0;i<c.length;i++){
    tr.push(i===0?high[i]-low[i]:Math.max(
      high[i]-low[i],Math.abs(high[i]-c[i-1]),
      Math.abs(low[i]-c[i-1])));
  }
  const atrSeries=rma(tr,14);
  const atr=atrSeries.at(-1);

  const avgVol=vol.slice(-21,-1).reduce((x,y)=>x+y,0)/20;
  const volumeRatio=avgVol>0?vol.at(-1)/avgVol:0;

  const prevHigh=Math.max(...high.slice(-21,-1));
  const breakout=price>prevHigh;
  const pullback=Number.isFinite(e20)&&price>=e20&&
                 (price-e20)/e20<=0.012;
  const trend=price>e20&&e20>e50;
  const macdUp=m.line>m.signal;
  const notLate=price<=e20*1.025&&r<72;
  const volumeOk=volumeRatio>=1.2;
  const above4h=c4>ema200;
  const sideways=Number.isFinite(a)&&a<17;

  let score=0;
  if(above4h)score+=25;
  if(trend)score+=20;
  if(r>=50&&r<=68)score+=15;
  else if(r>45&&r<72)score+=8;
  if(macdUp)score+=15;
  if(volumeRatio>=1.5)score+=15;
  else if(volumeRatio>=1.1)score+=8;
  if(Number.isFinite(a)&&a>=25)score+=10;
  else if(Number.isFinite(a)&&a>=20)score+=6;

  const entryOk=trend&&(breakout||pullback)&&notLate&&!sideways;
  let signal="BEKLE";
  if(!above4h||sideways)signal="UYGUN DEĞİL";
  else if(entryOk&&score>=80&&macdUp&&volumeOk)signal="GÜÇLÜ AL";
  else if(entryOk&&score>=65&&macdUp)signal="AL";

  return {
    symbol,price:+ticker.lastPrice,
    volume:+ticker.quoteVolume,
    score,rsi:r,adx:a,signal,
    change:+ticker.priceChangePercent
  };
}

async function pool(items,worker,limit,onProgress){
  let next=0,done=0;
  const out=[];
  async function run(){
    while(true){
      const i=next++;
      if(i>=items.length)return;
      try{
        const value=await worker(items[i]);
        if(value)out.push(value);
      }catch(e){
        // Bir coin hata verirse diğer coinlerin taraması devam eder.
      }
      done++;
      if(onProgress)onProgress(done,items.length);
      await sleep(80);
    }
  }
  await Promise.all(Array.from({length:Math.min(limit,items.length)},run));
  return out;
}

function signalClass(s){
  if(s==="GÜÇLÜ AL")return "strongbuy";
  if(s==="AL")return "buy";
  if(s==="UYGUN DEĞİL")return "bad";
  return "wait";
}

function render(rows){
  rows.sort((a,b)=>{
    const rank={"GÜÇLÜ AL":3,"AL":2,"BEKLE":1,"UYGUN DEĞİL":0};
    return rank[b.signal]-rank[a.signal]||b.score-a.score||b.volume-a.volume;
  });
  $("total").textContent=rows.length;
  $("buyCount").textContent=rows.filter(x=>x.signal==="AL").length;
  $("strongCount").textContent=rows.filter(x=>x.signal==="GÜÇLÜ AL").length;
  if(!rows.length){
    $("results").innerHTML='<tr><td colspan="8">Sonuç bulunamadı. Daha sonra tekrar dene.</td></tr>';
    return;
  }
  $("results").innerHTML=rows.map((x,i)=>`
    <tr>
      <td>${i+1}</td>
      <td><b>${x.symbol.replace("USDT","")}</b></td>
      <td>${fmt(x.price,8)}</td>
      <td>${fmt(x.volume/1e6,2)}</td>
      <td>${fmt(x.score,0)}/100</td>
      <td>${fmt(x.rsi,1)}</td>
      <td>${fmt(x.adx,1)}</td>
      <td class="${signalClass(x.signal)}">${x.signal}</td>
    </tr>`).join("");
}

async function startScan(){
  if(running)return;
  running=true;
  $("scanBtn").disabled=true;
  $("results").innerHTML="";
  $("total").textContent="0";
  $("buyCount").textContent="0";
  $("strongCount").textContent="0";
  try{
    $("status").textContent="Binance Spot pariteleri alınıyor...";
    const [info,tickers]=await Promise.all([
      api("/api/v3/exchangeInfo"),
      api("/api/v3/ticker/24hr")
    ]);
    const tickMap=new Map(tickers.map(t=>[t.symbol,t]));
    const excluded=/(UPUSDT|DOWNUSDT|BULLUSDT|BEARUSDT)$/;
    const symbols=info.symbols.filter(s=>
      s.status==="TRADING"&&s.isSpotTradingAllowed&&
      s.quoteAsset==="USDT"&&s.symbol.endsWith("USDT")&&
      !excluded.test(s.symbol)
    ).map(s=>({symbol:s.symbol,ticker:tickMap.get(s.symbol)}))
     .filter(x=>x.ticker&&+x.ticker.quoteVolume>=MIN_VOLUME)
     .sort((a,b)=>+b.ticker.quoteVolume-+a.ticker.quoteVolume);

    if(!symbols.length)throw new Error("Hacim şartını karşılayan parite bulunamadı.");

    $("status").textContent=
      symbols.length+" uygun USDT paritesi bulundu. Tarama başladı...";
    const rows=await pool(symbols,
      x=>analyze(x.symbol,x.ticker),
      CONCURRENCY,
      (done,total)=>{
        $("status").textContent=
          "Taranıyor: "+done+" / "+total+
          " coin ("+Math.round(done/total*100)+"%)";
      }
    );
    render(rows);
    $("status").textContent=
      "Tarama tamamlandı. Hacim şartını karşılayan "+symbols.length+
      " pariteden "+rows.length+" tanesi analiz edildi.";
  }catch(e){
    $("status").textContent=
      "Hata: "+e.message+
      " İnternet bağlantısını kontrol et veya biraz sonra tekrar dene.";
  }finally{
    running=false;
    $("scanBtn").disabled=false;
  }
}
</script>
</body>
</html>
