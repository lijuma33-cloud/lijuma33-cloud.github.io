# https-lijuma33-nube.github.io

<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>QueueFlow Pro - Local Edition</title>
    <!-- Frameworks desde CDN para evitar instalaciones locales -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <style>
        @keyframes pulse-slow {
            0%, 100% { opacity: 1; transform: scale(1); }
            50% { opacity: 0.95; transform: scale(0.98); }
        }
        .animate-pulse-slow { animation: pulse-slow 4s infinite ease-in-out; }
        body { background-color: #f8fafc; font-family: system-ui, -apple-system, sans-serif; }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;

        const App = () => {
            const [view, setView] = useState('kiosk'); 
            const [queue, setQueue] = useState(() => JSON.parse(localStorage.getItem('local_queue') || '[]'));
            const [currentTurn, setCurrentTurn] = useState(() => JSON.parse(localStorage.getItem('local_current') || 'null'));
            const [historyCount, setHistoryCount] = useState(() => parseInt(localStorage.getItem('local_count') || '0'));
            const [clientName, setClientName] = useState('');
            const [stationName, setStationName] = useState(localStorage.getItem('stationName') || 'Caja 1');
            const [audioEnabled, setAudioEnabled] = useState(false);
            
            // Referencia para evitar anuncios duplicados por el intervalo de polling
            const lastAnnouncedId = useRef(null);

            // Sincronización automática entre pestañas y dispositivos
            useEffect(() => {
                const syncData = () => {
                    const latestQueue = JSON.parse(localStorage.getItem('local_queue') || '[]');
                    const latestCurrent = JSON.parse(localStorage.getItem('local_current') || 'null');
                    const latestCount = parseInt(localStorage.getItem('local_count') || '0');

                    setQueue(latestQueue);
                    setHistoryCount(latestCount);

                    if (latestCurrent && latestCurrent.id !== lastAnnouncedId.current) {
                        if (view === 'display' && audioEnabled) {
                            announceTurn(latestCurrent);
                            lastAnnouncedId.current = latestCurrent.id;
                        }
                        setCurrentTurn(latestCurrent);
                    }
                };

                window.addEventListener('storage', syncData);
                const interval = setInterval(syncData, 1000); 
                return () => {
                    window.removeEventListener('storage', syncData);
                    clearInterval(interval);
                };
            }, [view, audioEnabled]);

            const announceTurn = (turn) => {
                if ('speechSynthesis' in window) {
                    // Cancelar cualquier discurso previo para evitar colas infinitas
                    window.speechSynthesis.cancel();

                    const nombreCliente = turn.clientName && turn.clientName !== "Cliente" ? turn.clientName : "cliente";
                    const mensaje = new SpeechSynthesisUtterance();
                    
                    // Texto del anuncio
                    mensaje.text = `Turno ${turn.number}. ${nombreCliente}. Por favor pase a la ${turn.station}`;
                    mensaje.lang = 'es-ES';
                    mensaje.rate = 0.85; 
                    mensaje.pitch = 1;
                    mensaje.volume = 1;

                    // Reproducir
                    window.speechSynthesis.speak(mensaje);
                }
            };

            const enableAudioAndWakeUp = () => {
                setAudioEnabled(true);
                // "Despertar" el motor de voz con un mensaje silencioso tras la interacción del usuario
                if ('speechSynthesis' in window) {
                    const wakeUp = new SpeechSynthesisUtterance("");
                    wakeUp.volume = 0;
                    window.speechSynthesis.speak(wakeUp);
                }
            };

            const generateTurn = (type) => {
                const nextCount = historyCount + 1;
                const prefix = type === 'P' ? 'P' : 'C';
                const newTurn = {
                    id: Date.now(),
                    number: `${prefix}-${nextCount}`,
                    clientName: clientName.trim() || "Cliente",
                    type,
                    createdAt: new Date().toISOString()
                };
                
                const newQueue = [...queue, newTurn];
                localStorage.setItem('local_queue', JSON.stringify(newQueue));
                localStorage.setItem('local_count', nextCount.toString());
                
                setQueue(newQueue);
                setHistoryCount(nextCount);
                setClientName('');
            };

            const callNext = () => {
                if (queue.length === 0) return;
                const sorted = [...queue].sort((a, b) => (a.type === 'P' ? -1 : 1));
                const next = sorted[0];
                const updatedQueue = queue.filter(q => q.id !== next.id);
                const turnCalled = { ...next, station: stationName, timestamp: Date.now() };

                localStorage.setItem('local_queue', JSON.stringify(updatedQueue));
                localStorage.setItem('local_current', JSON.stringify(turnCalled));
                
                setQueue(updatedQueue);
                setCurrentTurn(turnCalled);
                window.dispatchEvent(new Event('storage'));
            };

            const resetSystem = () => {
                if(confirm("¿Desea reiniciar todos los turnos del día?")) {
                    localStorage.clear();
                    location.reload();
                }
            };

            return (
                <div className="min-h-screen flex flex-col">
                    <nav className="bg-white border-b px-6 py-4 flex justify-between items-center shadow-md">
                        <div className="flex items-center gap-2 font-black text-2xl text-blue-600">
                            <span className="bg-blue-600 text-white px-2 py-1 rounded-lg">Q</span>
                            <span>QueueFlow</span>
                        </div>
                        <div className="flex bg-slate-100 p-1 rounded-xl gap-1">
                            {['kiosk', 'staff', 'display'].map(v => (
                                <button 
                                    key={v}
                                    onClick={() => setView(v)}
                                    className={`px-4 py-2 rounded-lg text-xs font-bold uppercase transition ${view === v ? 'bg-white shadow text-blue-600' : 'text-slate-500 hover:text-slate-700'}`}
                                >
                                    {v === 'kiosk' ? 'Ingreso' : v === 'staff' ? 'Caja' : 'TV'}
                                </button>
                            ))}
                        </div>
                    </nav>

                    <main className="flex-1 p-6">
                        {view === 'kiosk' && (
                            <div className="max-w-xl mx-auto py-12 text-center bg-white p-10 rounded-[3rem] shadow-xl border border-slate-100">
                                <h1 className="text-3xl font-black mb-10 text-slate-800 tracking-tight">SOLICITE SU TURNO</h1>
                                <div className="text-left mb-8">
                                    <label className="text-sm font-bold text-slate-400 mb-2 block uppercase">Nombre del Cliente</label>
                                    <input 
                                        type="text" 
                                        value={clientName}
                                        onChange={e => setClientName(e.target.value)}
                                        placeholder="Ej: Juan Pérez"
                                        className="w-full p-4 rounded-2xl border-2 border-slate-100 text-xl focus:border-blue-400 outline-none transition-all shadow-inner"
                                    />
                                </div>
                                <div className="grid grid-cols-1 gap-4">
                                    <button onClick={() => generateTurn('C')} className="bg-blue-600 text-white p-6 rounded-2xl font-black text-xl hover:bg-blue-700 active:scale-95 transition shadow-lg">
                                        TURNO GENERAL
                                    </button>
                                    <button onClick={() => generateTurn('P')} className="bg-amber-500 text-white p-6 rounded-2xl font-black text-xl hover:bg-amber-600 active:scale-95 transition shadow-lg">
                                        TURNO PRIORITARIO
                                    </button>
                                </div>
                            </div>
                        )}

                        {view === 'staff' && (
                            <div className="max-w-6xl mx-auto grid grid-cols-1 lg:grid-cols-3 gap-8">
                                <div className="lg:col-span-2 space-y-6">
                                    <div className="bg-white p-8 rounded-3xl shadow-sm border border-slate-200">
                                        <div className="flex justify-between items-center mb-8">
                                            <h2 className="font-black text-xl text-slate-700">Panel de Operador</h2>
                                            <div className="flex flex-col">
                                                <label className="text-[10px] font-bold text-slate-400 uppercase mb-1">Caja Actual</label>
                                                <select 
                                                    value={stationName} 
                                                    onChange={e => {
                                                        const val = e.target.value;
                                                        setStationName(val); 
                                                        localStorage.setItem('stationName', val);
                                                    }}
                                                    className="border-2 border-slate-100 rounded-xl px-4 py-2 font-bold text-blue-600 bg-slate-50 outline-none focus:border-blue-400 cursor-pointer"
                                                >
                                                    <option value="Caja 1">Caja 1</option>
                                                    <option value="Caja 2">Caja 2</option>
                                                    <option value="Caja 3">Caja 3</option>
                                                </select>
                                            </div>
                                        </div>
                                        <div className="bg-slate-50 rounded-[2rem] p-12 text-center border-2 border-dashed border-slate-200">
                                            <p className="text-slate-400 font-bold text-xs uppercase tracking-widest mb-2">Atendiendo Ahora ({stationName})</p>
                                            <h3 className="text-8xl font-black text-slate-800 mb-2 leading-none">{currentTurn?.number || '---'}</h3>
                                            <p className="text-2xl text-blue-600 font-bold mb-10 truncate px-4">{currentTurn?.clientName || 'Sin turnos activos'}</p>
                                            <button onClick={callNext} className="w-full bg-blue-600 text-white font-black py-6 rounded-2xl text-2xl shadow-xl hover:bg-blue-700 active:scale-95 transition-all">
                                                LLAMAR SIGUIENTE
                                            </button>
                                        </div>
                                    </div>
                                    <button onClick={resetSystem} className="text-red-400 font-bold text-sm uppercase tracking-tighter hover:text-red-600 transition">Reiniciar Base de Datos</button>
                                </div>
                                <div className="bg-white p-6 rounded-3xl shadow-sm border border-slate-200 flex flex-col h-[600px]">
                                    <h2 className="font-black text-lg mb-6 flex justify-between items-center">
                                        COLA DE ESPERA
                                        <span className="bg-blue-100 text-blue-600 text-xs px-2 py-1 rounded-full">{queue.length}</span>
                                    </h2>
                                    <div className="space-y-3 overflow-y-auto pr-2">
                                        {queue.map(t => (
                                            <div key={t.id} className={`p-4 rounded-2xl border-2 ${t.type === 'P' ? 'border-amber-100 bg-amber-50' : 'border-slate-50 bg-slate-50/50'}`}>
                                                <p className="font-black text-xl text-slate-800">{t.number}</p>
                                                <p className="text-sm font-bold text-slate-500 truncate">{t.clientName}</p>
                                            </div>
                                        ))}
                                        {queue.length === 0 && <p className="text-center text-slate-300 py-10 font-bold italic">Cola vacía</p>}
                                    </div>
                                </div>
                            </div>
                        )}

                        {view === 'display' && (
                            <div className="h-full relative max-w-7xl mx-auto">
                                {!audioEnabled && (
                                    <div className="absolute inset-0 z-50 bg-slate-900/95 backdrop-blur-sm rounded-[3rem] flex flex-col items-center justify-center text-white p-12 text-center border-8 border-blue-600">
                                        <h2 className="text-5xl font-black mb-6 tracking-tighter">SISTEMA TV</h2>
                                        <p className="mb-12 text-slate-400 text-xl max-w-md">Para escuchar los anuncios por voz, por favor habilite el sonido en esta pantalla.</p>
                                        <button onClick={enableAudioAndWakeUp} className="bg-blue-600 px-16 py-6 rounded-full text-3xl font-black shadow-2xl hover:bg-blue-500 transition-all animate-bounce">
                                            ACTIVAR AUDIO
                                        </button>
                                    </div>
                                )}
                                <div className="bg-slate-900 rounded-[4rem] min-h-[80vh] flex flex-col md:flex-row shadow-2xl overflow-hidden border-8 border-slate-800">
                                    <div className="flex-1 p-20 flex flex-col items-center justify-center text-center bg-gradient-to-b from-slate-800 to-slate-900 border-r border-slate-800">
                                        <h2 className="text-blue-400 font-black text-4xl mb-12 tracking-[0.3em]">TURNO</h2>
                                        <div className="bg-white/5 p-16 rounded-[5rem] border border-white/10 shadow-inner animate-pulse-slow w-full max-w-lg">
                                            <div className="text-[14rem] font-black text-white leading-none tracking-tighter">{currentTurn?.number || '---'}</div>
                                            <div className="text-4xl text-blue-300 font-black mt-6 truncate uppercase">{currentTurn?.clientName || ''}</div>
                                        </div>
                                        <div className="mt-12">
                                            <p className="text-slate-500 text-2xl font-bold mb-4 uppercase tracking-widest">DIRIGIRSE A</p>
                                            <p className="text-white text-8xl font-black uppercase tracking-tighter text-blue-500">{currentTurn?.station || 'ESPERANDO'}</p>
                                        </div>
                                    </div>
                                    <div className="w-full md:w-[400px] bg-slate-900 p-12">
                                        <h3 className="text-slate-500 font-black text-xl mb-10 tracking-widest border-b border-slate-800 pb-4">PRÓXIMOS</h3>
                                        <div className="space-y-6">
                                            {queue.slice(0, 5).map((t, idx) => (
                                                <div key={t.id} className={`p-8 rounded-[2.5rem] flex flex-col ${idx === 0 ? 'bg-blue-600 shadow-[0_0_40px_rgba(37,99,235,0.3)]' : 'bg-white/5 border border-white/5'}`}>
                                                    <div className="flex justify-between items-center mb-2">
                                                        <span className="text-5xl font-black text-white">{t.number}</span>
                                                        <span className={`text-xs font-black px-3 py-1 rounded-full ${idx === 0 ? 'bg-white/20 text-white' : 'bg-slate-700 text-slate-400'}`}>
                                                            {t.type === 'P' ? 'PRIO' : 'GEN'}
                                                        </span>
                                                    </div>
                                                    <p className={`font-black truncate uppercase tracking-widest text-sm ${idx === 0 ? 'text-blue-100' : 'text-slate-500'}`}>{t.clientName}</p>
                                                </div>
                                            ))}
                                        </div>
                                    </div>
                                </div>
                            </div>
                        )}
                    </main>
                    <footer className="p-4 text-center text-slate-400 text-[10px] font-bold uppercase tracking-[0.5em]">
                        QueueFlow Local Hub • Sin Conexión a Internet Requerida
                    </footer>
                </div>
            );
        };

            const root = ReactDOM.createRoot(document.getElementById('root'));
            root.render(<App />);
    </script>
</body>
</html>
