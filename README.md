import React, { useState, useEffect, useContext, createContext, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot, updateDoc } from 'firebase/firestore';

// --- CONFIGURACIÓN E INICIALIZACIÓN DE FIREBASE (MANDATORIO) ---
// Variables globales disponibles en el entorno de Canvas
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const appId = typeof __app_id !== 'undefined' ? __app_id : 'rutina-express-app';
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Inicialización
let app, db, auth;
if (Object.keys(firebaseConfig).length > 0) {
    app = initializeApp(firebaseConfig);
    db = getFirestore(app);
    auth = getAuth(app);
}

// --- CONTEXTO DE LA APLICACIÓN ---
const AppContext = createContext();

const initialUserData = {
    displayName: '',
    email: '',
    age: '',
    level: 1,
    xp: 0,
    trainings: 0,
    minutes: 0,
    calories: 0,
    streak: 0,
    lastActivity: 0,
    goals: { weeklyMinutes: 60 },
    achievements: [],
    challenges: [],
    customChallenge: null,
};

// Hook personalizado para Firebase y estado de usuario
const useAppState = () => {
    const [userData, setUserData] = useState(initialUserData);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);
    const [isLoading, setIsLoading] = useState(true);
    const [currentPage, setCurrentPage] = useState('onboarding');

    // 1. Inicialización de Firebase y Autenticación
    useEffect(() => {
        if (!auth) {
            console.error("Firebase no inicializado.");
            setIsLoading(false);
            return;
        }

        const setupAuth = async () => {
            try {
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Error durante la autenticación:", error);
            }
        };

        const unsubscribe = onAuthStateChanged(auth, (user) => {
            if (user) {
                setUserId(user.uid);
                setIsAuthReady(true);
            } else {
                setUserId(null);
                setIsAuthReady(true);
                setupAuth();
            }
        });

        return () => unsubscribe();
    }, []);

    // 2. Carga y Suscripción a Datos del Usuario
    useEffect(() => {
        if (!isAuthReady || !userId || !db) {
            if (isAuthReady) setIsLoading(false);
            return;
        }

        const userDocRef = doc(db, 'artifacts', appId, 'users', userId, 'profile', 'data');
        
        const initializeUser = async (uid) => {
            const now = Date.now();
            const initialDataWithDefaults = {
                ...initialUserData,
                lastActivity: now,
                displayName: `Usuario_${uid.substring(0, 4)}`,
                customChallenge: { // Inicializar el reto personalizado
                    isActive: false,
                    goalMinutes: 0,
                    xpReward: 0,
                    progress: 0,
                    isCompleted: false,
                    isVerified: false, // Nuevo: indicador de verificación con foto
                },
            };
            try {
                await setDoc(userDocRef, initialDataWithDefaults, { merge: true });
                setUserData(initialDataWithDefaults);
                setCurrentPage('home');
            } catch (e) {
                console.error("Error inicializando los datos del usuario:", e);
            }
        };

        const unsubscribeSnapshot = onSnapshot(userDocRef, (docSnap) => {
            if (docSnap.exists()) {
                const data = docSnap.data();
                if (!data.customChallenge) {
                    data.customChallenge = { isActive: false, goalMinutes: 0, xpReward: 0, progress: 0, isCompleted: false, isVerified: false };
                }
                setUserData(data);
                setCurrentPage(data.displayName ? 'home' : 'onboarding');
            } else {
                initializeUser(userId);
            }
            setIsLoading(false);
        }, (error) => {
            console.error("Error escuchando los datos del usuario:", error);
            setIsLoading(false);
        });

        return () => unsubscribeSnapshot();
    }, [userId, isAuthReady]);

    // Lógica para el sistema de Gamificación
    const checkAchievements = (newXP, currentAchievements) => {
        const achievementsList = [
            { id: 'first_activation', name: 'Primera Activación', xpNeeded: 10, icon: '🌟' },
            { id: 'weekly_warrior', name: 'Guerrero Semanal', xpNeeded: 50, icon: '🛡️' },
            { id: 'streak_initiated', name: 'Racha Iniciada', xpNeeded: 100, icon: '🔥' },
            { id: 'maestro_tiempo', name: 'Maestro del Tiempo', xpNeeded: 250, icon: '⏱️' },
            { id: 'ultra_constancia', name: 'Ultra Constancia', xpNeeded: 500, icon: '✨' },
            { id: 'leyenda_express', name: 'Leyenda Express', xpNeeded: 1000, icon: '👑' },
        ];

        let newAchievements = [...currentAchievements];
        achievementsList.forEach(ach => {
            if (newXP >= ach.xpNeeded && !currentAchievements.includes(ach.id)) {
                newAchievements.push(ach.id);
                console.log(`¡Logro desbloqueado: ${ach.name}!`);
            }
        });
        return newAchievements;
    };
    
    // Función para actualizar los datos del usuario
    const updateUserData = async (updates) => {
        if (!db || !userId) return;

        const userDocRef = doc(db, 'artifacts', appId, 'users', userId, 'profile', 'data');

        let currentData = { ...userData };

        // Lógica de gamificación antes de actualizar
        if (updates.xp) {
            const newXP = (currentData.xp || 0) + updates.xp;
            const newLevel = Math.floor(newXP / 100) + 1;
            updates.xp = newXP;
            updates.level = newLevel;

            updates.achievements = checkAchievements(newXP, currentData.achievements || []);
        }

        // Lógica: Si se actualizan los minutos, actualizar el progreso del reto personalizado
        if (updates.minutes && currentData.customChallenge?.isActive && !currentData.customChallenge?.isCompleted) {
            const newProgress = Math.min(updates.minutes, currentData.customChallenge.goalMinutes);
            updates.customChallenge = { ...currentData.customChallenge, progress: newProgress };
            
            // Si el reto se completa
            if (newProgress === currentData.customChallenge.goalMinutes) {
                updates.customChallenge.isCompleted = true;
                // NOTA: Los XP del reto personalizado SÓLO se otorgan cuando se verifica con la foto.
                console.log(`¡Reto Personalizado completado! Requiere verificación con foto.`);
            }
        }
        
        // Manejo de XP si se actualiza el customChallenge
        if (updates.customChallenge?.isVerified && !currentData.customChallenge?.isVerified && currentData.customChallenge?.isCompleted) {
             updates.xp = (updates.xp || currentData.xp) + currentData.customChallenge.xpReward;
             updates.customChallenge = { ...currentData.customChallenge, isVerified: true };
             console.log(`¡Reto Personalizado verificado! Ganaste ${currentData.customChallenge.xpReward} XP.`);
        }


        try {
            await updateDoc(userDocRef, updates);
        } catch (e) {
            console.error("Error actualizando datos del usuario:", e);
        }
    };

    const value = useMemo(() => ({
        userData,
        userId,
        isLoading,
        currentPage,
        setCurrentPage,
        updateUserData
    }), [userData, userId, isLoading, currentPage]);

    return value;
};

