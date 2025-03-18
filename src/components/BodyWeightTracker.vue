<script setup>
import { ref, watch, onMounted } from "vue";
import { db } from "../firebaseConfig";
import { collection, getDocs, setDoc, doc, query, orderBy } from "firebase/firestore";
import { Chart, registerables } from "chart.js";
import { onMounted as onChartMounted } from "vue";

Chart.register(...registerables);

const weeks = ref([]); // Speichert die Wochen als Spalten
const weekData = ref({}); // Speichert das Gewicht für jede Woche + Tag
const isLoading = ref(false);

const weekDays = ["MO", "DI", "MI", "DO", "FR", "SA", "SO"];

// 📌 Firebase-Daten abrufen
const fetchData = async () => {
  isLoading.value = true;
  const q = query(collection(db, "bodyWeight"), orderBy("week", "asc"));
  const querySnapshot = await getDocs(q);

  querySnapshot.forEach((doc) => {
    const data = doc.data();
    if (!weeks.value.includes(data.week)) {
      weeks.value.push(data.week);
    }
    weekData.value[data.week] = data.days || {};
  });

  isLoading.value = false;
};

// 📌 Firebase-Daten speichern
const saveData = async (week) => {
  await setDoc(doc(db, "bodyWeight", week.toString()), {
    week: week,
    days: weekData.value[week]
  });
};

// 📌 Neue Woche hinzufügen
const addWeek = () => {
  const newWeek = weeks.value.length > 0 ? Math.max(...weeks.value) + 1 : 1;
  weeks.value.push(newWeek);
  weekData.value[newWeek] = {}; // Leere Woche vorbereiten
};

// 📌 Durchschnittsgewicht berechnen
const getAverageWeight = (week) => {
  const weights = Object.values(weekData.value[week] || {}).map(Number).filter((w) => !isNaN(w));
  return weights.length > 0 ? (weights.reduce((a, b) => a + b, 0) / weights.length).toFixed(1) : "-";
};

// 📌 Chart.js Diagramm initialisieren
const chartCanvas = ref(null);
let weightChart;

onChartMounted(() => {
  weightChart = new Chart(chartCanvas.value, {
    type: "line",
    data: {
      labels: weeks.value.map((w) => `Woche ${w}`),
      datasets: [
        {
          label: "",
          data: weeks.value.map((w) => getAverageWeight(w)),
          borderColor: "#a4d3f4",
          borderWidth: 4, // Dicke der Linie
          pointBackgroundColor: "#d8e488", // Punkte einfärben
          pointBorderColor: "#d8e488",
          tension: 0.4
        }
      ]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      plugins: {
        legend: {
          display: false // ❌ Legende komplett deaktivieren
        }
      },
    
    scales: {
        x: {
          grid: {
            color: "#fff", // 🟡 X-Achsen-Gitterlinienfarbe (Grau)
            lineWidth: 0.5 // Dünne Linien
          },
          ticks: {
            color: "#fff", // ⚫️ X-Achsen-Text dunkler machen
            font: { size: 9 } // Schriftgröße anpassen
          }
        },
        y: {
          grid: {
            color: "#fff", // ⚪️ Y-Achsen-Gitterlinien heller machen
            lineWidth: 0.5 // Dickere Linien
          },
          ticks: {
            color: "#fff", // ⚫️ Y-Achsen-Textfarbe anpassen
            font: { size: 9 } // Schriftgröße anpassen
          }
        }
      }
    }
  });
});

// 📌 Firebase-Daten abrufen, wenn die Komponente geladen wird
onMounted(fetchData);

// 📌 Watcher: Aktualisiert das Diagramm bei Änderungen
watch(weekData, () => {
  weightChart.data.labels = weeks.value.map((w) => `W${w}`);
  weightChart.data.datasets[0].data = weeks.value.map((w) => getAverageWeight(w));
  weightChart.update();
}, { deep: true });

</script>

<template>
  <div>

    <div class="table-container">
      <table>
        <thead>
          <tr>
            <th class="sticky-left">Tag</th>
            <th v-for="week in weeks" :key="week">W{{ week }}</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="day in weekDays" :key="day">
            <td class="sticky-left">{{ day }}</td>
            <td v-for="week in weeks" :key="week">
              <input
                type="number"
                v-model="weekData[week][day]"
                @input="saveData(week)"
              />
            </td>
          </tr>
        </tbody>
        <tfoot>
          <tr>
            <td class="sticky-left durchschnitt">Ø</td>
            <td v-for="week in weeks" :key="week">
              {{ getAverageWeight(week) }}
            </td>
          </tr>
        </tfoot>
      </table>
    </div>

    <!-- Button zum Hinzufügen neuer Wochen -->
    <div class="button-container">
      <button @click="addWeek">+ Neue Woche</button>
    </div>

    <!-- Diagramm -->
    <div class="chart-container">
      <canvas ref="chartCanvas"></canvas>
    </div>

    <p v-if="isLoading">Lade Daten...</p>
  </div>
</template>

<style scoped>
.table-container {
  overflow-x: auto; /* Erlaubt horizontales Scrollen */
  white-space: nowrap; /* Verhindert Umbrüche */
  max-width: 80vw;
}

table {
  border-collapse: separate;
  border-spacing: 10px;
  width: auto; /* Automatische Breite */
  
}

th, td {
  padding: 0.2rem;
  text-align: center;
  background: var(--dark);
  font-size: 1.2rem;
  color: var(--white);
  border-radius: 5px;
}

th {
  background: var(--grey);
  position: sticky;
  top: 0;
  z-index: 2;
  border-radius: 5px;
  padding: 0.8rem;
}

.sticky-left {
  position: sticky;
  left: 0;
  background: var(--grey);
  color: var(--white);
  z-index: 5;
  backdrop-filter: blur(5px);
}

.durchschnitt{
    padding: 0.8rem;
}

input {
    background-color: var(--dark);
    color: var(--white);
    border: none;
    border-radius: 5px;
    padding: 0.5rem;
    font-family: "Shrikhand", cursive;
    text-align: center;
    width: 5rem;
    font-size: 1.2rem;
}

input::-webkit-outer-spin-button,
input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

input:focus {
  outline: none;
}

.button-container {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

button {
    cursor: pointer;
    background-color: var(--grey);
    color: var(--white);
    border: none;
    border-radius: 5px;
    padding: 0.5rem;
    font-family: "Shrikhand", cursive;
    font-size: 1.2rem;
    width: auto;
}

.chart-container {
  width: 100%;
  height: 300px;
  margin-top: 20px;
  max-width: 80vw;
  background-color: var(--dark);
  padding: 1rem;
  border-radius: 10px;
}
</style>
