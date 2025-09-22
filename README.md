<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Block Blast Melhorado</title>
<style>
  :root{--bg:#111;--panel:#222;--cell:#333}
  *{box-sizing:border-box}
  html,body{height:100%;margin:0;background:var(--panel);color:#fff;font-family:Arial,Helvetica,sans-serif;-webkit-tap-highlight-color:transparent}
  .center{min-height:100vh;display:flex;align-items:center;justify-content:center;padding:12px}
  #canvas{background:var(--bg);border-radius:10px;touch-action:none;display:block;box-shadow:0 10px 30px rgba(0,0,0,0.6)}
  .ui{position:fixed;left:12px;top:12px;background:rgba(0,0,0,0.35);padding:8px 10px;border-radius:8px;font-weight:600}
  .menu{position:fixed;left:50%;top:12%;transform:translateX(-50%);background:rgba(0,0,0,0.6);padding:18px;border-radius:12px;text-align:center}
  .btn{padding:8px 12px;margin:6px;border-radius:8px;border:0;background:#444;color:#fff;cursor:pointer}
  .overlay{position:fixed;inset:0;display:none;align-items:center;justify-content:center;background:rgba(0,0,0,0.6)}
  .overlay.active{display:flex}
  .card{background:#222;padding:18px;border-radius:12px;text-align:center}
</style>
</head>
<body>
<div class="center">
  <canvas id="canvas"></canvas>
</div>

<div class="menu" id="menu">
  <h2>Block Blast</h2>
  <button class="btn" id="startBtn">Novo Jogo</button>
  <button class="btn" id="continueBtn">Continuar Jogo</button>
  <button class="btn" id="musicBtn">Música</button>
  <div style="font-size:13px;color:#ddd;margin-top:6px">Arraste as peças para o tabuleiro</div>
</div>

<div class="ui" id="scoreUI">Pontuação: 0 | Recorde: 0</div>

<div class="overlay" id="gameOverOverlay">
  <div class="card">
    <h3>Fim de jogo!</h3>
    <div id="finalMsg" style="margin:10px 0"></div>
    <div style="margin-top:8px">
      <button class="btn" id="retryBtn">Tentar novamente</button>
    </div>
  </div>
</div>

<script>
const ROWS=10,COLS=10,CELL=42,FOOTER=120,PAD=10;
const COLORS=["#e74c3c","#3498db","#2ecc71","#f1c40f","#9b59b6","#ff9800","#00bcd4"];

const canvas=document.getElementById('canvas');
const ctx=canvas.getContext('2d');
canvas.width=COLS*CELL+PAD*2;
canvas.height=ROWS*CELL+PAD*2+FOOTER;

const menu=document.getElementById('menu');
const startBtn=document.getElementById('startBtn');
const continueBtn=document.getElementById('continueBtn');
const musicBtn=document.getElementById('musicBtn');
const scoreUI=document.getElementById('scoreUI');
const overlay=document.getElementById('gameOverOverlay');
const retryBtn=document.getElementById('retryBtn');
const finalMsg=document.getElementById('finalMsg');

let board=[],pieces=[],score=0,record=0,dragging=null,dragOffset={x:0,y:0},activePointer=null;

record=parseInt(localStorage.getItem("record")||"0");

const BASE=[[[1]],[[1,1]],[[1,1,1]],[[1,1,1,1]],[[1],[1],[1]],[[1,1],[1,1]],[[1,1,1],[1,1,1],[1,1,1]],[[1,0],[1,1]],[[1,0],[1,0],[1,1]],[[1,1,1],[0,1,0]],[[0,1,1],[1,1,0]],[[1,1,0],[0,1,1]],[[0,1,0],[1,1,1],[0,1,0]]];
function rotate(s){const r=s.length,c=s[0].length,o=Array.from({length:c},()=>Array(r).fill(0));for(let i=0;i<r;i++)for(let j=0;j<c;j++)o[j][r-1-i]=s[i][j];return o}
function key(s){return s.map(r=>r.join("")).join("|")}
const SHAPES=(function(){const s=[],seen=new Set();for(const b of BASE){let cur=b;for(let i=0;i<4;i++){const k=key(cur);if(!seen.has(k)){seen.add(k);s.push(cur.map(r=>r.slice()))}cur=rotate(cur)}}return s})();

function initBoard(){
  board=Array.from({length:ROWS},()=>Array(COLS).fill(0));
  // adiciona 3 linhas aleatórias preenchidas no início
  for(let r=ROWS-3;r<ROWS;r++){
    for(let c=0;c<COLS;c++){
      if(Math.random()<0.5) board[r][c]=1;
    }
  }
}

function setScore(v){score=v;scoreUI.textContent="Pontuação: "+score+" | Recorde: "+record}

function spawnPieces(){
  pieces=[];
  for(let i=0;i<3;i++){
    const shape=SHAPES[Math.floor(Math.random()*SHAPES.length)];
    const color=COLORS[Math.floor(Math.random()*COLORS.length)];
    const p={shape,rows:shape.length,cols:shape[0].length,color,
      x:PAD+16+i*((canvas.width-PAD*2-16)/3),
      y:PAD+ROWS*CELL+12,
      w:shape[0].length*CELL,h:shape.length*CELL};
    pieces.push(p);
  }
}

function draw(){
  ctx.fillStyle="#111";ctx.fillRect(0,0,canvas.width,canvas.height);
  // tabuleiro
  for(let r=0;r<ROWS;r++){
    for(let c=0;c<COLS;c++){
      const x=PAD+c*CELL,y=PAD+r*CELL;
      if(board[r][c]){
        ctx.fillStyle="#0ea5a4";
        ctx.fillRect(x,y,CELL,CELL);
        ctx.strokeStyle="#000";ctx.lineWidth=2;
        ctx.strokeRect(x,y,CELL,CELL);
      }else{
        ctx.fillStyle="#07202b";
        ctx.fillRect(x,y,CELL,CELL);
      }
    }
  }
  // peças
  for(const p of pieces){
    for(let r=0;r<p.rows;r++){
      for(let c=0;c<p.cols;c++){
        if(p.shape[r][c]){
          const px=Math.round(p.x+c*CELL),py=Math.round(p.y+r*CELL);
          ctx.fillStyle=p.color;
          ctx.fillRect(px,py,CELL,CELL);
          ctx.strokeStyle="#000";ctx.lineWidth=2;
          ctx.strokeRect(px,py,CELL,CELL);
          ctx.fillStyle="rgba(255,255,255,0.2)";
          ctx.fillRect(px+CELL/4,py+CELL/4,CELL/2,CELL/2);
        }
      }
    }
  }
  requestAnimationFrame(draw);
}

function canPlaceAt(p,row,col){
  if(row<0||col<0||row+p.rows>ROWS||col+p.cols>COLS)return false;
  for(let r=0;r<p.rows;r++)for(let c=0;c<p.cols;c++)
    if(p.shape[r][c]&&board[row+r][col+c])return false;
  return true;
}
function placeAt(p,row,col){for(let r=0;r<p.rows;r++)for(let c=0;c<p.cols;c++)if(p.shape[r][c])board[row+r][col+c]=1}
function anyMoveAvailable(){for(const p of pieces){for(let r=0;r<=ROWS-p.rows;r++){for(let c=0;c<=COLS-p.cols;c++){if(canPlaceAt(p,r,c))return true}}}return false}
function clearLines(){let cl=0;for(let r=0;r<ROWS;r++){if(board[r].every(v=>v===1)){for(let c=0;c<COLS;c++)board[r][c]=0;cl++}}for(let c=0;c<COLS;c++){if(board.every(row=>row[c]===1)){for(let r=0;r<ROWS;r++)board[r][c]=0;cl++}}return cl}

// eventos
canvas.addEventListener("pointerdown",e=>{
  const rect=canvas.getBoundingClientRect();
  const x=e.clientX-rect.left,y=e.clientY-rect.top;
  for(let i=pieces.length-1;i>=0;i--){
    const p=pieces[i];
    if(x>=p.x&&x<=p.x+p.w&&y>=p.y&&y<=p.y+p.h){
      dragging=p;activePointer=e.pointerId;
      dragOffset.x=x-p.x;dragOffset.y=y-p.y;
      pieces.splice(i,1);pieces.push(p);
      canvas.setPointerCapture(e.pointerId);break;
    }
  }
});
canvas.addEventListener("pointermove",e=>{
  if(!dragging||e.pointerId!==activePointer)return;
  const rect=canvas.getBoundingClientRect();
  dragging.x=e.clientX-rect.left-dragOffset.x;
  dragging.y=e.clientY-rect.top-dragOffset.y;
});
canvas.addEventListener("pointerup",e=>{
  if(!dragging||e.pointerId!==activePointer)return;
  const rect=canvas.getBoundingClientRect();
  const col=Math.round((dragging.x-PAD)/CELL),row=Math.round((dragging.y-PAD)/CELL);
  if(canPlaceAt(dragging,row,col)){
    placeAt(dragging,row,col);
    pieces=pieces.filter(pp=>pp!==dragging);
    score+=10;
    const cl=clearLines();if(cl>0)score+=cl*100;
    setScore(score);
    if(pieces.length===0)spawnPieces();
    saveGame();
    if(!anyMoveAvailable()){gameOver()}
  }else{
    for(let i=0;i<pieces.length;i++){
      pieces[i].x=PAD+16+i*((canvas.width-PAD*2-16)/3);
      pieces[i].y=PAD+ROWS*CELL+12;
    }
  }
  try{canvas.releasePointerCapture(activePointer)}catch(e){}
  activePointer=null;dragging=null;
});

function gameOver(){
  overlay.classList.add("active");
  if(score>record){record=score;localStorage.setItem("record",record)}
  finalMsg.textContent="Sua pontuação: "+score+" | Recorde: "+record;
  localStorage.removeItem("savegame");
}
function saveGame(){const data={board,score,pieces};localStorage.setItem("savegame",JSON.stringify(data))}
function loadGame(){const d=localStorage.getItem("savegame");if(!d)return false;const data=JSON.parse(d);board=data.board;score=data.score;pieces=data.pieces;setScore(score);return true}

// botões
startBtn.addEventListener("click",()=>{menu.style.display="none";initBoard();setScore(0);spawnPieces();saveGame();draw()});
continueBtn.addEventListener("click",()=>{if(loadGame()){menu.style.display="none";draw()}});
retryBtn.addEventListener("click",()=>{overlay.classList.remove("active");initBoard();setScore(0);spawnPieces();saveGame();draw()});

// boot
(function boot(){initBoard();setScore(0);spawnPieces();menu.style.display="block"})();
</script>
</body>
</html>