// Componente Proveedor de Contexto
const AppProvider = ({ children }) => {
    const state = useAppState();
    return <AppContext.Provider value={state}>{children}</AppContext.Provider>;
};

// Hook para usar el contexto
const useAppContext = () => useContext(AppContext);

// --- ÍCONOS (Usando Lucide-React / Inline SVG) ---
const HomeIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="m3 9 9-7 9 7v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"/><polyline points="9 22 9 12 15 12 15 22"/></svg>;
const TargetIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><circle cx="12" cy="12" r="10"/><circle cx="12" cy="12" r="6"/><circle cx="12" cy="12" r="2"/></svg>;
const TrophyIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M6 9H4.5a2.5 2.5 0 0 1 0-5H6"/><path d="M18 9h1.5a2.5 2.5 0 0 0 0-5H18"/><path d="M12 22V8"/><path d="M5 9a5 5 0 0 1 14 0"/><path d="M12 8h.01"/></svg>;
const UserIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M19 21v-2a4 4 0 0 0-4-4H9a4 4 0 0 0-4 4v2"/><circle cx="12" cy="7" r="4"/></svg>;
const LogOutIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4"/><polyline points="16 17 21 12 16 7"/><line x1="21" x2="9" y1="12" y2="12"/></svg>;
const ClockIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><circle cx="12" cy="12" r="10"/><polyline points="12 6 12 12 16 14"/></svg>;
const FireIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M8.5 14.5A2.5 2.5 0 0 0 11 12c0-1.38-2.5-3-2.5-3S6 10.62 6 12c0 1.38 2.5 3 2.5 3Z"/><path d="M13.5 15.5c0 1.38-2.5 3-2.5 3s-2.5-1.62-2.5-3c0-1.38 2.5-3 2.5-3s2.5 1.62 2.5 3Z"/><path d="M17.5 13.5c0 1.38-2.5 3-2.5 3s-2.5-1.62-2.5-3c0-1.38 2.5-3 2.5-3s2.5 1.62 2.5 3Z"/></svg>;
const ActivityIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M22 12h-4l-3 9L11 3l-3 9H2"/></svg>;
const DumbbellIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="m14.4 17.6-4.8-4.8-3.6 3.6-1-1 3.6-3.6L3 7.6l2-2 3.6 3.6L13.2 4l-1-1 3.6-3.6 2 2 3.6 3.6L20 9.6l-2-2-3.6-3.6L12.4 1.2l-2-2 3.6 3.6L14.4 7.6l2 2-3.6 3.6L16.4 16.4l1 1-3.6 3.6L14.4 17.6z"/><path d="M4 22l4-4"/></svg>;
const CameraIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M14.5 4h-5L7 7H4a2 2 0 0 0-2 2v9a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2V9a2 2 0 0 0-2-2h-3l-2.5-3z"/><circle cx="12" cy="13" r="3"/></svg>;
const CheckCircleIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg>;
const HeartPulseIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M19 14c1.49-1.46 3-3.21 3-5.5A5.5 5.5 0 0 0 16.5 3c-1.76 0-3 .5-4.5 2-1.5-1.5-2.74-2-4.5-2A5.5 5.5 0 0 0 2 8.5c0 2.3 1.5 4.05 3 5.5l7 7Z"/><path d="M3.2 11H9l2-6 3 9 2-3h5.5"/></svg>;
const WeightsIcon = (props) => <svg {...props} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M10 20h4a2 2 0 0 0 2-2v-2a2 2 0 0 0-2-2h-4a2 2 0 0 0-2 2v2a2 2 0 0 0 2 2z"/><path d="M16 12h2a2 2 0 0 0 2-2V8a2 2 0 0 0-2-2h-2"/><path d="M8 12H6a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h2"/><path d="M16 18h2a2 2 0 0 1 2 2v2a2 2 0 0 1-2 2h-2"/><path d="M8 18H6a2 2 0 0 0-2 2v2a2 2 0 0 0 2 2h2"/></svg>;

