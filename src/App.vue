<template>
  <div class="app-shell">
    <!-- Login and registration follow the proposal's flow: register or log in first. -->
    <section v-if="page === 'login' || page === 'register'" class="auth-page">
      <div class="auth-card">
        <div class="brand-mark">AMS</div>
        <p class="eyebrow">CENTRAL PHILIPPINES STATE UNIVERSITY</p>
        <h1>{{ page === 'login' ? 'Attendance Monitoring System' : 'Create an account' }}</h1>
        <p class="muted">{{ page === 'login' ? 'Sign in to manage attendance records.' : 'Register an instructor or administrator account.' }}</p>

        <form v-if="page === 'login'" @submit.prevent="login">
          <label>Username</label>
          <input v-model.trim="username" autocomplete="username" placeholder="Enter username" required />
          <label>Password</label>
          <input v-model="password" type="password" autocomplete="current-password" placeholder="Enter password" required />
          <p v-if="error" class="message error">{{ error }}</p>
          <button class="primary full" type="submit">Log in</button>
          <button class="secondary full" type="button" @click="page = 'register'; error = ''">Create account</button>
          <div class="demo-login">
            <strong>Demo accounts</strong>
            <span>Admin: <code>admin</code> / <code>1234</code></span>
            <span>Instructor: <code>instructor</code> / <code>1234</code></span>
          </div>
        </form>

        <form v-else @submit.prevent="register">
          <label>Full name</label>
          <input v-model.trim="registerName" placeholder="Enter full name" required />
          <label>Username</label>
          <input v-model.trim="registerUsername" placeholder="Choose a username" required />
          <label>Password</label>
          <input v-model="registerPassword" type="password" minlength="4" placeholder="At least 4 characters" required />
          <label>Account role</label>
          <select v-model="registerRole" required>
            <option disabled value="">Select role</option>
            <option>Student</option>
            <option>Instructor</option>
            <option>Admin</option>
          </select>
          <p v-if="registerMessage" class="message" :class="registerError ? 'error' : 'success'">{{ registerMessage }}</p>
          <button class="primary full" type="submit">Register account</button>
          <button class="secondary full" type="button" @click="page = 'login'; registerMessage = ''">Back to login</button>
        </form>
        <p class="auth-foot">Student attendance records • Authorized access only</p>
      </div>
    </section>

    <template v-else>
      <aside class="sidebar">
        <div class="side-brand"><div class="brand-mark small">AMS</div><div><strong>Attendance</strong><small>Monitoring System</small></div></div>
        <p class="side-label">WORKSPACE</p>
        <button :class="{ active: section === 'dashboard' }" @click="section = 'dashboard'">▦ <span>Dashboard</span></button>
        <button v-if="currentUser.role === 'Instructor'" :class="{ active: section === 'record' }" @click="section = 'record'">✓ <span>Record attendance</span></button>
        <button :class="{ active: section === 'records' }" @click="section = 'records'">▤ <span>Attendance records</span></button>
        <button :class="{ active: section === 'reports' }" @click="section = 'reports'">▥ <span>Reports</span></button>
        <template v-if="currentUser.role === 'Admin'">
          <p class="side-label">ADMINISTRATION</p>
          <button :class="{ active: section === 'users' }" @click="section = 'users'">♙ <span>Manage users</span></button>
          <button :class="{ active: section === 'subjects' }" @click="section = 'subjects'">▣ <span>Classes & subjects</span></button>
          <button :class="{ active: section === 'database' }" @click="section = 'database'">▤ <span>Centralized records</span></button>
        </template>
        <div class="sidebar-bottom">
          <div class="user-mini"><div class="avatar">{{ initials(currentUser.name) }}</div><div><strong>{{ currentUser.name }}</strong><small>{{ currentUser.role }}</small></div></div>
          <button class="logout" @click="logout">↪ <span>Log out</span></button>
        </div>
      </aside>

      <main class="main-area">
        <header class="topbar">
          <div><p class="eyebrow">ATTENDANCE MANAGEMENT</p><h1>{{ sectionTitle }}</h1></div>
          <div class="top-user"><div class="avatar">{{ initials(currentUser.name) }}</div><div><strong>{{ currentUser.name }}</strong><small>{{ currentUser.role }}</small></div></div>
        </header>

        <div v-if="notice" class="notice" :class="noticeType">{{ notice }} <button @click="notice = ''">×</button></div>

        <section v-if="section === 'dashboard'">
          <div class="welcome"><div><h2>Hello, {{ currentUser.name.split(' ')[0] }} 👋</h2><p>Here is the current overview of attendance information.</p></div><div class="date-chip">{{ today }}</div></div>
          <div class="stat-grid">
            <article class="stat-card"><div class="stat-icon green">♙</div><div><p>Total students</p><strong>{{ students.length }}</strong></div></article>
            <article class="stat-card"><div class="stat-icon blue">✓</div><div><p>Present today</p><strong>{{ countStatus('Present') }}</strong></div></article>
            <article class="stat-card"><div class="stat-icon orange">◷</div><div><p>Late today</p><strong>{{ countStatus('Late') }}</strong></div></article>
            <article class="stat-card"><div class="stat-icon red">!</div><div><p>Absent today</p><strong>{{ countStatus('Absent') }}</strong></div></article>
          </div>
          <div class="panel">
            <div class="panel-head"><div><h3>Recent attendance records</h3><p>Latest saved attendance entries</p></div><button class="secondary" @click="section = 'records'">View all records →</button></div>
            <div class="table-wrap"><table><thead><tr><th>Student</th><th>Subject</th><th>Date</th><th>Status</th></tr></thead>
              <tbody><tr v-for="record in recentRecords" :key="record.id"><td><strong>{{ studentName(record.studentId) }}</strong></td><td>{{ record.subject }}</td><td>{{ record.date }}</td><td><span class="status" :class="statusClass(record.status)">{{ record.status }}</span></td></tr>
                <tr v-if="!recentRecords.length"><td colspan="4" class="empty">No attendance has been saved yet.</td></tr></tbody></table></div>
          </div>
        </section>

        <section v-else-if="section === 'record'" class="panel">
          <div class="panel-head"><div><h3>Record student attendance</h3><p>Select a subject and mark each student's attendance.</p></div></div>
          <form @submit.prevent="saveAttendance" class="record-form">
            <div class="field-grid"><div><label>Subject / class</label><select v-model="selectedSubject" required><option disabled value="">Select subject</option><option v-for="subject in subjects" :key="subject.name" :value="subject.name">{{ subject.name }} — {{ subject.schedule }}</option></select></div><div><label>Attendance date</label><input v-model="attendanceDate" type="date" required /></div></div>
            <div class="table-wrap"><table><thead><tr><th>Student ID</th><th>Student name</th><th>Attendance status</th></tr></thead><tbody>
              <tr v-for="student in students" :key="student.id"><td>{{ student.id }}</td><td><strong>{{ student.name }}</strong></td><td><select v-model="draftStatuses[student.id]"><option>Present</option><option>Absent</option><option>Late</option><option>Excused</option></select></td></tr>
            </tbody></table></div>
            <div class="form-actions"><span class="muted">{{ students.length }} students in this sample list</span><button class="primary" type="submit">Save attendance</button></div>
          </form>
        </section>

        <section v-else-if="section === 'records' || section === 'reports'" class="panel">
          <div class="panel-head"><div><h3>{{ section === 'reports' ? 'Attendance reports' : 'Attendance history' }}</h3><p>{{ section === 'reports' ? 'Filter and export a summary of recorded attendance.' : 'Search and review saved attendance entries.' }}</p></div><button v-if="section === 'reports'" class="primary" @click="downloadReport">Download CSV report</button></div>
          <div class="filters"><input v-model="searchText" placeholder="Search student or subject..." /><input v-model="filterDate" type="date" aria-label="Filter by date" /><select v-model="filterStatus"><option value="">All statuses</option><option>Present</option><option>Absent</option><option>Late</option><option>Excused</option></select></div>
          <div class="report-summary" v-if="section === 'reports'"><div><span>Records shown</span><strong>{{ filteredRecords.length }}</strong></div><div><span>Present</span><strong>{{ filteredRecords.filter(r => r.status === 'Present').length }}</strong></div><div><span>Absent</span><strong>{{ filteredRecords.filter(r => r.status === 'Absent').length }}</strong></div><div><span>Late</span><strong>{{ filteredRecords.filter(r => r.status === 'Late').length }}</strong></div></div>
          <div class="table-wrap"><table><thead><tr><th>Student</th><th>Subject</th><th>Date</th><th>Time saved</th><th>Status</th></tr></thead><tbody>
            <tr v-for="record in filteredRecords" :key="record.id"><td><strong>{{ studentName(record.studentId) }}</strong></td><td>{{ record.subject }}</td><td>{{ record.date }}</td><td>{{ record.time }}</td><td><span class="status" :class="statusClass(record.status)">{{ record.status }}</span></td></tr>
            <tr v-if="!filteredRecords.length"><td colspan="5" class="empty">No matching records found.</td></tr>
          </tbody></table></div>
        </section>

        <section v-else-if="section === 'users'" class="panel">
          <div class="panel-head"><div><h3>System users</h3><p>Accounts registered in this browser demo.</p></div></div>
          <div class="table-wrap"><table><thead><tr><th>Name</th><th>Username</th><th>Role</th></tr></thead><tbody><tr v-for="user in users" :key="user.username"><td>{{ user.name }}</td><td>{{ user.username }}</td><td><span class="status status-present">{{ user.role }}</span></td></tr></tbody></table></div>
        </section>

        <section v-else-if="section === 'subjects'" class="panel">
          <div class="panel-head"><div><h3>Classes and subjects</h3><p>Sample class schedule for the prototype.</p></div></div>
          <div class="table-wrap"><table><thead><tr><th>Subject</th><th>Schedule</th></tr></thead><tbody><tr v-for="subject in subjects" :key="subject.name"><td><strong>{{ subject.name }}</strong></td><td>{{ subject.schedule }}</td></tr></tbody></table></div>
        </section>

        <section v-else-if="section === 'database'" class="panel">
          <div class="panel-head"><div><h3>Centralized records overview</h3><p>Information groups represented in this prototype.</p></div></div>
          <div class="database-grid"><article><span>♙</span><h3>Student profiles</h3><p>{{ students.length }} sample student profiles</p></article><article><span>▣</span><h3>Classes & subjects</h3><p>{{ subjects.length }} sample subjects</p></article><article><span>✓</span><h3>Attendance records</h3><p>{{ records.length }} saved entries</p></article><article><span>♙</span><h3>System users</h3><p>{{ users.length }} accounts</p></article></div>
          <p class="info-note">This demo stores data in the browser's local storage. It is not a shared or secured university database.</p>
        </section>

        <footer>Attendance Monitoring System <span>•</span> Prototype for academic project use</footer>
      </main>
    </template>
  </div>
