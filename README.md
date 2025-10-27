<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rutina Express - Fitness Gamificado</title>
    <!-- Carga de Librerías -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
    <!-- Carga de Firebase: Inicialización de la base de datos y autenticación -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, onSnapshot, updateDoc, collection, query, where, getDocs } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import * as Lucide from "https://unpkg.com/lucide@latest/dist/umd/lucide.js";

        // Hacer que las utilidades de Firebase y Lucide estén globalmente disponibles
        window.FirebaseServices = { initializeApp, getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, getFirestore, doc, setDoc, onSnapshot, updateDoc, collection, query, where, getDocs };
        window.LucideIcons = Lucide;
    </script>

    <style>
        /* Fuentes */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f7f7f9;
        }
        /* Garantiza que la aplicación ocupe toda la pantalla */
        #root {
            min-height: 100vh;
        }
    </style>
</head>
<body>
    <div id="root"></div>

    <!-- El código React (JSX) va dentro de este script para que Babel lo compile -->
    <script type="text/babel">
        const { useState, useEffect, useCallback, useMemo } = React;
        const { createRoot } = ReactDOM;

        // Recuperamos las utilidades de Firebase del ámbito global
        const { initializeApp, getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, getFirestore, doc, setDoc, onSnapshot, updateDoc, collection, query, where, getDocs } = window.FirebaseServices || {};
        const { Home, Zap, Award, User, LogOut, TrendingUp, Target, Clock, Dumbbell, Heart, CheckCircle, XCircle } = window.LucideIcons || {};


        // --- CONFIGURACIÓN DE FIREBASE (MANDATORIA) ---
        // Variables globales (si existen)
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

        let db, auth, app;
        if (Object.keys(firebaseConfig).length > 0 && initializeApp) {
            app = initializeApp(firebaseConfig);
            db = getFirestore(app);
            auth = getAuth(app);
        } else {
            console.error("Firebase no está configurado o las librerías no cargaron. La persistencia de datos no funcionará.");
        }
        
        // --- CONSTANTES Y LÓGICA DE GAMIFICACIÓN ---

        const MAX_XP_PER_LEVEL = 1000;

        const ACHIEVEMENTS = [
            { id: 'firstActivation', name: 'Primera Activación', description: 'Completa tu primer entrenamiento.', xp: 100, icon: Zap },
            { id: 'weeklyWarrior', name: 'Guerrero Semanal', description: 'Alcanza tu meta semanal de minutos.', xp: 500, icon: Dumbbell },
            { id: 'streakInitiate', name: 'Racha Iniciada', description: 'Entrena 3 días seguidos.', xp: 300, icon: TrendingUp },
            { id: 'caloryBurner', name: 'Quema Calorías', description: 'Quema más de 500 calorías totales.', xp: 200, icon: Heart },
            { id: 'timeMaster', name: 'Maestro del Tiempo', description: 'Acumula 180 minutos de entrenamiento.', xp: 800, icon: Clock },
        ];

        const DEFAULT_CHALLENGES = [
            { id: 'daily_5', name: 'Misión Diaria', description: 'Entrena por lo menos 5 minutos hoy.', goal: 5, type: 'daily', xpReward: 50, progress: 0, isCompleted: false },
            { id: 'weekly_30', name: 'Reto Semanal', description: 'Acumula 30 minutos esta semana.', goal: 30, type: 'weekly', xpReward: 200, progress: 0, isCompleted: false },
        ];

        const INITIAL_USER_DATA = {
            name: '',
            email: '',
            age: 0,
            level: 1,
            xp: 0,
            streak: 0,
            lastWorkoutDate: 0,
            weeklyGoalMinutes: 150, // Meta predeterminada
            totalMinutes: 0,
            totalWorkouts: 0,
            totalCalories: 0,
            achievements: ACHIEVEMENTS.reduce((acc, a) => ({ ...acc, [a.id]: false }), {}),
            history: [],
            challenges: DEFAULT_CHALLENGES,
            isRegistered: false,
        };

        // Función para calcular el nivel y el progreso de XP
        const calculateLevelProgress = (xp) => {
            const level = Math.floor(xp / MAX_XP_PER_LEVEL) + 1;
            const xpCurrent = xp % MAX_XP_PER_LEVEL;
            const progress = (xpCurrent / MAX_XP_PER_LEVEL) * 100;
            return { level, xpCurrent, progress };
        };

        // Función de utilidad para obtener la fecha en formato YYYY-MM-DD
        const getTodayDate = () => {
            return new Date().toISOString().split('T')[0];
        };


        // --- COMPONENTES DE VISTA REUTILIZABLES ---

        const Header = ({ title, user }) => (
            <div className="p-4 pt-8 bg-white shadow-sm border-b border-gray-100">
                <h1 className="text-3xl font-bold text-gray-800">
                    {title === 'Inicio' ? `Hola, ${user?.name || 'Guerrero'}` : title}
                </h1>
                {title === 'Inicio' && (
                    <p className="text-sm text-indigo-500 mt-1">¡A cumplir la meta!</p>
                )}
            </div>
        );

        const StatCard = ({ title, value, unit, icon: Icon, color = 'text-indigo-600' }) => (
            <div className="bg-white p-4 rounded-xl shadow-md border border-gray-100 flex items-center space-x-4">
                <div className={`p-3 rounded-full bg-opacity-10 ${color} bg-indigo-500`}>
                    {Icon && <Icon size={24} />}
                </div>
                <div>
                    <p className="text-xs text-gray-500 font-medium">{title}</p>
                    <p className="text-xl font-bold text-gray-800">{value}
                        <span className="text-sm font-normal text-gray-500 ml-1">{unit}</span>
                    </p>
                </div>
            </div>
        );

        const ProgressBar = ({ progress, label, color = 'bg-indigo-500' }) => (
            <div className="w-full">
                <div className="flex justify-between mb-1 text-sm font-medium text-gray-700">
                    <span>{label}</span>
                    <span>{Math.round(progress)}%</span>
                </div>
                <div className="w-full bg-gray-200 rounded-full h-2.5">
                    <div 
                        className={`h-2.5 rounded-full transition-all duration-500 ease-out ${color}`} 
                        style={{ width: `${Math.min(100, progress)}%` }}>
                    </div>
                </div>
            </div>
        );


        // --- PANTALLAS DE LA APLICACIÓN ---

        // Pantalla 1: Onboarding y Registro
        const OnboardingScreen = ({ onRegister }) => {
            const [step, setStep] = useState(1);
            const [name, setName] = useState('');
            const [email, setEmail] = useState('');
            const [age, setAge] = useState('');

            const slides = [
                {
                    title: "Desafía tu Rutina",
                    description: "Convierte tus entrenamientos en una aventura con XP y niveles.",
                    icon: Target,
                },
                {
                    title: "Rachas y Recompensas",
                    description: "Mantén tu racha diaria y desbloquea logros por tus esfuerzos.",
                    icon: Award,
                },
                {
                    title: "Progreso Visible",
                    description: "Mira tus estadísticas, minutos acumulados y calorías quemadas.",
                    icon: TrendingUp,
                },
            ];

            const handleNext = () => {
                if (step < slides.length) {
                    setStep(step + 1);
                }
            };

            const handleRegistration = () => {
                if (!name || !email || !age || isNaN(parseInt(age)) || parseInt(age) < 18) {
                    console.error("Por favor, completa todos los campos válidos.");
                    // Usar un componente modal o un div de error en lugar de alert()
                    return;
                }
                onRegister(name, email, parseInt(age));
            };

            const currentSlide = slides[step - 1];

            return (
                <div className="min-h-screen bg-gradient-to-br from-indigo-500 to-indigo-800 flex flex-col items-center justify-center p-6">
                    <div className="w-full max-w-sm bg-white p-8 rounded-2xl shadow-2xl">
                        <div className="flex justify-center mb-6">
                            {currentSlide.icon && <currentSlide.icon size={60} className="text-indigo-600" />}
                        </div>
                        
                        <h2 className="text-3xl font-extrabold text-center text-gray-800 mb-2">{currentSlide.title}</h2>
                        <p className="text-center text-gray-600 mb-8">{currentSlide.description}</p>
                        
                        {/* Indicadores de diapositiva */}
                        <div className="flex justify-center space-x-2 mb-8">
                            {slides.map((_, index) => (
                                <div key={index} 
                                    className={`h-2 rounded-full transition-all duration-300 ${index + 1 === step ? 'w-6 bg-indigo-500' : 'w-2 bg-gray-300'}`}>
                                </div>
                            ))}
                        </div>

                        {step <= slides.length - 1 && (
                            <button 
                                onClick={handleNext}
                                className="w-full py-3 bg-indigo-600 text-white font-semibold rounded-xl hover:bg-indigo-700 transition duration-200 shadow-md">
                                Siguiente
                            </button>
                        )}
                        
                        {/* Formulario de Registro (Último paso) */}
                        {step === slides.length && (
                            <div className="space-y-4">
                                <input
                                    type="text"
                                    placeholder="Nombre Completo"
                                    value={name}
                                    onChange={(e) => setName(e.target.value)}
                                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                                />
                                <input
                                    type="email"
                                    placeholder="Correo Electrónico"
                                    value={email}
                                    onChange={(e) => setEmail(e.target.value)}
                                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                                />
                                <input
                                    type="number"
                                    placeholder="Edad (Mín. 18)"
                                    value={age}
                                    onChange={(e) => setAge(e.target.value)}
                                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                                />
                                <button
                                    onClick={handleRegistration}
                                    className="w-full py-3 bg-green-500 text-white font-semibold rounded-xl hover:bg-green-600 transition duration-200 shadow-md"
                                >
                                    ¡Empezar a Ganar XP!
                                </button>
                            </div>
                        )}
                    </div>
                </div>
            );
        };

        // Pantalla 2: Inicio (Home)
        const HomeScreen = ({ user, handleWorkoutComplete }) => {
            const { level, xpCurrent, progress } = calculateLevelProgress(user.xp);
            const [minutes, setMinutes] = useState(15);
            const [description, setDescription] = useState('Rutina rápida');

            const handleComplete = () => {
                if (minutes > 0) {
                    handleWorkoutComplete(minutes, description);
                    setMinutes(15);
                    setDescription('Rutina rápida');
                }
            };

            return (
                <div className="p-4 space-y-6">
                    {/* Tarjeta de Nivel y XP */}
                    <div className="bg-gradient-to-r from-indigo-500 to-purple-600 p-5 rounded-2xl shadow-xl text-white">
                        <p className="text-sm opacity-80">Nivel Actual</p>
                        <h2 className="text-5xl font-extrabold mb-3">
                            {level}
                            <span className="text-xl font-normal ml-2 opacity-90">{user.name}</span>
                        </h2>
                        <ProgressBar progress={progress} label={`XP: ${xpCurrent} / ${MAX_XP_PER_LEVEL}`} color="bg-yellow-400" />
                    </div>

                    {/* Estadísticas Rápidas */}
                    <div className="grid grid-cols-2 gap-4">
                        <StatCard 
                            title="Racha" 
                            value={user.streak} 
                            unit="días" 
                            icon={TrendingUp} 
                            color="text-yellow-600"
                        />
                        <StatCard 
                            title="Entrenamientos" 
                            value={user.totalWorkouts} 
                            unit="total" 
                            icon={Dumbbell} 
                            color="text-green-600"
                        />
                    </div>

                    {/* Registro de Entrenamiento */}
                    <div className="bg-white p-5 rounded-xl shadow-lg border border-gray-100 space-y-4">
                        <h3 className="text-lg font-bold text-gray-800 border-b pb-2 mb-3 flex items-center">
                            {Zap && <Zap size={20} className="mr-2 text-indigo-500" />}
                            Registrar Activación Rápida
                        </h3>
                        
                        <input
                            type="number"
                            placeholder="Minutos de entrenamiento"
                            value={minutes}
                            onChange={(e) => setMinutes(Math.max(0, parseInt(e.target.value) || 0))}
                            className="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                        />
                        <select
                            value={description}
                            onChange={(e) => setDescription(e.target.value)}
                            className="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                        >
                            <option value="Rutina rápida">Rutina rápida (5-15 min)</option>
                            <option value="Cardio Intenso">Cardio Intenso (20+ min)</option>
                            <option value="Fuerza/Pesas">Fuerza/Pesas</option>
                            <option value="Yoga/Estiramiento">Yoga/Estiramiento</option>
                            <option value="Personalizado">Personalizado</option>
                        </select>

                        <button
                            onClick={handleComplete}
                            disabled={minutes <= 0}
                            className="w-full py-3 bg-green-500 text-white font-semibold rounded-xl hover:bg-green-600 transition duration-200 shadow-md disabled:bg-gray-400"
                        >
                            Registrar {minutes} Minutos
                        </button>
                    </div>
                    
                </div>
            );
        };

        // Pantalla 3: Retos
        const ChallengesScreen = ({ user, handleClaimChallenge }) => {
            
            // Filtramos los desafíos que son parte del estado del usuario
            const activeChallenges = useMemo(() => user.challenges || [], [user.challenges]);

            return (
                <div className="p-4 space-y-6">
                    <h2 className="text-2xl font-bold text-gray-800">Retos Activos</h2>

                    {activeChallenges.map((challenge) => {
                        const isClaimable = challenge.progress >= challenge.goal && !challenge.isCompleted;
                        const progressPercentage = (challenge.progress / challenge.goal) * 100;
                        
                        return (
                            <div key={challenge.id} className="bg-white p-4 rounded-xl shadow-lg border border-gray-100 space-y-3">
                                <div className="flex items-center justify-between">
                                    <h3 className="text-lg font-bold text-indigo-600">{challenge.name}</h3>
                                    <span className={`px-2 py-0.5 text-xs font-semibold rounded-full ${challenge.type === 'daily' ? 'bg-red-100 text-red-600' : 'bg-blue-100 text-blue-600'}`}>
                                        {challenge.type === 'daily' ? 'Diario' : 'Semanal'}
                                    </span>
                                </div>
                                
                                <p className="text-gray-600 text-sm">{challenge.description}</p>
                                
                                <ProgressBar 
                                    progress={progressPercentage} 
                                    label={`Progreso: ${challenge.progress} / ${challenge.goal} minutos`}
                                    color={challenge.isCompleted ? 'bg-green-500' : 'bg-indigo-500'}
                                />
                                
                                <div className="pt-2">
                                    {isClaimable ? (
                                        <button
                                            onClick={() => handleClaimChallenge(challenge.id, challenge.xpReward)}
                                            className="w-full py-2 bg-yellow-500 text-white font-semibold rounded-lg hover:bg-yellow-600 transition duration-200"
                                        >
                                            ¡Reclamar {challenge.xpReward} XP!
                                        </button>
                                    ) : challenge.isCompleted ? (
                                        <div className="flex items-center text-green-600 font-semibold justify-center py-2 bg-green-50 rounded-lg">
                                            {CheckCircle && <CheckCircle size={18} className="mr-2" />}
                                            ¡Reto Completado!
                                        </div>
                                    ) : (
                                        <p className="text-sm text-gray-500 text-center">Sigue entrenando para completar este reto.</p>
                                    )}
                                </div>
                            </div>
                        );
                    })}
                </div>
            );
        };

        // Pantalla 4: Recompensas y Logros
        const RewardsScreen = ({ user, level, progress }) => {
            
            // Obtener la lista completa de logros con su estado de desbloqueo
            const achievementsList = ACHIEVEMENTS.map(achievement => ({
                ...achievement,
                isUnlocked: user.achievements[achievement.id] || false,
            }));

            return (
                <div className="p-4 space-y-6">
                    {/* Tarjeta de Nivel */}
                    <div className="bg-gradient-to-r from-purple-600 to-pink-600 p-5 rounded-2xl shadow-xl text-white">
                        <p className="text-sm opacity-80">Tu Potencial</p>
                        <h2 className="text-4xl font-extrabold mb-1">Nivel {level}</h2>
                        <ProgressBar progress={progress} label={`Siguiente Nivel en: ${100 - Math.round(progress)}%`} color="bg-yellow-300" />
                    </div>

                    <h2 className="text-2xl font-bold text-gray-800 border-b pb-2">Logros Desbloqueables</h2>
                    
                    <div className="grid grid-cols-1 gap-4">
                        {achievementsList.map((a) => (
                            <div 
                                key={a.id} 
                                className={`p-4 rounded-xl shadow-md flex items-center space-x-4 transition duration-300 ${a.isUnlocked ? 'bg-yellow-50 border-yellow-300' : 'bg-white border-gray-100 opacity-60'}`}
                            >
                                <div className={`p-3 rounded-full ${a.isUnlocked ? 'bg-yellow-400 text-white' : 'bg-gray-200 text-gray-500'}`}>
                                    {a.icon && <a.icon size={20} />}
                                </div>
                                <div className="flex-1">
                                    <h3 className={`font-bold ${a.isUnlocked ? 'text-yellow-800' : 'text-gray-600'}`}>{a.name}</h3>
                                    <p className="text-sm text-gray-500">{a.description}</p>
                                </div>
                                <div className={`text-xs font-semibold ${a.isUnlocked ? 'text-green-600' : 'text-gray-400'}`}>
                                    {a.isUnlocked ? 'DESBLOQUEADO' : `${a.xp} XP`}
                                </div>
                            </div>
                        ))}
                    </div>
                </div>
            );
        };

        // Pantalla 5: Perfil
        const ProfileScreen = ({ user, handleLogout }) => {
            const { level } = calculateLevelProgress(user.xp);
            const [goal, setGoal] = useState(user.weeklyGoalMinutes);

            const handleGoalUpdate = async () => {
                if (goal > 0 && auth?.currentUser) {
                    try {
                        const userDocRef = doc(db, 'artifacts', appId, 'users', auth.currentUser.uid, 'data', 'user_profile');
                        await updateDoc(userDocRef, { weeklyGoalMinutes: goal });
                        console.log('¡Meta semanal actualizada con éxito!');
                    } catch (error) {
                        console.error("Error al actualizar la meta:", error);
                    }
                }
            };

            const totalAchievements = ACHIEVEMENTS.length;
            const unlockedAchievements = Object.values(user.achievements).filter(Boolean).length;
            const achievementProgress = (unlockedAchievements / totalAchievements) * 100;

            return (
                <div className="p-4 space-y-6">
                    {/* Tarjeta de Perfil */}
                    <div className="bg-white p-6 rounded-2xl shadow-xl border-t-4 border-indigo-500 text-center">
                        {User && <User size={60} className="mx-auto mb-3 text-indigo-500" />}
                        <h2 className="text-2xl font-bold text-gray-800">{user.name}</h2>
                        <p className="text-sm text-gray-500">{user.email}</p>
                        {/* Muestra el ID de usuario para apps multi-usuario */}
                        <p className="text-xs text-gray-400 mt-1">ID de Usuario: {auth?.currentUser?.uid || 'N/A'}</p>
                        <div className="mt-4 inline-block px-3 py-1 bg-indigo-100 text-indigo-600 font-semibold rounded-full text-sm">
                            NIVEL {level}
                        </div>
                    </div>

                    {/* Panel de Estadísticas Detalladas */}
                    <h3 className="text-xl font-bold text-gray-800 border-b pb-2">Estadísticas Clave</h3>
                    <div className="grid grid-cols-2 gap-4">
                        <StatCard title="Minutos Acumulados" value={user.totalMinutes} unit="min" icon={Clock} color="text-indigo-600" />
                        <StatCard title="Calorías Quemadas" value={user.totalCalories} unit="kcal" icon={Heart} color="text-red-600" />
                        <StatCard title="Logros Desbloqueados" value={unlockedAchievements} unit={`de ${totalAchievements}`} icon={Award} color="text-yellow-600" />
                        <StatCard title="Racha Actual" value={user.streak} unit="días" icon={TrendingUp} color="text-green-600" />
                    </div>

                    {/* Meta Semanal Personalizable */}
                    <div className="bg-white p-4 rounded-xl shadow-lg border border-gray-100 space-y-3">
                        <h3 className="font-bold text-gray-800 flex items-center">{Target && <Target size={20} className="mr-2 text-indigo-500" />} Meta Semanal (Minutos)</h3>
                        <div className="flex space-x-2">
                            <input
                                type="number"
                                value={goal}
                                onChange={(e) => setGoal(Math.max(0, parseInt(e.target.value) || 0))}
                                className="flex-1 p-2 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
                            />
                            <button
                                onClick={handleGoalUpdate}
                                className="py-2 px-4 bg-indigo-500 text-white font-semibold rounded-lg hover:bg-indigo-600 transition duration-200"
                            >
                                Guardar
                            </button>
                        </div>
                        <p className="text-xs text-gray-500">Minutos a completar cada semana: {user.weeklyGoalMinutes}</p>
                    </div>

                    {/* Historial de Actividad */}
                    <h3 className="text-xl font-bold text-gray-800 border-b pb-2">Historial Reciente</h3>
                    <div className="space-y-2">
                        {user.history.slice(0, 5).map((entry, index) => (
                            <div key={index} className="bg-white p-3 rounded-lg shadow-sm flex justify-between items-center text-sm border-l-4 border-green-400">
                                <div>
                                    <p className="font-semibold">{entry.description}</p>
                                    <p className="text-xs text-gray-500">{new Date(entry.date).toLocaleDateString()}</p>
                                </div>
                                <div className="text-right">
                                    <p className="font-bold text-green-600">{entry.minutes} min</p>
                                    <p className="text-xs text-indigo-500">+{entry.xpGained} XP</p>
                                </div>
                            </div>
                        ))}
                        {user.history.length === 0 && <p className="text-center text-gray-500 text-sm py-4">Aún no tienes entrenamientos registrados.</p>}
                    </div>

                    {/* Botón de Cerrar Sesión */}
                    <button
                        onClick={handleLogout}
                        className="w-full py-3 mt-8 bg-red-500 text-white font-semibold rounded-xl hover:bg-red-600 transition duration-200 flex items-center justify-center shadow-md"
                    >
                        {LogOut && <LogOut size={20} className="mr-2" />}
                        Cerrar Sesión
                    </button>
                </div>
            );
        };

        // --- NAVEGACIÓN Y COMPONENTE PRINCIPAL ---

        const TabNavigation = ({ currentScreen, setCurrentScreen }) => {
            const tabs = [
                { name: 'Inicio', icon: Home, screen: 'Home' },
                { name: 'Retos', icon: Target, screen: 'Challenges' },
                { name: 'Recompensas', icon: Award, screen: 'Rewards' },
                { name: 'Perfil', icon: User, screen: 'Profile' },
            ];

            return (
                <div className="fixed bottom-0 left-0 right-0 bg-white border-t border-gray-200 shadow-2xl z-10">
                    <div className="flex justify-around max-w-lg mx-auto">
                        {tabs.map((tab) => {
                            const isActive = currentScreen === tab.screen;
                            return (
                                <button
                                    key={tab.screen}
                                    onClick={() => setCurrentScreen(tab.screen)}
                                    className={`flex flex-col items-center p-3 sm:p-4 transition duration-200 ${isActive ? 'text-indigo-600' : 'text-gray-500 hover:text-indigo-500'}`}
                                >
                                    {tab.icon && <tab.icon size={24} />}
                                    <span className="text-xs mt-1 hidden sm:block">{tab.name}</span>
                                </button>
                            );
                        })}
                    </div>
                </div>
            );
        };


        const App = () => {
            const [currentScreen, setCurrentScreen] = useState('Loading'); // 'Onboarding', 'Home', 'Challenges', 'Rewards', 'Profile'
            const [user, setUser] = useState(null);
            const [userId, setUserId] = useState(null);
            const [error, setError] = useState('');
            const [isAuthReady, setIsAuthReady] = useState(false);

            // --- LÓGICA DE FIREBASE Y ESTADO ---

            // 1. Inicialización y Autenticación
            useEffect(() => {
                if (!app) {
                    setError("Error: Firebase no está configurado.");
                    return;
                }

                const authenticate = async () => {
                    try {
                        if (initialAuthToken) {
                            await signInWithCustomToken(auth, initialAuthToken);
                        } else {
                            await signInAnonymously(auth);
                        }
                    } catch (e) {
                        console.error("Error de autenticación de Firebase:", e);
                        setError("Fallo la autenticación. Intenta recargar.");
                    }
                };

                // Listener de estado de autenticación
                const unsubscribeAuth = onAuthStateChanged(auth, (user) => {
                    if (user) {
                        setUserId(user.uid);
                    } else {
                        setUserId(null);
                    }
                    setIsAuthReady(true);
                });

                authenticate();
                return () => unsubscribeAuth();
            }, []);

            // 2. Listener de Datos del Usuario (Firestore)
            useEffect(() => {
                if (!db || !userId) return;

                const userDocRef = doc(db, 'artifacts', appId, 'users', userId, 'data', 'user_profile');

                const unsubscribeSnapshot = onSnapshot(userDocRef, (docSnap) => {
                    if (docSnap.exists()) {
                        const data = docSnap.data();
                        setUser(data);
                        // Si ya está registrado, vamos a Home, si no, a Onboarding
                        setCurrentScreen(data.isRegistered ? 'Home' : 'Onboarding');
                    } else {
                        // Si no existe, forzamos Onboarding
                        setUser(INITIAL_USER_DATA);
                        setCurrentScreen('Onboarding');
                    }
                }, (err) => {
                    console.error("Error al escuchar datos:", err);
                    setError("Error al cargar datos del usuario.");
                });

                return () => unsubscribeSnapshot();
            }, [userId]);

            // --- HANDLERS DE LA APLICACIÓN ---

            // Maneja el registro del usuario desde Onboarding
            const handleRegistration = useCallback(async (name, email, age) => {
                if (!userId) {
                    setError("Error de autenticación. Intenta recargar.");
                    return;
                }
                
                const newUserData = {
                    ...INITIAL_USER_DATA,
                    name,
                    email,
                    age,
                    isRegistered: true,
                };

                try {
                    const userDocRef = doc(db, 'artifacts', appId, 'users', userId, 'data', 'user_profile');
                    await setDoc(userDocRef, newUserData);
                    setUser(newUserData);
                    setCurrentScreen('Home');
                } catch (e) {
                    console.error("Error al registrar usuario:", e);
                    setError("No se pudo guardar la información del perfil.");
                }
            }, [userId]);


            // Actualiza datos en Firestore
            const updateFirestore = useCallback(async (updates) => {
                if (!userId) return;
                try {
                    const userDocRef = doc(db, 'artifacts', appId, 'users', userId, 'data', 'user_profile');
                    await updateDoc(userDocRef, updates);
                } catch (e) {
                    console.error("Error al actualizar Firestore:", e);
                    setError("Fallo al actualizar tus datos.");
                }
            }, [userId]);

            // Lógica para registrar un entrenamiento y calcular XP/Logros/Racha/Retos
            const handleWorkoutComplete = useCallback(async (minutes, description) => {
                if (!user || minutes <= 0) return;

                const xpGained = minutes * 5; // 5 XP por minuto
                const caloriesBurned = minutes * 7; // Simulación: 7 calorías por minuto

                let updates = {
                    totalMinutes: user.totalMinutes + minutes,
                    totalWorkouts: user.totalWorkouts + 1,
                    totalCalories: user.totalCalories + caloriesBurned,
                    xp: user.xp + xpGained,
                    history: [{ date: Date.now(), minutes, calories: caloriesBurned, xpGained, description }, ...user.history.slice(0, 19)], // Mantener historial corto
                };

                let newAchievements = { ...user.achievements };
                const today = getTodayDate();
                
                // 1. Lógica de Rachas (Streak)
                let newStreak = user.streak;
                const yesterdayDate = new Date();
                yesterdayDate.setDate(yesterdayDate.getDate() - 1);
                const yesterday = yesterdayDate.toISOString().split('T')[0];
                
                if (user.lastWorkoutDate === 0 || user.lastWorkoutDate === today) {
                    newStreak = user.lastWorkoutDate === 0 ? 1 : newStreak;
                } else if (user.lastWorkoutDate === yesterday) {
                    newStreak += 1;
                } else {
                    newStreak = 1; // Racha rota
                }
                
                updates.streak = newStreak;
                updates.lastWorkoutDate = today;

                // 2. Revisión de Logros
                let extraXp = 0;
                
                if (!newAchievements.firstActivation) {
                    newAchievements.firstActivation = true;
                    extraXp += ACHIEVEMENTS.find(a => a.id === 'firstActivation').xp;
                }
                if (!newAchievements.streakInitiate && newStreak >= 3) {
                    newAchievements.streakInitiate = true;
                    extraXp += ACHIEVEMENTS.find(a => a.id === 'streakInitiate').xp;
                }
                if (!newAchievements.caloryBurner && updates.totalCalories >= 500) {
                    newAchievements.caloryBurner = true;
                    extraXp += ACHIEVEMENTS.find(a => a.id === 'caloryBurner').xp;
                }
                if (!newAchievements.timeMaster && updates.totalMinutes >= 180) {
                    newAchievements.timeMaster = true;
                    extraXp += ACHIEVEMENTS.find(a => a.id === 'timeMaster').xp;
                }

                updates.xp += extraXp;
                updates.achievements = newAchievements;

                // 3. Revisión de Retos
                let newChallenges = user.challenges.map(c => {
                    if (!c.isCompleted) {
                        c.progress = c.progress + minutes;
                    }
                    return c;
                });

                updates.challenges = newChallenges;

                // Finalmente, actualizamos Firestore
                await updateFirestore(updates);
                
                // Alerta de éxito (usando una simulación de modal simple)
                console.log(`¡Entrenamiento completado! Ganaste ${xpGained + extraXp} XP.`);

            }, [user, updateFirestore]);
            
            // Lógica para reclamar la recompensa de un Reto
            const handleClaimChallenge = useCallback(async (challengeId, xpReward) => {
                if (!user) return;
                
                const challengeIndex = user.challenges.findIndex(c => c.id === challengeId);
                if (challengeIndex === -1 || user.challenges[challengeIndex].isCompleted) return;

                let newChallenges = [...user.challenges];
                // Marcar como completado y reiniciar progreso para hacerlo re-jugable
                newChallenges[challengeIndex] = { ...newChallenges[challengeIndex], isCompleted: true, progress: newChallenges[challengeIndex].type === 'daily' ? 0 : newChallenges[challengeIndex].progress };

                let updates = {
                    xp: user.xp + xpReward,
                    challenges: newChallenges,
                };
                
                // Revisar si se alcanzó la meta semanal (se activa el logro si se reclama el reto semanal)
                if (challengeId === 'weekly_30' && !user.achievements.weeklyWarrior) {
                    updates.achievements = { ...user.achievements, weeklyWarrior: true };
                    updates.xp += ACHIEVEMENTS.find(a => a.id === 'weeklyWarrior').xp;
                }

                await updateFirestore(updates);
                console.log(`¡Reclamaste el reto ${challengeId} y ganaste ${xpReward} XP!`);

            }, [user, updateFirestore]);

            const handleLogout = useCallback(async () => {
                if (auth) {
                    await auth.signOut();
                    // Para la próxima sesión, forzamos un nuevo inicio anónimo/token
                    setCurrentScreen('Onboarding'); 
                }
            }, []);

            // Calcula nivel y progreso para pasar a los componentes de recompensa
            const { level, progress } = user ? calculateLevelProgress(user.xp) : { level: 1, progress: 0 };


            // --- RENDERIZADO PRINCIPAL ---

            if (error) {
                return <div className="min-h-screen flex items-center justify-center bg-red-50 p-4 text-red-800 font-semibold">{error}</div>;
            }

            if (currentScreen === 'Loading' || !isAuthReady) {
                return (
                    <div className="min-h-screen flex flex-col items-center justify-center bg-gray-50">
                        <svg className="animate-spin h-8 w-8 text-indigo-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                            <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                            <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                        </svg>
                        <p className="mt-3 text-indigo-600">Cargando datos...</p>
                    </div>
                );
            }

            if (currentScreen === 'Onboarding' || !user?.isRegistered) {
                return <OnboardingScreen onRegister={handleRegistration} />;
            }

            // Contenido de las pestañas
            const renderScreen = () => {
                switch (currentScreen) {
                    case 'Home':
                        return <HomeScreen user={user} handleWorkoutComplete={handleWorkoutComplete} />;
                    case 'Challenges':
                        return <ChallengesScreen user={user} handleClaimChallenge={handleClaimChallenge} />;
                    case 'Rewards':
                        return <RewardsScreen user={user} level={level} progress={progress} />;
                    case 'Profile':
                        return <ProfileScreen user={user} handleLogout={handleLogout} />;
                    default:
                        return null;
                }
            };

            return (
                <div className="min-h-screen bg-gray-50 pb-20">
                    <div className="max-w-md mx-auto min-h-screen">
                        <Header title={currentScreen} user={user} />
                        <main className="pb-4">
                            {renderScreen()}
                        </main>
                        <TabNavigation currentScreen={currentScreen} setCurrentScreen={setCurrentScreen} />
                    </div>
                </div>
            );
        };

        // Renderizado inicial
        document.addEventListener('DOMContentLoaded', () => {
            const container = document.getElementById('root');
            if (container) {
                const root = createRoot(container);
                root.render(<App />);
            }
        });
    </script>
</body>
</html>