// --- COMPONENTES DE PANTALLA ---

// --- 0. ONBOARDING (Registro) ---
const OnboardingScreen = () => {
    const { updateUserData, setCurrentPage } = useAppContext();
    const [step, setStep] = useState(0);
    const [name, setName] = useState('');
    const [email, setEmail] = useState('');
    const [age, setAge] = useState('');
    const [error, setError] = useState('');

    const slides = [
        { title: '¡Hola! 🏃', text: 'Bienvenido a tu app de rutinas rápidas y gamificadas. ¡Vamos a empezar!', icon: <ActivityIcon className="w-16 h-16 text-white" /> },
        { title: 'Motivación con XP 🏆', text: 'Gana experiencia, sube de nivel y desbloquea logros por cada minuto activo.', icon: <TrophyIcon className="w-16 h-16 text-white" /> },
        { title: 'Desafíos Personalizados 📝', text: 'Crea tu propia rutina y verifica tu progreso con fotos para ganar XP extra.', icon: <CameraIcon className="w-16 h-16 text-white" /> },
    ];

    const handleRegister = async () => {
        if (!name || !email || !age || isNaN(age) || age < 18 || age > 65) {
            setError('Por favor, ingresa un nombre, email válido y edad entre 18-65.');
            return;
        }
        setError('');
        await updateUserData({ displayName: name, email, age: parseInt(age), xp: 10, achievements: ['first_activation'] });
        setCurrentPage('home');
    };

    return (
        <div className="min-h-screen flex flex-col justify-center items-center p-6 bg-gradient-to-br from-indigo-500 to-purple-600">
            <div className="text-white text-center w-full max-w-sm">
                
                <div className="mb-8 p-6 bg-white/15 backdrop-blur-sm rounded-3xl border border-white/20 transition-all duration-500 ease-in-out shadow-2xl">
                    {slides[step].icon}
                    <h2 className="text-3xl font-bold mt-4">{slides[step].title}</h2>
                    <p className="mt-2 text-lg opacity-90">{slides[step].text}</p>
                    <div className="flex justify-center mt-6 space-x-2">
                        {slides.map((_, index) => (
                            <div key={index} className={`w-3 h-3 rounded-full transition-colors ${step === index ? 'bg-yellow-300 scale-110' : 'bg-gray-400 opacity-50'}`}></div>
                        ))}
                    </div>
                </div>

                {step < slides.length - 1 ? (
                    <button
                        onClick={() => setStep(step + 1)}
                        className="w-full py-3 bg-yellow-400 text-indigo-800 font-extrabold text-lg rounded-full shadow-xl transition duration-200 hover:bg-yellow-500 transform hover:scale-[1.02]"
                    >
                        Continuar
                    </button>
                ) : (
                    <div className="bg-white p-6 rounded-2xl shadow-xl text-black">
                        <h3 className="text-xl font-bold mb-4 text-center text-indigo-600">Completar Perfil</h3>
                        <input
                            type="text"
                            placeholder="Tu Nombre"
                            value={name}
                            onChange={(e) => setName(e.target.value)}
                            className="w-full p-3 mb-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500"
                        />
                         <input
                            type="email"
                            placeholder="Correo Electrónico"
                            value={email}
                            onChange={(e) => setEmail(e.target.value)}
                            className="w-full p-3 mb-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500"
                        />
                        <input
                            type="number"
                            placeholder="Edad (18-65)"
                            value={age}
                            onChange={(e) => setAge(e.target.value)}
                            className="w-full p-3 mb-4 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500"
                        />
                        {error && <p className="text-red-500 text-sm mb-4">{error}</p>}
                        <button
                            onClick={handleRegister}
                            className="w-full py-3 bg-green-500 text-white font-bold rounded-full shadow-lg transition duration-200 hover:bg-green-600"
                        >
                            ¡Empezar a Entrenar!
                        </button>
                    </div>
                )}
            </div>
        </div>
    );
};

