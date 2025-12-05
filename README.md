import React, { useState, useEffect } from 'react';
import { initializeApp } from "firebase/app";
import { 
  getFirestore, collection, doc, setDoc, onSnapshot, 
  updateDoc, deleteDoc, query, where, serverTimestamp, 
  orderBy, getDocs, writeBatch 
} from "firebase/firestore";
import { getAuth, signInAnonymously } from "firebase/auth";
import { Trophy, Users, Save, Trash2, Zap, Clock, Share2, Smartphone, Monitor } from 'lucide-react';

// --- CONFIGURACIÓN DE FIREBASE ---
// NOTA PARA GITHUB: Cuando subas esto a GitHub, tendrás que reemplazar estas variables
// con tu propia configuración de Firebase Console.
    apiKey: "AIzaSyAKge21Uy94wXdsygmqf9kbrxlaZm3H-r4",
  authDomain: "cuarenta-5af3b.firebaseapp.com",
  projectId: "cuarenta-5af3b",
  storageBucket: "cuarenta-5af3b.firebasestorage.app",
  messagingSenderId: "221307851852",
  appId: "1:221307851852:web:39bd3457135abf3f4a50e9",
  measurementId: "G-JWGB4DLZN0"

// ID global para la demo. En producción el usuario elige su sala.
const APP_ID = "cuarenta-pro-v1";