</template>

<script>
const STORAGE_KEY = 'ams-vue-demo-v1'
const seedUsers = [
  { name: 'Administrator', username: 'admin', password: '1234', role: 'Admin' },
  { name: 'Juan Dela Cruz', username: 'instructor', password: '1234', role: 'Instructor' }
]
const seedStudents = [
  { id: '2026-001', name: 'Student 1' },
  { id: '2026-002', name: 'Student 2' },
  { id: '2026-003', name: 'Student 3' },
  { id: '2026-004', name: 'Student 4' }
]
const seedSubjects = [
  { name: 'PCIT 11', schedule: 'Monday · 8:00 AM' },
  { name: 'Web Development', schedule: 'Tuesday · 10:00 AM' },
  { name: 'Database Management', schedule: 'Wednesday · 1:00 PM' }
]

export default {
  data() {
    return {
      page: 'login', section: 'dashboard', username: '', password: '', error: '',
      registerName: '', registerUsername: '', registerPassword: '', registerRole: '', registerMessage: '', registerError: false,
      currentUser: null, users: [], students: seedStudents.map(s => ({ ...s })), subjects: seedSubjects.map(s => ({ ...s })),
      records: [], selectedSubject: '', attendanceDate: this.localDate(), draftStatuses: {}, searchText: '', filterDate: '', filterStatus: '',
      notice: '', noticeType: 'success'
    }
  },
  computed: {
    today() { return new Date().toLocaleDateString('en-PH', { year: 'numeric', month: 'short', day: 'numeric' }) },
    sectionTitle() {
      return ({ dashboard: 'Dashboard', record: 'Record attendance', records: 'Attendance records', reports: 'Reports', users: 'Manage users', subjects: 'Classes & subjects', database: 'Centralized records' })[this.section] || 'Dashboard'
    },
    recentRecords() { return [...this.records].sort((a, b) => b.id - a.id).slice(0, 5) },
    filteredRecords() {
      const q = this.searchText.toLowerCase()
      return [...this.records].filter(r => {
        const name = this.studentName(r.studentId).toLowerCase()
        return (!q || name.includes(q) || r.subject.toLowerCase().includes(q)) &&
          (!this.filterDate || r.date === this.filterDate) && (!this.filterStatus || r.status === this.filterStatus)
      }).sort((a, b) => b.id - a.id)
    }
  },
  created() {
    try {
      const saved = JSON.parse(localStorage.getItem(STORAGE_KEY) || '{}')
      this.users = saved.users?.length ? saved.users : seedUsers.map(u => ({ ...u }))
      this.records = saved.records || []
    } catch {
      this.users = seedUsers.map(u => ({ ...u }))
      this.records = []
    }
    this.resetDraft()
  },
  methods: {
    localDate() {
      const d = new Date()
      return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`
    },
    persist() {
      localStorage.setItem(STORAGE_KEY, JSON.stringify({ users: this.users, records: this.records }))
    },
    login() {
      const found = this.users.find(u => u.username === this.username && u.password === this.password)
      if (!found) { this.error = 'Invalid username or password.'; return }
      this.currentUser = { name: found.name, username: found.username, role: found.role }
      this.page = 'app'; this.section = 'dashboard'; this.error = ''; this.notice = ''
    },
    register() {
      if (this.users.some(u => u.username.toLowerCase() === this.registerUsername.toLowerCase())) {
        this.registerError = true; this.registerMessage = 'That username is already in use.'; return
      }
      this.users.push({ name: this.registerName, username: this.registerUsername, password: this.registerPassword, role: this.registerRole })
      this.persist()
      this.registerError = false; this.registerMessage = 'Account created. You can now log in.'
      this.registerName = ''; this.registerUsername = ''; this.registerPassword = ''; this.registerRole = ''
      this.page = 'login'; this.username = this.users[this.users.length - 1].username; this.password = ''
      this.registerMessage = ''
    },
    logout() {
      this.currentUser = null; this.page = 'login'; this.username = ''; this.password = ''
      this.selectedSubject = ''; this.notice = ''; this.section = 'dashboard'
    },
    initials(name) { return name.split(/\s+/).map(part => part[0]).slice(0, 2).join('').toUpperCase() },
    studentName(id) { return this.students.find(s => s.id === id)?.name || id },
    countStatus(status) {
      return this.records.filter(r => r.date === this.localDate() && r.status === status).length
    },
    statusClass(status) { return `status-${status.toLowerCase()}` },
    resetDraft() {
      const draft = {}
      this.students.forEach(s => { draft[s.id] = 'Present' })
      this.draftStatuses = draft
    },
    saveAttendance() {
      if (!this.selectedSubject) return
      const time = new Date().toLocaleTimeString('en-PH', { hour: '2-digit', minute: '2-digit' })
      this.students.forEach(student => {
        const existing = this.records.find(r => r.studentId === student.id && r.subject === this.selectedSubject && r.date === this.attendanceDate)
        if (existing) { existing.status = this.draftStatuses[student.id]; existing.time = time }
        else this.records.push({ id: Date.now() + Math.random(), studentId: student.id, subject: this.selectedSubject, date: this.attendanceDate, time, status: this.draftStatuses[student.id] })
      })
      this.persist(); this.notice = 'Attendance saved successfully.'; this.noticeType = 'success'; this.section = 'records'
    },
    downloadReport() {
      const rows = [['Student ID', 'Student name', 'Subject', 'Date', 'Time', 'Status'], ...this.filteredRecords.map(r => [r.studentId, this.studentName(r.studentId), r.subject, r.date, r.time, r.status])]
      const csv = rows.map(row => row.map(value => `"${String(value ?? '').replaceAll('"', '""')}"`).join(',')).join('\r\n')
      const url = URL.createObjectURL(new Blob([csv], { type: 'text/csv;charset=utf-8;' }))
      const a = document.createElement('a'); a.href = url; a.download = 'attendance-report.csv'; a.click(); URL.revokeObjectURL(url)
      this.notice = 'CSV report downloaded.'; this.noticeType = 'success'
    }
  }
}
</script>
