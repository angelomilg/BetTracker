// Configuración inicial de Firebase
const firebaseConfig = {
  apiKey: "AIzaSyAcVHefZmWRvlaGi0o98CiINW7ZKDm-lzM",
  authDomain: "omil-4fde2.firebaseapp.com",
  databaseURL: "https://omil-4fde2-default-rtdb.europe-west1.firebasedatabase.app",
  projectId: "omil-4fde2",
  storageBucket: "omil-4fde2.firebasestorage.app",
  messagingSenderId: "568300195857",
  appId: "1:568300195857:web:8dc37413e3703e686c61b0"
};

// Intentar cargar la configuración desde localStorage
const savedConfig = localStorage.getItem('firebaseConfig');
if (savedConfig) {
    try {
        firebaseConfig = JSON.parse(savedConfig);
        document.getElementById('firebase-config').value = savedConfig;
    } catch (e) {
        console.error('Error al cargar la configuración de Firebase:', e);
    }
}

// Inicializar Firebase
firebase.initializeApp(firebaseConfig);
const auth = firebase.auth();
const database = firebase.database();

// Variables globales
let currentUser = null;
let userBets = [];
let userStrategies = [];
let userSettings = {
    initialBank: 100,
    stakeUnit: 10,
    defaultStake: 1
};
let bankHistory = [];
let bankChart = null;
let strategyChart = null;
let strategyDetailChart = null;

// Elementos del DOM
const authScreen = document.getElementById('auth-screen');
const appContainer = document.getElementById('app-container');
const loginEmail = document.getElementById('login-email');
const loginPassword = document.getElementById('login-password');
const loginBtn = document.getElementById('login-btn');
const registerBtn = document.getElementById('register-btn');
const logoutBtn = document.getElementById('logout-btn');
const userEmailSpan = document.getElementById('user-email');
const firebaseConfigTextarea = document.getElementById('firebase-config');
const saveConfigBtn = document.getElementById('save-config-btn');
const themeSwitch = document.getElementById('theme-switch');

// Secciones de la aplicación
const sections = {
    dashboard: document.getElementById('dashboard'),
    bets: document.getElementById('bets'),
    'add-bet': document.getElementById('add-bet'),
    strategies: document.getElementById('strategies'),
    settings: document.getElementById('settings')
};

// Nav buttons
const navButtons = document.querySelectorAll('.nav-btn');

// Event Listeners
document.addEventListener('DOMContentLoaded', initApp);
loginBtn.addEventListener('click', loginUser);
registerBtn.addEventListener('click', registerUser);
logoutBtn.addEventListener('click', logoutUser);
saveConfigBtn.addEventListener('click', saveFirebaseConfig);
themeSwitch.addEventListener('change', toggleTheme);

// Inicializar la aplicación
function initApp() {
    // Verificar el tema guardado
    const savedTheme = localStorage.getItem('theme') || 'dark';
    document.body.className = `${savedTheme}-theme`;
    themeSwitch.checked = savedTheme === 'light';
    
    // Configurar listeners para navegación
    navButtons.forEach(button => {
        button.addEventListener('click', () => {
            const section = button.getAttribute('data-section');
            showSection(section);
        });
    });
    
    // Configurar Firebase Auth state observer
    auth.onAuthStateChanged(user => {
        if (user) {
            // Usuario ha iniciado sesión
            currentUser = user;
            userEmailSpan.textContent = user.email;
            authScreen.classList.add('hidden');
            appContainer.classList.remove('hidden');
            loadUserData();
            showSection('dashboard');
        } else {
            // Usuario no ha iniciado sesión
            currentUser = null;
            authScreen.classList.remove('hidden');
            appContainer.classList.add('hidden');
        }
    });
    
    // Configurar formulario de apuestas
    setupBetForm();
    
    // Configurar filtros
    setupFilters();
    
    // Configurar estrategias
    setupStrategies();
    
    // Configurar ajustes
    setupSettings();
    
    // Configurar modales
    setupModals();
}

// Mostrar sección específica
function showSection(sectionId) {
    // Ocultar todas las secciones
    Object.values(sections).forEach(section => {
        section.classList.add('hidden');
    });
    
    // Mostrar la sección seleccionada
    sections[sectionId].classList.remove('hidden');
    
    // Actualizar botones de navegación
    navButtons.forEach(button => {
        if (button.getAttribute('data-section') === sectionId) {
            button.classList.add('active');
        } else {
            button.classList.remove('active');
        }
    });
    
    // Actualizar gráficos si es necesario
    if (sectionId === 'dashboard' && bankChart) {
        bankChart.update();
        strategyChart.update();
    }
    
    if (sectionId === 'strategies' && strategyDetailChart) {
        strategyDetailChart.update();
    }
}

// Iniciar sesión
function loginUser() {
    const email = loginEmail.value.trim();
    const password = loginPassword.value.trim();
    
    if (!email || !password) {
        alert('Por favor ingresa email y contraseña');
        return;
    }
    
    auth.signInWithEmailAndPassword(email, password)
        .catch(error => {
            alert(`Error al iniciar sesión: ${error.message}`);
        });
}

// Registrar nuevo usuario
function registerUser() {
    const email = loginEmail.value.trim();
    const password = loginPassword.value.trim();
    
    if (!email || !password) {
        alert('Por favor ingresa email y contraseña');
        return;
    }
    
    if (password.length < 6) {
        alert('La contraseña debe tener al menos 6 caracteres');
        return;
    }
    
    auth.createUserWithEmailAndPassword(email, password)
        .then(() => {
            // Crear estructura inicial de datos para el nuevo usuario
            const user = auth.currentUser;
            if (user) {
                const userRef = database.ref(`users/${user.uid}`);
                
                // Configuración inicial
                userRef.child('settings').set({
                    initialBank: 100,
                    stakeUnit: 10,
                    defaultStake: 1
                });
                
                // Estrategias iniciales
                userRef.child('strategies').set({
                    default: {
                        name: 'General',
                        createdAt: firebase.database.ServerValue.TIMESTAMP
                    }
                });
            }
        })
        .catch(error => {
            alert(`Error al registrar usuario: ${error.message}`);
        });
}

// Cerrar sesión
function logoutUser() {
    auth.signOut()
        .then(() => {
            // Limpiar datos del usuario actual
            currentUser = null;
            userBets = [];
            userStrategies = [];
            userSettings = {
                initialBank: 100,
                stakeUnit: 10,
                defaultStake: 1
            };
            bankHistory = [];
            
            // Resetear formularios
            document.getElementById('add-bet-form').reset();
            
            // Limpiar tablas
            document.querySelector('#recent-bets-table tbody').innerHTML = '';
            document.querySelector('#bets-table tbody').innerHTML = '';
            document.querySelector('#strategies-ul').innerHTML = '';
            
            // Destruir gráficos
            if (bankChart) {
                bankChart.destroy();
                bankChart = null;
            }
            
            if (strategyChart) {
                strategyChart.destroy();
                strategyChart = null;
            }
            
            if (strategyDetailChart) {
                strategyDetailChart.destroy();
                strategyDetailChart = null;
            }
        })
        .catch(error => {
            alert(`Error al cerrar sesión: ${error.message}`);
        });
}

