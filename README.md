<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>高校教員用 週案管理アプリ</title>
    
    <!-- React & ReactDOM -->
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    
    <!-- Babel (for JSX) -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <!-- Phosphor Icons (Lightweight icons) -->
    <script src="https://unpkg.com/@phosphor-icons/web"></script>

    <style>
        /* Base Styles & Resets */
        html, body {
            height: 100%;
            width: 100%;
            overflow: hidden; 
            position: fixed; 
        }

        body {
            font-family: 'BIZ UDPGothic', 'Hiragino Kaku Gothic ProN', 'Hiragino Sans', Meiryo, 'Helvetica Neue', Arial, sans-serif;
            background-color: #f3f4f6;
            touch-action: manipulation; 
        }
        
        overscroll-behavior-y: contain;

        ::-webkit-scrollbar {
            width: 8px;
            height: 8px;
        }
        ::-webkit-scrollbar-track {
            background: #f1f1f1; 
        }
        ::-webkit-scrollbar-thumb {
            background: #c1c1c1; 
            border-radius: 4px;
        }
        ::-webkit-scrollbar-thumb:hover {
            background: #a8a8a8; 
        }

        .modal-overlay { z-index: 50; }
        .modal-enter { opacity: 0; transform: scale(0.95); }
        .modal-enter-active { opacity: 1; transform: scale(1); transition: all 200ms ease-out; }
        .modal-exit { opacity: 1; transform: scale(1); }
        .modal-exit-active { opacity: 0; transform: scale(0.95); transition: all 200ms ease-in; }

        .timetable-grid {
            display: grid;
            grid-template-columns: 35px repeat(6, 1fr);
            gap: 2px;
            background-color: #e5e7eb;
            border: 1px solid #e5e7eb;
        }
        .timetable-cell {
            background-color: white;
            min-height: 80px;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            text-align: center;
            font-size: 0.85rem;
            cursor: pointer;
            padding: 2px;
            transition: background-color 0.1s;
        }
        .timetable-cell:active {
            background-color: #f9fafb;
        }
        .daily-memo-cell {
            background-color: #fffbeb;
            min-height: 50px; 
            display: flex;
            flex-direction: column;
            justify-content: flex-start;
            align-items: center;
            text-align: center;
            font-size: 0.75rem;
            cursor: pointer;
            padding: 4px 2px;
            transition: background-color 0.1s;
            color: #92400e;
            font-weight: bold;
            line-height: 1.2;
        }
        .timetable-header {
            background-color: #f9fafb;
            font-weight: bold;
            display: flex;
            align-items: center;
            justify-content: center;
            padding: 8px 0;
            position: sticky;
            top: 0;
            z-index: 10;
        }
        .period-header {
            background-color: #f9fafb;
            font-weight: bold;
            display: flex;
            align-items: center;
            justify-content: center;
            color: #6b7280;
            font-size: 1rem; 
            word-break: break-all;
            padding: 2px;
            line-height: 1.1;
        }

        @media print {
            @page {
                size: landscape;
                margin: 5mm; 
            }
            html, body {
                height: auto !important;
                overflow: visible !important;
                position: static !important;
                background-color: white !important;
                zoom: 0.95;
            }
            #root, .main-container {
                height: auto !important;
                overflow: visible !important;
                max-width: none !important;
                box-shadow: none !important;
            }
            nav, .no-print, button:not(.hidden), .settings-btn {
                display: none !important;
            }
            main {
                padding-top: 0 !important;
                overflow: visible !important;
            }
            .pb-10, .pb-20 {
                padding-bottom: 0 !important;
            }

            .timetable-grid {
                border: 2px solid #000;
                gap: 0;
                width: 100% !important;
                min-width: 0 !important;
            }
            .timetable-header, .period-header, .timetable-cell, .daily-memo-cell {
                border: 1px solid #666;
            }
            .timetable-header {
                background-color: #f0f0f0 !important;
                color: #000 !important;
                padding: 2px 0 !important;
            }
            
            .timetable-cell {
                min-height: 77px !important; 
                height: auto !important;
                font-size: 10pt !important;
                padding: 1px !important;
            }
            .daily-memo-cell {
                min-height: 40px !important;
                font-size: 9pt !important;
            }
            
            .print-header {
                display: block !important;
                text-align: center;
                margin-bottom: 10px;
            }
            .print-header h1 {
                font-size: 20px;
                font-weight: bold;
                margin: 0;
            }
            body {
                -webkit-print-color-adjust: exact;
                print-color-adjust: exact;
            }
        }

        .print-header {
            display: none;
        }

        @media (max-width: 640px) {
            .timetable-cell {
                font-size: 0.70rem;
                min-height: 70px;
            }
            .daily-memo-cell {
                font-size: 0.65rem;
                min-height: 60px;
            }
        }
    </style>