const App = () => {
  // --- ESTADOS ---
  const [tournamentId, setTournamentId] = useState('torneo-principal');
  const [userId, setUserId] = useState(null);
  const [view, setView] = useState('setup'); // setup, lobby, active
  
  // Datos
  const [teams, setTeams] = useState([]);
  const [matches, setMatches] = useState([]);
  const [status, setStatus] = useState({ state: 'registration', round: 1 });

  // Inputs Formulario
  const [teamName, setTeamName] = useState('');
  const [p1, setP1] = useState('');
  const [p2, setP2] = useState('');

  // --- 1. AUTENTICACIÓN Y CONEXIÓN ---
  useEffect(() => {
    const initAuth = async () => {
       try {
         if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
            // Entorno de prueba
            await import('firebase/auth').then(async ({ signInWithCustomToken }) => {
                await signInWithCustomToken(auth, __initial_auth_token);
            });
         } else {
            // Entorno público / GitHub
            await signInAnonymously(auth);
         }
       } catch (error) {
         console.error("Error auth:", error);
       }
    };
    
    initAuth();
    
    const unsubAuth = auth.onAuthStateChanged((user) => {
      if (user) setUserId(user.uid);
    });
    return () => unsubAuth();
  }, []);

  // --- 2. ESCUCHAR CAMBIOS EN TIEMPO REAL (SNAPSHOTS) ---
  useEffect(() => {
    if (!userId || !tournamentId) return;

    // Referencias a colecciones usando la estructura segura
    // Estructura: artifacts/{APP_ID}/public/data/{tournamentId}_[teams|matches|status]
    // Usamos un prefijo en el nombre de la colección para simular subcolecciones en la ruta pública
    
    const teamsRef = collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`);
    const matchesRef = collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`);
    const statusRef = doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_status`, 'info');

    // Escuchar Equipos
    const unsubTeams = onSnapshot(query(teamsRef, orderBy('createdAt', 'desc')), (snapshot) => {
      setTeams(snapshot.docs.map(d => ({ id: d.id, ...d.data() })));
    }, (err) => console.error("Error teams", err));

    // Escuchar Partidos
    const unsubMatches = onSnapshot(query(matchesRef, orderBy('matchNumber', 'asc')), (snapshot) => {
      setMatches(snapshot.docs.map(d => ({ id: d.id, ...d.data() })));
    }, (err) => console.error("Error matches", err));

    // Escuchar Estado del Torneo
    const unsubStatus = onSnapshot(statusRef, (docSnap) => {
      if (docSnap.exists()) {
        setStatus(docSnap.data());
        setView(docSnap.data().state === 'registration' ? 'lobby' : 'active');
      } else {
        // Si no existe, crearlo por defecto
        setDoc(statusRef, { state: 'registration', round: 1 });
        setStatus({ state: 'registration', round: 1 });
        setView('lobby');
      }
    });

    return () => {
      unsubTeams();
      unsubMatches();
      unsubStatus();
    };
  }, [userId, tournamentId]);


  // --- 3. FUNCIONES DE GESTIÓN ---

  const addTeam = async () => {
    if (!teamName || !p1 || !p2) return alert("Llena todos los campos");
    
    const teamsRef = collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`);
    await setDoc(doc(teamsRef), {
      name: teamName,
      p1, p2,
      createdAt: serverTimestamp(),
      active: true // Para saber si siguen en el torneo
    });
    setTeamName(''); setP1(''); setP2('');
  };

  const deleteTeam = async (id) => {
    await deleteDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`, id));
  };

  // --- 4. LÓGICA DEL TORNEO (CORE) ---

  const shuffle = (array) => {
    let currentIndex = array.length, randomIndex;
    while (currentIndex !== 0) {
      randomIndex = Math.floor(Math.random() * currentIndex);
      currentIndex--;
      [array[currentIndex], array[randomIndex]] = [array[randomIndex], array[currentIndex]];
    }
    return array;
  };

  // GENERAR RONDA 1
  const startTournament = async () => {
    if (teams.length < 2) return alert("Se necesitan mínimo 2 equipos");
    
    const batch = writeBatch(db);
    const shuffled = shuffle([...teams]);
    
    // Crear partidos Ronda 1
    let matchCount = 1;
    
    // Lógica simple de pares para la primera ronda
    for (let i = 0; i < shuffled.length; i += 2) {
      const teamA = shuffled[i];
      const teamB = shuffled[i+1];
      
      const matchRef = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
      
      if (teamB) {
        batch.set(matchRef, {
          round: 1,
          matchNumber: matchCount++,
          teamA, teamB,
          scoreA: 0, scoreB: 0,
          winner: null,
          finishedAt: null, // Importante para el desempate por tiempo
          status: 'playing'
        });
      } else {
        // Impar en Ronda 1: Pase directo automático (Bye)
        batch.set(matchRef, {
          round: 1,
          matchNumber: matchCount++,
          teamA, teamB: null,
          scoreA: 2, scoreB: 0,
          winner: teamA,
          finishedAt: serverTimestamp(), // Pasa "rápido" virtualmente
          status: 'finished',
          isBye: true
        });
      }
    }

    // Actualizar estado global
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
    
    // Verificar Ganador (Mejor de 2 mesas/chicas)
    // NOTA: Aquí capturamos el tiempo exacto (serverTimestamp) para saber quién acabó primero
    if (newScore >= 2) {
      const match = matches.find(m => m.id === matchId);
      updates.winner = team === 'A' ? match.teamA : match.teamB;
      updates.status = 'finished';
      updates.finishedAt = serverTimestamp(); 
    }

    await updateDoc(matchRef, updates);
  };

  // GENERAR SIGUIENTE RONDA (LA LÓGICA QUE PEDISTE)
  const generateNextRound = async () => {
    // 1. Obtener ganadores de la ronda actual
    const currentRoundMatches = matches.filter(m => m.round === status.round);
    
    // Validar que todos terminaron
    if (currentRoundMatches.some(m => !m.winner)) return alert("Faltan partidas por terminar.");

    // 2. Ordenar ganadores por TIEMPO DE FINALIZACIÓN (El más rápido primero)
    // Nota: Firebase devuelve timestamps objetos, hay que compararlos con cuidado
    const winners = currentRoundMatches
      .map(m => ({ 
        team: m.winner, 
        time: m.finishedAt?.seconds || 0, // Segundos desde epoch
        isBye: m.isBye // Saber si vino de un bye
      }))
      .sort((a, b) => {
        // Si a es 0 (error), ponlo al final. Si no, ordena ascendente (menor tiempo = más rápido)
        if (!a.time) return 1;
        if (!b.time) return -1;
        return a.time - b.time;
      });

    const nextRound = status.round + 1;
    const batch = writeBatch(db);
    let matchCount = 1;

    // 3. Emparejar para la siguiente ronda
    let pool = [...winners];
    
    // SI ES IMPAR: El primero de la lista (el más rápido) pasa directo
    if (pool.length % 2 !== 0) {
      const fastest = pool.shift(); // Saca al primero
      
      // Crear registro de su "Pase Directo"
      const matchRef = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
      batch.set(matchRef, {
        round: nextRound,
        matchNumber: matchCount++,
        teamA: fastest.team,
        teamB: null, // Bye
        scoreA: 2, scoreB: 0,
        winner: fastest.team,
        finishedAt: serverTimestamp(), // Pasa con tiempo actual
        status: 'finished',
        isBye: true,
        note: "Pase por rapidez" // Etiqueta visual
      });
    }

    // Emparejar el resto
    for (let i = 0; i < pool.length; i += 2) {
      const p1 = pool[i];
      const p2 = pool[i+1];
      
      const matchRef = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
      batch.set(matchRef, {
        round: nextRound,
        matchNumber: matchCount++,
        teamA: p1.team,
        teamB: p2.team,
        scoreA: 0, scoreB: 0,
        winner: null,
        finishedAt: null,
        status: 'playing'
      });
    }

    // Actualizar estado
    const statusRef = doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_status`, 'info');
    batch.update(statusRef, { round: nextRound });

    await batch.commit();
  };

  // ELIMINAR TODO (RESET)
  const nukeTournament = async () => {
    if (!confirm("¿ESTÁS SEGURO? Esto borrará todo para todos los dispositivos.")) return;
    
    // Borrar equipos
    const tQ = query(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_teams`));
    const tDocs = await getDocs(tQ);
    tDocs.forEach(d => deleteDoc(d.ref));
    
    // Borrar partidos
    const mQ = query(collection(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_matches`));
    const mDocs = await getDocs(mQ);
    mDocs.forEach(d => deleteDoc(d.ref));

    // Reset estado
    const statusRef = doc(db, 'artifacts', APP_ID, 'public', 'data', `${tournamentId}_status`, 'info');
    await setDoc(statusRef, { state: 'registration', round: 1 });
  };


  // --- 5. RENDERIZADO UI ---

  const currentMatches = matches.filter(m => m.round === status.round);
  const finishedMatchesCount = currentMatches.filter(m => m.winner).length;
  const totalMatchesCount = currentMatches.length;
  const isRoundComplete = totalMatchesCount > 0 && finishedMatchesCount === totalMatchesCount;

  if (view === 'setup') {
    return (
      <div className="min-h-screen bg-slate-950 text-white flex items-center justify-center p-4">
        <div className="max-w-md w-full bg-slate-900 p-8 rounded-2xl border border-slate-800 shadow-2xl">
          <div className="flex justify-center mb-6">
            <div className="bg-blue-600 p-4 rounded-xl shadow-lg shadow-blue-900/50">
              <Share2 size={40} className="text-white" />
            </div>
          </div>
          <h1 className="text-2xl font-bold text-center mb-2">Cuarenta Multidispositivo</h1>
          <p className="text-slate-400 text-center text-sm mb-6">
            Ingresa un nombre para tu torneo. <br/>
            <span className="text-blue-400">Usa el mismo nombre en todos los celulares</span> para ver los mismos datos.
          </p>
          
          <div className="space-y-4">
             <div>
               <label className="text-xs font-bold text-slate-500 uppercase">ID del Torneo (Sala)</label>
               <input 
                value={tournamentId} 
                onChange={(e) => setTournamentId(e.target.value.toLowerCase().replace(/\s/g, '-'))}
                className="w-full bg-slate-950 border border-slate-700 p-4 rounded-lg font-mono text-center text-lg focus:border-blue-500 outline-none"
               />
             </div>
             <button 
              onClick={() => setView('lobby')}
              className="w-full bg-blue-600 hover:bg-blue-500 text-white font-bold py-4 rounded-lg transition"
             >
               Entrar a la Sala
             </button>
          </div>
        </div>
      </div>
    )
  }

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 font-sans p-4 pb-20">
      <div className="max-w-3xl mx-auto">
        
        {/* HEADER */}
        <header className="flex justify-between items-center mb-6 border-b border-slate-800 pb-4">
          <div>
             <h2 className="font-bold text-xl flex items-center gap-2">
               <Monitor className="text-blue-500"/> Sala: <span className="font-mono text-blue-300">{tournamentId}</span>
             </h2>
             <p className="text-xs text-slate-500 flex items-center gap-1">
               <Smartphone size={12}/> Los datos se sincronizan solos
             </p>
          </div>
          {status.state !== 'registration' && (
            <div className="text-right">
              <div className="text-sm font-bold text-orange-400">Ronda {status.round}</div>
              <div className="text-xs text-slate-500">{finishedMatchesCount}/{totalMatchesCount} terminados</div>
            </div>
          )}
        </header>

        {/* VISTA REGISTRO (LOBBY) */}
        {status.state === 'registration' && (
          <div className="space-y-6 animate-in slide-in-from-bottom-4">
            <div className="bg-slate-900/50 p-6 rounded-xl border border-slate-800">
               <h3 className="font-bold text-lg mb-4 text-blue-400 flex items-center gap-2"><Users size={20}/> Registrar Equipo</h3>
               <div className="grid gap-3">
                 <input placeholder="Nombre del Equipo (Ej. Los Intocables)" className="bg-slate-950 border border-slate-700 p-3 rounded text-white" value={teamName} onChange={e => setTeamName(e.target.value)} />
                 <div className="grid grid-cols-2 gap-3">
                    <input placeholder="Jugador 1" className="bg-slate-950 border border-slate-700 p-3 rounded text-white" value={p1} onChange={e => setP1(e.target.value)} />
                    <input placeholder="Jugador 2" className="bg-slate-950 border border-slate-700 p-3 rounded text-white" value={p2} onChange={e => setP2(e.target.value)} />
                 </div>
                 <button onClick={addTeam} className="bg-green-600 hover:bg-green-500 text-white font-bold py-3 rounded flex items-center justify-center gap-2 mt-2"><Save size={18}/> Guardar Equipo</button>
               </div>
            </div>

            <div className="space-y-2">
              <h3 className="text-xs font-bold text-slate-500 uppercase">Equipos en Sala ({teams.length})</h3>
              {teams.length === 0 ? <p className="text-slate-600 italic">Esperando equipos...</p> : 
                teams.map(t => (
                  <div key={t.id} className="bg-slate-800 p-3 rounded flex justify-between items-center border border-slate-700">
                    <div>
                      <div className="font-bold text-white">{t.name}</div>
                      <div className="text-xs text-slate-400">{t.p1} & {t.p2}</div>
                    </div>
                    <button onClick={() => deleteTeam(t.id)} className="text-red-400 p-2"><Trash2 size={16}/></button>
                  </div>
                ))
              }
            </div>

            {teams.length >= 2 && (
              <div className="sticky bottom-4">
                <button onClick={startTournament} className="w-full bg-blue-600 hover:bg-blue-500 text-white font-bold py-4 rounded-xl shadow-lg shadow-blue-900/20 text-lg flex items-center justify-center gap-2">
                  <Trophy /> Iniciar Torneo
                </button>
              </div>
            )}
          </div>
        )}

        {/* VISTA JUEGO ACTIVO */}
        {status.state === 'playing' && (
          <div className="space-y-6">
            
            {/* Lista de Partidos */}
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              {currentMatches.map(match => {
                if (match.isBye) {
                  return (
                    <div key={match.id} className="bg-slate-900/50 border border-dashed border-slate-700 p-4 rounded-lg flex items-center justify-between opacity-70">
                       <div>
                         <span className="text-orange-400 text-xs font-bold uppercase tracking-wider mb-1 block">
                           {match.note || "Pase Directo"}
                         </span>
                         <span className="font-bold text-white">{match.teamA.name}</span>
                       </div>
                       <span className="bg-green-900/30 text-green-400 text-xs px-2 py-1 rounded">ESPERANDO RONDA {status.round + 1}</span>
                    </div>
                  )
                }

                return (
                  <div key={match.id} className={`bg-slate-800 border ${match.winner ? 'border-green-600' : 'border-slate-700'} p-4 rounded-lg shadow-lg relative overflow-hidden`}>
                    <div className="flex justify-between items-center mb-4">
                      <span className="text-xs font-mono text-slate-500">Mesa #{match.matchNumber}</span>
                      {match.winner ? 
                        <span className="bg-green-500 text-black text-xs font-bold px-2 py-0.5 rounded flex items-center gap-1"><Clock size={12}/> FINALIZADO</span> : 
                        <span className="bg-blue-600/20 text-blue-400 text-xs font-bold px-2 py-0.5 rounded animate-pulse">JUGANDO</span>
                      }
                    </div>

                    {/* Team A */}
                    <div className={`flex justify-between items-center p-2 rounded mb-1 ${match.winner?.name === match.teamA.name ? 'bg-green-900/20' : ''}`}>
                      <div className="overflow-hidden">
                        <div className="font-bold truncate">{match.teamA.name}</div>
                        <div className="text-xs text-slate-400 truncate">{match.teamA.p1}/{match.teamA.p2}</div>
                      </div>
                      <div className="flex items-center gap-3">
                        <span className="text-2xl font-mono">{match.scoreA}</span>
                        {!match.winner && <button onClick={() => updateScore(match.id, 'A', match.scoreA)} className="bg-blue-600 h-8 w-8 rounded flex items-center justify-center font-bold text-white active:scale-95 transition">+</button>}
                      </div>
                    </div>

                    <div className="h-px bg-slate-700 w-full my-2"></div>

                    {/* Team B */}
                    <div className={`flex justify-between items-center p-2 rounded ${match.winner?.name === match.teamB.name ? 'bg-green-900/20' : ''}`}>
                      <div className="overflow-hidden">
                        <div className="font-bold truncate">{match.teamB.name}</div>
                        <div className="text-xs text-slate-400 truncate">{match.teamB.p1}/{match.teamB.p2}</div>
                      </div>
                      <div className="flex items-center gap-3">
                        <span className="text-2xl font-mono">{match.scoreB}</span>
                        {!match.winner && <button onClick={() => updateScore(match.id, 'B', match.scoreB)} className="bg-blue-600 h-8 w-8 rounded flex items-center justify-center font-bold text-white active:scale-95 transition">+</button>}
                      </div>
                    </div>
                  </div>
                )
              })}
            </div>

            {/* Panel de Control de Ronda */}
            <div className="sticky bottom-4 bg-slate-900/90 backdrop-blur border border-slate-700 p-4 rounded-xl shadow-2xl flex justify-between items-center">
               <div className="text-sm">
                 <span className="text-slate-400">Progreso Ronda {status.round}:</span>
                 <div className="font-bold text-white">{finishedMatchesCount} de {totalMatchesCount} finalizados</div>
               </div>
               
               {isRoundComplete ? (
                 <button onClick={generateNextRound} className="bg-orange-600 hover:bg-orange-500 text-white font-bold py-3 px-6 rounded-lg animate-bounce flex items-center gap-2">
                   <Zap size={18}/> Siguiente Ronda
                 </button>
               ) : (
                 <div className="text-xs text-slate-500 italic px-4">
                   Esperando que terminen todos...
                 </div>
               )}
            </div>

            <div className="pt-10 flex justify-center">
               <button onClick={nukeTournament} className="text-red-500 text-xs flex items-center gap-1 hover:text-red-400 border border-red-900/30 p-2 rounded">
                 <Trash2 size={12}/> ELIMINAR TORNEO
               </button>
            </div>

          </div>
        )}

      </div>
    </div>
  );
};

export default App;
