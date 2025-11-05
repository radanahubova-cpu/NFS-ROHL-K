<!doctype html>
<html lang="cs">
<head>
<meta charset="utf-8">
<title>Need for Speed Rohlík — Poruba 2025</title>
<meta name="viewport" content="width=device-width,initial-scale=1">
<style>
html,body{height:100%; margin:0; background:#87ceeb; font-family:Arial,Helvetica,sans-serif;}
#game{display:block; width:100%; height:100vh;}
#hud{position:absolute; left:12px; top:12px; background:rgba(0,0,0,0.45); padding:10px; border-radius:8px; color:#fff;}
#hud div{margin:6px 0;}
#instructions{position:absolute; right:12px; bottom:12px; background:rgba(0,0,0,0.35); padding:10px; border-radius:8px; font-size:13px;}
#finish{position:absolute; left:50%; top:40%; transform:translateX(-50%); display:none; background:rgba(0,0,0,0.8); padding:14px; border-radius:8px;}
button{padding:8px 10px; border-radius:6px; border:0; background:#2a9df4; color:#fff; cursor:pointer;}
</style>
</head>
<body>
<canvas id="game"></canvas>
<div id="hud">
  <div><strong>Hráč:</strong> Rohlík.cz</div>
  <div id="speed">Rychlost: 0 km/h</div>
  <div id="lap">Kolo: 0 / 3</div>
  <div id="pos">Pořadí: 1</div>
  <div id="time">Čas: 0.00 s</div>
</div>
<div id="instructions">
Ovládání:<br>
W/A/S/D nebo šipky = pohyb, mezerník = brzda, R = restart<br>
Tlačítko Start pro zvuk motoru
</div>
<div id="finish"></div>
<button id="startSound" style="position:absolute; top:50%; left:50%; transform:translate(-50%,-50%);">Start / Zvuk</button>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r150/three.min.js"></script>

<script>
// --- Inicializace ---
const canvas=document.getElementById('game');
const renderer=new THREE.WebGLRenderer({canvas,antialias:true});
renderer.setSize(window.innerWidth,window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);

const scene=new THREE.Scene();
scene.background=new THREE.Color(0x87ceeb);

const camera=new THREE.PerspectiveCamera(60,window.innerWidth/window.innerHeight,0.1,1000);
camera.position.set(0,8,-14);

// Světla
scene.add(new THREE.HemisphereLight(0xffffff,0x444444,0.9));
const dir=new THREE.DirectionalLight(0xffffff,0.7);
dir.position.set(-10,20,10);
scene.add(dir);

// Zem a trať
const ground=new THREE.Mesh(new THREE.PlaneGeometry(400,400),new THREE.MeshStandardMaterial({color:0x2e8b57}));
ground.rotation.x=-Math.PI/2;
ground.receiveShadow=true;
scene.add(ground);

// Jednoduché domy
function buildCity(){
  const bMat=new THREE.MeshStandardMaterial({color:0x8d8d8d});
  for(let i=-1;i<=1;i+=2){
    for(let j=0;j<8;j++){
      const h=3+Math.random()*6;
      const box=new THREE.Mesh(new THREE.BoxGeometry(6,h,6),bMat);
      box.position.set(i*18,h/2,-40+j*12);scene.add(box);
    }
  }
  const road=new THREE.Mesh(new THREE.PlaneGeometry(120,24),new THREE.MeshStandardMaterial({color:0x222222}));
  road.rotation.x=-Math.PI/2; road.position.z=0; scene.add(road);
}
buildCity();

const waypoints=[];
for(let i=0;i<64;i++){
  const t=i/64*Math.PI*2;
  waypoints.push(new THREE.Vector3(Math.cos(t)*35,0,Math.sin(t)*18));
}

// --- Auto class ---
class SimpleCar{
  constructor(name,color,isPlayer=false,logoText=null){
    this.name=name; this.isPlayer=isPlayer;
    const body=new THREE.Mesh(new THREE.BoxGeometry(2.6,1.8,4.6), new THREE.MeshStandardMaterial({color}));
    const cabin=new THREE.Mesh(new THREE.BoxGeometry(1.6,0.6,1.8), new THREE.MeshStandardMaterial({color:0xffffff,opacity:0.4,transparent:true}));
    cabin.position.set(0,0.6,0); body.add(cabin);
    if(logoText){
      const canvasLogo=document.createElement('canvas');
      canvasLogo.width=128; canvasLogo.height=32;
      const ctx=canvasLogo.getContext('2d');
      ctx.fillStyle='#ffffff'; ctx.font='28px Arial'; ctx.fillText(logoText,2,24);
      const tex=new THREE.CanvasTexture(canvasLogo);
      const logoMesh=new THREE.Mesh(new THREE.PlaneGeometry(2,0.5),new THREE.MeshBasicMaterial({map:tex,transparent:true}));
      logoMesh.position.set(0,0.9,-2); logoMesh.rotation.x=-0.05; body.add(logoMesh);
    }
    this.mesh=body; scene.add(this.mesh);
    this.pos=new THREE.Vector3(); this.vel=new THREE.Vector3();
    this.heading=0; this.speed=0; this.maxSpeed=isPlayer?0.8:0.65;
    this.acc=0.02; this.turnSpeed=0.035; this.lap=0; this.nextWP=1; this.finished=false;
  }
  setPosition(v,heading=0){this.pos.copy(v); this.heading=heading; this.mesh.position.copy(this.pos); this.mesh.rotation.y=this.heading;}
  updateVisual(){this.mesh.position.copy(this.pos); this.mesh.rotation.y=this.heading;}
}

// --- Hráč a soupeři ---
const player=new SimpleCar('Rohlík.cz',0x0088ff,true,'Rohlík.cz');
const ai1=new SimpleCar('Dodo',0xff4444,false,'Dodo');
const ai2=new SimpleCar('Albert',0x00cc66,false,'Albert');
const ai3=new SimpleCar('Košík.cz',0xffaa00,false,'Košík.cz');

player.setPosition(waypoints[0].clone().add(new THREE.Vector3(0,0,-4)),Math.atan2(waypoints[1].x-waypoints[0].x,waypoints[1].z-waypoints[0].z));
ai1.setPosition(waypoints[0].clone().add(new THREE.Vector3(2.8,0,-4)),player.heading);
ai2.setPosition(waypoints[0].clone().add(new THREE.Vector3(-2.8,0,-4)),player.heading);
ai3.setPosition(waypoints[0].clone().add(new THREE.Vector3(0,0,-7)),player.heading);

const cars=[player,ai1,ai2,ai3];

// --- Ovládání ---
const keys={};
window.addEventListener('keydown',e=>keys[e.key.toLowerCase()]=true);
window.addEventListener('keyup',e=>keys[e.key.toLowerCase()]=false);

// --- Zvuky ---
const audioMotor=new Audio('https://freesound.org/data/previews/331/331912_3248244-lq.mp3'); // ukázkový motor
const audioCrash=new Audio('https://freesound.org/data/previews/35/35653_35107-lq.mp3'); // kolize

document.getElementById('startSound').addEventListener('click',()=>{
  audioMotor.loop=true; audioMotor.play();
  document.getElementById('startSound').style.display='none';
});

// --- AI update ---
function updateAI(ai,dt){
  if(ai.finished) return;
  const target=waypoints[ai.nextWP]; const toTarget=new THREE.Vector3().subVectors(target,ai.pos);
  const desired=Math.atan2(toTarget.x,toTarget.z);
  let diff=desired-ai.heading; while(diff>Math.PI) diff-=Math.PI*2; while(diff<=-Math.PI) diff+=Math.PI*2;
  const steer=Math.max(-1,Math.min(1,diff*2)); ai.heading+=steer*ai.turnSpeed*0.9;
  const cornerSlow