</head>
<body>
    <div id="root" style="height: 100%;"></div>

    <script type="text/babel">
        const { useState, useEffect, useMemo, useRef, useCallback } = React;

        // --- Utils ---
        
        const getMonday = (d) => {
            d = new Date(d);
            const day = d.getDay();
            const diff = d.getDate() - day + (day === 0 ? -6 : 1);
            return new Date(d.setDate(diff));
        };

        const addDays = (date, days) => {
            const result = new Date(date);
            result.setDate(result.getDate() + days);
            return result;
        };

        const formatDateKey = (date) => {
            return date.toISOString().split('T')[0];
        };

        const formatDateJP = (date) => {
            return `${date.getFullYear()}年${date.getMonth() + 1}月${date.getDate()}日`;
        };
        
        const formatDateMMDD = (date) => {
            return `${date.getMonth() + 1}/${date.getDate()}`;
        };

        const getYearMonthString = (monday) => {
            const saturday = addDays(monday, 5);
            const startYear = monday.getFullYear();
            const startMonth = monday.getMonth() + 1;
            const endYear = saturday.getFullYear();
            const endMonth = saturday.getMonth() + 1;

            if (startYear === endYear && startMonth === endMonth) {
                return `${startYear}年${startMonth}月`;
            } else {
                return `${startYear}年${startMonth}月-${endYear}年${endMonth}月`;
            }
        };

        const getWeekRangeString = (monday) => {
            const saturday = addDays(monday, 5);
            return `${formatDateJP(monday)} 〜 ${formatDateJP(saturday)}`;
        };

        const getFiscalYear = (date) => {
            const year = date.getFullYear();
            const month = date.getMonth() + 1; 
            if (month <= 3) {
                return year - 1;
            }
            return year;
        };

        // --- Holiday Logic ---
        const getHolidayNameRaw = (date) => {
            const year = date.getFullYear();
            const month = date.getMonth() + 1;
            const day = date.getDate();
            const dayOfWeek = date.getDay();

            if (month === 1 && day === 1) return "元日";
            if (month === 2 && day === 11) return "建国記念の日";
            if (month === 2 && day === 23) return "天皇誕生日";
            if (month === 4 && day === 29) return "昭和の日";
            if (month === 5 && day === 3) return "憲法記念日";
            if (month === 5 && day === 4) return "みどりの日";
            if (month === 5 && day === 5) return "こどもの日";
            if (month === 8 && day === 11) return "山の日";
            if (month === 11 && day === 3) return "文化の日";
            if (month === 11 && day === 23) return "勤労感謝の日";

            const isHappyMonday = (m, n) => {
                if (month !== m) return false;
                if (dayOfWeek !== 1) return false;
                const firstDayOfWeek = new Date(year, month - 1, 1).getDay();
                const firstMondayDate = firstDayOfWeek <= 1 ? 1 + (1 - firstDayOfWeek) : 1 + (8 - firstDayOfWeek);
                const targetDate = firstMondayDate + (n - 1) * 7;
                return day === targetDate;
            }

            if (isHappyMonday(1, 2)) return "成人の日";
            if (isHappyMonday(7, 3)) return "海の日";
            if (isHappyMonday(9, 3)) return "敬老の日";
            if (isHappyMonday(10, 2)) return "スポーツの日";

            const vernal = Math.floor(20.8431 + 0.242194 * (year - 1980) - Math.floor((year - 1980) / 4));
            const autumnal = Math.floor(23.2488 + 0.242194 * (year - 1980) - Math.floor((year - 1980) / 4));
            if (month === 3 && day === vernal) return "春分の日";
            if (month === 9 && day === autumnal) return "秋分の日";

            return null;
        };

        const getHolidayName = (date) => {
            const rawName = getHolidayNameRaw(date);
            if (rawName) return rawName;

            const yesterday = new Date(date);
            yesterday.setDate(date.getDate() - 1);
            if (yesterday.getDay() === 0) {
                const yName = getHolidayNameRaw(yesterday);
                if (yName) return `振替休日（${yName}）`;
            }

            const month = date.getMonth() + 1;
            const day = date.getDate();
            if (month === 5 && day === 6) {
                const d3 = new Date(date.getFullYear(), 4, 3).getDay();
                const d4 = new Date(date.getFullYear(), 4, 4).getDay();
                if ((d3 === 0 || d4 === 0) && date.getDay() > 1) { 
                     return "振替休日";
                }
            }
            return null;
        };

        // --- Storage Logic ---
        const APP_ID = 'teacher_schedule_app_v30'; 
        const LEGACY_KEYS = ['teacher_schedule_app_v27', 'teacher_schedule_app_v18', 'teacher_schedule_app_v4'];

        const getMetaKey = () => `${APP_ID}_meta`;
        const getDataKey = (year) => `${APP_ID}_data_${year}`;

        // Initial Data
        const INITIAL_COURSES = [
            { id: 'c1', name: 'LHR', color: '#bbf7d0' }, 
        ];

        const INITIAL_TERMS = [
            { id: 't1', name: '1学期中間', start: '', end: '' },
            { id: 't2', name: '1学期末', start: '', end: '' },
            { id: 't3', name: '2学期中間', start: '', end: '' },
            { id: 't4', name: '2学期末', start: '', end: '' },
            { id: 't5', name: '学年末', start: '', end: '' },
        ];

        const PRESET_COLORS = [
            '#fee2e2', '#ffedd5', '#fef9c3', '#dcfce7', '#dbeafe', '#e0e7ff', '#f3e8ff', '#fae8ff', '#f1f5f9', '#ffffff'
        ];
        
        // Migration
        const migrateLegacyData = () => {
            let legacyData = null;
            for (const key of LEGACY_KEYS) {
                const raw = localStorage.getItem(key);
                if (raw) {
                    try {
                        legacyData = JSON.parse(raw);
                        break; 
                    } catch(e) {}
                }
            }

            if (!legacyData) return null;

            const yearsData = {}; 
            
            const addToYear = (year, key, value, type) => {
                if (!yearsData[year]) {
                    yearsData[year] = {
                        courses: legacyData.courses || INITIAL_COURSES,
                        basicSchedules: legacyData.basicSchedules || (legacyData.basicSchedule ? { A: legacyData.basicSchedule, B: {}, C: {} } : { A: {}, B: {}, C: {} }),
                        weeklySchedules: {},
                        weeklyMemos: {},
                        termSettings: legacyData.termSettings || INITIAL_TERMS,
                        statsRange: legacyData.statsRange || null,
                        pmCutPeriod: legacyData.pmCutPeriod || 5,
                        periodLabels: legacyData.periodLabels || ['1', '2', '3', '4', '5', '6', '7']
                    };
                }
                if (type === 'schedule') yearsData[year].weeklySchedules[key] = value;
                if (type === 'memo') yearsData[year].weeklyMemos[key] = value;
            };

            if (legacyData.weeklySchedules) {
                Object.keys(legacyData.weeklySchedules).forEach(dateKey => {
                    const year = getFiscalYear(new Date(dateKey));
                    addToYear(year, dateKey, legacyData.weeklySchedules[dateKey], 'schedule');
                });
            }

            if (legacyData.weeklyMemos) {
                Object.keys(legacyData.weeklyMemos).forEach(dateKey => {
                    if (dateKey.startsWith('daily-')) return; 
                    const year = getFiscalYear(new Date(dateKey));
                    addToYear(year, dateKey, legacyData.weeklyMemos[dateKey], 'memo');
                });
            }

            if (Object.keys(yearsData).length === 0) {
                const currentY = getFiscalYear(new Date());
                yearsData[currentY] = {
                    courses: legacyData.courses || INITIAL_COURSES,
                    basicSchedules: legacyData.basicSchedules || { A: {}, B: {}, C: {} },
                    weeklySchedules: {},
                    weeklyMemos: {},
                    termSettings: legacyData.termSettings || INITIAL_TERMS,
                    statsRange: legacyData.statsRange,
                    pmCutPeriod: legacyData.pmCutPeriod || 5,
                    periodLabels: legacyData.periodLabels || ['1', '2', '3', '4', '5', '6', '7']
                };
            }

            const availableYears = Object.keys(yearsData).map(Number).sort();
            availableYears.forEach(year => {
                localStorage.setItem(getDataKey(year), JSON.stringify(yearsData[year]));
            });

            const meta = {
                years: availableYears,
                lastActiveYear: getFiscalYear(new Date()) 
            };
            localStorage.setItem(getMetaKey(), JSON.stringify(meta));
            return meta;
        };

        // --- Components ---
        const Icon = ({ name, size = 20, className = "", weight = "regular" }) => {
            return <i className={`ph ph-${name} ${className}`} style={{ fontSize: size, fontWeight: weight === "bold" ? 700 : 400 }}></i>;
        };

        const Modal = ({ isOpen, onClose, title, children }) => {
            if (!isOpen) return null;
            return (
                <div className="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-50 p-4 transition-opacity modal-overlay">
                    <div className="bg-white rounded-xl shadow-xl w-full max-w-md overflow-hidden animate-fade-in-up">
                        <div className="flex justify-between items-center p-4 border-b bg-gray-50">
                            <h3 className="font-bold text-lg text-gray-800">{title}</h3>
                            <button onClick={onClose} className="p-2 rounded-full hover:bg-gray-200 text-gray-500">
                                <Icon name="x" size={24} />
                            </button>
                        </div>
                        <div className="p-4 max-h-[70vh] overflow-y-auto">
                            {children}
                        </div>
                    </div>
                </div>
            );
        };

        const App = () => {
            // Global State
            const [currentDate, setCurrentDate] = useState(getMonday(new Date()));
            const currentFiscalYear = useMemo(() => getFiscalYear(currentDate), [currentDate]);
            const loadedFiscalYearRef = useRef(null); 
            const [isDataReady, setIsDataReady] = useState(false);

            const [courses, setCourses] = useState(INITIAL_COURSES);
            const [basicSchedules, setBasicSchedules] = useState({ A: {}, B: {}, C: {} });
            const [weeklySchedules, setWeeklySchedules] = useState({});
            const [weeklyMemos, setWeeklyMemos] = useState({});
            const [pmCutPeriod, setPmCutPeriod] = useState(5); 
            const [periodLabels, setPeriodLabels] = useState(['1', '2', '3', '4', '5', '6', '7']);
            const [statsRange, setStatsRange] = useState({ start: "", end: "" });
            const [termSettings, setTermSettings] = useState(INITIAL_TERMS);

            const [activeTab, setActiveTab] = useState('schedule');
            const [isLoaded, setIsLoaded] = useState(false);
            const [isSettingsOpen, setIsSettingsOpen] = useState(false); 
            const [toast, setToast] = useState(null);

            // 1. Initial Load / Migration
            useEffect(() => {
                const metaRaw = localStorage.getItem(getMetaKey());
                if (!metaRaw) {
                    migrateLegacyData();
                }
                setIsLoaded(true); 
            }, []);

            const loadDataForYear = (year) => {
                let nextData = null;
                const raw = localStorage.getItem(getDataKey(year));
                
                if (raw) {
                    try {
                        nextData = JSON.parse(raw);
                    } catch (e) {
                        console.error("Data parse error, initializing fresh", e);
                    }
                }

                if (!nextData) {
                    const prevRaw = localStorage.getItem(getDataKey(year - 1));
                    if (prevRaw) {
                        try {
                            const prev = JSON.parse(prevRaw);
                            nextData = {
                                courses: prev.courses,
                                basicSchedules: prev.basicSchedules,
                                weeklySchedules: {},
                                weeklyMemos: {},
                                statsRange: { start: `${year}-04-01`, end: `${year}-04-30` },
                                termSettings: prev.termSettings,
                                pmCutPeriod: prev.pmCutPeriod,
                                periodLabels: prev.periodLabels
                            };
                            showToast(`${year}年度のデータを新規作成しました（設定継承）`);
                        } catch (e) {}
                    }
                }

                if (!nextData) {
                    nextData = {
                        courses: INITIAL_COURSES,
                        basicSchedules: { A: {}, B: {}, C: {} },
                        weeklySchedules: {}, 
                        weeklyMemos: {}, 
                        statsRange: { start: `${year}-04-01`, end: `${year}-04-30` },
                        termSettings: INITIAL_TERMS,
                        pmCutPeriod: 5,
                        periodLabels: ['1', '2', '3', '4', '5', '6', '7']
                    };
                }

                setCourses(nextData.courses || INITIAL_COURSES);
                setBasicSchedules(nextData.basicSchedules || { A: {}, B: {}, C: {} });
                setWeeklySchedules(nextData.weeklySchedules || {});
                setWeeklyMemos(nextData.weeklyMemos || {});
                setPmCutPeriod(nextData.pmCutPeriod || 5);
                setPeriodLabels(nextData.periodLabels || ['1', '2', '3', '4', '5', '6', '7']);
                setTermSettings(nextData.termSettings || INITIAL_TERMS);
                setStatsRange(nextData.statsRange || { start: `${year}-04-01`, end: `${year}-04-30` });

                loadedFiscalYearRef.current = year;
                setIsDataReady(true);
            };

            const saveDataForYear = (year) => {
                if (!year) return;
                const data = {
                    courses,
                    basicSchedules,
                    weeklySchedules,
                    weeklyMemos,
                    statsRange,
                    termSettings,
                    pmCutPeriod,
                    periodLabels
                };
                localStorage.setItem(getDataKey(year), JSON.stringify(data));
            };

            useEffect(() => {
                if (!isLoaded) return;
                if (loadedFiscalYearRef.current !== currentFiscalYear) {
                    setIsDataReady(false);
                    if (loadedFiscalYearRef.current !== null) {
                        saveDataForYear(loadedFiscalYearRef.current);
                    }
                    loadDataForYear(currentFiscalYear);
                }
            }, [currentFiscalYear, isLoaded]);

            useEffect(() => {
                if (isLoaded && isDataReady && loadedFiscalYearRef.current === currentFiscalYear) {
                    saveDataForYear(currentFiscalYear);
                }
            }, [courses, basicSchedules, weeklySchedules, weeklyMemos, statsRange, termSettings, pmCutPeriod, periodLabels, currentFiscalYear, isLoaded, isDataReady]);

            const showToast = (msg) => {
                setToast(msg);
                setTimeout(() => setToast(null), 3000);
            };

            const goToNextWeek = () => setCurrentDate(addDays(currentDate, 7));
            const goToPrevWeek = () => setCurrentDate(addDays(currentDate, -7));
            const goToThisWeek = () => setCurrentDate(getMonday(new Date()));
            const jumpToDate = (targetDate) => setCurrentDate(getMonday(targetDate));

            const currentMondayKey = formatDateKey(currentDate);
            const currentWeekData = weeklySchedules[currentMondayKey] || {};
            const currentWeekMemos = weeklyMemos[currentMondayKey] || {};

            const updateSchedule = (dayIndex, periodIndex, courseId) => {
                const key = `${dayIndex}-${periodIndex}`;
                const newSchedule = { ...weeklySchedules };
                if (!newSchedule[currentMondayKey]) newSchedule[currentMondayKey] = {};
                if (courseId === null) delete newSchedule[currentMondayKey][key];
                else newSchedule[currentMondayKey][key] = courseId;
                setWeeklySchedules(newSchedule);
            };

            const updateMemo = (dayIndex, periodIndex, text) => {
                const key = `${dayIndex}-${periodIndex}`;
                const newMemos = { ...weeklyMemos };
                if (!newMemos[currentMondayKey]) newMemos[currentMondayKey] = {};
                if (!text || text.trim() === "") delete newMemos[currentMondayKey][key];
                else newMemos[currentMondayKey][key] = text;
                setWeeklyMemos(newMemos);
            };

            // Direct Memo Update for global editing
            const updateMemoDirect = (mondayKey, dayIndex, period, text) => {
                const key = `${dayIndex}-${period}`;
                const newMemos = { ...weeklyMemos };
                if (!newMemos[mondayKey]) newMemos[mondayKey] = {};
                if (!text || text.trim() === "") delete newMemos[mondayKey][key];
                else newMemos[mondayKey][key] = text;
                setWeeklyMemos(newMemos);
            };

            const updateDailyMemo = (dayIndex, text) => {
                const key = `daily-${dayIndex}`;
                const newMemos = { ...weeklyMemos };
                if (!newMemos[currentMondayKey]) newMemos[currentMondayKey] = {};
                if (!text || text.trim() === "") delete newMemos[currentMondayKey][key];
                else newMemos[currentMondayKey][key] = text;
                setWeeklyMemos(newMemos);
            };

            const copyPrevWeek = () => {
                const prevMonday = addDays(currentDate, -7);
                const prevKey = formatDateKey(prevMonday);
                if (weeklySchedules[prevKey]) {
                    if (confirm("先週の時間割を今週にコピーしますか？（現在の入力は上書きされます）")) {
                        const newSchedule = { ...weeklySchedules };
                        newSchedule[currentMondayKey] = {}; 
                        let skipped = [];
                        const daysMap = ['月', '火', '水', '木', '金', '土'];
                        [0, 1, 2, 3, 4, 5].forEach(dayIndex => {
                            const targetDate = addDays(currentDate, dayIndex);
                            if (getHolidayName(targetDate)) {
                                skipped.push(`${daysMap[dayIndex]}曜`);
                            } else {
                                Object.keys(weeklySchedules[prevKey]).forEach(key => {
                                    if (key.startsWith(`${dayIndex}-`)) newSchedule[currentMondayKey][key] = weeklySchedules[prevKey][key];
                                });
                            }
                        });
                        setWeeklySchedules(newSchedule);
                        skipped.length > 0 ? showToast(`祝日(${skipped.length}日)を除外してコピーしました`) : showToast("先週の内容をコピーしました");
                    }
                } else {
                    alert("先週のデータがありません");
                }
            };

            const applyBasicSchedule = (patternKey) => {
                const targetBasic = basicSchedules[patternKey] || {};
                const newSchedule = { ...weeklySchedules };
                newSchedule[currentMondayKey] = {}; 
                let skipped = [];
                const daysMap = ['月', '火', '水', '木', '金', '土'];
                [0, 1, 2, 3, 4, 5].forEach(dayIndex => {
                    const targetDate = addDays(currentDate, dayIndex);
                    if (getHolidayName(targetDate)) {
                        skipped.push(`${daysMap[dayIndex]}曜`);
                    } else {
                        Object.keys(targetBasic).forEach(key => {
                            if (key.startsWith(`${dayIndex}-`)) newSchedule[currentMondayKey][key] = targetBasic[key];
                        });
                    }
                });
                setWeeklySchedules(newSchedule);
                skipped.length > 0 ? showToast(`パターン${patternKey}: 祝日(${skipped.length}日)を除外して適用しました`) : showToast(`基本時間割（パターン${patternKey}）を適用しました`);
            };

            const handlePmCut = () => {
                if(confirm(`今週の ${pmCutPeriod}限 以降をすべて空きコマにしますか？`)) {
                    const newSchedule = { ...weeklySchedules };
                    if (!newSchedule[currentMondayKey]) newSchedule[currentMondayKey] = {};
                    const currentWeek = newSchedule[currentMondayKey];
                    Object.keys(currentWeek).forEach(key => {
                        const parts = key.split('-');
                        if (parts.length === 2 && parseInt(parts[1], 10) >= pmCutPeriod) delete currentWeek[key];
                    });
                    setWeeklySchedules(newSchedule);
                    showToast(`${pmCutPeriod}限以降をカットしました`);
                }
            };

            const clearCurrentWeek = () => {
                if (confirm("今週のデータを全て空にしますか？")) {
                    const newSchedule = { ...weeklySchedules };
                    delete newSchedule[currentMondayKey];
                    setWeeklySchedules(newSchedule);
                    const newMemos = { ...weeklyMemos };
                    delete newMemos[currentMondayKey];
                    setWeeklyMemos(newMemos);
                    showToast("今週のデータをクリアしました");
                }
            };

            const handleImportData = (e) => {
                const file = e.target.files[0];
                if (!file) return;
                const reader = new FileReader();
                reader.onload = (event) => {
                    try {
                        const data = JSON.parse(event.target.result);
                        if (data && data.courses) { 
                            if(confirm("データを読み込みますか？\n現在の年度データが上書きされます。")) {
                                setCourses(data.courses);
                                if(data.basicSchedules) setBasicSchedules(data.basicSchedules);
                                if(data.weeklySchedules) setWeeklySchedules(data.weeklySchedules);
                                if(data.weeklyMemos) setWeeklyMemos(data.weeklyMemos);
                                if(data.statsRange) setStatsRange(data.statsRange);
                                if(data.termSettings) setTermSettings(data.termSettings);
                                if(data.pmCutPeriod) setPmCutPeriod(data.pmCutPeriod);
                                if(data.periodLabels) setPeriodLabels(data.periodLabels);
                                showToast("データの復元が完了しました");
                            }
                        } else { alert("無効なファイル形式です"); }
                    } catch (error) { alert("読み込みエラー"); }
                };
                reader.readAsText(file);
                e.target.value = '';
            };

            const handlePrint = () => window.print();
            
            const touchStartX = useRef(0);
            const handleTouchStart = (e) => touchStartX.current = e.touches[0].clientX;
            const handleTouchEnd = (e) => {
                const diff = touchStartX.current - e.changedTouches[0].clientX;
                if (Math.abs(diff) > 50) { 
                    if (diff > 0) goToNextWeek(); else goToPrevWeek();
                }
            };

            const renderContent = () => {
                switch (activeTab) {
                    case 'schedule': return <ScheduleView date={currentDate} scheduleData={currentWeekData} memoData={currentWeekMemos} courses={courses} periodLabels={periodLabels} onUpdate={updateSchedule} onUpdateMemo={updateMemo} onUpdateDailyMemo={updateDailyMemo} onPrev={goToPrevWeek} onNext={goToNextWeek} onToday={goToThisWeek} onJumpToDate={jumpToDate} onCopyPrev={copyPrevWeek} onApplyBasic={applyBasicSchedule} onClear={clearCurrentWeek} onPmCut={handlePmCut} touchHandlers={{ onTouchStart: handleTouchStart, onTouchEnd: handleTouchEnd }} />;
                    case 'basic': return <BasicScheduleView basicSchedules={basicSchedules} setBasicSchedules={setBasicSchedules} courses={courses} periodLabels={periodLabels} showToast={showToast} />;
                    case 'courses': return <CourseManager courses={courses} setCourses={setCourses} showToast={showToast} />;
                    case 'memos': return <MemoListView weeklySchedules={weeklySchedules} courses={courses} weeklyMemos={weeklyMemos} statsRange={statsRange} setStatsRange={setStatsRange} termSettings={termSettings} setTermSettings={setTermSettings} periodLabels={periodLabels} onUpdateMemo={updateMemoDirect} />;
                    case 'stats': return <StatsView weeklySchedules={weeklySchedules} courses={courses} basicSchedules={basicSchedules} weeklyMemos={weeklyMemos} onImport={handleImportData} statsRange={statsRange} setStatsRange={setStatsRange} termSettings={termSettings} setTermSettings={setTermSettings} periodLabels={periodLabels} currentYear={currentFiscalYear} />;
                    default: return null;
                }
            };

            if (!isLoaded) return <div className="flex h-screen items-center justify-center">読み込み中...</div>;

            return (
                <div className="main-container flex flex-col h-[100dvh] max-w-md mx-auto bg-white shadow-2xl overflow-hidden sm:max-w-full md:max-w-3xl lg:max-w-4xl">
                    {/* Header with Settings Icon - Only show on 'schedule' tab */}
                    {activeTab === 'schedule' && (
                        <div className="relative no-print">
                            <button onClick={() => setIsSettingsOpen(true)} className="absolute top-0 right-0 p-4 text-gray-500 z-20 hover:text-gray-700 settings-btn">
                                <Icon name="gear" size={24} weight="fill" />
                            </button>
                        </div>
                    )}

                    <main className="flex-1 overflow-hidden relative bg-gray-50 pt-safe-top">
                        {renderContent()}
                    </main>

                    <nav className="bg-white border-t border-gray-200 flex justify-around pb-safe z-50">
                        <NavButton icon="calendar" label="週案" active={activeTab === 'schedule'} onClick={() => setActiveTab('schedule')} />
                        <NavButton icon="table" label="基本時間割" active={activeTab === 'basic'} onClick={() => setActiveTab('basic')} />
                        <NavButton icon="list-checks" label="講座管理" active={activeTab === 'courses'} onClick={() => setActiveTab('courses')} />
                        <NavButton icon="notebook" label="授業メモ" active={activeTab === 'memos'} onClick={() => setActiveTab('memos')} />
                        <NavButton icon="chart-bar" label="集計" active={activeTab === 'stats'} onClick={() => setActiveTab('stats')} />
                    </nav>

                    {toast && <div className="toast absolute bottom-20 left-1/2 transform -translate-x-1/2 bg-gray-800 text-white px-4 py-2 rounded-full shadow-lg text-sm animate-bounce z-50">{toast}</div>}

                    {/* Global Settings Modal */}
                    <Modal isOpen={isSettingsOpen} onClose={() => setIsSettingsOpen(false)} title="全体設定">
                        <div className="space-y-4">
                            <div className="bg-gray-50 p-4 rounded-lg border text-center">
                                <p className="text-sm font-bold text-gray-700 mb-1">現在の表示年度: <span className="text-indigo-600 text-lg">{currentFiscalYear}年度</span></p>
                                <p className="text-xs text-gray-500">※カレンダーの日付を移動すると自動で切り替わります</p>
                            </div>
                            
                            {/* Period Labels */}
                            <div className="bg-gray-50 p-4 rounded-lg border">
                                <h4 className="font-bold text-gray-700 text-sm mb-2 flex items-center gap-2"><Icon name="list-numbers" /> 時限表記の設定</h4>
                                <div className="grid grid-cols-7 gap-2">
                                    {periodLabels.map((label, index) => (
                                        <div key={index} className="flex flex-col items-center">
                                            <span className="text-[10px] text-gray-400 mb-1">{index + 1}限</span>
                                            <input type="text" value={label} onChange={(e) => {
                                                const newLabels = [...periodLabels]; newLabels[index] = e.target.value; setPeriodLabels(newLabels);
                                            }} className="w-full text-center border rounded text-sm p-1 focus:border-indigo-500 focus:outline-none" />
                                        </div>
                                    ))}
                                </div>
                            </div>
                            
                            {/* Print */}
                            <div className="bg-gray-50 p-4 rounded-lg border">
                                <h4 className="font-bold text-gray-700 text-sm mb-2 flex items-center gap-2"><Icon name="printer" /> 印刷</h4>
                                <button onClick={() => { setIsSettingsOpen(false); setTimeout(handlePrint, 300); }} className="w-full py-2 bg-gray-800 text-white rounded font-bold shadow hover:bg-gray-700 flex items-center justify-center gap-2">
                                    <Icon name="printer" /> 週案を印刷する
                                </button>
                            </div>

                            {/* PM Cut */}
                            <div className="bg-gray-50 p-4 rounded-lg border">
                                <h4 className="font-bold text-gray-700 text-sm mb-2 flex items-center gap-2"><Icon name="scissors" /> PMカット設定</h4>
                                <div className="flex gap-2">
                                    {[4, 5, 6, 7].map(num => (
                                        <button key={num} onClick={() => setPmCutPeriod(num)} className={`flex-1 py-2 rounded border text-sm font-bold transition-colors ${pmCutPeriod === num ? 'bg-indigo-600 text-white border-indigo-600' : 'bg-white text-gray-600 hover:bg-gray-100'}`}>
                                            {periodLabels[num - 1] || num}限〜
                                        </button>
                                    ))}
                                </div>
                            </div>
                            <button onClick={() => setIsSettingsOpen(false)} className="w-full py-3 bg-gray-600 text-white rounded-lg font-bold shadow">閉じる</button>
                        </div>
                    </Modal>
                </div>
            );
        };

        const NavButton = ({ icon, label, active, onClick }) => (
            <button onClick={onClick} className={`flex flex-col items-center justify-center w-full py-2 ${active ? 'text-indigo-600' : 'text-gray-400'}`}>
                <Icon name={icon} size={24} weight={active ? "fill" : "regular"} />
                <span className={`${label.length > 4 ? 'text-[9px]' : 'text-[10px]'} mt-1 font-medium`}>{label}</span>
            </button>
        );

        const ActionButton = ({ icon, label, onClick, className = "" }) => (
            <button onClick={onClick} className={`flex items-center gap-1 px-3 py-1.5 bg-white border border-gray-300 rounded text-xs font-medium text-gray-700 shadow-sm active:bg-gray-100 ${className}`}>
                <Icon name={icon} size={14} /> {label}
            </button>
        );

        // ... ScheduleView, BasicScheduleView, CourseManager ...
        // (Identical to previous, omitted for brevity, ensure they are present in final output)
        const ScheduleView = ({ date, scheduleData, memoData, courses, periodLabels, onUpdate, onUpdateMemo, onUpdateDailyMemo, onPrev, onNext, onToday, onJumpToDate, onCopyPrev, onApplyBasic, onPmCut, onClear, touchHandlers }) => {
            const days = ['月', '火', '水', '木', '金', '土'];
            const periods = [1, 2, 3, 4, 5, 6, 7];
            const [selectedCell, setSelectedCell] = useState(null);
            const [dailyMemoModal, setDailyMemoModal] = useState(null); 
            const [applyPatternModal, setApplyPatternModal] = useState(false);
            const datePickerRef = useRef(null);
            const getCourse = (day, period) => { const id = scheduleData[`${day}-${period}`]; return courses.find(c => c.id === id); };
            const getMemo = (day, period) => memoData[`${day}-${period}`] || "";
            const getDailyMemo = (dayIndex) => memoData[`daily-${dayIndex}`] || "";
            const headerDates = useMemo(() => { const start = getMonday(date); return Array.from({ length: 6 }).map((_, i) => addDays(start, i)); }, [date]);
            const isToday = (d) => { const today = new Date(); return d.getDate() === today.getDate() && d.getMonth() === today.getMonth() && d.getFullYear() === today.getFullYear(); };
            const holidayNames = useMemo(() => headerDates.map(d => getHolidayName(d)), [headerDates]);
            return (
                <div className="h-full flex flex-col" {...touchHandlers}>
                    <div className="print-header"><h1>週間指導計画（{getWeekRangeString(date)}）</h1></div>
                    <div className="bg-white border-b p-2 flex items-center justify-between shadow-sm z-10 pr-12 no-print"> 
                        <div className="flex items-center justify-between flex-1 max-w-[340px]">
                            <button onClick={onPrev} className="p-2 hover:bg-gray-100 rounded-full flex-shrink-0 text-gray-600"><Icon name="caret-left" size={24} /></button>
                            <div className="text-center flex-1 mx-1 overflow-hidden relative group rounded hover:bg-gray-50 transition-colors cursor-pointer" onClick={() => datePickerRef.current?.showPicker ? datePickerRef.current.showPicker() : datePickerRef.current?.click()}>
                                <div className="text-sm font-bold text-gray-800 whitespace-nowrap overflow-hidden text-ellipsis flex items-center justify-center gap-1 py-1">
                                    {getYearMonthString(date)} <Icon name="caret-down" size={12} className="text-gray-400" />
                                </div>
                                <input type="date" ref={datePickerRef} className="absolute top-0 left-0 w-0 h-0 opacity-0" tabIndex={-1} onChange={(e) => { if(e.target.value) { onJumpToDate(new Date(e.target.value)); e.target.value = ''; } }} />
                            </div>
                            <button onClick={onNext} className="p-2 hover:bg-gray-100 rounded-full flex-shrink-0 text-gray-600"><Icon name="caret-right" size={24} /></button>
                        </div>
                        <button onClick={onToday} className="flex items-center gap-1 px-3 py-2 text-xs bg-indigo-600 text-white rounded-lg font-bold shadow-md hover:bg-indigo-700 active:scale-95 transition-all flex-shrink-0 ml-2"><Icon name="calendar-check" weight="bold" /> 今週へジャンプ</button>
                    </div>
                    <div className="flex gap-2 p-2 overflow-x-auto whitespace-nowrap bg-gray-50 border-b no-scrollbar no-print">
                        <ActionButton icon="copy" label="前週コピー" onClick={onCopyPrev} />
                        <ActionButton icon="arrow-square-in" label="基本時間割適用" onClick={() => setApplyPatternModal(true)} />
                        <ActionButton icon="scissors" label="PMカット" onClick={onPmCut} />
                        <ActionButton icon="trash" label="今週クリア" onClick={onClear} />
                    </div>
                    <div className="flex-1 overflow-auto bg-white relative pb-10">
                        <div className="timetable-grid min-w-[420px]">
                            <div className="timetable-header border-b border-r bg-gray-100"></div>
                            {days.map((day, i) => (
                                <div key={day} className={`timetable-header border-b ${isToday(headerDates[i]) ? 'bg-indigo-50 text-indigo-700' : ''}`}>
                                    <div className="flex flex-col items-center justify-center h-full py-1 relative w-full">
                                        {holidayNames[i] && <div className="absolute top-0 right-0 p-0.5"><div className="w-2 h-2 rounded-full bg-red-400" title={holidayNames[i]}></div></div>}
                                        <span className={`text-xs font-bold leading-none mb-1 ${holidayNames[i] ? 'text-red-500' : ''}`}>{formatDateMMDD(headerDates[i])}</span>
                                        <span className={`text-sm leading-none ${holidayNames[i] ? 'text-red-500' : ''}`}>{day}</span>
                                    </div>
                                </div>
                            ))}
                            {periods.map(period => (
                                <React.Fragment key={period}>
                                    <div className="period-header border-r">{periodLabels[period-1] !== undefined ? periodLabels[period-1] : period}</div>
                                    {days.map((_, dayIndex) => {
                                        const course = getCourse(dayIndex, period);
                                        const memo = getMemo(dayIndex, period);
                                        const isHol = !!holidayNames[dayIndex];
                                        return (
                                            <div key={`${dayIndex}-${period}`} className="timetable-cell" style={{ backgroundColor: isHol ? '#fff5f5' : (course ? course.color : 'white') }} onClick={() => setSelectedCell({ dayIndex, period, currentCourseId: course?.id, initialMemo: memo })}>
                                                {course ? <span className="font-semibold text-gray-800 leading-tight break-words w-full px-1">{course.name}</span> : <span className="text-gray-300 text-xs">-</span>}
                                                {memo && <span className="block mt-1 text-[9px] text-gray-600 bg-white/50 w-full truncate px-1 rounded"><Icon name="note-pencil" size={10} className="inline mr-0.5" />{memo}</span>}
                                            </div>
                                        );
                                    })}
                                </React.Fragment>
                            ))}
                            <div className="period-header border-r border-t bg-yellow-50 text-[10px]">メモ</div>
                            {days.map((_, dayIndex) => (
                                <div key={`daily-${dayIndex}`} className="daily-memo-cell border-t" onClick={() => setDailyMemoModal({ dayIndex, text: getDailyMemo(dayIndex) })}>
                                    {getDailyMemo(dayIndex) ? <span className="w-full break-words">{getDailyMemo(dayIndex)}</span> : <span className="text-gray-300 text-xl">+</span>}
                                </div>
                            ))}
                        </div>
                    </div>
                    <Modal isOpen={!!selectedCell} onClose={() => setSelectedCell(null)} title={`${days[selectedCell?.dayIndex]}曜 ${periodLabels[selectedCell?.period - 1] || selectedCell?.period}限 選択`}>
                        <div className="space-y-4">
                            <div className="bg-yellow-50 p-3 rounded-lg border border-yellow-100">
                                <label className="block text-xs font-bold text-yellow-800 mb-1 flex items-center gap-1"><Icon name="note-pencil" /> 授業メモ</label>
                                <textarea className="w-full p-2 text-sm border rounded focus:ring-2 focus:ring-indigo-500 outline-none" rows="2" placeholder="ここに入力すると自動保存されます" defaultValue={selectedCell?.initialMemo || ""} onChange={(e) => onUpdateMemo(selectedCell.dayIndex, selectedCell.period, e.target.value)}></textarea>
                            </div>
                            <div className="grid grid-cols-2 gap-2">
                                <button onClick={() => { onUpdate(selectedCell.dayIndex, selectedCell.period, null); setSelectedCell(null); }} className="col-span-2 p-3 border-2 border-dashed border-gray-300 rounded-lg text-gray-500 hover:bg-gray-50 flex items-center justify-center gap-2"><Icon name="eraser" /> 空きコマにする</button>
                                {courses.map(course => (
                                    <button key={course.id} onClick={() => { onUpdate(selectedCell.dayIndex, selectedCell.period, course.id); setSelectedCell(null); }} className="p-3 rounded-lg text-sm font-bold shadow-sm text-gray-800 border" style={{ backgroundColor: course.color, borderColor: 'rgba(0,0,0,0.05)' }}>{course.name}</button>
                                ))}
                            </div>
                        </div>
                    </Modal>
                    <Modal isOpen={!!dailyMemoModal} onClose={() => setDailyMemoModal(null)} title={`${formatDateMMDD(addDays(getMonday(date), dailyMemoModal?.dayIndex || 0))}のメモ`}>
                        <div className="space-y-4">
                            <p className="text-xs text-gray-500">学校行事、放課後の予定などを入力してください。</p>
                            <textarea className="w-full p-3 text-base border rounded-lg focus:ring-2 focus:ring-indigo-500 outline-none h-32" placeholder="例：14:00〜三者懇談&#13;&#10;短縮授業45分" defaultValue={dailyMemoModal?.text || ""} autoFocus onChange={(e) => onUpdateDailyMemo(dailyMemoModal.dayIndex, e.target.value)}></textarea>
                            <button onClick={() => setDailyMemoModal(null)} className="w-full py-3 bg-indigo-600 text-white rounded-lg font-bold shadow">閉じる（保存）</button>
                        </div>
                    </Modal>
                    <Modal isOpen={applyPatternModal} onClose={() => setApplyPatternModal(false)} title="基本時間割を適用">
                        <div className="space-y-3">
                            <p className="text-sm text-gray-600 mb-2">適用するパターンを選択してください。<br/><span className="text-xs text-red-500">※現在の入力内容は上書きされます。</span></p>
                            {['A', 'B', 'C'].map(pattern => (
                                <button key={pattern} onClick={() => { if (confirm(`パターン${pattern}を今週に適用しますか？`)) { onApplyBasic(pattern); setApplyPatternModal(false); } }} className="w-full flex items-center justify-between p-4 border rounded-lg hover:bg-gray-50 font-bold text-gray-700"><span>パターン {pattern}</span><Icon name="caret-right" /></button>
                            ))}
                        </div>
                    </Modal>
                </div>
            );
        };

        const BasicScheduleView = ({ basicSchedules, setBasicSchedules, courses, periodLabels, showToast }) => {
            const [currentPattern, setCurrentPattern] = useState('A');
            const [selectedCell, setSelectedCell] = useState(null);
            const days = ['月', '火', '水', '木', '金', '土'];
            const periods = [1, 2, 3, 4, 5, 6, 7];
            const currentSchedule = basicSchedules[currentPattern] || {};
            const updateBasic = (day, period, courseId) => {
                const key = `${day}-${period}`;
                const newSchedule = { ...currentSchedule };
                if (courseId === null) delete newSchedule[key];
                else newSchedule[key] = courseId;
                setBasicSchedules({ ...basicSchedules, [currentPattern]: newSchedule });
            };
            const resetCurrentPattern = () => {
                if (confirm(`基本時間割（パターン${currentPattern}）をリセットしてもよろしいですか？`)) {
                    setBasicSchedules({ ...basicSchedules, [currentPattern]: {} });
                    if(showToast) showToast(`パターン${currentPattern}をリセットしました`);
                }
            };
            const getCourse = (day, period) => { const id = currentSchedule[`${day}-${period}`]; return courses.find(c => c.id === id); };
            return (
                <div className="h-full flex flex-col">
                    <div className="p-4 bg-yellow-50 border-b border-yellow-100 shadow-sm"><h2 className="text-sm font-bold text-yellow-800 flex items-center gap-2 mb-2"><Icon name="info" /> 基本時間割の設定</h2><div className="flex gap-1 bg-yellow-200/50 p-1 rounded-lg">{['A', 'B', 'C'].map(pattern => <button key={pattern} onClick={() => setCurrentPattern(pattern)} className={`flex-1 py-1.5 text-xs font-bold rounded-md transition-colors ${currentPattern === pattern ? 'bg-white text-yellow-900 shadow-sm' : 'text-yellow-700 hover:bg-yellow-100'}`}>パターン {pattern}</button>)}</div></div>
                    <div className="flex-1 overflow-auto bg-white pb-20"><div className="flex justify-end p-2"><button onClick={resetCurrentPattern} className="text-xs text-red-500 underline flex items-center gap-1"><Icon name="trash" /> パターン{currentPattern}をリセット</button></div><div className="timetable-grid min-w-[420px]"><div className="timetable-header border-b border-r bg-gray-100"></div>{days.map(day => <div key={day} className="timetable-header border-b bg-gray-50">{day}</div>)}{periods.map(period => <React.Fragment key={period}><div className="period-header border-r">{periodLabels[period-1] !== undefined ? periodLabels[period-1] : period}</div>{days.map((_, dayIndex) => { const course = getCourse(dayIndex, period); return <div key={`${dayIndex}-${period}`} className="timetable-cell" style={{ backgroundColor: course ? course.color : 'white' }} onClick={() => setSelectedCell({ dayIndex, period })}>{course ? <span className="font-semibold text-gray-800">{course.name}</span> : <span className="text-gray-300 text-xs">-</span>}</div>; })}</React.Fragment>)}</div></div>
                     <Modal isOpen={!!selectedCell} onClose={() => setSelectedCell(null)} title={`${days[selectedCell?.dayIndex]}曜 ${periodLabels[selectedCell?.period - 1] || selectedCell?.period}限 (基本パターン${currentPattern})`}><div className="grid grid-cols-2 gap-2"><button onClick={() => { updateBasic(selectedCell.dayIndex, selectedCell.period, null); setSelectedCell(null); }} className="col-span-2 p-3 border-2 border-dashed border-gray-300 rounded-lg text-gray-500 hover:bg-gray-50 flex items-center justify-center gap-2"><Icon name="eraser" /> 設定なし</button>{courses.map(course => <button key={course.id} onClick={() => { updateBasic(selectedCell.dayIndex, selectedCell.period, course.id); setSelectedCell(null); }} className="p-3 rounded-lg text-sm font-bold shadow-sm text-gray-800 border" style={{ backgroundColor: course.color }}>{course.name}</button>)}</div></Modal>
                </div>
            );
        };

        const CourseManager = ({ courses, setCourses, showToast }) => {
            const [isEditing, setIsEditing] = useState(null);
            const [editName, setEditName] = useState("");
            const [editColor, setEditColor] = useState(PRESET_COLORS[0]);
            const handleSave = () => { if (!editName.trim()) return; if (isEditing === 'NEW') { setCourses([...courses, { id: 'c' + Date.now(), name: editName, color: editColor }]); } else { setCourses(courses.map(c => c.id === isEditing ? { ...c, name: editName, color: editColor } : c)); } setIsEditing(null); setEditName(""); };
            const handleDelete = (id) => { if (confirm("削除しますか？")) setCourses(courses.filter(c => c.id !== id)); };
            const resetCourses = () => { if(confirm("リセットしますか？")) { setCourses(INITIAL_COURSES); if(showToast) showToast("リセットしました"); } };
            const moveCourse = (index, dir) => { const newC = [...courses]; if (dir === 'up' && index > 0) [newC[index], newC[index-1]] = [newC[index-1], newC[index]]; else if (dir === 'down' && index < newC.length-1) [newC[index], newC[index+1]] = [newC[index+1], newC[index]]; setCourses(newC); };
            return (
                <div className="h-full flex flex-col bg-gray-50">
                    <div className="p-4 bg-white border-b shadow-sm sticky top-0 z-10 flex justify-between items-center"><h2 className="text-lg font-bold text-gray-800">登録済み講座一覧</h2><div className="flex gap-2"><button onClick={resetCourses} className="text-red-500 text-xs underline p-2">リセット</button><button onClick={() => { setIsEditing('NEW'); setEditName(""); setEditColor(PRESET_COLORS[1]); }} className="bg-indigo-600 text-white px-4 py-2 rounded-lg text-sm font-bold shadow hover:bg-indigo-700 flex items-center gap-2"><Icon name="plus" weight="bold" /> 新規追加</button></div></div>
                    <div className="p-4 space-y-3 overflow-y-auto pb-20">{courses.map((course, index) => <div key={course.id} className="bg-white p-3 rounded-xl shadow-sm border flex items-center justify-between"><div className="flex items-center gap-3"><div className="flex flex-col gap-1 mr-1"><button onClick={() => moveCourse(index, 'up')} disabled={index === 0} className={`p-1 rounded hover:bg-gray-100 ${index === 0 ? 'text-gray-200' : 'text-gray-500'}`}><Icon name="caret-up" size={16} /></button><button onClick={() => moveCourse(index, 'down')} disabled={index === courses.length - 1} className={`p-1 rounded hover:bg-gray-100 ${index === courses.length - 1 ? 'text-gray-200' : 'text-gray-500'}`}><Icon name="caret-down" size={16} /></button></div><div className="w-10 h-10 rounded-full border shadow-inner" style={{ backgroundColor: course.color }}></div><span className="font-bold text-gray-800 text-lg">{course.name}</span></div><div className="flex gap-2"><button onClick={() => { setIsEditing(course.id); setEditName(course.name); setEditColor(course.color); }} className="p-2 text-indigo-600 hover:bg-indigo-50 rounded"><Icon name="pencil-simple" size={20} /></button><button onClick={() => handleDelete(course.id)} className="p-2 text-red-500 hover:bg-red-50 rounded"><Icon name="trash" size={20} /></button></div></div>)}</div>
                    <Modal isOpen={!!isEditing} onClose={() => setIsEditing(null)} title={isEditing === 'NEW' ? "講座の新規登録" : "講座の編集"}><div className="space-y-4"><div><label className="block text-sm font-bold text-gray-700 mb-1">講座名</label><input type="text" className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-indigo-500 outline-none text-lg" value={editName} onChange={(e) => setEditName(e.target.value)} autoFocus /></div><div><label className="block text-sm font-bold text-gray-700 mb-2">表示色</label><div className="grid grid-cols-5 gap-3">{PRESET_COLORS.map(color => <button key={color} onClick={() => setEditColor(color)} className={`w-10 h-10 rounded-full border-2 shadow-sm ${editColor === color ? 'border-indigo-600 scale-110 ring-2 ring-indigo-200' : 'border-transparent'}`} style={{ backgroundColor: color }} />)}</div></div><div className="pt-4 flex gap-3"><button onClick={() => setIsEditing(null)} className="flex-1 py-3 bg-gray-100 text-gray-700 rounded-lg font-bold">キャンセル</button><button onClick={handleSave} className="flex-1 py-3 bg-indigo-600 text-white rounded-lg font-bold shadow hover:bg-indigo-700">保存する</button></div></div></Modal>
                </div>
            );
        };

        // Update: Added onUpdateMemo prop to handle live editing
        const MemoListView = ({ weeklySchedules, courses, weeklyMemos, statsRange, setStatsRange, termSettings, setTermSettings, periodLabels, onUpdateMemo }) => {
            const startDate = statsRange.start;
            const endDate = statsRange.end;
            const [selectedCourse, setSelectedCourse] = useState(null);
            const [isSettingTerms, setIsSettingTerms] = useState(false);

            // Dynamically calculate memos based on current props
            const memos = useMemo(() => {
                if (!selectedCourse) return []; // use state object for selection, but filter by ID
                const courseId = selectedCourse.id;
                const list = [];
                // Sort keys to ensure chronological order
                const sortedKeys = Object.keys(weeklySchedules).sort();
                
                sortedKeys.forEach(mondayKey => {
                    const mondayDate = new Date(mondayKey);
                    [0, 1, 2, 3, 4, 5].forEach(dayIndex => {
                        const currentDate = addDays(mondayDate, dayIndex);
                        const dateKey = formatDateKey(currentDate);
                        
                        // Filter by date range
                        if (dateKey >= startDate && dateKey <= endDate) {
                            [1, 2, 3, 4, 5, 6, 7].forEach(period => {
                                const key = `${dayIndex}-${period}`;
                                const weekData = weeklySchedules[mondayKey];
                                
                                // Check if this slot belongs to the selected course
                                // FIX: Removed requirement for existing memo to allow input
                                if (weekData && weekData[key] === courseId) {
                                    list.push({
                                        mondayKey: mondayKey, // Needed for update
                                        dayIndex: dayIndex,   // Needed for update
                                        dateStr: formatDateMMDD(currentDate),
                                        period: period,
                                        // Use existing memo or empty string
                                        text: weeklyMemos[mondayKey]?.[key] || "" 
                                    });
                                }
                            });
                        }
                    });
                });
                return list;
            }, [selectedCourse, weeklySchedules, weeklyMemos, startDate, endDate]);

            // Need to update selectedCourse ref when memos change? 
            // Better to separate selection state from data.
            // But for modal, we can use the memoized list.

            const setRange = (type) => {
                const now = new Date();
                const m = getMonday(now);
                if (type === 'today') {
                     setStatsRange({ ...statsRange, start: formatDateKey(now) });
                }
            };

            const applyTerm = (term) => { if (!term.start || !term.end) { alert("期間を設定してください"); setIsSettingTerms(true); return; } setStatsRange({ start: term.start, end: term.end }); };

            return (
                <div className="h-full flex flex-col bg-white p-4 overflow-y-auto pb-20">
                    <h2 className="text-lg font-bold mb-4 flex items-center gap-2"><Icon name="notebook" /> 授業メモ確認</h2>
                    
                    {/* Range Selection Reuse from StatsView */}
                    <div className="mb-4">
                        <div className="flex justify-between items-center mb-2">
                            <span className="text-xs font-bold text-gray-500">期間プリセット</span>
                            {/* Update: Changed label from "設定" to "期間設定" */}
                            <button onClick={() => setIsSettingTerms(true)} className="text-indigo-600 text-xs flex items-center gap-1">
                                <Icon name="gear" /> 期間設定
                            </button>
                        </div>
                        <div className="flex flex-wrap gap-2">
                            {termSettings.map(term => (
                                <button
                                    key={term.id}
                                    onClick={() => applyTerm(term)}
                                    className={`px-3 py-1.5 rounded-lg text-xs font-bold border transition-colors ${
                                        statsRange.start === term.start && statsRange.end === term.end
                                        ? 'bg-indigo-600 text-white border-indigo-600'
                                        : 'bg-white text-gray-700 border-gray-200 hover:bg-gray-50'
                                    }`}
                                >
                                    {term.name}
                                </button>
                            ))}
                        </div>
                    </div>

                    <div className="bg-gray-50 p-4 rounded-xl border mb-6">
                        <div className="flex gap-2 mb-4">
                            <button onClick={() => setRange('today')} className="flex-1 py-1 text-xs bg-white border rounded shadow-sm hover:bg-gray-50">今日</button>
                        </div>
                        <div className="flex items-center gap-2 justify-between mb-2">
                            <input 
                                type="date" 
                                value={startDate} 
                                onChange={e => setStatsRange({...statsRange, start: e.target.value})} 
                                className="border p-1 rounded text-sm w-[45%]" 
                            />
                            <span className="text-gray-400">~</span>
                            <input 
                                type="date" 
                                value={endDate} 
                                onChange={e => setStatsRange({...statsRange, end: e.target.value})} 
                                className="border p-1 rounded text-sm w-[45%]" 
                            />
                        </div>
                    </div>

                    <div className="space-y-3">
                        {courses.map(course => (
                            <div 
                                key={course.id} 
                                className="bg-white p-3 rounded-xl shadow-sm border flex items-center justify-between cursor-pointer hover:bg-gray-50 transition-colors"
                                onClick={() => setSelectedCourse(course)}
                            >
                                <div className="flex items-center gap-3">
                                    <div className="w-10 h-10 rounded-full border shadow-inner" style={{ backgroundColor: course.color }}></div>
                                    <span className="font-bold text-gray-800 text-lg">{course.name}</span>
                                </div>
                                <Icon name="caret-right" className="text-gray-400" />
                            </div>
                        ))}
                    </div>

                    {/* Memo Detail Modal - Now Editable */}
                    <Modal 
                        isOpen={!!selectedCourse} 
                        onClose={() => setSelectedCourse(null)} 
                        title={`${selectedCourse?.name} のメモ`}
                    >
                        <div className="space-y-4">
                            <p className="text-xs text-gray-500">期間: {startDate} 〜 {endDate}</p>
                            {memos.length > 0 ? <div className="space-y-3">
                                {memos.map((memo, idx) => (
                                    <div key={idx} className="bg-yellow-50 p-3 rounded-lg border border-yellow-100 shadow-sm">
                                        <div className="flex justify-between items-center mb-1">
                                            {/* Update: Use custom label */}
                                            <span className="font-bold text-sm text-yellow-900">{memo.dateStr} <span className="text-xs font-normal">({periodLabels[memo.period - 1] || memo.period}限)</span></span>
                                        </div>
                                        {/* Editable Textarea */}
                                        <textarea 
                                            className="w-full p-2 text-sm bg-white border rounded focus:ring-2 focus:ring-indigo-500 outline-none"
                                            rows="2"
                                            value={memo.text}
                                            onChange={(e) => onUpdateMemo(memo.mondayKey, memo.dayIndex, memo.period, e.target.value)}
                                        />
                                    </div>
                                ))}
                            </div> : <div className="text-center py-8 text-gray-400"><Icon name="notebook" size={48} className="mx-auto mb-2 opacity-20" /><p>期間内に授業はありません</p></div>}
                            <button 
                                onClick={() => setSelectedCourse(null)}
                                className="w-full py-3 bg-gray-100 text-gray-700 rounded-lg font-bold mt-4"
                            >
                                閉じる
                            </button>
                        </div>
                    </Modal>

                    {/* Reuse Term Settings Modal logic via props update if needed, but for simplicity reusing logic inside component */}
                    <Modal
                        isOpen={isSettingTerms}
                        onClose={() => setIsSettingTerms(false)}
                        title="集計期間の設定"
                    >
                        <div className="space-y-4">
                            <p className="text-xs text-gray-500">よく使う集計期間（学期など）を設定してください。</p>
                            {termSettings.map((term, index) => (
                                <div key={term.id} className="bg-gray-50 p-3 rounded-lg border">
                                    <div className="flex justify-between items-center mb-2">
                                        <input 
                                            type="text" 
                                            value={term.name}
                                            onChange={(e) => {
                                                const newTerms = [...termSettings];
                                                newTerms[index].name = e.target.value;
                                                setTermSettings(newTerms);
                                            }}
                                            className="font-bold text-sm bg-transparent border-b border-dashed border-gray-400 w-full mr-2"
                                        />
                                    </div>
                                    <div className="flex items-center gap-2">
                                        <input 
                                            type="date" 
                                            value={term.start}
                                            onChange={(e) => {
                                                const newTerms = [...termSettings];
                                                newTerms[index].start = e.target.value;
                                                setTermSettings(newTerms);
                                            }}
                                            className="border rounded px-1 py-1 text-xs flex-1"
                                        />
                                        <span className="text-gray-400">~</span>
                                        <input 
                                            type="date" 
                                            value={term.end}
                                            onChange={(e) => {
                                                const newTerms = [...termSettings];
                                                newTerms[index].end = e.target.value;
                                                setTermSettings(newTerms);
                                            }}
                                            className="border rounded px-1 py-1 text-xs flex-1"
                                        />
                                    </div>
                                </div>
                            ))}
                            <button 
                                onClick={() => setIsSettingTerms(false)}
                                className="w-full py-3 bg-indigo-600 text-white rounded-lg font-bold shadow"
                            >
                                設定を完了
                            </button>
                        </div>
                    </Modal>
                </div>
            );
        };

        const StatsView = ({ weeklySchedules, courses, basicSchedules, weeklyMemos, onImport, statsRange, setStatsRange, termSettings, setTermSettings, periodLabels, currentYear }) => {
            const startDate = statsRange.start;
            const endDate = statsRange.end;
            const [isSettingTerms, setIsSettingTerms] = useState(false);
            const fileInputRef = useRef(null);

            const stats = useMemo(() => {
                const counts = {};
                const dates = {}; 
                courses.forEach(c => { counts[c.id] = 0; dates[c.id] = []; });
                let total = 0;
                Object.keys(weeklySchedules).sort().forEach(mondayKey => {
                    const mondayDate = new Date(mondayKey);
                    [0, 1, 2, 3, 4, 5].forEach(dayIndex => {
                        const currentDay = addDays(mondayDate, dayIndex);
                        const dateKey = formatDateKey(currentDay);
                        if (dateKey >= startDate && dateKey <= endDate) {
                            [1, 2, 3, 4, 5, 6, 7].forEach(period => {
                                if (weeklySchedules[mondayKey]?.[`${dayIndex}-${period}`] && counts[weeklySchedules[mondayKey][`${dayIndex}-${period}`]] !== undefined) {
                                    const cId = weeklySchedules[mondayKey][`${dayIndex}-${period}`];
                                    counts[cId]++;
                                    dates[cId].push(formatDateMMDD(currentDay));
                                    total++;
                                }
                            });
                        }
                    });
                });
                return { counts, dates, total };
            }, [weeklySchedules, courses, startDate, endDate]);

            const copyStats = () => {
                let text = `【${currentYear}年度 授業時数集計】\n期間：${startDate} 〜 ${endDate}\n\n`;
                courses.forEach(c => { if (stats.counts[c.id] > 0) text += `${c.name}: ${stats.counts[c.id]}コマ (${stats.dates[c.id].join(', ')})\n`; });
                text += `\n合計: ${stats.total}コマ`;
                navigator.clipboard.writeText(text).then(() => alert("コピーしました"));
            };

            const exportData = () => {
                const data = { courses, basicSchedules, weeklySchedules, weeklyMemos, statsRange, termSettings, version: 'v31', exportDate: new Date().toISOString() };
                const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
                const url = URL.createObjectURL(blob);
                const a = document.createElement('a'); a.href = url; a.download = `teacher_schedule_backup_${currentYear}_${new Date().toISOString().slice(0,10)}.json`; document.body.appendChild(a); a.click(); document.body.removeChild(a); URL.revokeObjectURL(url);
            };

            const setRange = (type) => {
                const now = new Date();
                const m = getMonday(now);
                if (type === 'week') {
                    setStatsRange({
                        start: formatDateKey(m),
                        end: formatDateKey(addDays(m, 5))
                    });
                } else if (type === 'month') {
                    const firstDay = new Date(now.getFullYear(), now.getMonth(), 1);
                    const lastDay = new Date(now.getFullYear(), now.getMonth() + 1, 0);
                    setStatsRange({
                        start: formatDateKey(firstDay),
                        end: formatDateKey(lastDay)
                    });
                } else if (type === 'today') {
                    setStatsRange({ ...statsRange, start: formatDateKey(now) });
                }
            };

            const applyTerm = (term) => { if (!term.start || !term.end) { alert("期間を設定してください"); setIsSettingTerms(true); return; } setStatsRange({ start: term.start, end: term.end }); };
            
            return (
                <div className="h-full flex flex-col bg-white p-4 overflow-y-auto pb-20">
                    <h2 className="text-lg font-bold mb-4 flex items-center gap-2"><Icon name="chart-pie-slice" /> 時数集計 ({currentYear}年度)</h2>
                    
                    {/* Update: Term Preset Section */}
                    <div className="mb-4">
                        <div className="flex justify-between items-center mb-2">
                            <span className="text-xs font-bold text-gray-500">期間プリセット</span>
                            {/* Update: Changed label from "設定" to "期間設定" */}
                            <button onClick={() => setIsSettingTerms(true)} className="text-indigo-600 text-xs flex items-center gap-1">
                                <Icon name="gear" /> 期間設定
                            </button>
                        </div>
                        <div className="flex flex-wrap gap-2">
                            {termSettings.map(term => (
                                <button
                                    key={term.id}
                                    onClick={() => applyTerm(term)}
                                    className={`px-3 py-1.5 rounded-lg text-xs font-bold border transition-colors ${
                                        statsRange.start === term.start && statsRange.end === term.end
                                        ? 'bg-indigo-600 text-white border-indigo-600'
                                        : 'bg-white text-gray-700 border-gray-200 hover:bg-gray-50'
                                    }`}
                                >
                                    {term.name}
                                </button>
                            ))}
                        </div>
                    </div>

                    <div className="bg-gray-50 p-4 rounded-xl border mb-6">
                        <div className="flex gap-2 mb-4">
                            <button onClick={() => setRange('week')} className="flex-1 py-1 text-xs bg-white border rounded shadow-sm hover:bg-gray-50">今週</button>
                            <button onClick={() => setRange('month')} className="flex-1 py-1 text-xs bg-white border rounded shadow-sm hover:bg-gray-50">今月</button>
                            {/* Update: Added "Today" button */}
                            <button onClick={() => setRange('today')} className="flex-1 py-1 text-xs bg-white border rounded shadow-sm hover:bg-gray-50">今日</button>
                        </div>
                        <div className="flex items-center gap-2 justify-between mb-2">
                            <input 
                                type="date" 
                                value={startDate} 
                                onChange={e => setStatsRange({...statsRange, start: e.target.value})} 
                                className="border p-1 rounded text-sm w-[45%]" 
                            />
                            <span className="text-gray-400">~</span>
                            <input 
                                type="date" 
                                value={endDate} 
                                onChange={e => setStatsRange({...statsRange, end: e.target.value})} 
                                className="border p-1 rounded text-sm w-[45%]" 
                            />
                        </div>
                    </div>

                    <div className="space-y-4">{courses.filter(c => stats.counts[c.id] > 0).sort((a, b) => stats.counts[b.id] - stats.counts[a.id]).map(c => <div key={c.id} className="flex items-start gap-3 border-b pb-3 last:border-0"><div className="w-3 h-10 rounded-full flex-shrink-0 mt-1" style={{ backgroundColor: c.color }}></div><div className="flex-1"><div className="flex justify-between items-baseline mb-1"><span className="font-bold text-gray-800">{c.name}</span><span className="font-bold text-lg">{stats.counts[c.id]}<span className="text-xs font-normal text-gray-500 ml-1">コマ</span></span></div><div className="text-[10px] text-gray-500 leading-tight">{stats.dates[c.id].join(', ')}</div></div></div>)}</div>
                    <button onClick={copyStats} className="mt-8 w-full py-3 bg-gray-800 text-white rounded-lg font-bold flex items-center justify-center gap-2 shadow hover:bg-gray-700"><Icon name="copy" /> 結果をコピー</button>
                    <div className="mt-8 pt-8 border-t"><h3 className="text-xs font-bold text-gray-400 mb-4">データ管理</h3><div className="grid grid-cols-2 gap-3 mb-6"><button onClick={exportData} className="flex items-center justify-center gap-2 py-2 px-4 bg-green-600 text-white rounded-lg font-bold text-sm shadow hover:bg-green-700"><Icon name="download-simple" /> エクスポート</button><button onClick={() => fileInputRef.current.click()} className="flex items-center justify-center gap-2 py-2 px-4 bg-blue-600 text-white rounded-lg font-bold text-sm shadow hover:bg-blue-700"><Icon name="upload-simple" /> インポート</button><input type="file" ref={fileInputRef} className="hidden" accept=".json" onChange={onImport} /></div><button onClick={() => { if(confirm('本当に全てのデータを削除しますか？')) { localStorage.clear(); window.location.reload(); } }} className="text-red-500 text-xs underline block w-full text-center">全てのデータをリセット</button></div>
                    <Modal isOpen={isSettingTerms} onClose={() => setIsSettingTerms(false)} title="集計期間の設定"><div className="space-y-4"><p className="text-xs text-gray-500">よく使う集計期間を設定してください。</p>{termSettings.map((term, index) => <div key={term.id} className="bg-gray-50 p-3 rounded-lg border"><div className="flex justify-between items-center mb-2"><input type="text" value={term.name} onChange={(e) => { const newTerms = [...termSettings]; newTerms[index].name = e.target.value; setTermSettings(newTerms); }} className="font-bold text-sm bg-transparent border-b border-dashed border-gray-400 w-full mr-2" /></div><div className="flex items-center gap-2"><input type="date" value={term.start} onChange={(e) => { const newTerms = [...termSettings]; newTerms[index].start = e.target.value; setTermSettings(newTerms); }} className="border rounded px-1 py-1 text-xs flex-1" /><span className="text-gray-400">~</span><input type="date" value={term.end} onChange={(e) => { const newTerms = [...termSettings]; newTerms[index].end = e.target.value; setTermSettings(newTerms); }} className="border rounded px-1 py-1 text-xs flex-1" /></div></div>)}<button onClick={() => setIsSettingTerms(false)} className="w-full py-3 bg-indigo-600 text-white rounded-lg font-bold shadow">設定を完了</button></div></Modal>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
