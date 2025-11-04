
fetch('cast.json')
  .then(response => response.json())
  .then(data => {
    allData = data;
    cargarSelects(); // llama a tu función que llena los menús de concursantes
  })
  .catch(error => console.error('Error al cargar cast.json:', error));
// app.js - Versión final: fotos + historial + expulsado desaparece (simulación aleatoria)

// --- Datos ---
// Si ya tienes `window.allData` embebido (p.ej. en otro script), se usará.
// Si no, el init() intentará cargar data/cast.json.
let allData = window.allData || {};
let selectedCast = []; // { name, edition, photo }
let weekCounter = parseInt(localStorage.getItem('gh_week') || '1', 10);

// --- util DOM ---
const $ = sel => document.querySelector(sel);
const create = (tag, props = {}) => {
  const n = document.createElement(tag);
  Object.entries(props).forEach(([k, v]) => {
    if (k in n) n[k] = v;
    else n.setAttribute(k, v);
  });
  return n;
};

// --- init ---
async function init() {
  // cargar si no hay datos embebidos
  if (!Object.keys(allData).length) {
    try {
      const res = await fetch('data/cast.json');
      if (!res.ok) throw new Error('No se pudo cargar data/cast.json');
      allData = await res.json();
    } catch (e) {
      $('#results').innerText = 'Error cargando cast.json: ' + e.message;
      return;
    }
  }

  // asegurar history y photo minimal
  Object.keys(allData).forEach(ed => {
    allData[ed].forEach(p => {
      if (!Array.isArray(p.history)) p.history = [];
      if (!p.photo) p.photo = 'images/placeholder.png'; // pon tu placeholder en images/
    });
  });

  renderEditionSelects();
  renderWeekInfo();
  renderTrackRecord(); // inicial vacío
}

// --- render selects ---
function renderEditionSelects() {
  const container = $('#editionsContainer');
  container.innerHTML = '';
  Object.keys(allData).forEach(edition => {
    const block = create('div', { className: 'edition-block' });
    const label = create('label', { innerText: edition });
    const select = create('select', { id: 'select_' + edition, multiple: true });
    select.size = Math.min(allData[edition].length, 6);

    // opción vacía para estética
    const empty = create('option', { value: '' });
    empty.innerText = '-- (sin seleccionar) --';
    select.appendChild(empty);

    allData[edition].forEach(p => {
      const opt = create('option', { value: p.name, innerText: p.name });
      select.appendChild(opt);
    });

    block.appendChild(label);
    block.appendChild(select);
    container.appendChild(block);
  });
}

// --- cargar selección y mostrar cards con fotos ---
function loadSelectedCast() {
  selectedCast = [];
  Object.keys(allData).forEach(edition => {
    const sel = document.getElementById('select_' + edition);
    if (!sel) return;
    Array.from(sel.selectedOptions).forEach(opt => {
      if (opt.value && opt.value.trim() !== '') {
        const obj = allData[edition].find(x => x.name === opt.value);
        if (obj) selectedCast.push({ name: obj.name, edition, photo: obj.photo });
      }
    });
  });

  const resultsDiv = $('#results');
  if (!selectedCast.length) {
    resultsDiv.innerText = 'No has seleccionado concursantes.';
    renderTrackRecord();
    return;
  }

  // Mostrar grid de cards con foto + nombre
  resultsDiv.innerHTML = '<strong>Concursantes seleccionados:</strong><div class="cast-grid"></div>';
  const grid = resultsDiv.querySelector('.cast-grid');

  selectedCast.forEach(c => {
    const card = create('div', { className: 'card' });
    const img = create('img', { src: c.photo, alt: c.name, className: 'card-photo' });
    const p = create('p', { innerHTML: `<strong>${c.name}</strong><br><small>${c.edition}</small>` });
    card.appendChild(img);
    card.appendChild(p);
    grid.appendChild(card);
  });

  renderTrackRecord();
}

