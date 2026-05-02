const http = require('http');
const WebSocket = require('ws');
const fs = require('fs');
const path = require('path');

const server = http.createServer((req, res) => {
  let filePath = path.join(__dirname, req.url === '/' ? 'index.html' : req.url);
  const ext = path.extname(filePath);
  const contentTypes = { '.html': 'text/html', '.js': 'application/javascript', '.css': 'text/css' };
  fs.readFile(filePath, (err, data) => {
    if (err) { res.writeHead(404); res.end('Not found'); return; }
    res.writeHead(200, { 'Content-Type': contentTypes[ext] || 'text/plain' });
    res.end(data);
  });
});

const wss = new WebSocket.Server({ server });

const WORD_LIST = [
  'cat','dog','house','tree','sun','moon','car','boat','fish','bird',
  'flower','cloud','mountain','pizza','guitar','elephant','rainbow','castle',
  'rocket','crown','dragon','cactus','umbrella','bicycle','clock','candle',
  'diamond','ghost','heart','island','jellyfish','kite','lemon','mushroom',
  'noodles','octopus','penguin','queen','robot','snake','tiger','volcano',
  'watermelon','xylophone','zebra','airplane','balloon','camera','disco ball',
  'earthquake','football','glasses','hammer','igloo','jungle','keyboard',
  'lighthouse','microphone','newspaper','orange','parrot','quicksand','river',
  'sandwich','telescope','unicorn','vampire','windmill','youtube','zombie',
  'ice cream','fire truck','hot dog','birthday cake','swimming pool','roller coaster',
  'spider web','treasure chest','broken heart','shooting star'
];

let rooms = {};

function getWords() {
  const shuffled = [...WORD_LIST].sort(() => Math.random() - 0.5);
  return shuffled.slice(0, 3);
}

function broadcast(room, msg, excludeIndex = -1) {
  room.players.forEach((p, i) => {
    if (i !== excludeIndex && p.ws.readyState === WebSocket.OPEN) {
      p.ws.send(JSON.stringify(msg));
    }
  });
}

function sendTo(player, msg) {
  if (player.ws.readyState === WebSocket.OPEN) {
    player.ws.send(JSON.stringify(msg));
  }
}

function startTurn(room) {
  room.currentWord = null;
  room.guessed = false;
  room.drawingData = [];
  room.turnStartTime = null;
  room.wordChoices = getWords();
  room.choosingWord = true;

  const drawer = room.players[room.drawerIndex];
  const guesser = room.players[1 - room.drawerIndex];

  // Send word choices to drawer
  sendTo(drawer, {
    type: 'chooseWord',
    words: room.wordChoices,
    round: room.round,
    totalRounds: room.totalRounds
  });

  // Tell guesser to wait
  sendTo(guesser, {
    type: 'waitingForWord',
    drawerName: drawer.name,
    round: room.round,
    totalRounds: room.totalRounds
  });
}

function startDrawing(room) {
  room.choosingWord = false;
  room.turnStartTime = Date.now();
  room.timeLimit = 80; // seconds

  const drawer = room.players[room.drawerIndex];
  const guesser = room.players[1 - room.drawerIndex];

  sendTo(drawer, {
    type: 'drawingStart',
    word: room.currentWord,
    role: 'drawer',
    timeLimit: room.timeLimit
  });

  // Send word as blanks to guesser
  const hint = room.currentWord.split('').map(c => c === ' ' ? ' ' : '_').join('');
  sendTo(guesser, {
    type: 'drawingStart',
    hint,
    wordLength: room.currentWord.length,
    role: 'guesser',
    timeLimit: room.timeLimit
  });

  // Timer
  room.timer = setTimeout(() => {
    if (!room.guessed) {
      broadcast(room, {
        type: 'turnEnd',
        word: room.currentWord,
        guessed: false,
        scores: room.scores
      });
      nextTurn(room);
    }
  }, room.timeLimit * 1000);
}

function nextTurn(room) {
  clearTimeout(room.timer);
  room.drawerIndex = 1 - room.drawerIndex;
  room.round++;

  if (room.round > room.totalRounds * 2) {
    // Game over
    const p0 = room.players[0];
    const p1 = room.players[1];
    const winner = room.scores[0] > room.scores[1] ? 0 : room.scores[1] > room.scores[0] ? 1 : -1;
    broadcast(room, {
      type: 'gameOver',
      scores: room.scores,
      players: room.players.map(p => p.name),
      winner
    });
    return;
  }

  setTimeout(() => startTurn(room), 3000);
}

