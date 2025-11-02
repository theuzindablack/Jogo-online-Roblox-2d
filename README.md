# Jogo-online-Roblox-2d

<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Multiplayer 2D Top-Down PeerJS (Conexão Única)</title>
<script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
<style>
/* Estilos principais */
body, html { margin:0; padding:0; overflow:hidden; background:#6ab04c; touch-action:none; }
canvas { display:block; background:#7bed9f; }
#joystick { position:absolute; bottom:50px; left:50px; width:120px; height:120px;
  background:rgba(255,255,255,0.2); border-radius:50%; touch-action:none; }
#stick { position:absolute; left:40px; top:40px; width:40px; height:40px;
  background:rgba(255,255,255,0.6); border-radius:50%; }
#playerList { position:absolute; top:10px; right:10px; background:rgba(0,0,0,0.6);
  color:white; padding:5px 10px; border-radius:10px; font-family:sans-serif; max-height:200px;
  overflow-y:auto; }

/* Interface de Conexão */
#connectionInfo {
    position:absolute; top:10px; left:10px; background:rgba(0,0,0,0.8);
    color:white; padding:10px; border-radius:10px; font-family:sans-serif;
    line-height: 1.5; font-size: 14px;
    display: none; 
}
#connectionInfo input, #connectionInfo button {
    padding: 5px; margin-top: 5px; border-radius: 5px; border: none;
    font-size: 14px;
}

/* Tela de Setup Inicial */
#setupScreen {
    position:absolute; top:0; left:0; width:100%; height:100%; 
    background:rgba(0,0,0,0.9); color:white; 
    display:flex; flex-direction:column; justify-content:center; align-items:center; 
    z-index:100; font-family:sans-serif;
}
#setupScreen button {
    padding: 10px 20px; margin: 10px; cursor: pointer; border-radius: 8px; border: none;
    font-size: 16px;
}
#setupScreen input {
    padding: 8px; border-radius: 5px; border: 1px solid #ccc; font-size: 16px;
}

/* Estilos do Chat */
#chatBox {
    position:absolute; bottom:10px; right:10px; width:300px; height:200px;
    background:rgba(0,0,0,0.7); color:white; padding:10px; border-radius:10px;
    font-family:sans-serif; display:flex; flex-direction:column;
    z-index: 50;
    display: none; 
}
#messages {
    flex-grow: 1;
    overflow-y: auto;
    margin-bottom: 5px;
    padding-right: 5px; 
    font-size: 14px;
}
#chatBox input {
    padding: 8px; margin-right: 5px; flex-grow: 1; border-radius: 5px; border: none;
    box-sizing: border-box; 
    width: calc(100% - 80px); 
}
#chatBox button {
    padding: 8px 10px; border-radius: 5px; border: none; background:#4CAF50; color:white;
    cursor: pointer;
    width: 70px; 
    box-sizing: border-box;
}
#chatInput, #sendButton {
    margin-top: 5px;
    float: left; 
}
/* Estilo do botão de Fechar (X) */
#closeChat {
    position: absolute;
    top: 5px;
    right: 10px;
    background: none;
    color: white;
    border: none;
    font-size: 20px;
    font-weight: bold;
    cursor: pointer;
    padding: 0 5px;
}

/* Estilo do botão de Abrir */
#openChatButton {
    position: absolute;
    bottom: 10px;
    right: 10px;
    padding: 10px 15px;
    background: #007bff;
    color: white;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-weight: bold;
    z-index: 51;
}
</style>
</head>
<body>

<div id="setupScreen">
    <h2>Bem-vindo!</h2>
    <p style="font-size: 1.2em;">
        <label for="inputName">Seu Nome no Jogo:</label>
        <input type="text" id="inputName" value="Player" style="margin-left: 10px;">
    </p>
    <p style="font-size: 1.2em;">
        <label for="hostIdDisplay">ID do Host (Fixo):</label>
        <span id="hostIdDisplay" style="color:#00ff88; font-weight: bold;">main-room</span>
    </p>
    
    <div style="margin-top: 20px;">
        <button onclick="initGame(true)" style="background:#4CAF50; color:white;">
            INICIAR COMO HOST (main-room)
        </button>
        <br>
        <button onclick="initGame(false)" style="background:#2196F3; color:white;">
            INICIAR COMO CLIENTE
        </button>
    </div>
</div>

<div id="connectionInfo">
    <div id="myIdDisplay">Aguardando ID...</div>
    <div id="connectionStatus">Status: Desconectado</div>
    <input type="text" id="targetIdInput" placeholder="ID do Host para Conectar" value="main-room">
    <button onclick="connectToPeer()">Conectar</button>