// --- simular semana (totalmente aleatorio) ---
function simulateWeek() {
  if (selectedCast.length === 0) {
    alert('Primero selecciona y carga concursantes.');
    return;
  }

  // Ganador aleatorio
  const winner = selectedCast[Math.floor(Math.random() * selectedCast.length)];

  // Nominados: 3 aleatorios sin repetir (pueden incluir al ganador si hay pocos; evitamos ganador)
  let pool = selectedCast.filter(s => !(s.name === winner.name && s.edition === winner.edition));
  // si pool es vacío (solo 1 participante), nominated será vacío
  pool = shuffleArray(pool);
  const nominated = pool.slice(0, Math.min(3, pool.length));

  // Si no hay nominados (p.ej. solo 1 participante), expulsado = null
  const expelled = nominated.length ? nominated[Math.floor(Math.random() * nominated.length)] : null;

  // Actualizar history: añadir registro a cada seleccionado (incluye expulsado)
  selectedCast.forEach(c => {
    // buscar objeto original en allData (si está allí)
    const personArr = allData[c.edition] || [];
    let personObj = personArr.find(p => p.name === c.name);

    // Si no está encontrado en allData (por alguna razón), crear temporal para history
    if (!personObj) {
      // crea objeto ligero para no romper
      personObj = { name: c.name, photo: c.photo, history: [] };
      // no lo añadimos a allData aquí (porque si no existía, quizás quieras mantenerlo así)
    }

    const entry = { semana: weekCounter, resultado: 'safe' };
    if (winner && c.name === winner.name && c.edition === winner.edition) entry.resultado = 'ganador';
    if (expelled && c.name === expelled.name && c.edition === expelled.edition) entry.resultado = 'expulsado';
    else if (nominated.find(n => n.name === c.name && n.edition === c.edition)) {
      if (entry.resultado !== 'expulsado' && entry.resultado !== 'ganador') entry.resultado = 'nominado';
    }

    if (!Array.isArray(personObj.history)) personObj.history = [];
    personObj.history.push(entry);
  });

  // Antes de eliminar expulsado, guardamos texto resultado
  const winnerText = `${winner.name} (${winner.edition})`;
  const nominatedText = nominated.map(n => `${n.name} (${n.edition})`).join(', ') || '—';
  const expelledText = expelled ? `${expelled.name} (${expelled.edition})` : '—';

  // Si hay expulsado: eliminarlo de la "casa" (allData, selects y selectedCast)
  if (expelled) {
    // 1) eliminar del allData[edition]
    if (Array.isArray(allData[expelled.edition])) {
      allData[expelled.edition] = allData[expelled.edition].filter(p => p.name !== expelled.name);
    }
    // 2) eliminar opción del select DOM
    const selDOM = document.getElementById('select_' + expelled.edition);
    if (selDOM) {
      Array.from(selDOM.options).forEach(opt => {
        if (opt.value === expelled.name) opt.remove();
      });
    }
    // 3) eliminar de selectedCast
    selectedCast = selectedCast.filter(c => !(c.name === expelled.name && c.edition === expelled.edition));

    // 4) opcional: mostrar mensaje de expulsión adicional en results (se hará de todas formas abajo)
  }

  // avanzar semana
  weekCounter++;
  localStorage.setItem('gh_week', String(weekCounter));

  // mostrar resumen
  $('#results').innerHTML = `<strong>Semana ${weekCounter - 1} — Resultados</strong><br>
    Ganador: ${winnerText}<br>
    Nominados: ${nominatedText}<br>
    Expulsado/a: ${expelledText}`;

  // actualizar UI
  renderEditionSelects(); // actualiza selects por si expulsado fue eliminado
  renderTrackRecord();    // tabla actualizada (expulsado ya no aparece)
  renderWeekInfo();
}

