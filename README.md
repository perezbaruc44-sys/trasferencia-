<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Cobros - Registro rápido</title>
  <style>
    :root{--bg:#f7f7fb;--card:#fff;--accent:#0b74da;--muted:#666}
    body{font-family:Inter, system-ui, -apple-system, sans-serif;background:var(--bg);margin:0;padding:24px;color:#111}
    .container{max-width:920px;margin:0 auto}
    h1{margin:0 0 12px;font-size:1.5rem}
    .card{background:var(--card);border-radius:12px;box-shadow:0 6px 18px rgba(20,20,40,.06);padding:18px;margin-bottom:18px}
    form .row{display:flex;gap:12px}
    label{font-size:.85rem;color:var(--muted)}
    input,select,textarea{width:100%;padding:8px;border-radius:8px;border:1px solid #e3e6ef;background:white}
    button{background:var(--accent);color:white;border:0;padding:9px 12px;border-radius:8px;cursor:pointer}
    table{width:100%;border-collapse:collapse;margin-top:12px}
    th,td{padding:8px;border-bottom:1px solid #eef2f7;text-align:left;font-size:.95rem}
    .muted{color:var(--muted);font-size:.85rem}
    .pill{display:inline-block;padding:6px 8px;border-radius:999px;font-size:.8rem}
    .pill.cash{background:#f0f6ff;color:#0b60c9}
    .pill.transfer{background:#eef6ec;color:#1b8a3e}
    .actions button{background:transparent;color:var(--accent);padding:4px 6px;border-radius:6px}
    .tools{display:flex;gap:8px;align-items:center}
    .totals{display:flex;gap:12px;align-items:center;margin-top:8px}
    @media (max-width:720px){.row{flex-direction:column}}
  </style>
</head>
<body>
  <div class="container">
    <h1>Registro de cobros</h1>
    <div class="card">
      <form id="paymentForm">
        <div class="row">
          <div style="flex:1">
            <label>Nombre del pagador</label>
            <input id="payer" required placeholder="Ej. Juan Pérez" />
          </div>
          <div style="width:150px">
            <label>Monto (MXN)</label>
            <input id="amount" type="number" min="0" step="0.01" required placeholder="0.00" />
          </div>
          <div style="width:160px">
            <label>Método</label>
            <select id="method">
              <option value="transfer">Transferencia</option>
              <option value="cash">Efectivo</option>
            </select>
          </div>
        </div>
        <div class="row" style="margin-top:8px">
          <div style="flex:1">
            <label>Referencia / Concepto</label>
            <input id="ref" placeholder="Referencia bancaria, descripción..." />
          </div>
          <div style="width:160px">
            <label>Fecha</label>
            <input id="date" type="date" />
          </div>
          <div style="width:120px;display:flex;align-items:flex-end">
            <button id="saveBtn">Agregar cobro</button>
          </div>
        </div>
      </form>
      <div class="totals muted">
        <div>Total cobros: <strong id="count">0</strong></div>
        <div>Total MXN: <strong id="sum">0.00</strong></div>
        <div>Transferencias: <strong id="sumTransfer">0.00</strong></div>
        <div>Efectivo: <strong id="sumCash">0.00</strong></div>
      </div>
    </div>

    <div class="card">
      <div style="display:flex;justify-content:space-between;align-items:center">
        <div class="tools">
          <label class="muted">Buscar:</label>
          <input id="search" placeholder="nombre, referencia..." />
          <label class="muted">Filtrar:</label>
          <select id="filter">
            <option value="all">Todos</option>
            <option value="transfer">Transferencia</option>
            <option value="cash">Efectivo</option>
          </select>
        </div>
        <div>
          <button id="exportBtn">Exportar CSV</button>
          <button id="clearBtn" style="background:#e74c3c;margin-left:6px">Borrar todo</button>
        </div>
      </div>

      <table id="paymentsTable">
        <thead>
          <tr><th>Nombre</th><th>Monto</th><th>Método</th><th>Referencia</th><th>Fecha</th><th></th></tr>
        </thead>
        <tbody></tbody>
      </table>
    </div>

    <div class="card muted">
      <strong>Notas:</strong>
      <ul>
        <li>Este sistema <em>no procesa pagos reales</em>. Sirve para llevar un registro y generar comprobantes simples.</li>
        <li>Los datos se guardan localmente en tu navegador (localStorage). Si lo usas en otro equipo, copia/exporta el CSV.</li>
      </ul>
    </div>
  </div>

  <!-- Modal sencillo para impresión/recibo -->
  <div id="receipt" style="display:none;position:fixed;top:0;left:0;right:0;bottom:0;background:rgba(0,0,0,.5);align-items:center;justify-content:center">
    <div style="background:white;padding:18px;border-radius:12px;min-width:320px;max-width:720px">
      <div id="receiptContent"></div>
      <div style="display:flex;gap:8px;justify-content:flex-end;margin-top:12px">
        <button id="printReceipt">Imprimir</button>
        <button id="closeReceipt" style="background:#ccc;color:#111">Cerrar</button>
      </div>
    </div>
  </div>

  <script>
    // Estructura y utilidades
    const STORAGE_KEY = 'cobros_prepa_v1'
    const form = document.getElementById('paymentForm')
    const payer = document.getElementById('payer')
    const amount = document.getElementById('amount')
    const method = document.getElementById('method')
    const ref = document.getElementById('ref')
    const date = document.getElementById('date')
    const tbody = document.querySelector('#paymentsTable tbody')
    const countEl = document.getElementById('count')
    const sumEl = document.getElementById('sum')
    const sumTransferEl = document.getElementById('sumTransfer')
    const sumCashEl = document.getElementById('sumCash')
    const search = document.getElementById('search')
    const filter = document.getElementById('filter')

    let payments = load()

    // Inicializar fecha por defecto a hoy
    if(!date.value){
      const d = new Date(); date.value = d.toISOString().slice(0,10)
    }

    render()

    form.addEventListener('submit', e =>{
      e.preventDefault()
      const p = payer.value.trim()
      const a = parseFloat(amount.value)
      if(!p || isNaN(a) || a<=0) return alert('Ingresa nombre y monto válido')
      const item = {
        id: Date.now().toString(),
        payer: p,
        amount: Number(a.toFixed(2)),
        method: method.value,
        ref: ref.value.trim(),
        date: date.value || new Date().toISOString().slice(0,10)
      }
      payments.unshift(item)
      save()
      render()
      form.reset()
      // restaurar fecha hoy
      const d = new Date(); date.value = d.toISOString().slice(0,10)
      payer.focus()
    })

    function render(){
      // aplicar filtros
      const q = search.value.trim().toLowerCase()
      const f = filter.value
      const visible = payments.filter(p =>{
        if(f!=='all' && p.method!==f) return false
        if(!q) return true
        return (p.payer + ' ' + p.ref).toLowerCase().includes(q)
      })

      tbody.innerHTML = ''
      visible.forEach(p =>{
        const tr = document.createElement('tr')
        tr.innerHTML = `
          <td>${escapeHtml(p.payer)}</td>
          <td>$ ${p.amount.toFixed(2)}</td>
          <td>${p.method==='cash'? '<span class="pill cash">Efectivo</span>' : '<span class="pill transfer">Transferencia</span>'}</td>
          <td>${escapeHtml(p.ref||'—')}</td>
          <td>${p.date}</td>
          <td class="actions">
            <button onclick="editPayment('${p.id}')">Editar</button>
            <button onclick="deletePayment('${p.id}')">Eliminar</button>
            <button onclick="showReceipt('${p.id}')">Recibo</button>
          </td>
        `
        tbody.appendChild(tr)
      })

      // totales
      const total = payments.reduce((s,x)=>s + x.amount,0)
      const tTransfer = payments.filter(p=>p.method==='transfer').reduce((s,x)=>s + x.amount,0)
      const tCash = payments.filter(p=>p.method==='cash').reduce((s,x)=>s + x.amount,0)
      countEl.textContent = payments.length
      sumEl.textContent = total.toFixed(2)
      sumTransferEl.textContent = tTransfer.toFixed(2)
      sumCashEl.textContent = tCash.toFixed(2)
    }

    function save(){
      localStorage.setItem(STORAGE_KEY, JSON.stringify(payments))
    }
    function load(){
      try{ const raw = localStorage.getItem(STORAGE_KEY); return raw? JSON.parse(raw) : [] }catch(e){ return [] }
    }

    function editPayment(id){
      const p = payments.find(x=>x.id===id)
      if(!p) return
      payer.value = p.payer
      amount.value = p.amount
      method.value = p.method
      ref.value = p.ref
      date.value = p.date
      // remove old entry
      payments = payments.filter(x=>x.id!==id)
      save(); render();
      payer.focus()
    }

    function deletePayment(id){
      if(!confirm('¿Eliminar este cobro?')) return
      payments = payments.filter(x=>x.id!==id)
      save(); render();
    }

    // Export CSV
    document.getElementById('exportBtn').addEventListener('click', ()=>{
      if(payments.length===0) return alert('No hay datos para exportar')
      const rows = [['id','payer','amount','method','ref','date']]
      payments.slice().reverse().forEach(p=> rows.push([p.id,p.payer,p.amount,p.method,p.ref,p.date]))
      const csv = rows.map(r => r.map(c => '"' + String(c).replace(/"/g,'""') + '"').join(',')).join('\n')
      const blob = new Blob([csv],{type:'text/csv'})
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url; a.download = 'cobros.csv'; a.click(); URL.revokeObjectURL(url)
    })

    // Clear all
    document.getElementById('clearBtn').addEventListener('click', ()=>{
      if(!confirm('Borrar todos los registros? Esta acción no se puede deshacer')) return
      payments = []
      save(); render();
    })

    // search & filter
    search.addEventListener('input', render)
    filter.addEventListener('change', render)

    // simple receipt
    window.showReceipt = function(id){
      const p = payments.find(x=>x.id===id)
      if(!p) return
      const receipt = document.getElementById('receipt')
      const content = document.getElementById('receiptContent')
      content.innerHTML = `
        <div style="text-align:center;margin-bottom:12px"><strong>RECIBO DE PAGO</strong></div>
        <div><strong>Pagador:</strong> ${escapeHtml(p.payer)}</div>
        <div><strong>Monto:</strong> $ ${p.amount.toFixed(2)}</div>
        <div><strong>Método:</strong> ${p.method==='cash' ? 'Efectivo' : 'Transferencia'}</div>
        <div><strong>Referencia:</strong> ${escapeHtml(p.ref||'—')}</div>
        <div><strong>Fecha:</strong> ${p.date}</div>
        <div style="margin-top:12px;font-size:.9rem" class="muted">Generado desde Cobros - Registro rápido</div>
      `
      receipt.style.display = 'flex'
    }
    document.getElementById('closeReceipt').addEventListener('click', ()=>{ document.getElementById('receipt').style.display='none' })
    document.getElementById('printReceipt').addEventListener('click', ()=>{
      const content = document.getElementById('receiptContent').innerHTML
      const w = window.open('','_blank')
      w.document.write('<html><head><title>Recibo</title></head><body>'+content+'</body></html>')
      w.document.close(); w.print();
    })

    function escapeHtml(s){ return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;') }
  </script>
</body>
</html>