// Guardar configuración de Firebase
function saveFirebaseConfig() {
    const configText = firebaseConfigTextarea.value.trim();
    
    if (!configText) {
        alert('Por favor ingresa la configuración de Firebase');
        return;
    }
    
    try {
        const config = JSON.parse(configText);
        
        // Validar campos mínimos requeridos
        if (!config.apiKey || !config.authDomain || !config.databaseURL || !config.projectId) {
            throw new Error('La configuración de Firebase no es válida. Faltan campos requeridos.');
        }
        
        // Guardar en localStorage
        localStorage.setItem('firebaseConfig', configText);
        
        // Reiniciar la aplicación con la nueva configuración
        alert('Configuración guardada. La página se recargará para aplicar los cambios.');
        location.reload();
    } catch (e) {
        alert(`Error al guardar la configuración: ${e.message}`);
    }
}

// Alternar entre tema oscuro y claro
function toggleTheme() {
    if (themeSwitch.checked) {
        document.body.classList.remove('dark-theme');
        document.body.classList.add('light-theme');
        localStorage.setItem('theme', 'light');
    } else {
        document.body.classList.remove('light-theme');
        document.body.classList.add('dark-theme');
        localStorage.setItem('theme', 'dark');
    }
    
    // Actualizar gráficos si existen
    if (bankChart) bankChart.update();
    if (strategyChart) strategyChart.update();
    if (strategyDetailChart) strategyDetailChart.update();
}

// Cargar datos del usuario
// Cargar datos del usuario
function loadUserData() {
    createEmptyCharts();
    
    if (!currentUser) return;
    const userRef = database.ref(`users/${currentUser.uid}`);
    
    // 1. Primero cargamos la configuración
    userRef.child('settings').once('value').then(snapshot => {
        if (snapshot.exists()) {
            userSettings = snapshot.val();
            
            // Asegurar valores por defecto si algún campo está vacío
            userSettings.initialBank = userSettings.initialBank || 100;
            userSettings.stakeUnit = userSettings.stakeUnit || 10;
            userSettings.defaultStake = userSettings.defaultStake || 1;
            
            // Actualizar UI
            document.getElementById('initial-bank').value = userSettings.initialBank;
            document.getElementById('stake-unit').value = userSettings.stakeUnit;
            document.getElementById('default-stake').value = userSettings.defaultStake;
            document.getElementById('bet-stake').value = userSettings.defaultStake;
            
            // Ahora cargamos el resto de datos
            loadBetsAndStrategies(userRef);
        } else {
            // Si no existe configuración, creamos una por defecto
            userSettings = {
                initialBank: 100,
                stakeUnit: 10,
                defaultStake: 1
            };
            userRef.child('settings').set(userSettings)
                .then(() => loadBetsAndStrategies(userRef));
        }
    }).catch(error => {
        console.error("Error loading settings:", error);
    });
}

// Función separada para cargar apuestas y estrategias
function loadBetsAndStrategies(userRef) {
    // Cargar apuestas
    userRef.child('bets').orderByChild('date').on('value', snapshot => {
        userBets = [];
        snapshot.forEach(childSnapshot => {
            const bet = childSnapshot.val();
            bet.id = childSnapshot.key;
            userBets.unshift(bet);
        });
        
        calculateBankHistory(); // ¡Ahora con userSettings correctos!
        updateDashboard();
        updateBetsTable();
        updateRecentBets();
    });
    
    // Cargar estrategias
    userRef.child('strategies').on('value', snapshot => {
        userStrategies = [];
        snapshot.forEach(childSnapshot => {
            const strategy = childSnapshot.val();
            strategy.id = childSnapshot.key;
            userStrategies.push(strategy);
        });
        
        updateStrategySelects();
        updateStrategiesList();
    });

    // Cargar configuración
    userRef.child('settings').once('value').then(snapshot => {
        if (snapshot.exists()) {
            userSettings = snapshot.val();
            
            // Actualizar campos de configuración
            document.getElementById('initial-bank').value = userSettings.initialBank || 100;
            document.getElementById('stake-unit').value = userSettings.stakeUnit || 10;
            document.getElementById('default-stake').value = userSettings.defaultStake || 1;
            
            // Actualizar stake predeterminado en el formulario
            document.getElementById('bet-stake').value = userSettings.defaultStake || 1;
        }
        
        // Calcular historial de bank
        calculateBankHistory();
    });
}

// Configurar formulario de apuestas
function setupBetForm() {
    const form = document.getElementById('add-bet-form');
    
    form.addEventListener('submit', e => {
        e.preventDefault();
        
        if (!currentUser) {
            alert('Debes iniciar sesión para añadir apuestas');
            return;
        }
        
        const bet = {
            date: document.getElementById('bet-date').value,
            sport: document.getElementById('bet-sport').value.trim(),
            event: document.getElementById('bet-event').value.trim(),
            type: document.getElementById('bet-type').value,
            odds: parseFloat(document.getElementById('bet-odds').value),
            stake: parseFloat(document.getElementById('bet-stake').value),
            result: document.getElementById('bet-result').value,
            strategy: document.getElementById('bet-strategy').value,
            bookmaker: document.getElementById('bet-bookmaker').value.trim() || 'No especificado',
            comments: document.getElementById('bet-comments').value.trim() || '',
            createdAt: firebase.database.ServerValue.TIMESTAMP
        };
        
        // Validar campos requeridos
        if (!bet.date || !bet.sport || !bet.event || !bet.type || !bet.odds || !bet.stake || !bet.result || !bet.strategy) {
            alert('Por favor completa todos los campos requeridos');
            return;
        }
        
        // Validar valores numéricos
        if (isNaN(bet.odds)) {
            alert('La cuota debe ser un número válido');
            return;
        }
        
        if (isNaN(bet.stake)) {
            alert('El stake debe ser un número válido');
            return;
        }
        
        // Guardar la apuesta en Firebase
        database.ref(`users/${currentUser.uid}/bets`).push(bet)
            .then(() => {
                alert('Apuesta guardada correctamente');
                form.reset();
                // Restablecer stake al valor predeterminado
                document.getElementById('bet-stake').value = userSettings.defaultStake || 1;
            })
            .catch(error => {
                alert(`Error al guardar la apuesta: ${error.message}`);
            });
    });
}

// Actualizar selects de estrategias
function updateStrategySelects() {
    const selects = [
        document.getElementById('bet-strategy'),
        document.getElementById('edit-bet-strategy'),
        document.getElementById('strategy-filter'),
        document.getElementById('strategy-stats-select')
    ];
    
    selects.forEach(select => {
        // Guardar valor seleccionado actual
        const selectedValue = select.value;
        
        // Limpiar opciones (excepto la primera si es un select de filtro)
        select.innerHTML = '';
        
        if (select.id === 'strategy-filter' || select.id === 'strategy-stats-select') {
            const defaultOption = document.createElement('option');
            defaultOption.value = select.id === 'strategy-filter' ? 'all' : '';
            defaultOption.textContent = select.id === 'strategy-filter' ? 'Todas las estrategias' : 'Selecciona una estrategia';
            select.appendChild(defaultOption);
        }
        
        // Añadir estrategias
        userStrategies.forEach(strategy => {
            const option = document.createElement('option');
            option.value = strategy.id;
            option.textContent = strategy.name;
            select.appendChild(option);
        });
        
        // Restaurar valor seleccionado si existe en las nuevas opciones
        if (selectedValue && Array.from(select.options).some(opt => opt.value === selectedValue)) {
            select.value = selectedValue;
        }
    });
}

