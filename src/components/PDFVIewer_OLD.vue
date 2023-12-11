<template>
  <div>
    <input type="file" @change="loadPdf" accept="application/pdf" />
    <div
      v-for="(zoom, index) in zooms"
      :key="index"
      class="mb-3 border-b-2 pb-3"
    >
      <div class="flex justify-between items-center mb-2">
        <span>Page {{ index + 1 }}</span>
        <div>
          <span class="font-bold text-sm mr-4 w-24">{{ zoom }}%</span>
          <button @click="() => zoomOut(index)" class="w-10 mr-2">-</button>
          <button @click="() => zoomIn(index)" class="w-10">+</button>
        </div>
      </div>
      <div class="flex">
        <!-- Metadata Column -->
        <div class="metadata flex-none basis-1/3">
          <!-- Page metadata goes here -->
          <p>Metadata for Page {{ index + 1 }}</p>
          <!-- Add more metadata as needed -->
        </div>

        <!-- PDF Viewer Column -->
        <div
          ref="pdfViewers"
          class="pdf-viewer border border-gray-100 overflow-scroll max-h-[32rem] basis-2/3"
        ></div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from "vue";
import * as pdf from "../../public/pdfjs/pdf.mjs";

const pdfViewers = ref([]);
let pdfDoc = null;
const zooms = ref([]);
const totalPages = ref(0);

const setPdfViewer = (el) => {
  if (el) {
    pdfViewers.value.push(el);
  }
};

onMounted(() => {
  // Initialize or load resources if needed
});

const loadPdf = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  var { pdfjsLib } = globalThis;

  // The workerSrc property shall be specified.
  pdfjsLib.GlobalWorkerOptions.workerSrc = "../../public/pdfjs/pdf.worker.mjs";

  const arrayBuffer = await file.arrayBuffer();
  pdfDoc = await pdfjsLib.getDocument(new Uint8Array(arrayBuffer)).promise;
  totalPages.value = pdfDoc.numPages;
  zooms.value = Array(totalPages.value).fill(100); // Initialize with 100%

  for (let i = 1; i <= totalPages.value; i++) {
    displayPage(i);
  }
};

const displayPage = async (pageNum) => {
  const page = await pdfDoc.getPage(pageNum);
  const viewport = page.getViewport({ scale: zooms.value[pageNum - 1] / 100 });

  const canvas = document.createElement("canvas");
  const context = canvas.getContext("2d");
  canvas.height = viewport.height;
  canvas.width = viewport.width;
  canvas.classList = "border border-gray-300";

  pdfViewers.value[pageNum - 1].innerHTML = "";
  pdfViewers.value[pageNum - 1].appendChild(canvas);

  const renderContext = {
    canvasContext: context,
    viewport: viewport,
  };

  await page.render(renderContext).promise;
};

const zoomIn = (index) => {
  zooms.value[index] += 25; // Increase zoom by 25%
  displayPage(index + 1);
};

const zoomOut = (index) => {
  if (zooms.value[index] > 25) {
    // Prevent zooming out below 25%
    zooms.value[index] -= 25; // Decrease zoom by 25%
  }
  displayPage(index + 1);
};
</script>

<style scoped>
.pdf-viewer {
}
</style>
