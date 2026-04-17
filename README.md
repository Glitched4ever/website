<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Fluid Sim - Fixed & Final</title>
    <style>
        :root {
            --bg-color: #121212;
            --panel-bg: rgba(20, 20, 20, 0.95);
            --border: #444;
            --accent: #00ffcc;
            --text: #ddd;
        }

        body {
            margin: 0;
            background-color: var(--bg-color);
            color: var(--text);
            font-family: 'Segoe UI', sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            overflow: hidden;
            user-select: none;
        }

        #container {
            position: relative;
            width: 512px;
            height: 512px;
            background: #000;
            box-shadow: 0 0 60px rgba(0,0,0,0.8);
            border: 1px solid #333;
        }

        canvas {
            display: block;
            width: 100%;
            height: 100%;
            cursor: crosshair;
        }

        /* --- Floating Menu --- */
        #menu {
            position: absolute;
            top: 10px;
            right: 10px;
            width: 190px;
            background: var(--panel-bg);
            border: 1px solid var(--border);
            border-radius: 8px;
            padding: 12px;
            display: flex;
            flex-direction: column;
            gap: 12px;
            z-index: 10;
        }

        .group {
            display: flex;
            flex-direction: column;
            gap: 8px;
            border-bottom: 1px solid #444;
            padding-bottom: 12px;
        }
        .group:last-child { border-bottom: none; padding-bottom: 0; }

        h4 { margin: 0 0 4px 0; font-size: 11px; text-transform: uppercase; color: #888; letter-spacing: 1px; }

        button.btn {
            background: #2a2a2a;
            color: #fff;
            border: 1px solid #555;
            padding: 6px 10px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 12px;
            transition: 0.2s;
            flex: 1;
        }
        button.btn:hover { background: #3a3a3a; border-color: #777; }
        button.btn.active {
            background: var(--accent);
            color: #000;
            border-color: var(--accent);
            font-weight: bold;
        }

        .row { display: flex; gap: 6px; }

        /* Swatches */
        .palette { display: flex; gap: 5px; flex-wrap: wrap; }
        .swatch {
            width: 20px; height: 20px;
            border-radius: 3px;
            border: 1px solid #555;
            cursor: pointer;
        }
        .swatch:hover { transform: scale(1.1); border-color: #fff; }

        /* Custom Picker */
        .picker-wrap {
            position: relative; height: 25px; border: 1px solid #555;
            border-radius: 4px; overflow: hidden;
        }
        input[type="color"] {
            position: absolute; top: -50%; left: -50%; width: 200%; height: 200%;
            cursor: pointer; border: none; padding: 0;
        }
        .picker-label {
            position: absolute; width: 100%; height: 100%;
            pointer-events: none; display: flex; align-items: center; justify-content: center;
            font-size: 11px; color: white; background: rgba(0,0,0,0.3); text-shadow: 0 1px 2px black;
        }

        /* Slider */
        input[type=range] { width: 100%; cursor: pointer; }
        
        label.check-label {
            display: flex; align-items: center; gap: 8px; font-size: 12px; cursor: pointer;
        }
    </style>
</head>
<body>

    <div id="container">
        <canvas id="fluidCanvas" width="512" height="512"></canvas>
        
        <div id="menu">
            <div class="group">
                <h4>Tool</h4>
                <div class="row">
                    <button id="btnDraw" class="btn active">Draw</button>
                    <button id="btnErase" class="btn">Erase</button>
                </div>
            </div>

            <div class="group">
                <h4>Ink Lifespan</h4>
                <input type="range" id="sldLife" min="0.9" max="1.0" step="0.001" value="0.995">
            </div>

            <div class="group">
                <h4>Color</h4>
                <label class="check-label">
                    <input type="checkbox" id="chkRainbow" checked> Rainbow Flow
                </label>
                <div class="palette">
                    <div class="swatch" style="background:#ff3333" data-c="#ff3333"></div>
                    <div class="swatch" style="background:#ffaa00" data-c="#ffaa00"></div>
                    <div class="swatch" style="background:#ffff00" data-c="#ffff00"></div>
                    <div class="swatch" style="background:#00ff00" data-c="#00ff00"></div>
                    <div class="swatch" style="background:#00ccff" data-c="#00ccff"></div>
                    <div class="swatch" style="background:#8800ff" data-c="#8800ff"></div>
                    <div class="swatch" style="background:#ffffff" data-c="#ffffff"></div>
                </div>
                <div class="picker-wrap">
                    <input type="color" id="inpColor" value="#00ccff">
                    <div class="picker-label">Custom Color</div>
                </div>
            </div>

            <div class="group">
                <div class="row">
                    <button id="btnPause" class="btn">Pause</button>
                    <button id="btnClear" class="btn">Clear</button>
                </div>
            </div>
        </div>
    </div>

<script>
    const canvas = document.getElementById('fluidCanvas');
    const ctx = canvas.getContext('2d');

    // --- Configuration ---
    const N = 160; // Slightly lower res for absolute stability
    const size = (N + 2) * (N + 2);
    const dt = 0.1;
    let fadeRate = 0.995;
    
    // State
    let paused = false;
    let isMouseDown = false;
    let currentTool = 'draw'; 
    let lastX = 0, lastY = 0;

    // Color State
    let rainbowMode = true;
    let hue = 0;
    let renderColor = { r: 0, g: 0, b: 0 }; 
    let manualColor = { r: 0, g: 204, b: 255 }; 

    // Fluid Arrays
    let u = new Float32Array(size); // Velocity X
    let v = new Float32Array(size); // Velocity Y
    let u_prev = new Float32Array(size);
    let v_prev = new Float32Array(size);
    let dens = new Float32Array(size); // Density
    let dens_prev = new Float32Array(size);

    // --- Core Fluid Mathematics ---
    function IX(x, y) { return x + (N + 2) * y; }

    function set_bnd(b, x) {
        for (let i = 1; i <= N; i++) {
            x[IX(0, i)] = b === 1 ? -x[IX(1, i)] : x[IX(1, i)];
            x[IX(N + 1, i)] = b === 1 ? -x[IX(N, i)] : x[IX(N, i)];
            x[IX(i, 0)] = b === 2 ? -x[IX(i, 1)] : x[IX(i, 1)];
            x[IX(i, N + 1)] = b === 2 ? -x[IX(i, N)] : x[IX(i, N)];
        }
        x[IX(0, 0)] = 0.5 * (x[IX(1, 0)] + x[IX(0, 1)]);
        x[IX(0, N + 1)] = 0.5 * (x[IX(1, N + 1)] + x[IX(0, N)]);
        x[IX(N + 1, 0)] = 0.5 * (x[IX(N, 0)] + x[IX(N + 1, 1)]);
        x[IX(N + 1, N + 1)] = 0.5 * (x[IX(N, N + 1)] + x[IX(N + 1, N)]);
    }

    function lin_solve(b, x, x0, a, c) {
        let cRecip = 1.0 / c;
        for (let k = 0; k < 4; k++) { 
            for (let j = 1; j <= N; j++) {
                for (let i = 1; i <= N; i++) {
                    x[IX(i, j)] = (x0[IX(i, j)] + a * (x[IX(i+1, j)] + x[IX(i-1, j)] + x[IX(i, j+1)] + x[IX(i, j-1)])) * cRecip;
                }
            }
            set_bnd(b, x);
        }
    }

    function project(u, v, p, div) {
        for (let j = 1; j <= N; j++) {
            for (let i = 1; i <= N; i++) {
                div[IX(i, j)] = (-0.5 * (u[IX(i+1, j)] - u[IX(i-1, j)] + v[IX(i, j+1)] - v[IX(i, j-1)])) / N;
                p[IX(i, j)] = 0;
            }
        }
        set_bnd(0, div); set_bnd(0, p);
        lin_solve(0, p, div, 1, 4);
        for (let j = 1; j <= N; j++) {
            for (let i = 1; i <= N; i++) {
                u[IX(i, j)] -= 0.5 * (p[IX(i+1, j)] - p[IX(i-1, j)]) * N;
                v[IX(i, j)] -= 0.5 * (p[IX(i, j+1)] - p[IX(i, j-1)]) * N;
            }
        }
        set_bnd(1, u); set_bnd(2, v);
    }

    function advect(b, d, d0, u, v, dt) {
        let i0, i1, j0, j1;
        let x, y, s0, t0, s1, t1, dt0 = dt * (N - 2);
        for (let j = 1; j <= N; j++) {
            for (let i = 1; i <= N; i++) {
                x = i - dt0 * u[IX(i, j)];
                y = j - dt0 * v[IX(i, j)];
                if (x < 0.5) x = 0.5; if (x > N + 0.5) x = N + 0.5;
                i0 = Math.floor(x); i1 = i0 + 1;
                if (y < 0.5) y = 0.5; if (y > N + 0.5) y = N + 0.5;
                j0 = Math.floor(y); j1 = j0 + 1;
                s1 = x - i0; s0 = 1.0 - s1; t1 = y - j0; t0 = 1.0 - t1;
                d[IX(i, j)] = s0 * (t0 * d0[IX(i0, j0)] + t1 * d0[IX(i0, j1)]) + s1 * (t0 * d0[IX(i1, j0)] + t1 * d0[IX(i1, j1)]);
            }
        }
        set_bnd(b, d);
    }

    // --- Main Logic ---

    function physicsStep() {
        if (paused) return;

        // 1. Velocity Step
        // Use current velocity (u) as source for previous (u_prev)
        u_prev.set(u); 
        v_prev.set(v);
        
        // Viscosity (Diffusion) - skipped for performance/crispness
        // lin_solve(1, u, u_prev, visc, 1+6*visc); ...
        
        project(u, v, u_prev, v_prev); // Calculate pressure
        
        // Use calculated pressure-corrected velocity to advect itself
        u_prev.set(u);
        v_prev.set(v);
        advect(1, u, u_prev, u_prev, v_prev, dt);
        advect(2, v, v_prev, u_prev, v_prev, dt);
        
        project(u, v, u_prev, v_prev); // Ensure mass conservation

        // 2. Density Step
        // THIS WAS THE BUG FIX: We must copy current Density to Prev before advecting
        dens_prev.set(dens);
        advect(0, dens, dens_prev, u, v, dt);

        // 3. Fade
        for(let i=0; i<size; i++) dens[i] *= fadeRate;
    }

    function updateColors() {
        if (rainbowMode) {
            if (!paused) hue = (hue + 1) % 360;
            let s = 100, l = 60;
            let c = (1 - Math.abs(2 * l / 100 - 1)) * s / 100;
            let x = c * (1 - Math.abs((hue / 60) % 2 - 1));
            let m = l / 100 - c / 2;
            let r=0, g=0, b=0;
            if (0 <= hue && hue < 60) { r=c; g=x; b=0; }
            else if (60 <= hue && hue < 120) { r=x; g=c; b=0; }
            else if (120 <= hue && hue < 180) { r=0; g=c; b=x; }
            else if (180 <= hue && hue < 240) { r=0; g=x; b=c; }
            else if (240 <= hue && hue < 300) { r=x; g=0; b=c; }
            else if (300 <= hue && hue < 360) { r=c; g=0; b=x; }
            
            renderColor = { r: Math.floor((r+m)*255), g: Math.floor((g+m)*255), b: Math.floor((b+m)*255) };
        } else {
            renderColor = manualColor;
        }
    }

    function draw() {
        let imgData = ctx.createImageData(N + 2, N + 2);
        let data = imgData.data;

        for (let i = 0; i < size; i++) {
            let d = dens[i];
            let idx = i * 4;
            if (d > 0.01) {
                let intensity = Math.min(1.0, d);
                data[idx]     = renderColor.r * intensity;
                data[idx + 1] = renderColor.g * intensity;
                data[idx + 2] = renderColor.b * intensity;
                data[idx + 3] = 255; 
            } else {
                data[idx + 3] = 0;
            }
        }

        let tempCanvas = document.createElement('canvas');
        tempCanvas.width = N + 2;
        tempCanvas.height = N + 2;
        tempCanvas.getContext('2d').putImageData(imgData, 0, 0);

        ctx.clearRect(0, 0, canvas.width, canvas.height);
        ctx.save();
        ctx.scale(canvas.width / (N + 2), canvas.height / (N + 2));
        ctx.drawImage(tempCanvas, 0, 0);
        ctx.restore();
    }

    function interact(x, y, dx, dy) {
        let rect = canvas.getBoundingClientRect();
        let scaleX = (N + 2) / 512; 
        let scaleY = (N + 2) / 512;
        let gridX = Math.floor((x - rect.left) * scaleX);
        let gridY = Math.floor((y - rect.top) * scaleY);

        if (gridX < 1 || gridX > N || gridY < 1 || gridY > N) return;
        let idx = IX(gridX, gridY);

        if (currentTool === 'draw') {
            dens[idx] += 100; 
            u[idx] += dx * 0.5; 
            v[idx] += dy * 0.5;
        } else {
            dens[idx] = 0; u[idx] = 0; v[idx] = 0;
            let neighbors = [idx-1, idx+1, idx-(N+2), idx+(N+2)];
            for(let n of neighbors) if(n > 0 && n < size) dens[n] = 0;
        }
    }

    // --- Inputs ---

    canvas.addEventListener('mousedown', e => {
        isMouseDown = true;
        lastX = e.clientX; lastY = e.clientY;
        interact(e.clientX, e.clientY, 0, 0);
    });
    window.addEventListener('mouseup', () => isMouseDown = false);
    canvas.addEventListener('mousemove', e => {
        if(!isMouseDown) return;
        let dx = e.clientX - lastX;
        let dy = e.clientY - lastY;
        interact(e.clientX, e.clientY, dx, dy);
        lastX = e.clientX; lastY = e.clientY;
    });

    // UI Hookups
    const btnDraw = document.getElementById('btnDraw');
    const btnErase = document.getElementById('btnErase');
    const btnPause = document.getElementById('btnPause');
    const btnClear = document.getElementById('btnClear');
    const chkRainbow = document.getElementById('chkRainbow');
    const sldLife = document.getElementById('sldLife');

    btnDraw.onclick = () => { currentTool = 'draw'; btnDraw.classList.add('active'); btnErase.classList.remove('active'); };
    btnErase.onclick = () => { currentTool = 'erase'; btnErase.classList.add('active'); btnDraw.classList.remove('active'); };
    
    btnPause.onclick = () => { paused = !paused; btnPause.textContent = paused ? "Play" : "Pause"; btnPause.classList.toggle('active'); };
    btnClear.onclick = () => { dens.fill(0); u.fill(0); v.fill(0); draw(); };
    
    chkRainbow.onchange = (e) => { rainbowMode = e.target.checked; updateColors(); };
    
    // Slider Logic
    sldLife.oninput = (e) => { fadeRate = parseFloat(e.target.value); };

    // Color Swatches
    function setManualColor(hex) {
        let result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
        if(result) {
            manualColor = { r: parseInt(result[1],16), g: parseInt(result[2],16), b: parseInt(result[3],16) };
            rainbowMode = false;
            chkRainbow.checked = false;
        }
    }
    document.querySelectorAll('.swatch').forEach(s => s.onclick = () => setManualColor(s.dataset.c));
    document.getElementById('inpColor').oninput = (e) => setManualColor(e.target.value);

    // Loop
    function loop() {
        updateColors();
        physicsStep();
        draw();
        requestAnimationFrame(loop);
    }
    loop();

</script>
</body>
</html>
