.github/workflows/azure-webapps-node.yml<!DOCTYPE html>
<html lang="ar" dir="rtl"> <!-- Added dir="rtl" for right-to-left layout -->
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>إصلاح مصادقة Firebase</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            font-family: "Inter", sans-serif;
            background-color: #f0f4f8;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            margin: 0;
            padding: 1rem;
            box-sizing: border-box; /* Include padding in element's total width and height */
        }
        .container {
            background-color: #ffffff;
            padding: 2.5rem;
            border-radius: 0.75rem;
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
            max-width: 90%;
            width: 400px;
            text-align: center;
            display: flex; /* Use flexbox for internal layout */
            flex-direction: column; /* Stack items vertically */
            gap: 1rem; /* Space between elements */
        }
        .message-box {
            background-color: #ffe0b2; /* برتقالي فاتح */
            color: #e65100; /* برتقالي داكن */
            padding: 1rem;
            border-radius: 0.5rem;
            margin-bottom: 1rem;
            border: 1px solid #ff9800;
            display: none; /* مخفي افتراضياً */
        }
        #appContent {
            margin-top: 1.5rem;
            padding-top: 1.5rem;
            border-top: 1px solid #e2e8f0; /* Light gray border */
            text-align: right; /* Align content to the right for RTL */
            display: none; /* Hidden by default using CSS */
        }
        #appContent h2 {
            font-size: 1.75rem; /* text-2xl */
            font-weight: 700; /* font-bold */
            color: #334155; /* slate-700 */
            margin-bottom: 0.75rem; /* mb-3 */
        }
        #appContent p {
            font-size: 1rem; /* text-base */
            color: #475569; /* slate-600 */
            line-height: 1.5;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1 class="text-3xl font-bold text-gray-800 mb-6">مصادقة Firebase</h1>
        <div id="messageBox" class="message-box"></div>
        <p id="authStatus" class="text-lg text-gray-700 mb-4">تهيئة Firebase...</p>
        <p class="text-sm text-gray-500">معرف المستخدم الخاص بك: <span id="userIdDisplay" class="font-mono">جار التحميل...</span></p>

        <!-- New section for application content - now hidden by CSS initially -->
        <div id="appContent">
            <h2>مرحباً بك في تطبيقك!</h2>
            <p>هذا هو المحتوى الذي يظهر بعد تسجيل الدخول بنجاح. يمكنك إضافة وظائف أو معلومات هنا.</p>
            <p>تم جلب بيانات ملفك الشخصي من Firestore:</p>
            <pre id="profileData" class="bg-gray-100 p-3 rounded-md mt-2 text-sm text-gray-700 text-right overflow-auto"></pre>
        </div>
    </div>

    <script type="module">
        // استيراد وحدات Firebase من CDN
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, onSnapshot } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // متغيرات Firebase العامة
        let app;
        let auth;
        let db;
        let userId;

        const messageBox = document.getElementById('messageBox');
        const authStatusElement = document.getElementById('authStatus');
        const userIdDisplayElement = document.getElementById('userIdDisplay');
        const appContentElement = document.getElementById('appContent');
        const profileDataElement = document.getElementById('profileData');

        /**
         * تعرض رسالة للمستخدم في صندوق الرسائل.
         * @param {string} message - الرسالة المراد عرضها.
         * @param {boolean} isError - إذا كانت صحيحة، تعرض الرسالة كخطأ.
         */
        function showMessage(message, isError = false) {
            messageBox.textContent = message;
            messageBox.style.display = 'block';
            if (isError) {
                messageBox.style.backgroundColor = '#ffcdd2'; /* أحمر فاتح */
                messageBox.style.color = '#b71c1c'; /* أحمر داكن */
                messageBox.style.borderColor = '#ef5350';
            } else {
                messageBox.style.backgroundColor = '#e8f5e9'; /* أخضر فاتح */
                messageBox.style.color = '#2e7d32'; /* أخضر داكن */
                messageBox.style.borderColor = '#66bb6a';
            }
        }

        // تهيئة Firebase والتعامل مع المصادقة
        window.onload = async function() {
            console.log("Window loaded. Initializing Firebase...");
            // appContent is now hidden by default via CSS. No need for JS hide here.

            try {
                // الوصول إلى المتغيرات العامة المقدمة من بيئة Canvas
                const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
                const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;a
                const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

                if (!firebaseConfig) {
                    console.error("لم يتم العثور على تهيئة Firebase.");
                    showMessage("خطأ: تهيئة Firebase مفقودة. لا يمكن تهيئة التطبيق.", true);
                    return;
                }

                // تهيئة تطبيق Firebase
                app = initializeApp(firebaseConfig);
                auth = getAuth(app);
                db = getFirestore(app);
                console.log("Firebase app, auth, and db initialized.");

                authStatusElement.textContent = "جاري المصادقة...";

                // الاستماع إلى تغييرات حالة المصادقة
                onAuthStateChanged(auth, async (user) => {
                    console.log("onAuthStateChanged triggered. User:", user ? user.uid : "null");
                    if (user) {
                        userId = user.uid;
                        authStatusElement.textContent = `تمت المصادقة كـ: ${userId}`;
                        userIdDisplayElement.textContent = userId;
                        console.log("المصادقة ناجحة، معرف المستخدم:", userId);
                        showMessage("المصادقة ناجحة!");

                        // عرض قسم المحتوى الخاص بالتطبيق
                        if (appContentElement) { // Safety check
                            appContentElement.style.display = 'block';
                            console.log("Authenticated: appContent set to display: block.");
                        } else {
                            console.error("Error: appContent element not found for display.");
                        }


                        // مثال على عملية Firestore بعد المصادقة
                        const usersCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/profile`);
                        const userDocRef = doc(usersCollectionRef, "data");

                        await setDoc(userDocRef, {
                            lastLogin: new Date().toISOString(),
                            appId: appId,
                            welcomeMessage: "أهلاً بك في تطبيقنا المتصل بـ Firebase!"
                        }, { merge: true });
                        console.log("تم تحديث بيانات ملف تعريف المستخدم في Firestore.");

                        // إعداد مستمع في الوقت الفعلي لبيانات المستخدم
                        onSnapshot(userDocRef, (docSnap) => {
                            if (docSnap.exists()) {
                                const data = docSnap.data();
                                console.log("بيانات الملف الشخصي الحالية:", data);
                                if (profileDataElement) { // Safety check
                                    profileDataElement.textContent = JSON.stringify(data, null, 2);
                                }
                            } else {
                                console.log("لم يتم العثور على بيانات ملف تعريف للمستخدم.");
                                if (profileDataElement) { // Safety check
                                    profileDataElement.textContent = "لا توجد بيانات ملف شخصي.";
                                }
                            }
                        }, (error) => {
                            console.error("خطأ أثناء الاستماع إلى بيانات الملف الشخصي:", error);
                            showMessage(`خطأ أثناء الاستماع إلى البيانات: ${error.message}`, true);
                        });

                    } else {
                        userId = 'not-authenticated';
                        authStatusElement.textContent = "لم تتم المصادقة. جاري محاولة تسجيل الدخول...";
                        userIdDisplayElement.textContent = "غير متاح";
                        // إخفاء قسم المحتوى الخاص بالتطبيق
                        if (appContentElement) { // Safety check
                            appContentElement.style.display = 'none';
                            console.log("Not authenticated: appContent set to display: none.");
                        }
                        if (profileDataElement) { // Safety check
                            profileDataElement.textContent = "";
                        }
                        console.log("المستخدم مسجل خروج. جاري محاولة تسجيل الدخول الأولي.");

                        try {
                            if (initialAuthToken) {
                                console.log("محاولة تسجيل الدخول باستخدام الرمز المميز المخصص...");
                                await signInWithCustomToken(auth, initialAuthToken);
                                console.log("تم تسجيل الدخول باستخدام الرمز المميز المخصص.");
                            } else {
                                console.log("محاولة تسجيل الدخول بشكل مجهول...");
                                await signInAnonymously(auth);
                                console.log("تم تسجيل الدخول بشكل مجهول.");
                            }
                        } catch (error) {
                            console.error("فشل تسجيل الدخول:", error);
                            showMessage(`فشل تسجيل الدخول: ${error.message}. يرجى التأكد من تمكين طرق المصادقة في لوحة تحكم Firebase.`, true);
                            authStatusElement.textContent = "فشلت المصادقة.";
                            userIdDisplayElement.textContent = "خطأ";
                        }
                    }
                });

            } catch (error) {
                console.error("خطأ في تهيئة Firebase:", error);
                showMessage(`خطأ في تهيئة Firebase: ${error.message}`, true);
            }
        };
    </script>
</body>
</html>
