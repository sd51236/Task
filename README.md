# 代辦
import React, { useState, useEffect } from 'react';
import { 
  Plus, 
  Trash2, 
  CheckCircle2, 
  Circle, 
  Clock, 
  Calendar,
  Flame, 
  Zap, 
  Diamond,
  Leaf, 
  Ghost,
  X,
  Sparkles,
  Loader2
} from 'lucide-react';

// 優先級定義
const PRIORITIES = {
  CRITICAL: { id: 'critical', label: '急迫且重要', color: 'text-rose-500', bg: 'bg-rose-500/20', border: 'border-rose-500', icon: Flame },
  URGENT: { id: 'urgent', label: '急迫但不重要', color: 'text-amber-500', bg: 'bg-amber-500/20', border: 'border-amber-500', icon: Zap },
  IMPORTANT: { id: 'important', label: '重要但不急迫', color: 'text-violet-500', bg: 'bg-violet-500/20', border: 'border-violet-500', icon: Diamond },
  NORMAL: { id: 'normal', label: '一般雜務', color: 'text-emerald-500', bg: 'bg-emerald-500/20', border: 'border-emerald-500', icon: Leaf },
};

const App = () => {
  // 初始化任務
  const [tasks, setTasks] = useState(() => {
    try {
      const saved = localStorage.getItem('vibe_tasks_v2');
      return saved ? JSON.parse(saved) : [];
    } catch (e) {
      return [];
    }
  });
  
  const [isAiLoading, setIsAiLoading] = useState(false);
  const [ghostMessage, setGhostMessage] = useState('未完成的過去...');

  // 獲取今天的日期字串 YYYY-MM-DD
  const getTodayString = () => new Date().toISOString().split('T')[0];

  const [newTask, setNewTask] = useState('');
  const [newDate, setNewDate] = useState(getTodayString()); // 預設今天
  const [newTime, setNewTime] = useState('');
  const [newPriority, setNewPriority] = useState('NORMAL');
  const [currentTime, setCurrentTime] = useState(new Date());
  const [showInput, setShowInput] = useState(false);

  // Gemini API 呼叫通用函數
  const callGemini = async (prompt) => {
    const apiKey = ""; // 系统將自動注入 API Key
    try {
      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }]
          })
        }
      );
      const data = await response.json();
      return data.candidates?.[0]?.content?.parts?.[0]?.text;
    } catch (error) {
      console.error("Gemini API Error:", error);
      return null;
    }
  };

  // 1. 幽靈低語功能
  useEffect(() => {
    const generateGhostWhisper = async () => {
      // 只有在有過期任務且尚未生成過訊息時才呼叫（避免頻繁請求）
      const hasOverdue = tasks.some(t => isOverdue(t));
      if (hasOverdue && ghostMessage === '未完成的過去...') {
        const prompt = "你是一個時間的幽靈，看著使用者的待辦清單中有一堆過期未完成的任務。請用繁體中文寫一句簡短、帶點詩意、略微憂鬱但能激勵人心的話，提醒使用者時間一去不復返。不要超過20個字。";
        const text = await callGemini(prompt);
        if (text) setGhostMessage(text.trim());
      }
    };
    
    // 延遲執行以免影響初次渲染效能
    const timer = setTimeout(generateGhostWhisper, 2000);
    return () => clearTimeout(timer);
  }, [tasks]);

  // 2. 智能拆解任務功能
  const handleAiBreakdown = async () => {
    if (!newTask.trim()) return;
    setIsAiLoading(true);

    const prompt = `
      我有一個任務：「${newTask}」。
      請幫我將這個任務拆解成 3 到 5 個具體的子步驟，以便我能立即開始執行。
      請以 JSON 陣列格式回傳，每個物件包含以下欄位：
      - "text": 子任務名稱 (繁體中文)
      - "offsetMinutes": 距離現在幾分鐘後開始 (例如 0, 15, 30...)
      - "priority": 建議的優先級 (只能是 "CRITICAL", "URGENT", "IMPORTANT", "NORMAL" 其中之一)
      
      請直接回傳 JSON，不要包含 Markdown 格式標記。
    `;

    try {
      const resultText = await callGemini(prompt);
      if (!resultText) throw new Error("No response");

      // 清理可能存在的 markdown 標記
      const cleanJson = resultText.replace(/```json/g, '').replace(/```/g, '').trim();
      const subtasks = JSON.parse(cleanJson);

      if (Array.isArray(subtasks)) {
        const now = new Date();
        const createdTasks = subtasks.map((st, index) => {
          // 計算建議時間
          const taskTime = new Date(now.getTime() + (st.offsetMinutes || index * 15) * 60000);
          const timeString = taskTime.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', hour12: false });
          
          return {
            id: Date.now() + index,
            text: st.text,
            priority: PRIORITIES[st.priority] ? st.priority : 'NORMAL',
            date: getTodayString(),
            time: timeString,
            completed: false,
            createdAt: Date.now(),
          };
        });

        setTasks(prev => [...prev, ...createdTasks]);
        setNewTask('');
        setNewTime('');
        setShowInput(false);
      }
    } catch (e) {
      console.error("AI Parsing Error", e);
      // 如果失敗，退回到普通新增
      alert("幽靈暫時無法回應...已轉為普通任務。");
      addTask({ preventDefault: () => {} });
    } finally {
      setIsAiLoading(false);
    }
  };

  // 時鐘更新
  useEffect(() => {
    const timer = setInterval(() => setCurrentTime(new Date()), 60000); // 每分鐘更新
    return () => clearInterval(timer);
  }, []);

  // 保存到 LocalStorage
  useEffect(() => {
    localStorage.setItem('vibe_tasks_v2', JSON.stringify(tasks));
  }, [tasks]);

  const addTask = (e) => {
    e.preventDefault();
    if (!newTask.trim()) return;

    const task = {
      id: Date.now(),
      text: newTask,
      priority: newPriority,
      date: newDate, // YYYY-MM-DD
      time: newTime || null, // HH:MM
      completed: false,
      createdAt: Date.now(),
    };

    setTasks([...tasks, task]);
    // 重置表單，保留日期為今天，方便連續輸入
    setNewTask('');
    setNewTime(''); 
    setNewPriority('NORMAL');
    setShowInput(false);
  };

  const toggleTask = (id) => {
    setTasks(tasks.map(t => t.id === id ? { ...t, completed: !t.completed } : t));
  };

  const deleteTask = (id) => {
    setTasks(tasks.filter(t => t.id !== id));
  };

  // 判斷是否過期 (Overdue)
  const isOverdue = (task) => {
    if (task.completed) return false;
    if (!task.date) return false; // 沒有日期的任務不會過期（雖然我們預設都有日期）

    const now = new Date();
    // 設置比較用的任務時間
    const taskDateTime = new Date(task.date);
    
    if (task.time) {
      const [hours, minutes] = task.time.split(':').map(Number);
      taskDateTime.setHours(hours, minutes, 0);
    } else {
      // 如果只有日期沒有時間，設為該日期的最後一刻，過了那天才算過期
      taskDateTime.setHours(23, 59, 59);
    }

    return now > taskDateTime;
  };

  // 格式化日期顯示 (例如: 11月27日)
  const formatDateDisplay = (dateString) => {
    if (!dateString) return '';
    const date = new Date(dateString);
    const today = new Date();
    const isToday = date.toDateString() === today.toDateString();
    
    // 如果是今天，不顯示日期，只依賴時間顯示
    if (isToday) return '今天';

    return `${date.getMonth() + 1}月${date.getDate()}日`;
  };

  // 分類與排序任務
  const sortedTasks = [...tasks].sort((a, b) => {
    // 先比日期
    const dateA = a.date || '9999-99-99';
    const dateB = b.date || '9999-99-99';
    if (dateA !== dateB) return dateA.localeCompare(dateB);
    
    // 日期相同比時間
    const timeA = a.time || '00:00';
    const timeB = b.time || '00:00';
    return timeA.localeCompare(timeB);
  });

  const overdueTasks = sortedTasks.filter(t => isOverdue(t));
  const upcomingTasks = sortedTasks.filter(t => !t.completed && !isOverdue(t));
  const completedTasks = sortedTasks.filter(t => t.completed);

  const progress = tasks.length > 0 ? Math.round((completedTasks.length / tasks.length) * 100) : 0;

  return (
    <div className="min-h-screen bg-slate-950 text-slate-200 font-sans selection:bg-violet-500/30 overflow-hidden relative">
      {/* 背景裝飾 */}
      <div className="fixed top-[-20%] left-[-10%] w-[500px] h-[500px] bg-violet-600/10 rounded-full blur-[100px] pointer-events-none" />
      <div className="fixed bottom-[-20%] right-[-10%] w-[600px] h-[600px] bg-indigo-600/10 rounded-full blur-[120px] pointer-events-none" />

      <div className="max-w-md mx-auto h-screen flex flex-col relative z-10">
        
        {/* Header Area */}
        <div className="p-6 pb-2 backdrop-blur-sm sticky top-0 z-20 bg-slate-950/80 border-b border-white/5">
          <div className="flex justify-between items-center mb-4">
            <div>
              <h1 className="text-2xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-violet-400 to-fuchsia-400">
                VibeFlow
              </h1>
              <p className="text-sm text-slate-400 tracking-wider flex items-center gap-2 mt-1 font-mono">
                {/* 顯示格式：11月27日 星期四 */}
                <span>
                  {currentTime.toLocaleDateString('zh-TW', { month: 'long', day: 'numeric', weekday: 'long' })}
                </span>
                <span className="w-1 h-1 rounded-full bg-slate-600"></span> 
                <span>
                  {currentTime.toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'})}
                </span>
              </p>
            </div>
            <div className="relative w-12 h-12 flex items-center justify-center">
               <svg className="w-full h-full -rotate-90" viewBox="0 0 36 36">
                <path className="text-slate-800" d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831" fill="none" stroke="currentColor" strokeWidth="3" />
                <path className="text-violet-500 transition-all duration-1000 ease-out" strokeDasharray={`${progress}, 100`} d="M18 2.0845 a 15.9155 15.9155 0 0 1 0 31.831 a 15.9155 15.9155 0 0 1 0 -31.831" fill="none" stroke="currentColor" strokeWidth="3" />
              </svg>
              <span className="absolute text-[10px] font-bold text-slate-400">{progress}%</span>
            </div>
          </div>

          {/* 幽靈提醒區 (Overdue) */}
          {overdueTasks.length > 0 && (
            <div className="mb-4 animate-pulse">
              <div className="bg-rose-950/30 border border-rose-500/30 rounded-xl p-3 flex items-start gap-3 shadow-[0_0_15px_rgba(244,63,94,0.1)]">
                <div className="mt-1">
                  <Ghost size={18} className="text-rose-400" />
                </div>
                <div className="flex-1">
                  <h3 className="text-rose-200 text-sm font-medium mb-1 transition-all duration-1000">
                    {ghostMessage}
                  </h3>
                  <div className="space-y-2">
                    {overdueTasks.map(task => (
                      <div key={task.id} className="flex justify-between items-center text-xs group">
                        <span className="text-rose-300/80 truncate flex-1 mr-2">{task.text}</span>
                        <div className="flex items-center gap-2 shrink-0">
                           <span className="text-rose-500 font-mono text-[10px] border border-rose-500/30 px-1 rounded">
                             {formatDateDisplay(task.date)} {task.time}
                           </span>
                           <button onClick={() => toggleTask(task.id)} className="hover:text-white transition-colors">
                             <CheckCircle2 size={14} />
                           </button>
                        </div>
                      </div>
                    ))}
                  </div>
                </div>
              </div>
            </div>
          )}
        </div>

        {/* Main Scrollable Area */}
        <div className="flex-1 overflow-y-auto px-6 py-4 space-y-6 scrollbar-hide pb-32">
          
          {/* Timeline */}
          <div className="relative pl-4 border-l-2 border-slate-800 space-y-8">
            
            {upcomingTasks.length === 0 && overdueTasks.length === 0 && (
              <div className="text-center py-20 opacity-50">
                <p className="text-slate-500 text-sm">時間軸是一片寧靜的海洋...</p>
                <button 
                  onClick={() => setShowInput(true)} 
                  className="mt-4 text-violet-400 text-sm hover:underline"
                >
                  添加新波紋
                </button>
              </div>
            )}

            {upcomingTasks.map((task, index) => {
              // 安全獲取優先級配置，防止舊資料或錯誤資料導致崩潰
              const priorityConfig = PRIORITIES[task.priority] || PRIORITIES.NORMAL;
              const PriorityIcon = priorityConfig.icon;
              
              const isToday = task.date === getTodayString();
              
              return (
                <div key={task.id} className="relative group">
                  {/* Timeline Dot */}
                  <div className={`absolute -left-[21px] top-4 w-3 h-3 rounded-full border-2 ${task.completed ? 'bg-slate-800 border-slate-600' : 'bg-slate-950 ' + priorityConfig.border} z-10 transition-colors`} />
                  
                  {/* Task Card */}
                  <div className={`
                    relative p-4 rounded-2xl border transition-all duration-300
                    ${task.completed 
                      ? 'bg-slate-900/50 border-slate-800 opacity-50' 
                      : `bg-slate-900/80 ${priorityConfig.border.replace('border', 'border-opacity-30')} hover:bg-slate-800`
                    }
                  `}>
                    <div className="flex justify-between items-start mb-2">
                      <div className="flex flex-wrap items-center gap-2">
                         <span className={`p-1 rounded-md ${priorityConfig.bg} ${priorityConfig.color}`}>
                           <PriorityIcon size={14} />
                         </span>
                         
                         {/* 日期顯示：非今天才顯示日期 */}
                         {!isToday && (
                           <span className="text-xs font-medium text-slate-300 bg-slate-800 px-2 py-0.5 rounded">
                             {formatDateDisplay(task.date)}
                           </span>
                         )}

                         {/* 時間顯示 */}
                         {task.time && (
                           <span className="text-xs font-mono text-slate-400 bg-slate-950 px-2 py-0.5 rounded border border-slate-800">
                             {task.time}
                           </span>
                         )}
                      </div>
                      <button 
                        onClick={() => deleteTask(task.id)}
                        className="opacity-0 group-hover:opacity-100 text-slate-500 hover:text-rose-400 transition-all"
                      >
                        <Trash2 size={16} />
                      </button>
                    </div>

                    <h3 className={`text-base font-medium mb-1 break-words ${task.completed ? 'line-through text-slate-500' : 'text-slate-200'}`}>
                      {task.text}
                    </h3>

                    <button 
                      onClick={() => toggleTask(task.id)}
                      className="absolute bottom-4 right-4 text-slate-500 hover:text-violet-400 transition-colors"
                    >
                      {task.completed ? <CheckCircle2 size={20} className="text-emerald-500" /> : <Circle size={20} />}
                    </button>
                  </div>
                </div>
              );
            })}
          </div>

          {/* Completed History */}
          {completedTasks.length > 0 && (
            <div className="pt-8">
              <h4 className="text-xs font-bold text-slate-600 uppercase tracking-widest mb-4">已完成的歷史</h4>
              <div className="space-y-2 opacity-60 hover:opacity-100 transition-opacity">
                {completedTasks.map(task => (
                  <div key={task.id} className="flex items-center gap-3 p-2 rounded-lg bg-slate-900/30 border border-slate-800/50">
                    <CheckCircle2 size={14} className="text-emerald-500/50" />
                    <span className="text-sm text-slate-500 line-through decoration-slate-700 flex-1 truncate">{task.text}</span>
                    <button onClick={() => deleteTask(task.id)} className="text-slate-700 hover:text-rose-900">
                      <X size={12} />
                    </button>
                  </div>
                ))}
              </div>
            </div>
          )}
        </div>

        {/* Floating Action Button */}
        <div className="absolute bottom-6 right-6 z-30">
          <button 
            onClick={() => {
              setShowInput(!showInput);
              if (!showInput) setNewDate(getTodayString()); // 打開時重置為今天
            }}
            className={`
              w-14 h-14 rounded-full flex items-center justify-center shadow-lg transition-all duration-300
              ${showInput ? 'bg-slate-700 rotate-45' : 'bg-gradient-to-r from-violet-600 to-indigo-600 hover:scale-105 shadow-violet-500/30'}
            `}
          >
            <Plus size={24} className="text-white" />
          </button>
        </div>

        {/* Input Overlay */}
        {showInput && (
          <div className="absolute inset-x-4 bottom-24 bg-slate-900/95 backdrop-blur-xl border border-slate-700 p-5 rounded-2xl shadow-2xl animate-in slide-in-from-bottom-10 fade-in duration-200 z-30">
            <form onSubmit={addTask} className="space-y-4">
              <div>
                <input 
                  autoFocus
                  type="text" 
                  placeholder="接下來做什麼..." 
                  className="w-full bg-transparent text-lg text-white placeholder-slate-500 border-b border-slate-700 focus:border-violet-500 focus:outline-none pb-2"
                  value={newTask}
                  onChange={(e) => setNewTask(e.target.value)}
                  disabled={isAiLoading}
                />
              </div>
              
              {/* 日期與時間選擇區 */}
              <div className="flex gap-3">
                {/* 日期選擇 */}
                <div className="flex-1 bg-slate-800/50 rounded-lg px-3 py-2 flex items-center gap-2 border border-slate-700 focus-within:border-violet-500">
                  <Calendar size={16} className="text-slate-400 shrink-0" />
                  <input 
                    type="date" 
                    className="bg-transparent text-sm text-white w-full focus:outline-none [color-scheme:dark]"
                    value={newDate}
                    onChange={(e) => setNewDate(e.target.value)}
                  />
                </div>
                
                {/* 時間選擇 */}
                <div className="flex-1 bg-slate-800/50 rounded-lg px-3 py-2 flex items-center gap-2 border border-slate-700 focus-within:border-violet-500">
                  <Clock size={16} className="text-slate-400 shrink-0" />
                  <input 
                    type="time" 
                    className="bg-transparent text-sm text-white w-full focus:outline-none [color-scheme:dark]"
                    value={newTime}
                    onChange={(e) => setNewTime(e.target.value)}
                    disabled={isAiLoading}
                  />
                </div>
              </div>

              <div>
                <label className="text-xs text-slate-500 uppercase tracking-wider mb-2 block">輕重緩急</label>
                <div className="grid grid-cols-2 gap-2">
                  {/* 使用 Object.entries 確保讀取到正確的 Key (CRITICAL, NORMAL...) */}
                  {Object.entries(PRIORITIES).map(([key, p]) => (
                    <button
                      key={key}
                      type="button"
                      onClick={() => setNewPriority(key)}
                      disabled={isAiLoading}
                      className={`
                        flex items-center gap-2 px-3 py-2 rounded-lg text-xs font-medium border transition-all
                        ${newPriority === key 
                          ? `${p.bg} ${p.border} ${p.color}` 
                          : 'bg-slate-800 border-transparent text-slate-400 hover:bg-slate-750'}
                      `}
                    >
                      <p.icon size={14} />
                      {p.label}
                    </button>
                  ))}
                </div>
              </div>

              <div className="flex gap-3">
                <button 
                  type="button"
                  onClick={handleAiBreakdown}
                  disabled={!newTask.trim() || isAiLoading}
                  className="flex-1 bg-slate-800 text-violet-300 border border-violet-500/30 font-bold py-3 rounded-xl hover:bg-violet-900/20 transition-all flex items-center justify-center gap-2 disabled:opacity-50 disabled:cursor-not-allowed"
                >
                  {isAiLoading ? <Loader2 className="animate-spin" size={18} /> : <Sparkles size={18} />}
                  <span>智能拆解</span>
                </button>
                
                <button 
                  type="submit" 
                  disabled={isAiLoading}
                  className="flex-[2] bg-slate-100 text-slate-900 font-bold py-3 rounded-xl hover:bg-white transition-colors disabled:opacity-50"
                >
                  確認
                </button>
              </div>
            </form>
          </div>
        )}

      </div>
    </div>
  );
};

export default App;
