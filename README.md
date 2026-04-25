# 🦖 Chrome Dino Game Debugger & Tweaker

A lightweight, draggable UI injected directly into the Chrome Dino game (`chrome://dino` or when offline) for testing, debugging, and hardware hacking. 

**Perfect for Arduino, PSoC, and Microcontroller Projects!** If you are building a physical Dino bot using optical sensors (LDRs/photoresistors) and servo motors, testing your hardware can be a nightmare. This script was specifically built for hardware hackers. It allows you to freeze the screen to calibrate ADC values, force the day/night transition, or lock the speed—all in real-time, without having to manually play the game for 5 minutes just to test a single edge case.

## Features

* **Force Night Mode**: Instantly switch the game to inverted colors to calibrate light sensors.
* **Night Distance**: Change the score required to trigger the automatic day/night cycle.
* **Speed & Acceleration Control**: Freeze the game speed or set it to a specific value to analyze hardware reaction times.
* **Invincibility**: Run right through obstacles so you can leave the game running indefinitely while plotting serial logs.
* **Obstacle Filtering**: Choose exactly what spawns (Small Cacti, Large Cacti, Pterodactyls).
* **Force Size**: Force the game to spawn obstacles of a specific cluster size (e.g., strictly 1 cactus at a time to calibrate jump timing).
* **Advanced Physics**: Modify Gravity and Jump Velocity to match your servo motor's mechanical limits.
* **Time Travel (Set Score)**: Teleport to a specific score (e.g., right before the night transition) to test edge cases.
* **Pause/Play**: Freeze the screen instantly to read your sensor's ADC values on a static frame.
* **Draggable UI**: Move the menu anywhere on the screen by dragging the top header.

## How to Use

1. Open Google Chrome and type `chrome://dino` in the address bar (or turn off your Wi-Fi).
2. Press **Space** to start the game.
3. Open the Developer Tools:
   * Windows / Linux: `F12` or `Ctrl + Shift + I`
   * Mac: `Cmd + Option + I`
4. Go to the **Console** tab.
5. Copy the entire script below, paste it into the console, and hit **Enter**.
   > **Note:** If Chrome blocks you from pasting with a security warning, manually type `allow pasting` into the console and press Enter first. Then, try pasting the script again.
6. The Debugger Menu will appear in the top right corner of the game!

> **Note:** To completely remove the menu and revert all changes, you can either click the **Reset All** button in the menu, or simply refresh the page (`F5`).

## The Script

