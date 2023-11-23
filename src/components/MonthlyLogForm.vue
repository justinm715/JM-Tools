<template>
  <div class="container mx-auto">
    <div class="mb-4">
      <select v-model="selectedMonth" class="border p-1 mr-2">
        <option v-for="month in months" :key="month.value" :value="month.value">
          {{ month.text }}
        </option>
      </select>
      <select v-model="selectedYear" class="border p-1">
        <option v-for="year in yearRange" :key="year" :value="year">
          {{ year }}
        </option>
      </select>
      <button
        @click="generateEntries"
        class="bg-blue-500 hover:bg-blue-700 text-white px-3 ml-2"
      >
        Generate
      </button>
    </div>
    <component :is="QuillEditor" v-if="QuillEditor" ref="myEditor" />
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import "@vueup/vue-quill/dist/vue-quill.snow.css";

const QuillEditor = ref(null);
const myEditor = ref(null);

const selectedMonth = ref(new Date().getMonth());
const selectedYear = ref(new Date().getFullYear());
const months = [
  { text: "January", value: 0 },
  { text: "February", value: 1 },
  { text: "March", value: 2 },
  { text: "April", value: 3 },
  { text: "May", value: 4 },
  { text: "June", value: 5 },
  { text: "July", value: 6 },
  { text: "August", value: 7 },
  { text: "September", value: 8 },
  { text: "October", value: 9 },
  { text: "November", value: 10 },
  { text: "December", value: 11 },
];

const yearRange = ref([]);

const initializeYearRange = () => {
  const currentYear = new Date().getFullYear();
  for (let i = currentYear - 5; i <= currentYear + 5; i++) {
    yearRange.value.push(i);
  }
};

function generateAsciiCalendar(year, month) {
  const weekdays = ["Su", "Mo", "Tu", "We", "Th", "Fr", "Sa"];
  let calendar = `<pre>${new Date(year, month, 1).toLocaleString("default", {
    month: "long",
  })} ${year}\n\n`;
  calendar += `${weekdays.join(" ")}\n`;

  const startDate = new Date(year, month, 1);
  const startDay = startDate.getDay();
  const daysInMonth = new Date(year, month + 1, 0).getDate();

  for (let i = 0; i < startDay; i++) {
    calendar += "   ";
  }

  for (let day = 1; day <= daysInMonth; day++) {
    calendar += day.toString().padStart(2, " ") + " ";
    if ((day + startDay) % 7 === 0) {
      calendar += "\n";
    }
  }

  calendar += "</pre><br /><br />";
  return calendar;
}

const generateEntries = () => {
  const asciiCalendar = generateAsciiCalendar(
    selectedYear.value,
    selectedMonth.value,
  );
  let entries = asciiCalendar;

  const daysInMonth = new Date(
    selectedYear.value,
    selectedMonth.value + 1,
    0,
  ).getDate();

  for (let day = 1; day <= daysInMonth; day++) {
    const date = new Date(selectedYear.value, selectedMonth.value, day);
    const dayString = date.toISOString().split("T")[0];
    const dayName = date.toLocaleDateString(undefined, { weekday: "long" });
    entries +=
      `<h2 style="font-size: 1.25rem">${dayString} ${dayName}</h2>` +
      `<ol style="margin-left:1rem">` +
      `<li>Time in XXX` +
      `</li>` +
      `<li>Item</li>` +
      `</ol>` +
      `<br /><br />`;
  }
  myEditor.value.setHTML(entries);
};

onMounted(async () => {
  initializeYearRange();
  if (import.meta.env.SSR) return;
  import("@vueup/vue-quill").then((module) => {
    QuillEditor.value = module.QuillEditor;
  });
});
</script>

<style>

</style>