// --- 1. HOME ---
const HomeScreen = () => {
    const { userData, updateUserData } = useAppContext();
    const { displayName, level, xp, trainings, minutes, streak } = userData;
    const xpForNextLevel = 100 * level;
    const currentLevelXp = xp % 100;
    const progress = (currentLevelXp / 100) * 100;

    const handleStartWorkout = (xpGain, time) => {
        const newMinutes = (userData.minutes || 0) + time;
        
        updateUserData({
            trainings: (userData.trainings || 0) + 1,
            minutes: newMinutes,
            calories: (userData.calories || 0) + (time * 6),
            xp: xpGain,
            streak: (Date.now() - (userData.lastActivity || 0) < 86400000) ? (userData.streak || 0) + 1 : 1, 
            lastActivity: Date.now(),
        });
        console.log(`¡Rutina completada! Ganaste ${xpGain} XP.`);
    };

    return (
        <div className="p-4 space-y-5">
            <div className="flex justify-between items-center border-b pb-3 border-gray-200">
                <h2 className="text-2xl font-bold text-gray-800">Hola, {displayName}!</h2>
                <span className="bg-indigo-500 text-white text-xs font-semibold px-3 py-1 rounded-full shadow-md">Nivel {level}</span>
            </div>
            
            {/* Tarjeta de Progreso de Nivel */}
            <div className="bg-white p-4 rounded-xl shadow-2xl border border-gray-100">
                <div className="flex justify-between items-center mb-2">
                    <span className="text-lg font-semibold text-indigo-600">Próximo Nivel</span>
                    <span className="text-sm text-gray-500">{currentLevelXp} / 100 XP</span>
                </div>
                <div className="w-full bg-gray-200 rounded-full h-3">
                    <div
                        className="bg-purple-600 h-3 rounded-full transition-all duration-500"
                        style={{ width: `${progress}%` }}
                    ></div>
                </div>
                <p className="text-xs text-gray-500 mt-2 font-medium">XP total acumulado: {xp}</p>
            </div>

            {/* Estadísticas Rápidas */}
            <div className="grid grid-cols-3 gap-3">
                <StatCard icon={<FireIcon className="w-6 h-6 text-orange-500" />} label="Racha" value={streak} unit="días" />
                <StatCard icon={<ActivityIcon className="w-6 h-6 text-green-500" />} label="Entrenos" value={trainings} unit="total" />
                <StatCard icon={<ClockIcon className="w-6 h-6 text-blue-500" />} label="Minutos" value={minutes} unit="acum." />
            </div>

            {/* Rutinas de Hoy - Organiza los ejercicios */}
            <div className="space-y-4 pt-2">
                <h3 className="text-xl font-bold text-gray-800 border-b pb-2 mb-2 flex items-center"><ActivityIcon className="w-5 h-5 mr-2 text-indigo-600"/> Rutinas para Hoy</h3>
                
                <RoutineButton 
                    title="Activación en el Asiento" 
                    time={5} 
                    xp={15} 
                    onStart={() => handleStartWorkout(15, 5)} 
                    icon="🚇"
                />
                <RoutineButton 
                    title="Pausa de Escritorio" 
                    time={10} 
                    xp={25} 
                    onStart={() => handleStartWorkout(25, 10)} 
                    icon="💻"
                />

                <h4 className="text-lg font-semibold text-gray-700 mt-4 border-l-4 border-pink-400 pl-2">⚡ Cardio Express</h4>
                <RoutineButton 
                    title="HIIT Sin Saltar" 
                    time={8} 
                    xp={20} 
                    onStart={() => handleStartWorkout(20, 8)} 
                    icon={<HeartPulseIcon className="w-6 h-6 text-pink-500" />}
                />
                <RoutineButton 
                    title="Escaleras y Rodillas" 
                    time={10} 
                    xp={30} 
                    onStart={() => handleStartWorkout(30, 10)} 
                    icon="⛰️"
                />
                
                <h4 className="text-lg font-semibold text-gray-700 mt-4 border-l-4 border-green-400 pl-2">🏋️‍♀️ Fuerza Rápida</h4>
                <RoutineButton 
                    title="Mini Circuito de Pesas" 
                    time={12} 
                    xp={35} 
                    onStart={() => handleStartWorkout(35, 12)} 
                    icon={<WeightsIcon className="w-6 h-6 text-green-500" />}
                />
                <RoutineButton 
                    title="Fuerza en Pared" 
                    time={7} 
                    xp={18} 
                    onStart={() => handleStartWorkout(18, 7)} 
                    icon="🧱"
                />
            </div>
        </div>
    );
};

const StatCard = ({ icon, label, value, unit }) => (
    <div className="bg-white p-3 rounded-xl shadow-md text-center border border-gray-100">
        <div className="flex justify-center mb-1">{icon}</div>
        <div className="text-xl font-bold text-gray-800">{value}</div>
        <div className="text-xs text-gray-500">{label} {unit}</div>
    </div>
);

const RoutineButton = ({ title, time, xp, onStart, icon }) => (
    <div className="flex justify-between items-center bg-white p-4 rounded-xl shadow-lg border border-indigo-100 hover:shadow-xl transition duration-200 transform hover:scale-[1.01]">
        <div className="flex items-center">
            {React.isValidElement(icon) ? icon : <span className="text-2xl mr-3">{icon}</span>}
            <div>
                <p className="font-semibold text-gray-800">{title}</p>
                <p className="text-sm text-gray-500">{time} min · Gana {xp} XP</p>
            </div>
        </div>
        <button 
            onClick={onStart} 
            className="bg-indigo-500 text-white text-sm font-bold py-2 px-4 rounded-full shadow-md hover:bg-indigo-600 transition duration-200"
        >
            Iniciar
        </button>
    </div>
);

