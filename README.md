<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Pixel Billionaire</title>
  <style>
    body { font-family: sans-serif; text-align: center; background: #111; color: #fff; }
    #grid { display: grid; grid-template-columns: repeat(75, 8px); gap: 1px; justify-content: center; margin: 20px auto; }
    .pixel { width: 8px; height: 8px; background: #333; cursor: pointer; }
    .bought { background: #0f0 !important; }
    #money, #comboCounter { font-size: 24px; margin: 10px; }
    #powerups { margin: 20px auto; max-width: 600px; display: flex; flex-wrap: wrap; justify-content: center; gap: 10px; }
    button { padding: 5px 10px; margin: 5px; cursor: pointer; background-color: green; color: white; border: none; border-radius: 4px; }
    #endgameModal, #adminPanel, #codeInputPanel { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.8); display: none; align-items: center; justify-content: center; z-index: 9999; }
    .modal-content { background: white; padding: 2rem; text-align: center; border-radius: 1rem; color: black; }
    .powerup-btn { transition: transform 0.2s ease; }
    .powerup-btn:hover { transform: scale(1.1); }
  </style>
</head>
<body>
  <h1>Pixel Billionaire</h1>
  <div id="money">Money: $100</div>
  <div id="comboCounter">Combo: 0</div>
  <button onclick="exportSave()">Export Save</button>
  <button onclick="importSave()">Import Save</button>
  <button onclick="document.getElementById('codeInputPanel').style.display = 'flex'">Enter Code</button>
  <div id="grid"></div>
  <div id="powerups"></div>

  <div id="endgameModal">
    <div class="modal-content">
      <h2>🎉 You Win! 🎉</h2>
      <p>You've become a Pixel Billionaire!</p>
      <button onclick="location.reload()">Play Again</button>
    </div>
  </div>

  <div id="codeInputPanel">
    <div class="modal-content">
      <h3>Enter Secret Code</h3>
      <input type="text" id="secretCodeInput" />
      <button onclick="checkCode()">Submit</button>
    </div>
  </div>

  <div id="adminPanel">
    <div class="modal-content">
      <h3>Admin Panel</h3>
      <input type="number" id="adminMoneyInput" placeholder="Money Amount" />
      <button onclick="addMoney(parseInt(document.getElementById('adminMoneyInput').value))">Add Money</button>
      <input type="number" id="adminPixelsInput" placeholder="Pixels Amount" />
      <button onclick="buyPixels(parseInt(document.getElementById('adminPixelsInput').value))">Add Pixels</button>
      <button onclick="resetGame()">Reset Game</button>
      <button onclick="document.getElementById('adminPanel').style.display = 'none'">Close</button>
    </div>
  </div>

  <script>
    let balance = 100;
    let totalEarned = 0;
    let pixelsBought = [];
    let gridSize = 75 * 75;
    let incomePerPixel = 1;
    let multiplier = 1;
    let combo = 0;
    let lastClickTime = 0;
    let comboTimer;
    let tickInterval;
    let activePowerups = [];

    const grid = document.getElementById("grid");
    const moneyDisplay = document.getElementById("money");
    const comboDisplay = document.getElementById("comboCounter");
    const powerupsContainer = document.getElementById("powerups");

    const powerups = [
      { name: "Double Income (60s)", cost: 1000, action: () => activateMultiplier(2, 60000) },
      { name: "Mega Surge (5s)", cost: 5000, action: () => activateMultiplier(10, 5000) },
      { name: "Passive Bonus", cost: 2000, action: () => incomePerPixel += 0.05 },
      { name: "Auto Buyer (30s)", cost: 1500, action: startAutoBuyer },
      { name: "Fast Forward (15s)", cost: 3000, action: () => activateFastForward(3, 15000) },
      { name: "Smart Collector (30s)", cost: 2500, action: startSmartCollector },
      { name: "Loan Shark", cost: 0, action: () => { balance += 10000; setTimeout(() => balance -= 12000, 60000); } },
      { name: "Mass Buy (50 Pixels)", cost: 10000, action: () => buyPixels(50) },
      { name: "Stock Boost", cost: 3000, action: () => activateMultiplier(pixelsBought.length / 100, 10000) },
      { name: "Color Explosion", cost: 500, action: () => {
        document.querySelectorAll(".bought").forEach(p => {
          p.style.background = `hsl(${Math.random() * 360}, 100%, 50%)`;
        });
      } },
      { name: "Multiplier Stack", cost: 5000, action: () => {} },
      { name: "Save Booster", cost: 10000, action: () => { offlineMultiplier = 1.0; } },
    ];

    function renderPowerups() {
      powerupsContainer.innerHTML = "";
      powerups.forEach((p, idx) => {
        const btn = document.createElement("button");
        btn.innerText = `${p.name} ($${p.cost})`;
        btn.className = "powerup-btn";
        btn.onclick = () => {
          if (balance >= p.cost) {
            balance -= p.cost;
            p.action();
            updateMoneyDisplay();
          }
        };
        powerupsContainer.appendChild(btn);
      });
    }

    function updateMoneyDisplay() {
      moneyDisplay.innerText = `Money: $${Math.floor(balance)}`;
    }

    function updateComboDisplay() {
      comboDisplay.innerText = `Combo: ${combo}`;
    }

    function createGrid() {
      for (let i = 0; i < gridSize; i++) {
        const pixel = document.createElement("div");
        pixel.className = "pixel";
        pixel.dataset.index = i;
        pixel.onclick = () => buyPixel(i, pixel);
        grid.appendChild(pixel);
      }
    }

    function buyPixel(index, pixelElement) {
      const cost = 100;
      if (pixelsBought.includes(index) || balance < cost) return;
      balance -= cost;
      pixelsBought.push(index);
      pixelElement.classList.add("bought");
      updateMoneyDisplay();

      const now = Date.now();
      if (now - lastClickTime < 500) {
        combo++;
        if (combo % 10 === 0) balance += 50;
      } else {
        combo = 1;
      }
      lastClickTime = now;
      updateComboDisplay();

      clearTimeout(comboTimer);
      comboTimer = setTimeout(() => {
        combo = 0;
        updateComboDisplay();
      }, 1500);
    }

    function incomeTick() {
      const income = pixelsBought.length * incomePerPixel * multiplier;
      balance += income;
      totalEarned += income;
      updateMoneyDisplay();
      if (balance >= 1000000000 && !window.endGameShown) {
        window.endGameShown = true;
        clearInterval(tickInterval);
        document.getElementById("endgameModal").style.display = "flex";
      }
    }

    function activateMultiplier(mult, duration) {
      const original = multiplier;
      multiplier *= mult;
      setTimeout(() => multiplier = original, duration);
    }

    function activateFastForward(factor, duration) {
      clearInterval(tickInterval);
      tickInterval = setInterval(incomeTick, 1000 / factor);
      setTimeout(() => {
        clearInterval(tickInterval);
        tickInterval = setInterval(incomeTick, 1000);
      }, duration);
    }

    function startAutoBuyer() {
      const interval = setInterval(() => {
        const pixel = document.querySelector(".pixel:not(.bought)");
        if (pixel && balance >= 100) buyPixel(parseInt(pixel.dataset.index), pixel);
      }, 1000);
      setTimeout(() => clearInterval(interval), 30000);
    }

    function startSmartCollector() {
      const interval = setInterval(() => {
        const pixel = document.querySelector(".pixel:not(.bought)");
        if (pixel && balance >= 100) buyPixel(parseInt(pixel.dataset.index), pixel);
      }, 1000);
      setTimeout(() => clearInterval(interval), 30000);
    }

    function exportSave() {
      const save = { balance, pixelsBought, totalEarned };
      const blob = new Blob([JSON.stringify(save)], { type: 'application/json' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = "pixel-save.json";
      a.click();
      URL.revokeObjectURL(url);
    }

    function importSave() {
      const input = document.createElement("input");
      input.type = "file";
      input.accept = ".json";
      input.onchange = e => {
        const file = e.target.files[0];
        const reader = new FileReader();
        reader.onload = event => {
          const save = JSON.parse(event.target.result);
          balance = save.balance;
          pixelsBought = save.pixelsBought || [];
          totalEarned = save.totalEarned || 0;
          document.querySelectorAll(".pixel").forEach(p => {
            if (pixelsBought.includes(Number(p.dataset.index))) {
              p.classList.add("bought");
            }
          });
          updateMoneyDisplay();
        };
        reader.readAsText(file);
      };
      input.click();
    }

    function checkCode() {
      const code = document.getElementById("secretCodeInput").value;
      if (code === "12345678") {
        document.getElementById("adminPanel").style.display = "flex";
      }
      document.getElementById("codeInputPanel").style.display = "none";
    }

    function addMoney(amount) {
      if (!isNaN(amount)) {
        balance += amount;
        updateMoneyDisplay();
      }
    }

    function buyPixels(amount) {
      let added = 0;
      document.querySelectorAll(".pixel").forEach((p, idx) => {
        if (!pixelsBought.includes(idx) && added < amount) {
          pixelsBought.push(idx);
          p.classList.add("bought");
          added++;
        }
      });
    }

    function resetGame() {
      localStorage.clear();
      location.reload();
    }

    function init() {
      createGrid();
      renderPowerups();
      const lastPlayed = localStorage.getItem("lastPlayed");
      if (lastPlayed) {
        const secondsAway = Math.floor((Date.now() - parseInt(lastPlayed)) / 1000);
        const offlineIncome = secondsAway * pixelsBought.length * incomePerPixel * 0.5;
        if (offlineIncome > 0) {
          balance += offlineIncome;
          totalEarned += offlineIncome;
          alert(`While you were away, you earned $${Math.floor(offlineIncome)}!`);
        }
      }
      updateMoneyDisplay();
      updateComboDisplay();
      tickInterval = setInterval(incomeTick, 1000);
    }

    window.onload = init;
    window.addEventListener("beforeunload", () => {
      localStorage.setItem("lastPlayed", Date.now());
    });
  </script>
</body>
</html>
