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
          <button @click="() => zoomOut(index)" class="w-10 mr-2">-</button>
          <span class="font-bold text-sm mr-4 w-24">{{ zoom }}%</span>
          <button @click="() => zoomIn(index)" class="w-10">+</button>
        </div>
      </div>
      <div class="flex">
        <div
          ref="pdfViewers"
          class="pdf-viewer relative border border-gray-100 overflow-scroll h-[32rem] max-h-[32rem] basis-2/3"
        >
          <canvas ref="pdfCanvases" class="absolute top-0 left-0"></canvas>
          <div
            ref="svgOverlays"
            class="svgContainer absolute top-0 left-0"
          ></div>
        </div>
        <!-- Metadata Column -->
        <div class="metadata flex-none basis-1/3">
          <p>Metadata for Page {{ index + 1 }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import * as d3 from "d3";

const pdfViewers = ref([]);
const pdfCanvases = ref([]);
let pdfDoc = null;
const zooms = ref([]);
const totalPages = ref(0);

onMounted(() => {
  let pdfjsScript = document.createElement("script");
  pdfjsScript.setAttribute(
    "src",
    "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.269/pdf.min.mjs",
  );
  pdfjsScript.setAttribute("type", "module");
  document.head.appendChild(pdfjsScript);
});

const loadPdf = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const { pdfjsLib } = globalThis;
  pdfjsLib.GlobalWorkerOptions.workerSrc =
    "//cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.269/pdf.worker.min.mjs";

  const arrayBuffer = await file.arrayBuffer();
  pdfDoc = await pdfjsLib.getDocument(new Uint8Array(arrayBuffer)).promise;
  totalPages.value = pdfDoc.numPages;
  zooms.value = Array(totalPages.value).fill(100);

  for (let i = 1; i <= totalPages.value; i++) {
    displayPage(i);
  }
};

const displayPage = async (pageNum) => {
  const page = await pdfDoc.getPage(pageNum);
  const scale = zooms.value[pageNum - 1] / 100;
  const viewport = page.getViewport({ scale });

  // Canvas setup
  const pdfCanvas = pdfCanvases.value[pageNum - 1];
  pdfCanvas.width = viewport.width;
  pdfCanvas.height = viewport.height;
  const context = pdfCanvas.getContext("2d");

  // Render PDF page
  const renderContext = { canvasContext: context, viewport };
  await page.render(renderContext).promise;

  // SVG setup
  createOrUpdateSvg(pageNum, scale);
};

const createOrUpdateSvg = (pageNum, scale) => {
  const svgContainer = document.querySelectorAll(".svgContainer")[pageNum - 1];
  svgContainer.innerHTML = ""; // Clear existing SVG to redraw

  const svg = d3
    .create("svg")
    .attr("width", pdfCanvases.value[pageNum - 1].width)
    .attr("height", pdfCanvases.value[pageNum - 1].height);

  svg
    .append("circle")
    .attr("cx", 100 * scale)
    .attr("cy", 200 * scale)
    .attr("r", 25 * scale) // radius adjusted for scale
    .attr("fill", "blue");

  svgContainer.appendChild(svg.node());
};

const zoomIn = (index) => {
  if (zooms.value[index] < 400) {
    zooms.value[index] += 25;
    displayPage(index + 1);
  }
};

const zoomOut = (index) => {
  if (zooms.value[index] > 25) {
    zooms.value[index] -= 25;
    displayPage(index + 1);
  }
};
</script>

<style scoped>
.pdf-viewer {
  /* Add styles for the PDF viewer here */
}
</style>