// Actualizar lista de estrategias en la UI
function updateStrategiesList() {
    const strategiesList = document.getElementById('strategies-ul');
    strategiesList.innerHTML = '';
    
    userStrategies.forEach(strategy => {
        const li = document.createElement('li');
        
        const nameSpan = document.createElement('span');
        nameSpan.textContent = strategy.name;
        
        const deleteBtn = document.createElement('button');
        deleteBtn.textContent = 'Eliminar';
        deleteBtn.className = 'btn btn-small btn-danger';
        deleteBtn.addEventListener('click', () => confirmDeleteStrategy(strategy.id, strategy.name));
        
        // No permitir eliminar la estrategia "default"
        if (strategy.id === 'default') {
            deleteBtn.disabled = true;
            deleteBtn.title = 'Esta estrategia no se puede eliminar';
        }
        
        li.appendChild(nameSpan);
        li.appendChild(deleteBtn);
        strategiesList.appendChild(li);
    });
}

// Configurar gestión de estrategias
function setupStrategies() {
    // Añadir estrategia desde el modal
    document.getElementById('add-strategy-main-btn').addEventListener('click', () => {
        const strategyName = document.getElementById('new-strategy-name').value.trim();
        
        if (!strategyName) {
            alert('Por favor ingresa un nombre para la estrategia');
            return;
        }
        
        addNewStrategy(strategyName);
        document.getElementById('new-strategy-name').value = '';
    });
    
    // Añadir estrategia desde el formulario de apuestas
    document.getElementById('add-strategy-btn').addEventListener('click', () => {
        document.getElementById('new-strategy-modal').classList.remove('hidden');
    });
    
    // Guardar estrategia desde el modal
    document.getElementById('save-strategy-btn').addEventListener('click', () => {
        const strategyName = document.getElementById('modal-strategy-name').value.trim();
        
        if (!strategyName) {
            alert('Por favor ingresa un nombre para la estrategia');
            return;
        }
        
        addNewStrategy(strategyName);
        document.getElementById('modal-strategy-name').value = '';
        document.getElementById('new-strategy-modal').classList.add('hidden');
    });
    
    // Cancelar modal de estrategia
    document.getElementById('cancel-strategy-btn').addEventListener('click', () => {
        document.getElementById('modal-strategy-name').value = '';
        document.getElementById('new-strategy-modal').classList.add('hidden');
    });
    
    // Seleccionar estrategia para ver estadísticas
    document.getElementById('strategy-stats-select').addEventListener('change', function() {
        const strategyId = this.value;
        const strategyStatsContainer = document.getElementById('strategy-stats-container');
        
        if (strategyId) {
            updateStrategyStats(strategyId);
            strategyStatsContainer.classList.remove('hidden');
        } else {
            strategyStatsContainer.classList.add('hidden');
        }
    });
}

// Añadir nueva estrategia
function addNewStrategy(name) {
    if (!currentUser) return;
    
    const newStrategy = {
        name: name,
        createdAt: firebase.database.ServerValue.TIMESTAMP
    };
    
    database.ref(`users/${currentUser.uid}/strategies`).push(newStrategy)
        .then(() => {
            // Actualizar las estadísticas después de añadir
            saveStrategyStats();
            
            // Opcional: Actualizar selects y lista
            updateStrategySelects();
            updateStrategiesList();
            
            // Mantener tu feedback al usuario
            alert('Estrategia añadida correctamente');
        })
        .catch(error => {
            alert(`Error al añadir estrategia: ${error.message}`);
        });
}

// Confirmar eliminación de estrategia
function confirmDeleteStrategy(strategyId, strategyName) {
    if (strategyId === 'default') {
        alert('No se puede eliminar la estrategia por defecto');
        return;
    }
    
    showConfirmModal(
        `Eliminar estrategia "${strategyName}"`,
        `¿Estás seguro de que quieres eliminar la estrategia "${strategyName}"? Las apuestas asociadas no se eliminarán, pero perderán la referencia a esta estrategia.`,
        () => deleteStrategy(strategyId)
    );
}

// Eliminar estrategia
function deleteStrategy(strategyId) {
    if (!currentUser || !strategyId) return;
    
    database.ref(`users/${currentUser.uid}/strategies/${strategyId}`).remove()
        .then(() => {
            // 1. Actualizar apuestas que usaban esta estrategia
            const betsRef = database.ref(`users/${currentUser.uid}/bets`);
            
            betsRef.orderByChild('strategy').equalTo(strategyId).once('value', snapshot => {
                const updates = {};
                
                snapshot.forEach(childSnapshot => {
                    updates[`${childSnapshot.key}/strategy`] = 'default';
                });
                
                // 2. Si había apuestas para actualizar
                if (Object.keys(updates).length > 0) {
                    betsRef.update(updates)
                        .then(() => {
                            // 3. Actualizar estadísticas después de cambiar las apuestas
                            saveStrategyStats();
                            updateDashboard(); // Para refrescar toda la UI
                        });
                } else {
                    // 4. Si no había apuestas, igual actualizar stats
                    saveStrategyStats();
                }
            });
            
            // 5. Actualizar la lista de estrategias inmediatamente
            updateStrategiesList();
            updateStrategySelects();
            
            // 6. Feedback al usuario
            alert('Estrategia eliminada correctamente');
        })
        .catch(error => {
            alert(`Error al eliminar estrategia: ${error.message}`);
        });
}

// Actualizar estadísticas de estrategia
function updateStrategyStats(strategyId) {
    const strategy = userStrategies.find(s => s.id === strategyId);
    if (!strategy) return;
    
    // Filtrar apuestas por estrategia
    const strategyBets = userBets.filter(bet => bet.strategy === strategyId);
    const totalBets = strategyBets.length;
    
    if (totalBets === 0) {
        // Mostrar estadísticas vacías
        document.getElementById('strategy-total-bets').textContent = '0';
        document.getElementById('strategy-won-bets').textContent = '0';
        document.getElementById('strategy-lost-bets').textContent = '0';
        document.getElementById('strategy-void-bets').textContent = '0';
        document.getElementById('strategy-hit-rate').textContent = '0%';
        document.getElementById('strategy-profit').textContent = '0.00';
        document.getElementById('strategy-roi').textContent = '0%';
        document.getElementById('strategy-yield').textContent = '0%';
        
        // Actualizar gráfico vacío
        updateStrategyDetailChart(strategyBets, strategy.name);
        return;
    }
    
    // Calcular estadísticas
    const wonBets = strategyBets.filter(bet => bet.result === 'won').length;
    const lostBets = strategyBets.filter(bet => bet.result === 'lost').length;
    const voidBets = strategyBets.filter(bet => bet.result === 'void').length;
    
    const hitRate = totalBets > 0 ? (wonBets / (totalBets - voidBets)) * 100 : 0;
    
    const totalStaked = strategyBets.reduce((sum, bet) => sum + (bet.stake * userSettings.stakeUnit), 0);
    const totalProfit = strategyBets.reduce((sum, bet) => {
        if (bet.result === 'won') {
            return sum + ((bet.odds - 1) * bet.stake * userSettings.stakeUnit);
        } else if (bet.result === 'lost') {
            return sum - (bet.stake * userSettings.stakeUnit);
        }
        return sum; // void no afecta
    }, 0);
    
    const roi = totalStaked > 0 ? (totalProfit / totalStaked) * 100 : 0;
    const yield = totalStaked > 0 ? (totalProfit / totalStaked) * 100 : 0;
    
    // Actualizar UI
    document.getElementById('strategy-total-bets').textContent = totalBets;
    document.getElementById('strategy-won-bets').textContent = wonBets;
    document.getElementById('strategy-lost-bets').textContent = lostBets;
    document.getElementById('strategy-void-bets').textContent = voidBets;
    document.getElementById('strategy-hit-rate').textContent = hitRate.toFixed(2) + '%';
    document.getElementById('strategy-profit').textContent = totalProfit.toFixed(2);
    document.getElementById('strategy-roi').textContent = roi.toFixed(2) + '%';
    document.getElementById('strategy-yield').textContent = yield.toFixed(2) + '%';
    
    // Actualizar gráfico
    updateStrategyDetailChart(strategyBets, strategy.name);
}