wss.on('connection', (ws) => {
  let playerRoom = null;
  let playerIndex = null;

  ws.on('message', (raw) => {
    let msg;
    try { msg = JSON.parse(raw); } catch { return; }

    if (msg.type === 'join') {
      const roomId = msg.roomId.toUpperCase();
      if (!rooms[roomId]) {
        rooms[roomId] = {
          players: [], scores: [0, 0], drawerIndex: 0,
          round: 1, totalRounds: 3,
          currentWord: null, guessed: false,
          drawingData: [], timer: null,
          choosingWord: false, wordChoices: []
        };
      }
      const room = rooms[roomId];
      if (room.players.length >= 2) {
        ws.send(JSON.stringify({ type: 'error', message: 'Room is full!' }));
        return;
      }
      playerRoom = roomId;
      playerIndex = room.players.length;
      room.players.push({ ws, name: msg.name });
      room.scores[playerIndex] = 0;

      ws.send(JSON.stringify({ type: 'joined', index: playerIndex, roomId }));

      if (room.players.length === 2) {
        broadcast(room, { type: 'gameStart', players: room.players.map(p => p.name) });
        setTimeout(() => startTurn(room), 1500);
      } else {
        ws.send(JSON.stringify({ type: 'waiting', message: 'Waiting for your person...' }));
      }
    }

    if (msg.type === 'wordChosen') {
      const room = rooms[playerRoom];
      if (!room || playerIndex !== room.drawerIndex) return;
      room.currentWord = msg.word;
      startDrawing(room);
    }

    if (msg.type === 'draw') {
      const room = rooms[playerRoom];
      if (!room) return;
      room.drawingData.push(msg.data);
      // Forward drawing to other player
      const other = room.players[1 - playerIndex];
      sendTo(other, { type: 'draw', data: msg.data });
    }

    if (msg.type === 'clearCanvas') {
      const room = rooms[playerRoom];
      if (!room) return;
      room.drawingData = [];
      broadcast(room, { type: 'clearCanvas' }, playerIndex);
    }

    if (msg.type === 'guess') {
      const room = rooms[playerRoom];
      if (!room || room.guessed) return;
      if (playerIndex === room.drawerIndex) return; // drawer can't guess

      const guess = msg.guess.trim().toLowerCase();
      const word = room.currentWord.toLowerCase();

      // Broadcast the guess to both (so drawer can see)
      broadcast(room, {
        type: 'guessMessage',
        name: room.players[playerIndex].name,
        guess: msg.guess,
        correct: guess === word
      });

      if (guess === word) {
        room.guessed = true;
        clearTimeout(room.timer);

        // Points based on time remaining
        const elapsed = (Date.now() - room.turnStartTime) / 1000;
        const timeLeft = Math.max(0, room.timeLimit - elapsed);
        const points = Math.round(100 + (timeLeft / room.timeLimit) * 200);

        room.scores[playerIndex] += points; // guesser gets points
        room.scores[room.drawerIndex] += 50; // drawer gets some too

        broadcast(room, {
          type: 'turnEnd',
          word: room.currentWord,
          guessed: true,
          scores: room.scores,
          pointsGained: points
        });

        nextTurn(room);
      }
    }

    if (msg.type === 'rematch') {
      const room = rooms[playerRoom];
      if (!room) return;
      if (!room.rematchVotes) room.rematchVotes = new Set();
      room.rematchVotes.add(playerIndex);
      if (room.rematchVotes.size === 2) {
        room.scores = [0, 0];
        room.round = 1;
        room.drawerIndex = 0;
        room.guessed = false;
        room.drawingData = [];
        room.rematchVotes = new Set();
        broadcast(room, { type: 'gameStart', players: room.players.map(p => p.name) });
        setTimeout(() => startTurn(room), 1500);
      } else {
        broadcast(room, { type: 'rematchWaiting', name: room.players[playerIndex].name });
      }
    }
  });

  ws.on('close', () => {
    if (playerRoom && rooms[playerRoom]) {
      clearTimeout(rooms[playerRoom].timer);
      broadcast(rooms[playerRoom], { type: 'playerLeft' });
      delete rooms[playerRoom];
    }
  });
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => console.log(`Ermdraw running on port ${PORT}`));
