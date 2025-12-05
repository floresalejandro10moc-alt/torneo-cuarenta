<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Torneo Cuarenta Pro</title>
    
    <!-- 1. ESTILOS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.6.0/dist/confetti.browser.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700;900&display=swap');
        body { font-family: 'Inter', sans-serif; background-color: #020617; color: white; }
        .animate-fade { animation: fadeIn 0.5s ease-out; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
        .glass { background: rgba(30, 41, 59, 0.7); backdrop-filter: blur(10px); }
        /* Caja de error */
        #error-box { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: #000; z-index: 9999; padding: 20px; color: #ff5555; font-family: monospace; white-space: pre-wrap; overflow: auto; }
    </style>

    <!-- 2. LIBRERÍAS (Orden Importante) -->
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
</head>
<body>
    <!-- Caja para mostrar errores si algo falla -->
    <div id="error-box"></div>
    <div id="root"></div>

    <!-- Script de Seguridad (Atrapa errores) -->
    <script>
        window.onerror = function(message, source, lineno, colno, error) {
            const box = document.getElementById('error-box');
            box.style.display = 'block';
            box.innerHTML = `<h1>¡UPS! ALGO FALLÓ 😵</h1><hr/>Error: ${message}<br/>Línea: ${lineno}<br/><br/>Verifica que hayas creado la 'Firestore Database' en tu consola de Firebase.<br/>Si el error persiste, envía una captura de esto.`;
        };
    </script>

    <!-- 3. TU APLICACIÓN -->
    <script type="text/babel" data-type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-auth.js";
        import { 
            getFirestore, collection, doc, setDoc, onSnapshot, 
            updateDoc, deleteDoc, query, orderBy, serverTimestamp, 
            writeBatch, getDocs, where 
        } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-firestore.js";

        // --- TUS CLAVES (Ya las puse yo) ---
        const firebaseConfig = {
          apiKey: "AIzaSyAKge21Uy94wXdsygmqf9kbrxlaZm3H-r4",
          authDomain: "cuarenta-5af3b.firebaseapp.com",
          projectId: "cuarenta-5af3b",
          storageBucket: "cuarenta-5af3b.firebasestorage.app",
          messagingSenderId: "221307851852",
          appId: "1:221307851852:web:39bd3457135abf3f4a50e9",
          measurementId: "G-JWGB4DLZN0"
        };

        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const APP_ID = "cuarenta-app-v2";

        // --- ÍCONOS ---
        const IconTrophy = () => <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2"><path d="M6 9H4.5a2.5 2.5 0 0 1 0-5H6"/><path d="M18 9h1.5a2.5 2.5 0 0 0 0-5H18"/><path d="M4 22h16"/><path d="M10 14.66V17c0 .55-.47.98-.97 1.21C7.85 18.75 7 20.24 7 22"/><path d="M14 14.66V17c0 .55.47.98.97 1.21C16.15 18.75 17 20.24 17 22"/><path d="M18 2H6v7a6 6 0 0 0 12 0V2Z"/></svg>;
        const IconUsers = () => <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2"><path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M22 21v-2a4 4 0 0 0-3-3.87"/><path d="M16 3.13a4 4 0 0 1 0 7.75"/></svg>;
        const IconTrash = () => <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/></svg>;
        const IconCrown = () => <svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" className="text-yellow-400"><path d="M2 4l3 12h14l3-12-6 7-4-7-4 7-6-7zm3 16h14v2H5z"/></svg>;

        const App = () => {
            const [tournamentId, setTournamentId] = React.useState('torneo-principal');
            const [userId, setUserId] = React.useState(null);
            const [view, setView] = React.useState('setup'); 
            const [tab, setTab] = React.useState('main'); 
            
            const [teams, setTeams] = React.useState([]);
            const [matches, setMatches] = React.useState([]); 
            const [consolationMatches, setConsolationMatches] = React.useState([]); 
            const [status, setStatus] = React.useState({ state: 'registration', round: 1, roundName: 'Ronda 1' });
            const [champion, setChampion] = React.useState(null);

            const [teamName, setTeamName] = React.useState('');
            const [p1, setP1] = React.useState('');
            const [p2, setP2] = React.useState('');

            // Autenticación
            React.useEffect(() => {
                signInAnonymously(auth).catch(err => console.error("Error Auth:", err));
                onAuthStateChanged(auth, (u) => u && setUserId(u.uid));
            }, []);

            // Escuchar cambios
            React.useEffect(() => {
                if (!userId || !tournamentId) return;

                const publicRef = collection(db, 'artifacts', APP_ID, 'public', 'data');
                
                const unsubTeams = onSnapshot(query(collection(publicRef, `${tournamentId}_teams`), orderBy('createdAt', 'desc')), 
                    s => setTeams(s.docs.map(d => ({id: d.id, ...d.data()}))),
                    err => console.error("Error Teams:", err)
                );

                const unsubMatches = onSnapshot(query(collection(publicRef, `${tournamentId}_matches`), orderBy('matchNumber', 'asc')), 
                    s => {
                        const all = s.docs.map(d => ({id: d.id, ...d.data()}));
                        setMatches(all.filter(m => !m.isConsolation));
                        setConsolationMatches(all.filter(m => m.isConsolation));
                        
                        const finalMatch = all.find(m => m.roundName === 'GRAN FINAL' && m.winner);
                        if (finalMatch) {
                            setChampion(finalMatch.winner);
                            try { confetti({ particleCount: 150, spread: 70, origin: { y: 0.6 } }); } catch(e){}
                        } else {
                            setChampion(null);
                        }
                    },
                    err => console.error("Error Matches:", err)
                );

                const unsubStatus = onSnapshot(doc(publicRef, `${tournamentId}_status`, 'info'), s => {
                    if (s.exists()) {
                        setStatus(s.data());
                        setView(s.data().state === 'registration' ? 'lobby' : 'active');
                    } else {
                        setDoc(doc(publicRef, `${tournamentId}_status`, 'info'), { state: 'registration', round: 1 });
                    }
                }, err => console.error("Error Status:", err));

                return () => { unsubTeams(); unsubMatches(); unsubStatus(); };
            }, [userId, tournamentId]);

            const addTeam = async () => {
                if (!teamName) return;
                const ref = collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`);
                await setDoc(doc(ref), { name: teamName, p1, p2, createdAt: serverTimestamp() });
                setTeamName(''); setP1(''); setP2('');
            };

            const deleteTeam = async (id) => deleteDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`, id));

            const shuffle = (arr) => arr.sort(() => Math.random() - 0.5);

            const startTournament = async () => {
                if (teams.length < 2) return;
                
                const batch = writeBatch(db);
                const shuffledTeams = shuffle([...teams]);
                const N = shuffledTeams.length;
                
                let powerOf2 = 1;
                while (powerOf2 * 2 <= N) powerOf2 *= 2; 
                
                // Si es potencia exacta (ej: 4, 8, 16), todos juegan. Si no, ajustamos.
                let numMatches = 0;
                let numByes = 0;

                if (powerOf2 === N) {
                    numMatches = N / 2;
                    numByes = 0;
                } else {
                    // Diferencia para llegar a la potencia anterior
                    // Ej: N=6. Potencia=4. Sobran 2.
                    // Partidos preliminares = N - Potencia (6-4 = 2 partidos).
                    // Byes = Potencia - Partidos (4-2 = 2 byes).
                    numMatches = N - powerOf2;
                    numByes = powerOf2 - numMatches;
                }

                let matchCount = 1;
                let teamIdx = 0;

                // 1. Partidos Preliminares
                for (let i = 0; i < numMatches; i++) {
                    const tA = shuffledTeams[teamIdx++];
                    const tB = shuffledTeams[teamIdx++];
                    const ref = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
                    batch.set(ref, {
                        round: 1, roundName: "Clasificatoria", matchNumber: matchCount++,
                        teamA: tA, teamB: tB, scoreA: 0, scoreB: 0, winner: null,
                        isConsolation: false
                    });
                }

                // 2. Byes
                for (let i = 0; i < numByes; i++) {
                    const t = shuffledTeams[teamIdx++];
                    const ref = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
                    batch.set(ref, {
                        round: 1, roundName: "Clasificatoria", matchNumber: matchCount++,
                        teamA: t, teamB: null, scoreA: 2, scoreB: 0, winner: t,
                        isBye: true, note: "Pase a Siguiente Ronda",
                        isConsolation: false
                    });
                }

                const statusRef = doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_status`, 'info');
                batch.update(statusRef, { state: 'playing', round: 1, roundName: 'Clasificatoria' });
                await batch.commit();
            };

            const updateScore = async (match, team) => {
                if (match.winner) return; 
                
                const ref = doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`, match.id);
                const updates = {};
                const newScore = (team === 'A' ? match.scoreA : match.scoreB) + 1;
                
                if (team === 'A') updates.scoreA = newScore;
                else updates.scoreB = newScore;

                if (newScore >= 2) {
                    updates.winner = team === 'A' ? match.teamA : match.teamB;
                    updates.finishedAt = serverTimestamp();
                    
                    if (!match.isConsolation && !match.isBye) {
                        const loser = team === 'A' ? match.teamB : match.teamA;
                        addLoserToPool(loser);
                    }
                }
                await updateDoc(ref, updates);
            };

            const addLoserToPool = async (team) => {
                 const ref = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_losers`));
                 await setDoc(ref, { team, available: true });
            };

            const generateNextRound = async () => {
                const currentRoundM = matches.filter(m => m.round === status.round);
                if (currentRoundM.some(m => !m.winner)) return alert("Faltan partidos por terminar");

                const winners = currentRoundM.map(m => m.winner);
                if (winners.length === 1) return alert("¡Ya hay campeón!");

                const nextRound = status.round + 1;
                let roundName = `Ronda ${nextRound}`;
                if (winners.length === 2) roundName = "GRAN FINAL";
                else if (winners.length === 4) roundName = "Semifinales";
                else if (winners.length === 8) roundName = "Cuartos de Final";

                const batch = writeBatch(db);
                let matchCount = 1;
                for (let i = 0; i < winners.length; i += 2) {
                    const tA = winners[i];
                    const tB = winners[i+1];
                    const ref = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
                    batch.set(ref, {
                        round: nextRound, roundName, matchNumber: matchCount++,
                        teamA: tA, teamB: tB, scoreA: 0, scoreB: 0, winner: null,
                        isConsolation: false
                    });
                }

                const statusRef = doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_status`, 'info');
                batch.update(statusRef, { round: nextRound, roundName });
                await batch.commit();
            };

            const generateConsolationRound = async () => {
                const losersSnap = await getDocs(query(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_losers`), where('available', '==', true)));
                const losersDocs = losersSnap.docs;

                if (losersDocs.length < 2) return alert("No hay suficientes perdedores aún.");

                const batch = writeBatch(db);
                // Barajar perdedores
                const shuffledDocs = losersDocs.sort(() => Math.random() - 0.5);
                
                // Solo tomar pares
                const pairCount = Math.floor(shuffledDocs.length / 2) * 2;

                for (let i = 0; i < pairCount; i += 2) {
                    const docA = shuffledDocs[i];
                    const docB = shuffledDocs[i+1];
                    const tA = docA.data().team;
                    const tB = docB.data().team;

                    const ref = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
                    batch.set(ref, {
                        round: 99, roundName: "Repechaje", matchNumber: Date.now(),
                        teamA: tA, teamB: tB, scoreA: 0, scoreB: 0, winner: null,
                        isConsolation: true
                    });
                    
                    batch.update(docA.ref, { available: false });
                    batch.update(docB.ref, { available: false });
                }
                await batch.commit();
            };
            
            const nuke = async () => {
                if(!confirm("¿BORRAR TODO? Se reiniciará el torneo.")) return;
                const batch = writeBatch(db);
                const t = await getDocs(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`));
                t.forEach(d => batch.delete(d.ref));
                const m = await getDocs(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
                m.forEach(d => batch.delete(d.ref));
                const l = await getDocs(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_losers`));
                l.forEach(d => batch.delete(d.ref));
                
                batch.set(doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_status`, 'info'), { state: 'registration', round: 1 });
                await batch.commit();
            };

            // VISTAS
            if (view === 'setup') return (
                <div className="min-h-screen flex items-center justify-center p-4 bg-slate-950">
                    <div className="bg-slate-900 p-8 rounded-xl border border-slate-800 shadow-2xl max-w-sm w-full text-center">
                        <div className="mb-4 inline-block p-4 rounded-full bg-blue-600"><IconTrophy /></div>
                        <h1 className="text-2xl font-bold mb-2">Cuarenta Pro</h1>
                        <input value={tournamentId} onChange={e => setTournamentId(e.target.value)} className="w-full bg-slate-950 border border-slate-700 p-3 rounded mb-4 text-center text-white font-mono" />
                        <button onClick={() => setView('lobby')} className="w-full bg-blue-600 py-3 rounded font-bold hover:bg-blue-500 transition">Entrar</button>
                    </div>
                </div>
            );

            if (champion) return (
                <div className="min-h-screen flex flex-col items-center justify-center bg-slate-950 p-4 text-center relative overflow-hidden">
                    <div className="absolute inset-0 bg-gradient-to-t from-blue-900/40 to-transparent"></div>
                    <div className="z-10 animate-fade">
                        <div className="text-yellow-400 mb-4 flex justify-center"><IconCrown /></div>
                        <h1 className="text-6xl font-black text-transparent bg-clip-text bg-gradient-to-r from-yellow-300 to-amber-500 mb-4">¡CAMPEONES!</h1>
                        <div className="bg-slate-900/80 p-8 rounded-2xl border border-yellow-500/50 shadow-yellow-500/20 shadow-2xl">
                            <h2 className="text-4xl font-bold text-white mb-2">{champion.name}</h2>
                            <p className="text-xl text-slate-400">{champion.p1} & {champion.p2}</p>
                        </div>
                        <button onClick={nuke} className="mt-12 text-slate-500 hover:text-white underline cursor-pointer">Reiniciar Torneo</button>
                    </div>
                </div>
            );

            return (
                <div className="min-h-screen p-4 max-w-4xl mx-auto pb-24">
                    <div className="flex justify-between items-center mb-6 border-b border-slate-800 pb-4">
                        <div>
                            <h2 className="font-bold text-xl text-blue-400 uppercase tracking-wider">{tournamentId}</h2>
                            {status.state === 'playing' && <span className="bg-blue-900/50 text-blue-200 px-2 py-1 rounded text-xs font-bold">{status.roundName}</span>}
                        </div>
                        <button onClick={nuke} className="text-red-500 p-2 hover:bg-red-900/20 rounded"><IconTrash /></button>
                    </div>

                    {status.state === 'registration' && (
                        <div className="animate-fade">
                            <div className="glass p-6 rounded-xl border border-slate-700 mb-6">
                                <h3 className="font-bold mb-4 flex items-center gap-2"><IconUsers /> Registrar Equipo</h3>
                                <div className="space-y-3">
                                    <input placeholder="Nombre Equipo" className="w-full bg-slate-950 border border-slate-600 p-3 rounded text-white" value={teamName} onChange={e => setTeamName(e.target.value)} />
                                    <div className="grid grid-cols-2 gap-3">
                                        <input placeholder="Jugador 1" className="bg-slate-950 border border-slate-600 p-3 rounded text-white" value={p1} onChange={e => setP1(e.target.value)} />
                                        <input placeholder="Jugador 2" className="bg-slate-950 border border-slate-600 p-3 rounded text-white" value={p2} onChange={e => setP2(e.target.value)} />
                                    </div>
                                    <button onClick={addTeam} className="w-full bg-green-600 hover:bg-green-500 py-3 rounded font-bold mt-2 transition">Guardar</button>
                                </div>
                            </div>
                            <div className="grid gap-2">
                                {teams.map(t => (
                                    <div key={t.id} className="bg-slate-800 p-3 rounded flex justify-between items-center border border-slate-700">
                                        <div><div className="font-bold">{t.name}</div><div className="text-xs text-slate-400">{t.p1}/{t.p2}</div></div>
                                        <button onClick={() => deleteTeam(t.id)} className="text-red-400"><IconTrash /></button>
                                    </div>
                                ))}
                            </div>
                            {teams.length >= 2 && <button onClick={startTournament} className="fixed bottom-6 left-4 right-4 bg-blue-600 py-4 rounded-xl font-bold shadow-lg text-lg animate-bounce">¡Iniciar Torneo!</button>}
                        </div>
                    )}

                    {status.state === 'playing' && (
                        <div className="animate-fade">
                            <div className="flex gap-2 mb-6">
                                <button onClick={() => setTab('main')} className={`flex-1 py-2 rounded-lg font-bold transition ${tab === 'main' ? 'bg-blue-600 text-white' : 'bg-slate-800 text-slate-400'}`}>🏆 Principal</button>
                                <button onClick={() => setTab('losers')} className={`flex-1 py-2 rounded-lg font-bold transition ${tab === 'losers' ? 'bg-orange-600 text-white' : 'bg-slate-800 text-slate-400'}`}>🛡️ Repechaje</button>
                            </div>

                            {tab === 'main' && (
                                <div className="space-y-4">
                                    {matches.filter(m => m.round === status.round).map(m => (
                                        <div key={m.id} className={`bg-slate-800 border ${m.winner ? 'border-green-600' : 'border-slate-600'} p-4 rounded-xl relative overflow-hidden shadow-lg`}>
                                            {m.isBye ? (
                                                <div className="text-center py-2 opacity-70">
                                                    <span className="text-green-400 font-bold uppercase text-xs block mb-1">Pase Directo</span>
                                                    <h3 className="text-xl font-bold">{m.teamA.name}</h3>
                                                </div>
                                            ) : (
                                                <>
                                                    <div className="flex justify-between items-center mb-3">
                                                        <span className="text-xs text-slate-500 font-mono">Mesa {m.matchNumber}</span>
                                                        {m.winner && <span className="text-green-400 text-xs font-bold">TERMINADO</span>}
                                                    </div>
                                                    <div className={`flex justify-between items-center p-2 rounded ${m.winner?.name === m.teamA.name ? 'bg-green-900/30' : ''}`}>
                                                        <span className="font-bold truncate w-32">{m.teamA.name}</span>
                                                        <div className="flex items-center gap-3">
                                                            <span className="text-2xl font-mono">{m.scoreA}</span>
                                                            {!m.winner && <button onClick={() => updateScore(m, 'A')} className="bg-blue-600 w-8 h-8 rounded flex items-center justify-center font-bold">+</button>}
                                                        </div>
                                                    </div>
                                                    <div className="h-px bg-slate-700 my-2"></div>
                                                    <div className={`flex justify-between items-center p-2 rounded ${m.winner?.name === m.teamB.name ? 'bg-green-900/30' : ''}`}>
                                                        <span className="font-bold truncate w-32">{m.teamB.name}</span>
                                                        <div className="flex items-center gap-3">
                                                            <span className="text-2xl font-mono">{m.scoreB}</span>
                                                            {!m.winner && <button onClick={() => updateScore(m, 'B')} className="bg-blue-600 w-8 h-8 rounded flex items-center justify-center font-bold">+</button>}
                                                        </div>
                                                    </div>
                                                </>
                                            )}
                                        </div>
                                    ))}
                                    <div className="fixed bottom-0 left-0 right-0 bg-slate-900/90 backdrop-blur border-t border-slate-700 p-4">
                                        <div className="max-w-4xl mx-auto flex justify-between items-center">
                                            <div className="text-xs text-slate-400">
                                                {matches.filter(m => m.round === status.round && m.winner).length} / {matches.filter(m => m.round === status.round).length} listos
                                            </div>
                                            <button onClick={generateNextRound} className="bg-orange-500 hover:bg-orange-600 px-6 py-2 rounded-lg font-bold shadow-lg transition-transform active:scale-95">Siguiente Ronda ➔</button>
                                        </div>
                                    </div>
                                </div>
                            )}

                            {tab === 'losers' && (
                                <div className="space-y-4">
                                    <div className="bg-orange-900/20 p-4 rounded-lg border border-orange-700/50 mb-4">
                                        <h3 className="font-bold text-orange-400">Zona de Perdedores</h3>
                                        <p className="text-xs text-slate-400">Aquí juegan por honor los que cayeron en el torneo principal.</p>
                                        <button onClick={generateConsolationRound} className="mt-3 bg-orange-700 text-xs px-3 py-2 rounded font-bold hover:bg-orange-600">+ Armar Nueva Mesa</button>
                                    </div>
                                    {consolationMatches.length === 0 ? (
                                        <p className="text-center text-slate-600 py-8">Aún no hay partidas de consuelo.</p>
                                    ) : (
                                        consolationMatches.slice().reverse().map(m => (
                                            <div key={m.id} className="bg-slate-800 border border-slate-700 p-4 rounded-lg opacity-80">
                                                <div className="text-xs text-orange-500 font-bold mb-2">Mesa Amistosa</div>
                                                <div className="flex justify-between items-center mb-2">
                                                    <span>{m.teamA.name}</span>
                                                    <div className="flex gap-2">
                                                        <span className="font-mono">{m.scoreA}</span>
                                                        {!m.winner && <button onClick={() => updateScore(m, 'A')} className="bg-orange-700 w-6 h-6 rounded text-xs">+</button>}
                                                    </div>
                                                </div>
                                                <div className="flex justify-between items-center">
                                                    <span>{m.teamB.name}</span>
                                                    <div className="flex gap-2">
                                                        <span className="font-mono">{m.scoreB}</span>
                                                        {!m.winner && <button onClick={() => updateScore(m, 'B')} className="bg-orange-700 w-6 h-6 rounded text-xs">+</button>}
                                                    </div>
                                                </div>
                                                {m.winner && <div className="mt-2 text-center text-xs text-green-400 font-bold">Ganó {m.winner.name}</div>}
                                            </div>
                                        ))
                                    )}
                                </div>
                            )}
                        </div>
                    )}
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
