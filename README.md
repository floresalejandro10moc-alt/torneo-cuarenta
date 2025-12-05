<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Torneo de Cuarenta - Árbol</title>
    
    <!-- 1. Cargar React y ReactDOM desde CDN -->
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    
    <!-- 2. Cargar Babel para entender JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
    <!-- 3. Cargar Tailwind CSS para el diseño -->
    <script src="https://cdn.tailwindcss.com"></script>

    <style>
        /* Animaciones suaves */
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
        .animate-fade-in { animation: fadeIn 0.5s ease-out forwards; }
    </style>
</head>
<body class="bg-slate-900 text-slate-100">
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect } = React;

        // --- ICONOS SVG (Para no depender de librerías externas) ---
        const IconTrophy = () => <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M6 9H4.5a2.5 2.5 0 0 1 0-5H6"/><path d="M18 9h1.5a2.5 2.5 0 0 0 0-5H18"/><path d="M4 22h16"/><path d="M10 14.66V17c0 .55-.47.98-.97 1.21C7.85 18.75 7 20.24 7 22"/><path d="M14 14.66V17c0 .55.47.98.97 1.21C16.15 18.75 17 20.24 17 22"/><path d="M18 2H6v7a6 6 0 0 0 12 0V2Z"/></svg>;
        const IconTrash = () => <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/></svg>;
        const IconSave = () => <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"/><polyline points="17 21 17 13 7 13 7 21"/><polyline points="7 3 7 8 15 8"/></svg>;
        const IconShield = () => <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12 22s8-4 8-10V5l-8-3-8 3v7c0 6 8 10 8 10z"/></svg>;
        const IconSwords = () => <svg xmlns="http://www.w3.org/2000/svg" width="28" height="28" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><polyline points="14.5 17.5 3 6 3 3 6 3 17.5 14.5"/><line x1="13" y1="19" x2="19" y2="13"/><line x1="16" y1="16" x2="20" y2="20"/><line x1="19" y1="21" x2="21" y2="19"/></svg>;

        // --- APP PRINCIPAL ---
        const App = () => {
            // Estados
            const [teams, setTeams] = useState([]);
            const [newTeamName, setNewTeamName] = useState('');
            const [bracket, setBracket] = useState([]); 
            const [consolationBracket, setConsolationBracket] = useState([]);
            const [isTournamentActive, setIsTournamentActive] = useState(false);
            const [view, setView] = useState('registration'); 

            // Persistencia (Guardar al recargar)
            useEffect(() => {
                const savedData = localStorage.getItem('cuarentaTreeDataGH');
                if (savedData) {
                    try {
                        const parsed = JSON.parse(savedData);
                        setTeams(parsed.teams || []);
                        setBracket(parsed.bracket || []);
                        setConsolationBracket(parsed.consolationBracket || []);
                        setIsTournamentActive(parsed.isTournamentActive || false);
                        if (parsed.isTournamentActive) setView('main');
                    } catch (e) { console.error("Error cargando datos", e); }
                }
            }, []);

            useEffect(() => {
                const data = { teams, bracket, consolationBracket, isTournamentActive };
                localStorage.setItem('cuarentaTreeDataGH', JSON.stringify(data));
            }, [teams, bracket, consolationBracket, isTournamentActive]);

            // Gestión de equipos
            const addTeam = () => {
                if (!newTeamName.trim()) return;
                const newTeam = {
                    id: Date.now() + Math.random(),
                    name: newTeamName,
                };
                setTeams([...teams, newTeam]);
                setNewTeamName('');
            };

            const removeTeam = (id) => {
                setTeams(teams.filter(t => t.id !== id));
            };

            const resetAll = () => {
                if (window.confirm('¿Borrar todo y empezar de cero?')) {
                    setTeams([]);
                    setBracket([]);
                    setConsolationBracket([]);
                    setIsTournamentActive(false);
                    setView('registration');
                    localStorage.removeItem('cuarentaTreeDataGH');
                }
            };

            // Lógica del Torneo
            const shuffle = (array) => {
                let currentIndex = array.length, randomIndex;
                while (currentIndex !== 0) {
                    randomIndex = Math.floor(Math.random() * currentIndex);
                    currentIndex--;
                    [array[currentIndex], array[randomIndex]] = [array[randomIndex], array[currentIndex]];
                }
                return array;
            };

            const startTournament = () => {
                if (teams.length < 2) {
                    alert("Mínimo 2 equipos.");
                    return;
                }

                let powerOfTwo = 2;
                while (powerOfTwo < teams.length) powerOfTwo *= 2;

                let bracketTeams = [...teams];
                const totalSlots = powerOfTwo;
                const byesNeeded = totalSlots - teams.length;

                for (let i = 0; i < byesNeeded; i++) {
                    bracketTeams.push({ id: `bye-${i}`, name: 'PASE DIRECTO', isBye: true });
                }

                bracketTeams = shuffle(bracketTeams);
                const newBracket = generateTreeStructure(totalSlots, bracketTeams);
                const resolvedBracket = autoAdvanceByes(newBracket);

                setBracket(resolvedBracket);
                setConsolationBracket([]);
                setIsTournamentActive(true);
                setView('main');
            };

            const generateTreeStructure = (totalSlots, initialTeams) => {
                const rounds = [];
                let matchCount = totalSlots / 2;
                let currentTeams = initialTeams;

                // Ronda 1
                const round1Matches = [];
                for (let i = 0; i < matchCount; i++) {
                    round1Matches.push({
                        id: `R0-M${i}`,
                        roundIndex: 0,
                        matchIndex: i,
                        teamA: currentTeams[i * 2],
                        teamB: currentTeams[i * 2 + 1],
                        scoreA: 0,
                        scoreB: 0,
                        winner: null
                    });
                }
                rounds.push(round1Matches);

                // Rondas Siguientes
                let roundIndex = 1;
                while (matchCount > 1) {
                    matchCount = matchCount / 2;
                    const roundMatches = [];
                    for (let i = 0; i < matchCount; i++) {
                        roundMatches.push({
                            id: `R${roundIndex}-M${i}`,
                            roundIndex: roundIndex,
                            matchIndex: i,
                            teamA: null, teamB: null, scoreA: 0, scoreB: 0, winner: null
                        });
                    }
                    rounds.push(roundMatches);
                    roundIndex++;
                }
                return rounds;
            };

            const autoAdvanceByes = (currentBracket) => {
                let newBracket = JSON.parse(JSON.stringify(currentBracket));
                newBracket[0].forEach((match, idx) => {
                    if (match.teamA.isBye && !match.teamB.isBye) handleWin(newBracket, 0, idx, match.teamB);
                    else if (!match.teamA.isBye && match.teamB.isBye) handleWin(newBracket, 0, idx, match.teamA);
                });
                return newBracket;
            };

            const updateScore = (roundIndex, matchIndex, team, increment) => {
                const isMain = view === 'main';
                const currentBracketState = isMain ? bracket : consolationBracket;
                let newBracket = JSON.parse(JSON.stringify(currentBracketState));
                
                const match = newBracket[roundIndex][matchIndex];
                if (match.winner) return;

                if (team === 'A') match.scoreA += increment;
                else match.scoreB += increment;

                if (match.scoreA >= 2) handleWin(newBracket, roundIndex, matchIndex, match.teamA, isMain);
                else if (match.scoreB >= 2) handleWin(newBracket, roundIndex, matchIndex, match.teamB, isMain);
                else {
                    if (isMain) setBracket(newBracket);
                    else setConsolationBracket(newBracket);
                }
            };

            const handleWin = (bracketState, roundIndex, matchIndex, winnerTeam, isMain = true) => {
                const match = bracketState[roundIndex][matchIndex];
                match.winner = winnerTeam;
                
                if (isMain && roundIndex === 0) {
                    const loser = winnerTeam.id === match.teamA.id ? match.teamB : match.teamA;
                    if (!loser.isBye) addToConsolation(loser);
                }

                const nextRoundIndex = roundIndex + 1;
                if (nextRoundIndex < bracketState.length) {
                    const nextMatchIndex = Math.floor(matchIndex / 2);
                    const isTopPosition = matchIndex % 2 === 0;
                    const nextMatch = bracketState[nextRoundIndex][nextMatchIndex];
                    
                    if (isTopPosition) nextMatch.teamA = winnerTeam;
                    else nextMatch.teamB = winnerTeam;
                    
                    if (nextMatch.teamA?.isBye && nextMatch.teamB) handleWin(bracketState, nextRoundIndex, nextMatchIndex, nextMatch.teamB, isMain);
                    if (nextMatch.teamB?.isBye && nextMatch.teamA) handleWin(bracketState, nextRoundIndex, nextMatchIndex, nextMatch.teamA, isMain);
                }

                if (isMain) setBracket(bracketState);
                else setConsolationBracket(bracketState);
            };

            const addToConsolation = (team) => {
                setConsolationBracket(prev => {
                    const newBracket = [...prev];
                    let lastRound = newBracket.length > 0 ? newBracket[0] : [];
                    const waitingMatch = lastRound.find(m => m.teamA && !m.teamB && !m.winner);

                    if (waitingMatch) {
                        waitingMatch.teamB = team;
                    } else {
                        const newMatch = {
                            id: `C-M${Date.now()}`,
                            teamA: team, teamB: null, scoreA: 0, scoreB: 0, winner: null
                        };
                        if (newBracket.length === 0) newBracket.push([newMatch]);
                        else newBracket[0].push(newMatch);
                    }
                    return newBracket;
                });
            };

            // --- RENDERIZADO VISUAL ---
            const renderMatchCard = (match, roundIdx, matchIdx) => {
                if (!match.teamA && !match.teamB) {
                    return (
                        <div key={matchIdx} className="bg-slate-800/50 border border-slate-700 p-3 rounded-lg w-64 h-32 flex items-center justify-center mb-4">
                            <span className="text-slate-500 text-sm">Esperando rivales...</span>
                        </div>
                    );
                }
                const isFinished = !!match.winner;
                return (
                    <div key={matchIdx} className={`relative bg-slate-800 border ${isFinished ? 'border-green-600' : 'border-slate-600'} p-3 rounded-lg w-64 mb-4 shadow-lg flex flex-col justify-center`}>
                        {['A', 'B'].map(side => {
                            const team = side === 'A' ? match.teamA : match.teamB;
                            const score = side === 'A' ? match.scoreA : match.scoreB;
                            const isWinner = match.winner?.id === team?.id;
                            
                            return (
                                <div key={side}>
                                    <div className={`flex justify-between items-center p-2 rounded ${isWinner ? 'bg-green-900/40' : ''}`}>
                                        <span className={`font-bold truncate w-32 ${team?.isBye ? 'text-slate-500 italic' : 'text-white'}`}>
                                            {team ? team.name : '...'}
                                        </span>
                                        {team && !team.isBye && (
                                            <div className="flex items-center gap-2">
                                                <span className="font-mono text-xl">{score}</span>
                                                {!isFinished && <button onClick={() => updateScore(roundIdx, matchIdx, side, 1)} className="bg-blue-600 w-6 h-6 rounded text-xs flex items-center justify-center hover:bg-blue-500 text-white font-bold">+</button>}
                                            </div>
                                        )}
                                    </div>
                                    {side === 'A' && <div className="h-px bg-slate-700 my-1 w-full"></div>}
                                </div>
                            )
                        })}
                        <div className="absolute -right-4 top-1/2 w-4 h-0.5 bg-slate-600 hidden md:block"></div>
                    </div>
                );
            };

            return (
                <div className="min-h-screen p-4 md:p-8 overflow-x-hidden font-sans">
                    <div className="max-w-[95%] mx-auto">
                        {/* HEADER */}
                        <header className="flex flex-col md:flex-row justify-between items-center mb-6 border-b border-slate-700 pb-4 gap-4">
                            <div className="flex items-center gap-3">
                                <div className="bg-gradient-to-br from-indigo-600 to-purple-600 p-3 rounded-lg shadow-lg text-white"><IconSwords /></div>
                                <div>
                                    <h1 className="text-2xl md:text-3xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-blue-400 to-purple-400">Torneo Cuarenta</h1>
                                    <p className="text-slate-400 text-xs">Árbol Directo • Mejor de 2 mesas</p>
                                </div>
                            </div>
                            <div className="flex items-center gap-4">
                                {isTournamentActive && (
                                    <div className="bg-slate-800 rounded-lg p-1 flex">
                                        <button onClick={() => setView('main')} className={`px-4 py-2 rounded flex gap-2 items-center ${view === 'main' ? 'bg-blue-600 text-white' : 'text-slate-400'}`}>
                                            <IconTrophy /> Principal
                                        </button>
                                        <button onClick={() => setView('consolation')} className={`px-4 py-2 rounded flex gap-2 items-center ${view === 'consolation' ? 'bg-orange-600 text-white' : 'text-slate-400'}`}>
                                            <IconShield /> Consuelo
                                        </button>
                                    </div>
                                )}
                                <button onClick={resetAll} className="text-red-400 hover:bg-red-900/20 p-2 rounded transition"><IconTrash /></button>
                            </div>
                        </header>

                        {/* REGISTRO */}
                        {!isTournamentActive && (
                            <div className="max-w-xl mx-auto space-y-6 animate-fade-in">
                                <div className="bg-slate-800 p-6 rounded-xl border border-slate-700 shadow-xl">
                                    <h2 className="text-xl font-semibold mb-4 text-blue-300">Inscribir Pareja</h2>
                                    <div className="flex gap-2 mb-4">
                                        <input type="text" placeholder="Nombre Equipo (Ej. Los Shyris)" className="flex-1 bg-slate-900 border border-slate-600 rounded p-3 text-white focus:border-blue-500 outline-none" value={newTeamName} onChange={(e) => setNewTeamName(e.target.value)} onKeyDown={(e) => e.key === 'Enter' && addTeam()} />
                                        <button onClick={addTeam} className="bg-blue-600 hover:bg-blue-500 text-white p-3 rounded flex items-center gap-2"><IconSave /> Añadir</button>
                                    </div>
                                    <div className="max-h-60 overflow-y-auto space-y-2">
                                        {teams.map(t => (
                                            <div key={t.id} className="bg-slate-700/50 p-2 rounded flex justify-between items-center px-4">
                                                <span className="font-bold">{t.name}</span>
                                                <button onClick={() => removeTeam(t.id)} className="text-red-400 hover:text-red-300"><IconTrash /></button>
                                            </div>
                                        ))}
                                        {teams.length === 0 && <p className="text-center text-slate-500 py-4">No hay equipos aún.</p>}
                                    </div>
                                </div>
                                <div className="flex justify-center">
                                    <button onClick={startTournament} disabled={teams.length < 2} className={`font-bold py-4 px-12 rounded-full shadow-lg transform hover:scale-105 transition ${teams.length < 2 ? 'bg-slate-700 text-slate-500' : 'bg-green-600 hover:bg-green-500 text-white'}`}>
                                        Generar Llaves y Jugar
                                    </button>
                                </div>
                            </div>
                        )}

                        {/* BRACKET PRINCIPAL */}
                        {isTournamentActive && view === 'main' && (
                            <div className="overflow-x-auto pb-8 animate-fade-in">
                                <div className="flex gap-12 min-w-max px-4">
                                    {bracket.map((round, rIndex) => (
                                        <div key={rIndex} className="flex flex-col justify-around relative">
                                            <h3 className="text-center text-blue-300 font-bold mb-4 bg-slate-800/90 p-2 rounded sticky top-0 z-10 border border-slate-700">
                                                {rIndex === bracket.length - 1 ? 'GRAN FINAL' : `Ronda ${rIndex + 1}`}
                                            </h3>
                                            <div className="flex flex-col justify-around flex-grow gap-8">
                                                {round.map((match, mIndex) => renderMatchCard(match, rIndex, mIndex))}
                                            </div>
                                        </div>
                                    ))}
                                </div>
                            </div>
                        )}

                        {/* CONSUELO */}
                        {isTournamentActive && view === 'consolation' && (
                            <div className="max-w-4xl mx-auto animate-fade-in">
                                <div className="bg-orange-900/20 p-4 rounded mb-6 border border-orange-600/30 flex gap-3 items-center">
                                    <div className="text-orange-500"><IconShield /></div>
                                    <div>
                                        <h3 className="font-bold text-orange-400">Torneo de Perdedores</h3>
                                        <p className="text-sm text-slate-400">Partidas rápidas para los que cayeron en primera ronda.</p>
                                    </div>
                                </div>
                                <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
                                    {consolationBracket.length > 0 && consolationBracket[0].map((match, mIndex) => (
                                        <div key={match.id} className="bg-slate-800 border border-orange-900/50 p-4 rounded relative shadow-lg">
                                            <div className="text-xs text-orange-500 mb-2 uppercase tracking-wide font-bold">Repechaje</div>
                                            {/* Team A */}
                                            <div className="flex justify-between items-center mb-2">
                                                <span className="text-white font-bold truncate w-24">{match.teamA.name}</span>
                                                <div className="flex items-center gap-2">
                                                    <span className="font-mono text-xl">{match.scoreA}</span>
                                                    {!match.winner && <button onClick={() => updateScore(0, mIndex, 'A', 1)} className="bg-orange-700 hover:bg-orange-600 w-6 h-6 rounded flex items-center justify-center text-xs font-bold text-white">+</button>}
                                                </div>
                                            </div>
                                            <div className="text-center text-xs text-slate-600 font-bold my-1">VS</div>
                                            {/* Team B */}
                                            <div className="flex justify-between items-center mt-2">
                                                <span className="text-white font-bold truncate w-24">{match.teamB ? match.teamB.name : 'Esperando...'}</span>
                                                {match.teamB && (
                                                    <div className="flex items-center gap-2">
                                                        <span className="font-mono text-xl">{match.scoreB}</span>
                                                        {!match.winner && <button onClick={() => updateScore(0, mIndex, 'B', 1)} className="bg-orange-700 hover:bg-orange-600 w-6 h-6 rounded flex items-center justify-center text-xs font-bold text-white">+</button>}
                                                    </div>
                                                )}
                                            </div>
                                            {match.winner && <div className="absolute inset-0 bg-black/80 flex items-center justify-center rounded text-green-400 font-bold border-2 border-green-500 backdrop-blur-sm">Ganador: {match.winner.name}</div>}
                                        </div>
                                    ))}
                                    {consolationBracket.length === 0 && <p className="text-slate-500 italic col-span-3 text-center py-8">Aún no hay perdedores. Espera a que termine la Ronda 1.</p>}
                                </div>
                            </div>
                        )}
                    </div>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