```javascript
(function() {
    // Check if the game instance is running
    if (!Runner || !Runner.getInstance()) {
        console.error("Oops! Start the game first (press Space) before injecting the menu.");
        return;
    }

    const dinoGame = Runner.getInstance();
    
    // --- CAPTURE ORIGINAL VALUES FOR FULL RESET ---
    const originalInvert = dinoGame.invert.bind(dinoGame);
    const originalGameOver = dinoGame.gameOver.bind(dinoGame);
    const originalAcceleration = dinoGame.config.acceleration;
    const originalSpeed = dinoGame.config.speed;
    const originalInvertDistance = dinoGame.config.invertDistance || 700;
    const originalGravity = dinoGame.tRex.config.gravity || 0.6;
    const originalJumpVelocity = dinoGame.tRex.config.initialJumpVelocity || -10;

    // --- HELPER: PROTECT BUTTONS FROM DINO CSS ---
    function styleMiniButton(btn) {
        btn.style.all = 'revert'; 
        btn.style.background = '#444';
        btn.style.color = '#fff';
        btn.style.border = '1px solid #666';
        btn.style.borderRadius = '2px';
        btn.style.cursor = 'pointer';
        btn.style.fontFamily = 'monospace';
        btn.style.boxSizing = 'border-box';
        btn.style.margin = '0';
    }

    // --- OBSTACLE HOOK LOGIC ---
    const horizon = dinoGame.horizon;
    if (!horizon.originalAddNewObstacle) {
        horizon.originalAddNewObstacle = horizon.addNewObstacle;
    }

    const obsConfig = {
        cactusSmall: true,
        cactusLarge: true,
        pterodactyl: true,
        forceSizeActive: false,
        forcedSize: 1
    };

    horizon.addNewObstacle = function(currentSpeed) {
        const backupTypes = this.obstacleTypes;
        let allowedTypes = backupTypes.filter(o => obsConfig[o.type]);
        
        if (allowedTypes.length === 0) return;

        this.obstacleTypes = allowedTypes;
        this.originalAddNewObstacle.call(this, currentSpeed); 
        this.obstacleTypes = backupTypes; 

        if (obsConfig.forceSizeActive) {
            const newObs = this.obstacles[this.obstacles.length - 1];
            if (newObs && newObs.typeConfig.type.includes('cactus')) {
                let fSize = parseInt(obsConfig.forcedSize, 10);
                if (isNaN(fSize) || fSize < 1) fSize = 1;

                newObs.size = fSize;
                newObs.width = newObs.typeConfig.width * newObs.size;
                
                newObs.collisionBoxes = [];
                newObs.typeConfig.collisionBoxes.forEach(box => {
                    newObs.collisionBoxes.push({x: box.x, y: box.y, width: box.width, height: box.height});
                });

                if (newObs.size > 1 && newObs.collisionBoxes.length >= 3) {
                    newObs.collisionBoxes[1].width = newObs.width - newObs.collisionBoxes[0].width - newObs.collisionBoxes[2].width;
                    newObs.collisionBoxes[2].x = newObs.width - newObs.collisionBoxes[2].width;
                }
            }
        }
    };

    // --- UI CREATION ---
    const panel = document.createElement('div');
    panel.style.position = 'absolute';
    panel.style.top = '5px';
    panel.style.right = '5px';
    panel.style.zIndex = '9999';
    panel.style.background = 'rgba(20, 20, 20, 0.9)'; 
    panel.style.border = '1px solid #777';
    panel.style.padding = '5px';
    panel.style.borderRadius = '4px';
    panel.style.fontFamily = 'monospace';
    panel.style.color = '#ddd';
    panel.style.fontSize = '8px'; 
    panel.style.boxShadow = '1px 1px 5px rgba(0,0,0,0.5)';
    panel.style.width = '110px'; 
    panel.style.userSelect = 'none';

    const header = document.createElement('div');
    header.style.display = 'flex';
    header.style.justifyContent = 'space-between';
    header.style.alignItems = 'center';
    header.style.cursor = 'grab'; 
    header.style.paddingBottom = '3px';
    header.style.borderBottom = '1px solid #444';
    header.style.marginBottom = '3px';
    
    const title = document.createElement('strong');
    title.innerText = 'Dino Debug';
    
    const toggleBtn = document.createElement('button');
    toggleBtn.innerText = '−';
    styleMiniButton(toggleBtn);
    toggleBtn.style.fontSize = '8px';
    toggleBtn.style.padding = '0 4px';
    
    toggleBtn.addEventListener('mousedown', (e) => e.stopPropagation());

    header.appendChild(title);
    header.appendChild(toggleBtn);
    panel.appendChild(header);

    const content = document.createElement('div');
    panel.appendChild(content);

    let isMinimized = false;
    toggleBtn.addEventListener('click', () => {
        isMinimized = !isMinimized;
        content.style.display = isMinimized ? 'none' : 'block';
        toggleBtn.innerText = isMinimized ? '+' : '−';
    });

    // --- DRAG AND DROP LOGIC ---
    let pos1 = 0, pos2 = 0, pos3 = 0, pos4 = 0;
    
    header.onmousedown = dragMouseDown;

    function dragMouseDown(e) {
        e = e || window.event;
        e.preventDefault();
        pos3 = e.clientX;
        pos4 = e.clientY;
        document.onmouseup = closeDragElement;
        document.onmousemove = elementDrag;
        header.style.cursor = 'grabbing';
    }

    function elementDrag(e) {
        e = e || window.event;
        e.preventDefault();
        pos1 = pos3 - e.clientX;
        pos2 = pos4 - e.clientY;
        pos3 = e.clientX;
        pos4 = e.clientY;
        
        if (panel.style.right) {
            panel.style.left = (window.innerWidth - panel.offsetWidth - parseInt(panel.style.right)) + "px";
            panel.style.right = ''; 
        }
        
        panel.style.top = (panel.offsetTop - pos2) + "px";
        panel.style.left = (panel.offsetLeft - pos1) + "px";
    }

    function closeDragElement() {
        document.onmouseup = null;
        document.onmousemove = null;
        header.style.cursor = 'grab';
    }

    function createMenuCheckbox(labelText, onChangeCallback, isChecked = false) {
        const lbl = document.createElement('label');
        lbl.style.display = 'flex';
        lbl.style.alignItems = 'center';
        lbl.style.cursor = 'pointer';
        lbl.style.marginBottom = '4px';
        
        const chk = document.createElement('input');
        chk.type = 'checkbox';
        chk.checked = isChecked;
        chk.style.appearance = 'checkbox';
        chk.style.webkitAppearance = 'checkbox';
        chk.style.opacity = '1';
        chk.style.width = '10px';
        chk.style.height = '10px';
        chk.style.margin = '0 4px 0 0';
        
        lbl.appendChild(chk);
        lbl.appendChild(document.createTextNode(labelText));
        chk.addEventListener('change', onChangeCallback);
        
        return { label: lbl, checkbox: chk };
    }

    // --- MAIN OPTIONS ---
    const darkObj = createMenuCheckbox('Force Night', (e) => {
        if (e.target.checked) {
            dinoGame.invert = function() {};
            document.documentElement.classList.add('inverted');
            dinoGame.inverted = true;
        } else {
            dinoGame.invert = originalInvert;
            document.documentElement.classList.remove('inverted');
            dinoGame.inverted = false;
        }
    });
    content.appendChild(darkObj.label);

    const distLabel = document.createElement('label');
    distLabel.style.display = 'flex';
    distLabel.style.alignItems = 'center';
    distLabel.style.marginBottom = '4px';
    distLabel.innerText = 'Night Dist:';
    const distInput = document.createElement('input');
    distInput.type = 'number';
    distInput.value = originalInvertDistance;
    distInput.style.width = '35px';
    distInput.style.fontSize = '8px';
    distInput.style.padding = '1px';
    distInput.style.marginLeft = '4px';
    distLabel.appendChild(distInput);
    content.appendChild(distLabel);
    distInput.addEventListener('change', (e) => dinoGame.config.invertDistance = parseInt(e.target.value, 10));

    const speedLabel = document.createElement('label');
    speedLabel.style.display = 'flex';
    speedLabel.style.alignItems = 'center';
    speedLabel.style.marginBottom = '4px';
    speedLabel.innerText = 'Speed:     ';
    
    const speedInput = document.createElement('input');
    speedInput.type = 'number';
    speedInput.step = '0.5';
    speedInput.value = parseFloat(dinoGame.currentSpeed.toFixed(2));
    speedInput.style.width = '35px';
    speedInput.style.fontSize = '8px';
    speedInput.style.padding = '1px';
    speedInput.style.marginLeft = '4px';
    speedLabel.appendChild(speedInput);

    const speedSetBtn = document.createElement('button');
    speedSetBtn.innerText = 'Set';
    styleMiniButton(speedSetBtn);
    speedSetBtn.style.fontSize = '7px';
    speedSetBtn.style.padding = '1px 3px';
    speedSetBtn.style.marginLeft = '4px';
    speedLabel.appendChild(speedSetBtn);
    content.appendChild(speedLabel);

    const updateSpeed = () => {
        const val = parseFloat(speedInput.value);
        if (!isNaN(val)) {
            dinoGame.currentSpeed = val;
            dinoGame.config.speed = val; 
        }
    };
    speedInput.addEventListener('change', updateSpeed);
    speedSetBtn.addEventListener('click', updateSpeed);

    const invObj = createMenuCheckbox('Invincible', (e) => {
        dinoGame.gameOver = e.target.checked ? function() {} : originalGameOver; 
    });
    content.appendChild(invObj.label);

    const resetAllBtn = document.createElement('button');
    resetAllBtn.innerText = 'Reset All';
    styleMiniButton(resetAllBtn);
    resetAllBtn.style.fontSize = '8px';
    resetAllBtn.style.padding = '3px';
    resetAllBtn.style.width = '100%';
    resetAllBtn.style.marginBottom = '4px';
    resetAllBtn.style.display = 'block';
    content.appendChild(resetAllBtn);

    // --- OBSTACLES SECTION ---
    const obsHeader = document.createElement('div');
    obsHeader.style.marginTop = '6px';
    obsHeader.style.paddingTop = '4px';
    obsHeader.style.borderTop = '1px solid #555';
    obsHeader.style.cursor = 'pointer';
    obsHeader.style.color = '#aaa';
    obsHeader.innerHTML = 'Obstacles <span id="obs-arrow">▼</span>';
    content.appendChild(obsHeader);

    const obsContent = document.createElement('div');
    obsContent.style.display = 'none'; 
    obsContent.style.marginTop = '4px';
    obsContent.style.paddingLeft = '4px';
    obsContent.style.borderLeft = '1px solid #555';
    content.appendChild(obsContent);

    obsHeader.addEventListener('click', () => {
        const isHidden = obsContent.style.display === 'none';
        obsContent.style.display = isHidden ? 'block' : 'none';
        document.getElementById('obs-arrow').innerText = isHidden ? '▲' : '▼';
    });

    const cSmall = createMenuCheckbox('Small Cactus', (e) => obsConfig.cactusSmall = e.target.checked, true);
    const cLarge = createMenuCheckbox('Large Cactus', (e) => obsConfig.cactusLarge = e.target.checked, true);
    const cPtero = createMenuCheckbox('Pterodactyls', (e) => obsConfig.pterodactyl = e.target.checked, true);
    obsContent.appendChild(cSmall.label);
    obsContent.appendChild(cLarge.label);
    obsContent.appendChild(cPtero.label);
    
    const sep = document.createElement('div');
    sep.style.borderBottom = '1px dashed #555';
    sep.style.margin = '4px 0';
    obsContent.appendChild(sep);

    const forceSizeContainer = document.createElement('div');
    forceSizeContainer.style.display = 'flex';
    forceSizeContainer.style.alignItems = 'center';
    forceSizeContainer.style.marginBottom = '4px';

    const cSizeObj = createMenuCheckbox('Force Size:', (e) => obsConfig.forceSizeActive = e.target.checked, false);
    cSizeObj.label.style.marginBottom = '0'; 
    forceSizeContainer.appendChild(cSizeObj.label);

    const sizeInput = document.createElement('input');
    sizeInput.type = 'number';
    sizeInput.min = '1';
    sizeInput.value = '1';
    sizeInput.style.width = '30px';
    sizeInput.style.fontSize = '8px';
    sizeInput.style.padding = '1px';
    sizeInput.style.marginLeft = '4px';
    forceSizeContainer.appendChild(sizeInput);

    sizeInput.addEventListener('change', (e) => {
        let val = parseInt(e.target.value, 10);
        if (isNaN(val) || val < 1) val = 1;
        e.target.value = val;
        obsConfig.forcedSize = val;
    });

    obsContent.appendChild(forceSizeContainer);

    // --- ADVANCED SECTION ---
    const advHeader = document.createElement('div');
    advHeader.style.marginTop = '6px';
    advHeader.style.paddingTop = '4px';
    advHeader.style.borderTop = '1px solid #555';
    advHeader.style.cursor = 'pointer';
    advHeader.style.color = '#aaa';
    advHeader.innerHTML = 'Advanced <span id="adv-arrow">▼</span>';
    content.appendChild(advHeader);

    const advContent = document.createElement('div');
    advContent.style.display = 'none'; 
    advContent.style.marginTop = '4px';
    content.appendChild(advContent);

    advHeader.addEventListener('click', () => {
        const isHidden = advContent.style.display === 'none';
        advContent.style.display = isHidden ? 'block' : 'none';
        document.getElementById('adv-arrow').innerText = isHidden ? '▲' : '▼';
    });

    const pauseBtn = document.createElement('button');
    pauseBtn.innerText = 'Pause / Play';
    styleMiniButton(pauseBtn);
    pauseBtn.style.fontSize = '7px';
    pauseBtn.style.padding = '3px';
    pauseBtn.style.width = '100%';
    pauseBtn.style.marginBottom = '6px';
    pauseBtn.style.display = 'block';
    advContent.appendChild(pauseBtn);

    pauseBtn.addEventListener('click', () => dinoGame.paused || !dinoGame.playing ? dinoGame.play() : dinoGame.stop());

    const scoreDiv = document.createElement('div');
    scoreDiv.style.display = 'flex';
    scoreDiv.style.alignItems = 'center';
    scoreDiv.style.marginBottom = '4px';

    const scoreLabel = document.createElement('label');
    scoreLabel.innerText = 'Score:';
    
    const scoreInput2 = document.createElement('input');
    scoreInput2.type = 'number';
    scoreInput2.placeholder = '600';
    scoreInput2.style.width = '35px';
    scoreInput2.style.fontSize = '8px';
    scoreInput2.style.padding = '1px';
    scoreInput2.style.marginLeft = '4px';

    const scoreBtn = document.createElement('button');
    scoreBtn.innerText = 'Go';
    styleMiniButton(scoreBtn);
    scoreBtn.style.fontSize = '7px';
    scoreBtn.style.padding = '1px 3px';
    scoreBtn.style.marginLeft = '4px';

    scoreDiv.appendChild(scoreLabel);
    scoreDiv.appendChild(scoreInput2);
    scoreDiv.appendChild(scoreBtn);
    advContent.appendChild(scoreDiv);

    scoreBtn.addEventListener('click', () => {
        if (!dinoGame.playing) {
            alert("Please start running (press Space) BEFORE setting the score!");
            return;
        }
        const targetScore = parseInt(scoreInput2.value, 10);
        if (isNaN(targetScore)) return;

        dinoGame.distanceRan = targetScore / 0.025; 
        let newSpeed = dinoGame.config.speed + (targetScore / 100);
        if (newSpeed > dinoGame.config.maxSpeed) newSpeed = dinoGame.config.maxSpeed;
        dinoGame.currentSpeed = newSpeed;
        speedInput.value = parseFloat(newSpeed.toFixed(2)); 
        dinoGame.distanceMeter.update(0, Math.ceil(dinoGame.distanceRan));
    });

    const accLabel = document.createElement('label');
    accLabel.style.display = 'flex';
    accLabel.style.alignItems = 'center';
    accLabel.style.marginBottom = '4px';
    accLabel.innerText = 'Accel:';
    const accInput = document.createElement('input');
    accInput.type = 'number';
    accInput.step = '0.001';
    accInput.value = originalAcceleration;
    accInput.style.width = '35px';
    accInput.style.fontSize = '8px';
    accInput.style.padding = '1px';
    accInput.style.marginLeft = '4px';
    accLabel.appendChild(accInput);
    advContent.appendChild(accLabel);

    accInput.addEventListener('change', (e) => dinoGame.config.acceleration = parseFloat(e.target.value));

    const gravLabel = document.createElement('label');
    gravLabel.style.display = 'flex';
    gravLabel.style.alignItems = 'center';
    gravLabel.style.marginBottom = '4px';
    gravLabel.innerText = 'Gravity:';
    const gravInput = document.createElement('input');
    gravInput.type = 'number';
    gravInput.step = '0.1';
    gravInput.value = originalGravity;
    gravInput.style.width = '30px';
    gravInput.style.fontSize = '8px';
    gravInput.style.padding = '1px';
    gravInput.style.marginLeft = '4px';
    gravLabel.appendChild(gravInput);
    advContent.appendChild(gravLabel);

    gravInput.addEventListener('change', (e) => dinoGame.tRex.config.gravity = parseFloat(e.target.value));

    const jumpLabel = document.createElement('label');
    jumpLabel.style.display = 'flex';
    jumpLabel.style.alignItems = 'center';
    jumpLabel.style.marginBottom = '4px';
    jumpLabel.innerText = 'Jump:   ';
    const jumpInput = document.createElement('input');
    jumpInput.type = 'number';
    jumpInput.step = '1';
    jumpInput.value = Math.abs(originalJumpVelocity); 
    jumpInput.style.width = '30px';
    jumpInput.style.fontSize = '8px';
    jumpInput.style.padding = '1px';
    jumpInput.style.marginLeft = '4px';
    jumpLabel.appendChild(jumpInput);
    advContent.appendChild(jumpLabel);

    jumpInput.addEventListener('change', (e) => dinoGame.tRex.config.initialJumpVelocity = -Math.abs(parseFloat(e.target.value)));

    // --- RESET ALL LOGIC ---
    resetAllBtn.addEventListener('click', () => {
        darkObj.checkbox.checked = false;
        dinoGame.invert = originalInvert;
        document.documentElement.classList.remove('inverted');
        dinoGame.inverted = false;
        
        distInput.value = originalInvertDistance;
        dinoGame.config.invertDistance = originalInvertDistance;

        speedInput.value = originalSpeed;
        dinoGame.config.speed = originalSpeed;
        if (dinoGame.playing) dinoGame.currentSpeed = originalSpeed;

        invObj.checkbox.checked = false;
        dinoGame.gameOver = originalGameOver;

        accInput.value = originalAcceleration;
        dinoGame.config.acceleration = originalAcceleration;

        obsConfig.cactusSmall = true; cSmall.checkbox.checked = true;
        obsConfig.cactusLarge = true; cLarge.checkbox.checked = true;
        obsConfig.pterodactyl = true; cPtero.checkbox.checked = true;
        
        obsConfig.forceSizeActive = false; cSizeObj.checkbox.checked = false;
        obsConfig.forcedSize = 1; sizeInput.value = 1;

        gravInput.value = originalGravity;
        dinoGame.tRex.config.gravity = originalGravity;
        jumpInput.value = Math.abs(originalJumpVelocity);
        dinoGame.tRex.config.initialJumpVelocity = originalJumpVelocity;

        console.log("All Dino settings have been reset to default.");
    });

    document.body.appendChild(panel);
    console.log("Dino Debugger V10 successfully injected!");
})();
```

## Modding / Contributing

Feel free to fork this project and add more tweaks! The underlying Chrome Dino class is called `Runner`. You can poke around its variables (`Runner.getInstance()`) in the console to find new things to manipulate.
