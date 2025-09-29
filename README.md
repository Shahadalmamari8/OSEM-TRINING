import React, { useState, useEffect, useMemo } from 'react';
import { Home, Calendar, Users, Briefcase, BookOpen, Menu, X, DollarSign, Clock, MapPin, CheckCircle, BarChart2, Edit, Save } from 'lucide-react';

// --- Firebase Imports ---
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, addDoc, collection, query, onSnapshot, serverTimestamp } from 'firebase/firestore';

// --- Placeholder Data & Mocks (This data will ideally be moved to Firestore as well) ---

const calendarEvents = [
  { id: 1, date: 'Oct 15', day: 'Mon', title: 'Python Fundamentals Start', time: 'Online', color: 'bg-indigo-500' },
  { id: 2, date: 'Oct 18', day: 'Thu', title: 'Data Science Info Session', time: '7:00 PM EST', color: 'bg-green-500' },
];

const instructorsData = [
  { id: 1, name: 'Dr. Evelyn Reed', title: 'Lead Data Science Instructor', description: 'Specializes in scalable machine learning and cloud infrastructure.', imageUrl: 'https://placehold.co/100x100/10b981/ffffff?text=ER', expertise: ['Machine Learning', 'Python'] },
  { id: 2, name: 'Mr. Kenji Tanaka', title: 'Full Stack Development Specialist', description: 'An expert in modern web frameworks like React and Node.js.', imageUrl: 'https://placehold.co/100x100/3b82f6/ffffff?text=KT', expertise: ['React', 'Node.js', 'DevOps'] },
];

const coursesData = [
    { id: 'c1', name: 'Advanced Python for Data', duration: '8 Weeks', price: 999, location: 'Online', specialty: 'Data Science' },
    { id: 'c2', name: 'Modern React & Tailwind CSS', duration: '6 Weeks', price: 799, location: 'Online', specialty: 'Web Dev' },
    { id: 'c3', name: 'User Experience (UX) Mastery', duration: '10 Weeks', price: 1200, location: 'In-Person (NY)', specialty: 'Design' },
    { id: 'c4', name: 'Cloud Infrastructure with AWS', duration: '4 Weeks', price: 599, location: 'Online', specialty: 'DevOps' },
];

// --- Shared Components ---

const EnrollmentFormModal = ({ course, onClose, onEnrollSubmit }) => {
  const [formData, setFormData] = useState({
    name: '', age: '', idNumber: '', placeOfWork: '', specialization: ''
  });
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [isSubmitted, setIsSubmitted] = useState(false);

  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsSubmitting(true);
    
    // Call the real submit handler which includes Firestore logic
    await onEnrollSubmit(course.id, formData);

    setIsSubmitting(false);
    setIsSubmitted(true);
    
    // Close after a delay
    setTimeout(onClose, 3000); 
  };

  return (
    <div className="fixed inset-0 bg-gray-900 bg-opacity-80 flex items-center justify-center z-50 p-4">
      <div className="bg-white p-6 sm:p-8 rounded-xl shadow-2xl w-full max-w-lg max-h-[90vh] overflow-y-auto">
        
        <div className="flex justify-between items-center mb-6 border-b pb-3">
          <h3 className="text-2xl font-bold text-indigo-700">Enrollment Form</h3>
          <button onClick={onClose} className="text-gray-500 hover:text-gray-800 transition">
            <X className="w-6 h-6" />
          </button>
        </div>

        <p className="mb-4 text-gray-600">Registering for: <span className="font-semibold text-indigo-600">{course.name}</span></p>

        {isSubmitted ? (
            <div className="text-center py-10">
                <CheckCircle className="w-16 h-16 mx-auto text-green-500 mb-4" />
                <h4 className="text-xl font-bold text-gray-800">Registration Confirmed!</h4>
                <p className="text-gray-600 mt-2">We have received your details and will contact you shortly.</p>
            </div>
        ) : (
            <form onSubmit={handleSubmit} className="space-y-4">
            
            <label className="block">
                <span className="text-gray-700 font-medium">Full Name</span>
                <input type="text" name="name" value={formData.name} onChange={handleChange} required className="mt-1 block w-full rounded-lg border-gray-300 shadow-sm p-3 border focus:ring-indigo-500 focus:border-indigo-500" />
            </label>
            <div className="grid grid-cols-2 gap-4">
                <label className="block">
                    <span className="text-gray-700 font-medium">Age</span>
                    <input type="number" name="age" value={formData.age} onChange={handleChange} required className="mt-1 block w-full rounded-lg border-gray-300 shadow-sm p-3 border focus:ring-indigo-500 focus:border-indigo-500" />
                </label>
                <label className="block">
                    <span className="text-gray-700 font-medium">ID Number</span>
                    <input type="text" name="idNumber" value={formData.idNumber} onChange={handleChange} required className="mt-1 block w-full rounded-lg border-gray-300 shadow-sm p-3 border focus:ring-indigo-500 focus:border-indigo-500" />
                </label>
            </div>
            <label className="block">
                <span className="text-gray-700 font-medium">Place of Work</span>
                <input type="text" name="placeOfWork" value={formData.placeOfWork} onChange={handleChange} required className="mt-1 block w-full rounded-lg border-gray-300 shadow-sm p-3 border focus:ring-indigo-500 focus:border-indigo-500" />
            </label>
            <label className="block">
                <span className="text-gray-700 font-medium">Specialization / Role</span>
                <input type="text" name="specialization" value={formData.specialization} onChange={handleChange} required className="mt-1 block w-full rounded-lg border-gray-300 shadow-sm p-3 border focus:ring-indigo-500 focus:border-indigo-500" />
            </label>

            <div className="pt-4">
                <button 
                    type="submit" 
                    disabled={isSubmitting}
                    className="w-full bg-indigo-600 text-white py-3 rounded-lg font-semibold hover:bg-indigo-700 transition duration-150 shadow-md disabled:bg-gray-400 disabled:cursor-not-allowed"
                >
                    {isSubmitting ? 'Processing...' : `Complete Registration for $${course.price}`}
                </button>
            </div>
            </form>
        )}
      </div>
    </div>
  );
};

