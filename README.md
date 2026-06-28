import { useState, useEffect } from 'react';
import { Sparkles, MapPin, History, LogOut, CheckCircle2, AlertTriangle, Users, Bell, ArrowRight, Heart, BookOpen, ShieldAlert } from 'lucide-react';
import { LoginScreen } from './components/LoginScreen';
import { StatsGrid } from './components/StatsGrid';
import { ThreeDCard } from './components/ThreeDCard';
import { ThreeDButton } from './components/ThreeDButton';
import { NewVisitFlow } from './components/NewVisitFlow';
import { HistorySyncCenter } from './components/HistorySyncCenter';
import { RegistryManager } from './components/RegistryManager';
import { VisitSubmission, CurrentUser, RCData } from './types';
import { RC_DATA } from './data';
import { initAuth } from './lib/googleAuth';
import { saveVisitToGoogleSheets } from './lib/googleSheets';

const STORAGE_KEY = 'rc_monitoring_visits_km_v1';
const USER_KEY = 'rc_monitoring_user_km_v1';

// Polished pre-populated sample visits to show off dashboard state immediately on startup
const INITIAL_DEMO_VISITS: VisitSubmission[] = [
  {
    id: "V1782485938000",
    communeId: "C002",
    communeName: "ស្រះរាំង",
    schoolId: "P017-17.បឋមសិក្សា ស្រះរាំង",
    schoolName: "បឋមសិក្សា ស្រះរាំង",
    teacherId: "T144-ឌី សុខឧត្តម",
    teacherName: "ឌី សុខឧត្តម",
    date: "2026-06-25",
    topic: "តាមដានវត្តមានកុមារប្រចាំខែ",
    incomplete: false,
    incompleteReason: "",
    savedAt: "2026-06-25T14:30:00.000Z",
    children: [
      {
        id: "CAM-199774-6002",
        name: "ជៀវ ដាណៃ",
        grade: "5",
        villageName: "បន្ទាយនាង",
        flags: [
          { key: "presence", label: "វត្តមាន", note: "អវត្តមាន ៥ថ្ងៃក្នុងខែនេះ ដោយសារការងារគ្រួសារ" }
        ]
      },
      {
        id: "CAM-199774-3973",
        name: "ឈួន គុណពេជ្រ",
        grade: "6ក",
        villageName: "តាចាន់",
        flags: []
      }
    ]
  },
  {
    id: "V1782140338000",
    communeId: "C003",
    communeName: "តាឡំ",
    schoolId: "P001-1.បឋមសក្សា បឹងវែង",
    schoolName: "បឋមសក្សា បឹងវែង",
    teacherId: "T003-ហូ សារុន",
    teacherName: "ហូ សារុន",
    date: "2026-06-18",
    topic: "12. បណ្តុះបណ្តាលអាន/សរសេរ, 80. បណ្តុះបណ្តាលអនាម័យ (WASH)",
    incomplete: false,
    incompleteReason: "",
    savedAt: "2026-06-18T10:15:00.000Z",
    children: [
      {
        id: "CAM-199774-3546",
        name: "ឃាន សីហា",
        grade: "6 ក",
        villageName: "បឹងវែង",
        flags: [
          { key: "education", label: "ការសិក្សា", note: "ខ្វះខាតសៀវភៅសិក្សា និងសម្ភារៈសរសេរខ្លាំង" }
        ]
      },
      {
        id: "CAM-199774-3537",
        name: "ផល អ៊ីស៊ុល",
        grade: "6 ក",
        villageName: "បឹងវែង",
        flags: []
      }
    ]
  },
  {
    id: "V1781845938000",
    communeId: "C003",
    communeName: "តាឡំ",
    schoolId: "P020-22. អនុវិទ្យាល័យ តាឡំ",
    schoolName: "អនុវិទ្យាល័យ តាឡំ",
    teacherId: "T177-ស៊ុន រួម",
    teacherName: "ស៊ុន រួម",
    date: "2026-06-10",
    topic: "19. បណ្តុះបណ្តាលសិទ្ធិកុមារ",
    incomplete: true,
    incompleteReason: "គ្រូបង្រៀនអវត្តមាន ដោយសារមានធុរៈគ្រួសារបន្ទាន់",
    savedAt: "2026-06-10T08:45:00.000Z",
    children: []
  }
];

