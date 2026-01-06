import React, { useEffect, useRef } from 'react';
import { Zap, Activity, BatteryCharging, TrendingUp } from 'lucide-react';

const App = () => {
  // --- Refs for High Performance Animation (No State in Loop) ---
  const canvasRef = useRef(null);
  const containerRef = useRef(null);
  const requestRef = useRef(null);
  
  // Physics State (Mutable Refs)
  const timeRef = useRef(0);
  const historyRef = useRef([]); // For Oscilloscope
  const particlesRef = useRef([]); // For Spark system
  const energyRef = useRef(0); // For Battery gamification
  const batteryVisualRef = useRef(null); // Direct DOM manipulation for performance

  // Constants
  const MAGNET_AMPLITUDE = 200;
  const FREQUENCY = 0.08; // Fast paced!
  const COIL_Y = 0; // Center of screen relative to canvas center

  // --- Animation Loop ---
  useEffect(() => {
    const animate = () => {
      const cvs = canvasRef.current;
      if (!cvs || !containerRef.current) return;

      const ctx = cvs.getContext('2d');
      const w = containerRef.current.clientWidth;
      const h = containerRef.current.clientHeight;
      const dpr = window.devicePixelRatio || 2; // Force high res
      
      // Handle Resize
      if (cvs.width !== w * dpr || cvs.height !== h * dpr) {
        cvs.width = w * dpr;
        cvs.height = h * dpr;
        ctx.scale(dpr, dpr);
      }

      const cx = w / 2;
      const cy = h / 2;

      // --- 1. Physics Calculations ---
      timeRef.current += 1; // Frame count
      
      // Simple Harmonic Motion: y = A * sin(wt)
      // Velocity is derivative: v = A * w * cos(wt)
      const t = timeRef.current;
      const magY = Math.sin(t * FREQUENCY) * MAGNET_AMPLITUDE;
      const velocity = Math.cos(t * FREQUENCY) * MAGNET_AMPLITUDE * FREQUENCY;
      const speed = Math.abs(velocity);
      // const isApproaching = (magY < 0 && velocity > 0) || (magY > 0 && velocity < 0); // Unused but kept for logic reference
      
      // Color Logic based on Lenz's Law
      // Down (approaching) = Cyan (Opposing)
      // Up (leaving) = Magenta (Attracting)
      const primaryColor = velocity > 0 ? '#22d3ee' : '#e879f9'; // Cyan vs Fuchsia
      const glowIntensity = Math.min(speed / 10, 1); // 0 to 1 based on speed

      // Battery Logic
      if (speed > 5) energyRef.current = Math.min(100, energyRef.current + 0.2);
      if (energyRef.current >= 100) energyRef.current = 0; // Loop reset

      // Update Battery DOM directly (avoiding React render loop)
      if (batteryVisualRef.current) {
        batteryVisualRef.current.style.height = `${energyRef.current}%`;
      }

      // Oscilloscope History
      historyRef.current.push(velocity);
      if (historyRef.current.length > w / 2) historyRef.current.shift();

      // --- 2. Rendering ---

      // Clear with Void Black
      ctx.fillStyle = '#000000';
      ctx.fillRect(0, 0, w, h);

      // A. Background Field Vectors (Visualization of B)
      ctx.strokeStyle = '#333';
      ctx.lineWidth = 1;
      const gridSize = 40;
      for (let x = 0; x <= w; x += gridSize) {
        for (let y = 0; y <= h; y += gridSize) {
          // Vector points away from magnet N pole (simple approximation)
          const dx = x - cx;
          const dy = y - (cy + magY);
          const dist = Math.sqrt(dx*dx + dy*dy);
          const angle = Math.atan2(dy, dx);
          
          // Vectors react to magnet proximity
          const len = Math.min(20, 10000 / (dist * dist + 100)); 
          
          ctx.beginPath();
          ctx.moveTo(x, y);
          ctx.lineTo(x + Math.cos(angle) * len, y + Math.sin(angle) * len);
          ctx.stroke();
        }
      }

      // B. The Coil (Solenoid)
      const coilR = 80;
      const coilH = 100;
      const turns = 8;
      
      ctx.shadowBlur = glowIntensity * 40;
      ctx.shadowColor = primaryColor;
      ctx.strokeStyle = `rgba(255, 255, 255, ${0.2 + glowIntensity})`;
      ctx.lineWidth = 4;

      for (let i = 0; i < turns; i++) {
        const yOffset = -coilH/2 + (i * (coilH/turns));
        ctx.beginPath();
        // Draw ellipses to simulate 3D coil
        ctx.ellipse(cx, cy + yOffset, coilR, 20, 0, 0, Math.PI * 2);
        ctx.stroke();
      }
      ctx.shadowBlur = 0; // Reset

      // C. Spark Particles (High Voltage Breakdown)
      // Spawn sparks when speed is high
      if (speed > 12) {
        for(let k=0; k<3; k++) {
            particlesRef.current.push({
                x: cx + (Math.random() - 0.5) * coilR * 2,
                y: cy + (Math.random() - 0.5) * coilH,
                vx: (Math.random() - 0.5) * 10,
                vy: (Math.random() - 0.5) * 10,
                life: 1.0,
                color: primaryColor
            });
        }
      }

      // Update & Draw Particles
      for (let i = particlesRef.current.length - 1; i >= 0; i--) {
        const p = particlesRef.current[i];
        p.x += p.vx;
        p.y += p.vy;
        p.life -= 0.05;
        
        ctx.fillStyle = p.color;
        ctx.globalAlpha = p.life;
        ctx.beginPath();
        ctx.arc(p.x, p.y, 2, 0, Math.PI * 2);
        ctx.fill();
        ctx.globalAlpha = 1.0;

        if (p.life <= 0) particlesRef.current.splice(i, 1);
      }

      // D. The Magnet
      const magW = 60;
      const magH = 100;
      const my = cy + magY;
      
      ctx.save();
      ctx.translate(cx - magW/2, my - magH/2);
      
      // South Pole (Red)
      ctx.fillStyle = '#ef4444'; 
      ctx.fillRect(0, 0, magW, magH/2);
      
      // North Pole (Blue) - Standard Physics Convention
      ctx.fillStyle = '#3b82f6'; 
      ctx.fillRect(0, magH/2, magW, magH/2);
      
      // Magnet Shine
      ctx.fillStyle = 'rgba(255,255,255,0.2)';
      ctx.fillRect(40, 0, 10, magH);

      // N / S Labels
      ctx.fillStyle = 'white';
      ctx.font = 'bold 20px Arial';
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      ctx.fillText('S', magW/2, magH/4);
      ctx.fillText('N', magW/2, magH*0.75);
      
      ctx.restore();

      // E. Oscilloscope (Top Layer)
      const graphH = 100;
      const graphY = 80;
      
      ctx.beginPath();
      ctx.strokeStyle = primaryColor;
      ctx.lineWidth = 2;
      for (let i = 0; i < historyRef.current.length; i++) {
        const val = historyRef.current[i];
        const x = w - (historyRef.current.length - i) * 2;
        // Normalize val to graph height
        const y = graphY - (val / 20) * (graphH/2);
        if (i===0) ctx.moveTo(x, y);
        else ctx.lineTo(x, y);
      }
      ctx.stroke();

      requestRef.current = requestAnimationFrame(animate);
    };

    requestRef.current = requestAnimationFrame(animate);
    return () => cancelAnimationFrame(requestRef.current);
  }, []);

  return (
    <div className="flex flex-col h-screen bg-black text-white font-sans overflow-hidden mx-auto border-x border-zinc-800 shadow-2xl relative w-full max-w-md">
      
      {/* --- CANVAS LAYER --- */}
      <div ref={containerRef} className="flex-grow relative">
        <canvas ref={canvasRef} className="absolute inset-0 w-full h-full" />
        
        {/* --- UI OVERLAY: HUD --- */}
        <div className="absolute top-0 left-0 w-full p-4 pointer-events-none bg-gradient-to-b from-black via-black/80 to-transparent pb-20">
            <div className="flex justify-between items-start">
                <div>
                    <h1 className="text-3xl font-black italic tracking-tighter text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-fuchsia-400 flex items-center gap-2">
                        <Zap className="fill-cyan-400 text-cyan-400" />
                        FARADAY
                    </h1>
                    <p className="text-[10px] text-zinc-400 font-mono mt-0 uppercase tracking-widest pl-1">
                        Electromagnetic Induction v2.0
                    </p>
                </div>
                <div className="text-right">
                    <div className="flex items-center justify-end gap-1 text-xs font-bold text-red-500 animate-pulse">
                        <Activity size={14} /> LIVE
                    </div>
                </div>
            </div>

            {/* Oscilloscope Label */}
            <div className="mt-2 pl-1 border-l-2 border-zinc-700">
                <div className="text-[9px] text-zinc-500 font-mono">INDUCED VOLTAGE (ε)</div>
            </div>
        </div>

        {/* --- EDUCATIONAL CARDS (Bottom) --- */}
        <div className="absolute bottom-0 left-0 w-full p-4 pointer-events-none bg-gradient-to-t from-black via-black/90 to-transparent pt-20">
            
            {/* The Equation */}
            <div className="flex items-center justify-between mb-4">
                <div className="bg-zinc-900/80 backdrop-blur border border-zinc-700 p-4 rounded-xl shadow-2xl w-2/3">
                    <div className="text-[10px] text-zinc-400 font-mono uppercase mb-1 flex items-center gap-2">
                        <TrendingUp size={12}/> The Master Equation
                    </div>
                    <div className="text-2xl font-serif italic text-white">
                        ε = -N <span className="text-fuchsia-400 font-bold">dΦ</span>/<span className="text-cyan-400 font-bold">dt</span>
                    </div>
                    <div className="text-[9px] text-zinc-500 mt-1 leading-tight">
                        <span className="text-white">Change in Flux</span> over Time = <span className="text-yellow-400 font-bold">VOLTAGE</span>
                    </div>
                </div>

                {/* Gamified Battery */}
                <div className="bg-zinc-900/80 backdrop-blur border border-zinc-700 p-2 rounded-xl flex flex-col items-center w-1/4 h-full justify-between">
                    <BatteryCharging className="text-yellow-400 mb-1" size={20} />
                    <div className="w-4 h-16 bg-zinc-800 rounded-full overflow-hidden border border-zinc-600 relative">
                        {/* Connected to animation loop via ref */}
                        <div 
                            ref={batteryVisualRef}
                            className="absolute bottom-0 w-full bg-yellow-400 transition-all duration-75" 
                            style={{height: '0%'}}
                        ></div>
                    </div>
                    <span className="text-[9px] font-bold text-yellow-400 mt-1">PWR</span>
                </div>
            </div>

            {/* Speed Explanation */}
            <div className="flex gap-2 text-[10px] font-mono text-zinc-400 uppercase tracking-tight justify-center opacity-80 mb-2">
               <span className="text-cyan-400">● FAST MOTION</span> = <span className="text-white">HIGH VOLTAGE</span>
            </div>
            
            {/* Watermark */}
            <div className="text-right mt-2 opacity-50">
                <span className="text-sm font-black text-white/20 tracking-widest font-sans">@VizzlLab</span>
            </div>
        </div>

        {/* --- SIDE GAUGE --- */}
        <div className="absolute right-4 top-1/2 -translate-y-1/2 flex flex-col gap-1 items-center opacity-80">
            <div className="w-1 h-2 bg-zinc-800"></div>
            <div className="w-1 h-2 bg-zinc-700"></div>
            <div className="w-1 h-2 bg-zinc-600"></div>
            <div className="w-1 h-2 bg-fuchsia-500 shadow-[0_0_10px_#d946ef]"></div>
            <div className="w-1 h-2 bg-white shadow-[0_0_15px_white]"></div>
            <div className="w-1 h-2 bg-cyan-500 shadow-[0_0_10px_#06b6d4]"></div>
            <div className="w-1 h-2 bg-zinc-600"></div>
            <div className="w-1 h-2 bg-zinc-700"></div>
            <div className="w-1 h-2 bg-zinc-800"></div>
            <div className="text-[8px] font-bold text-zinc-500 rotate-90 mt-2">RPM</div>
        </div>

      </div>
    </div>
  );
};

export default App;
