<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Face Ratio Lab | 顔バランス分析</title>
<style>
:root{--bg:#f7f5f2;--card:#fff;--ink:#222;--muted:#777;--line:#e7e2dc;--accent:#8a6f5a}
*{box-sizing:border-box} body{margin:0;background:var(--bg);color:var(--ink);font-family:-apple-system,BlinkMacSystemFont,"Noto Sans JP",sans-serif}
.wrap{max-width:760px;margin:auto;padding:24px 16px 60px}.hero{text-align:center;padding:28px 0}
h1{font-size:30px;margin:0 0 8px;letter-spacing:.04em}.sub{color:var(--muted);line-height:1.7}
.card{background:var(--card);border:1px solid var(--line);border-radius:20px;padding:20px;margin:16px 0;box-shadow:0 8px 30px rgba(0,0,0,.04)}
.drop{border:1.5px dashed #cfc6bd;border-radius:16px;padding:28px;text-align:center;cursor:pointer}
.drop:hover{background:#faf8f6}.drop input{display:none}.btn{border:0;border-radius:999px;padding:13px 20px;background:var(--ink);color:#fff;font-size:15px;cursor:pointer}
.btn:disabled{opacity:.4;cursor:not-allowed}.row{display:grid;grid-template-columns:1fr 1fr;gap:12px}
.stat{padding:14px;border:1px solid var(--line);border-radius:14px}.label{font-size:12px;color:var(--muted)}.value{font-size:22px;font-weight:700;margin-top:5px}
#stage{position:relative;display:none;margin-top:16px;background:#111;border-radius:16px;overflow:hidden}
#photo{display:block;width:100%;height:auto}.overlay{position:absolute;inset:0;width:100%;height:100%}
.note{font-size:12px;color:var(--muted);line-height:1.7}.status{padding:12px;border-radius:12px;background:#f3eee9;margin-top:12px}
.hidden{display:none}.bar{height:8px;background:#eee8e2;border-radius:9px;overflow:hidden}.bar i{display:block;height:100%;background:var(--accent)}
@media(max-width:520px){h1{font-size:25px}.row{grid-template-columns:1fr}}
</style>
</head>
<body>
<div class="wrap">
  <section class="hero">
    <h1>Face Ratio Lab</h1>
    <div class="sub">写真から顔のランドマークを検出し、顔の比率を可視化するMVP</div>
  </section>

  <section class="card">
    <label class="drop" id="drop">
      <input id="file" type="file" accept="image/*">
      <div style="font-size:18px;font-weight:700">写真を選択</div>
      <div class="sub">正面・明るい・顔全体が写った写真がおすすめ</div>
    </label>
    <div class="status" id="status">写真を選択すると分析を開始します。</div>
    <div id="stage">
      <img id="photo" alt="uploaded">
      <canvas id="overlay" class="overlay"></canvas>
    </div>
  </section>

  <section class="card hidden" id="results">
    <h2>顔バランス</h2>
    <div class="row" id="stats"></div>
    <p class="note" style="margin-top:16px">
      ※写真の撮影距離、レンズ、顔の角度、表情などで数値は変化します。医療上の診断・施術適応の判定ではありません。
    </p>
  </section>

  <section class="card hidden" id="interpret">
    <h2>分析メモ</h2>
    <div id="memo" class="sub"></div>
  </section>
</div>

<script type="module">
import { FaceLandmarker, FilesetResolver } from
"https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.22/vision_bundle.mjs";

const fileEl=document.querySelector("#file"), drop=document.querySelector("#drop");
const statusEl=document.querySelector("#status"), img=document.querySelector("#photo");
const stage=document.querySelector("#stage"), canvas=document.querySelector("#overlay");
const results=document.querySelector("#results"), stats=document.querySelector("#stats");
const memo=document.querySelector("#memo"), interpret=document.querySelector("#interpret");

let landmarker=null;

fileEl.addEventListener("change",e=>{if(e.target.files[0]) loadImage(e.target.files[0])});

async function init(){
  statusEl.textContent="顔分析エンジンを準備しています…";
  const vision=await FilesetResolver.forVisionTasks(
    "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.22/wasm"
  );
  landmarker=await FaceLandmarker.createFromOptions(vision,{
    baseOptions:{
      modelAssetPath:"https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task",
      delegate:"GPU"
    },
    runningMode:"IMAGE",
    numFaces:1,
    outputFaceBlendshapes:false,
    outputFacialTransformationMatrixes:false
  });
  statusEl.textContent="準備完了。写真を選択してください。";
}
init().catch(e=>{
  console.error(e);
  statusEl.textContent="分析エンジンを読み込めませんでした。この試作品はHTTPSのWebページ上で実行する必要があります。";
});

function loadImage(file){
  const reader=new FileReader();
  reader.onload=()=>{
    img.onload=()=>{
      stage.style.display="block";
      if(!landmarker){
        statusEl.textContent="写真は読み込めましたが、顔分析エンジンがまだ利用できません。HTTPSのWebページとして開いてください。";
        return;
      }
      analyze();
    };
    img.src=reader.result;
    stage.style.display="block";
    statusEl.textContent="写真を読み込みました。顔を分析しています…";
  };
  reader.onerror=()=>{statusEl.textContent="写真の読み込みに失敗しました。別の写真で試してください。";};
  reader.readAsDataURL(file);
}

function d(a,b){
  return Math.hypot(a.x-b.x,a.y-b.y);
}
function pct(x){return (x*100).toFixed(1)+"%"}
function addStat(label,value){
  const el=document.createElement("div");el.className="stat";
  el.innerHTML=`<div class="label">${label}</div><div class="value">${value}</div>`;
  stats.appendChild(el);
}

/* MediaPipe Face Landmarker indices used here:
   33/263 eye outer corners, 133/362 inner corners,
   1 nose tip, 61/291 mouth corners, 10 forehead-ish,
   152 chin, 234/454 face sides.
*/
async function analyze(){
  if(!landmarker)return;
  try{
    const r=landmarker.detect(img);
    if(!r.faceLandmarks?.length){
      statusEl.textContent="顔を検出できませんでした。正面に近い写真で試してください。";
      return;
    }
    const p=r.faceLandmarks[0];
    draw(p);

    const faceW=d(p[234],p[454]);
    const faceH=d(p[10],p[152]);
    const eyeL=d(p[133],p[33]);
    const eyeR=d(p[362],p[263]);
    const eyeGap=d(p[33],p[263]);
    const noseW=d(p[129],p[358]);
    const noseL=d(p[6],p[2]);
    const mouthW=d(p[61],p[291]);
    const upperFace=d(p[10],p[168]);
    const midFace=d(p[168],p[2]);
    const lowerFace=d(p[2],p[152]);

    stats.innerHTML="";
    addStat("顔の縦横比", (faceH/faceW).toFixed(2));
    addStat("左目 / 顔幅", pct(eyeL/faceW));
    addStat("右目 / 顔幅", pct(eyeR/faceW));
    addStat("両目間 / 顔幅", pct(eyeGap/faceW));
    addStat("鼻幅 / 顔幅", pct(noseW/faceW));
    addStat("口幅 / 顔幅", pct(mouthW/faceW));
    addStat("鼻の長さ / 顔高", pct(noseL/faceH));
    addStat("上顔面 / 顔高", pct(upperFace/faceH));
    addStat("中顔面 / 顔高", pct(midFace/faceH));
    addStat("下顔面 / 顔高", pct(lowerFace/faceH));

    const asym=Math.abs((p[33].y-p[263].y))*100;
    memo.innerHTML=
      `検出点を使って、顔幅・顔高・目・鼻・口・顔の縦分割を数値化しました。`+
      `<br>左右の目の高さ差（画像上の相対値）：約 ${asym.toFixed(1)}%`+
      `<br><br><b>次の開発</b>：理想タイプを選択し、現在値との差分を表示する機能を追加できます。`;

    results.classList.remove("hidden"); interpret.classList.remove("hidden");
    statusEl.textContent="分析が完了しました。";
  }catch(e){
    console.error(e);statusEl.textContent="分析中にエラーが発生しました。";
  }
}

function draw(p){
  const rect=img.getBoundingClientRect();
  canvas.width=img.naturalWidth;canvas.height=img.naturalHeight;
  canvas.style.width=rect.width+"px";canvas.style.height=rect.height+"px";
  const ctx=canvas.getContext("2d");
  ctx.clearRect(0,0,canvas.width,canvas.height);
  ctx.fillStyle="rgba(255,255,255,.9)";
  for(const q of p){
    ctx.beginPath();ctx.arc(q.x*canvas.width,q.y*canvas.height,Math.max(2,canvas.width/260),0,Math.PI*2);ctx.fill();
  }
}
</script>
</body>
</html>