export default function App() {
  const [currentUser, setCurrentUser] = useState<CurrentUser | null>(() => {
    try {
      const saved = localStorage.getItem(USER_KEY);
      return saved ? JSON.parse(saved) : null;
    } catch {
      return null;
    }
  });

  const [visits, setVisits] = useState<VisitSubmission[]>(() => {
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      if (saved) {
        return JSON.parse(saved);
      } else {
        localStorage.setItem(STORAGE_KEY, JSON.stringify(INITIAL_DEMO_VISITS));
        return INITIAL_DEMO_VISITS;
      }
    } catch {
      return INITIAL_DEMO_VISITS;
    }
  });

  const [route, setRoute] = useState<'dashboard' | 'newVisit' | 'history' | 'summary' | 'registry'>('dashboard');
  const [lastSubmittedVisit, setLastSubmittedVisit] = useState<VisitSubmission | null>(null);

  // Master baseline registry that can be customized or imported via Excel
  const [masterRegistry, setMasterRegistry] = useState<RCData>(() => {
    try {
      const saved = localStorage.getItem('rc_monitoring_master_registry_v1');
      if (saved) {
        return JSON.parse(saved);
      }
    } catch (e) {
      console.error(e);
    }
    return RC_DATA;
  });

  // Dynamic RC Data state initialized with masterRegistry, enriched by visits
  const [rcData, setRcData] = useState<RCData>(masterRegistry);

  // Automatically enrich communes, schools, teachers, and children from ALL visits
  useEffect(() => {
    const baseCommunes = [...masterRegistry.communes];
    const baseSchools = [...masterRegistry.schools];
    const baseTeachers = [...masterRegistry.teachers];
    const baseChildren = [...masterRegistry.children];

    const communeIds = new Set(baseCommunes.map(c => c.id));
    const schoolIds = new Set(baseSchools.map(s => s.id));
    const teacherIds = new Set(baseTeachers.map(t => t.id));
    const childIds = new Set(baseChildren.map(c => c.id));

    visits.forEach(v => {
      // 1. Enrich Commune
      if (v.communeId && !communeIds.has(v.communeId)) {
        baseCommunes.push({
          id: v.communeId,
          name: v.communeName || `ឃុំ ${v.communeId}`
        });
        communeIds.add(v.communeId);
      }

      // 2. Enrich School
      if (v.schoolId && !schoolIds.has(v.schoolId)) {
        baseSchools.push({
          id: v.schoolId,
          communeId: v.communeId,
          name: v.schoolName || `សាលា ${v.schoolId}`
        });
        schoolIds.add(v.schoolId);
      }

      // 3. Enrich Teacher
      if (v.teacherId && !teacherIds.has(v.teacherId)) {
        baseTeachers.push({
          id: v.teacherId,
          schoolId: v.schoolId,
          name: v.teacherName || `គ្រូ ${v.teacherName || v.teacherId}`
        });
        teacherIds.add(v.teacherId);
      }

      // 4. Enrich Children
      v.children.forEach(c => {
        if (c.id && !childIds.has(c.id)) {
          baseChildren.push({
            id: c.id,
            name: c.name,
            gender: "ស្រី",
            villageId: "V_UNKNOWN",
            villageName: c.villageName || "មិនបានបញ្ជាក់",
            age: "12",
            grade: c.grade || "6",
            status: "មានអ្នកទំនុកបំរុង",
            communeId: v.communeId,
            schoolId: v.schoolId,
            teacherId: v.teacherId,
            baseline: {
              presence: { status: "good", note: "" },
              health: { status: "good", note: "" },
              education: { status: "good", note: "" },
              protection: { status: "good", note: "" },
              participation: { status: "good", note: "" }
            }
          });
          childIds.add(c.id);
        }
      });
    });

    // End of visits loop

    const enriched: RCData = {
      communes: baseCommunes,
      schools: baseSchools,
      teachers: baseTeachers,
      children: baseChildren
    };

    setRcData(enriched);
    try {
      localStorage.setItem('rc_monitoring_rcdata_km_v1', JSON.stringify(enriched));
    } catch (e) {
      console.error(e);
    }
  }, [visits, masterRegistry]);

  // Google OAuth token state for background auto-push
  const [googleToken, setGoogleToken] = useState<string | null>(null);

  useEffect(() => {
    const unsubscribe = initAuth(
      (user, token) => {
        setGoogleToken(token);
      },
      () => {
        setGoogleToken(null);
      }
    );
    return () => unsubscribe();
  }, []);

  // Persistence triggers
  useEffect(() => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(visits));
  }, [visits]);

  useEffect(() => {
    if (currentUser) {
      localStorage.setItem(USER_KEY, JSON.stringify(currentUser));
    } else {
      localStorage.removeItem(USER_KEY);
    }
  }, [currentUser]);

  const handleLogin = (user: CurrentUser) => {
    setCurrentUser(user);
    setRoute('dashboard');
  };

  const handleLogout = () => {
    if (confirm('🔒 តើអ្នកពិតជាចង់ចាកចេញពីគណនីមែនទេ?')) {
      setCurrentUser(null);
      setRoute('dashboard');
    }
  };

  const handleSaveVisit = async (newVisit: VisitSubmission) => {
    // 1. Save locally first (instant and robust)
    setVisits([newVisit, ...visits]);
    setLastSubmittedVisit(newVisit);
    setRoute('summary');

    // 2. If signed in with Google, push to Google Sheets automatically!
    if (googleToken) {
      try {
        await saveVisitToGoogleSheets(googleToken, newVisit);
        console.log("Successfully auto-pushed visit to Google Sheets in the background.");
      } catch (err) {
        console.error("Failed to auto-push to Google Sheets:", err);
      }
    }
  };

  // Compile active flagged alerts from all past submissions
  const getFlaggedAlerts = () => {
    const alerts: Array<{ childName: string, id: string, schoolName: string, date: string, flagLabel: string, note: string, key: string }> = [];
    visits.forEach(v => {
      v.children.forEach(c => {
        c.flags.forEach(f => {
          alerts.push({
            id: c.id,
            childName: c.name,
            schoolName: v.schoolName,
            date: v.date,
            flagLabel: f.label,
            note: f.note,
            key: f.key
          });
        });
      });
    });
    return alerts.slice(0, 5); // display most recent 5
  };

  const flaggedAlerts = getFlaggedAlerts();

  if (!currentUser) {
    return <LoginScreen rcData={rcData} onLoginSuccess={handleLogin} />;
  }

  return (
    <div className="min-h-screen bg-gradient-to-b from-[#E2F1FF] via-[#E8F8EE] to-[#FAF7F2] text-slate-800 flex flex-col font-sans selection:bg-[#4CAF50]/10 select-none antialiased">
      {/* Friendly Cartoon-Styled Header */}
      <header className="sticky top-0 z-40 bg-white/85 backdrop-blur-md border-b-2 border-[#2E5E35]/10 shadow-sm px-4 py-3.5">
        <div className="max-w-7xl mx-auto flex items-center justify-between gap-4">
          <div className="flex items-center gap-2.5">
            <span className="w-3.5 h-3.5 rounded-full bg-[#4CAF50] shadow-[0_0_8px_rgba(76,175,80,0.6)] animate-pulse" />
            <div>
              <div className="text-[10px] font-black text-[#2E7D32] tracking-wider uppercase flex items-center gap-1">
                🎒 World Vision Cambodia
              </div>
              <h2 className="text-sm md:text-base font-black text-slate-900 tracking-tight">
                កម្មវិធីតាមដានសាលារៀន (School Tracker)
              </h2>
            </div>
          </div>

          <div className="flex items-center gap-3">
            <div className="hidden sm:block text-right">
              <div className="text-xs font-black text-slate-800">🧑‍🏫 {currentUser.name}</div>
              <div className="text-[10px] font-bold text-slate-400 mt-0.5">{currentUser.role}</div>
            </div>
            
            <button
              onClick={handleLogout}
              className="p-2.5 text-slate-400 hover:text-red-500 hover:bg-red-50 rounded-xl transition-all active:scale-95 flex items-center justify-center border-2 border-transparent hover:border-red-200 cursor-pointer"
              title="ចាកចេញ (Logout)"
            >
              <LogOut className="w-5 h-5" />
            </button>
          </div>
        </div>
      </header>

      {/* Main Container */}
      <main className="flex-1 w-full max-w-7xl mx-auto p-4 md:p-6 pb-24">
        {route === 'dashboard' && (
          <div className="space-y-6">
            {/* Split layout: Warm welcome text on the left, beautiful Ghibli-style cartoon illustration on the right */}
            <ThreeDCard
              className="border-2 border-[#2E5E35]/15 bg-white p-6 md:p-8 shadow-[0_12px_40px_rgba(46,94,53,0.06)] relative overflow-hidden rounded-3xl"
              hoverEffect={false}
            >
              <div className="grid grid-cols-1 lg:grid-cols-12 gap-6 items-center">
                <div className="lg:col-span-7 space-y-4 relative z-10">
                  <span className="px-3.5 py-1.5 bg-emerald-50 border-2 border-emerald-100 text-[#2E7D32] rounded-full text-xs font-black tracking-wide flex items-center gap-1.5 w-fit shadow-sm">
                    <Sparkles className="w-3.5 h-3.5 text-amber-500 animate-pulse" />
                    តំបន់ប្រតិបត្តិការកម្ពុជា (Cambodia Area)
                  </span>
                  <h1 className="text-2xl md:text-4xl font-black text-slate-900 tracking-tight leading-tight">
                    សូមស្វាគមន៍មកកាន់ប្រព័ន្ធ <br />
                    តាមដានវត្តមាន និងសុខុមាលភាពកុមារ 🌾
                  </h1>
                  <p className="text-slate-600 text-sm md:text-base font-semibold leading-relaxed">
                    ជួយសម្របសម្រួល និងសហការជាមួយគ្រូបង្រៀន ដើម្បីធានាថាកុមារគ្រប់រូបទទួលបានការអប់រំ សុខភាព និងការការពារពេញលេញស្របតាមគោលការណ៍ World Vision។
                  </p>
                </div>
                <div className="lg:col-span-5 relative">
                  <div className="absolute inset-0 bg-emerald-100 rounded-2xl transform rotate-1 scale-102 opacity-50" />
                  <img
                    src="/src/assets/images/nature_school_banner_1782618173240.jpg"
                    alt="Countryside School Banner"
                    className="w-full h-48 md:h-56 rounded-2xl object-cover relative z-10 border-2 border-[#2E5E35]/10 shadow-md"
                    referrerPolicy="no-referrer"
                  />
                </div>
              </div>
            </ThreeDCard>

            {/* Bento Stats Grid */}
            <StatsGrid rcData={rcData} totalVisits={visits.length} />

            {/* Interactive Grid Actions */}
            <div className="grid grid-cols-1 md:grid-cols-2 gap-6 pt-2">
              <div className="space-y-4">
                <h3 className="text-sm font-black text-[#2E7D32] uppercase tracking-widest flex items-center gap-1.5">
                  <MapPin className="w-4 h-4 text-[#2E7D32]" />
                  📍 ប្រតិបត្តិការរហ័ស (Quick Actions)
                </h3>
                
                <ThreeDCard
                  onClick={() => setRoute('newVisit')}
                  className="hover:border-[#4CAF50]/50 flex items-center gap-4 py-5 bg-white border-2 border-[#2E5E35]/10"
                >
                  <div className="p-4 bg-emerald-50 text-[#2E7D32] rounded-2xl shadow-inner border border-emerald-100">
                    <MapPin className="w-6 h-6" />
                  </div>
                  <div>
                    <h4 className="font-black text-slate-900 text-lg">ចាប់ផ្ដើមការតាមដានថ្មី</h4>
                    <p className="text-xs text-[#2E7D32] font-black mt-1 uppercase tracking-wide">
                      Start New School Visit Monitoring
                    </p>
                  </div>
                </ThreeDCard>

                <ThreeDCard
                  onClick={() => setRoute('history')}
                  className="hover:border-[#4CAF50]/40 flex items-center gap-4 py-5 bg-white border-2 border-[#2E5E35]/10"
                >
                  <div className="p-4 bg-sky-50 text-sky-600 rounded-2xl shadow-inner border border-sky-100">
                    <History className="w-6 h-6" />
                  </div>
                  <div>
                    <h4 className="font-black text-slate-900 text-lg">ប្រវត្តិការងារ & របាយការណ៍</h4>
                    <p className="text-xs text-sky-600 font-black mt-1 uppercase tracking-wide">
                      Visit Records & Excel Export
                    </p>
                  </div>
                </ThreeDCard>

                {currentUser.isAdmin && (
                  <ThreeDCard
                    onClick={() => setRoute('registry')}
                    className="hover:border-amber-500/50 flex items-center gap-4 py-5 bg-gradient-to-r from-amber-50/50 to-white border-2 border-amber-500/20 cursor-pointer animate-fade-in"
                  >
                    <div className="p-4 bg-amber-100/60 text-amber-700 rounded-2xl shadow-inner border border-amber-200">
                      <Users className="w-6 h-6" />
                    </div>
                    <div>
                      <h4 className="font-black text-slate-900 text-lg">គ្រប់គ្រងបញ្ជី & ទិន្នន័យមេ</h4>
                      <p className="text-xs text-amber-700 font-black mt-1 uppercase tracking-wide">
                        Master Registry & Excel Importer
                      </p>
                    </div>
                  </ThreeDCard>
                )}
              </div>

              {/* Live Alerts Telemetry Center */}
              <div className="space-y-4">
                <h3 className="text-sm font-black text-amber-600 uppercase tracking-widest flex items-center gap-1.5">
                  <Bell className="w-4 h-4 text-amber-500 animate-bounce" />
                  🔔 ទិន្នន័យចាំបាច់ត្រូវដោះស្រាយ (Priority Actions Center)
                </h3>

                <ThreeDCard className="border-2 border-[#2E5E35]/10 space-y-3 bg-white" hoverEffect={false}>
                  {flaggedAlerts.length === 0 ? (
                    <div className="text-center py-8">
                      <div className="text-3xl mb-1 animate-bounce">☀️</div>
                      <h4 className="font-black text-[#2E7D32]">គ្មានសញ្ញាទង់សញ្ញាគ្រោះថ្នាក់ទេ</h4>
                      <p className="text-xs text-slate-500 mt-1 font-semibold leading-relaxed">
                        កុមារទាំងអស់ទទួលបានការវាយតម្លៃថា ល្អប្រសើរធម្មតា។
                      </p>
                    </div>
                  ) : (
                    <div className="divide-y divide-[#2E5E35]/10 max-h-[260px] overflow-y-auto pr-1">
                      {flaggedAlerts.map((alert, idx) => {
                        const iconType = alert.key === 'health' ? '❤️' : alert.key === 'education' ? '📚' : '⚠️';
                        return (
                          <div key={idx} className="py-2.5 first:pt-0 last:pb-0 flex items-start gap-3">
                            <span className="text-lg bg-slate-50 border border-slate-200 p-1.5 rounded-lg flex-shrink-0 mt-0.5">{iconType}</span>
                            <div className="flex-1 min-w-0">
                              <div className="flex justify-between items-baseline gap-2">
                                <span className="font-black text-slate-900 text-sm">{alert.childName}</span>
                                <span className="text-[10px] text-slate-400 font-bold">{alert.date}</span>
                              </div>
                              <div className="text-[11px] text-amber-600 font-black mt-0.5 uppercase tracking-wide">
                                ⚑ ត្រូវការអន្តរាគមន៍: {alert.flagLabel}
                              </div>
                              <p className="text-xs text-slate-600 leading-relaxed font-semibold mt-1">
                                {alert.note || 'មិនមានការកត់ត្រាបញ្ហាលម្អិត'}
                              </p>
                              <p className="text-[10px] text-slate-400 font-bold mt-0.5">
                                សាលា: {alert.schoolName}
                              </p>
                            </div>
                          </div>
                        );
                      })}
                    </div>
                  )}
                </ThreeDCard>
              </div>
            </div>
          </div>
        )}

        {route === 'newVisit' && (
          <NewVisitFlow
            currentUser={currentUser}
            rcData={rcData}
            onSaveVisit={handleSaveVisit}
            onCancel={() => setRoute('dashboard')}
          />
        )}

        {route === 'history' && (
          <HistorySyncCenter
            visits={visits}
            currentUser={currentUser}
            rcData={rcData}
            onUpdateVisits={setVisits}
            onBackToDashboard={() => setRoute('dashboard')}
          />
        )}

        {route === 'registry' && (
          <RegistryManager
            masterRegistry={masterRegistry}
            onUpdateMasterRegistry={setMasterRegistry}
            onBackToDashboard={() => setRoute('dashboard')}
          />
        )}

        {route === 'summary' && lastSubmittedVisit && (
          <div className="max-w-xl mx-auto space-y-6">
            <div className="text-center space-y-3">
              <div className="w-20 h-20 bg-emerald-50 text-emerald-600 rounded-full flex items-center justify-center mx-auto shadow-md border-2 border-emerald-100 animate-bounce">
                <CheckCircle2 className="w-12 h-12" />
              </div>
              <h1 className="text-2xl font-black text-slate-900 tracking-tight leading-tight">
                បានរក្សាទុកកំណត់ត្រាដោយជោគជ័យ! 🎉
              </h1>
              <p className="text-sm text-slate-600 leading-relaxed font-semibold">
                ការតាមដានសម្រាប់គ្រូ <strong>{lastSubmittedVisit.teacherName}</strong> នៅសាលា <strong>{lastSubmittedVisit.schoolName}</strong> ត្រូវបានបញ្ចូលទៅក្នុងទូរស័ព្ទ និងកំពុងបញ្ជូនទៅ Google Sheets។
              </p>
            </div>

            <ThreeDCard className="border-2 border-[#2E5E35]/15 bg-white p-5 space-y-4 shadow-lg rounded-3xl" hoverEffect={false}>
              <h3 className="text-sm font-black text-slate-800 uppercase tracking-widest border-b border-[#2E5E35]/10 pb-2 flex items-center gap-1.5">
                📝 សេចក្ដីសង្ខេបលទ្ធផល (Submission Summary)
              </h3>
              
              <div className="grid grid-cols-2 gap-4">
                <div>
                  <div className="text-xs text-slate-400 font-bold uppercase">📅 ថ្ងៃតាមដាន:</div>
                  <div className="text-sm font-black text-slate-800 mt-0.5">{lastSubmittedVisit.date}</div>
                </div>
                <div>
                  <div className="text-xs text-slate-400 font-bold uppercase">🏁 ស្ថានភាព:</div>
                  <div className={`text-sm font-black mt-0.5 ${lastSubmittedVisit.incomplete ? 'text-red-500' : 'text-emerald-600'}`}>
                    {lastSubmittedVisit.incomplete ? 'មិនបានបញ្ចប់' : 'បញ្ចប់ជោគជ័យ'}
                  </div>
                </div>
              </div>

              {lastSubmittedVisit.incomplete && lastSubmittedVisit.incompleteReason && (
                <div className="p-3.5 bg-red-50 border border-red-200 rounded-2xl">
                  <div className="text-xs text-red-600 font-black uppercase">មូលហេតុមិនបានបញ្ចប់:</div>
                  <div className="text-xs text-red-500 font-bold mt-1 leading-relaxed">{lastSubmittedVisit.incompleteReason}</div>
                </div>
              )}

              {!lastSubmittedVisit.incomplete && (
                <div className="grid grid-cols-3 gap-2 text-center pt-2">
                  <div className="bg-slate-50 border border-slate-200 px-3 py-2 rounded-2xl text-slate-800">
                    <div className="text-lg font-black">{lastSubmittedVisit.children.length}</div>
                    <div className="text-[10px] font-bold text-slate-400">កុមារសរុប</div>
                  </div>
                  <div className="bg-emerald-50 border border-emerald-100 px-3 py-2 rounded-2xl text-emerald-600">
                    <div className="text-lg font-black">
                      {lastSubmittedVisit.children.length - lastSubmittedVisit.children.filter(c => c.flags.length > 0).length}
                    </div>
                    <div className="text-[10px] font-bold text-emerald-500">ល្អទាំងអស់</div>
                  </div>
                  <div className="bg-amber-50 border border-amber-100 px-3 py-2 rounded-2xl text-amber-600">
                    <div className="text-lg font-black">{lastSubmittedVisit.children.filter(c => c.flags.length > 0).length}</div>
                    <div className="text-[10px] font-bold text-amber-500">មានបញ្ហា</div>
                  </div>
                </div>
              )}

              {lastSubmittedVisit.children.filter(c => c.flags.length > 0).length > 0 && (
                <div className="pt-2 border-t border-slate-100 space-y-2">
                  <div className="text-xs font-black text-slate-400 uppercase tracking-wide">⚠️ កុមារមានបញ្ហា:</div>
                  <div className="space-y-2 max-h-[160px] overflow-y-auto pr-1">
                    {lastSubmittedVisit.children.filter(c => c.flags.length > 0).map(c => (
                      <div key={c.id} className="p-2.5 bg-amber-50/50 rounded-2xl border border-amber-100 flex items-start justify-between gap-3 text-xs leading-relaxed text-slate-700">
                        <div>
                          <span className="font-extrabold text-slate-900">{c.name}</span>
                          <span className="text-[10px] text-slate-400 font-bold ml-1">ថ្នាក់ទី {c.grade}</span>
                          <div className="text-[10px] text-amber-600 font-extrabold mt-1">
                            ⚑ {c.flags.map(f => f.label).join(', ')}
                          </div>
                        </div>
                      </div>
                    ))}
                  </div>
                </div>
              )}
            </ThreeDCard>

            <div className="flex gap-3 justify-center">
              <ThreeDButton onClick={() => setRoute('dashboard')} variant="secondary" className="w-full">
                ត្រឡប់ទំព័រដើម
              </ThreeDButton>
              <ThreeDButton onClick={() => setRoute('newVisit')} variant="primary" className="w-full">
                ការតាមដានថ្មីទៀត
              </ThreeDButton>
            </div>
          </div>
        )}
      </main>
    </div>
  );
}