// --- General Site Components (Calendar, Catalog, Instructors) ---
// (Unchanged from previous version, using placeholder data)

const CalendarSection = () => (
    <div className="bg-white p-6 rounded-xl shadow-lg border border-gray-100">
      <div className="flex items-center text-xl font-bold mb-4 text-gray-900">
        <Calendar className="w-6 h-6 mr-3 text-indigo-600" />
        Upcoming Course Calendar
      </div>
      <div className="space-y-4">
        {calendarEvents.map(event => (
          <div key={event.id} className="flex items-start space-x-4 p-3 rounded-lg hover:bg-gray-50 transition border-b last:border-b-0">
            <div className={`flex-shrink-0 w-12 h-12 flex flex-col items-center justify-center rounded-lg text-white ${event.color} shadow-md`}>
              <span className="text-xs font-light">{event.day}</span>
              <span className="text-lg font-bold -mt-1">{event.date.split(' ')[1]}</span>
            </div>
            <div className="flex-grow">
              <p className="text-base font-semibold text-gray-900">{event.title}</p>
              <p className="text-sm text-gray-500">{event.time}</p>
            </div>
          </div>
        ))}
      </div>
    </div>
);

const CourseCatalogSection = ({ onEnrollClick }) => (
    <div className="bg-white p-6 rounded-xl shadow-lg border border-gray-100">
        <div className="flex items-center text-xl font-bold mb-6 text-gray-900 border-b pb-2">
            <BookOpen className="w-6 h-6 mr-3 text-indigo-600" />
            Available Courses for Registration
        </div>
        <div className="space-y-6">
            {coursesData.map(course => (
                <div key={course.id} className="p-4 bg-gray-50 rounded-lg shadow-sm flex flex-col sm:flex-row justify-between items-start sm:items-center border border-gray-200">
                    <div className="mb-3 sm:mb-0">
                        <p className="text-lg font-bold text-gray-800">{course.name}</p>
                        <div className="flex space-x-4 text-sm text-gray-600 mt-1">
                            <span className="flex items-center"><Clock className="w-4 h-4 mr-1 text-indigo-500" /> {course.duration}</span>
                            <span className="flex items-center"><MapPin className="w-4 h-4 mr-1 text-indigo-500" /> {course.location}</span>
                        </div>
                    </div>
                    <div className="flex items-center space-x-4">
                        <span className="text-xl font-extrabold text-green-600"><DollarSign className="w-5 h-5 inline-block -mt-1" />{course.price}</span>
                        <button 
                            onClick={() => onEnrollClick(course)}
                            className="bg-indigo-600 text-white px-5 py-2 rounded-lg font-semibold hover:bg-indigo-700 transition duration-150 shadow-md"
                        >
                            Enroll Now
                        </button>
                    </div>
                </div>
            ))}
        </div>
    </div>
);