// --- 2. RETOS ---
const ChallengesScreen = () => {
    const { userData, updateUserData } = useAppContext();
    const [customGoal, setCustomGoal] = useState(30);
    const [customXp, setCustomXp] = useState(50);
    const [customError, setCustomError] = useState('');
    const [showVerificationModal, setShowVerificationModal] = useState(false);

    // Retos de ejemplo
    const challenges = [
        { id: 'c1', name: "3 Entrenamientos en una Semana", xp: 50, required: 3, progressKey: 'trainings', type: 'weekly', icon: '📅' },
        { id: 'c2', name: "30 Minutos Acumulados", xp: 30, required: 30, progressKey: 'minutes', type: 'total', icon: '⏱️' },
        { id: 'c3', name: "Guerrero de la Computadora (5 ses.)", xp: 40, required: 5, progressKey: 'trainings', type: 'total', icon: '🖥️' },
        { id: 'c4', name: "Racha de 3 días seguidos", xp: 75, required: 3, progressKey: 'streak', type: 'max', icon: '🔥' },
        { id: 'c5', name: "Más de 500 Kcal quemadas", xp: 100, required: 500, progressKey: 'calories', type: 'total', icon: '🌶️' },
        { id: 'c6', name: "20 Minutos de Pausa Activa", xp: 50, required: 20, progressKey: 'minutes', type: 'total', icon: '🧘' },
    ];

    const getCurrentProgress = (challenge) => {
        let value = userData[challenge.progressKey] || 0;
        if (challenge.progressKey === 'streak') {
            return value;
        }
        return value;
    };

    const handleClaimChallenge = (challenge) => {
        const currentValue = getCurrentProgress(challenge);
        const isCompleted = userData.challenges.includes(challenge.id);
        
        if (currentValue >= challenge.required && !isCompleted) {
            updateUserData({ xp: challenge.xp, challenges: [...userData.challenges, challenge.id] });
            console.log(`¡Reto ${challenge.name} reclamado! Ganaste ${challenge.xp} XP.`);
        } else {
            console.log("Aún no cumples los requisitos o ya lo reclamaste.");
        }
    };

    // Lógica para el Reto Personalizado
    const handleCreateCustomChallenge = async () => {
        const goal = parseInt(customGoal);
        const xpReward = parseInt(customXp);

        if (isNaN(goal) || goal < 10 || isNaN(xpReward) || xpReward < 10) {
            setCustomError('Ingresa una meta de minutos (mín. 10) y una recompensa de XP (mín. 10) válidas.');
            return;
        }

        if (userData.customChallenge?.isActive && !userData.customChallenge?.isCompleted) {
            setCustomError('Ya tienes un reto personalizado activo. ¡Termínalo primero!');
            return;
        }

        const newChallenge = {
            isActive: true,
            goalMinutes: goal,
            xpReward: xpReward,
            progress: 0,
            isCompleted: false,
            isVerified: false,
        };

        await updateUserData({ 
            customChallenge: newChallenge,
            // Restablecer minutos para el progreso del reto personalizado.
            minutes: userData.minutes, 
        }); 
        setCustomError('');
        console.log(`Nuevo Reto Personalizado Creado: ${goal} minutos por ${xpReward} XP. (Requiere verificación)`);
    };

    // Lógica de SIMULACIÓN de Subida de Foto (Control)
    const handleUploadPhoto = () => {
        setShowVerificationModal(false);
        // Simulación de subida exitosa y verificación
        updateUserData({ 
            customChallenge: { ...userData.customChallenge, isVerified: true },
            xp: userData.customChallenge.xpReward // Otorgamos el XP aquí
        });
        console.log("Foto de reto subida y verificada con éxito (simulado). XP otorgado.");
    };

    const customChallengeData = userData.customChallenge || initialUserData.customChallenge.customChallenge;
    const customProgress = customChallengeData.goalMinutes > 0 
        ? Math.min(100, (customChallengeData.progress / customChallengeData.goalMinutes) * 100)
        : 0;
    
    // Determine el estado del botón de Reclamar/Verificar
    const isReadyToVerify = customChallengeData.isActive && customChallengeData.isCompleted && !customChallengeData.isVerified;
    const isClaimable = customChallengeData.isActive && customChallengeData.isCompleted && customChallengeData.isVerified;

    return (
        <div className="p-4 space-y-6">
            <h2 className="text-2xl font-bold text-gray-800">Tus Desafíos 🎯</h2>

            {/* MODAL DE VERIFICACIÓN */}
            {showVerificationModal && (
                <VerificationModal onConfirm={handleUploadPhoto} onClose={() => setShowVerificationModal(false)} />
            )}

            {/* Sección de Reto Personalizado */}
            <div className="bg-indigo-50 p-4 rounded-xl shadow-inner border border-indigo-200">
                <h3 className="text-xl font-bold text-indigo-700 mb-3 flex items-center">
                    <DumbbellIcon className="w-6 h-6 mr-2" /> Desafío Personalizado
                </h3>
                
                {customChallengeData.isActive ? (
                    // Vista de Progreso del Reto Activo
                    <div>
                        <p className="font-semibold text-gray-700">Meta: {customChallengeData.goalMinutes} minutos</p>
                        <p className="text-sm text-gray-500 mb-3">Recompensa: {customChallengeData.xpReward} XP</p>
                        
                        <div className="w-full bg-gray-300 rounded-full h-3">
                            <div
                                className="bg-indigo-500 h-3 rounded-full transition-all duration-500"
                                style={{ width: `${customProgress}%` }}
                            ></div>
                        </div>
                        <p className="text-sm text-gray-600 mt-2">
                            {customChallengeData.progress} de {customChallengeData.goalMinutes} minutos ({customProgress.toFixed(0)}%)
                        </p>

                        <div className="mt-4">
                            {customChallengeData.isCompleted ? (
                                isClaimable ? (
                                    <p className="text-center font-bold text-green-600 bg-green-100 p-2 rounded-lg flex items-center justify-center">
                                        <CheckCircleIcon className="w-5 h-5 mr-1" /> ¡VERIFICADO Y RECLAMADO!
                                    </p>
                                ) : (
                                    <button
                                        onClick={() => setShowVerificationModal(true)}
                                        className="w-full py-2 bg-yellow-500 text-gray-800 font-bold rounded-full shadow-md hover:bg-yellow-600 transition duration-200 flex items-center justify-center"
                                        disabled={customChallengeData.isVerified}
                                    >
                                        <CameraIcon className="w-5 h-5 mr-2" /> 
                                        {customChallengeData.isVerified ? 'EN ESPERA (Ya Verificado)' : 'VERIFICAR CON FOTO (¡Listo!)'}
                                    </button>
                                )
                            ) : (
                                <p className="text-center text-sm text-gray-500 italic">Sigue entrenando para completar la meta.</p>
                            )}
                            
                            {/* Botón para crear nuevo reto después de completar/verificar */}
                            {(isClaimable || !customChallengeData.isCompleted) && (
                                <button
                                    onClick={() => {
                                        updateUserData({ 
                                            customChallenge: { isActive: false, goalMinutes: 0, xpReward: 0, progress: 0, isCompleted: false, isVerified: false } 
                                        });
                                        setCustomError('');
                                    }}
                                    className="mt-2 w-full py-2 bg-gray-200 text-gray-700 text-sm font-bold rounded-full hover:bg-gray-300 transition duration-200"
                                >
                                    Crear Nuevo Reto
                                </button>
                            )}
                        </div>
                    </div>
                ) : (
                    // Vista para Crear Reto
                    <div>
                        <p className="text-sm text-gray-600 mb-3">Define tu meta, la recompensa y sube una foto al finalizar para verificación.</p>
                        <div className="flex space-x-2 mb-3">
                            <input
                                type="number"
                                placeholder="Minutos (mín. 10)"
                                value={customGoal}
                                onChange={(e) => setCustomGoal(e.target.value)}
                                className="w-1/2 p-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 text-sm"
                                min="10"
                            />
                            <input
                                type="number"
                                placeholder="XP (mín. 10)"
                                value={customXp}
                                onChange={(e) => setCustomXp(e.target.value)}
                                className="w-1/2 p-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 text-sm"
                                min="10"
                            />
                        </div>
                        {customError && <p className="text-red-500 text-xs mb-3">{customError}</p>}
                        <button
                            onClick={handleCreateCustomChallenge}
                            className="w-full py-2 bg-indigo-600 text-white font-bold rounded-full shadow-md hover:bg-indigo-700 transition duration-200"
                        >
                            Activar Desafío
                        </button>
                    </div>
                )}
            </div>
            
            {/* Lista de Retos Predeterminados */}
            <h3 className="text-xl font-bold text-gray-800 border-b pb-2">Retos de la Comunidad</h3>

            <div className="space-y-3">
                {challenges.map(challenge => {
                    const isCompleted = userData.challenges.includes(challenge.id);
                    const currentValue = getCurrentProgress(challenge);
                    const progress = Math.min(100, (currentValue / challenge.required) * 100);
                    const isReadyToClaim = currentValue >= challenge.required && !isCompleted;
                    const claimButtonText = isCompleted ? 'Reclamado' : isReadyToClaim ? '¡Reclamar!' : 'Progresando...';

                    return (
                        <div key={challenge.id} className={`bg-white p-4 rounded-xl shadow-lg transition duration-200 border-l-4 ${isCompleted ? 'border-green-400 opacity-80' : 'border-purple-400 hover:shadow-xl'}`}>
                            <div className="flex justify-between items-center">
                                <span className="text-2xl mr-3">{challenge.icon}</span>
                                <div className="flex-1">
                                    <p className="font-semibold text-gray-800">{challenge.name}</p>
                                    <p className="text-xs text-gray-500">{challenge.type === 'weekly' ? 'Semanal' : 'Acumulado'} · +{challenge.xp} XP</p>
                                </div>
                                <button
                                    onClick={() => handleClaimChallenge(challenge)}
                                    disabled={!isReadyToClaim}
                                    className={`ml-4 text-sm font-bold py-2 px-4 rounded-full transition duration-200 ${isCompleted 
                                        ? 'bg-gray-400 text-white cursor-default'
                                        : isReadyToClaim 
                                            ? 'bg-yellow-500 text-gray-800 hover:bg-yellow-600'
                                            : 'bg-indigo-500 text-white opacity-90'
                                    }`}
                                >
                                    {claimButtonText}
                                </button>
                            </div>
                            <div className="w-full bg-gray-200 rounded-full h-2 mt-2">
                                <div
                                    className={`h-2 rounded-full ${isCompleted ? 'bg-green-500' : 'bg-purple-500'}`}
                                    style={{ width: `${progress}%` }}
                                ></div>
                            </div>
                            <p className="text-xs text-gray-500 mt-1">
                                {currentValue} de {challenge.required} {challenge.progressKey} ({progress.toFixed(0)}%)
                            </p>
                        </div>
                    );
                })}
            </div>
        </div>
    );
};

