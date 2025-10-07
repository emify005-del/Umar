<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<title>Student Comment Generator — Full Single File</title>
<meta name="viewport" content="width=device-width,initial-scale=1" />
<style>
  /* Layout & fonts */
  body { font-family: Calibri, sans-serif; font-size: 10pt; background:#f4fdf8; color:#003447; margin:18px; }
  h1 { margin:0 0 8px 0; color:#034f4a; }
  .grid { display:grid; grid-template-columns: 1fr 420px; gap:18px; align-items:start; }
  .card { background:white; border-radius:8px; padding:12px; box-shadow:0 6px 18px rgba(2,48,43,0.06); border:1px solid #e6f3ef; }
  label { display:block; color:#0b7a6a; font-weight:600; margin-top:8px; }
  input[type="text"], textarea, select { width:100%; padding:8px; margin-top:6px; border-radius:6px; border:1px solid #cfeee4; box-sizing:border-box; font-family:Calibri; font-size:10pt; }
  select { background:#e9f9f5; color:#003447; }
  textarea { background:#fbfff9; min-height:100px; resize:vertical; border:1px solid #d6f0e6; }
  button { margin-top:10px; padding:8px 12px; border-radius:8px; border:0; cursor:pointer; background:#0b8f73; color:white; font-weight:600; }
  button.secondary { background:#e6f7ff; color:#045a78; border:1px solid #cfeefb; }
  .small { font-size:9pt; color:#2f6b63; margin-top:6px; }
  .task-list-wrap { display:flex; gap:10px; align-items:flex-start; }
  .task-list { width:100%; }
  .task-display { background:#e8f8ef; border:1px solid #bfead0; padding:10px; border-radius:6px; color:#0b6a4f; font-weight:700; margin-top:8px; min-height:44px; }
  .controls { display:flex; gap:8px; flex-wrap:wrap; margin-top:8px; }
  .muted { color:#4a6b66; font-size:9pt; }
  .export-import { display:flex; gap:8px; margin-top:8px; }
  .file-input { display:none; }
  .footer { margin-top:12px; color:#436c66; font-size:9pt; }
  .mode-toggle { display:flex; gap:6px; align-items:center; margin-top:6px; }
  .chip { padding:6px 8px; border-radius:999px; background:#e6fffa; border:1px solid #bfeee3; color:#064e44; font-weight:600; font-size:9pt; }
  @media(max-width:900px){
    .grid { grid-template-columns: 1fr; }
  }
</style>
</head>
<body>
  <h1>Student Comment Generator</h1>
  <div class="grid">
    <!-- Left main panel -->
    <div class="card">
      <label>1 — Student Name (type)</label>
      <input id="studentName" type="text" placeholder="Type student full name (e.g., Ali Khan)" />

      <label>2 — Task list (select code(s)) <span class="small muted"> — click to select; Ctrl/Cmd or Shift for multi-select</span></label>
      <input id="taskSearch" type="text" placeholder="Search by code or words (filters list)" oninput="filterTaskList()" />
      <!-- freeze / scrollable / multi-select list -->
      <select id="taskSelect" size="12" multiple></select>

      <div class="controls">
        <button onclick="selectFirstIfNone()">Ensure Selection</button>
        <button class="secondary" onclick="clearSelection()">Clear Selection</button>
        <div style="flex:1"></div>
        <div class="chip" id="selectedCount">0 selected</div>
      </div>

      <label>3 — Selected Task Name(s) (auto-filled)</label>
      <div id="taskDetails" class="task-display">No task selected</div>

      <div class="mode-toggle small">
        <label style="margin:0; font-weight:700; color:#0b6a4f;">Generate mode:</label>
        <label style="margin:0;"><input type="radio" name="mode" value="combined" checked> Combined (one comment)</label>
        <label style="margin:0;"><input type="radio" name="mode" value="separate"> Separate (one per task)</label>
      </div>

      <div style="margin-top:8px;">
        <button onclick="generate()">Generate Comment(s)</button>
        <button class="secondary" onclick="speakOutput()">Speak</button>
        <button class="secondary" onclick="copyOutput()">Copy</button>
      </div>

      <label style="margin-top:12px">4 — Generated Comment(s)</label>
      <textarea id="output" readonly placeholder="Generated comment(s) will appear here"></textarea>

      <div class="footer">Font: Calibri 10pt · Colors: gentle green / white / blue. Use the export/import buttons to share task lists via OneDrive.</div>
    </div>

    <!-- Right sidebar: task management & import/export -->
    <div>
      <div class="card">
        <label>Manage Tasks (add or update)</label>
        <div class="small muted">Enter code exactly as you want it to appear (e.g., <code>CHCDIV001 - AT 1</code>).</div>
        <input id="taskCode" type="text" placeholder="Task code (e.g., CHCDIV001 - AT 1)" />
        <input id="taskName" type="text" placeholder="Task name (e.g., Sense of Work with diverse people)" />
        <div class="controls">
          <button onclick="addOrUpdate()">Add / Update</button>
          <button class="secondary" onclick="deleteSelectedTask()">Delete Selected</button>
        </div>
        <div class="small muted" style="margin-top:8px">When you select a code from the list, the right-side fields auto-fill for quick edits.</div>
      </div>

      <div class="card" style="margin-top:14px;">
        <label>Export / Import Task List</label>
        <div class="export-import">
          <button onclick="exportTasks()">Export JSON</button>
          <label class="secondary" style="cursor:pointer; padding:8px 12px;">
            Import JSON <input id="importFile" class="file-input" type="file" accept="application/json" onchange="importTasks(event)">
          </label>
        </div>
        <div class="small muted" style="margin-top:8px">Export to save on OneDrive. Others can import this file into their page.</div>
      </div>

      <div class="card" style="margin-top:14px;">
        <label>Quick Templates (editable)</label>
        <div class="small muted">You can edit templates below in localStorage (they persist in this browser).</div>
        <textarea id="templatesArea" rows="8"></textarea>
        <div class="controls">
          <button onclick="saveTemplates()">Save Templates</button>
          <button class="secondary" onclick="resetTemplates()">Reset Templates</button>
        </div>
      </div>
    </div>
  </div>

<script>
// ------------------- Data: initial tasks & templates -------------------
const DEFAULT_TASKS = {
  "CHCDIV001 - AT 1": "Sense of Work with diverse people",
  "CHCDIV001 - AT 2": "Cultural Diversity in Australia",
  "CHCDIV001 - AT 3": "Role in Communicating with Culturally Diverse People",
  "CHCDIV001 - AT 4": "Reflect on Own and Other Cultures",
  "CHCEDS033 - AT1": "Meet legal and ethical obligations in an education support environment",
  "CHCEDS033 - AT2": "Identify and Comply with Legal and Ethical Obligations",
  "CHCEDS033 - AT3 - A": "HEALTH SAFETY AND WELLBEING",
  "CHCEDS033 - AT3 - B": "HEALTH SAFETY AND WELLBEING COMPLETE REQUIRED DOCUMENTATION",
  "CHCEDS033 - AT3 - C": "RESPONDING TO EMERGENCY SITUATIONS",
  "CHCEDS059 - AT 1": "Contribute to the health, safety and wellbeing of students",
  "CHCEDS059 - AT 2": "Complete an Emergency Evacuation and Hazard Report",
  "CHCEDS059 - AT 3": "Educate Students About Health, Safety and Wellbeing",
  "CHCEDS059 - AT 4": "Health, Safety and Wellbeing Outside the Classroom",
  "CHCEDS045 - AT 1": "Develop and Implement Strategies to Support Maths",
  "CHCEDS045 - AT - 2 - A": "ASSESS STUDENT KNOWLEDGE AND SKILL AND BUILD REPORT",
  "CHCEDS045 - AT - 2 - B": "CONTRIBUTE TO PLANNING IN CONSULTATION WITH THE TEACHER",
  "CHCEDS045 - AT - 2 - C": "IMPLEMENT STRATEGIES AND REPORT BACK TO THE TEACHER",
  "CHCEDS046 and CHCEDS047 - AT1": "Literacy and Learning",
  "CHCEDS046 and CHCEDS047 - AT2": "Learning Approaches and Practices",
  "CHCEDS046 and CHCEDS047 - AT3": "Facilitate Literacy Learning",
  "CHCEDS046 and CHCEDS047 - AT4": "Work Performance Reflections",
  "CHCEDS046 and CHCEDS047 - AT5": "Supervisor Report",
  "CHCEDS042 - AT - 1": "Provide Support For E-Learning",
  "CHCEDS042 - AT - 2": "E-learning Case Studies",
  "CHCEDS042 - AT - 3": "E-learning System Research",
  "CHCEDS042 - AT - 4": "Reflections on E-learning Experiences",
  "CHCEDS042 - AT - 5": "Supervisor Report",
  "CHCEDS049 - AT - 1": "Supervise Students Outside the Classroom",
  "CHCEDS049 - AT - 2": "Plan, Implement and Reflect on Supervision Strategies",
  "CHCEDS049 - AT - 2 - A": "Complete a Risk Assessment and Management Plan For Outside The Classroom",
  "CHCEDS049 - AT - 2 - B": \"Assessor's Observations\",
  "CHCEDS049 - AT - 2 - C": \"Reflection Strategies Used for Supervision and Interaction With Children\",
  "CHCEDS049 - AT - 2 - D": \"Complete Incident Debrief and Incident Report Documents\",
  "CHCEDS049 - AT3": "Supervisor Report",
  "CHCEDS053 - AT - 1": "Assist in Production of Language Resources",
  "CHCEDS053 - AT - 2": "Produce Language Resources",
  "CHCEDS053 - AT - 3": "Supervisor Report",
  "CHCEDS048 and CHCEDS056 - AT - 1": "Supporting Additional Needs",
  "CHCEDS048 and CHCEDS056 - AT - 2": "Autism Spectrum Disorder (ASD)",
  "CHCEDS048 and CHCEDS056 - AT 3": "Provide Additional Learning Support",
  "CHCEDS048 and CHCEDS056 - AT - 4": "Reflect on Supporting Additional Needs Students",
  "CHCEDS058 and CHCEDS051 - AT - 1": "Supporting Disability and Behaviours",
  "CHCEDS058 and CHCEDS051 - AT - 2": "Design Behaviour and Inclusion Plans",
  "CHCEDS058 and CHCEDS051 - AT - 3": "Implement Behaviour and Inclusion Plans",
  "CHCEDS058 and CHCEDS051 - AT - 4": "Behaviour and Inclusion Plans Review",
  "CHCEDS058 and CHCEDS051 - AT - 5": "Supervisor Report",
  "CHCPRP003 - AT - 1": "Reflect on and Improve own Professional Practice",
  "CHCPRP003 - AT - 2": "Practice Improvement Research Report - Sector Developments",
  "CHCPRP003 - AT - 3": "Feedback to Improve Practice",
  "CHCPRP003 - AT - 4": "Personal Development Plan",
  "CHCPRT001 - AT - 1": "Identify and Respond to Children and Young People at Risk",
  "CHCPRT001 - AT - 2": "Justin and Heidi",
  "CHCPRT001 - AT - 3 - A": "Disclosure of Abuse From Justin (Role play)",
  "CHCPRT001 - AT - 3 - B": "Report to Supervisor and Coordinator",
  "CHCPRT001 - AT - 3 - C": "Report Suspicions of Child Abuse",
  "CHCPRT001 - AT - 4": "Portfolio of Procedures and Experiences",
  "CHCECE054 and CHCEDS050 - AT1": "Written Questions",
  "CHCECE054 and CHCEDS050 - AT2": "Reflection",
  "CHCECE054 and CHCEDS050 - AT3 - A (Step - 1)": "Support an Aboriginal and/or Torres Strait Islander  RESEARCH",
  "CHCECE054 and CHCEDS050 - AT - 3 - A (Step - 2)": "Support an Aboriginal and/or Torres Strait Islander  COLLABORATE",
  "CHCECE054 and CHCEDS050 - AT 3 - B": "EXAMINE DIFFERENT LEARNING OPPORTUNITIES",
  "CHCECE054 and CHCEDS050 - AT3 - C": "COMMUNICATE LEARNING OPPORTUNITIES TO OTHERS",
  "CHCECE054 and CHCEDS050 - AT3 - D": "PLANNING FOR SUPPORT OF THE GROUP EXPERIENCE",
  "CHCECE054 and CHCEDS050 - AT3 - E": "Support an Aboriginal and/or Torres Strait Islander Cultures reseah and group discusion role play",
  "CHCECE054 and CHCEDS050 - AT4": "Supervisor Report"
};

const DEFAULT_TEMPLATES = [
  "Hi {name}, you achieved a satisfactory result for the task, as you have great information about the role of {task}, and I hope you have a great scope.",
  "Well done {name}, your work on {task} shows a solid understanding of the subject. Keep aiming higher!",
  "{name}, you did a good job completing {task}. Your effort is evident, and I encourage you to keep improving.",
  "Great progress {name}! Your performance in {task} highlights your strong knowledge and growing confidence.",
  "Hi {name}, your work on {task} was clear and well presented. You're developing valuable skills.",
  "{name}, you showed determination in completing {task}. With continued focus, you can achieve even more.",
  "Congratulations {name}, you successfully handled {task}. Your consistency is paying off.",
  "Nice effort {name}! The way you approached {task} shows critical thinking and commitment.",
  "{name}, your results in {task} reflect your ability to apply concepts effectively. Keep building on this.",
  "Hi {name}, completing {task} demonstrates your growing expertise and dedication to learning."
];

// ------------------- Persistence keys -------------------
const LS_TASKS = 'cg_tasks_v1';
const LS_TEMPLATES = 'cg_templates_v1';

// ------------------- Helpers -------------------
function loadTasksFromStorage() {
  try {
    const raw = localStorage.getItem(LS_TASKS);
    if (!raw) return {...DEFAULT_TASKS};
    return JSON.parse(raw);
  } catch (e) { return {...DEFAULT_TASKS}; }
}
function saveTasksToStorage(obj) {
  localStorage.setItem(LS_TASKS, JSON.stringify(obj));
}
function loadTemplatesFromStorage() {
  try {
    const raw = localStorage.getItem(LS_TEMPLATES);
    if (!raw) return DEFAULT_TEMPLATES.slice();
    return JSON.parse(raw);
  } catch(e) { return DEFAULT_TEMPLATES.slice(); }
}
function saveTemplatesToStorage(arr) {
  localStorage.setItem(LS_TEMPLATES, JSON.stringify(arr));
}

// ------------------- UI population -------------------
let tasks = loadTasksFromStorage();
let templates = loadTemplatesFromStorage();

const taskSelect = document.getElementById('taskSelect');
const taskSearch = document.getElementById('taskSearch');
const taskDetails = document.getElementById('taskDetails');
const selectedCount = document.getElementById('selectedCount');
const templatesArea = document.getElementById('templatesArea');

function populateTaskList(filter='') {
  taskSelect.innerHTML = '';
  const keys = Object.keys(tasks).sort((a,b)=>a.localeCompare(b));
  for (const code of keys) {
    if (filter && !(`${code} ${tasks[code]}`.toLowerCase().includes(filter.toLowerCase()))) continue;
    const opt = document.createElement('option');
    opt.value = code;
    opt.textContent = code; // code visible in list as required
    taskSelect.appendChild(opt);
  }
  updateSelectedInfo();
}
function filterTaskList() {
  populateTaskList(taskSearch.value.trim());
}
function updateSelectedInfo() {
  const selected = Array.from(taskSelect.selectedOptions).map(o=>o.value);
  selectedCount.textContent = `${selected.length} selected`;
  if (selected.length === 0) {
    taskDetails.textContent = 'No task selected';
    // clear manage fields
    document.getElementById('taskCode').value = '';
    document.getElementById('taskName').value = '';
  } else {
    // show names list
    const names = selected.map(code => tasks[code] || '(no name)').map(n => `• ${n}`);
    taskDetails.textContent = names.join('\\n');
    // auto-fill manage inputs with first selected
    document.getElementById('taskCode').value = selected[0];
    document.getElementById('taskName').value = tasks[selected[0]] || '';
  }
  // update templates area
  templatesArea.value = templates.join('\\n---\\n');
}

// ------------------- Generate comments -------------------
function getMode() {
  const r = document.querySelector('input[name="mode"]:checked');
  return r ? r.value : 'combined';
}
function generate() {
  const student = document.getElementById('studentName').value.trim();
  const selectedCodes = Array.from(taskSelect.selectedOptions).map(o=>o.value);
  if (!student) { alert('Enter student name'); return; }
  if (selectedCodes.length === 0) { alert('Select at least one task code'); return; }
  const selectedNames = selectedCodes.map(c => tasks[c] || c);
  const mode = getMode();
  if (mode === 'separate') {
    // separate comment per task using random template
    const out = selectedNames.map(n => randomTemplate(student, n));
    document.getElementById('output').value = out.join('\\n\\n');
  } else {
    // combined: produce sentence that lists tasks and inserts into combined template
    const listText = joinWithAnd(selectedNames);
    // Use a friendly combined template
    const combined = `Hi ${student}, well done on your work for ${listText}. Your knowledge and effort are clear — keep building on this progress.`;
    document.getElementById('output').value = combined;
  }
}

// helper to fill templates
function randomTemplate(name, task) {
  if (!templates.length) templates = DEFAULT_TEMPLATES.slice();
  const t = templates[Math.floor(Math.random()*templates.length)];
  return t.replace(/{name}/g, name).replace(/{task}/g, task);
}
function joinWithAnd(arr) {
  if (arr.length === 1) return arr[0];
  if (arr.length === 2) return `${arr[0]} and ${arr[1]}`;
  return arr.slice(0,-1).join(', ') + ', and ' + arr[arr.length-1];
}

// ------------------- Management: add/update/delete -------------------
function addOrUpdate() {
  const code = document.getElementById('taskCode').value.trim();
  const name = document.getElementById('taskName').value.trim();
  if (!code || !name) { alert('Provide both code and name'); return; }
  tasks[code] = name;
  saveTasksToStorage(tasks);
  populateTaskList(taskSearch.value.trim());
  // select the added/updated one
  for (const opt of taskSelect.options) {
    if (opt.value === code) { opt.selected = true; break; }
  }
  updateSelectedInfo();
  alert(`Task "${code}" added/updated.`);
}
function deleteSelectedTask() {
  const sel = Array.from(taskSelect.selectedOptions).map(o=>o.value);
  if (!sel.length) { alert('Select task(s) to delete'); return; }
  if (!confirm(`Delete ${sel.length} task(s)? This cannot be undone.`)) return;
  for (const code of sel) delete tasks[code];
  saveTasksToStorage(tasks);
  populateTaskList(taskSearch.value.trim());
  alert('Deleted.');
}
function selectFirstIfNone() {
  if (taskSelect.selectedOptions.length === 0 && taskSelect.options.length>0) {
    taskSelect.options[0].selected = true;
    updateSelectedInfo();
  }
}
function clearSelection() {
  Array.from(taskSelect.options).forEach(o=>o.selected=false);
  updateSelectedInfo();
}

// ------------------- Export / Import -------------------
function exportTasks() {
  const data = { tasks: tasks, templates: templates };
  const blob = new Blob([JSON.stringify(data, null, 2)], {type:'application/json'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = 'comment_generator_export.json'; document.body.appendChild(a); a.click(); a.remove();
  URL.revokeObjectURL(url);
}
function importTasks(ev) {
  const f = ev.target.files && ev.target.files[0]; if (!f) return;
  const reader = new FileReader();
  reader.onload = e => {
    try {
      const obj = JSON.parse(e.target.result);
      if (obj.tasks) tasks = obj.tasks;
      if (obj.templates) templates = obj.templates;
      saveTasksToStorage(tasks); saveTemplatesToStorage(templates);
      populateTaskList(''); document.getElementById('taskSearch').value='';
      updateSelectedInfo();
      alert('Imported successfully.');
    } catch (err) { alert('Invalid JSON file.'); }
  };
  reader.readAsText(f);
  // reset input
  ev.target.value = '';
}

// ------------------- Templates area -------------------
function saveTemplates() {
  const raw = templatesArea.value.trim();
  if (!raw) return alert('Templates cannot be empty.');
  templates = raw.split('\\n---\\n').map(s=>s.trim()).filter(Boolean);
  saveTemplatesToStorage(templates);
  alert('Templates saved to browser.');
}
function resetTemplates() {
  if (!confirm('Reset templates to defaults?')) return;
  templates = DEFAULT_TEMPLATES.slice();
  saveTemplatesToStorage(templates);
  templatesArea.value = templates.join('\\n---\\n');
  alert('Templates reset.');
}

// ------------------- Utility: speak + copy -------------------
function speakOutput() {
  const text = document.getElementById('output').value.trim();
  if (!text) return alert('Nothing to speak.');
  if (!('speechSynthesis' in window)) return alert('Speech not supported in this browser.');
  window.speechSynthesis.cancel();
  const u = new SpeechSynthesisUtterance(text);
  u.rate = 1; u.pitch = 1;
  window.speechSynthesis.speak(u);
}
async function copyOutput() {
  const text = document.getElementById('output').value.trim();
  if (!text) return alert('Nothing to copy.');
  try { await navigator.clipboard.writeText(text); alert('Copied to clipboard.'); }
  catch(e){ prompt('Copy this text (Ctrl+C then Enter)', text); }
}

// ------------------- Events -------------------
taskSelect.addEventListener('change', updateSelectedInfo);
taskSelect.addEventListener('dblclick', () => { /* double-click could be used to quick-generate in future */ });

templatesArea.addEventListener('focus', ()=> {
  if (!templatesArea.value.trim()) templatesArea.value = templates.join('\\n---\\n');
});

// ------------------- Init -------------------
(function init(){
  // if no saved tasks, seed defaults
  if (!localStorage.getItem(LS_TASKS)) saveTasksToStorage(DEFAULT_TASKS);
  if (!localStorage.getItem(LS_TEMPLATES)) saveTemplatesToStorage(DEFAULT_TEMPLATES);
  tasks = loadTasksFromStorage();
  templates = loadTemplatesFromStorage();
  populateTaskList();
  templatesArea.value = templates.join('\\n---\\n');
  updateSelectedInfo();
})();
</script>
</body>
</html>