const InstructorsSection = () => (
    <div className="bg-white p-6 rounded-xl shadow-lg border border-gray-100">
        <div className="flex items-center text-xl font-bold mb-6 text-gray-900 border-b pb-2">
            <Users className="w-6 h-6 mr-3 text-indigo-600" />
            Expert Instructors
        </div>
        <div className="grid grid-cols-1 sm:grid-cols-2 gap-6">
            {instructorsData.slice(0, 2).map(instructor => (
                <div key={instructor.id} className="flex items-center p-3 space-x-4 border border-gray-200 rounded-lg bg-gray-50">
                    <img
                        src={instructor.imageUrl}
                        alt={`Profile of ${instructor.name}`}
                        className="w-16 h-16 rounded-full object-cover border-2 border-indigo-300"
                        onError={(e) => { e.target.onerror = null; e.target.src = 'https://placehold.co/64x64/6366f1/ffffff?text=NA'; }}
                    />
                    <div>
                        <h4 className="text-base font-bold text-gray-900">{instructor.name}</h4>
                        <p className="text-sm font-medium text-indigo-600">{instructor.title}</p>
                        <div className="flex flex-wrap gap-1 mt-1">
                            {instructor.expertise.map((skill, index) => (
                                <span key={index} className="px-2 py-0.5 text-xs bg-indigo-100 text-indigo-800 rounded-full">
                                  {skill}
                                </span>
                            ))}
                        </div>
                    </div>
                </div>
            ))}
        </div>
        <p className="mt-4 text-center">
            <button 
                className="text-indigo-600 hover:text-indigo-800 font-medium text-sm transition"
            >
                View All Instructors &rarr;
            </button>
        </p>
    </div>
);

const GeneralHomeView = ({ onEnrollClick }) => (
  <div className="space-y-10">
    {/* Hero Section */}
    <div className="bg-indigo-700 text-white p-8 sm:p-12 rounded-2xl shadow-xl text-center">
        <h2 className="text-4xl sm:text-5xl font-extrabold mb-3">Your Path to Expert Certification</h2>
        <p className="text-xl text-indigo-200 max-w-3xl mx-auto">Discover professional training, track your progress, and connect with top industry mentors.</p>
    </div>

    {/* Content Grid */}
    <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
        <div className="lg:col-span-2 space-y-8">
            <CourseCatalogSection onEnrollClick={onEnrollClick} />
            <CalendarSection />
        </div>
        <div className="lg:col-span-1">
            <InstructorsSection />
        </div>
    </div>
  </div>
);

// --- Admin Components ---

const AdminManagementForm = ({ title, placeholder, onSave }) => {
    const [content, setContent] = useState(placeholder);
    const [status, setStatus] = useState(null);

    const handleSave = () => {
        // Mock save for placeholder data
        console.log(`Saving ${title}:`, content);
        setStatus('Saved!');
        setTimeout(() => setStatus(null), 2000);
        onSave(content); // Call parent handler if needed
    };

    return (
        <div className="p-4 border border-indigo-200 bg-indigo-50 rounded-lg">
            <h4 className="font-semibold text-indigo-700 mb-2">{title}</h4>
            <textarea
                value={content}
                onChange={(e) => setContent(e.target.value)}
                rows="5"
                className="w-full p-2 border rounded-lg resize-none focus:ring-indigo-500 focus:border-indigo-500"
            />
            <div className="flex justify-between items-center mt-3">
                <button
                    onClick={handleSave}
                    className="flex items-center bg-indigo-600 text-white px-4 py-2 rounded-lg text-sm hover:bg-indigo-700 transition"
                >
                    <Save className="w-4 h-4 mr-2" /> Save Changes
                </button>
                {status && <span className="text-green-600 font-medium text-sm">{status}</span>}
            </div>
        </div>
    );
};

