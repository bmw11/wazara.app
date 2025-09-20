<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>نظام إدارة المحروقات</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                }
            }
        }
    </script>
    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signOut, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, collection, query, where, onSnapshot, addDoc, updateDoc, setDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { getStorage, ref, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-storage.js";

        // Global variables for Firebase configuration
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let app, db, auth, storage;
        let userId = null;
        let username = null;
        let userRole = null;
        
        // Hardcoded users and their credentials
        const users = {
            "ابو العز": { passcode: "1110", role: "employee" },
            "غسان": { passcode: "6543", role: "employee" },
            "خباب": { passcode: "2342", role: "employee" },
            "قصي": { passcode: "9876", role: "employee" },
            "ابو علي": { passcode: "2882", role: "employee" },
            "صالح": { passcode: "100", role: "supervisor" },
            "حازم": { passcode: "200", role: "supervisor" },
            "عمر": { passcode: "300", role: "supervisor" }
        };

        // DOM elements
        const loadingScreen = document.getElementById('loadingScreen');
        const loginView = document.getElementById('loginView');
        const employeeDashboard = document.getElementById('employeeDashboard');
        const supervisorDashboard = document.getElementById('supervisorDashboard');
        const requestModal = document.getElementById('requestModal');
        const rejectModal = document.getElementById('rejectModal');
        const messageModal = document.getElementById('messageModal');
        const employeeDashboardTitle = document.getElementById('employeeDashboardTitle');
        const employeeRequestsList = document.getElementById('employeeRequestsList');
        const supervisorRequestsList = document.getElementById('supervisorRequestsList');
        
        let pendingSupervisorAction = {};

        // --- Helper Functions ---
        const showView = (view) => {
            const views = [loginView, employeeDashboard, supervisorDashboard, loadingScreen];
            views.forEach(v => v.classList.add('hidden'));
            view.classList.remove('hidden');
        };

        const showMessage = (title, message) => {
            document.getElementById('messageTitle').textContent = title;
            document.getElementById('messageBody').textContent = message;
            messageModal.classList.remove('hidden');
        };

        const hideMessage = () => {
            messageModal.classList.add('hidden');
        };

        const setLoading = (isLoading) => {
            loadingScreen.classList.toggle('hidden', !isLoading);
        };
        
        // --- Authentication & Initialization ---
        const initFirebase = async () => {
            setLoading(true);
            try {
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                storage = getStorage(app);

                onAuthStateChanged(auth, async (user) => {
                    if (user) {
                        userId = user.uid;
                        if (userRole === 'supervisor') {
                            showView(supervisorDashboard);
                            fetchSupervisorRequests();
                        } else if (userRole === 'employee') {
                            employeeDashboardTitle.textContent = `مرحباً بك يا ${username}!`;
                            showView(employeeDashboard);
                            fetchEmployeeRequests();
                        } else {
                            // Initial state, show login
                            showView(loginView);
                        }
                    } else {
                        userId = null;
                        username = null;
                        userRole = null;
                        showView(loginView);
                    }
                    setLoading(false);
                });

                window.handleSignOut = handleSignOut;
                window.handleLogin = handleLogin;
                window.handleSubmitRequest = handleSubmitRequest;
                window.showRequestDetails = showRequestDetails;
                window.updateRequestStatus = updateRequestStatus;
                window.showRejectModal = showRejectModal;
                window.handleRejectSubmit = handleRejectSubmit;
                window.hideMessage = hideMessage;

            } catch (error) {
                console.error("Firebase Initialization Error:", error);
                showMessage('خطأ', 'حدث خطأ أثناء تهيئة التطبيق. يرجى إعادة المحاولة.');
                setLoading(false);
            }
        };

        const handleLogin = async () => {
            const usernameInput = document.getElementById('username').value.trim();
            const passcodeInput = document.getElementById('passcode').value.trim();

            const user = users[usernameInput];
            if (!user || user.passcode !== passcodeInput) {
                showMessage('خطأ في تسجيل الدخول', 'اسم المستخدم أو رمز المرور غير صحيح.');
                return;
            }

            setLoading(true);
            username = usernameInput;
            userRole = user.role;
            try {
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }
            } catch(error) {
                console.error("Login Error:", error);
                showMessage('خطأ في تسجيل الدخول', 'فشل تسجيل الدخول. يرجى المحاولة مرة أخرى.');
                setLoading(false);
            }
        };

        const handleSignOut = async () => {
            await signOut(auth);
        };
        
        // --- Employee Functions ---
        const handleSubmitRequest = async () => {
            // A more robust check to ensure the user is authenticated before attempting an action that requires permissions.
            if (!auth.currentUser) {
                showMessage('خطأ', 'يجب عليك تسجيل الدخول أولاً لإرسال طلب.');
                return;
            }

            const form = document.getElementById('fuelForm');
            const driverName = form.driverName.value;
            const fuelType = form.fuelType.value;
            const cardNumber = form.cardNumber.value;
            const receiptNumber = form.receiptNumber.value;
            const vehicleName = form.vehicleName.value;
            const filledAmount = form.filledAmount.value;
            const remainingAmount = form.remainingAmount.value;
            
            // Note: The card image file input and handling have been removed.

            if (!driverName || !fuelType || !cardNumber || !receiptNumber || !vehicleName || !filledAmount || !remainingAmount) {
                showMessage('خطأ', 'الرجاء ملء جميع الحقول.');
                return;
            }
            
            setLoading(true);
            try {
                // Add request to Firestore
                await addDoc(collection(db, `artifacts/${appId}/public/data/requests`), {
                    userId: auth.currentUser.uid, // Use auth.currentUser.uid for consistency
                    username: username,
                    driverName: driverName,
                    fuelType: fuelType,
                    cardNumber: cardNumber,
                    receiptNumber: receiptNumber,
                    vehicleName: vehicleName,
                    filledAmount: parseFloat(filledAmount),
                    remainingAmount: parseFloat(remainingAmount),
                    status: 'pending',
                    rejectionReason: '',
                    timestamp: new Date()
                });
                
                showMessage('تم الإرسال', 'تم إرسال طلبك بنجاح. سيتم معالجته قريباً.');
                form.reset();
            } catch (error) {
                console.error("Submission Error:", error);
                showMessage('خطأ', 'فشل في إرسال الطلب. يرجى المحاولة مرة أخرى.');
            } finally {
                setLoading(false);
            }
        };

        const fetchEmployeeRequests = () => {
            if (!userId) return;
            const q = query(collection(db, `artifacts/${appId}/public/data/requests`), where("userId", "==", userId));
            
            onSnapshot(q, (querySnapshot) => {
                employeeRequestsList.innerHTML = '';
                if (querySnapshot.empty) {
                    employeeRequestsList.innerHTML = `<p class="text-gray-500 text-center mt-4">لا توجد طلبات وقود حتى الآن.</p>`;
                } else {
                    querySnapshot.forEach((doc) => {
                        const data = doc.data();
                        const statusColor = data.status === 'accepted' ? 'text-green-600' : (data.status === 'rejected' ? 'text-red-600' : 'text-yellow-600');
                        employeeRequestsList.innerHTML += `
                            <div class="bg-white p-4 rounded-lg shadow-md mb-4 border border-gray-200">
                                <p class="text-sm text-gray-500">تم الإرسال: ${data.timestamp && data.timestamp.toDate ? new Date(data.timestamp.toDate()).toLocaleString('ar-SA') : 'N/A'}</p>
                                <p class="font-bold text-lg mt-2">الحالة: <span class="${statusColor}">${data.status === 'pending' ? 'في انتظار المعالجة' : (data.status === 'accepted' ? 'مقبول' : 'مرفوض')}</span></p>
                                ${data.status === 'rejected' ? `<p class="text-red-500 mt-2">سبب الرفض: ${data.rejectionReason}</p>` : ''}
                            </div>
                        `;
                    });
                }
            });
        };

        // --- Supervisor Functions ---
        const fetchSupervisorRequests = () => {
            const q = query(collection(db, `artifacts/${appId}/public/data/requests`));
            
            onSnapshot(q, (querySnapshot) => {
                supervisorRequestsList.innerHTML = '';
                if (querySnapshot.empty) {
                    supervisorRequestsList.innerHTML = `<p class="text-gray-500 text-center mt-4">لا توجد طلبات وقود جديدة.</p>`;
                } else {
                    querySnapshot.forEach((doc) => {
                        const data = doc.data();
                        const statusColor = data.status === 'accepted' ? 'bg-green-100' : (data.status === 'rejected' ? 'bg-red-100' : 'bg-yellow-100');
                        supervisorRequestsList.innerHTML += `
                            <div class="bg-white p-4 rounded-lg shadow-md mb-4 border border-gray-200 ${statusColor}">
                                <p class="text-sm text-gray-500">تم الإرسال: ${data.timestamp && data.timestamp.toDate ? new Date(data.timestamp.toDate()).toLocaleString('ar-SA') : 'N/A'}</p>
                                <p class="font-bold text-lg mt-2">اسم الموظف: ${data.username}</p>
                                <p>نوع الوقود: ${data.fuelType}</p>
                                <p>الكمية: ${data.filledAmount} لتر</p>
                                <p>الحالة: ${data.status === 'pending' ? 'في انتظار المعالجة' : (data.status === 'accepted' ? 'مقبول' : 'مرفوض')}</p>
                                ${data.status !== 'pending' ? `<p class="mt-2 text-sm text-gray-600">تم اتخاذ الإجراء بواسطة المشرف</p>` : ''}
                                <div class="mt-4 flex flex-col sm:flex-row gap-2">
                                    <button onclick="showRequestDetails('${doc.id}', '${encodeURIComponent(JSON.stringify(data))}')" class="w-full sm:w-1/2 bg-blue-500 text-white p-2 rounded-lg hover:bg-blue-600 transition-colors">عرض التفاصيل</button>
                                </div>
                            </div>
                        `;
                    });
                }
            });
        };

        const showRequestDetails = (docId, dataString) => {
            try {
                const data = JSON.parse(decodeURIComponent(dataString));
                document.getElementById('modalDriverName').textContent = data.driverName || 'N/A';
                document.getElementById('modalFuelType').textContent = data.fuelType || 'N/A';
                document.getElementById('modalCardNumber').textContent = data.cardNumber || 'N/A';
                document.getElementById('modalReceiptNumber').textContent = data.receiptNumber || 'N/A';
                document.getElementById('modalVehicleName').textContent = data.vehicleName || 'N/A';
                document.getElementById('modalFilledAmount').textContent = `${data.filledAmount || 'N/A'} لتر`;
                document.getElementById('modalRemainingAmount').textContent = `${data.remainingAmount || 'N/A'} لتر`;
                document.getElementById('modalTimestamp').textContent = data.timestamp && data.timestamp.toDate ? new Date(data.timestamp.toDate()).toLocaleString('ar-SA') : 'N/A';
            
                const actionButtons = document.getElementById('modalActionButtons');
                actionButtons.innerHTML = '';
                if (data.status === 'pending') {
                    actionButtons.innerHTML = `
                        <button onclick="updateRequestStatus('${docId}', 'accepted')" class="w-full sm:w-1/2 bg-green-500 text-white p-2 rounded-lg hover:bg-green-600 transition-colors">قبول</button>
                        <button onclick="showRejectModal('${docId}')" class="w-full sm:w-1/2 bg-red-500 text-white p-2 rounded-lg hover:bg-red-600 transition-colors">رفض</button>
                    `;
                } else {
                    actionButtons.innerHTML = `<p class="text-center font-bold text-gray-600">تمت معالجة هذا الطلب.</p>`;
                }
                requestModal.classList.remove('hidden');

            } catch (e) {
                console.error("Failed to parse request data", e);
                showMessage('خطأ', 'فشل في عرض التفاصيل.');
            }
        };

        const updateRequestStatus = async (docId, status, rejectionReason = '') => {
            setLoading(true);
            try {
                await updateDoc(doc(db, `artifacts/${appId}/public/data/requests/${docId}`), {
                    status: status,
                    rejectionReason: rejectionReason,
                    processedBy: username
                });
                requestModal.classList.add('hidden');
                rejectModal.classList.add('hidden');
                showMessage('تم', `تم ${status === 'accepted' ? 'قبول' : 'رفض'} الطلب بنجاح.`);
            } catch (error) {
                console.error("Update status error:", error);
                showMessage('خطأ', 'فشل في تحديث حالة الطلب.');
            } finally {
                setLoading(false);
            }
        };

        const showRejectModal = (docId) => {
            pendingSupervisorAction.docId = docId;
            rejectModal.classList.remove('hidden');
        };

        const handleRejectSubmit = () => {
            const reason = document.getElementById('rejectReason').value;
            if (!reason) {
                showMessage('خطأ', 'الرجاء كتابة سبب الرفض.');
                return;
            }
            updateRequestStatus(pendingSupervisorAction.docId, 'rejected', reason);
        };
        
        // Initialize the app on page load
        window.onload = initFirebase;
    </script>
