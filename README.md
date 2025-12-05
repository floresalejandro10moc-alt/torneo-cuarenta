<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Torneo Cuarenta Pro</title>
    
    <!-- 1. Estilos y Fuentes -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap');
        body { font-family: 'Inter', sans-serif; }
        .animate-fade { animation: fadeIn 0.5s ease-out; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
    </style>

    <!-- 2. React y Babel -->
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
</head>
<body class="bg-slate-950 text-white selection:bg-blue-500 selection:text-white">
    <div id="root"></div>

    <!-- 3. Lógica de la Aplicación -->
    <script type="text/babel" data-type="module">
        // Importamos Firebase directamente desde Google
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-auth.js";
        import { 
            getFirestore, collection, doc, setDoc, onSnapshot, 
            updateDoc, deleteDoc, query, orderBy, serverTimestamp, 
            writeBatch, getDocs 
        } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-firestore.js";

        // ==========================================
        //  CONFIGURACIÓN CORREGIDA
        // ==========================================
        
        // Aquí estaba el error. He restaurado la línea "const firebaseConfig = {"
        // y he puesto tus claves dentro correctamente.
        const firebaseConfig = {
          apiKey: "AIzaSyAKge21Uy94wXdsygmqf9kbrxlaZm3H-r4",
          authDomain: "cuarenta-5af3b.firebaseapp.com",
          projectId: "cuarenta-5af3b",
          storageBucket: "cuarenta-5af3b.firebasestorage.app",
          messagingSenderId: "221307851852",
          appId: "1:221307851852:web:39bd3457135abf3f4a50e9",
          measurementId: "G-JWGB4DLZN0"
        };
        // ==========================================

        // Inicializar Firebase
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        
        // ID de la App (Fijo para que coincida con la ruta)
        const APP_ID = "cuarenta-app";

        // --- ÍCONOS (Para no depender de librerías externas) ---
        const IconTrophy = () => <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M6 9H4.5a2.5 2.5 0 0 1 0-5H6"/><path d="M18 9h1.5a2.5 2.5 0 0 0 0-5H18"/><path d="M4 22h16"/><path d="M10 14.66V17c0 .55-.47.98-.97 1.21C7.85 18.75 7 20.24 7 22"/><path d="M14 14.66V17c0 .55.47.98.97 1.21C16.15 18.75 17 20.24 17 22"/><path d="M18 2H6v7a6 6 0 0 0 12 0V2Z"/></svg>;
        const IconUsers = () => <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M22 21v-2a4 4 0 0 0-3-3.87"/><path d="M16 3.13a4 4 0 0 1 0 7.75"/></svg>;
        const IconTrash = () => <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/></svg>;
        const IconZap = () => <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><polygon points="13 2 3 14 12 14 11 22 21 10 12 10 13 2"/></svg>;
        
        // --- COMPONENTE PRINCIPAL ---
        const App = () => {
            const [tournamentId, setTournamentId] = React.useState('torneo-principal');
            const [userId, setUserId] = React.useState(null);
            const [view, setView] = React.useState('setup'); // setup, lobby, active
            
            const [teams, setTeams] = React.useState([]);
            const [matches, setMatches] = React.useState([]);
            const [status, setStatus] = React.useState({ state: 'registration', round: 1 });

            // Inputs
            const [teamName, setTeamName] = React.useState('');
            const [p1, setP1] = React.useState('');
            const [p2, setP2] = React.useState('');
            const [errorMsg, setErrorMsg] = React.useState('');

            // 1. Autenticación Anónima
            React.useEffect(() => {
                signInAnonymously(auth).catch(err => console.error("Error Auth", err));
                onAuthStateChanged(auth, (user) => {
                    if (user) setUserId(user.uid);
                });
            }, []);

            // 2. Escuchar cambios en tiempo real
            React.useEffect(() => {
                if (!userId || !tournamentId) return;

                // Definimos referencias
                const teamsRef = collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`);
                const matchesRef = collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`);
                const statusRef = doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_status`, 'info');

                // Suscripciones
                const unsubTeams = onSnapshot(query(teamsRef, orderBy('createdAt', 'desc')), snap => {
                    setTeams(snap.docs.map(d => ({ id: d.id, ...d.data() })));
                });

                const unsubMatches = onSnapshot(query(matchesRef, orderBy('matchNumber', 'asc')), snap => {
                    setMatches(snap.docs.map(d => ({ id: d.id, ...d.data() })));
                });

                const unsubStatus = onSnapshot(statusRef, snap => {
                    if (snap.exists()) {
                        setStatus(snap.data());
                        setView(snap.data().state === 'registration' ? 'lobby' : 'active');
                    } else {
                        // Crear si no existe
                        setDoc(statusRef, { state: 'registration', round: 1 });
                        setStatus({ state: 'registration', round: 1 });
                        setView('lobby');
                    }
                });

                return () => { unsubTeams(); unsubMatches(); unsubStatus(); };
            }, [userId, tournamentId]);

            // 3. Funciones
            const addTeam = async () => {
                if (!teamName || !p1 || !p2) { setErrorMsg("Llena todos los campos"); return; }
                setErrorMsg('');
                const teamsRef = collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`);
                await setDoc(doc(teamsRef), {
                    name: teamName, p1, p2,
                    createdAt: serverTimestamp()
                });
                setTeamName(''); setP1(''); setP2('');
            };

            const deleteTeam = async (id) => {
                await deleteDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`, id));
            };

            const shuffle = (array) => {
                let currentIndex = array.length, randomIndex;
                while (currentIndex != 0) {
                    randomIndex = Math.floor(Math.random() * currentIndex);
                    currentIndex--;
                    [array[currentIndex], array[randomIndex]] = [array[randomIndex], array[currentIndex]];
                }
                return array;
            };

            // INICIAR TORNEO (Ronda 1)
            const startTournament = async () => {
                if (teams.length < 2) return;
                const batch = writeBatch(db);
                const shuffled = shuffle([...teams]);
                let matchCount = 1;

                // Crear partidos
                for (let i = 0; i < shuffled.length; i += 2) {
                    const teamA = shuffled[i];
                    const teamB = shuffled[i+1];
                    const matchRef = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
                    
                    if (teamB) {
                        batch.set(matchRef, {
                            round: 1, matchNumber: matchCount++,
                            teamA, teamB, scoreA: 0, scoreB: 0,
                            winner: null, finishedAt: null
                        });
                    } else {
                        // Bye automático Ronda 1
                        batch.set(matchRef, {
                            round: 1, matchNumber: matchCount++,
                            teamA, teamB: null, scoreA: 2, scoreB: 0,
                            winner: teamA, finishedAt: serverTimestamp(),
                            isBye: true
                        });
                    }
                }

                // Actualizar estado
                const statusRef = doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_status`, 'info');
                batch.update(statusRef, { state: 'playing', round: 1 });
                await batch.commit();
            };

            // ACTUALIZAR PUNTAJE
            const updateScore = async (matchId, team, currentScore) => {
                const matchRef = doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`, matchId);
                const updates = {};
                const newScore = currentScore + 1;
                
                if (team === 'A') updates.scoreA = newScore;
                else updates.scoreB = newScore;

                if (newScore >= 2) {
                    const match = matches.find(m => m.id === matchId);
                    updates.winner = team === 'A' ? match.teamA : match.teamB;
                    updates.finishedAt = serverTimestamp(); // Marca de tiempo para desempate
                }
                await updateDoc(matchRef, updates);
            };

            // GENERAR SIGUIENTE RONDA (Lógica "El más rápido")
            const generateNextRound = async () => {
                const currentRoundMatches = matches.filter(m => m.round === status.round);
                if (currentRoundMatches.some(m => !m.winner)) { setErrorMsg("Aún hay partidas jugando"); return; }
                
                // Ordenar ganadores por tiempo (El menor timestamp = más rápido)
                const winners = currentRoundMatches
                    .map(m => ({ team: m.winner, time: m.finishedAt ? m.finishedAt.seconds : 9999999999 }))
                    .sort((a, b) => a.time - b.time);

                const nextRound = status.round + 1;
                const batch = writeBatch(db);
                let matchCount = 1;
                let pool = [...winners];

                // Si es impar, el primero (más rápido) pasa directo
                if (pool.length % 2 !== 0) {
                    const fastest = pool.shift();
                    const matchRef = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
                    batch.set(matchRef, {
                        round: nextRound, matchNumber: matchCount++,
                        teamA: fastest.team, teamB: null,
                        scoreA: 2, scoreB: 0, winner: fastest.team,
                        finishedAt: serverTimestamp(), isBye: true, note: "Pase por rapidez"
                    });
                }

                // El resto juega
                for (let i = 0; i < pool.length; i += 2) {
                    const p1 = pool[i];
                    const p2 = pool[i+1];
                    const matchRef = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
                    batch.set(matchRef, {
                        round: nextRound, matchNumber: matchCount++,
                        teamA: p1.team, teamB: p2.team,
                        scoreA: 0, scoreB: 0, winner: null, finishedAt: null
                    });
                }

                const statusRef = doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_status`, 'info');
                batch.update(statusRef, { round: nextRound });
                await batch.commit();
            };

            const resetTournament = async () => {
                if(!confirm("¿Borrar todo?")) return;
                const batch = writeBatch(db);
                // Borrar equipos y partidos (en una app real usaríamos funciones recursivas, aquí simplificado)
                teams.forEach(t => batch.delete(doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`, t.id)));
                matches.forEach(m => batch.delete(doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`, m.id)));
                batch.set(doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_status`, 'info'), { state: 'registration', round: 1 });
                await batch.commit();
            };

            // --- RENDERIZADO UI ---
            if (view === 'setup') {
                return (
                    <div className="min-h-screen flex items-center justify-center p-4">
                        <div className="bg-slate-900 p-8 rounded-xl border border-slate-800 shadow-2xl max-w-md w-full text-center">
                            <div className="bg-blue-600 w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-4"><IconTrophy /></div>
                            <h1 className="text-2xl font-bold mb-2">Cuarenta Pro</h1>
                            <p className="text-slate-400 mb-6 text-sm">Ingresa el mismo ID en todos los celulares.</p>
                            <input value={tournamentId} onChange={e => setTournamentId(e.target.value)} className="w-full bg-slate-950 border border-slate-700 p-3 rounded mb-4 text-center font-mono uppercase" />
                            <button onClick={() => setView('lobby')} className="w-full bg-blue-600 py-3 rounded font-bold hover:bg-blue-500 transition">Entrar a Sala</button>
                        </div>
                    </div>
                );
            }

            const currentMatches = matches.filter(m => m.round === status.round);
            const finishedCount = currentMatches.filter(m => m.winner).length;

            return (
                <div className="min-h-screen p-4 max-w-3xl mx-auto pb-20">
                    {/* Header */}
                    <div className="flex justify-between items-center mb-6 border-b border-slate-800 pb-4">
                        <div>
                            <h2 className="font-bold text-xl text-blue-400">Sala: {tournamentId}</h2>
                            {status.state === 'playing' && <span className="text-xs text-slate-500">Ronda {status.round}</span>}
                        </div>
                        <button onClick={resetTournament} className="text-red-500 hover:bg-red-900/20 p-2 rounded"><IconTrash /></button>
                    </div>

                    {/* VISTA REGISTRO */}
                    {status.state === 'registration' && (
                        <div className="animate-fade">
                            <div className="bg-slate-900 p-6 rounded-xl border border-slate-800 mb-6">
                                <h3 className="font-bold mb-4 flex items-center gap-2"><IconUsers /> Nuevo Equipo</h3>
                                <div className="space-y-3">
                                    <input placeholder="Nombre Equipo" className="w-full bg-slate-950 border border-slate-700 p-3 rounded" value={teamName} onChange={e => setTeamName(e.target.value)} />
                                    <div className="grid grid-cols-2 gap-3">
                                        <input placeholder="Jugador 1" className="bg-slate-950 border border-slate-700 p-3 rounded" value={p1} onChange={e => setP1(e.target.value)} />
                                        <input placeholder="Jugador 2" className="bg-slate-950 border border-slate-700 p-3 rounded" value={p2} onChange={e => setP2(e.target.value)} />
                                    </div>
                                    <button onClick={addTeam} className="w-full bg-green-600 py-3 rounded font-bold mt-2">Guardar</button>
                                    {errorMsg && <p className="text-red-400 text-xs text-center">{errorMsg}</p>}
                                </div>
                            </div>
                            
                            <div className="space-y-2">
                                <h4 className="text-xs text-slate-500 uppercase font-bold">Inscritos ({teams.length})</h4>
                                {teams.map(t => (
                                    <div key={t.id} className="bg-slate-800 p-3 rounded flex justify-between border border-slate-700">
                                        <div><div className="font-bold">{t.name}</div><div className="text-xs text-slate-400">{t.p1} & {t.p2}</div></div>
                                        <button onClick={() => deleteTeam(t.id)} className="text-red-400"><IconTrash /></button>
                                    </div>
                                ))}
                            </div>

                            {teams.length >= 2 && (
                                <button onClick={startTournament} className="fixed bottom-6 left-4 right-4 bg-blue-600 py-4 rounded-xl font-bold shadow-lg text-lg">Iniciar Torneo</button>
                            )}
                        </div>
                    )}

                    {/* VISTA JUEGO */}
                    {status.state === 'playing' && (
                        <div className="space-y-4 animate-fade">
                            {currentMatches.map(m => (
                                <div key={m.id} className={`bg-slate-800 border ${m.winner ? 'border-green-600' : 'border-slate-700'} p-4 rounded-xl relative overflow-hidden shadow-lg`}>
                                    {m.isBye ? (
                                        <div className="text-center py-2">
                                            <span className="text-orange-400 font-bold uppercase text-sm block mb-1">{m.note || 'Pase Directo'}</span>
                                            <h3 className="text-xl font-bold">{m.teamA.name}</h3>
                                            <p className="text-xs text-green-400 mt-2">Espera en la siguiente ronda</p>
                                        </div>
                                    ) : (
                                        <>
                                            <div className="flex justify-between items-center mb-4">
                                                <span className="text-xs font-mono text-slate-500">Mesa {m.matchNumber}</span>
                                                {m.winner && <span className="bg-green-500 text-black text-xs font-bold px-2 py-0.5 rounded">FINALIZADO</span>}
                                            </div>
                                            
                                            {/* Equipo A */}
                                            <div className={`flex justify-between items-center p-2 rounded ${m.winner?.name === m.teamA.name ? 'bg-green-900/20' : ''}`}>
                                                <div><div className="font-bold">{m.teamA.name}</div></div>
                                                <div className="flex gap-3 items-center">
                                                    <span className="text-2xl font-mono">{m.scoreA}</span>
                                                    {!m.winner && <button onClick={() => updateScore(m.id, 'A', m.scoreA)} className="bg-blue-600 w-8 h-8 rounded flex items-center justify-center font-bold">+</button>}
                                                </div>
                                            </div>
                                            <div className="h-px bg-slate-700 my-2"></div>
                                            {/* Equipo B */}
                                            <div className={`flex justify-between items-center p-2 rounded ${m.winner?.name === m.teamB.name ? 'bg-green-900/20' : ''}`}>
                                                <div><div className="font-bold">{m.teamB.name}</div></div>
                                                <div className="flex gap-3 items-center">
                                                    <span className="text-2xl font-mono">{m.scoreB}</span>
                                                    {!m.winner && <button onClick={() => updateScore(m.id, 'B', m.scoreB)} className="bg-blue-600 w-8 h-8 rounded flex items-center justify-center font-bold">+</button>}
                                                </div>
                                            </div>
                                        </>
                                    )}
                                </div>
                            ))}

                            {/* Panel Inferior */}
                            <div className="fixed bottom-0 left-0 right-0 bg-slate-900/90 backdrop-blur border-t border-slate-700 p-4">
                                <div className="max-w-3xl mx-auto flex justify-between items-center">
                                    <div className="text-sm">
                                        <span className="text-slate-400">Progreso:</span>
                                        <span className="ml-2 font-bold text-white">{finishedCount} / {currentMatches.length}</span>
                                    </div>
                                    {finishedCount === currentMatches.length && (
                                        <button onClick={generateNextRound} className="bg-orange-600 px-6 py-2 rounded-lg font-bold hover:bg-orange-500 animate-bounce flex items-center gap-2">
                                            <IconZap /> Siguiente Ronda
                                        </button>
                                    )}
                                </div>
                            </div>
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
