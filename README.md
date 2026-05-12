<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Space Game FINAL SAVE</title>

<style>
body{
    margin:0;
    font-family:Arial;
    background:black;
    overflow:hidden;
    color:white;

    display:flex;
    justify-content:center;
    align-items:center;
    height:100vh;
}

canvas{
    width:900px;
    height:500px;
    border:2px solid #222;
}

.screen{
    position:fixed;
    top:0;
    left:0;
    width:100%;
    height:100%;
    background:linear-gradient(#02020a,#000);
    display:flex;
    flex-direction:column;
    justify-content:center;
    align-items:center;
    z-index:10;
}

button{
    padding:10px 15px;
    margin:5px;
    cursor:pointer;
}

#ui{
    position:absolute;
    top:10px;
    left:10px;
    background:rgba(0,0,0,0.6);
    padding:8px;
    font-size:12px;
    display:none;
}

#shopItems{
    display:grid;
    grid-template-columns:repeat(4,1fr);
    gap:8px;
    max-width:600px;
}

#levels{
    display:grid;
    grid-template-columns:repeat(5,1fr);
    gap:5px;
    max-width:600px;
}
</style>
</head>

<body>

<!-- MENU -->
<div id="menu" class="screen">
    <h1>🚀 SPACE GAME</h1>
    <button onclick="startGame()">PLAY</button>
    <button onclick="openShop()">SHOP</button>
    <button onclick="openLevels()">LEVELS</button>
</div>

<!-- SHOP -->
<div id="shop" class="screen" style="display:none;">
    <h2>🛒 SHOP</h2>

    <div style="margin-bottom:10px;font-size:14px;">
        💰 MONEY: $<span id="shopMoney">0</span>
    </div>

    <div id="shopItems"></div>
    <button onclick="backMenu()">BACK</button>
</div>

<!-- LEVELS -->
<div id="levelsScreen" class="screen" style="display:none;">
    <h2>📊 LEVELS</h2>
    <div id="levels"></div>
    <button onclick="backMenu()">BACK</button>
</div>

<!-- GAME OVER -->
<div id="gameOver" class="screen" style="display:none;">
    <h1>GAME OVER</h1>
    <button onclick="resetGame()">RESTART</button>
</div>

<!-- UI -->
<div id="ui">
💰 <span id="money">0</span> |
🏆 <span id="score">0</span> |
❤️ <span id="hpText">100</span>
</div>

<canvas id="game"></canvas>

<script>

const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");

canvas.width=900;
canvas.height=500;

// ================= SAVE SYSTEM =================
let money = parseInt(localStorage.getItem("money")) || 0;

// STATE
let running=false;

let player={x:450,y:420,targetX:450,hp:100};

let bullets=[];
let rocks=[];
let enemies=[];

let score=0;
let selectedLevel=0;

// LEVELS
const levelNames=[
"Easy 1","Easy 2",
"Normal 1","Normal 2",
"Hard 1","Hard 2",
"Expert 1","Expert 2",
"Master","Final"
];

// SHIPS
let ships=[
 {name:"Scout",color:"#00d9ff",price:0},
 {name:"Fighter",color:"#00ff88",price:30},
 {name:"Blazer",color:"#ffcc00",price:50},
 {name:"Shadow",color:"#ff00ff",price:70},
 {name:"Storm",color:"#00ffff",price:90},
 {name:"Titan",color:"#ff3300",price:120},
 {name:"Ghost",color:"#ffffff",price:150},
 {name:"Ultra",color:"#66ff66",price:200}
];

let owned=[true,false,false,false,false,false,false,false];
let ship=0;

// SHOOT
let shooting=false;
let lastShot=0;

// ================= MENU =================
function startGame(){
    hideAll();
    document.getElementById("ui").style.display="block";
    running=true;
}

function openShop(){
    hideAll();
    document.getElementById("shop").style.display="flex";
    renderShop();
}

function openLevels(){
    hideAll();
    document.getElementById("levelsScreen").style.display="flex";
    renderLevels();
}

function backMenu(){
    hideAll();
    document.getElementById("menu").style.display="flex";
}

function resetGame(){
    location.reload();
}

function hideAll(){
    document.querySelectorAll(".screen").forEach(s=>s.style.display="none");
}

