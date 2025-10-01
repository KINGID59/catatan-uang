<html lang="id">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Catatan Keuangan Harian Multi-User</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    body { font-family: Arial, sans-serif; margin: 0; background: #f9fafb; }
    header { background: #2563eb; color: white; padding: 1rem; text-align: center; }
    main { max-width: 1000px; margin: 1rem auto; background: white; padding: 1rem; border-radius: 12px; box-shadow: 0 2px 6px rgba(0,0,0,0.1);}
    h2 { margin-top: 0; }
    form { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px,1fr)); gap: 0.7rem; margin-bottom: 1.2rem; }
    input, select, button, textarea {
      padding: 0.6rem; border-radius: 8px; border: 1px solid #ccc; font-size: 0.9rem; width: 100%;
    }
    button { cursor: pointer; background: #2563eb; color: white; border: none; transition: 0.2s; }
    button:hover { background: #1e40af; }
    table { width: 100%; border-collapse: collapse; margin-top: 1rem; font-size: 0.85rem; }
    th, td { border: 1px solid #ddd; padding: 0.5rem; text-align: center; }
    th { background: #f3f4f6; }
    .table-container { overflow-x: auto; }
    .summary { display: flex; flex-wrap: wrap; justify-content: space-between; margin-top: 1rem; font-weight: bold; gap: 0.5rem; }
    .income { color: green; }
    .expense { color: red; }
    .balance { color: #1e40af; }
    .actions button { margin: 0.2rem; padding: 0.4rem 0.6rem; font-size: 0.75rem; }
    .tools { margin-top: 1rem; display: flex; gap: 0.5rem; flex-wrap: wrap; }
    canvas { margin-top: 1rem; max-width: 100%; }
    #auth { max-width: 400px; margin: 3rem auto; background: white; padding: 1.5rem; border-radius: 12px; box-shadow: 0 2px 6px rgba(0,0,0,0.1);}
    #auth h2 { text-align: center; }
    .logout { float: right; margin-top: -2.5rem; }

    /* 📱 Mobile Portrait */
    @media (max-width: 600px) {
      main { margin: 0.5rem; padding: 0.8rem; }
      header { font-size: 0.9rem; padding: 0.8rem; }
      form { grid-template-columns: 1fr; } /* form jadi 1 kolom */
      table { font-size: 0.75rem; }
      .summary { flex-direction: column; align-items: flex-start; }
      .logout { float: none; display: block; margin: 0.5rem auto; }
    }
  </style>
</head>
<body>
  <header>
    <h1>📊 Catatan Keuangan Harian</h1>
  </header>

  <!-- Login / Register -->
  <div id="auth">
    <h2 id="authTitle">Login</h2>
    <form id="authForm">
      <input type="text" id="username" placeholder="Username" required>
      <input type="password" id="password" placeholder="Password" required>
      <button type="submit">Login</button>
    </form>
    <p id="toggleAuth">Belum punya akun? <a href="#" onclick="toggleAuthForm()">Daftar</a></p>
  </div>

  <!-- Aplikasi -->
  <main id="app" style="display:none;">
    <button class="logout" onclick="logout()">🚪 Logout</button>
    <h2>Tambah Catatan</h2>
    <form id="noteForm">
      <input type="date" id="date" required>
      <input type="text" id="category" placeholder="Kategori" required>
      <select id="type" required>
        <option value="pemasukan">Pemasukan</option>
        <option value="pengeluaran">Pengeluaran</option>
      </select>
      <input type="number" id="amount" placeholder="Jumlah" required>
      <textarea id="note" placeholder="Catatan"></textarea>
      <button type="submit">Simpan</button>
    </form>

    <h2>Laporan Bulanan</h2>
    <label>Pilih Bulan:
      <input type="month" id="filterMonth">
    </label>

    <div class="table-container">
      <table id="notesTable">
        <thead>
          <tr>
            <th>Tanggal</th>
            <th>Kategori</th>
            <th>Jenis</th>
            <th>Jumlah</th>
            <th>Catatan</th>
            <th>Aksi</th>
          </tr>
        </thead>
        <tbody></tbody>
      </table>
    </div>

    <div class="summary">
      <div class="income">Total Pemasukan: Rp <span id="totalIncome">0</span></div>
      <div class="expense">Total Pengeluaran: Rp <span id="totalExpense">0</span></div>
      <div class="balance">Saldo Akhir: Rp <span id="balance">0</span></div>
    </div>

    <div class="tools">
      <button onclick="exportCSV()">📑 Export CSV</button>
      <button onclick="printReport()">🖨 Cetak</button>
    </div>

    <canvas id="chart" width="400" height="200"></canvas>
  </main>

  <script>
    // ====== AUTH & DATA ======
    let users = JSON.parse(localStorage.getItem("users")) || {}
    let currentUser = localStorage.getItem("currentUser") || null
    let editIndex = null
    let chart
    let isLoginMode = true

    const authForm = document.getElementById("authForm")
    const authTitle = document.getElementById("authTitle")
    const noteForm = document.getElementById("noteForm")
    const tableBody = document.querySelector("#notesTable tbody")
    const totalIncomeEl = document.getElementById("totalIncome")
    const totalExpenseEl = document.getElementById("totalExpense")
    const balanceEl = document.getElementById("balance")
    const filterMonth = document.getElementById("filterMonth")

    function toggleAuthForm(){
      isLoginMode = !isLoginMode
      authTitle.textContent = isLoginMode ? "Login" : "Register"
      authForm.querySelector("button").textContent = isLoginMode ? "Login" : "Register"
      document.getElementById("toggleAuth").innerHTML = isLoginMode
        ? 'Belum punya akun? <a href="#" onclick="toggleAuthForm()">Daftar</a>'
        : 'Sudah punya akun? <a href="#" onclick="toggleAuthForm()">Login</a>'
    }

    authForm.addEventListener("submit", e=>{
      e.preventDefault()
      const username = document.getElementById("username").value
      const password = document.getElementById("password").value

      if(isLoginMode){
        if(users[username] && users[username].password === password){
          currentUser = username
          localStorage.setItem("currentUser", currentUser)
          showApp()
        }else{
          alert("Username atau password salah!")
        }
      }else{
        if(users[username]){
          alert("Username sudah dipakai!")
        }else{
          users[username] = {password, notes:[]}
          localStorage.setItem("users", JSON.stringify(users))
          alert("Registrasi berhasil, silakan login")
          toggleAuthForm()
        }
      }
    })

    function showApp(){
      document.getElementById("auth").style.display = "none"
      document.getElementById("app").style.display = "block"
      renderNotes()
    }

    function logout(){
      currentUser = null
      localStorage.removeItem("currentUser")
      document.getElementById("app").style.display = "none"
      document.getElementById("auth").style.display = "block"
    }

    function getNotes(){ return users[currentUser]?.notes || [] }
    function saveNotes(n){ users[currentUser].notes = n; localStorage.setItem("users", JSON.stringify(users)) }

    // ====== CATATAN ======
    function renderNotes(){
      let notes = getNotes()
      tableBody.innerHTML = ""
      let income = 0, expense = 0
      const monthFilter = filterMonth.value
      let filteredNotes = notes
      if(monthFilter){ filteredNotes = notes.filter(n => n.date.startsWith(monthFilter)) }

      filteredNotes.forEach((n, i) => {
        const row = document.createElement("tr")
        row.innerHTML = `
          <td>${n.date}</td>
          <td>${n.category}</td>
          <td>${n.type}</td>
          <td>Rp ${n.amount.toLocaleString()}</td>
          <td>${n.note || "-"}</td>
          <td class="actions">
            <button onclick="editNote(${i})">Edit</button>
            <button onclick="deleteNote(${i})">Hapus</button>
          </td>
        `
        tableBody.appendChild(row)
        if(n.type==="pemasukan") income+=n.amount; else expense+=n.amount
      })
      totalIncomeEl.textContent = income.toLocaleString()
      totalExpenseEl.textContent = expense.toLocaleString()
      balanceEl.textContent = (income-expense).toLocaleString()
      renderChart(income, expense)
    }

    noteForm.addEventListener("submit", e=>{
      e.preventDefault()
      let notes = getNotes()
      const note = {
        date: document.getElementById("date").value,
        category: document.getElementById("category").value,
        type: document.getElementById("type").value,
        amount: parseFloat(document.getElementById("amount").value),
        note: document.getElementById("note").value
      }
      if(editIndex!==null){ notes[editIndex] = note; editIndex=null }
      else{ notes.push(note) }
      saveNotes(notes)
      noteForm.reset()
      renderNotes()
    })

    function editNote(i){
      const n = getNotes()[i]
      document.getElementById("date").value = n.date
      document.getElementById("category").value = n.category
      document.getElementById("type").value = n.type
      document.getElementById("amount").value = n.amount
      document.getElementById("note").value = n.note
      editIndex = i
    }

    function deleteNote(i){
      if(confirm("Hapus catatan ini?")){
        let notes = getNotes()
        notes.splice(i,1)
        saveNotes(notes)
        renderNotes()
      }
    }

    filterMonth.addEventListener("change", renderNotes)

    // ====== CHART ======
    function renderChart(income, expense){
      const ctx = document.getElementById("chart").getContext("2d")
      if(chart) chart.destroy()
      chart = new Chart(ctx, {
        type: "pie",
        data: {
          labels:["Pemasukan","Pengeluaran"],
          datasets:[{data:[income,expense], backgroundColor:["#16a34a","#dc2626"]}]
        }
      })
    }

    // ====== TOOLS ======
    function exportCSV(){
      let notes = getNotes()
      let csv="Tanggal,Kategori,Jenis,Jumlah,Catatan\n"
      notes.forEach(n=>{ csv+=`${n.date},${n.category},${n.type},${n.amount},${n.note}\n` })
      const blob=new Blob([csv],{type:"text/csv"})
      const url=URL.createObjectURL(blob)
      const a=document.createElement("a")
      a.href=url;a.download=`catatan_${currentUser}.csv`;a.click()
    }

    function printReport(){ window.print() }

    // ====== AUTO LOGIN JIKA SUDAH LOGIN ======
    if(currentUser){ showApp() }
  </script>
</body>
</html>