const AdminDataAnalysis = ({ registrations }) => {
    const totalRegistrations = registrations.length;
    
    // Simple analysis: Registrations by Specialization
    const analysis = useMemo(() => {
        return registrations.reduce((acc, reg) => {
            const spec = reg.specialization || 'Unknown';
            acc[spec] = (acc[spec] || 0) + 1;
            return acc;
        }, {});
    }, [registrations]);

    const registrationTable = registrations.slice().sort((a, b) => b.registeredAt?.toDate() - a.registeredAt?.toDate());

    return (
        <div className="space-y-8">
            <h3 className="text-2xl font-bold text-gray-900 flex items-center mb-4">
                <BarChart2 className="w-6 h-6 mr-2 text-indigo-600" /> Analytics Summary
            </h3>
            
            <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                <div className="p-5 bg-green-500 text-white rounded-xl shadow-lg">
                    <p className="text-sm font-medium opacity-80">Total Enrollments</p>
                    <p className="text-3xl font-extrabold mt-1">{totalRegistrations}</p>
                </div>
                <div className="p-5 bg-blue-500 text-white rounded-xl shadow-lg">
                    <p className="text-sm font-medium opacity-80">Courses Offered</p>
                    <p className="text-3xl font-extrabold mt-1">{coursesData.length}</p>
                </div>
                <div className="p-5 bg-yellow-500 text-white rounded-xl shadow-lg">
                    <p className="text-sm font-medium opacity-80">Top Specialization</p>
                    <p className="text-3xl font-extrabold mt-1">
                        {Object.keys(analysis).length > 0 ? 
                            Object.keys(analysis).reduce((a, b) => analysis[a] > analysis[b] ? a : b) : 'N/A'}
                    </p>
                </div>
            </div>

            <div className="bg-white p-6 rounded-xl shadow-lg">
                <h4 className="text-xl font-bold mb-4">Recent Registrations (Live Feed)</h4>
                <div className="overflow-x-auto">
                    <table className="min-w-full divide-y divide-gray-200">
                        <thead className="bg-gray-50">
                            <tr>
                                {['Date', 'Name', 'Course', 'Specialization', 'Status'].map(header => (
                                    <th key={header} className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">{header}</th>
                                ))}
                            </tr>
                        </thead>
                        <tbody className="bg-white divide-y divide-gray-200">
                            {registrationTable.map(reg => (
                                <tr key={reg.id} className="hover:bg-gray-50">
                                    <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                                        {reg.registeredAt ? new Date(reg.registeredAt.seconds * 1000).toLocaleDateString() : 'N/A'}
                                    </td>
                                    <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">{reg.name}</td>
                                    <td className="px-6 py-4 whitespace-nowrap text-sm text-indigo-600 font-medium">{reg.courseName}</td>
                                    <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">{reg.specialization}</td>
                                    <td className="px-6 py-4 whitespace-nowrap">
                                        <span className={`px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${reg.status === 'Pending Review' ? 'bg-yellow-100 text-yellow-800' : 'bg-green-100 text-green-800'}`}>
                                            {reg.status}
                                        </span>
                                    </td>
                                </tr>
                            ))}
                            {totalRegistrations === 0 && (
                                <tr>
                                    <td colSpan="5" className="px-6 py-4 text-center text-gray-500">No registrations found yet.</td>
                                </tr>
                            )}
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    );
};

const AdminView = ({ registrations }) => {
    return (
        <div className="space-y-10">
            <h2 className="text-3xl font-extrabold text-indigo-700 border-b pb-2">Admin Dashboard</h2>

            {/* Section 1: Data Analysis */}
            <AdminDataAnalysis registrations={registrations} />

            {/* Section 2: Content Management */}
            <div className="space-y-6">
                <h3 className="text-2xl font-bold text-gray-900 flex items-center pt-4">
                    <Edit className="w-6 h-6 mr-2 text-indigo-600" /> Content Management
                </h3>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                    <AdminManagementForm 
                        title="Course Descriptions & Flyers" 
                        placeholder={coursesData.map(c => `${c.id}: ${c.name}`).join('\n') + "\n\n(Use this section to edit course details/add flyer links)"}
                        onSave={(content) => console.log('Course content updated:', content)}
                    />
                    <AdminManagementForm 
                        title="Calendar Updates" 
                        placeholder={calendarEvents.map(e => `${e.date}: ${e.title}`).join('\n') + "\n\n(Edit/add new events here)"}
                        onSave={(content) => console.log('Calendar updated:', content)}
                    />
                    <AdminManagementForm 
                        title="Instructor Details" 
                        placeholder={instructorsData.map(i => `${i.id}: ${i.name}, ${i.title}`).join('\n') + "\n\n(Update instructor profiles/bios)"}
                        onSave={(content) => console.log('Instructors updated:', content)}
                    />
                </div>
            </div>
        </div>
    );
};


// --- Main Application Component ---