const VerificationModal = ({ onConfirm, onClose }) => {
    return (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
            <div className="bg-white p-6 rounded-xl shadow-2xl max-w-sm w-full text-center">
                <CameraIcon className="w-10 h-10 text-indigo-600 mx-auto mb-4"/>
                <h3 className="text-xl font-bold mb-2">Verificación de Reto</h3>
                <p className="text-gray-600 mb-4 text-sm">
                    Para reclamar tu XP de reto personalizado, **simularemos** la subida de una foto que verifique tu actividad.
                </p>
                <div className="flex flex-col space-y-3">
                    <button
                        onClick={onConfirm}
                        className="w-full py-2 bg-indigo-500 text-white font-bold rounded-full hover:bg-indigo-600 transition"
                    >
                        Simular Subida de Foto y Reclamar XP
                    </button>
                    <button
                        onClick={onClose}
                        className="w-full py-2 text-gray-600 font-semibold rounded-full border border-gray-300 hover:bg-gray-100 transition"
                    >
                        Cancelar
                    </button>
                </div>
            </div>
        </div>
    );
};

// --- 3. RECOMPENSAS ---
const RewardsScreen = () => {
    const { userData } = useAppContext();
    
    const achievementsList = [
        { id: 'first_activation', name: 'Primera Activación', desc: 'Completa tu registro y tu primer entrenamiento.', xpNeeded: 10, icon: '🌟' },
        { id: 'weekly_warrior', name: 'Guerrero Semanal', desc: 'Alcanza los 50 XP. Demuestras constancia.', xpNeeded: 50, icon: '🛡️' },
        { id: 'streak_initiated', name: 'Racha Iniciada', desc: 'Alcanza los 100 XP. La consistencia es clave.', xpNeeded: 100, icon: '🔥' },
        { id: 'maestro_tiempo', name: 'Maestro del Tiempo', desc: 'Alcanza los 250 XP. Eres un experto en rutinas exprés.', xpNeeded: 250, icon: '⏱️' },
        { id: 'ultra_constancia', name: 'Ultra Constancia', desc: 'Alcanza los 500 XP. ¡Estás imparable!', xpNeeded: 500, icon: '✨' },
        { id: 'leyenda_express', name: 'Leyenda Express', desc: 'Alcanza los 1000 XP. ¡Felicidades, Leyenda!', xpNeeded: 1000, icon: '👑' },
    ];

    return (
        <div className="p-4 space-y-5">
            <h2 className="text-2xl font-bold text-gray-800">Logros Desbloqueados 🏆</h2>
            <p className="text-sm text-gray-600 bg-yellow-50 p-3 rounded-lg border-l-4 border-yellow-400 font-medium">Tu progreso se mide con estas insignias. ¡Acumula más XP para desbloquear todos!</p>

            <div className="grid grid-cols-1 gap-4">
                {achievementsList.map(ach => {
                    const isUnlocked = userData.xp >= ach.xpNeeded; // Usamos XP para determinar si está desbloqueado visualmente
                    return (
                        <div key={ach.id} className={`bg-white p-4 rounded-xl shadow-lg flex items-center transition duration-300 ${!isUnlocked ? 'opacity-50 grayscale' : 'border-2 border-yellow-500 shadow-xl'}`}>
                            <span className={`text-4xl mr-4 ${isUnlocked ? '' : 'text-gray-400'}`}>{ach.icon}</span>
                            <div className="flex-1">
                                <p className="font-bold text-gray-800">{ach.name}</p>
                                <p className="text-sm text-gray-500">{ach.desc}</p>
                            </div>
                            <div className="text-right">
                                <span className={`text-xs font-bold px-2 py-1 rounded-full ${isUnlocked ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700'}`}>
                                    {isUnlocked ? 'Desbloqueado' : `${ach.xpNeeded} XP`}
                                </span>
                            </div>
                        </div>
                    );
                })}
            </div>
        </div>
    );
};

