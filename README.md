# kilometerregistratiesysteem
!DOCTYPE html>
<html lang="nl">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Kilometerregistratie</title>
  <style>
    body {
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      background-color: #f0f4f8;
      margin: 0;
      padding: 2rem;
      color: #333;
    }
    h1, h3 {
      color: #2c3e50;
    }
    form, .filters, .tabel-container, .totaal-container {
      background-color: white;
      padding: 1.5rem;
      margin-bottom: 1.5rem;
      border-radius: 10px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    }
    input, button, select {
      padding: 0.6rem;
      margin: 0.4rem 0.5rem 0.4rem 0;
      border-radius: 6px;
      border: 1px solid #ccc;
    }
    button {
      background-color: #3498db;
      color: white;
      border: none;
      cursor: pointer;
    }
    button:hover {
      background-color: #2980b9;
    }
    table {
      border-collapse: collapse;
      width: 100%;
    }
    th, td {
      border: 1px solid #ddd;
      padding: 0.75rem;
      text-align: left;
    }
    th {
      background-color: #ecf0f1;
    }
    .actions button {
      background-color: #e74c3c;
    }
    .actions button:hover {
      background-color: #c0392b;
    }
    ul#perDagList {
      list-style-type: none;
      padding-left: 0;
    }
    ul#perDagList li {
      background: #fff;
      margin: 0.3rem 0;
      padding: 0.5rem;
      border-radius: 6px;
      box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }
  </style>
  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.11.0/firebase-app.js";
    import { getFirestore, collection, addDoc, getDocs, deleteDoc, doc, updateDoc } from "https://www.gstatic.com/firebasejs/10.11.0/firebase-firestore.js";

    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_AUTH_DOMAIN",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_STORAGE_BUCKET",
      messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
      appId: "YOUR_APP_ID"
    };

    const app = initializeApp(firebaseConfig);
    const db = getFirestore(app);

    const form = document.getElementById('ritForm');
    const tabelBody = document.querySelector('#ritTabel tbody');
    const totaalSpan = document.getElementById('totaal');
    const filterDatum = document.getElementById('filterDatum');
    const sortSelect = document.getElementById('sorteer');
    const perDagList = document.getElementById('perDagList');

    let ritten = [];

    async function loadRitten() {
      const querySnapshot = await getDocs(collection(db, 'ritten'));
      ritten = [];
      querySnapshot.forEach((docSnap) => {
        ritten.push({ ...docSnap.data(), id: docSnap.id });
      });
      updateTabel();
      updatePerDag();
    }

    function berekenKilometers(rit) {
      if (rit.kilometers && rit.kilometers !== '') return parseFloat(rit.kilometers);
      const begin = parseFloat(rit.beginstand);
      const eind = parseFloat(rit.eindstand);
      if (!isNaN(begin) && !isNaN(eind)) {
        const verschil = eind - begin;
        return verschil >= 0 ? verschil : 0;
      }
      return 0;
    }

    function updateTabel(data = ritten) {
      tabelBody.innerHTML = '';
      let totaal = 0;
      data.forEach((rit) => {
        const kilometers = berekenKilometers(rit);
        const rij = document.createElement('tr');
        rij.innerHTML = `
          <td><input type="date" value="${rit.datum}" onchange="bewerkRit('${rit.id}', 'datum', this.value)"></td>
          <td><input type="time" value="${rit.begintijd || ''}" onchange="bewerkRit('${rit.id}', 'begintijd', this.value)"></td>
          <td><input type="number" value="${rit.beginstand || ''}" onchange="bewerkRit('${rit.id}', 'beginstand', this.value)"></td>
          <td><input type="text" value="${rit.vertrek}" onchange="bewerkRit('${rit.id}', 'vertrek', this.value)"></td>
          <td><input type="time" value="${rit.eindtijd || ''}" onchange="bewerkRit('${rit.id}', 'eindtijd', this.value)"></td>
          <td><input type="number" value="${rit.eindstand || ''}" onchange="bewerkRit('${rit.id}', 'eindstand', this.value)"></td>
          <td><input type="text" value="${rit.opdrachtnummers || ''}" onchange="bewerkRit('${rit.id}', 'opdrachtnummers', this.value)"></td>
          <td><input type="number" value="${rit.kilometers}" onchange="bewerkRit('${rit.id}', 'kilometers', this.value)"></td>
          <td class="actions">
            <button onclick="verwijderRit('${rit.id}')">Verwijderen</button>
          </td>
        `;
        totaal += kilometers;
        tabelBody.appendChild(rij);
      });
      totaalSpan.textContent = totaal.toFixed(2);
    }

    function updatePerDag() {
      const kmPerDag = {};
      ritten.forEach(rit => {
        const km = berekenKilometers(rit);
        if (!kmPerDag[rit.datum]) kmPerDag[rit.datum] = 0;
        kmPerDag[rit.datum] += km;
      });

      perDagList.innerHTML = '';
      for (const datum in kmPerDag) {
        const li = document.createElement('li');
        li.textContent = `${datum}: ${kmPerDag[datum].toFixed(2)} km`;
        perDagList.appendChild(li);
      }
    }

    window.verwijderRit = async (id) => {
      await deleteDoc(doc(db, 'ritten', id));
      loadRitten();
    };

    window.bewerkRit = async (id, veld, waarde) => {
      const ritDoc = doc(db, 'ritten', id);
      await updateDoc(ritDoc, { [veld]: waarde });
      loadRitten();
    };

    form.addEventListener('submit', async (e) => {
      e.preventDefault();
      const beginstand = parseFloat(document.getElementById('beginstand').value);
      const eindstand = parseFloat(document.getElementById('eindstand').value);
      let kilometers = document.getElementById('kilometers').value;

      if ((!kilometers || kilometers === '') && !isNaN(beginstand) && !isNaN(eindstand)) {
        kilometers = Math.max(0, eindstand - beginstand);
      }

      const rit = {
        datum: document.getElementById('datum').value,
        begintijd: document.getElementById('begintijd').value,
        beginstand: document.getElementById('beginstand').value,
        vertrek: document.getElementById('vertrek').value,
        eindtijd: document.getElementById('eindtijd').value,
        eindstand: document.getElementById('eindstand').value,
        opdrachtnummers: document.getElementById('opdrachtnummers').value,
        kilometers: kilometers
      };

      await addDoc(collection(db, 'ritten'), rit);
      loadRitten();
      form.reset();
    });

    window.filterOpDatum = () => {
      const filter = filterDatum.value;
      if (filter) {
        const gefilterd = ritten.filter(rit => rit.datum === filter);
        updateTabel(gefilterd);
      }
    };

    window.resetFilter = () => {
      filterDatum.value = '';
      updateTabel();
    };

    window.exportCSV = () => {
      let csv = 'Datum,Begintijd,Beginstand,Vertrek,Eindtijd,Eindstand,Opdrachtnummers,Kilometers\n';
      ritten.forEach(rit => {
        const km = berekenKilometers(rit);
        csv += `${rit.datum},${rit.begintijd || ''},${rit.beginstand || ''},${rit.vertrek},${rit.eindtijd || ''},${rit.eindstand || ''},${rit.opdrachtnummers || ''},${km}\n`;
      });
      const blob = new Blob([csv], { type: 'text/csv' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'kilometerregistratie.csv';
      a.click();
    };

    window.sorteerRitten = () => {
      const criterium = sortSelect.value;
      ritten.sort((a, b) => {
        if (criterium === 'datum') return a.datum.localeCompare(b.datum);
        if (criterium === 'kilometers') return berekenKilometers(b) - berekenKilometers(a);
        return 0;
      });
      updateTabel();
    };

    window.onload = loadRitten;
  </script>
</head>
<body>
  <h1>Kilometerregistratie</h1>

  <form id="ritForm">
    <input type="date" id="datum" required>
    <input type="time" id="begintijd" placeholder="Begintijd">
    <input type="number" id="beginstand" placeholder="Beginstand km">
    <input type="text" id="vertrek" placeholder="Vertrek" required>
    <input type="time" id="eindtijd" placeholder="Eindtijd">
    <input type="number" id="eindstand" placeholder="Eindstand km">
    <input type="text" id="opdrachtnummers" placeholder="Opdrachtnummers">
    <input type="number" id="kilometers" placeholder="Kilometers">
    <button type="submit">Toevoegen</button>
  </form>

  <div class="filters">
    <label for="filterDatum">Filter op datum:</label>
    <input type="date" id="filterDatum">
    <button onclick="filterOpDatum()">Filter</button>
    <button onclick="resetFilter()">Reset</button>

    <label for="sorteer">Sorteren op:</label>
    <select id="sorteer" onchange="sorteerRitten()">
      <option value="datum">Datum</option>
      <option value="kilometers">Kilometers (aflopend)</option>
    </select>

    <button onclick="exportCSV()">Exporteer CSV</button>
  </div>

  <div class="tabel-container">
    <table id="ritTabel">
      <thead>
        <tr>
          <th>Datum</th>
          <th>Begintijd</th>
          <th>Beginstand</th>
          <th>Vertrek</th>
          <th>Eindtijd</th>
          <th>Eindstand</th>
          <th>Opdrachtnummers</th>
          <th>Kilometers</th>
          <th>Acties</th>
        </tr>
      </thead>
      <tbody></tbody>
    </table>
  </div>

  <div class="totaal-container">
    <h3>Totaal kilometers: <span id="totaal">0</span></h3>
    <h3>Afstand per dag:</h3>
    <ul id="perDagList"></ul>
  </div>
</body>
</html>
