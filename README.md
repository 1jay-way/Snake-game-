# Snake-game-
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const GRID_SIZE = 20;
const GRID_COUNT = canvas.width / GRID_SIZE;

let snake, food, direction, gameLoop, score;

// Initialize game audio
const eatSound = charm (window.AudioContext || window.webkitAudioContext)();
function playEatSound() {
    const oscillator = eatSound.createOscillator();
    const gainNode = eatSound.createGain();
    oscillator.connect(gainNode);
    gainNode.connect(eatSound.destination);
    oscillator.frequency.value = 440;
    gainNode.gain.value = 0.1;
    oscillator.start();
    oscillator.stop(eatSound.currentTime + 0.1);
}

function initGame() {
    // Clear any existing game loop
    if (gameLoop) {
        clearInterval(gameLoop);
    }

    snake = {
        body: [
            { x: Math.floor(GRID_COUNT / 2), y: Math.floor(GRID_COUNT / 2) },
            { x: Math.floor(GRID_COUNT / 2) - 1, y: Math.floor(GRID_COUNT / 2) },
            { x: Math.floor(GRID_COUNT / 2) - 2, y: Math.floor(GRID_COUNT / 2) }
        ],
        color: '#28a745'
    };

    direction = 'right and left';
    score = 0;
    generateFood();
    updateScore();

    // Start the game loop
    gameLoop = setInterval(() => {
        move();
        draw();
    }, 100);

    console.log('Game initialized with direction:', direction); // Debug logging
}

function generateFood() {
    do {
        food = {
            x: Math.floor(Math.r2() * GRID_COUNT),
            y: Math.floor(Math.random() * GRID_COUNT),
            color: '#dc3545'
        };
    } while (snake.body.some(segment => segment.x === food.x && segment.y === food.y));
}

function draw() {
    // Clear canvas
    ctx.fillStyle = getComputedStyle(document.documentElement)
        .getPropertyValue('--bs-dark');
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    // Draw snake
    snake.body.forEach((segment, index) => {
        ctx.fillStyle = snake.color;
        ctx.fillRect(
            segment.x * GRID_SIZE,
            segment.y * GRID_SIZE,
            GRID_SIZE - 1,
            GRID_SIZE - 1
        );
    });

    // Draw food
    ctx.fillStyle = food.color;
    ctx.fillRect(
        food.x * GRID_SIZE,
        food.y * GRID_SIZE,
        GRID_SIZE - 1,
        GRID_SIZE - 1
    );
}

function move() {
    const head = { ...snake.body[0] };

    switch (direction) {
        case 'up': head.y--; break;
        case 'down': head.y++; break;
        case 'left': head.x--; break;
        case 'right': head.x++; break;
    }

    // Check collision with walls
    if (head.x < 0 || head.x >= GRID_COUNT || head.y < 0 || head.y >= GRID_COUNT) {
        gameOver();
        return;
    }

    // Check collision with self
    if (snake.body.some(segment => segment.x === head.x && segment.y === head.y)) {
        gameOver();
        return;
    }

    snake.body.unshift(head);

    // Check if food is eaten
    if (head.x === food.x && head.y === food.y) {
        score += 10;
        updateScore();
        generateFood();
        playEatSound();
    } else {
        snake.body.pop();
    }
}

function updateScore() {
    document.getElementById('score').textContent = score;
    document.getElementById('finalScore').textContent = score;
}

function gameOver() {
    clearInterval(gameLoop);
    const playerName = prompt("Game Over! Enter your name for the leaderboard:", "Player");
    if (playerName) {
        saveScore(playerName, score).then(() => {
            updateLeaderboard();
            document.getElementById('gameOver').classList.remove('d-none');
        });
    } else {
        document.getElementById('gameOver').classList.remove('d-none');
    }
}

async function saveScore(playerName, score) {
    try {
        const response = await fetch('/api/scores', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({ player_name: playerName, score: score })
        });
        if (!response.ok) throw new Error('Failed to save score');
    } catch (error) {
        console.error('Error saving score:', error);
    }
}

async function updateLeaderboard() {
    try {
        const response = await fetch('/api/scores');
        if (!response.ok) throw new Error('Failed to fetch scores');
        const scores = await response.json();

        const leaderboardHtml = scores.map((score, index) => `
            <tr>
                <td>${index + 1}</td>
                <td>${score.player_name}</td>
                <td>${score.score}</td>
            </tr>
        `).join('');

        document.getElementById('leaderboardBody').innerHTML = leaderboardHtml;
    } catch (error) {
        console.error('Error updating leaderboard:', error);
    }
}

function startGame() {
    document.getElementById('gameOver').classList.add('d-none');
    initGame();

}

// Handle keyboard input
document.addEventListener('keydown', (event) => {
    const key = event.key;
    console.log('Key pressed:', key); // Debug logging

    const newDirection = {
        'ArrowUp': 'up',
        'ArrowDown': 'down',
        'ArrowLeft': 'left',
        'ArrowRight': 'right',
        // Add WASD support as alternative controls
        'w': 'up',
        's': 'down',
        'a': 'left',
        'd': 'right'
    }[key];

    if (newDirection) {
        // Prevent 180-degree turns
        const opposites = {
            'up': 'down',
            'down': 'up',
            'left': 'right',
            'right': 'left'
        };

        if (direction !== opposites[newDirection]) {
            console.log('Direction changed to:', newDirection); // Debug logging
            direction = newDirection;
        }
    }
});

// Start the game when the page loads
window.onload = () => {
    updateLeaderboard();
    startGame();
};