// --- 4. PERFIL ---
const ProfileScreen = () => {
    const { userData, updateUserData } = useAppContext();
    const { displayName, email, age, trainings, minutes, calories, achievements, goals, userId, level, xp } = userData;
    const [weeklyGoal, setWeeklyGoal] = useState(goals?.weeklyMinutes || 60);

    const handleLogout = async () => {
        if (auth) {
            await updateUserData({ displayName: '' });
            console.log("Sesión cerrada. La próxima vez iniciarás con un nuevo Onboarding.");
        }
    };

    const handleSaveGoal = async () => {
        const goalValue = parseInt(weeklyGoal);
        if (isNaN(goalValue) || goalValue < 10) {
            console.error("Meta inválida.");
            return;
        }
        await updateUserData({ goals: { weeklyMinutes: goalValue } });
        console.log(`Meta semanal guardada: ${goalValue} minutos.`);
    };

    return (
        <div className="p-4 space-y-5">
            <h2 className="text-2xl font-bold text-gray-800">Mi Perfil 👤</h2>
            
            {/* Tarjeta de Información Básica */}
            <div className="bg-white p-4 rounded-xl shadow-xl border border-gray-100">
                <p className="text-xl font-bold text-indigo-600">{displayName}</p>
                <div className="mt-2 text-sm space-y-1 text-gray-700">
                    <p>Email: <span className="font-semibold">{email || 'N/A'}</span></p>
                    <p>Edad: <span className="font-semibold">{age || 'N/A'}</span></p>
                    <p>ID: <span className="font-mono text-xs select-all">{userId || 'Cargando...'}</span></p>
                </div>
            </div>

            {/* Estadísticas de Juego */}
            <h3 className="text-xl font-bold text-gray-800 pt-2 border-b pb-2">Progreso y Ranking</h3>
            <div className="grid grid-cols-2 gap-3">
                <StatBox label="Nivel Actual" value={level} unit="Nivel" />
                <StatBox label="XP Total" value={xp} unit="Puntos" />
                <StatBox label="Logros Desbloqueados" value={achievements.length} unit="total" />
                <StatBox label="Racha Máxima" value={userData.streak} unit="días" />
            </div>


            {/* Estadísticas de Entrenamiento */}
            <h3 className="text-xl font-bold text-gray-800 pt-2 border-b pb-2">Registro de Actividad</h3>
            <div className="grid grid-cols-3 gap-3">
                <StatCard icon={<ActivityIcon className="w-6 h-6 text-indigo-500" />} label="Entrenos" value={trainings} unit="sesiones" />
                <StatCard icon={<ClockIcon className="w-6 h-6 text-pink-500" />} label="Minutos" value={minutes} unit="acum." />
                <StatCard icon={<FireIcon className="w-6 h-6 text-orange-500" />} label="Calorías" value={calories} unit="kcal est." />
            </div>

            {/* Meta Semanal Personalizable */}
            <h3 className="text-xl font-bold text-gray-800 pt-2 border-b pb-2">Meta Personal</h3>
            <div className="bg-white p-4 rounded-xl shadow-lg flex items-center space-x-3 border border-gray-100">
                <input
                    type="number"
                    value={weeklyGoal}
                    onChange={(e) => setWeeklyGoal(e.target.value)}
                    className="w-20 p-2 border border-gray-300 rounded-lg text-center font-bold text-indigo-600 focus:ring-purple-500 focus:ring-2"
                />
                <span className="text-gray-600">minutos semanales (tu meta)</span>
                <button 
                    onClick={handleSaveGoal}
                    className="bg-purple-500 text-white font-bold py-2 px-4 rounded-full shadow-md hover:bg-purple-600 transition duration-200"
                >
                    Guardar
                </button>
            </div>

            {/* Opción de Cerrar Sesión */}
            <button 
                onClick={handleLogout}
                className="w-full mt-6 py-3 bg-red-500 text-white font-bold rounded-full shadow-lg flex items-center justify-center hover:bg-red-600 transition duration-200"
            >
                <LogOutIcon className="w-5 h-5 mr-2" />
                Cerrar Sesión
            </button>
        </div>
    );
};

