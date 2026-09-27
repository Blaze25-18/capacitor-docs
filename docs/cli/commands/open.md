const express = require('express');
const path = require('path');
const app = express();
const http = require('http').createServer(app);
const io = require('socket.io')(http);
const mongoose = require('mongoose');
const User = require('./models/User');

app.use(express.static(path.join(__dirname, 'public')));

mongoose.connect(process.env.MONGODB_URI || 'mongodb://127.0.0.1:27017/yonorummypro')
  .then(() => console.log("🍃 MongoDB Connection Secured."))
  .catch(() => console.log("⚠️ Database offline. Running memory-cache backup maps."));

let globalActiveLobbies = {};

function verifyPoolRummyDeclaration(cardHandArray, systemWildJokerValue) {
    if (!cardHandArray || cardHandArray.length < 13) {
        return { isValidMatchShow: false, pointsPenaltyCalculated: 80, errorStringReason: "A declaration requires exactly 13 cards." };
    }
    let suitGroupings = { 'H': [], 'D': [], 'C': [], 'S': [] };
    let penaltyPoints = 0;

    cardHandArray.forEach(card => {
        if (card.value >= 10 || card.value === 1) penaltyPoints += 10;
        else penaltyPoints += Number(card.value);

        if (Number(card.value) !== Number(systemWildJokerValue)) {
            suitGroupings[card.suit].push(Number(card.value));
        }
    });

    let detectedPureSequences = 0;
    for (let suit in suitGroupings) {
        let sorted = suitGroupings[suit].sort((a, b) => a - b);
        let run = 1;
        for (let i = 0; i < sorted.length - 1; i++) {
            if (sorted[i+1] === sorted[i] + 1) {
                run++; if (run >= 3) detectedPureSequences++;
            } else if (sorted[i+1] !== sorted[i]) {
                run = 1;
            }
        }
    }

    if (detectedPureSequences < 1) {
        return { isValidMatchShow: false, pointsPenaltyCalculated: Math.min(penaltyPoints, 80), errorStringReason: "Missing mandatory Pure Sequence (built without Jokers)!" };
    }
    return { isValidMatchShow: true, pointsPenaltyCalculated: 0 };
}

function generateRandomShuffledDeck() {
    const suits = ['H', 'D', 'C', 'S'];
    let deck = [];
    for (let s of suits) {
        for (let v = 1; v <= 13; v++) deck.push({ suit: s, value: v });
    }
    for (let i = deck.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [deck[i], deck[j]] = [deck[j], deck[i]];
    }
    return deck;
}

app.get('/api/leaderboard', async (req, res) => {
    const data = await User.find().sort({ winsRecordCount: -1 }).limit(10);
    res.json(data);
});

io.on('connection', (socket) => {
    socket.on('registerPlayerHandshakeContext', async (profile) => {
        let userDoc = await User.findOne({ username: profile.username });
        if (!userDoc) userDoc = await User.create({ username: profile.username });

        const rId = profile.roomId || "Global_Match_Lobby";
        socket.join(rId);

        if (!globalActiveLobbies[rId]) {
            let deck = generateRandomShuffledDeck();
            globalActiveLobbies[rId] = {
                deckDeckPile: deck, discardOpenPile: [deck.pop()],
                wildJokerValue: Math.floor(Math.random() * 13) + 1,
                connectedPlayerNodes: {}, maxCapacity: profile.maxCapacity || 2
            };
        }

        let lobby = globalActiveLobbies[rId];
        let hand = []; for (let i = 0; i < 13; i++) hand.push(lobby.deckDeckPile.pop());

        lobby.connectedPlayerNodes[socket.id] = { dbRef: userDoc, liveHandArray: hand };

        socket.emit('syncClientStateHand', {
            hand: hand, wildJokerValue: lobby.wildJokerValue,
            winsScoreRecord: userDoc.winsRecordCount, poolPoints: userDoc.poolPointsScore,
            openDiscardCardInstance: lobby.discardOpenPile[lobby.discardOpenPile.length - 1]
        });
    });

    socket.on('commitDrawTurnRequest', (action) => {
        let lobby = globalActiveLobbies["Global_Match_Lobby"];
        if (!lobby || !lobby.connectedPlayerNodes[socket.id]) return;
        let p = lobby.connectedPlayerNodes[socket.id];
        let card = (action.drawSourceType === 'OPEN') ? lobby.discardOpenPile.pop() : lobby.deckDeckPile.pop();
        if (!card) card = generateRandomShuffledDeck().pop();
        p.liveHandArray.push(card);

        socket.emit('syncClientStateHand', {
            hand: p.liveHandArray, wildJokerValue: lobby.wildJokerValue,
            winsScoreRecord: p.dbRef.winsRecordCount, poolPoints: p.dbRef.poolPointsScore,
            openDiscardCardInstance: lobby.discardOpenPile[lobby.discardOpenPile.length - 1] || { suit: 'H', value: 1 }
        });
    });

    socket.on('commitDiscardTurnRequest', (data) => {
        let lobby = globalActiveLobbies["Global_Match_Lobby"];
        if (!lobby || !lobby.connectedPlayerNodes[socket.id]) return;
        let p = lobby.connectedPlayerNodes[socket.id];
        let card = p.liveHandArray.splice(data.targetHandIndexPosition, 1)[0];
        lobby.discardOpenPile.push(card);

        socket.emit('syncClientStateHand', {
            hand: p.liveHandArray, wildJokerValue: lobby.wildJokerValue,
            winsScoreRecord: p.dbRef.winsRecordCount, poolPoints: p.dbRef.poolPointsScore,
            openDiscardCardInstance: card
        });
    });

    socket.on('submitMeldDeclarationToServer', async () => {
        let lobby = globalActiveLobbies["Global_Match_Lobby"];
        if (!lobby || !lobby.connectedPlayerNodes[socket.id]) return;
        let p = lobby.connectedPlayerNodes[socket.id];
        let report = verifyPoolRummyDeclaration(p.liveHandArray, lobby.wildJokerValue);

        if (report.isValidMatchShow) p.dbRef.winsRecordCount += 1;
        else p.dbRef.poolPointsScore += report.pointsPenaltyCalculated;
        
        await p.dbRef.save();
        socket.emit('responseMeldReport', report);
    });
});

http.listen(process.env.PORT || 3000, () => console.log("🚀 Server Core Online"));