</div>

<div id="playerList">Jogadores online:<br>0: Você</div>
<canvas id="game"></canvas>
<div id="joystick"><div id="stick"></div></div>

<button id="openChatButton" onclick="toggleChat(true)" style="display:none;">Abrir Chat</button>

<div id="chatBox">
    <button id="closeChat" onclick="toggleChat(false)">X</button>
    <div id="messages"></div>
    <input type="text" id="chatInput" placeholder="Digite sua mensagem..." disabled>
    <button onclick="sendMessage()" id="sendButton" disabled>Enviar</button>
</div>


<script>
// --- Configurações Iniciais ---
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");
canvas.width = innerWidth;
canvas.height = innerHeight;

const HOST_ID = "main-room";
const MESSAGE_DURATION_MS = 3000; // 3 segundos de exibição da mensagem no jogo

// Elementos da Interface
const setupScreen = document.getElementById("setupScreen");
const connectionInfoEl = document.getElementById("connectionInfo");
const myIdDisplay = document.getElementById("myIdDisplay");
const connectionStatus = document.getElementById("connectionStatus");
const targetIdInput = document.getElementById("targetIdInput");
const playerListEl = document.getElementById("playerList");

// Elementos do Chat
const chatBox = document.getElementById("chatBox");
const openChatButton = document.getElementById("openChatButton");
const chatInput = document.getElementById("chatInput");
const messagesEl = document.getElementById("messages");
const sendButton = document.getElementById("sendButton");


let localName = "";
let localId = "";
let players = {};
let isHost = false;

let peer;
let connection = null; 

// --- Carregar Nome Salvo ---
const inputNameEl = document.getElementById("inputName");
const savedName = localStorage.getItem('playerName');
if (savedName) {
    inputNameEl.value = savedName;
}
// --------------------------


// --- Joystick (Mantido) ---
const joystick = document.getElementById("joystick");
const stick = document.getElementById("stick");
let joy = { dx:0, dy:0 };

joystick.addEventListener("touchmove", e=>{
  const rect = joystick.getBoundingClientRect();
  const t = e.touches[0];
  const x = t.clientX - rect.left - rect.width/2;
  const y = t.clientY - rect.top - rect.height/2;
  const d = Math.sqrt(x*x+y*y), m=40;
  const a = Math.atan2(y,x);
  const mx = Math.cos(a)*Math.min(m,d), my = Math.sin(a)*Math.min(m,d);
  stick.style.left = 40+mx+"px";
  stick.style.top = 40+my+"px";
  joy.dx = mx/m; joy.dy = my/m;
});
joystick.addEventListener("touchend", ()=>{ stick.style.left="40px"; stick.style.top="40px"; joy.dx=joy.dy=0; });


// --- Funções de Interface ---

function updatePlayerList(){
  let html = "Jogadores online:<br>";
  let i = 1;
  for(const id in players){
    html += `${i}: ${players[id].name} (${id===localId?'Você':id})<br>`;
    i++;
  }
  playerListEl.innerHTML = html;
}

function updateConnectionInfo(id, status, isHostMode){
    myIdDisplay.innerHTML = `**Seu ID:** <span style="color:#00ff88;">${id}</span> ${isHostMode ? '(HOST)' : ''}`;
    connectionStatus.innerHTML = `**Status:** ${status}`;
}

// Mostra a mensagem no painel de chat (lateral)
function displayMessage(senderName, message, isLocal=false) {
    const msgDiv = document.createElement('div');
    
    let senderColor = 'white';
    if (senderName === localName && players[localId]) {
        senderColor = players[localId].color;
    } else if (connection && players[connection.peer]) {
        senderColor = players[connection.peer].color;
    } else if (senderName === "Sistema") {
        senderColor = '#00ffff'; 
    }
    
    msgDiv.innerHTML = `<span style="color:${senderColor}; font-weight:bold;">${senderName}:</span> ${message}`;
    messagesEl.appendChild(msgDiv);
    
    messagesEl.scrollTop = messagesEl.scrollHeight;
}

// Função para abrir/fechar o chat
function toggleChat(shouldOpen) {
    if (shouldOpen) {
        if (connection && connection.open) {
            chatBox.style.display = 'flex';
            openChatButton.style.display = 'none';
            chatInput.focus();
        } else {
            alert("Você precisa estar conectado a outro jogador para usar o chat!");
        }
    } else {
        chatBox.style.display = 'none';
        openChatButton.style.display = 'block';
    }
}


