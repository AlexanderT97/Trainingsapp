<script setup>
import { ref, watch, onMounted } from "vue";
import { db } from "../firebaseConfig";
import { collection, addDoc, getDocs, setDoc, doc, query, orderBy } from "firebase/firestore";

const props = defineProps({
  workoutName: String // Name der Firebase-Collection
});

const tableData = ref([{ weight: ["", "", ""], reps: ["", "", ""] }]);
const isLoading = ref(false);

// 🔥 Daten aus Firebase abrufen, aufsteigend nach ID sortieren
const fetchData = async () => {
  isLoading.value = true;
  const q = query(collection(db, props.workoutName), orderBy("id", "asc")); // 🔥 Aufsteigende Sortierung
  const querySnapshot = await getDocs(q);
  tableData.value = querySnapshot.docs.map(doc => ({ id: doc.data().id, ...doc.data() })) || [];
  isLoading.value = false;
};

onMounted(fetchData);

// 🔥 Automatisch speichern mit fortlaufender ID
watch(tableData, async (newData) => {
  newData.forEach(async (row) => {
    if (!row.id) {
      const maxId = tableData.value.length > 0 ? Math.max(...tableData.value.map(r => r.id || 0)) : 0;
      row.id = maxId + 1; // 🔥 Neue ID = letzte ID + 1
      await setDoc(doc(db, props.workoutName, row.id.toString()), row);
    } else {
      await setDoc(doc(db, props.workoutName, row.id.toString()), row);
    }
  });
}, { deep: true });

// 🔥 10 neue Zeilen hinzufügen mit korrekter ID
const addRows = () => {
  const maxId = tableData.value.length > 0 ? Math.max(...tableData.value.map(r => r.id || 0)) : 0;
  for (let i = 1; i <= 10; i++) {
    tableData.value.push({ id: maxId + i, weight: ["", "", ""], reps: ["", "", ""] });
  }
};
</script>

<template>
  <div>
    <table>
      <thead>
        <tr>
          <th>Kilo</th>
          <th>Reps</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="row in tableData" :key="row.id">
          <td>
            <input v-for="(w, i) in row.weight" :key="i" type="number" v-model="row.weight[i]" />
          </td>
          <td>
            <input v-for="(r, i) in row.reps" :key="i" type="number" v-model="row.reps[i]" />
          </td>
        </tr>
      </tbody>
    </table>

    <div class="button-container">
        <button @click="addRows">+ 10 Zeilen hinzufügen</button>
        <p v-if="isLoading">Lade Daten...</p>
    </div>
  </div>
</template>



<style scoped>
table {
  width: 100%;
  margin-bottom: 1rem;
  border-spacing: 0.5rem;
}
th, td {
  padding: 0.5rem;
  text-align: center;
}

thead{
  position: sticky;
  top: 0.2rem;
  z-index: 10;
}

th{
    background-color: var(--grey);
    color: var(--white);
    font-size: 1.5rem;
    border-radius: 5px;
    letter-spacing: 2px;
}
input {
    background-color: var(--dark);
    color: var(--white);
    border: none;
    border-radius: 5px;
    padding: 0.2rem;
    font-family: "Shrikhand", cursive;
    text-align: center;
    width: 100%;
    margin: 0.2rem;
    font-size: 1.2rem;
}

input::-webkit-outer-spin-button,
input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

input:focus {
  outline: var(--white);
}

.button-container {
  display: flex;
  justify-content: center;
  margin-bottom: 4rem;
  flex-direction: column;
  align-items: center;
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

@media screen and (min-width: 768px) {
  input {
    width: 4rem;
  }
}
</style>