// ================= SHOP =================
function renderShop(){

    document.getElementById("shopMoney").innerText=money;

    let box=document.getElementById("shopItems");
    box.innerHTML="";

    ships.forEach((s,i)=>{

        let d=document.createElement("div");
        d.style.background="#111";
        d.style.padding="6px";
        d.style.textAlign="center";
        d.style.fontSize="11px";

        d.innerHTML=`
            <svg width="40" height="40">
                <polygon points="20,5 5,35 20,25 35,35"
                fill="${s.color}" />
            </svg>
            <div>${s.name}</div>
            <div>$${s.price}</div>
            <button>${owned[i]?"SELECT":"BUY"}</button>
        `;

        d.querySelector("button").onclick=()=>{
            if(owned[i]) ship=i;
            else if(money>=s.price){
                money-=s.price;
                localStorage.setItem("money",money); // 💾 SAVE
                owned[i]=true;
                ship=i;
                updateUI();
                renderShop();
            }
        };

        box.appendChild(d);
    });
}

// ================= LEVELS =================
function renderLevels(){

    let box=document.getElementById("levels");
    box.innerHTML="";

    levelNames.forEach((n,i)=>{

        let b=document.createElement("div");

        b.innerText=(i+1)+" "+n;

        b.style.padding="6px";
        b.style.fontSize="10px";
        b.style.textAlign="center";
        b.style.border="1px solid #333";

        b.style.background=(i===selectedLevel)?"#00ff88":"#111";
        b.style.color=(i===selectedLevel)?"#000":"#fff";

        b.onclick=()=>{
            selectedLevel=i;
            renderLevels();
        };

        box.appendChild(b);
    });
}

// ================= INPUT =================
canvas.addEventListener("mousemove",(e)=>{
    let r=canvas.getBoundingClientRect();
    player.targetX=e.clientX-r.left;
});

canvas.addEventListener("mousedown",()=>shooting=true);
canvas.addEventListener("mouseup",()=>shooting=false);

// ================= SPAWN =================
setInterval(()=>{
    if(!running) return;

    rocks.push({
        x:Math.random()*850,
        y:-20,
        size:15,
        speed:1+selectedLevel*0.2
    });
},500);

// ================= UPDATE =================
function update(){

    player.x+=(player.targetX-player.x)*0.35;

    if(shooting && Date.now()-lastShot>120){
        bullets.push({x:player.x,y:player.y});
        lastShot=Date.now();
    }

    bullets.forEach((b,i)=>{
        b.y-=14;
        if(b.y<0) bullets.splice(i,1);
    });

    rocks.forEach((r,ri)=>{
        r.y+=r.speed;

        if(r.y>500){
            player.hp-=10;
            rocks.splice(ri,1);
        }

        bullets.forEach((b,bi)=>{
            if(Math.abs(b.x-r.x)<20 && Math.abs(b.y-r.y)<20){
                rocks.splice(ri,1);
                bullets.splice(bi,1);
                money+=5;

                localStorage.setItem("money",money); // 💾 SAVE
                score+=10;
                updateUI();
            }
        });
    });

    if(player.hp<=0){
        running=false;
        document.getElementById("gameOver").style.display="flex";
    }
}

// ================= DRAW =================
function draw(){

    // BACKGROUND (ORIGINAL)
    let g=ctx.createRadialGradient(450,250,50,450,250,600);
    g.addColorStop(0,"#0a0a2a");
    g.addColorStop(1,"#000");

    ctx.fillStyle=g;
    ctx.fillRect(0,0,900,500);

    // STARS
    for(let i=0;i<70;i++){
        let x=(i*120)%900;
        let y=(i*70)%500;
        ctx.fillStyle="white";
        ctx.fillRect(x,y,2,2);
    }

    // SHIP
    ctx.fillStyle=ships[ship].color;
    ctx.beginPath();
    ctx.moveTo(player.x,player.y-18);
    ctx.lineTo(player.x-14,player.y+18);
    ctx.lineTo(player.x+14,player.y+18);
    ctx.closePath();
    ctx.fill();

    // BULLETS
    ctx.fillStyle="yellow";
    bullets.forEach(b=>{
        ctx.fillRect(b.x,b.y,3,10);
    });

    // ROCKS (VISIBLE FIX)
    ctx.fillStyle="#bbbbbb";
    rocks.forEach(r=>{
        ctx.beginPath();
        ctx.arc(r.x,r.y,r.size,0,Math.PI*2);
        ctx.fill();
    });

    // HP BAR + NUMBER ❤️
    ctx.fillStyle="red";
    ctx.fillRect(10,460,player.hp*2,10);

    ctx.fillStyle="white";
    ctx.font="14px Arial";
    ctx.fillText("HP: "+player.hp,10,455);
}

// ================= UI =================
function updateUI(){
    document.getElementById("money").innerText=money;
    document.getElementById("score").innerText=score;
    document.getElementById("hpText").innerText=player.hp;
}

// ================= LOOP =================
function loop(){
    update();
    draw();
    requestAnimationFrame(loop);
}
loop();

</script>

</body>
</html> 
