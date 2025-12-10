<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Crystal Match Deluxe</title>
<style>
body {
    margin: 0;
    background: #111;
    color: white;
    user-select: none;
    font-family: Arial;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: flex-start;
    height: 100vh;
}
#wrap {
    text-align: center;
    margin-top: 20px;
}
#gameCanvas {
    background: #00000033;
    border-radius: 20px;
}
button {
    padding: 10px 20px;
    font-size: 18px;
    border-radius: 8px;
    margin-top: 10px;
}
#loginBox {
    background: #222;
    padding: 20px;
    border-radius: 10px;
    margin-bottom: 20px;
}
#leaderboard {
    margin-top: 20px;
}
</style>
</head>
<body>

<div id="wrap">

<!-- Вход -->
<div id="loginBox">
    <h2>Войти в игру</h2>
    <input id="nameInput" type="text" placeholder="Ваше имя">
    <br>
    <button onclick="login()">Играть</button>
</div>

<!-- UI -->
<div id="ui" style="display:none;">
    <div id="levelText"></div>
    <div id="taskText"></div>
    <div id="progressText"></div>
</div>

<!-- Canvas -->
<canvas id="gameCanvas" width="480" height="480" style="display:none;"></canvas>

<!-- Таблица лидеров -->
<div id="leaderboard" style="display:none;"></div>
</div>

<script>
// ---------------------- БАЗА ДАННЫХ -------------------------
let players = JSON.parse(localStorage.getItem("players") || "{}");
let currentPlayer = null;

// ---------------------- ПАРАМЕТРЫ ИГРЫ ----------------------
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");
const gridSize = 8;
const cellSize = 60;
const crystalColors = ["#ff4040","#4090ff","#ffe140","#40ff7a","#d840ff"];
let board = [];
let anim = [];
let selected = null;

// ---------------------- ФУНКЦИИ UI -------------------------
function updateUI(){
    document.getElementById("levelText").innerText =
        "Игрок: "+Object.keys(players).find(k=>players[k]===currentPlayer);
    document.getElementById("taskText").innerText =
        "Уровень: "+currentPlayer.level;
    document.getElementById("progressText").innerText =
        "Пройдено: "+currentPlayer.progress+" / "+currentPlayer.goal;
}

function drawLeaderboard(){
    let list = Object.entries(players)
        .sort((a,b)=>b[1].level-a[1].level)
        .map(([name,data],i)=>`${i+1}. ${name} — уровней: ${data.level}`)
        .join("<br>");
    document.getElementById("leaderboard").innerHTML =
        "<h3>Таблица игроков</h3>"+list;
}

// ---------------------- ВХОД -------------------------
function login(){
    const name = document.getElementById("nameInput").value.trim();
    if(!name) return alert("Введите имя!");

    if(!players[name]) {
        players[name] = {level:1,goal:20,progress:0,board:null};
    }

    currentPlayer = players[name];
    document.getElementById("loginBox").style.display="none";
    document.getElementById("ui").style.display="block";
    canvas.style.display="block";
    document.getElementById("leaderboard").style.display="block";

    // Восстанавливаем поле игрока если есть
    if(currentPlayer.board){
        board = JSON.parse(JSON.stringify(currentPlayer.board));
    } else {
        initBoard();
    }

    drawBoard();
    updateUI();
    drawLeaderboard();
}

// ---------------------- ИНИЦИАЛИЗАЦИЯ ПОЛЯ -------------------------
function randomCrystal(){return Math.floor(Math.random()*5);}
function initBoard(){
    board=[]; anim=[];
    for(let i=0;i<gridSize;i++){
        board[i]=[];
        for(let j=0;j<gridSize;j++){
            board[i][j]=randomCrystal();
        }
    }
}