// Configurar filtros
function setupFilters() {
    const timeFilter = document.getElementById('time-filter');
    const customDateRange = document.getElementById('custom-date-range');
    const applyRangeBtn = document.getElementById('apply-range-btn');
    const resetFiltersBtn = document.getElementById('reset-filters-btn');
    
    // Mostrar/ocultar rango de fechas personalizado
    timeFilter.addEventListener('change', function() {
        if (this.value === 'custom') {
            customDateRange.classList.remove('hidden');
        } else {
            customDateRange.classList.add('hidden');
            updateBetsTable();
        }
    });
    
    // Aplicar rango personalizado
    applyRangeBtn.addEventListener('click', () => {
        const startDate = document.getElementById('start-date').value;
        const endDate = document.getElementById('end-date').value;
        
        if (!startDate || !endDate) {
            alert('Por favor selecciona ambas fechas');
            return;
        }
        
        if (new Date(startDate) > new Date(endDate)) {
            alert('La fecha de inicio no puede ser mayor que la fecha final');
            return;
        }
        
        updateBetsTable();
    });
    
    // Aplicar filtros cuando cambian
    document.getElementById('strategy-filter').addEventListener('change', updateBetsTable);
    document.getElementById('sport-filter').addEventListener('change', updateBetsTable);
    document.getElementById('result-filter').addEventListener('change', updateBetsTable);
    
    // Resetear filtros
    resetFiltersBtn.addEventListener('click', () => {
        timeFilter.value = 'all';
        customDateRange.classList.add('hidden');
        document.getElementById('strategy-filter').value = 'all';
        document.getElementById('sport-filter').value = 'all';
        document.getElementById('result-filter').value = 'all';
        updateBetsTable();
    });
}

// Actualizar tabla de apuestas con filtros aplicados
function updateBetsTable() {
    if (!currentUser) return;
    
    const tbody = document.querySelector('#bets-table tbody');
    tbody.innerHTML = '';
    
    // Obtener valores de filtros
    const timeFilter = document.getElementById('time-filter').value;
    const strategyFilter = document.getElementById('strategy-filter').value;
    const sportFilter = document.getElementById('sport-filter').value;
    const resultFilter = document.getElementById('result-filter').value;
    
    // Obtener fechas para filtro personalizado
    let startDate, endDate;
    if (timeFilter === 'custom') {
        startDate = document.getElementById('start-date').value;
        endDate = document.getElementById('end-date').value;
        
        if (!startDate || !endDate) {
            alert('Por favor selecciona un rango de fechas válido');
            return;
        }
    }
    
    // Filtrar apuestas
    const filteredBets = userBets.filter(bet => {
        // Filtro por tiempo
        if (timeFilter !== 'all') {
            const betDate = new Date(bet.date);
            const now = new Date();
            
            switch (timeFilter) {
                case 'today':
                    if (!isSameDay(betDate, now)) return false;
                    break;
                case 'week':
                    const oneWeekAgo = new Date();
                    oneWeekAgo.setDate(now.getDate() - 7);
                    if (betDate < oneWeekAgo) return false;
                    break;
                case 'month':
                    if (betDate.getMonth() !== now.getMonth() || betDate.getFullYear() !== now.getFullYear()) {
                        return false;
                    }
                    break;
                case 'custom':
                    if (new Date(bet.date) < new Date(startDate) || new Date(bet.date) > new Date(endDate)) {
                        return false;
                    }
                    break;
            }
        }
        
        // Filtro por estrategia
        if (strategyFilter !== 'all' && bet.strategy !== strategyFilter) {
            return false;
        }
        
        // Filtro por deporte
        if (sportFilter !== 'all') {
            // Actualizar select de deportes si está vacío
            if (document.getElementById('sport-filter').options.length === 1) {
                updateSportFilterOptions();
            }
            
            if (!bet.sport.toLowerCase().includes(sportFilter.toLowerCase())) {
                return false;
            }
        }
        
        // Filtro por resultado
        if (resultFilter !== 'all' && bet.result !== resultFilter) {
            return false;
        }
        
        return true;
    });
    
    // Mostrar apuestas filtradas
    filteredBets.forEach(bet => {
        const strategy = userStrategies.find(s => s.id === bet.strategy) || { name: 'Desconocida' };
        
        const tr = document.createElement('tr');
        
        // Calcular beneficio
        let profit = 0;
        if (bet.result === 'won') {
            profit = (bet.odds - 1) * bet.stake * userSettings.stakeUnit;
        } else if (bet.result === 'lost') {
            profit = -bet.stake * userSettings.stakeUnit;
        }
        
        tr.innerHTML = `
            <td>${formatDate(bet.date)}</td>
            <td>${bet.sport}</td>
            <td>${bet.event}</td>
            <td>${bet.type}</td>
            <td>${strategy.name}</td>
            <td>${bet.odds.toFixed(2)}</td>
            <td>${bet.stake.toFixed(1)}</td>
            <td class="${bet.result}">${getResultText(bet.result)}</td>
            <td class="${profit >= 0 ? 'positive' : 'negative'}">${profit.toFixed(2)}</td>
            <td>
                <button class="btn btn-small edit-bet" data-id="${bet.id}">Editar</button>
                <button class="btn btn-small btn-danger delete-bet" data-id="${bet.id}">Eliminar</button>
            </td>
        `;
        
        tbody.appendChild(tr);
    });
    
    // Configurar eventos para botones de editar/eliminar
    document.querySelectorAll('.edit-bet').forEach(btn => {
        btn.addEventListener('click', () => editBet(btn.getAttribute('data-id')));
    });
    
    document.querySelectorAll('.delete-bet').forEach(btn => {
        btn.addEventListener('click', () => confirmDeleteBet(btn.getAttribute('data-id')));
    });
}

// Actualizar opciones del filtro de deportes
function updateSportFilterOptions() {
    const sportFilter = document.getElementById('sport-filter');
    
    // Obtener deportes únicos
    const sports = [...new Set(userBets.map(bet => bet.sport))].sort();
    
    // Limpiar opciones (excepto "Todos")
    while (sportFilter.options.length > 1) {
        sportFilter.remove(1);
    }
    
    // Añadir deportes
    sports.forEach(sport => {
        const option = document.createElement('option');
        option.value = sport;
        option.textContent = sport;
        sportFilter.appendChild(option);
    });
}