</head>
<body class="bg-gray-100 font-sans p-4 min-h-screen flex items-center justify-center">

    <!-- Loading Screen -->
    <div id="loadingScreen" class="fixed inset-0 bg-gray-100 z-50 flex flex-col items-center justify-center hidden">
        <svg class="animate-spin -ml-1 mr-3 h-10 w-10 text-blue-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
        </svg>
        <p class="mt-4 text-gray-600 text-lg">الرجاء الانتظار...</p>
    </div>

    <!-- Main Content Area -->
    <div class="w-full max-w-2xl bg-white p-8 rounded-xl shadow-lg border border-gray-200">
        <h1 class="text-3xl font-bold text-center mb-6 text-gray-800">نظام إدارة المحروقات</h1>

        <!-- Login View -->
        <div id="loginView" class="view">
            <h2 class="text-2xl font-semibold mb-4 text-center">تسجيل الدخول</h2>
            <div class="space-y-4">
                <div>
                    <label for="username" class="block text-gray-700 font-medium mb-2">اسم المستخدم:</label>
                    <input type="text" id="username" name="username" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500" placeholder="أدخل اسم المستخدم">
                </div>
                <div>
                    <label for="passcode" class="block text-gray-700 font-medium mb-2">رمز المرور:</label>
                    <input type="password" id="passcode" name="passcode" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500" placeholder="أدخل رمز المرور">
                </div>
            </div>
            <button onclick="handleLogin()" class="w-full mt-6 bg-blue-500 text-white p-3 rounded-lg font-semibold hover:bg-blue-600 transition-colors">تسجيل الدخول</button>
        </div>
        
        <!-- Employee Dashboard -->
        <div id="employeeDashboard" class="view hidden">
            <div class="flex items-center justify-between mb-6">
                <h2 class="text-2xl font-semibold" id="employeeDashboardTitle">مرحباً بك!</h2>
                <button onclick="handleSignOut()" class="text-red-500 hover:text-red-700 transition-colors">تسجيل الخروج</button>
            </div>
            
            <!-- Fuel Form Section -->
            <div>
                <h3 class="text-xl font-semibold mb-4 border-b pb-2">إضافة تعبئة وقود جديدة</h3>
                <form id="fuelForm" class="space-y-4">
                    <div>
                        <label for="driverName" class="block text-sm font-medium text-gray-700">اسم السائق</label>
                        <input type="text" id="driverName" name="driverName" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 p-2 border">
                    </div>
                    <div>
                        <label for="fuelType" class="block text-sm font-medium text-gray-700">نوع الوقود</label>
                        <select id="fuelType" name="fuelType" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 p-2 border">
                            <option value="بنزين">بنزين</option>
                            <option value="مازوت">مازوت</option>
                        </select>
                    </div>
                    <div>
                        <label for="cardNumber" class="block text-sm font-medium text-gray-700">رقم البطاقة</label>
                        <input type="text" id="cardNumber" name="cardNumber" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 p-2 border">
                    </div>
                    <div>
                        <label for="receiptNumber" class="block text-sm font-medium text-gray-700">رقم ايصال الدفع</label>
                        <input type="text" id="receiptNumber" name="receiptNumber" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 p-2 border">
                    </div>
                    <div>
                        <label for="vehicleName" class="block text-sm font-medium text-gray-700">اسم المركبة</label>
                        <input type="text" id="vehicleName" name="vehicleName" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 p-2 border">
                    </div>
                    <div>
                        <label for="filledAmount" class="block text-sm font-medium text-gray-700">كمية الوقود المعبأ (بالليتر)</label>
                        <input type="number" id="filledAmount" name="filledAmount" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 p-2 border">
                    </div>
                     <div>
                        <label for="remainingAmount" class="block text-sm font-medium text-gray-700">كمية الوقود الباقية في البطاقة (بالليتر)</label>
                        <input type="number" id="remainingAmount" name="remainingAmount" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 p-2 border">
                    </div>
                    <!-- The file input for the card image has been removed -->
                    <button type="button" onclick="handleSubmitRequest()" class="w-full bg-blue-500 text-white p-3 rounded-lg font-semibold hover:bg-blue-600 transition-colors">إرسال الطلب</button>
                </form>
            </div>

            <!-- Requests Section (Employee) -->
            <div class="mt-8">
                <h3 class="text-xl font-semibold mb-4 border-b pb-2">حالة طلباتك</h3>
                <div id="employeeRequestsList" class="space-y-4">
                    <!-- Requests will be populated here by onSnapshot -->
                </div>
            </div>
        </div>

        <!-- Supervisor Dashboard -->
        <div id="supervisorDashboard" class="view hidden">
             <div class="flex items-center justify-between mb-6">
                <h2 class="text-2xl font-semibold">لوحة تحكم المشرف</h2>
                <button onclick="handleSignOut()" class="text-red-500 hover:text-red-700 transition-colors">تسجيل الخروج</button>
            </div>
            <h3 class="text-xl font-semibold mb-4 border-b pb-2">الطلبات الواردة</h3>
            <div id="supervisorRequestsList" class="space-y-4">
                <!-- Requests will be populated here by onSnapshot -->
            </div>
        </div>

    </div>

    <!-- Modals -->
    <!-- Supervisor Request Details Modal -->
    <div id="requestModal" class="fixed inset-0 bg-gray-900 bg-opacity-75 flex items-center justify-center p-4 hidden">
        <div class="bg-white p-6 rounded-lg shadow-xl max-w-lg w-full">
            <h3 class="text-2xl font-bold mb-4">تفاصيل طلب الوقود</h3>
            <div class="space-y-3 text-gray-700">
                <p><strong>اسم السائق:</strong> <span id="modalDriverName"></span></p>
                <p><strong>نوع الوقود:</strong> <span id="modalFuelType"></span></p>
                <p><strong>رقم البطاقة:</strong> <span id="modalCardNumber"></span></p>
                <p><strong>رقم الإيصال:</strong> <span id="modalReceiptNumber"></span></p>
                <p><strong>اسم المركبة:</strong> <span id="modalVehicleName"></span></p>
                <p><strong>الكمية المعبأة:</strong> <span id="modalFilledAmount"></span></p>
                <p><strong>الكمية المتبقية:</strong> <span id="modalRemainingAmount"></span></p>
                <p><strong>تاريخ الإرسال:</strong> <span id="modalTimestamp"></span></p>
            </div>
            
            <div id="modalActionButtons" class="mt-6 flex flex-col sm:flex-row gap-2">
                <!-- Action buttons will be added here dynamically -->
            </div>
            <button onclick="requestModal.classList.add('hidden')" class="w-full mt-4 bg-gray-200 text-gray-800 p-2 rounded-lg hover:bg-gray-300 transition-colors">إغلاق</button>
        </div>
    </div>
    
    <!-- Reject Reason Modal -->
    <div id="rejectModal" class="fixed inset-0 bg-gray-900 bg-opacity-75 flex items-center justify-center p-4 hidden">
        <div class="bg-white p-6 rounded-lg shadow-xl max-w-sm w-full">
            <h3 class="text-2xl font-bold mb-4">سبب الرفض</h3>
            <textarea id="rejectReason" rows="4" class="w-full p-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-red-500" placeholder="اكتب سبب رفض الطلب..."></textarea>
            <div class="mt-4 flex gap-2">
                <button onclick="handleRejectSubmit()" class="w-full bg-red-500 text-white p-2 rounded-lg hover:bg-red-600 transition-colors">إرسال</button>
                <button onclick="rejectModal.classList.add('hidden')" class="w-full bg-gray-200 text-gray-800 p-2 rounded-lg hover:bg-gray-300 transition-colors">إلغاء</button>
            </div>
        </div>
    </div>

    <!-- General Message Modal -->
    <div id="messageModal" class="fixed inset-0 bg-gray-900 bg-opacity-75 flex items-center justify-center p-4 hidden">
        <div class="bg-white p-6 rounded-lg shadow-xl max-w-sm w-full text-center">
            <h3 id="messageTitle" class="text-xl font-bold mb-2"></h3>
            <p id="messageBody" class="text-gray-700"></p>
            <button onclick="hideMessage()" class="mt-4 bg-blue-500 text-white p-2 rounded-lg hover:bg-blue-600 transition-colors">حسناً</button>
        </div>
    </div>

</body>
</html>
