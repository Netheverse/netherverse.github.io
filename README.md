<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
  <meta name="theme-color" content="#0f172a">
  <title>IRL — Habit RPG</title>
  <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.6.0/dist/confetti.browser.min.js"></script>
  <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
  <style>
    /* iOS like system font */
    :root{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,"Helvetica Neue",Arial}

    /* Color variables */
    :root{
      --bg-start: #0f172a; --bg-end: #07103a;
      --accent: #10b981; --accent-2: #059669; /* global green */
      --stat-strength: #ef4444; /* red */
      --stat-charisma: #f59e0b; /* yellow */
      --stat-intellect: #60a5fa; /* blue */
      --stat-magic: #a78bfa; /* purple */
      --card: rgba(255,255,255,0.04); --glass: rgba(255,255,255,0.03);
      --ios-surface: rgba(255,255,255,0.96); --ios-surface-text: #0f172a;
    }

    html,body{height:100%}
    body{margin:0;background:linear-gradient(135deg,var(--bg-start),var(--bg-end));-webkit-font-smoothing:antialiased;-moz-osx-font-smoothing:grayscale;padding:env(safe-area-inset-top) env(safe-area-inset-right) env(safe-area-inset-bottom) env(safe-area-inset-left);color:#fff}
    .app-wrap{max-width:900px;margin:0 auto;padding:16px}

    /* Cards */
    .card{background:var(--card); border-radius:14px; padding:14px; color:inherit}
    .glass{background:var(--glass); border-radius:10px; padding:10px; color:inherit}

    /* XP & stat bars */
    .xp-track{height:12px;background:rgba(255,255,255,0.06);border-radius:999px;overflow:hidden}
    .pill{padding:.28rem .6rem;border-radius:999px;background:rgba(255,255,255,0.03);font-weight:600;color:inherit}
    .small{font-size:.85rem}
    .btn{transition: transform .12s ease, box-shadow .12s ease}
    .tap-large{padding:.9rem 1rem;font-size:1rem;border-radius:12px}

    /* Task layout */
    .task-row{display:flex;align-items:center;justify-content:space-between;gap:8px}
    .task-info{flex:1}
    .task-actions{display:flex;gap:8px}

    /* floating text */
    .float-msg{position:fixed;pointer-events:none;font-weight:800;text-shadow:0 6px 18px rgba(0,0,0,0.6);transform:translate(-50%,0);opacity:1;transition:transform .9s cubic-bezier(.2,.9,.2,1),opacity .9s}

    /* A2HS banner (iOS style) */
    .a2hs-banner{position:fixed;left:50%;transform:translateX(-50%);bottom:18px;background:var(--ios-surface);color:var(--ios-surface-text);box-shadow:0 6px 20px rgba(0,0,0,0.3);border-radius:14px;padding:10px 12px;display:flex;align-items:center;gap:12px;z-index:1200;max-width:95%;min-width:260px}
    .a2hs-icon{width:48px;height:48px;border-radius:10px;background:linear-gradient(180deg,var(--accent),var(--accent-2));display:flex;align-items:center;justify-content:center;color:#042018;font-weight:800}
    .a2hs-text{flex:1}
    .a2hs-actions{display:flex;gap:8px}
    .a2hs-btn{background:transparent;border-radius:10px;padding:8px 10px;border:none;font-weight:600}

    /* light/dark-safe text helpers */
    .on-light{color:var(--ios-surface-text)} /* for use on light backgrounds */
    .on-dark{color:#fff} /* for dark surfaces */

    /* responsive tweaks for mobile (iPhone 16 sized screens and smaller) */
    @media (max-width: 640px){
      .app-wrap{padding:12px}
      .task-row{flex-direction:column;align-items:flex-start}
      .task-actions{width:100%;display:flex;gap:8px}
      .task-actions button{flex:1}
      .card{padding:12px}
      .pill{font-size:14px}
    }

    /* ensure inputs are readable */
    input, select, textarea{color:var(--ios-surface-text);background:rgba(255,255,255,0.06);border:none;padding:10px;border-radius:10px}
    ::placeholder{color:rgba(255,255,255,0.6)}
  </style>
</head>
<body>
  <div id="root" class="app-wrap"></div>

  <script type="text/babel">
  const { useState, useEffect, useRef } = React;

  // helpers
  function uid(){return Date.now() + '-' + Math.floor(Math.random()*1e6)}
  function levelRequirement(level){ return Math.round(100 * Math.pow(1.1, level - 1)); }
  function formatNumber(n){return Math.round(n*100)/100}
  function load(key, fallback){ try{const v = localStorage.getItem(key); return v ? JSON.parse(v) : fallback}catch(e){return fallback} }
  function save(key, val){ localStorage.setItem(key, JSON.stringify(val)) }

  function ProgressBar({label, value, max, color}){
    const pct = Math.min(100, (value / max) * 100 || 0);
    return (
      <div style={{marginBottom:12}}>
        <div style={{display:'flex',justifyContent:'space-between',marginBottom:6,fontSize:13}}><div style={{textTransform:'capitalize'}}>{label}</div><div style={{opacity:.75}}>{formatNumber(value)} / {formatNumber(max)}</div></div>
        <div className="xp-track">
          <div style={{width: pct + '%', background: color, height: '100%'}}></div>
        </div>
      </div>
    )
  }

  function isIos(){ return /iphone|ipad|ipod/i.test(navigator.userAgent) }
  function isInStandalone(){ return (window.navigator.standalone === true) || window.matchMedia('(display-mode: standalone)').matches }

  function playChime(){
    try{
      const ctx = new (window.AudioContext || window.webkitAudioContext)();
      const o = ctx.createOscillator();
      const g = ctx.createGain();
      o.type = 'sine';
      o.frequency.setValueAtTime(880, ctx.currentTime);
      g.gain.setValueAtTime(0, ctx.currentTime);
      g.gain.linearRampToValueAtTime(0.12, ctx.currentTime + 0.01);
      o.connect(g); g.connect(ctx.destination); o.start();
      g.gain.exponentialRampToValueAtTime(0.0001, ctx.currentTime + 0.28);
      o.stop(ctx.currentTime + 0.29);
    }catch(e){}
  }

  function App(){
    // themes kept for user
    const themes = {
      cobalt: {"--bg-start":"#07103a","--bg-end":"#0f172a","--accent":"#10b981","--accent-2":"#059669","--card":"rgba(255,255,255,0.04)","--glass":"rgba(255,255,255,0.03)"},
      ember: {"--bg-start":"#2b0b0b","--bg-end":"#1b0b1b","--accent":"#fb923c","--accent-2":"#f43f5e","--card":"rgba(255,255,255,0.03)","--glass":"rgba(255,255,255,0.02)"},
      mint: {"--bg-start":"#042118","--bg-end":"#042827","--accent":"#34d399","--accent-2":"#06b6d4","--card":"rgba(255,255,255,0.035)","--glass":"rgba(255,255,255,0.02)"}
    }

    const defaultCaps = {strength:100,charisma:100,intellect:100,magic:100}
    const defaultTasks = [
      {id:uid(), name: 'Work Out', category: 'strength', xp: 20, repeatable:true, builtin:true},
      {id:uid(), name: 'Drink one glass of Water', category: 'strength', xp: 5, repeatable:true, builtin:true},
      {id:uid(), name: 'Plan a Hangout With Friends', category: 'charisma', xp: 15, repeatable:true, builtin:true},
      {id:uid(), name: 'Read Bible for 15 minutes', category: 'magic', xp: 18, repeatable:true, builtin:true},
      {id:uid(), name: 'Meditate (5-15 min)', category: 'magic', xp: 12, repeatable:true, builtin:true},
      {id:uid(), name: 'Read a Book Chapter', category: 'intellect', xp: 15, repeatable:true, builtin:true},
      {id:uid(), name: 'Go for a Walk Outside', category: 'strength', xp: 10, repeatable:true, builtin:true},
    ]

    const saved = load('adhd-rpg-v3-mobile', null)
    const initialState = saved || {
      stats: { strength:0, charisma:0, intellect:0, magic:0, level:1, xp:0 },
      caps: defaultCaps,
      statLevels: {strength:1,charisma:1,intellect:1,magic:1},
      tasks: defaultTasks,
      logs: [],
      theme: 'cobalt',
      a2hsDismissed: false
    }

    const [state, setState] = useState(initialState)
    const [newTask, setNewTask] = useState({name:'', category:'strength', xp:10, repeatable:true})
    const [notice, setNotice] = useState(null)
    const [floats, setFloats] = useState([]) // floating messages
    const clickRef = useRef({x: window.innerWidth/2, y: window.innerHeight/2})

    useEffect(()=>{ // apply theme
      const themeName = state.theme || 'cobalt'
      const t = themes[themeName]
      for(const k in t) document.documentElement.style.setProperty(k, t[k])
      save('adhd-rpg-v3-mobile', state)
    }, [state])

    useEffect(()=>{ save('adhd-rpg-v3-mobile', state) }, [state])

    function notify(text, ms=2200){ setNotice(text); setTimeout(()=>setNotice(null), ms) }
    function recordLog(entry){ setState(s=>({...s, logs: [{id:uid(), at:Date.now(), ...entry}, ...s.logs].slice(0,300)})) }

    function spawnFloat(text,x,y,color){
      const id = uid();
      setFloats(f=>[...f,{id,text,x,y,color,created:Date.now()}])
      setTimeout(()=>{setFloats(f=>f.filter(ff=>ff.id!==id))},900)
    }

    function onCompleteTask(task, event){
      let cx = clickRef.current.x || window.innerWidth/2
      let cy = clickRef.current.y || window.innerHeight/2
      if(event){
        if(event.type && event.type.startsWith('touch')){
          const t = event.touches && event.touches[0] ? event.touches[0] : null
          if(t){ cx = t.clientX; cy = t.clientY }
        } else if('clientX' in event){ cx = event.clientX; cy = event.clientY }
      }

      const normX = cx / window.innerWidth
      const normY = cy / window.innerHeight
      const mobile = /iphone|ipad|ipod/i.test(navigator.userAgent)

      confetti({ particleCount: mobile ? 8 : 12, spread: mobile ? 22 : 28, scalar: mobile ? 0.45 : 0.55, origin: { x: normX, y: normY } })
      playChime();

      spawnFloat('Well done!', cx, cy - 24, '#fff')
      spawnFloat('+' + task.xp + ' XP', cx + 28, cy - 8, 'var(--accent)')

      setState(s=>{
        const next = JSON.parse(JSON.stringify(s))
        next.stats.xp = (next.stats.xp || 0) + task.xp
        const cat = task.category
        const newVal = (next.stats[cat] || 0) + task.xp

        if(newVal >= next.caps[cat]){
          while(newVal >= next.caps[cat]){
            next.caps[cat] = Math.round(next.caps[cat] * 2)
            next.statLevels[cat] = (next.statLevels[cat] || 1) + 1
            notify(`${cat.toUpperCase()} tiered up! New cap: ${next.caps[cat]} (Level ${next.statLevels[cat]})`, 2000)
            next.stats.xp += Math.round(task.xp * 0.25)
          }
        }
        next.stats[cat] = Math.min(newVal, next.caps[cat])

        // global level-ups
        let req = levelRequirement(next.stats.level)
        while(next.stats.xp >= req){
          next.stats.xp -= req
          next.stats.level += 1
          req = levelRequirement(next.stats.level)
          notify(`Leveled up! Now level ${next.stats.level}`, 1800)
          spawnFloat('Level Up!', window.innerWidth/2, 120, 'var(--accent)')
        }

        recordLog({task:task.name, xp:task.xp, category:task.category})
        return next
      })

    }

    function addCustomTask(){ if(!newTask.name.trim()) return notify('Enter a task name'); const t = {...newTask, id:uid(), builtin:false}; setState(s=>({...s, tasks:[t,...s.tasks]})); setNewTask({name:'',category:'strength',xp:10, repeatable:true}); notify('Task added') }
    function deleteTask(id){ setState(s=>({...s, tasks: s.tasks.filter(t=>t.id !== id)})); notify('Task deleted') }
    function setTheme(name){ setState(s=>({...s, theme:name})); notify('Theme changed') }
    function dismissA2HS(){ setState(s=>({...s, a2hsDismissed:true})); save('adhd-rpg-v3-mobile', {...state, a2hsDismissed:true}) }

    function onPointerMove(e){
      try{
        if(e.touches && e.touches[0]){ clickRef.current = {x: e.touches[0].clientX, y: e.touches[0].clientY} }
        else if('clientX' in e){ clickRef.current = {x:e.clientX, y:e.clientY} }
      }catch(e){}
    }

    const showA2HS = isIos() && !isInStandalone() && !state.a2hsDismissed

    return (
      <div onTouchStart={onPointerMove} onTouchMove={onPointerMove} onMouseMove={onPointerMove}>
        <header style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:18}}>
          <div>
            <h1 style={{fontSize:22,fontWeight:800}}>IRL</h1>
            <div style={{opacity:.7,fontSize:13,marginTop:6}}>Tiny wins, RPG progression — built for low-energy momentum.</div>
          </div>
          <div style={{textAlign:'right'}}>
            <div className="pill" style={{fontSize:14}}>Level {state.stats.level}</div>
            <div className="small" style={{opacity:.75,marginTop:8}}>XP to next: {Math.max(0, levelRequirement(state.stats.level) - (state.stats.xp||0))}</div>
          </div>
        </header>

        <main style={{display:'grid',gridTemplateColumns:'1fr',gap:16}}>
          <section>
            <div className="card" style={{marginBottom:12}}>
              <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:12}}>
                <div style={{fontSize:13,opacity:.8}}>Global XP</div>
                <div style={{textAlign:'right'}}>
                  <div style={{fontSize:13,opacity:.65}}>Level {state.stats.level}</div>
                  <div style={{fontSize:16,fontWeight:700}}>{formatNumber(state.stats.xp)} / {levelRequirement(state.stats.level)}</div>
                </div>
              </div>
              <div style={{marginBottom:10}} className="xp-track"><div style={{width: Math.min(100, (state.stats.xp/levelRequirement(state.stats.level))*100) + '%', background:'var(--accent)', height:'100%'}} /></div>

              <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:10}}>
                <div className="glass">
                  <div style={{display:'flex',justifyContent:'space-between',marginBottom:6,fontSize:13}}><div>Strength (Lv {state.statLevels.strength})</div><div style={{opacity:.7}}>Cap {state.caps.strength}</div></div>
                  <ProgressBar label="Strength" value={state.stats.strength} max={state.caps.strength} color={'var(--stat-strength)'} />
                </div>
                <div className="glass">
                  <div style={{display:'flex',justifyContent:'space-between',marginBottom:6,fontSize:13}}><div>Charisma (Lv {state.statLevels.charisma})</div><div style={{opacity:.7}}>Cap {state.caps.charisma}</div></div>
                  <ProgressBar label="Charisma" value={state.stats.charisma} max={state.caps.charisma} color={'var(--stat-charisma)'} />
                </div>
                <div className="glass">
                  <div style={{display:'flex',justifyContent:'space-between',marginBottom:6,fontSize:13}}><div>Intellect (Lv {state.statLevels.intellect})</div><div style={{opacity:.7}}>Cap {state.caps.intellect}</div></div>
                  <ProgressBar label="Intellect" value={state.stats.intellect} max={state.caps.intellect} color={'var(--stat-intellect)'} />
                </div>
                <div className="glass">
                  <div style={{display:'flex',justifyContent:'space-between',marginBottom:6,fontSize:13}}><div>Magic (Lv {state.statLevels.magic})</div><div style={{opacity:.7}}>Cap {state.caps.magic}</div></div>
                  <ProgressBar label="Magic" value={state.stats.magic} max={state.caps.magic} color={'var(--stat-magic)'} />
                </div>
              </div>
            </div>

            <div className="card" style={{marginBottom:12}}>
              <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:12}}>
                <h2 style={{fontSize:16,fontWeight:700}}>Tasks</h2>
                <div style={{opacity:.7,fontSize:13}}>Repeatable — tap to complete</div>
              </div>

              <div style={{display:'flex',flexDirection:'column',gap:10}}>
                {state.tasks.map(task=> (
                  <div key={task.id} className="task-row card" style={{padding:12}}>
                    <div className="task-info">
                      <div style={{fontWeight:700}}>{task.name}</div>
                      <div style={{opacity:.7,fontSize:13,marginTop:6}}>{task.category} • {task.xp} XP {task.builtin? ' • built-in':''}</div>
                    </div>
                    <div className="task-actions">
                      <button onClick={(e)=>onCompleteTask(task,e)} className="tap-large btn" style={{background:'linear-gradient(90deg,var(--accent),var(--accent-2))',color:'#062018',fontWeight:800,border:'none'}}>
                        ✅ Complete
                      </button>
                      <button onClick={()=>deleteTask(task.id)} className="tap-large btn" style={{background:'var(--stat-strength)',color:'#fff',border:'none'}}>
                        Del
                      </button>
                    </div>
                  </div>
                ))}
              </div>
            </div>

            <div className="card" style={{marginBottom:12}}>
              <h3 style={{fontWeight:700,marginBottom:8}}>Activity Log</h3>
              <div style={{maxHeight:180,overflow:'auto',fontSize:13,opacity:.85}}>
                {state.logs.length === 0 ? <div style={{opacity:.5}}>No activity yet — complete a task to start building momentum.</div>
                : state.logs.map(l => (
                  <div key={l.id} style={{display:'flex',justifyContent:'space-between',padding:'6px 0',borderBottom:'1px solid rgba(255,255,255,0.03)'}}>
                    <div style={{fontSize:13}}>{new Date(l.at).toLocaleString()} — {l.task}</div>
                    <div style={{opacity:.7,fontSize:13}}>+{l.xp} XP</div>
                  </div>
                ))}
              </div>
            </div>
          </section>

          <section>
            <div className="card" style={{marginBottom:12}}>
              <h3 style={{fontWeight:700,marginBottom:8}}>Create Task</h3>
              <div style={{display:'flex',flexDirection:'column',gap:8}}>
                <input value={newTask.name} onChange={(e)=>setNewTask({...newTask,name:e.target.value})} placeholder="Task name" />
                <select value={newTask.category} onChange={(e)=>setNewTask({...newTask,category:e.target.value})}>
                  <option value="strength">Strength</option>
                  <option value="charisma">Charisma</option>
                  <option value="intellect">Intellect</option>
                  <option value="magic">Magic</option>
                </select>
                <div style={{display:'flex',gap:8}}>
                  <input type="number" value={newTask.xp} onChange={(e)=>setNewTask({...newTask,xp:Number(e.target.value)})} style={{width:96}} />
                  <label style={{display:'flex',alignItems:'center',gap:8,opacity:.8}}><input type="checkbox" checked={newTask.repeatable} onChange={(e)=>setNewTask({...newTask,repeatable:e.target.checked})} /> Repeatable</label>
                </div>
                <button onClick={addCustomTask} className="tap-large" style={{background:'#06b6d4',border:'none',color:'#042028',fontWeight:800,borderRadius:12}}>Add Task</button>
              </div>
            </div>

            <div className="card" style={{marginBottom:12}}>
              <h3 style={{fontWeight:700,marginBottom:8}}>Themes & Home Screen</h3>
              <div style={{display:'flex',gap:8,marginBottom:8}}>
                <button onClick={()=>setTheme('cobalt')} className="tap-large" style={{background:'linear-gradient(90deg,#07103a,#0f172a)',color:'#fff',border:'none'}}>Cobalt</button>
                <button onClick={()=>setTheme('ember')} className="tap-large" style={{background:'linear-gradient(90deg,#2b0b0b,#1b0b1b)',color:'#fff',border:'none'}}>Ember</button>
              </div>
              <div style={{fontSize:13,opacity:.8}}>To install on iPhone: tap Share → Add to Home Screen (Safari). This gives you an app-like experience.</div>
            </div>

            <div className="card">
              <h3 style={{fontWeight:700,marginBottom:8}}>Danger Zone</h3>
              <button onClick={()=>{ if(confirm('Reset all progress?')){ setState({stats:{strength:0,charisma:0,intellect:0,magic:0,level:1,xp:0}, caps: defaultCaps, statLevels:{strength:1,charisma:1,intellect:1,magic:1}, tasks: defaultTasks, logs: [], theme: state.theme}) } }} className="tap-large" style={{background:'#ef4444',border:'none',color:'#fff',fontWeight:800}}>Reset Progress</button>
            </div>

          </section>
        </main>

        {/* A2HS banner (iOS style) */}
        {showA2HS && (
          <div className="a2hs-banner" role="dialog" aria-label="Add IRL to Home Screen">
            <div className="a2hs-icon on-light">IRL</div>
            <div className="a2hs-text">
              <div style={{fontWeight:800}}>Add IRL to your Home Screen</div>
              <div style={{opacity:.7,fontSize:13}}>Tap the Share button in Safari, then tap "Add to Home Screen".</div>
            </div>
            <div className="a2hs-actions">
              <button className="a2hs-btn on-dark" style={{background:'linear-gradient(90deg,var(--accent),var(--accent-2))',color:'#042018'}} onClick={()=>{ alert('In Safari: tap Share → Add to Home Screen.'); }}>
                How
              </button>
              <button className="a2hs-btn on-light" style={{background:'transparent',color:'#0f172a'}} onClick={()=>dismissA2HS()}>
                Dismiss
              </button>
            </div>
          </div>
        )}

        {/* floating messages */}
        {floats.map(f=> (
          <div key={f.id} className="float-msg" style={{left: f.x + 'px', top: f.y + 'px', color: f.color, transform: 'translate(-50%,-6px) scale(1)', opacity: 1}}>{f.text}</div>
        ))}

        {/* notice */}
        {notice && <div style={{position:'fixed',left:'50%',bottom:18,transform:'translateX(-50%)',background:'rgba(0,0,0,0.7)',padding:'10px 16px',borderRadius:10}}>{notice}</div>}
      </div>
    )
  }

  ReactDOM.createRoot(document.getElementById('root')).render(<App />)
  </script>
</body>
</html>