// Actualizar apuestas recientes en el dashboard
function updateRecentBets() {
    const tbody = document.querySelector('#recent-bets-table tbody');
    tbody.innerHTML = '';
    
    // Mostrar las últimas 5 apuestas
    const recentBets = userBets.slice(0, 5);
    
    recentBets.forEach(bet => {
        // Calcular beneficio
        let profit = 0;
        if (bet.result === 'won') {
            profit = (bet.odds - 1) * bet.stake * userSettings.stakeUnit;
        } else if (bet.result === 'lost') {
            profit = -bet.stake * userSettings.stakeUnit;
        }
        
        const tr = document.createElement('tr');
        tr.innerHTML = `
            <td>${formatDate(bet.date)}</td>
            <td>${bet.event}</td>
            <td>${bet.type}</td>
            <td>${bet.odds.toFixed(2)}</td>
            <td>${bet.stake.toFixed(1)}</td>
            <td class="${bet.result}">${getResultText(bet.result)}</td>
            <td class="${profit >= 0 ? 'positive' : 'negative'}">${profit.toFixed(2)}</td>
        `;
        
        tbody.appendChild(tr);
    });
}

// Actualizar dashboard con estadísticas
function updateDashboard() {
    if (!currentUser || userBets.length === 0) {
        // Mostrar valores por defecto
        document.getElementById('current-bank').textContent = userSettings.initialBank.toFixed(2);
        document.getElementById('net-profit').textContent = '0.00';
        document.getElementById('roi').textContent = '0.00%';
        document.getElementById('yield').textContent = '0.00%';
        document.getElementById('hit-rate').textContent = '0.00%';
        document.getElementById('total-bets').textContent = '0';
        return;
    }
    
    // Calcular estadísticas generales
    const totalBets = userBets.length;
    const wonBets = userBets.filter(bet => bet.result === 'won').length;
    const lostBets = userBets.filter(bet => bet.result === 'lost').length;
    const voidBets = userBets.filter(bet => bet.result === 'void').length;
    
    const hitRate = totalBets > 0 ? (wonBets / (totalBets - voidBets)) * 100 : 0;
    
    const totalStaked = userBets.reduce((sum, bet) => sum + (bet.stake * userSettings.stakeUnit), 0);
    const totalProfit = userBets.reduce((sum, bet) => {
        if (bet.result === 'won') {
            return sum + ((bet.odds - 1) * bet.stake * userSettings.stakeUnit);
        } else if (bet.result === 'lost') {
            return sum - (bet.stake * userSettings.stakeUnit);
        }
        return sum; // void no afecta
    }, 0);
    
    const currentBank = userSettings.initialBank + totalProfit;
    const roi = userSettings.initialBank > 0 ? (totalProfit / userSettings.initialBank) * 100 : 0;
    const yield = totalStaked > 0 ? (totalProfit / totalStaked) * 100 : 0;
    
    // Actualizar UI
    document.getElementById('current-bank').textContent = currentBank.toFixed(2);
    document.getElementById('net-profit').textContent = totalProfit.toFixed(2);
    document.getElementById('roi').textContent = roi.toFixed(2) + '%';
    document.getElementById('yield').textContent = yield.toFixed(2) + '%';
    document.getElementById('hit-rate').textContent = hitRate.toFixed(2) + '%';
    document.getElementById('total-bets').textContent = totalBets;
    
    // Actualizar gráficos
    calculateBankHistory();
    updateBankChart();
    updateStrategyPerformanceChart();
    saveStrategyStats();
}
// Añade esta nueva función después de updateDashboard():
function createEmptyCharts() {
    const chartOptions = {
        responsive: true,
        maintainAspectRatio: true
    };
    
    // Gráfico de bank vacío
    const bankCtx = document.getElementById('bank-chart').getContext('2d');
    if (bankChart) bankChart.destroy();
    bankChart = new Chart(bankCtx, {
        type: 'line',
        data: { datasets: [{
            label: 'Bank',
            data: [{x: new Date(), y: userSettings.initialBank}],
            borderColor: '#4cc9f0',
            backgroundColor: 'rgba(76, 201, 240, 0.1)'
        }]},
        options: chartOptions
    });
    
    // Gráfico de estrategias vacío
    const strategyCtx = document.getElementById('strategy-chart').getContext('2d');
    if (strategyChart) strategyChart.destroy();
    strategyChart = new Chart(strategyCtx, {
        type: 'bar',
        data: { labels: ['No hay datos'], datasets: [{
            label: 'ROI (%)',
            data: [0],
            backgroundColor: 'rgba(201, 203, 207, 0.2)'
        }]},
        options: chartOptions
    });
}

// Calcular historial del bank para el gráfico
function calculateBankHistory() {
    if (!currentUser || userBets.length === 0) {
        bankHistory = [{ date: new Date().toISOString(), bank: userSettings.initialBank }];
        return;
    }
    
    // Ordenar apuestas por fecha (más antigua primero)
    const sortedBets = [...userBets].sort((a, b) => new Date(a.date) - new Date(b.date));
    
    bankHistory = [];
    let runningBank = userSettings.initialBank;
    
    // Añadir punto inicial
    bankHistory.push({
        date: new Date(sortedBets[0].date).toISOString(),
        bank: runningBank
    });
    
    // Calcular bank después de cada apuesta
    sortedBets.forEach(bet => {
        if (bet.result === 'won') {
            runningBank += (bet.odds - 1) * bet.stake * userSettings.stakeUnit;
        } else if (bet.result === 'lost') {
            runningBank -= bet.stake * userSettings.stakeUnit;
        }
        // void no afecta el bank
        
        bankHistory.push({
            date: new Date(bet.date).toISOString(),
            bank: runningBank
        });
    });
    
    // Añadir punto actual si no hay apuestas hoy
    const lastBetDate = new Date(sortedBets[sortedBets.length - 1].date);
    if (!isSameDay(lastBetDate, new Date())) {
        bankHistory.push({
            date: new Date().toISOString(),
            bank: runningBank
        });
    }
}

// Actualizar gráfico de evolución del bank
function updateBankChart() {
    const ctx = document.getElementById('bank-chart').getContext('2d');
    
    // Destruir gráfico anterior si existe
    if (bankChart) {
        bankChart.destroy();
    }
    
    const labels = bankHistory.map(item => formatDate(item.date, true));
    const data = bankHistory.map(item => item.bank);
    
    bankChart = new Chart(ctx, {
        type: 'line',
        data: {
            labels: labels,
            datasets: [{
                label: 'Bank',
                data: data,
                borderColor: '#4cc9f0',
                backgroundColor: 'rgba(76, 201, 240, 0.1)',
                tension: 0.1,
                fill: true
            }]
        },
        options: {
            responsive: true,
            plugins: {
                legend: {
                    display: false
                },
                tooltip: {
                    callbacks: {
                        label: function(context) {
                            return 'Bank: ' + context.parsed.y.toFixed(2);
                        }
                    }
                }
            },
            scales: {
                y: {
                    beginAtZero: false
                }
            }
        }
    });
}