const StatBox = ({ label, value, unit }) => (
    <div className="bg-gray-100 p-3 rounded-lg shadow-sm border border-gray-200">
        <p className="text-2xl font-bold text-purple-600">{value}</p>
        <p className="text-xs text-gray-600">{label} ({unit})</p>
    </div>
);


// --- NAVEGACIÓN Y LAYOUT ---
const TabBar = () => {
    const { currentPage, setCurrentPage } = useAppContext();
    const tabs = [
        { name: 'home', label: 'Inicio', icon: HomeIcon },
        { name: 'challenges', label: 'Retos', icon: TargetIcon },
        { name: 'rewards', label: 'Recompensas', icon: TrophyIcon },
        { name: 'profile', label: 'Perfil', icon: UserIcon },
    ];

    return (
        <div className="fixed bottom-0 left-0 right-0 max-w-sm mx-auto bg-white border-t-4 border-indigo-100 shadow-2xl flex justify-around p-2 z-20 rounded-t-xl">
            {tabs.map((tab) => (
                <button
                    key={tab.name}
                    onClick={() => setCurrentPage(tab.name)}
                    className={`flex flex-col items-center p-2 rounded-lg transition-colors ${
                        currentPage === tab.name ? 'text-indigo-600 font-bold bg-indigo-50' : 'text-gray-500 hover:text-indigo-400'
                    }`}
                >
                    <tab.icon className="w-6 h-6" />
                    <span className="text-xs mt-1">{tab.label}</span>
                </button>
            ))}
        </div>
    );
};

const Layout = ({ children }) => {
    return (
        <div className="min-h-screen bg-gray-50 pb-24">
            <header className="fixed top-0 left-0 right-0 max-w-sm mx-auto bg-white border-b border-gray-200 p-4 shadow-lg z-10">
                <h1 className="text-xl font-extrabold text-center text-indigo-600">Rutina Express</h1>
            </header>
            <main className="pt-16 max-w-sm mx-auto">
                {children}
            </main>
        </div>
    );
};

// --- COMPONENTE PRINCIPAL ---
const App = () => {
    const { isLoading, currentPage, userData } = useAppContext();

    if (isLoading) {
        return (
            <div className="min-h-screen flex items-center justify-center bg-gray-100">
                <div className="text-indigo-600 text-xl font-semibold">Cargando aplicación y datos...</div>
            </div>
        );
    }

    if (!userData.displayName || currentPage === 'onboarding') {
        return <OnboardingScreen />;
    }

    let ScreenComponent;
    switch (currentPage) {
        case 'challenges':
            ScreenComponent = ChallengesScreen;
            break;
        case 'rewards':
            ScreenComponent = RewardsScreen;
            break;
        case 'profile':
            ScreenComponent = ProfileScreen;
            break;
        case 'home':
        default:
            ScreenComponent = HomeScreen;
            break;
    }

    return (
        <Layout>
            <ScreenComponent />
            <TabBar />
        </Layout>
    );
};

// Envolver la App con el proveedor de contexto
const RootApp = () => (
    <AppProvider>
        <App />
    </AppProvider>
);

export default RootApp;
