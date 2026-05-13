# kameraai<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>NEURAL-EYE v2.0 | Tactical Reconnaissance</title>
    <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @keyframes scan {
            0% { top: 0; }
            100% { top: 100%; }
        }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        .matrix-bg {
            background: radial-gradient(circle, #050505 0%, #000 100%);
        }
        .scanline-effect {
            background: linear-gradient(rgba(18, 16, 16, 0) 50%, rgba(0, 0, 0, 0.1) 50%), 
                        linear-gradient(90deg, rgba(255, 0, 0, 0.03), rgba(0, 255, 0, 0.01), rgba(0, 0, 255, 0.03));
            background-size: 100% 2px, 3px 100%;
        }
    </style>
</head>
<body class="matrix-bg text-[#00ff41] font-mono overflow-x-hidden">
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef, useCallback } = React;

        const API_KEY = ""; 
        const AI_MODEL_URL = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${API_KEY}`;
        const AI_COOLDOWN_MS = 5000;
        const MOTION_THRESHOLD = 3;

        function App() {
            const [isMonitoring, setIsMonitoring] = useState(false);
            const [audioEnabled, setAudioEnabled] = useState(true);
            const [logs, setLogs] = useState([]);
            const [motionLevel, setMotionLevel] = useState(0);
            const [isAnalyzing, setIsAnalyzing] = useState(false);
            const [isTalking, setIsTalking] = useState(false);

            const videoRef = useRef(null);
            const canvasRef = useRef(null);
            const captureCanvasRef = useRef(null);
            const previousFrameRef = useRef(null);
            const requestRef = useRef(0);
            const lastAnalysisTimeRef = useRef(0);
            const logsContainerRef = useRef(null);

            const addLog = useCallback((message, type = 'info') => {
                setLogs(prev => [...prev.slice(-20), {
                    id: Math.random().toString(36).substr(2, 9),
                    timestamp: new Date().toLocaleTimeString('id-ID', { hour12: false }),
                    message,
                    type
                }]);
            }, []);

            const speakIndonesian = useCallback((text) => {
                if (!audioEnabled || !('speechSynthesis' in window)) return;
                window.speechSynthesis.cancel();
                const utterance = new SpeechSynthesisUtterance(text);
                utterance.lang = 'id-ID';
                utterance.onstart = () => setIsTalking(true);
                utterance.onend = () => setIsTalking(false);
                window.speechSynthesis.speak(utterance);
            }, [audioEnabled]);

            const analyzeImage = async (base64Image) => {
                setIsAnalyzing(true);
                addLog('NEURAL LINK: Pemindaian data...', 'info');
                const payload = {
                    contents: [{
                        parts: [
                            { text: "Sebutkan 1 objek dominan yang Anda lihat dalam bahasa Indonesia yang tegas." },
                            { inlineData: { mimeType: "image/jpeg", data: base64Image.split(',')[1] } }
                        ]
                    }]
                };
                try {
                    const response = await fetch(AI_MODEL_URL, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    });
                    const data = await response.json();
                    const resultText = data.candidates?.[0]?.content?.parts?.[0]?.text || "Objek tidak terdeteksi.";
                    addLog(`HASIL SCAN: ${resultText}`, 'ai');
                    speakIndonesian(resultText);
                } catch (error) {
                    addLog('KRITIS: Jaringan AI terputus.', 'alert');
                }
                setIsAnalyzing(false);
            };

            const processFrame = useCallback(() => {
                if (!isMonitoring || !videoRef.current || !canvasRef.current) return;
                const ctx = canvasRef.current.getContext('2d', { willReadFrequently: true });
                if (!ctx) return;
                ctx.drawImage(videoRef.current, 0, 0, 64, 48);
                const currentFrame = ctx.getImageData(0, 0, 64, 48);
                if (previousFrameRef.current) {
                    let diff = 0;
                    for (let i = 0; i < currentFrame.data.length; i += 4) {
                        if (Math.abs(currentFrame.data[i] - previousFrameRef.current.data[i]) > 25) diff++;
                    }
                    const motion = (diff / (currentFrame.data.length / 4)) * 100;
                    setMotionLevel(motion);
                    if (motion > MOTION_THRESHOLD && !isAnalyzing) {
                        const now = Date.now();
                        if (now - lastAnalysisTimeRef.current > AI_COOLDOWN_MS) {
                            lastAnalysisTimeRef.current = now;
                            const capCtx = captureCanvasRef.current.getContext('2d');
                            captureCanvasRef.current.width = videoRef.current.videoWidth;
                            captureCanvasRef.current.height = videoRef.current.videoHeight;
                            capCtx.drawImage(videoRef.current, 0, 0);
                            analyzeImage(captureCanvasRef.current.toDataURL('image/jpeg'));
                        }
                    }
                }
                previousFrameRef.current = currentFrame;
                requestRef.current = requestAnimationFrame(processFrame);
            }, [isMonitoring, isAnalyzing]);

            useEffect(() => {
                if (isMonitoring) requestRef.current = requestAnimationFrame(processFrame);
                return () => cancelAnimationFrame(requestRef.current);
            }, [isMonitoring, processFrame]);

            const initSystem = async () => {
                try {
                    const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
                    if (videoRef.current) videoRef.current.srcObject = stream;
                    setIsMonitoring(true);
                    addLog('SISTEM AKTIF: Tautan neural stabil.', 'info');
                    speakIndonesian('Sistem aktif.');
                } catch (err) {
                    addLog('FATAL: Kamera tidak merespons.', 'alert');
                }
            };

            return (
                <div className="min-h-screen p-4 sm:p-8 flex flex-col items-center">
                    <header className="w-full max-w-6xl flex justify-between items-center border-b border-[#003b00] pb-4 mb-8">
                        <div className="flex items-center gap-4">
                            <i data-lucide="shield-alert" className={`w-8 h-8 ${isMonitoring ? 'text-red-500 animate-pulse' : 'text-green-900'}`}></i>
                            <div>
                                <h1 className="text-2xl font-black tracking-tighter uppercase leading-none">
                                    Neural-Eye <span className="text-white">CCTV</span>
                                </h1>
                                <p className="text-[10px] text-green-800 tracking-widest uppercase">Tactical AI Vision v2.0</p>
                            </div>
                        </div>
                        <button onClick={() => setAudioEnabled(!audioEnabled)} className="p-2 border border-green-800 hover:bg-green-900/20">
                           <i data-lucide={audioEnabled ? "volume-2" : "volume-x"}></i>
                        </button>
                    </header>

                    <div className="w-full max-w-6xl grid grid-cols-1 lg:grid-cols-4 gap-6">
                        <div className="lg:col-span-3">
                            <div className="relative aspect-video bg-black border border-[#003b00] overflow-hidden group">
                                <video ref={videoRef} autoPlay playsInline muted className={`w-full h-full object-cover transition-opacity duration-1000 ${isMonitoring ? 'opacity-80' : 'opacity-20'}`} />
                                <div className="absolute inset-0 scanline-effect pointer-events-none"></div>
                                {isMonitoring && <div className="absolute top-0 left-0 w-full h-[1px] bg-green-500/30 animate-[scan_3s_linear_infinite]"></div>}
                                
                                <div className="absolute top-4 left-4 bg-black/80 border border-green-500 px-3 py-1 text-[10px] flex items-center gap-2">
                                    <div className={`w-2 h-2 rounded-full ${isMonitoring ? 'bg-red-500 animate-pulse' : 'bg-zinc-700'}`}></div>
                                    {isMonitoring ? "SYSTEM_LIVE" : "SYSTEM_OFFLINE"}
                                </div>

                                {isAnalyzing && (
                                    <div className="absolute inset-0 flex items-center justify-center bg-green-500/10 backdrop-blur-sm">
                                        <div className="bg-black border border-green-500 p-4 animate-pulse">
                                            <p className="text-xs font-bold tracking-widest">ANALYZING_NEURAL_STREAM...</p>
                                        </div>
                                    </div>
                                )}
                            </div>
                            
                            <button 
                                onClick={isMonitoring ? () => setIsMonitoring(false) : initSystem}
                                className={`w-full mt-4 py-4 font-black border-2 transition-all ${isMonitoring ? 'bg-red-950/20 border-red-500 text-red-500' : 'bg-green-950/20 border-green-500 text-green-500 shadow-[0_0_20px_rgba(0,255,65,0.2)]'}`}
                            >
                                {isMonitoring ? 'TERMINATE LINK' : 'INITIALIZE NEURAL LINK'}
                            </button>
                        </div>

                        <div className="space-y-6">
                            <div className="bg-zinc-950 border border-green-900 p-4">
                                <p className="text-[10px] text-green-700 uppercase mb-2">Motion Sensor</p>
                                <div className="h-4 w-full bg-zinc-900 border border-green-900">
                                    <div className={`h-full transition-all duration-75 ${motionLevel > MOTION_THRESHOLD ? 'bg-red-600 shadow-[0_0_10px_red]' : 'bg-green-600'}`} style={{width: `${Math.min(motionLevel * 5, 100)}%`}}></div>
                                </div>
                            </div>

                            <div className="bg-zinc-950 border border-green-900 h-[350px] flex flex-col">
                                <div className="p-2 border-b border-green-900 text-[10px] bg-green-900/10 uppercase">Event_Logs</div>
                                <div className="p-3 text-[10px] space-y-2 overflow-y-auto scrollbar-hide">
                                    {logs.map(log => (
                                        <div key={log.id} className={`${log.type === 'alert' ? 'text-red-500' : log.type === 'ai' ? 'text-yellow-400' : 'text-green-700'}`}>
                                            [{log.timestamp}] {log.message}
                                        </div>
                                    ))}
                                </div>
                            </div>
                        </div>
                    </div>
                    <canvas ref={canvasRef} width="64" height="48" className="hidden"></canvas>
                    <canvas ref={captureCanvasRef} className="hidden"></canvas>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
        setTimeout(() => lucide.createIcons(), 500);
    </script>
</body>
</html>