// Actualizar gráfico de rendimiento por estrategia
function updateStrategyPerformanceChart() {
    const ctx = document.getElementById('strategy-chart').getContext('2d');
    
    // Destruir gráfico anterior si existe
    if (strategyChart) {
        strategyChart.destroy();
    }
    
    // Calcular ROI por estrategia
    const strategiesWithStats = userStrategies.map(strategy => {
        const bets = userBets.filter(bet => bet.strategy === strategy.id);
        
        if (bets.length === 0) {
            return {
                ...strategy,
                roi: 0,
                profit: 0
            };
        }
        
        const totalStaked = bets.reduce((sum, bet) => sum + (bet.stake * userSettings.stakeUnit), 0);
        const totalProfit = bets.reduce((sum, bet) => {
            if (bet.result === 'won') {
                return sum + ((bet.odds - 1) * bet.stake * userSettings.stakeUnit);
            } else if (bet.result === 'lost') {
                return sum - (bet.stake * userSettings.stakeUnit);
            }
            return sum;
        }, 0);
        
        const roi = totalStaked > 0 ? (totalProfit / totalStaked) * 100 : 0;
        
        return {
            ...strategy,
            roi: roi,
            profit: totalProfit
        };
    }).filter(strategy => strategy.roi !== 0); // Filtrar estrategias sin apuestas
    
    if (strategiesWithStats.length === 0) {
        // Mostrar gráfico vacío
        strategyChart = new Chart(ctx, {
            type: 'bar',
            data: {
                labels: ['No hay datos'],
                datasets: [{
                    label: 'ROI (%)',
                    data: [0],
                    backgroundColor: 'rgba(201, 203, 207, 0.2)',
                    borderColor: 'rgb(201, 203, 207)',
                    borderWidth: 1
                }]
            },
            options: {
                responsive: true,
                scales: {
                    y: {
                        beginAtZero: true
                    }
                }
            }
        });
        return;
    }
    
    // Ordenar por ROI descendente
    strategiesWithStats.sort((a, b) => b.roi - a.roi);
    
    const labels = strategiesWithStats.map(strategy => strategy.name);
    const data = strategiesWithStats.map(strategy => strategy.roi);
    const backgroundColors = strategiesWithStats.map(strategy => 
        strategy.roi >= 0 ? 'rgba(75, 192, 192, 0.2)' : 'rgba(255, 99, 132, 0.2)'
    );
    const borderColors = strategiesWithStats.map(strategy => 
        strategy.roi >= 0 ? 'rgb(75, 192, 192)' : 'rgb(255, 99, 132)'
    );
    
    strategyChart = new Chart(ctx, {
        type: 'bar',
        data: {
            labels: labels,
            datasets: [{
                label: 'ROI (%)',
                data: data,
                backgroundColor: backgroundColors,
                borderColor: borderColors,
                borderWidth: 1
            }]
        },
        options: {
            responsive: true,
            plugins: {
                tooltip: {
                    callbacks: {
                        afterLabel: function(context) {
                            const strategy = strategiesWithStats[context.dataIndex];
                            return `Beneficio: ${strategy.profit.toFixed(2)}\nApuestas: ${userBets.filter(bet => bet.strategy === strategy.id).length}`;
                        }
                    }
                }
            },
            scales: {
                y: {
                    beginAtZero: true
                }
            }
        }
    });
}

// Actualizar gráfico de detalle de estrategia
function updateStrategyDetailChart(bets, strategyName) {
    const ctx = document.getElementById('strategy-detail-chart').getContext('2d');
    
    // Destruir gráfico anterior si existe
    if (strategyDetailChart) {
        strategyDetailChart.destroy();
    }
    
    if (bets.length === 0) {
        // Mostrar gráfico vacío
        strategyDetailChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: ['No hay datos'],
                datasets: [{
                    label: 'Bank',
                    data: [0],
                    borderColor: '#4cc9f0',
                    backgroundColor: 'rgba(76, 201, 240, 0.1)',
                    tension: 0.1,
                    fill: true
                }]
            },
            options: {
                responsive: true,
                plugins: {
                    title: {
                        display: true,
                        text: strategyName
                    }
                }
            }
        });
        return;
    }
    
    // Ordenar apuestas por fecha
    const sortedBets = [...bets].sort((a, b) => new Date(a.date) - new Date(b.date));
    
    // Calcular evolución del bank para esta estrategia
    let runningBank = userSettings.initialBank;
    const bankData = [{ x: new Date(sortedBets[0].date).toISOString(), y: runningBank }];
    
    sortedBets.forEach(bet => {
        if (bet.result === 'won') {
            runningBank += (bet.odds - 1) * bet.stake * userSettings.stakeUnit;
        } else if (bet.result === 'lost') {
            runningBank -= bet.stake * userSettings.stakeUnit;
        }
        
        bankData.push({
            x: new Date(bet.date).toISOString(),
            y: runningBank
        });
    });
    
    // Configurar gráfico
    strategyDetailChart = new Chart(ctx, {
        type: 'line',
        data: {
            datasets: [{
                label: 'Bank',
                data: bankData,
                borderColor: '#4cc9f0',
                backgroundColor: 'rgba(76, 201, 240, 0.1)',
                tension: 0.1,
                fill: true
            }]
        },
        options: {
            responsive: true,
            plugins: {
                title: {
                    display: true,
                    text: strategyName
                },
                tooltip: {
                    callbacks: {
                        label: function(context) {
                            return 'Bank: ' + context.parsed.y.toFixed(2);
                        }
                    }
                }
            },
            scales: {
                x: {
                    type: 'time',
                    time: {
                        unit: 'day',
                        tooltipFormat: 'DD MMM YYYY'
                    },
                    title: {
                        display: true,
                        text: 'Fecha'
                    }
                },
                y: {
                    title: {
                        display: true,
                        text: 'Bank'
                    }
                }
            }
        }
    });
}

// Configurar ajustes
function setupSettings() {
    const saveSettingsBtn = document.getElementById('save-settings-btn');
    const deleteAllBetsBtn = document.getElementById('delete-all-bets-btn');
    const deleteAccountBtn = document.getElementById('delete-account-btn');
    
    // Guardar ajustes - PARTE MODIFICADA
    saveSettingsBtn.addEventListener('click', () => {
        const initialBank = parseFloat(document.getElementById('initial-bank').value);
        const stakeUnit = parseFloat(document.getElementById('stake-unit').value);
        const defaultStake = parseFloat(document.getElementById('default-stake').value);
        
        // Validación mejorada
        if (isNaN(initialBank) || initialBank <= 0) {
            alert('Bank inicial debe ser un número mayor que cero');
            return;
        }
        
        if (isNaN(stakeUnit) || stakeUnit <= 0) {
            alert('Valor de unidad debe ser mayor que cero');
            return;
        }
        
        if (isNaN(defaultStake) || defaultStake <= 0) {
            alert('Stake predeterminado debe ser mayor que cero');
            return;
        }
        
        const newSettings = {
            initialBank: initialBank,
            stakeUnit: stakeUnit,
            defaultStake: defaultStake
        };
        
        // 1. Actualizar Firebase
        database.ref(`users/${currentUser.uid}/settings`).set(newSettings)
            .then(() => {
                // 2. Actualizar la variable local INMEDIATAMENTE
                userSettings = newSettings;
                
                // 3. Actualizar el stake en el formulario de apuestas
                document.getElementById('bet-stake').value = userSettings.defaultStake;
                
                // 4. Recalcular TODO el historial con los nuevos valores
                calculateBankHistory();
                updateDashboard();
                
                // 5. Feedback al usuario (puedes cambiar esto por showNotification si prefieres)
                alert('✅ Configuración guardada correctamente');
                
                // 6. [OPCIONAL] Forzar recarga de gráficos
                if (bankChart) bankChart.update();
                if (strategyChart) strategyChart.update();
            })
            .catch(error => {
                alert(`❌ Error al guardar: ${error.message}`);
            });
    });
    
    // Eliminar todas las apuestas
    deleteAllBetsBtn.addEventListener('click', () => {
        showConfirmModal(
            'Eliminar todas las apuestas',
            '¿Estás seguro de que quieres eliminar TODAS tus apuestas? Esta acción no se puede deshacer.',
            () => deleteAllBets()
        );
    });
    
    deleteAccountBtn.addEventListener('click', () => {
        showConfirmModal(
            'Eliminar cuenta',
            '¿Estás seguro de que quieres eliminar tu cuenta y TODOS tus datos? Esta acción no se puede deshacer.',
            () => deleteUserAccount()
        );
    });
}

