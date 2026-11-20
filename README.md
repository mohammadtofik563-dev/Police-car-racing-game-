# Police-car-racing-game-

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Police Chase Racing 3D</title>
<style>
*{box-sizing:border-box}
body{
 margin:0;background:#080d18;color:white;
 font-family:Arial;text-align:center;
 overflow:hidden;touch-action:none
}
h2{color:#00eaff;margin:8px}
#hud{
 display:flex;justify-content:space-around;
 padding:8px;font-size:15px;font-weight:bold
}
canvas{
 display:block;margin:auto;
 width:min(100vw,420px);
 height:auto;max-height:75vh;
 background:#222;
 border:2px solid #00d9ff
}
.controls{
 display:flex;justify-content:center;
 gap:20px;margin-top:10px
}
button{
 border:0;border-radius:13px;
 padding:14px 24px;font-size:22px;
 background:#00c8ff;color:#00121b;
 font-weight:bold;touch-action:none
}
.nitro{background:#ff9800}
#overlay{
 position:fixed;inset:0;background:#000d;
 display:flex;align-items:center;
 justify-content:center;z-index:5
}
.panel{
 background:#142235;padding:25px;
 border:2px solid #00d9ff;border-radius:20px;
 width:85%;max-width:340px
}
.panel h1{color:#00eaff}
.hidden{display:none!important}
</style>
</head>
<body>

<h2>🚓 POLICE CHASE RACING</h2>

<div id="hud">
 <span>🏆 <span id="score">0</span></span>
 <span>⚡ <span id="speed">0</span> KM/H</span>
 <span>🚨 <span id="level">1</span></span>
</div>

<canvas id="game" width="400" height="700"></canvas>

<div class="controls">
 <button id="left">⬅️</button>
 <button id="nitro" class="nitro">⚡</button>
 <button id="right">➡️</button>
</div>

<div id="overlay">
 <div class="panel">
  <h1>🚓 POLICE CHASE</h1>
  <p>Police से बचो और सबसे ज्यादा दूरी तय करो!</p>
  <p>⬅️ ➡️ कार चलाओ</p>
  <p>⚡ Nitro से स्पीड बढ़ाओ</p>
  <button onclick="startGame()">START GAME 🏁</button>
 </div>
</div>

<script>
const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");

let player,enemies,police;
let score=0,speed=5,frame=0;
let running=false,nitro=false;
let roadOffset=0,steer=0;
let audio=null;

const W=400,H=700;
const lanes=[135,200,265];

function sound(f,d=0.1){
 try{
  if(!audio) audio=new(window.AudioContext||window.webkitAudioContext)();
  let osc=audio.createOscillator();
  let gain=audio.createGain();
  osc.frequency.value=f;
  gain.gain.value=.035;
  osc.connect(gain);
  gain.connect(audio.destination);
  osc.start();
  gain.gain.exponentialRampToValueAtTime(.001,audio.currentTime+d);
  osc.stop(audio.currentTime+d);
 }catch(e){}
}

function startGame(){
 document.getElementById("overlay").classList.add("hidden");
 player={x:200,y:565,w:48,h:85};
 enemies=[];
 police={x:200,y:430};
 score=0;speed=5;frame=0;
 running=true;
 sound(500);
 requestAnimationFrame(loop);
}

function drawRoad(){
 let g=ctx.createLinearGradient(0,0,0,H);
 g.addColorStop(0,"#202a30");
 g.addColorStop(1,"#555");
 ctx.fillStyle=g;
 ctx.fillRect(0,0,W,H);

 // roadside
 ctx.fillStyle="#183b22";
 ctx.fillRect(0,0,55,H);
 ctx.fillRect(345,0,55,H);

 // road
 ctx.fillStyle="#292929";
 ctx.beginPath();
 ctx.moveTo(100,0);
 ctx.lineTo(300,0);
 ctx.lineTo(400,700);
 ctx.lineTo(0,700);
 ctx.closePath();
 ctx.fill();

 // road edges
 ctx.strokeStyle="white";
 ctx.lineWidth=5;
 ctx.beginPath();
 ctx.moveTo(100,0);ctx.lineTo(0,700);
 ctx.moveTo(300,0);ctx.lineTo(400,700);
 ctx.stroke();

 // lane markings
 ctx.strokeStyle="#fff";
 ctx.lineWidth=4;
 for(let x of [166,234]){
  for(let y=-50;y<750;y+=100){
   let yy=(y+roadOffset)%800;
   let width=3+yy/100;
   ctx.beginPath();
   ctx.moveTo(x-width,yy);
   ctx.lineTo(x+width,yy+40+yy/20);
   ctx.stroke();
  }
 }

 roadOffset+=speed*1.5;
}

function drawCar(x,y,color,policeCar=false){
 ctx.save();
 ctx.translate(x,y);

 // shadow
 ctx.fillStyle="#111";
 ctx.fillRect(-25,8,55,82);

 // body
 ctx.fillStyle=color;
 ctx.beginPath();
 ctx.roundRect(-23,0,46,82,9);
 ctx.fill();

 // windows
 ctx.fillStyle="#172e43";
 ctx.beginPath();
 ctx.roundRect(-17,13,34,25,5);
 ctx.fill();

 ctx.fillStyle="#111";
 ctx.fillRect(-17,48,34,15);

 // lights
 ctx.fillStyle=policeCar?"#fff":"#ffe66b";
 ctx.fillRect(-19,3,12,6);
 ctx.fillRect(7,3,12,6);

 // wheels
 ctx.fillStyle="#050505";
 ctx.fillRect(-28,13,7,20);
 ctx.fillRect(21,13,7,20);
 ctx.fillRect(-28,55,7,20);
 ctx.fillRect(21,55,7,20);

 if(policeCar){
  ctx.fillStyle="#fff";
  ctx.fillRect(-18,39,36,7);
  ctx.fillStyle=frame%20<10?"red":"blue";
  ctx.fillRect(-13,-5,12,6);
  ctx.fillStyle=frame%20<10?"blue":"red";
  ctx.fillRect(1,-5,12,6);
 }
 ctx.restore();
}

function spawnEnemy(){
 let x=lanes[Math.floor(Math.random()*3)];
 enemies.push({
  x:x,
  y:-100,
  color:["#ff3030","#e8e8e8","#ffd600","#29aaff"][Math.floor(Math.random()*4)]
 });
}

function crash(){
 running=false;
 sound(100,.5);
 document.getElementById("overlay").classList.remove("hidden");
 document.querySelector(".panel").innerHTML=`
 <h1>💥 GAME OVER</h1>
 <h2>Score: ${Math.floor(score)}</h2>
 <p>Police ने तुम्हें पकड़ लिया!</p>
 <button onclick="startGame()">PLAY AGAIN 🔄</button>`;
}

function loop(){
 if(!running)return;
 frame++;

 drawRoad();

 // police chase
 police.x+=(player.x-police.x)*.018;
 police.y=player.y-140+Math.sin(frame/15)*5;
 drawCar(police.x,police.y,"#eeeeee",true);

 // traffic
 if(frame%65===0)spawnEnemy();

 for(let e of enemies){
  e.y+=speed*(nitro?1.8:1);
  drawCar(e.x,e.y,e.color);

  if(Math.abs(player.x-e.x)<40 &&
     Math.abs(player.y-e.y)<70){
    crash();return;
  }
 }
 enemies=enemies.filter(e=>e.y<750);

 // player
 player.x+=steer*5;
 player.x=Math.max(112,Math.min(288,player.x));

 drawCar(player.x,player.y,"#00e5ff");

 score+=nitro?0.3:0.15;
 speed=Math.min(13,5+score/150);

 document.getElementById("score").innerText=Math.floor(score);
 document.getElementById("speed").innerText=
 Math.floor(speed*22*(nitro?1.5:1));
 document.getElementById("level").innerText=
 Math.floor(score/100)+1;

 if(frame%160===0)sound(180,.05);

 requestAnimationFrame(loop);
}

// Controls
function hold(id,value){
 let b=document.getElementById(id);
 b.addEventListener("pointerdown",e=>{
  e.preventDefault();
  if(id==="nitro")nitro=true;
  else steer=value;
 });
 function stop(){
  if(id==="nitro")nitro=false;
  else if(steer===value)steer=0;
 }
 b.addEventListener("pointerup",stop);
 b.addEventListener("pointerleave",stop);
 b.addEventListener("pointercancel",stop);
}
hold("left",-1);
hold("right",1);
hold("nitro",0);

document.addEventListener("keydown",e=>{
 if(e.key==="ArrowLeft")steer=-1;
 if(e.key==="ArrowRight")steer=1;
 if(e.code==="Space")nitro=true;
});
document.addEventListener("keyup",e=>{
 if(e.key==="ArrowLeft"||e.key==="ArrowRight")steer=0;
 if(e.code==="Space")nitro=false;
});
</script>
</body>
</html>