// --- render track record con mini-foto ---
function renderTrackRecord() {
  // crear/obtener contenedor
  let container = $('#trackRecord');
  if (!container) {
    container = create('div', { id: 'trackRecord' });
    container.style.marginTop = '12px';
    // insertarlo después de results
    const after = $('#results');
    after.parentNode.insertBefore(container, after.nextSibling);
  }
  container.innerHTML = '';

  if (!selectedCast.length) {
    container.innerHTML = '<em>Selecciona concursantes y pulsa "Cargar concursantes seleccionados" para ver su track record.</em>';
    return;
  }

  // número de semanas actuales
  const currentWeek = Math.max(1, weekCounter - 1);

  // build table
  const table = create('table');
  table.style.width = '100%';
  table.style.borderCollapse = 'collapse';
  table.style.marginTop = '10px';
  const thead = create('thead');
  const headerRow = create('tr');

  // encabezados
  ['Concursante','Edición', ...Array.from({length: currentWeek}, (_,i)=>'S' + (i+1))].forEach(h => {
    const th = create('th', { innerText: h });
    th.style.border = '1px solid #ddd';
    th.style.padding = '6px';
    th.style.background = '#f3f3f3';
    headerRow.appendChild(th);
  });
  thead.appendChild(headerRow);
  table.appendChild(thead);

  const tbody = create('tbody');

  // filas por cada participante activo (selectedCast)
  selectedCast.forEach(sc => {
    const tr = create('tr');

    // celda con mini foto + nombre
    const tdInfo = create('td');
    tdInfo.style.border = '1px solid #ddd';
    tdInfo.style.padding = '6px';
    tdInfo.style.verticalAlign = 'middle';
    tdInfo.innerHTML = `
      <div style="display:flex;align-items:center;gap:8px">
        <img src="${sc.photo}" alt="${sc.name}" style="width:36px;height:36px;border-radius:50%;object-fit:cover">
        <div>${sc.name}</div>
      </div>
    `;
    tr.appendChild(tdInfo);

    // celda edición
    const tdEd = create('td', { innerText: sc.edition });
    tdEd.style.border = '1px solid #ddd';
    tdEd.style.padding = '6px';
    tdEd.style.textAlign = 'center';
    tr.appendChild(tdEd);

    // celdas por semana
    for (let w = 1; w <= currentWeek; w++) {
      const td = create('td');
      td.style.border = '1px solid #ddd';
      td.style.padding = '6px';
      td.style.textAlign = 'center';
      // buscar history en allData (si expulsado ya eliminado, su history sigue en objeto anterior pero ya no mostramos)
      const personObj = (allData[sc.edition] || []).find(p => p.name === sc.name) || null;
      // IMPORTANT: if the person was expelled earlier but remains in selectedCast this shouldn't happen.
      // We also need to look up history from anywhere: scan previous allData backups? For simplicity we look in personObj.
      // If not found (rare), show blank.
      let histEntry = null;
      if (personObj && Array.isArray(personObj.history)) {
        histEntry = personObj.history.find(h => h.semana === w);
      } else {
        // if not found in current allData (maybe they were expelled earlier), try to find history in local storage fallback
        // but for simplicity we leave blank
        histEntry = null;
      }

      let label = '';
      if (histEntry) {
        if (histEntry.resultado === 'ganador') label = '🏆';
        else if (histEntry.resultado === 'expulsado') label = '❌';
        else if (histEntry.resultado === 'nominado') label = '⚠️';
        else if (histEntry.resultado === 'safe') label = '✔️';
        else label = histEntry.resultado;
      }
      td.innerText = label;
      tr.appendChild(td);
    }

    tbody.appendChild(tr);
  });

  table.appendChild(tbody);
  container.appendChild(table);
}

// --- render info semana ---
function renderWeekInfo() {
  let info = $('#weekInfo');
  if (!info) {
    info = create('div', { id: 'weekInfo' });
    info.style.marginTop = '8px';
    $('#results').parentNode.insertBefore(info, $('#results'));
  }
  info.innerHTML = `<em>Semana actual: ${Math.max(1, weekCounter - 1)}</em>`;
}

// --- reset historial completo (opcional) ---
function resetAllHistory() {
  if (!confirm('¿Resetear todo el historial y volver a semana 1?')) return;
  Object.keys(allData).forEach(ed => {
    allData[ed].forEach(p => p.history = []);
  });
  weekCounter = 1;
  localStorage.setItem('gh_week', String(weekCounter));
  selectedCast = [];
  // reset selects UI
  renderEditionSelects();
  $('#results').innerText = 'Historial reseteado.';
  renderTrackRecord();
  renderWeekInfo();
}

// --- util: shuffle ---
function shuffleArray(arr) {
  const a = [...arr];
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

// --- eventos ---
document.addEventListener('DOMContentLoaded', init);
document.addEventListener('click', (e) => {
  if (e.target.id === 'loadCast') loadSelectedCast();
  if (e.target.id === 'simulateWeek') simulateWeek();
  if (e.target.id === 'clearSelection') {
    // limpiar selects y memoria
    Object.keys(allData).forEach(ed => { const s = $('#select_' + ed); if (s) s.value = ''; });
    selectedCast = [];
    $('#results').innerText = '';
    renderTrackRecord();
  }
  if (e.target.id === 'resetHistory') resetAllHistory();
});