// Eliminar todas las apuestas del usuario
function deleteAllBets() {
    database.ref(`users/${currentUser.uid}/bets`).remove()
        .then(() => {
            alert('Todas las apuestas han sido eliminadas');
        })
        .catch(error => {
            alert(`Error al eliminar apuestas: ${error.message}`);
        });
}

// Eliminar cuenta de usuario y todos sus datos
function deleteUserAccount() {
    // Eliminar todos los datos del usuario
    database.ref(`users/${currentUser.uid}`).remove()
        .then(() => {
            // Eliminar la cuenta de autenticación
            return currentUser.delete();
        })
        .then(() => {
            alert('Cuenta eliminada correctamente. Serás redirigido a la pantalla de inicio.');
        })
        .catch(error => {
            alert(`Error al eliminar cuenta: ${error.message}`);
        });
}

// Configurar modales
function setupModals() {
    const editBetModal = document.getElementById('edit-bet-modal');
    const confirmModal = document.getElementById('confirm-modal');
    const newStrategyModal = document.getElementById('new-strategy-modal');
    
    // Cerrar modales al hacer clic en la X
    document.querySelectorAll('.close-modal').forEach(btn => {
        btn.addEventListener('click', () => {
            editBetModal.classList.add('hidden');
            confirmModal.classList.add('hidden');
            newStrategyModal.classList.add('hidden');
        });
    });
    
    // Cerrar modales al hacer clic fuera del contenido
    [editBetModal, confirmModal, newStrategyModal].forEach(modal => {
        modal.addEventListener('click', (e) => {
            if (e.target === modal) {
                modal.classList.add('hidden');
            }
        });
    });
    
    // Configurar formulario de edición de apuesta
    const editBetForm = document.getElementById('edit-bet-form');
    editBetForm.addEventListener('submit', e => {
        e.preventDefault();
        saveEditedBet();
    });
    
    // Configurar modal de confirmación
    document.getElementById('confirm-cancel').addEventListener('click', () => {
        confirmModal.classList.add('hidden');
    });
}

// Mostrar modal de confirmación
function showConfirmModal(title, message, callback) {
    document.getElementById('confirm-title').textContent = title;
    document.getElementById('confirm-message').textContent = message;
    
    const confirmBtn = document.getElementById('confirm-ok');
    
    // Reemplazar cualquier evento anterior
    const newConfirmBtn = confirmBtn.cloneNode(true);
    confirmBtn.parentNode.replaceChild(newConfirmBtn, confirmBtn);
    
    newConfirmBtn.addEventListener('click', () => {
        document.getElementById('confirm-modal').classList.add('hidden');
        callback();
    });
    
    document.getElementById('confirm-modal').classList.remove('hidden');
}

// Editar apuesta
function editBet(betId) {
    const bet = userBets.find(b => b.id === betId);
    if (!bet) return;
    
    // Llenar formulario con datos de la apuesta
    document.getElementById('edit-bet-id').value = bet.id;
    document.getElementById('edit-bet-date').value = bet.date;
    document.getElementById('edit-bet-sport').value = bet.sport;
    document.getElementById('edit-bet-event').value = bet.event;
    document.getElementById('edit-bet-type').value = bet.type;
    document.getElementById('edit-bet-odds').value = bet.odds;
    document.getElementById('edit-bet-stake').value = bet.stake;
    document.getElementById('edit-bet-result').value = bet.result;
    document.getElementById('edit-bet-bookmaker').value = bet.bookmaker;
    document.getElementById('edit-bet-comments').value = bet.comments;
    
    // Seleccionar estrategia correcta
    const strategySelect = document.getElementById('edit-bet-strategy');
    if (Array.from(strategySelect.options).some(opt => opt.value === bet.strategy)) {
        strategySelect.value = bet.strategy;
    } else {
        strategySelect.value = 'default';
    }
    
    // Mostrar modal
    document.getElementById('edit-bet-modal').classList.remove('hidden');
}

// Guardar apuesta editada
function saveEditedBet() {
    const betId = document.getElementById('edit-bet-id').value;
    const bet = userBets.find(b => b.id === betId);
    if (!betId || !bet) return;
    
    const updatedBet = {
        date: document.getElementById('edit-bet-date').value,
        sport: document.getElementById('edit-bet-sport').value.trim(),
        event: document.getElementById('edit-bet-event').value.trim(),
        type: document.getElementById('edit-bet-type').value,
        odds: parseFloat(document.getElementById('edit-bet-odds').value),
        stake: parseFloat(document.getElementById('edit-bet-stake').value),
        result: document.getElementById('edit-bet-result').value,
        strategy: document.getElementById('edit-bet-strategy').value,
        bookmaker: document.getElementById('edit-bet-bookmaker').value.trim() || 'No especificado',
        comments: document.getElementById('edit-bet-comments').value.trim() || '',
        updatedAt: firebase.database.ServerValue.TIMESTAMP
    };
    
    // Validar campos
    if (!updatedBet.date || !updatedBet.sport || !updatedBet.event || !updatedBet.type || 
        isNaN(updatedBet.odds) || isNaN(updatedBet.stake) || !updatedBet.result || !updatedBet.strategy) {
        alert('Por favor completa todos los campos requeridos');
        return;
    }
    
    // Actualizar en Firebase
    database.ref(`users/${currentUser.uid}/bets/${betId}`).update(updatedBet)
        .then(() => {
            alert('Apuesta actualizada correctamente');
            document.getElementById('edit-bet-modal').classList.add('hidden');
        })
        .catch(error => {
            alert(`Error al actualizar apuesta: ${error.message}`);
        });
}

// Confirmar eliminación de apuesta
function confirmDeleteBet(betId) {
    const bet = userBets.find(b => b.id === betId);
    if (!bet) return;
    
    showConfirmModal(
        'Eliminar apuesta',
        `¿Estás seguro de que quieres eliminar la apuesta en ${bet.event} (${bet.type})?`,
        () => deleteBet(betId)
    );
}

// Eliminar apuesta
function deleteBet(betId) {
    database.ref(`users/${currentUser.uid}/bets/${betId}`).remove()
        .catch(error => {
            alert(`Error al eliminar apuesta: ${error.message}`);
        });
}