// ---------------------- РИСОВАНИЕ КРИСТАЛЛОВ -------------------------
function drawCrystal(type,x,y,size,alpha=1){
    ctx.globalAlpha=alpha;
    ctx.save();
    ctx.translate(x+size/2,y+size/2);
    ctx.beginPath();
    ctx.fillStyle=crystalColors[type];
    if(type===0){ctx.moveTo(0,-20);ctx.lineTo(20,0);ctx.lineTo(0,20);ctx.lineTo(-20,0);}
    else if(type===1){ctx.moveTo(0,-22);ctx.lineTo(22,18);ctx.lineTo(-22,18);}
    else if(type===2){ctx.moveTo(-20,0);ctx.lineTo(-10,-18);ctx.lineTo(10,-18);ctx.lineTo(20,0);ctx.lineTo(10,18);ctx.lineTo(-10,18);}
    else if(type===3){ctx.moveTo(-15,-25);ctx.lineTo(15,-10);ctx.lineTo(0,25);}
    else if(type===4){ctx.ellipse(0,0,18,25,0,0,Math.PI*2);}
    ctx.fill(); ctx.restore(); ctx.globalAlpha=1;
}

// ---------------------- ОТРИСОВКА ПОЛЯ -------------------------
function drawBoard(){
    ctx.clearRect(0,0,canvas.width,canvas.height);
    for(let i=0;i<gridSize;i++){
        for(let j=0;j<gridSize;j++){
            drawCrystal(board[i][j], j*cellSize, i*cellSize, cellSize);
        }
    }
}

// ---------------------- СОВПАДЕНИЯ -------------------------
function findMatches(){
    let toRemove=[];
    for(let i=0;i<gridSize;i++){
        for(let j=0;j<gridSize-2;j++){
            let a=board[i][j];
            if(a===board[i][j+1]&&a===board[i][j+2]){toRemove.push([i,j],[i,j+1],[i,j+2]);}
        }
    }
    for(let j=0;j<gridSize;j++){
        for(let i=0;i<gridSize-2;i++){
            let a=board[i][j];
            if(a===board[i+1][j]&&a===board[i+2][j]){toRemove.push([i,j],[i+1,j],[i+2,j]);}
        }
    }
    return toRemove;
}

// ---------------------- АНИМАЦИЯ ИСЧЕЗНОВЕНИЯ -------------------------
function fadeOut(matches){
    anim=matches.map(([i,j])=>({i,j,alpha:1}));
    return new Promise(resolve=>{
        let fade=setInterval(()=>{
            anim.forEach(a=>a.alpha-=0.1);
            ctx.clearRect(0,0,canvas.width,canvas.height);
            for(let i=0;i<gridSize;i++){
                for(let j=0;j<gridSize;j++){
                    let alpha=1;
                    anim.forEach(a=>{if(a.i===i&&a.j===j) alpha=a.alpha;});
                    drawCrystal(board[i][j], j*cellSize, i*cellSize, cellSize, alpha);
                }
            }
            if(anim[0].alpha<=0){clearInterval(fade);resolve();}
        },40);
    });
}

// ---------------------- ГРАВИТАЦИЯ -------------------------
function gravity(){
    for(let j=0;j<gridSize;j++){
        let col=board.map(r=>r[j]).filter(v=>v!=null);
        while(col.length<gridSize) col.unshift(randomCrystal());
        for(let i=0;i<gridSize;i++) board[i][j]=col[i];
    }
}

// ---------------------- КЛИКИ -------------------------
canvas.onclick=async function(e){
    let j=Math.floor(e.offsetX/cellSize);
    let i=Math.floor(e.offsetY/cellSize);
    if(!selected){selected=[i,j];return;}
    let [i1,j1]=selected; selected=null;
    if(Math.abs(i-i1)+Math.abs(j-j1)!=1) return;
    [board[i][j],board[i1][j1]]=[board[i1][j1],board[i][j]];
    let matches=findMatches();
    if(matches.length===0){[board[i][j],board[i1][j1]]=[board[i1][j1],board[i][j]];return;}
    await fadeOut(matches);
    matches.forEach(([x,y])=>board[x][y]=null);
    gravity();
    drawBoard();
    // Обновляем прогресс игрока
    currentPlayer.progress+=matches.length;
    if(currentPlayer.progress>=currentPlayer.goal){
        alert("Уровень пройден!");
        currentPlayer.level++;
        currentPlayer.goal+=10;
        currentPlayer.progress=0;
    }
    // Сохраняем прогресс + текущее поле
    currentPlayer.board = JSON.parse(JSON.stringify(board));
    players[Object.keys(players).find(k=>players[k]===currentPlayer)] = currentPlayer;
    localStorage.setItem("players",JSON.stringify(players));
    updateUI();
    drawLeaderboard();
};

// ---------------------- СТАРТ -------------------------
drawBoard();
</script>

</body>
</html>