// --- Inicialização do Jogo ---

function initGame(shouldBeHost) {
    localName = inputNameEl.value || "Player";
    
    localStorage.setItem('playerName', localName);
    
    isHost = shouldBeHost;
    
    localId = isHost ? HOST_ID : Math.random().toString(36).substr(2, 9);
    
    players[localId] = { 
        id: localId, 
        x: canvas.width / 2, 
        y: canvas.height / 2, 
        color: '#' + Math.floor(Math.random() * 16777215).toString(16).padStart(6, '0'), 
        name: localName,
        // NOVO: Propriedades para exibir mensagem no jogo
        chatMsg: null,
        chatTime: 0
    };
    
    setupScreen.style.display = 'none';
    connectionInfoEl.style.display = 'block'; 
    openChatButton.style.display = 'block';
    
    startPeer();
}


// --- PeerJS / Conexão P2P (Simples) ---

function setupConnection(conn) {
    connection = conn;
    
    conn.on('data', data => handleData(data, conn));
    
    conn.on('close', () => {
        delete players[conn.peer]; 
        connection = null; 
        updatePlayerList();
        connectionStatus.innerHTML = `**Status:** Desconectado`;
        displayMessage("Sistema", `Conexão perdida com o outro jogador.`, false);
        chatInput.disabled = true;
        sendButton.disabled = true;
        toggleChat(false); 
    });
    
    chatInput.disabled = false;
    sendButton.disabled = false;
    displayMessage("Sistema", `Conectado com sucesso! Agora você pode usar o chat.`, true);
}

function startPeer(){
    peer = new Peer(localId); 
    updateConnectionInfo(localId, 'Conectando...', isHost);

    peer.on('open', id=>{
        updateConnectionInfo(id, isHost ? 'Aguardando Cliente' : 'ID Ativo', isHost);
        updatePlayerList();
        if(isHost) displayMessage("Sistema", "Aguardando um Cliente se conectar...", true);
    });

    peer.on('error', err => {
        connectionStatus.innerHTML = `**ERRO:** ${err.type}`;
        displayMessage("Sistema", `ERRO na conexão PeerJS: ${err.type}`, true);
    });

    peer.on('connection', conn=>{
        if (connection) {
            conn.close();
            return;
        }
        setupConnection(conn);
        connectionStatus.innerHTML = `**Status:** Conectado a ${conn.peer}`;
        conn.send({type:'update', data:players[localId]});
    });
}

function connectToPeer(){
    if (isHost) return alert("Você é o Host. Compartilhe seu ID.");
    if (connection && connection.open) return alert("Você já está conectado."); 

    const targetId = targetIdInput.value.trim();
    if(targetId === localId) return alert("Não pode conectar ao seu próprio ID.");
    
    connectionStatus.innerHTML = `Conectando a ${targetId}...`;
    displayMessage("Sistema", `Tentando conectar a ${targetId}...`, true);
    
    const conn = peer.connect(targetId);
    
    conn.on('open', ()=>{
        setupConnection(conn);
        connectionStatus.innerHTML = `**Status:** Conectado a ${targetId}`;
        conn.send({type:'join', data:players[localId]}); 
    });
    
    conn.on('error', err => {
        connectionStatus.innerHTML = `**Erro ao conectar** a ${targetId}`;
        displayMessage("Sistema", `ERRO ao conectar a ${targetId}`, true);
    });
}


// --- Lógica de Sincronização e Dados ---

function handleData(msg, conn=null){
  if(msg.type==='join'){
    // Lógica de Entrar/Reaparecer
    players[msg.data.id] = {...msg.data, chatMsg:null, chatTime:0}; // Inicializa propriedades de chat
    updatePlayerList();
    displayMessage("Sistema", `${msg.data.name} entrou no jogo!`, true);
    conn.send({type:'update', data:players[localId]});
    
  } else if(msg.type==='update'){
    // Quando recebe a posição do outro jogador, atualiza o chatTime/Msg também
    if(msg.data.id !== localId) {
        players[msg.data.id] = {...players[msg.data.id], ...msg.data};
    } else {
         players[msg.data.id] = msg.data;
    }
    updatePlayerList();
    
  } else if(msg.type==='chat'){
    // NOVO: Recebe mensagem de chat e a exibe no painel
    displayMessage(msg.data.name, msg.data.text, false);
    
    // NOVO: Define a mensagem para ser exibida em cima do personagem
    if(players[conn.peer]){
        players[conn.peer].chatMsg = msg.data.text;
        players[conn.peer].chatTime = Date.now() + MESSAGE_DURATION_MS;
    }
  }
}

