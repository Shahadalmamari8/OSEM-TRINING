<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>OSEM Training Center Platform</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap');
        body { font-family: 'Inter', sans-serif; }
        .antialiased { -webkit-font-smoothing: antialiased; -moz-osx-font-smoothing: grayscale; }
        .modal-enter { opacity: 0; transform: scale(0.95); }
        .modal-enter-active { opacity: 1; transform: scale(1); transition: opacity 300ms, transform 300ms; }
        .modal-exit { opacity: 1; transform: scale(1); }
        .modal-exit-active { opacity: 0; transform: scale(0.95); transition: opacity 300ms, transform 300ms; }
    </style>
</head>
<body class="bg-gray-50 antialiased min-h-screen">
    <div id="app-root">
        <!-- Initial loading state, replaced by JavaScript -->
        <div class="flex justify-center items-center h-screen text-xl text-gray-600">Initializing Application...</div>
    </div>

    <script type="module">
        // Import Firebase modules
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, onSnapshot, setDoc, addDoc, updateDoc, query, writeBatch, Timestamp, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- GLOBAL VARIABLES & CONFIG (MANDATORY USE) ---
        // These variables are provided by the canvas environment
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-osem-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        
        const appRoot = document.getElementById('app-root');

        // --- GLOBAL STATE ---
        let state = {
            db: null,
            auth: null,
            userId: null,
            isAuthReady: false,
            userEmail: "anonymous@osem.com",
            
            courses: [],
            enrollments: [],
            certificates: [],
            
            view: 'home', // 'home', 'catalog', 'details', 'dashboard', 'admin'
            selectedCourse: null,
            isLoading: false,
            error: null,
            
            // Admin State
            adminView: 'courses', // 'courses', 'certs'
            editingCourse: null, // null or course object for editing/creating
            
            // Catalog Filters
            searchTerm: '',
            filterSpecialty: 'All',
            filterLocation: 'All',
        };

        // --- STATE MANAGEMENT AND RENDER LOOP ---
        function updateState(newState, forceRender = true) {
            // Check if essential data arrays are changing, which always forces a full render
            const dataChanged = newState.hasOwnProperty('courses') || newState.hasOwnProperty('enrollments') || newState.hasOwnProperty('certificates');
            
            state = { ...state, ...newState };

            // Only call render if a major state part changed
            if (forceRender || dataChanged) {
                render();
            }
        }

        // --- UTILITIES & MOCK APIS ---

        const uuidv4 = () => 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
            const r = Math.random() * 16 | 0;
            const v = c === 'x' ? r : (r & 0x3 | 0x8);
            return v.toString(16);
        });

        const addDays = (date, days) => {
            const result = new Date(date);
            result.setDate(result.getDate() + days);
            return result;
        };
        
        // Mock payment processing via Stripe
        const processStripePayment = async (courseId, amount) => {
            console.log(`MOCK: Processing Stripe payment for Course ID: ${courseId}, Amount: $${amount}`);
            await new Promise(resolve => setTimeout(resolve, 1500)); // Simulate network delay
            return { success: true, transactionId: uuidv4() };
        };

        // Mock email service for certificate delivery
        const sendCertificateEmail = async (userEmail, courseName) => {
            console.log(`MOCK: Sending Certificate Email to ${userEmail} for ${courseName}`);
            await new Promise(resolve => setTimeout(resolve, 500));
            return { success: true };
        };

        // Mock email service for expiration reminders
        const sendExpirationReminder = async (userEmail, courseName) => {
            console.log(`MOCK: Sending Expiration Reminder Email to ${userEmail} for ${courseName}`);
            await new Promise(resolve => setTimeout(resolve, 500));
            return { success: true };
        };
        
        // Custom Alert/Toast (replacing window.alert)
        function showMessage(type, message) {
            const toastId = 'app-toast';
            let toast = document.getElementById(toastId);
            if (!toast) {
                toast = document.createElement('div');
                toast.id = toastId;
                toast.className = 'fixed top-4 right-4 z-50 p-4 rounded-lg shadow-xl text-white font-semibold transition-opacity duration-300';
                document.body.appendChild(toast);
            }

            const bgColor = type === 'success' ? 'bg-emerald-600' : type === 'error' ? 'bg-red-600' : 'bg-blue-600';
            
            toast.className = `fixed top-4 right-4 z-50 p-4 rounded-lg shadow-xl text-white font-semibold transition-opacity duration-300 opacity-100 ${bgColor}`;
            toast.innerHTML = message;

            clearTimeout(toast.timeout);
            toast.timeout = setTimeout(() => {
                toast.classList.add('opacity-0');
                // Remove from DOM after transition
                setTimeout(() => toast.remove(), 300); 
            }, 4000);
        }

        // --- FIREBASE INITIALIZATION AND LISTENERS ---

        let app, db, auth;

        async function initFirebase() {
            if (Object.keys(firebaseConfig).length === 0) {
                console.error("Firebase config is missing.");
                updateState({ isAuthReady: true, error: "Database config missing." });
                return;
            }

            setLogLevel('Debug');
            try {
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                updateState({ db, auth }, false);

                // Authentication Listener
                onAuthStateChanged(auth, async (user) => {
                    if (user) {
                        const uid = user.uid;
                        const email = `user-${uid.substring(0, 8)}@osem.com`; // Mock email
                        updateState({ userId: uid, userEmail: email, isAuthReady: true });
                    } else {
                        try {
                            if (initialAuthToken) {
                                await signInWithCustomToken(auth, initialAuthToken);
                            } else {
                                await signInAnonymously(auth);
                            }
                        } catch (e) {
                            console.error("Failed to sign in:", e);
                            updateState({ userId: uuidv4(), isAuthReady: true, error: "Authentication failed." });
                            return;
                        }
                        // Re-trigger listener for new user after sign-in
                    }
                });

                setupFirestoreListeners();
            } catch (e) {
                console.error("Firebase initialization failed:", e);
                updateState({ isAuthReady: true, error: "Failed to connect to the training center database." });
            }
        }
        
        function setupFirestoreListeners() {
            // 1. Courses (Public Data)
            const coursesRef = collection(db, 'artifacts', appId, 'public', 'data', 'courses');
            onSnapshot(coursesRef, (snapshot) => {
                const courseList = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                updateState({ courses: courseList });
                if (courseList.length === 0) seedInitialCourses(coursesRef);
            }, (e) => {
                console.error("Error fetching courses:", e);
                updateState({ error: "Could not load course catalog." });
            });

            // 2. Enrollments (Public Data)
            const enrollmentsRef = collection(db, 'artifacts', appId, 'public', 'data', 'enrollments');
            onSnapshot(enrollmentsRef, (snapshot) => {
                const enrollmentList = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                updateState({ enrollments: enrollmentList });
            }, (e) => {
                console.error("Error fetching enrollments:", e);
                updateState({ error: "Could not load enrollment data." });
            });

            // 3. Certificates (Public Data)
            const certificatesRef = collection(db, 'artifacts', appId, 'public', 'data', 'certificates');
            onSnapshot(certificatesRef, (snapshot) => {
                const certList = snapshot.docs.map(doc => ({ 
                    id: doc.id, 
                    ...doc.data(),
                    issueDate: doc.data().issueDate?.toDate(),
                    expirationDate: doc.data().expirationDate?.toDate(),
                }));
                updateState({ certificates: certList });
            }, (e) => {
                console.error("Error fetching certificates:", e);
                updateState({ error: "Could not load certificate data." });
            });
        }

        async function seedInitialCourses(coursesRef) {
            if (state.courses.length > 0) return; 
            
            console.log("Seeding initial course data...");
            const initialCourses = [
                { name: "Advanced Cardiac Life Support (ACLS)", specialty: "Emergency", provider: "AHA", description: "Comprehensive course on managing cardiopulmonary emergencies.", capacity: 20, currentEnrollment: 5, price: 350.00, durationDays: 2, featured: true, location: "Online", date: "2025-10-15" },
                { name: "Basic Life Support (BLS)", specialty: "Fundamentals", provider: "ARC", description: "Covers CPR, AED use, and relieving choking in adults, children, and infants.", capacity: 30, currentEnrollment: 15, price: 85.00, durationDays: 1, featured: true, location: "In-Person (NY)", date: "2025-11-01" },
                { name: "Pediatric Advanced Life Support (PALS)", specialty: "Pediatrics", provider: "AAP", description: "Focused on improving outcomes for pediatric patients with critical illnesses.", capacity: 15, currentEnrollment: 10, price: 420.00, durationDays: 2, featured: false, location: "Online", date: "2025-10-22" },
                { name: "Phlebotomy Technician Certification", specialty: "Laboratory", provider: "NHA", description: "Training for professional blood collection and processing.", capacity: 25, currentEnrollment: 20, price: 650.00, durationDays: 5, featured: false, location: "In-Person (CA)", date: "2025-12-05" },
            ];

            const batch = writeBatch(db);
            initialCourses.forEach(course => {
                const newDocRef = doc(coursesRef);
                batch.set(newDocRef, course);
            });

            try {
                await batch.commit();
                console.log("Initial courses successfully seeded.");
            } catch (e) {
                console.error("Error seeding data:", e);
            }
        }

        // --- ACTION HANDLERS ---
        
        async function handleEnrollment(courseId, isWaitlist = false) {
            if (!state.db || !state.userId || state.isLoading) {
                showMessage('error', "Authentication or processing in progress.");
                return;
            }
            const course = state.courses.find(c => c.id === courseId);
            if (!course) return showMessage('error', "Course not found.");

            const existingEnrollment = state.enrollments.find(e => e.userId === state.userId && e.courseId === courseId);
            if (existingEnrollment) {
                return showMessage('info', `You are already ${existingEnrollment.status} for this course.`);
            }

            updateState({ isLoading: true, error: null });

            try {
                // 1. MOCK Payment
                const paymentResult = await processStripePayment(course.id, course.price);
                if (!paymentResult.success) {
                    throw new Error("Payment failed. Please try again.");
                }

                // 2. Determine status and update course
                const status = isWaitlist ? 'waitlisted' : 'enrolled';
                const newEnrollmentCount = isWaitlist ? course.currentEnrollment : (course.currentEnrollment + 1);

                const batch = writeBatch(state.db);

                // a) Add new enrollment
                const enrollmentRef = doc(collection(state.db, 'artifacts', appId, 'public', 'data', 'enrollments'));
                batch.set(enrollmentRef, {
                    userId: state.userId,
                    courseId: course.id,
                    enrollmentDate: Timestamp.now(),
                    status: status,
                    email: state.userEmail,
                    transactionId: paymentResult.transactionId,
                });

                // b) Update course enrollment count
                const courseRef = doc(state.db, 'artifacts', appId, 'public', 'data', 'courses', course.id);
                batch.update(courseRef, {
                    currentEnrollment: newEnrollmentCount,
                });

                await batch.commit();
                showMessage('success', `SUCCESS: You have been ${status} in ${course.name}!`);
            } catch (e) {
                console.error("Enrollment failed:", e);
                showMessage('error', e.message || "Enrollment process failed.");
            } finally {
                updateState({ isLoading: false });
            }
        }
        
        async function issueCertificate(enrollment) {
            if (!state.db || state.isLoading) return;
            
            updateState({ isLoading: true, error: null });
            try {
                const course = state.courses.find(c => c.id === enrollment.courseId);
                if (!course) throw new Error("Course not found.");

                const issueDate = new Date();
                const expirationDate = addDays(issueDate, 730); 

                const certData = {
                    userId: enrollment.userId,
                    courseId: enrollment.courseId,
                    courseName: course.name,
                    issueDate: Timestamp.fromDate(issueDate),
                    expirationDate: Timestamp.fromDate(expirationDate),
                    certificateUrl: `https://osem.com/cert/${uuidv4()}`,
                    emailSent: false,
                };

                const batch = writeBatch(state.db);

                // a) Save Certificate
                const certRef = doc(collection(state.db, 'artifacts', appId, 'public', 'data', 'certificates'));
                batch.set(certRef, certData);

                // b) Update Enrollment Status
                const enrollmentRef = doc(state.db, 'artifacts', appId, 'public', 'data', 'enrollments', enrollment.id);
                batch.update(enrollmentRef, { status: 'completed' });

                await batch.commit();

                await sendCertificateEmail(enrollment.email, course.name);
                await updateDoc(certRef, { emailSent: true }); 

                showMessage('success', `Certificate for ${course.name} issued.`);

            } catch (e) {
                console.error("Certificate issuance failed:", e);
                showMessage('error', e.message || "Failed to issue certificate.");
            } finally {
                updateState({ isLoading: false });
            }
        }

        async function sendBulkExpirationReminders() {
            if (!state.db || state.isLoading) return;

            const today = new Date();
            const ninetyDays = addDays(today, 90);
            const expiringCertificates = state.certificates.filter(cert => {
                if (!cert.expirationDate) return false;
                return cert.expirationDate > today && cert.expirationDate <= ninetyDays;
            });
            
            if (expiringCertificates.length === 0) {
                return showMessage('info', "No certificates found expiring in the next 90 days.");
            }

            updateState({ isLoading: true, error: null });
            let sentCount = 0;
            
            try {
                const batch = writeBatch(state.db);

                for (const cert of expiringCertificates) {
                    const course = state.courses.find(c => c.id === cert.courseId);
                    if (course) {
                        await sendExpirationReminder(state.userEmail, course.name); 
                        
                        const certRef = doc(state.db, 'artifacts', appId, 'public', 'data', 'certificates', cert.id);
                        batch.update(certRef, { 
                            lastReminderDate: Timestamp.now(), 
                            reminderSent: true 
                        });
                        sentCount++;
                    }
                }

                await batch.commit();
                showMessage('success', `Successfully sent ${sentCount} expiration reminders.`);

            } catch (e) {
                console.error("Bulk reminder failed:", e);
                showMessage('error', "Failed to send bulk reminders.");
            } finally {
                updateState({ isLoading: false });
            }
        }
        
        async function saveCourse(courseData) {
            if (!state.db || state.isLoading) return;
            updateState({ isLoading: true, error: null });

            try {
                const courseToSave = {
                    ...courseData,
                    capacity: parseInt(courseData.capacity || 20),
                    price: parseFloat(courseData.price || 0),
                    featured: !!courseData.featured,
                };

                if (courseData.id) {
                    // Update existing course
                    const courseRef = doc(state.db, 'artifacts', appId, 'public', 'data', 'courses', courseData.id);
                    await updateDoc(courseRef, courseToSave);
                    showMessage('success', `Course "${courseData.name}" updated successfully.`);
                } else {
                    // Add new course
                    await addDoc(collection(state.db, 'artifacts', appId, 'public', 'data', 'courses'), {
                        ...courseToSave,
                        currentEnrollment: 0,
                    });
                    showMessage('success', `New course "${courseData.name}" created successfully.`);
                }
            } catch (e) {
                console.error("Course save failed:", e);
                showMessage('error', e.message || "Failed to save course.");
            } finally {
                updateState({ isLoading: false, editingCourse: null });
            }
        }

        // --- UI RENDERING FUNCTIONS ---

        function LoadingOverlay() {
            if (!state.isLoading) return '';
            return `
                <div class="fixed inset-0 bg-gray-900 bg-opacity-75 flex items-center justify-center z-50">
                    <div class="flex items-center space-x-2 text-white text-lg">
                        <svg class="animate-spin h-6 w-6 text-yellow-400" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                        </svg>
                        <span>Processing Request...</span>
                    </div>
                </div>
            `;
        }

        function Header() {
            return `
                <header class="bg-white shadow-md sticky top-0 z-40">
                    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-3 flex justify-between items-center">
                        <h1 class="text-2xl font-extrabold text-emerald-600 cursor-pointer" data-action="setView" data-view="home">
                            OSEM Training Center
                        </h1>
                        <nav class="hidden md:flex space-x-6 items-center">
                            <button class="text-gray-600 hover:text-emerald-600 transition duration-150 font-medium" data-action="setView" data-view="catalog">Course Catalog</button>
                            <button class="text-gray-600 hover:text-emerald-600 transition duration-150 font-medium" data-action="setView" data-view="dashboard">My Dashboard</button>
                            <button class="text-red-500 hover:text-red-700 transition duration-150 font-medium" data-action="setView" data-view="admin">Admin</button>
                            <div class="bg-blue-500 text-white text-xs font-bold px-3 py-1 rounded-full">
                                User ID: ${state.userId ? state.userId.substring(0, 8) + '...' : 'Guest'}
                            </div>
                        </nav>
                        <!-- Mobile Menu Icon (Hamburger) -->
                        <button class="md:hidden text-gray-600 hover:text-emerald-600 focus:outline-none">
                            <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16m-7 6h7"></path></svg>
                        </button>
                    </div>
                </header>
            `;
        }
        
        function CourseCard(course) {
            const enrollment = state.enrollments.find(e => e.userId === state.userId && e.courseId === course.id);
            const isFull = course.currentEnrollment >= course.capacity;
            let buttonText = 'Enroll Now';
            if (enrollment) {
                buttonText = 'View Details';
            } else if (isFull) {
                buttonText = 'Join Waitlist';
            }
            
            const statusClass = enrollment 
                ? (enrollment.status === 'enrolled' ? 'bg-emerald-100 text-emerald-800' : enrollment.status === 'completed' ? 'bg-blue-100 text-blue-800' : 'bg-yellow-100 text-yellow-800')
                : '';

            const statusHtml = enrollment 
                ? `<div class="text-sm font-medium p-2 rounded-lg text-center ${statusClass}">Status: ${enrollment.status.toUpperCase()}</div>`
                : '';

            return `
                <div class="bg-white p-6 rounded-xl shadow-lg hover:shadow-xl transition duration-300 flex flex-col justify-between border-t-4 border-emerald-500">
                    <div>
                        <span class="text-sm font-semibold text-gray-500 bg-gray-100 px-3 py-1 rounded-full">${course.specialty}</span>
                        <h3 class="text-2xl font-bold mt-2 text-gray-900">${course.name}</h3>
                        <p class="text-gray-600 mt-1 line-clamp-2">${course.description}</p>
                    </div>
                    <div class="mt-4 pt-4 border-t border-gray-100">
                        <p class="text-lg font-bold text-emerald-600">$${course.price.toFixed(2)}</p>
                        <p class="text-sm text-gray-500 mb-4">Date: ${course.date}</p>
                        ${statusHtml}
                        <button 
                            data-action="viewDetails" data-course-id="${course.id}"
                            class="mt-4 w-full bg-blue-600 text-white py-2 rounded-lg font-semibold hover:bg-blue-700 transition duration-150 shadow-md"
                        >
                            ${buttonText}
                        </button>
                    </div>
                </div>
            `;
        }
        
        // --- VIEW RENDERS ---

        function renderHomePage() {
            const featured = state.courses.filter(c => c.featured).slice(0, 3);
            const featuredHtml = featured.length > 0 ? `
                <div class="py-16">
                    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
                        <h3 class="text-3xl font-bold text-gray-900 mb-8 border-b-2 border-emerald-200 pb-2">Featured Programs</h3>
                        <div class="grid grid-cols-1 md:grid-cols-3 gap-8">
                            ${featured.map(CourseCard).join('')}
                        </div>
                    </div>
                </div>
            ` : '';

            return `
                <div class="min-h-screen bg-gray-50">
                    <div class="bg-emerald-50 pt-16 pb-24 border-b border-emerald-100">
                        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 text-center">
                            <h2 class="text-5xl font-extrabold text-gray-900 sm:text-6xl">
                                Elevate Your Healthcare Career
                            </h2>
                            <p class="mt-4 text-xl text-gray-600 max-w-3xl mx-auto">
                                Trusted certification courses and public health education. Track your credentials and never miss an expiration date.
                            </p>
                            <div class="mt-8">
                                <button 
                                    data-action="setView" data-view="catalog"
                                    class="px-8 py-4 bg-emerald-600 text-white text-lg font-semibold rounded-xl shadow-lg hover:bg-emerald-700 transition duration-300 transform hover:scale-105"
                                >
                                    View All Courses
                                </button>
                            </div>
                        </div>
                    </div>
                    ${featuredHtml}
                    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12">
                        <div class="bg-blue-600 p-8 rounded-2xl shadow-xl text-white flex flex-col md:flex-row justify-between items-center">
                            <div>
                                <h4 class="text-3xl font-bold">New Year, New Certifications!</h4>
                                <p class="mt-2 text-blue-100">Use code OSEM20 for 20% off all online courses this month.</p>
                            </div>
                            <button class="mt-4 md:mt-0 bg-white text-blue-600 font-semibold px-6 py-3 rounded-xl hover:bg-gray-100 transition shadow-lg">
                                Explore Deals
                            </button>
                        </div>
                    </div>
                </div>
            `;
        }

        function renderCourseCatalog() {
            const specialties = [...new Set(state.courses.map(c => c.specialty))];
            const locations = [...new Set(state.courses.map(c => c.location))];

            const filteredCourses = state.courses.filter(course => {
                const searchMatch = course.name.toLowerCase().includes(state.searchTerm.toLowerCase()) ||
                                    course.description.toLowerCase().includes(state.searchTerm.toLowerCase());
                const specialtyMatch = state.filterSpecialty === 'All' || course.specialty === state.filterSpecialty;
                const locationMatch = state.filterLocation === 'All' || course.location === state.filterLocation;

                return searchMatch && specialtyMatch && locationMatch;
            });
            
            const coursesHtml = filteredCourses.length > 0 ? filteredCourses.map(CourseCard).join('') : `
                <p class="col-span-full text-center text-gray-500">No courses match your criteria.</p>
            `;

            return `
                <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12 min-h-screen">
                    <h2 class="text-4xl font-bold text-gray-900 mb-8">Full Course Catalog</h2>

                    <div class="bg-white p-6 rounded-xl shadow-lg mb-8 grid grid-cols-1 md:grid-cols-4 gap-4">
                        <input
                            type="text"
                            placeholder="Search by title or keyword..."
                            value="${state.searchTerm}"
                            data-action="setSearchTerm"
                            class="col-span-1 md:col-span-2 p-3 border border-gray-300 rounded-lg focus:ring-emerald-500 focus:border-emerald-500"
                        />
                        <select
                            data-action="setFilterSpecialty"
                            class="p-3 border border-gray-300 rounded-lg focus:ring-emerald-500 focus:border-emerald-500"
                        >
                            <option value="All">All Specialties</option>
                            ${specialties.map(s => `<option value="${s}" ${state.filterSpecialty === s ? 'selected' : ''}>${s}</option>`).join('')}
                        </select>
                        <select
                            data-action="setFilterLocation"
                            class="p-3 border border-gray-300 rounded-lg focus:ring-emerald-500 focus:border-emerald-500"
                        >
                            <option value="All">All Locations</option>
                            ${locations.map(l => `<option value="${l}" ${state.filterLocation === l ? 'selected' : ''}>${l}</option>`).join('')}
                        </select>
                    </div>

                    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8" id="course-cards-container">
                        ${coursesHtml}
                    </div>
                </div>
            `;
        }
        
        function renderCourseDetails() {
            const course = state.selectedCourse;
            if (!course) return `<p class="p-8 text-center">Course not found. Click <button class="text-blue-600" data-action="setView" data-view="catalog">here</button> to go to the catalog.</p>`;

            const enrollment = state.enrollments.find(e => e.userId === state.userId && e.courseId === course.id);
            const isFull = course.currentEnrollment >= course.capacity;
            const remainingSeats = course.capacity - course.currentEnrollment;
            
            let actionButton;
            if (enrollment) {
                actionButton = `
                    <div class="p-4 bg-blue-100 text-blue-800 rounded-lg text-center font-semibold">
                        You are already ${enrollment.status.toUpperCase()} in this course.
                    </div>
                `;
            } else if (isFull) {
                actionButton = `
                    <button 
                        data-action="enroll" data-course-id="${course.id}" data-is-waitlist="true"
                        ${state.isLoading ? 'disabled' : ''}
                        class="w-full bg-yellow-500 text-white py-3 rounded-lg text-lg font-semibold hover:bg-yellow-600 transition disabled:bg-gray-400 shadow-lg"
                    >
                        ${state.isLoading ? 'Processing...' : 'Join Waitlist (Course Full)'}
                    </button>
                `;
            } else {
                actionButton = `
                    <button 
                        data-action="enroll" data-course-id="${course.id}" data-is-waitlist="false"
                        ${state.isLoading ? 'disabled' : ''}
                        class="w-full bg-emerald-600 text-white py-3 rounded-lg text-lg font-semibold hover:bg-emerald-700 transition disabled:bg-gray-400 shadow-lg"
                    >
                        ${state.isLoading ? 'Processing Payment...' : 'Enroll Now - $' + course.price.toFixed(2) + ` (${remainingSeats} seats left)`}
                    </button>
                `;
            }
            
            const errorHtml = state.error ? `<p class="text-red-500 mt-3 text-center">${state.error}</p>` : '';

            return `
                <div class="max-w-4xl mx-auto px-4 sm:px-6 lg:px-8 py-12 min-h-screen">
                    <button 
                        data-action="setView" data-view="catalog"
                        class="flex items-center text-emerald-600 hover:text-emerald-800 mb-6 font-medium"
                    >
                        <svg class="w-5 h-5 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 19l-7-7m0 0l7-7m-7 7h18"></path></svg>
                        Back to Catalog
                    </button>
                    <div class="bg-white p-8 rounded-2xl shadow-xl">
                        <span class="text-md font-semibold text-white bg-emerald-500 px-4 py-1 rounded-full">${course.specialty}</span>
                        <h2 class="text-4xl font-extrabold text-gray-900 mt-3">${course.name}</h2>
                        <p class="text-xl text-gray-600 mt-2">Provided by ${course.provider}</p>

                        <div class="mt-6 grid grid-cols-2 gap-4 text-gray-700 border-t pt-4">
                            <div>
                                <p class="font-semibold">Price:</p>
                                <p class="text-2xl text-emerald-600 font-bold">$${course.price.toFixed(2)}</p>
                            </div>
                            <div>
                                <p class="font-semibold">Schedule & Location:</p>
                                <p>${course.date} in ${course.location}</p>
                            </div>
                            <div>
                                <p class="font-semibold">Duration:</p>
                                <p>${course.durationDays} Days</p>
                            </div>
                            <div>
                                <p class="font-semibold">Capacity:</p>
                                <p>${course.currentEnrollment} / ${course.capacity} enrolled</p>
                            </div>
                        </div>

                        <div class="mt-6">
                            <h3 class="text-2xl font-bold text-gray-900 mb-2">Course Description</h3>
                            <p class="text-gray-700 leading-relaxed">${course.description}</p>
                        </div>

                        <div class="mt-8 pt-6 border-t border-gray-200">
                            ${actionButton}
                            ${errorHtml}
                        </div>
                    </div>
                </div>
            `;
        }

        function renderUserDashboard() {
            const userEnrollments = state.enrollments.filter(e => e.userId === state.userId);
            const userCertificates = state.certificates.filter(c => c.userId === state.userId);
            
            const enrollmentsHtml = userEnrollments.length > 0 ? userEnrollments.map(e => {
                const course = state.courses.find(c => c.id === e.courseId);
                if (!course) return '';
                const statusClass = e.status === 'enrolled' ? 'text-blue-600' : e.status === 'completed' ? 'text-emerald-600' : 'text-yellow-600';
                return `
                    <div class="p-4 bg-gray-100 rounded-lg flex justify-between items-center shadow-sm">
                        <div>
                            <p class="text-lg font-semibold text-gray-800">${course.name}</p>
                            <p class="text-sm font-medium ${statusClass}">
                                Status: ${e.status.toUpperCase()}
                            </p>
                        </div>
                        <button 
                            data-action="viewDetails" data-course-id="${course.id}"
                            class="text-sm bg-white border border-gray-300 px-4 py-2 rounded-lg hover:bg-gray-50"
                        >
                            View Course
                        </button>
                    </div>
                `;
            }).join('') : `
                <p class="text-gray-500">You are not currently enrolled in any courses. Check the <button class="text-emerald-600 hover:underline" data-action="setView" data-view="catalog">Catalog</button>!</p>
            `;

            const certificatesHtml = userCertificates.length > 0 ? userCertificates.map(c => {
                const isExpired = c.expirationDate < new Date();
                const expirationText = c.expirationDate ? c.expirationDate.toLocaleDateString() : 'N/A';
                const statusClass = isExpired ? 'bg-red-100 text-red-800' : 'bg-emerald-100 text-emerald-800';
                
                return `
                    <div class="p-4 bg-white rounded-lg shadow-md flex justify-between items-center">
                        <div>
                            <p class="text-lg font-semibold text-gray-800">${c.courseName}</p>
                            <p class="text-sm text-gray-500">Issued: ${c.issueDate?.toLocaleDateString()}</p>
                        </div>
                        <div class="text-right">
                            <p class="text-sm font-bold p-1 rounded ${statusClass}">
                                ${isExpired ? 'EXPIRED' : `Expires: ${expirationText}`}
                            </p>
                            <a 
                                href="${c.certificateUrl}" target="_blank" rel="noopener noreferrer"
                                class="mt-2 text-sm text-blue-600 hover:underline inline-block"
                            >
                                Download Certificate (MOCK)
                            </a>
                        </div>
                    </div>
                `;
            }).join('') : `
                <p class="text-gray-500">You have no completed certifications yet.</p>
            `;

            return `
                <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12 min-h-screen">
                    <h2 class="text-4xl font-bold text-gray-900 mb-8">My Professional Dashboard</h2>
                    <div class="bg-white p-6 rounded-xl shadow-lg mb-8">
                        <p class="text-xl font-semibold text-gray-700">User ID: <span class="text-emerald-600 text-sm break-all">${state.userId}</span></p>
                        <p class="text-lg text-gray-500">Mock Email: ${state.userEmail}</p>
                    </div>

                    <div class="mb-12">
                        <h3 class="text-2xl font-bold text-gray-900 mb-4 border-b pb-2">My Enrollments</h3>
                        <div class="space-y-4">
                            ${enrollmentsHtml}
                        </div>
                    </div>

                    <div>
                        <h3 class="text-2xl font-bold text-gray-900 mb-4 border-b pb-2">My Certificates</h3>
                        <div class="space-y-4">
                            ${certificatesHtml}
                        </div>
                    </div>
                </div>
            `;
        }

        function renderAdminDashboard() {
            const expiringCertificates = state.certificates.filter(cert => {
                const today = new Date();
                const ninetyDays = addDays(today, 90);
                if (!cert.expirationDate) return false;
                return cert.expirationDate > today && cert.expirationDate <= ninetyDays;
            });
            
            const renderCourseManagement = () => {
                const coursesHtml = state.courses.map(course => `
                    <tr key="${course.id}">
                        <td class="px-6 py-4 whitespace-nowrap">
                            <div class="text-sm font-medium text-gray-900">${course.name}</div>
                            <div class="text-sm text-gray-500">${course.specialty}</div>
                        </td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                            ${course.date} (${course.location})
                        </td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm">
                            <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${course.currentEnrollment >= course.capacity ? 'bg-red-100 text-red-800' : 'bg-blue-100 text-blue-800'}">
                                ${course.currentEnrollment} / ${course.capacity}
                            </span>
                        </td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm font-medium">
                            <button data-action="editCourse" data-course-id="${course.id}" class="text-indigo-600 hover:text-indigo-900 mr-3">Edit</button>
                            <button data-action="issueCert" data-course-id="${course.id}" class="text-green-600 hover:text-green-900">Issue Cert (1st Enrolled)</button>
                        </td>
                    </tr>
                `).join('');

                return `
                    <div class="mt-8">
                        <div class="flex justify-between items-center mb-6">
                            <h3 class="text-2xl font-bold text-gray-900">Manage Courses</h3>
                            <button 
                                data-action="createCourse"
                                class="bg-emerald-600 text-white px-4 py-2 rounded-lg hover:bg-emerald-700 transition flex items-center shadow-md"
                            >
                                <svg class="w-5 h-5 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6v6m0 0v6m0-6h6m-6 0H6"></path></svg>
                                Add New Course
                            </button>
                        </div>

                        <div class="bg-white rounded-xl shadow-lg overflow-x-auto">
                            <table class="min-w-full divide-y divide-gray-200">
                                <thead class="bg-gray-50">
                                    <tr>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Course</th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Date/Location</th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Enrollment</th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
                                    </tr>
                                </thead>
                                <tbody class="bg-white divide-y divide-gray-200">
                                    ${coursesHtml}
                                </tbody>
                            </table>
                        </div>
                    </div>
                `;
            };

            const renderCertificateManagement = () => {
                const certsHtml = expiringCertificates.length > 0 ? expiringCertificates.map(cert => `
                    <tr key="${cert.id}" class="hover:bg-red-50">
                        <td class="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">${cert.courseName}</td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">${cert.userId.substring(0, 8)}...</td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm text-red-600 font-semibold">
                            ${cert.expirationDate.toLocaleDateString()}
                        </td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm">
                            <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${cert.expirationDate < addDays(new Date(), 30) ? 'bg-red-100 text-red-800' : 'bg-yellow-100 text-yellow-800'}">
                                Expiring Soon
                            </span>
                        </td>
                    </tr>
                `).join('') : `
                    <tr>
                        <td colSpan="4" class="px-6 py-4 text-center text-gray-500">No certificates are expiring in the next 90 days.</td>
                    </tr>
                `;

                return `
                    <div class="mt-8">
                        <div class="flex justify-between items-center mb-6">
                            <h3 class="text-2xl font-bold text-gray-900">Expiring Certificates (${expiringCertificates.length})</h3>
                            <button 
                                data-action="sendBulkReminders"
                                ${state.isLoading || expiringCertificates.length === 0 ? 'disabled' : ''}
                                class="bg-red-500 text-white px-4 py-2 rounded-lg hover:bg-red-600 transition flex items-center shadow-md disabled:bg-gray-400"
                            >
                                <svg class="w-5 h-5 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v2a3 3 0 11-6 0v-2m6 0H9"></path></svg>
                                ${state.isLoading ? 'Sending...' : 'Send Bulk Reminders'}
                            </button>
                        </div>

                        <div class="bg-white rounded-xl shadow-lg overflow-x-auto">
                            <table class="min-w-full divide-y divide-gray-200">
                                <thead class="bg-gray-50">
                                    <tr>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Certificate</th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">User ID</th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Expiration Date</th>
                                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Status</th>
                                    </tr>
                                </thead>
                                <tbody class="bg-white divide-y divide-gray-200">
                                    ${certsHtml}
                                </tbody>
                            </table>
                        </div>
                    </div>
                `;
            };

            const adminContent = state.adminView === 'courses' ? renderCourseManagement() : renderCertificateManagement();

            return `
                <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12 min-h-screen">
                    <h2 class="text-4xl font-bold text-gray-900 mb-8">Admin Center</h2>
                    
                    <div class="border-b border-gray-200 mb-6">
                        <nav class="-mb-px flex space-x-8" aria-label="Tabs">
                            <button
                                data-action="setAdminView" data-admin-view="courses"
                                class="${state.adminView === 'courses' ? 'border-emerald-500 text-emerald-600' : 'border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300'} whitespace-nowrap py-3 px-1 border-b-2 font-medium text-sm transition"
                            >
                                Course & Enrollment Management
                            </button>
                            <button
                                data-action="setAdminView" data-admin-view="certs"
                                class="${state.adminView === 'certs' ? 'border-emerald-500 text-emerald-600' : 'border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300'} whitespace-nowrap py-3 px-1 border-b-2 font-medium text-sm transition"
                            >
                                Certificate Expiration Tracking
                            </button>
                        </nav>
                    </div>

                    ${adminContent}

                    ${state.error ? `<div class="mt-6 p-4 bg-red-100 text-red-700 rounded-lg">${state.error}</div>` : ''}
                </div>
            `;
        }
        
        function CourseForm(initialData) {
            const data = initialData || {
                name: '', specialty: '', provider: '', description: '', capacity: 20, price: 0.00, date: new Date().toISOString().split('T')[0], location: 'Online', durationDays: 1, featured: false
            };
            const isEditing = !!initialData?.id;

            return `
                <div class="fixed inset-0 bg-gray-900 bg-opacity-80 flex items-center justify-center z-50 p-4">
                    <form id="courseForm" class="bg-white p-8 rounded-xl shadow-2xl w-full max-w-2xl max-h-[90vh] overflow-y-auto">
                        <h3 class="text-2xl font-bold mb-6 text-gray-900">${isEditing ? 'Edit Course' : 'Create New Course'}</h3>
                        
                        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                            <label class="block">
                                <span class="text-gray-700">Course Name</span>
                                <input type="text" name="name" value="${data.name}" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2 border" />
                            </label>
                            <label class="block">
                                <span class="text-gray-700">Specialty</span>
                                <input type="text" name="specialty" value="${data.specialty}" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2 border" />
                            </label>
                            <label class="block">
                                <span class="text-gray-700">Provider</span>
                                <input type="text" name="provider" value="${data.provider}" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2 border" />
                            </label>
                            <label class="block">
                                <span class="text-gray-700">Date</span>
                                <input type="date" name="date" value="${data.date}" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2 border" />
                            </label>
                            <label class="block">
                                <span class="text-gray-700">Location</span>
                                <input type="text" name="location" value="${data.location}" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2 border" />
                            </label>
                            <label class="block">
                                <span class="text-gray-700">Capacity</span>
                                <input type="number" name="capacity" value="${data.capacity}" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2 border" />
                            </label>
                            <label class="block">
                                <span class="text-gray-700">Price ($)</span>
                                <input type="number" name="price" value="${data.price.toFixed(2)}" required step="0.01" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2 border" />
                            </label>
                            <label class="flex items-center space-x-2 mt-6">
                                <input type="checkbox" name="featured" ${data.featured ? 'checked' : ''} class="rounded border-gray-300 text-emerald-600 shadow-sm" />
                                <span class="text-gray-700">Feature on Homepage</span>
                            </label>
                        </div>
                        <label class="block mt-4">
                            <span class="text-gray-700">Description</span>
                            <textarea name="description" required rows="3" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2 border">${data.description}</textarea>
                        </label>

                        <div class="mt-6 flex justify-end space-x-3">
                            <button type="button" data-action="cancelEdit" class="px-4 py-2 bg-gray-200 text-gray-700 rounded-lg hover:bg-gray-300 transition">Cancel</button>
                            <button type="submit" ${state.isLoading ? 'disabled' : ''} class="px-4 py-2 bg-emerald-600 text-white rounded-lg hover:bg-emerald-700 transition disabled:bg-gray-400">
                                ${state.isLoading ? 'Saving...' : (isEditing ? 'Save Changes' : 'Create Course')}
                            </button>
                        </div>
                    </form>
                </div>
            `;
        }

        // --- MAIN RENDER FUNCTION ---

        function render() {
            if (!state.isAuthReady) {
                appRoot.innerHTML = `<div class="flex justify-center items-center h-screen text-xl text-gray-600">Connecting to OSEM Database...</div>`;
                return;
            }

            let content;
            switch (state.view) {
                case 'catalog':
                    content = renderCourseCatalog();
                    break;
                case 'details':
                    content = renderCourseDetails();
                    break;
                case 'dashboard':
                    content = renderUserDashboard();
                    break;
                case 'admin':
                    content = renderAdminDashboard();
                    break;
                case 'home':
                default:
                    content = renderHomePage();
                    break;
            }
            
            let modalHtml = state.editingCourse ? CourseForm(state.editingCourse) : '';

            // Main layout update
            appRoot.innerHTML = `
                ${Header()}
                <main>
                    ${content}
                </main>
                ${LoadingOverlay()}
                ${modalHtml}
            `;
            
            // Re-attach event listeners after rendering (Delegation is key)
            attachEventListeners();
        }

        // --- EVENT DELEGATION ---

        function attachEventListeners() {
            appRoot.onclick = (e) => {
                const target = e.target.closest('[data-action]');
                if (!target) return;
                
                const action = target.getAttribute('data-action');
                
                if (action === 'setView') {
                    updateState({ view: target.getAttribute('data-view') });
                } else if (action === 'viewDetails') {
                    const courseId = target.getAttribute('data-course-id');
                    const course = state.courses.find(c => c.id === courseId);
                    updateState({ selectedCourse: course, view: 'details' });
                } else if (action === 'enroll') {
                    const courseId = target.getAttribute('data-course-id');
                    const isWaitlist = target.getAttribute('data-is-waitlist') === 'true';
                    handleEnrollment(courseId, isWaitlist);
                } else if (action === 'setAdminView') {
                    updateState({ adminView: target.getAttribute('data-admin-view') });
                } else if (action === 'sendBulkReminders') {
                    sendBulkExpirationReminders();
                } else if (action === 'createCourse') {
                    updateState({ editingCourse: {} });
                } else if (action === 'editCourse') {
                    const courseId = target.getAttribute('data-course-id');
                    const course = state.courses.find(c => c.id === courseId);
                    updateState({ editingCourse: course });
                } else if (action === 'cancelEdit') {
                    updateState({ editingCourse: null });
                } else if (action === 'issueCert') {
                    const courseId = target.getAttribute('data-course-id');
                    const enrollment = state.enrollments.find(e => e.courseId === courseId && e.status === 'enrolled');
                    if (enrollment) {
                         issueCertificate(enrollment);
                    } else {
                         showMessage('info', "No enrolled user found for this course to issue a certificate.");
                    }
                }
            };
            
            appRoot.onchange = (e) => {
                const target = e.target;
                const action = target.getAttribute('data-action');
                
                if (state.view === 'catalog') {
                    if (action === 'setSearchTerm') {
                        updateState({ searchTerm: target.value });
                    } else if (action === 'setFilterSpecialty') {
                        updateState({ filterSpecialty: target.value });
                    } else if (action === 'setFilterLocation') {
                        updateState({ filterLocation: target.value });
                    }
                }
            };

            appRoot.onsubmit = (e) => {
                if (e.target.id === 'courseForm') {
                    e.preventDefault();
                    
                    const form = e.target;
                    const formData = new FormData(form);
                    const courseData = { id: state.editingCourse?.id };

                    for (const [key, value] of formData.entries()) {
                        if (key === 'featured') {
                            courseData[key] = true;
                        } else {
                            courseData[key] = value;
                        }
                    }
                    if (!courseData.hasOwnProperty('featured')) {
                        courseData.featured = false;
                    }
                    
                    saveCourse(courseData);
                }
            }
        }

        // Start the application
        window.onload = initFirebase;

    </script>
</body>
</html>
