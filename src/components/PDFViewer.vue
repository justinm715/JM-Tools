<template>
  <div>
    <input type="file" @change="loadPdf" accept="application/pdf" />
    <div v-for="(zoom, index) in zooms" :key="index" class="mb-4">
      <div class="flex justify-between items-center mb-2">
        <span>Page {{ index + 1 }}</span>
        <div>
          <button @click="() => zoomIn(index)" class="mr-2">Zoom In</button>
          <button @click="() => zoomOut(index)">Zoom Out</button>
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
import * as pdf from "../assets/pdf.mjs";

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
  pdfjsLib.GlobalWorkerOptions.workerSrc =
    "//mozilla.github.io/pdf.js/build/pdf.worker.mjs";

  const arrayBuffer = await file.arrayBuffer();
  pdfDoc = await pdfjsLib.getDocument(new Uint8Array(arrayBuffer)).promise;
  totalPages.value = pdfDoc.numPages;
  zooms.value = Array(totalPages.value).fill(1.0);

  for (let i = 1; i <= totalPages.value; i++) {
    displayPage(i);
  }
};

const displayPage = async (pageNum) => {
  const page = await pdfDoc.getPage(pageNum);
  const viewport = page.getViewport({ scale: zooms.value[pageNum - 1] });

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
  zooms.value[index] += 0.25;
  displayPage(index + 1);
};

const zoomOut = (index) => {
  zooms.value[index] -= 0.25;
  displayPage(index + 1);
};
</script>

<style scoped>
.pdf-viewer {
}
</style>