// Envia a mensagem no painel E a dispara localmente para o balão de fala
function sendMessage(){
    const text = chatInput.value.trim();
    if (text === "" || !connection || !connection.open) return;
    
    // 1. Envia para o outro peer
    connection.send({
        type: 'chat', 
        data: { name: localName, text: text }
    });
    
    // 2. Exibe no painel localmente
    displayMessage(localName, text, true);
    
    // 3. NOVO: Dispara a exibição no balão de fala local
    players[localId].chatMsg = text;
    players[localId].chatTime = Date.now() + MESSAGE_DURATION_MS;
    
    // 4. Limpa o input
    chatInput.value = "";
    chatInput.focus(); 
}

chatInput.addEventListener('keypress', (e) => {
    if (e.key === 'Enter') {
        sendMessage();
    }
});


// --- Game Loop ---

function update(){
  if (!players[localId]) return; 
  
  // Verifica se a mensagem de chat do jogador local expirou
  if (players[localId].chatMsg && Date.now() > players[localId].chatTime) {
      players[localId].chatMsg = null;
      players[localId].chatTime = 0;
  }

  // Movimento
  players[localId].x += joy.dx*5;
  players[localId].y += joy.dy*5;
  
  if (connection && connection.open) {
      // É importante enviar o chatTime e chatMsg no update para que o outro jogador 
      // saiba se a mensagem dele expirou.
      connection.send({
          type:'update', 
          data: {
              id: players[localId].id,
              x: players[localId].x,
              y: players[localId].y,
              // Não envia a mensagem de chat aqui, pois ela é enviada no tipo 'chat'
              // Apenas a posição é sincronizada no 'update'
          }
      });
  }
}

function draw(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  if (!players[localId]) return; 
  
  const offX = players[localId].x - canvas.width/2;
  const offY = players[localId].y - canvas.height/2;

  // Grid
  const grid=100;
  ctx.strokeStyle="#95afc0";
  for(let x=-grid;x<canvas.width+grid;x+=grid){
    ctx.beginPath(); ctx.moveTo(x-offX%grid,0); ctx.lineTo(x-offX%grid,canvas.height); ctx.stroke();
  }
  for(let y=-grid;y<canvas.height+grid;y+=grid){
    ctx.beginPath(); ctx.moveTo(0,y-offY%grid); ctx.lineTo(canvas.width,y-offY%grid); ctx.stroke();
  }

  // Desenha os Players
  for(const id in players){
    const p = players[id];
    const sx = (id === localId) ? canvas.width/2 : p.x-offX;
    const sy = (id === localId) ? canvas.height/2 : p.y-offY;

    // Desenha o círculo do jogador
    ctx.fillStyle=p.color;
    ctx.beginPath(); ctx.arc(sx,sy,25,0,Math.PI*2); ctx.fill();
    
    // Desenha o Nome do jogador
    ctx.font="16px sans-serif";
    ctx.textAlign="center";
    ctx.textBaseline="bottom";
    ctx.fillStyle="white";
    ctx.fillText(p.name, sx, sy-30);
    ctx.strokeStyle="black";
    ctx.lineWidth=2;
    ctx.strokeText(p.name, sx, sy-30);
    
    // --- NOVO: Desenha a Mensagem de Chat (Balão de Fala) ---
    if(p.chatMsg && Date.now() < p.chatTime){
        const text = p.chatMsg;
        ctx.font="14px sans-serif";
        ctx.textAlign="center";
        
        // Mede o texto para o balão
        const metrics = ctx.measureText(text);
        const textWidth = metrics.width;
        const padding = 10;
        const boxWidth = textWidth + padding * 2;
        const boxHeight = 25;
        
        const boxX = sx - boxWidth / 2;
        const boxY = sy - 80; // Posição acima do nome
        
        // Desenha o balão (retângulo arredondado)
        ctx.fillStyle = "white";
        ctx.strokeStyle = "black";
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.roundRect(boxX, boxY, boxWidth, boxHeight, 5);
        ctx.fill();
        ctx.stroke();

        // Desenha o texto dentro do balão
        ctx.fillStyle = "black";
        ctx.fillText(text, sx, boxY + 17); // Ajuste fino para centralizar verticalmente
    }
    // ---------------------------------------------------------
  }
}

function loop(){ requestAnimationFrame(loop); update(); draw(); }
loop(); 
</script>

</body>
</html>