const Sidebar = ({ currentPage, setCurrentPage, isSidebarOpen, setIsSidebarOpen, isAuthReady, isAdmin }) => {
  const baseMenuItems = [
    { name: 'Overview', page: 'home', Icon: Home },
    { name: 'My Dashboard', page: 'dashboard', Icon: BookOpen },
    { name: 'Careers', page: 'careers', Icon: Briefcase },
  ];
  
  const adminMenuItem = { name: 'Admin Dashboard', page: 'admin', Icon: BarChart2 };
  
  const menuItems = isAuthReady && isAdmin ? [...baseMenuItems, adminMenuItem] : baseMenuItems;

  return (
    <div className={`${isSidebarOpen ? 'translate-x-0' : '-translate-x-full'} fixed inset-y-0 left-0 z-30 w-64 bg-white border-r border-gray-200 p-4 transform transition-transform duration-300 ease-in-out
                   lg:relative lg:translate-x-0 lg:flex-shrink-0 lg:w-64 flex flex-col h-full`}>

      <div className="flex justify-end lg:hidden mb-4">
        <button onClick={() => setIsSidebarOpen(false)} className="text-gray-500 hover:text-gray-700 p-2 rounded-full hover:bg-gray-100 transition">
          <X className="w-6 h-6" />
        </button>
      </div>

      <div className="text-2xl font-bold text-indigo-700 mb-8">
        Learner Hub
      </div>

      <nav className="flex-1 space-y-2">
        {menuItems.map(({ name, page, Icon }) => {
          const isActive = currentPage === page;
          return (
            <button
              key={page}
              onClick={() => {
                setCurrentPage(page);
                setIsSidebarOpen(false); 
              }}
              className={`flex items-center w-full p-3 rounded-xl transition duration-150 ease-in-out
                ${isActive
                  ? 'bg-indigo-500 text-white shadow-md'
                  : 'text-gray-600 hover:bg-gray-100 hover:text-indigo-600'
                }`}
            >
              <Icon className="w-5 h-5 mr-3" />
              <span className="font-medium">{name}</span>
            </button>
          );
        })}
      </nav>
      
      <div className="mt-8 pt-4 border-t border-gray-100">
        <p className="text-xs text-gray-400">Auth Status: {isAuthReady ? 'Ready' : 'Initializing...'}</p>
        <p className="text-xs text-gray-400">User ID: {isAuthReady ? '...' + (userId || 'N/A').slice(-8) : 'N/A'}</p>
      </div>
    </div>
  );
};

const PageContent = ({ currentPage, onEnrollClick, registrations }) => {
  let content;

  switch (currentPage) {
    case 'home':
      content = <GeneralHomeView onEnrollClick={onEnrollClick} />;
      break;
    case 'dashboard':
      content = (
        <div className="space-y-4">
          <h2 className="text-3xl font-extrabold text-gray-900">My Dashboard (Student/User)</h2>
          <div className="p-6 bg-yellow-50 border border-yellow-200 text-yellow-800 rounded-lg">
            <p className="font-semibold">User Content Placeholder:</p>
            <p>This is where users can track their enrolled courses and progress.</p>
          </div>
        </div>
      );
      break;
    case 'admin':
        content = <AdminView registrations={registrations} />;
        break;
    case 'careers':
      content = (
        <div className="space-y-4">
          <h2 className="text-3xl font-extrabold text-gray-900">Career Resources</h2>
          <p className="text-gray-600">Access job boards, resume builders, and interview tips here.</p>
        </div>
      );
      break;
    default:
      content = <GeneralHomeView onEnrollClick={onEnrollClick} />;
  }

  return (
    <main className="flex-1 p-4 sm:p-8 lg:p-10 overflow-y-auto">
      <div className="max-w-7xl mx-auto">
        {content}
      </div>
    </main>
  );
};