// Exportar datos a CSV
document.getElementById('export-csv-btn').addEventListener('click', exportToCSV);

function exportToCSV() {
    if (userBets.length === 0) {
        alert('No hay apuestas para exportar');
        return;
    }
    
    // Encabezados CSV
    let csv = 'Fecha,Deporte,Evento,Tipo,Cuota,Stake,Resultado,Estrategia,Casa de Apuestas,Beneficio,Comentarios\n';
    
    // Datos de cada apuesta
    userBets.forEach(bet => {
        const strategy = userStrategies.find(s => s.id === bet.strategy) || { name: 'Desconocida' };
        
        // Calcular beneficio
        let profit = 0;
        if (bet.result === 'won') {
            profit = (bet.odds - 1) * bet.stake * userSettings.stakeUnit;
        } else if (bet.result === 'lost') {
            profit = -bet.stake * userSettings.stakeUnit;
        }
        
        // Escapar comas y comillas en los textos
        const escapeCsv = text => {
            if (text.includes(',') || text.includes('"') || text.includes('\n')) {
                return `"${text.replace(/"/g, '""')}"`;
            }
            return text;
        };
        
        csv += `${formatDate(bet.date)},${escapeCsv(bet.sport)},${escapeCsv(bet.event)},${bet.type},${bet.odds.toFixed(2)},${bet.stake.toFixed(1)},${getResultText(bet.result)},${escapeCsv(strategy.name)},${escapeCsv(bet.bookmaker)},${profit.toFixed(2)},${escapeCsv(bet.comments)}\n`;
    });
    
    // Crear archivo y descargar
    const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = `bettracker_apuestas_${formatDate(new Date().toISOString(), false, true)}.csv`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
}

// Exportar datos a JSON
document.getElementById('export-json-btn').addEventListener('click', exportToJSON);

function exportToJSON() {
    const data = {
        settings: userSettings,
        strategies: userStrategies,
        bets: userBets,
        exportedAt: new Date().toISOString()
    };
    
    const json = JSON.stringify(data, null, 2);
    const blob = new Blob([json], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = `bettracker_datos_${formatDate(new Date().toISOString(), false, true)}.json`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
}

// Importar datos desde JSON
document.getElementById('import-json-btn').addEventListener('click', () => {
    document.getElementById('json-import-file').click();
});

document.getElementById('json-import-file').addEventListener('change', importFromJSON);

function importFromJSON(e) {
    const file = e.target.files[0];
    if (!file) return;
    
    showConfirmModal(
        'Importar datos',
        '¿Estás seguro de que quieres importar estos datos? Esto sobrescribirá tu configuración, estrategias y apuestas actuales.',
        () => {
            const reader = new FileReader();
            
            reader.onload = event => {
                try {
                    const data = JSON.parse(event.target.result);
                    
                    // Validar estructura básica
                    if (!data.settings || !data.strategies || !data.bets) {
                        throw new Error('El archivo no tiene el formato correcto');
                    }
                    
                    // Mostrar resumen de lo que se importará
                    const summary = `
                        Configuración: 1 conjunto
                        Estrategias: ${data.strategies.length}
                        Apuestas: ${data.bets.length}
                    `;
                    
                    showConfirmModal(
                        'Confirmar importación',
                        `Se importarán los siguientes datos:\n${summary}\n¿Continuar?`,
                        () => {
                            // Preparar datos para Firebase
                            const updates = {};
                            updates['settings'] = data.settings;
                            
                            // Convertir arrays de estrategias y apuestas a objetos Firebase
                            const strategiesObj = {};
                            data.strategies.forEach(strategy => {
                                strategiesObj[strategy.id] = {
                                    name: strategy.name,
                                    createdAt: strategy.createdAt || firebase.database.ServerValue.TIMESTAMP
                                };
                            });
                            updates['strategies'] = strategiesObj;
                            
                            const betsObj = {};
                            data.bets.forEach(bet => {
                                const newKey = database.ref().child('bets').push().key;
                                betsObj[newKey] = {
                                    date: bet.date,
                                    sport: bet.sport,
                                    event: bet.event,
                                    type: bet.type,
                                    odds: bet.odds,
                                    stake: bet.stake,
                                    result: bet.result,
                                    strategy: bet.strategy,
                                    bookmaker: bet.bookmaker,
                                    comments: bet.comments,
                                    createdAt: bet.createdAt || firebase.database.ServerValue.TIMESTAMP
                                };
                            });
                            updates['bets'] = betsObj;
                            
                            // Guardar en Firebase
                            database.ref(`users/${currentUser.uid}`).update(updates)
                                .then(() => {
                                    alert('Datos importados correctamente');
                                })
                                .catch(error => {
                                    alert(`Error al importar datos: ${error.message}`);
                                });
                        }
                    );
                } catch (error) {
                    alert(`Error al importar datos: ${error.message}`);
                }
            };
            
            reader.readAsText(file);
        }
    );
    
    // Resetear el input para permitir la misma selección de archivo otra vez
    e.target.value = '';
}

function saveStrategyStats() {
    if (!currentUser || userStrategies.length === 0) return;
    
    const strategiesWithStats = userStrategies.map(strategy => {
        const bets = userBets.filter(bet => bet.strategy === strategy.id);
        const totalBets = bets.length;
        const wonBets = bets.filter(bet => bet.result === 'won').length;
        const lostBets = bets.filter(bet => bet.result === 'lost').length;
        const voidBets = bets.filter(bet => bet.result === 'void').length;
        
        const totalStaked = bets.reduce((sum, bet) => sum + (bet.stake * userSettings.stakeUnit), 0);
        const totalProfit = bets.reduce((sum, bet) => {
            if (bet.result === 'won') return sum + ((bet.odds - 1) * bet.stake * userSettings.stakeUnit);
            if (bet.result === 'lost') return sum - (bet.stake * userSettings.stakeUnit);
            return sum;
        }, 0);
        
        const roi = totalStaked > 0 ? (totalProfit / totalStaked) * 100 : 0;
        
        return {
            id: strategy.id,
            name: strategy.name,
            roi: roi,
            profit: totalProfit,
            totalBets: totalBets,
            wonBets: wonBets
        };
    });
    
    database.ref(`users/${currentUser.uid}/strategyStats`).set(strategiesWithStats)
        .catch(error => console.error("Error guardando stats de estrategias:", error));
}

// Funciones de utilidad
function formatDate(dateString, short = false, forFileName = false) {
    const date = new Date(dateString);
    
    if (forFileName) {
        return date.toISOString().slice(0, 10).replace(/-/g, '');
    }
    
    if (short) {
        return date.toLocaleDateString('es-ES', { 
            month: 'short', 
            day: 'numeric' 
        });
    }
    
    return date.toLocaleString('es-ES', {
        day: '2-digit',
        month: '2-digit',
        year: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
    });
}

function isSameDay(date1, date2) {
    return date1.getFullYear() === date2.getFullYear() &&
           date1.getMonth() === date2.getMonth() &&
           date1.getDate() === date2.getDate();
}

function getResultText(result) {
    switch (result) {
        case 'won': return 'Ganada';
        case 'lost': return 'Perdida';
        case 'void': return 'Void';
        default: return 'Desconocido';
    }
}
