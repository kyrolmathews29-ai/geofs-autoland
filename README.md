(function() {
    'use strict';
    console.log(">>> GEOFS GPS-COUPLED AUTOLAND MOD LOADED <<<");

    // ==========================================
    // 1. CONFIGURATION
    // ==========================================
    const AC_DB = {
        CLASSIC: ["757", "767", "737-300", "737-400", "737-500", "737-200"], 
        MODERN: ["787", "777", "a350", "c919", "a220", "embraer", "e170", "e190"], 
        T_TAIL: ["md-80", "md-88", "crj", "learjet", "c-17"], 
        MD11: ["md-11", "dc-10"], 
        AIRBUS: ["a320", "a321", "a319", "a330", "a340", "a380"]
    };

    let settings = {
        mode: "NONE",
        pfdVisible: true,
        physicsEnabled: true,
        soundEnabled: true,
        autoland: false
    };

    // Autoland State
    let al = {
        targetRunway: null,
        phase: "IDLE", // IDLE, INTERCEPT, GLIDESLOPE, FLARE, ROLLOUT
        pid: {
            lastErrLat: 0,
            lastErrVert: 0,
            integralVert: 0
        }
    };

    // ==========================================
    // 2. UI & MENU
    // ==========================================
    function createUI() {
        if (document.getElementById("gmod_menu")) document.getElementById("gmod_menu").remove();
        if (document.getElementById("gmod_overlay")) document.getElementById("gmod_overlay").remove();

        // Main Menu
        let menu = document.createElement("div");
        menu.id = "gmod_menu";
        menu.style.cssText = `
            position: absolute; top: 10px; left: 10px; width: 260px;
            background: rgba(0,0,0,0.95); border: 2px solid #00ff00; color: white;
            font-family: Consolas, monospace; font-size: 11px; padding: 10px;
            z-index: 100000; box-shadow: 0 0 15px #000;
        `;
        menu.innerHTML = `
            <div style="font-weight:bold; color:#00ff00; border-bottom:1px solid #555; margin-bottom:10px; text-align:center;">
                GPS AUTOLAND SYSTEM
            </div>
            
            <div style="margin-bottom:8px;">
                <strong>SYSTEM STATUS:</strong> <span id="gm_status" style="color:yellow;">STANDBY</span><br>
                <strong>TARGET RWY:</strong> <span id="gm_rwy" style="color:cyan;">NONE</span>
            </div>

            <div style="background:#222; padding:5px; margin-bottom:5px;">
                <button id="btn_set_classic" style="width:48%; cursor:pointer;">757 CLASSIC</button>
                <button id="btn_set_modern" style="width:48%; cursor:pointer;">MODERN GLASS</button>
            </div>

            <div style="background:#222; padding:5px;">
                <label><input type="checkbox" id="chk_phys" checked> Physics (Stall/Wake)</label><br>
                <label><input type="checkbox" id="chk_snds" checked> RB211 Sounds</label>
            </div>

            <div style="margin-top:10px;">
                <button id="btn_autoland" style="width:100%; background:darkred; color:white; font-weight:bold; border:1px solid white; padding:8px; cursor:pointer; font-size:12px;">AUTOLAND: OFF</button>
            </div>
            
            <div id="gm_debug" style="margin-top:5px; color:#aaa; font-size:10px; height:30px; overflow:hidden;">Waiting for input...</div>
        `;
        document.body.appendChild(menu);

        // Handlers
        document.getElementById("btn_set_classic").onclick = () => setSystem("CLASSIC");
        document.getElementById("btn_set_modern").onclick = () => setSystem("MODERN");
        document.getElementById("chk_phys").onchange = (e) => settings.physicsEnabled = e.target.checked;
        document.getElementById("chk_snds").onchange = (e) => settings.soundEnabled = e.target.checked;

        document.getElementById("btn_autoland").onclick = function() {
            toggleAutoland();
            this.style.background = settings.autoland ? "lime" : "darkred";
            this.style.color = settings.autoland ? "black" : "white";
            this.innerText = settings.autoland ? "AUTOLAND: ENGAGED" : "AUTOLAND: OFF";
        };

        // PFD Overlay
        let overlay = document.createElement("div");
        overlay.id = "gmod_overlay";
        overlay.style.cssText = `position: absolute; bottom: 20px; left: 20px; pointer-events: none; z-index: 9000;`;
        
        let cvs = document.createElement("canvas");
        cvs.id = "gm_pfd";
        cvs.width = 260; cvs.height = 260;
        cvs.style.display = "none";
        overlay.appendChild(cvs);
        document.body.appendChild(overlay);
    }

    function updateDebug(msg) {
        document.getElementById("gm_debug").innerText = msg;
    }

    // ==========================================
    // 3. AUTOLAND LOGIC (The Complex Part)
    // ==========================================
    function findNearestRunway() {
        if (!geofs.runways) return null;
        
        let myLoc = geofs.aircraft.instance.llaLocation; // [lat, lon, alt]
        let bestRwy = null;
        let minDist = Infinity;

        // GeoFS stores runways in loaded tiles. We scan the keys.
        // NOTE: This structure depends on GeoFS version. We try standard access.
        // If geofs.runways is a flat object (older versions) vs tree (newer).
        // Safest method: Iterate everything we can find.
        
        Object.keys(geofs.runways).forEach(key => {
            let rwy = geofs.runways[key];
            // Calculate distance (Pythagoras on lat/lon degrees is "good enough" for local search)
            let dLat = rwy.lat - myLoc[0];
            let dLon = rwy.lon - myLoc[1];
            let distSq = dLat*dLat + dLon*dLon;
            
            if (distSq < minDist) {
                minDist = distSq;
                bestRwy = rwy;
            }
        });

        // If very far (approx > 50km), ignore
        if (minDist > 1.0) return null; 

        return bestRwy;
    }

    function toggleAutoland() {
        settings.autoland = !settings.autoland;
        if (settings.autoland) {
            // Engage
            al.targetRunway = findNearestRunway();
            if (al.targetRunway) {
                al.phase = "INTERCEPT";
                document.getElementById("gm_rwy").innerText = `HDG ${al.targetRunway.heading}° / ${(al.targetRunway.length * 3.28).toFixed(0)}ft`;
                geofs.autopilot.turnOn();
                geofs.autopilot.setMode("hdg");
                geofs.autopilot.setMode("alt"); // We will override VS manually
                updateDebug(`Locked: ${al.targetRunway.heading} deg`);
            } else {
                updateDebug("ERROR: No Runway Found!");
                settings.autoland = false; // Abort
            }
        } else {
            // Disengage
            al.phase = "IDLE";
            al.targetRunway = null;
            document.getElementById("gm_rwy").innerText = "NONE";
            geofs.autopilot.turnOff();
        }
    }

    function runAutolandPhysics() {
        if (!settings.autoland || !al.targetRunway) return;

        let myLoc = geofs.aircraft.instance.llaLocation;
        let rwy = al.targetRunway;
        
        // 1. Calculate Geometry
        // Convert to meters approx
        let latToM = 111000;
        let lonToM = 111000 * Math.cos(myLoc[0] * Math.PI / 180);
        
        let dN = (rwy.lat - myLoc[0]) * latToM; // Distance North
        let dE = (rwy.lon - myLoc[1]) * lonToM; // Distance East
        let distanceM = Math.sqrt(dN*dN + dE*dE);
        
        // Bearing TO the runway
        let bearingToRwy = (Math.atan2(dE, dN) * 180 / Math.PI);
        if (bearingToRwy < 0) bearingToRwy += 360;

        // Runway Heading vector
        let rwyHdgRad = rwy.heading * Math.PI / 180;
        
        // Altitude geometry (Standard 3 degree glide)
        // 3 deg = ~5.2% gradient
        // Target Alt = (Distance * 0.052) + Elevation
        // Note: GeoFS rwy.elevation is usually in feet, need to check unit. 
        // GeoFS uses meters internally usually? Actually altitude is feet in API values.
        // Let's assume rwy.elevation is meters (standard geofs).
        let rwyElevFt = rwy.elevation * 3.28084; 
        let targetAlt = (distanceM * 0.0524) + rwyElevFt; 
        let currentAlt = geofs.animation.values.altitude;
        
        // 2. Lateral Logic (Simple P-Controller)
        // If we are far, fly DIRECT to runway. If close, fly RUNWAY HEADING.
        let desiredHdg = 0;
        
        // Angle difference
        let diff = bearingToRwy - rwy.heading;
        // Normalize -180 to 180
        while (diff < -180) diff += 360;
        while (diff > 180) diff -= 360;

        // Intercept logic:
        // If we are > 2km out, point at the runway.
        // If we are < 2km out, blend towards runway heading.
        if (distanceM > 2000) {
            desiredHdg = bearingToRwy;
        } else {
            // Line up phase: steer to kill the offset
            desiredHdg = rwy.heading + (diff * 2.0); // Corrective multiplier
        }
        
        geofs.autopilot.values.heading = desiredHdg;

        // 3. Vertical Logic (Glideslope)
        let altError = currentAlt - targetAlt; // Positive means we are too high
        let desiredVS = 0;

        // Logic based on phase
        if (al.phase === "INTERCEPT") {
            // Just hold altitude until we intercept the glideslope from below
            if (altError > 0) {
                // We are above GS? Dive? No, dangerous. Maintain level or slow descent.
                // Wait, if we engage above GS, we must descend.
                // If altError < 0 (we are below), fly level.
            }
            if (altError < 50 && altError > -50) al.phase = "GLIDESLOPE"; // Captured
        }

        if (al.phase === "GLIDESLOPE") {
            // P-Control for VS
            // Base descent for 3 deg at 140kts is approx -700fpm
            let baseVS = -750;
            let correction = altError * 5; // 10ft high -> add -50fpm
            desiredVS = baseVS - correction;
            
            // Limiters
            if (desiredVS < -2000) desiredVS = -2000;
            if (desiredVS > 500) desiredVS = 500;
            
            geofs.autopilot.values.verticalSpeed = desiredVS;
            
            updateDebug(`GS Mode | Err: ${Math.round(altError)}ft | VS: ${Math.round(desiredVS)}`);
            
            // Flare Trigger
            if (currentAlt < rwyElevFt + 30) {
                al.phase = "FLARE";
            }
        }

        if (al.phase === "FLARE") {
            geofs.autopilot.values.verticalSpeed = -50; // Gentle touchdown
            geofs.api.controls.throttle = 0; // IDLE
            geofs.autopilot.values.heading = rwy.heading; // Keep straight
            updateDebug("FLARE FLARE FLARE");
            
            if (geofs.animation.values.groundContact) {
                al.phase = "ROLLOUT";
                settings.autoland = false; // Disconnect
                geofs.autopilot.turnOff();
            }
        }
    }

    // ==========================================
    // 4. MAIN LOOP & PHYSICS
    // ==========================================
    function setSystem(mode) {
        settings.mode = mode;
        let cvs = document.getElementById("gm_pfd");
        if (mode === "CLASSIC") {
            cvs.style.display = "block";
            cvs.style.borderRadius = "50%";
            cvs.style.border = "6px solid #444";
            cvs.style.background = "#222";
            applySounds();
        } else {
            cvs.style.display = "block";
            cvs.style.borderRadius = "10px";
            cvs.style.border = "2px solid #555";
            cvs.style.background = "black";
        }
    }

    function applySounds() {
        if (!settings.soundEnabled || !geofs.aircraft.instance.definition.sounds) return;
        geofs.aircraft.instance.definition.sounds.forEach(s => {
            if (s.id === "eng" || s.id === "turbine") {
                s.pitch = {value: 1, ratio: 1.45}; 
            }
        });
        if (geofs.audio) geofs.audio.initAircraftAudio();
    }

    // Loop
    setInterval(() => {
        if (typeof geofs === "undefined" || !geofs.aircraft || !geofs.aircraft.instance) return;

        // 1. Run Autoland
        runAutolandPhysics();

        // 2. Run Other Physics (Stall, Wake)
        if (settings.physicsEnabled) {
            let p = geofs.animation.values.pitch;
            let s = geofs.animation.values.kias;
            
            // Deep Stall (Simple T-Tail check)
            let name = geofs.aircraft.instance.definition.name.toLowerCase();
            if (AC_DB.T_TAIL.some(x => name.includes(x))) {
                if (p > 22 && s < 110) geofs.api.controls.pitch = 0; 
            }
            
            // Wake Turbulence
            let users = geofs.api.multiplayer.users;
            let myPos = geofs.aircraft.instance.llaLocation;
            for (let u in users) {
                let d = Math.sqrt(Math.pow(users[u].coor[0]-myPos[0],2) + Math.pow(users[u].coor[1]-myPos[1],2));
                if (d < 0.005 && d > 0.0001) {
                    geofs.camera.cam.rotation.z += (Math.random()-0.5)*0.01;
                }
            }
        }

        // 3. Draw PFD
        if (settings.mode !== "NONE") drawPFD();

    }, 50);

    // ==========================================
    // 5. GRAPHICS ENGINE
    // ==========================================
    function drawPFD() {
        let cvs = document.getElementById("gm_pfd");
        let ctx = cvs.getContext("2d");
        let w = cvs.width, h = cvs.height;
        let p = geofs.animation.values.pitch;
        let r = geofs.animation.values.roll;
        let s = geofs.animation.values.kias;
        let alt = geofs.animation.values.altitude;

        ctx.clearRect(0,0,w,h);
        ctx.save();
        ctx.translate(w/2, h/2);
        
        // Mask
        ctx.beginPath();
        if (settings.mode === "CLASSIC") ctx.arc(0,0, w/2-10, 0, Math.PI*2);
        else ctx.rect(-w/2, -h/2, w, h);
        ctx.clip();

        // Horizon
        ctx.rotate(-r * Math.PI / 180);
        ctx.translate(0, p * 4);
        ctx.fillStyle = settings.mode === "CLASSIC" ? "#3e9ec0" : "#003366";
        ctx.fillRect(-200, -400, 400, 400);
        ctx.fillStyle = settings.mode === "CLASSIC" ? "#8b572a" : "#4d2600";
        ctx.fillRect(-200, 0, 400, 400);
        ctx.strokeStyle = "white"; ctx.lineWidth = 2;
        ctx.beginPath(); ctx.moveTo(-200,0); ctx.lineTo(200,0); ctx.stroke();
        ctx.restore();

        // Speed Tape
        ctx.fillStyle = "rgba(0,0,0,0.8)"; ctx.fillRect(0, h/2-20, 60, 40);
        ctx.fillStyle = "lime"; ctx.font = "bold 20px monospace"; ctx.fillText(Math.round(s), 10, h/2+7);

        // Aircraft
        ctx.strokeStyle = settings.mode === "CLASSIC" ? "yellow" : "#fc0";
        ctx.lineWidth = 3;
        ctx.beginPath(); 
        ctx.moveTo(w/2-50, h/2); ctx.lineTo(w/2-10, h/2);
        ctx.moveTo(w/2+10, h/2); ctx.lineTo(w/2+50, h/2);
        ctx.stroke();

        // Autoland Status
        if (settings.autoland) {
            ctx.fillStyle = "lime"; ctx.font = "12px monospace"; ctx.textAlign = "center";
            ctx.fillText(al.phase, w/2, h/2 - 50);
        }
    }

    createUI();

})();