const App = () => {
  const [currentPage, setCurrentPage] = useState('home');
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  
  // Firebase State
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [registrations, setRegistrations] = useState([]); // Live registrations from Firestore

  // Enrollment State
  const [showEnrollmentModal, setShowEnrollmentModal] = useState(false);
  const [selectedCourseToEnroll, setSelectedCourseToEnroll] = useState(null);
  
  const isAdmin = isAuthReady && !!userId; // Simplified admin check: if authenticated, you're an admin in this prototype

  // --- Firebase Initialization and Auth ---
  useEffect(() => {
      document.body.className = 'bg-gray-50';
      document.documentElement.style.fontFamily = 'Inter, sans-serif';

      try {
          const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
          const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');

          if (Object.keys(firebaseConfig).length === 0) {
              console.error("Firebase config is missing.");
              setIsAuthReady(true); // Complete auth process even if it fails
              return;
          }

          const app = initializeApp(firebaseConfig);
          const firestoreDb = getFirestore(app);
          const firebaseAuth = getAuth(app);

          setDb(firestoreDb);
          setAuth(firebaseAuth);

          const unsubscribe = onAuthStateChanged(firebaseAuth, async (user) => {
              if (user) {
                  setUserId(user.uid);
              } else {
                  if (typeof __initial_auth_token !== 'undefined') {
                      await signInWithCustomToken(firebaseAuth, __initial_auth_token).catch(e => console.error("Custom token sign-in failed:", e));
                  } else {
                      await signInAnonymously(firebaseAuth).catch(e => console.error("Anonymous sign-in failed:", e));
                  }
              }
              setIsAuthReady(true);
          });

          return () => unsubscribe();
      } catch (error) {
          console.error("Firebase setup failed:", error);
          setIsAuthReady(true);
      }
  }, []);

  // --- Fetch Registrations for Admin Dashboard (Live Firestore Listener) ---
  useEffect(() => {
      if (isAuthReady && db && isAdmin) {
          const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
          // Using a public collection for multi-user visibility (e.g., admin and other staff)
          const collectionPath = `/artifacts/${appId}/public/data/registrations`;
          
          const q = query(collection(db, collectionPath));

          const unsubscribe = onSnapshot(q, (snapshot) => {
              const fetchedRegistrations = snapshot.docs.map(doc => ({
                  id: doc.id,
                  ...doc.data()
              }));
              setRegistrations(fetchedRegistrations);
          }, (error) => {
              console.error("Error fetching registrations:", error);
          });

          return () => unsubscribe();
      }
  }, [isAuthReady, db, isAdmin]);
  
  // --- Handlers ---
  const handleEnrollClick = (course) => {
    setSelectedCourseToEnroll(course);
    setShowEnrollmentModal(true);
  };
  
  const handleEnrollmentSubmit = async (courseId, studentData) => {
    if (!db) {
        console.error("Firestore not initialized. Cannot save registration.");
        return;
    }
    
    // Use a public collection for registrations
    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
    const collectionPath = `/artifacts/${appId}/public/data/registrations`;
    const userIdentifier = userId || 'anonymous-' + Math.random().toString(36).substring(2, 9);

    try {
        await addDoc(collection(db, collectionPath), {
            ...studentData,
            courseId: courseId,
            courseName: coursesData.find(c => c.id === courseId)?.name || 'Unknown Course',
            registeredAt: serverTimestamp(),
            status: 'Pending Review',
            userId: userIdentifier
        });
        console.log("Registration successfully saved to Firestore.");
    } catch (error) {
        console.error("Error saving registration:", error);
    }
  };


  return (
    <div className="min-h-screen flex">
      {/* Mobile Menu Button */}
      <div className="lg:hidden fixed top-0 left-0 z-40 p-4">
        <button onClick={() => setIsSidebarOpen(true)} className="p-2 text-indigo-700 bg-white rounded-full shadow-lg">
          <Menu className="w-6 h-6" />
        </button>
      </div>

      {/* Sidebar Overlay for Mobile */}
      {isSidebarOpen && (
        <div 
          className="fixed inset-0 bg-black opacity-50 z-20 lg:hidden" 
          onClick={() => setIsSidebarOpen(false)}
        />
      )}

      {/* Sidebar Component */}
      <Sidebar 
        currentPage={currentPage} 
        setCurrentPage={setCurrentPage} 
        isSidebarOpen={isSidebarOpen}
        setIsSidebarOpen={setIsSidebarOpen}
        isAuthReady={isAuthReady}
        isAdmin={isAdmin}
        userId={userId}
      />

      {/* Main Content Area */}
      <PageContent 
        currentPage={currentPage} 
        onEnrollClick={handleEnrollClick}
        registrations={registrations} // Pass live data to AdminView
      />
      
      {/* Enrollment Modal */}
      {showEnrollmentModal && selectedCourseToEnroll && (
        <EnrollmentFormModal 
            course={selectedCourseToEnroll} 
            onClose={() => setShowEnrollmentModal(false)}
            onEnrollSubmit={handleEnrollmentSubmit}
        />
      )}
    </div>
  );
};

export default App;
