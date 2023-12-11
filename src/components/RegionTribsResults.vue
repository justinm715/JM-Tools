<template>
  <div>
    <div
      v-for="(zoom, index) in zooms"
      :key="index"
      class="mb-3 border-b-2 pb-3"
    >
      <div class="flex justify-between items-center mb-2">
        <h3>Page {{ index + 1 }}</h3>
        <div>
          <button @click="() => zoomOut(index)" class="w-10 mr-2">-</button>
          <span class="font-bold text-sm mr-4 w-24">{{ zoom }}%</span>
          <button @click="() => zoomIn(index)" class="w-10">+</button>
        </div>
      </div>
      <div class="flex page-results">
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
          <div v-if="results[index] != null" class="text-xs">
            <div
              v-for="(regionData, regionKey) in results[index]"
              :key="regionKey"
              class="mb-4"
            >
              <span
                class="block font-bold hover:bg-blue-100"
                :data-pdf-coords="regionData.PDFCoords"
                :data-page-index="index"
                @mouseover="highlightArea"
                @mouseout="clearHighlights"
                >{{ regionKey }}</span
              >
              <table>
                <tbody>
                  <tr
                    v-for="(
                      roofIntersectionData, roofIntersectionKey
                    ) in regionData['roof_intersections']"
                    :key="roofIntersectionKey"
                  >
                    <td class="w-24">
                      <span class="px-1">{{ roofIntersectionKey }}</span>
                    </td>
                    <td class="w-24">
                      <ol>
                        <li
                          v-for="(
                            intersectionData, intersectionDataIndex
                          ) in roofIntersectionData"
                          :key="intersectionDataIndex"
                          :data-pdf-coords="intersectionData.PDFCoords"
                          :data-page-index="index"
                          @mouseover="highlightArea"
                          @mouseout="clearHighlights"
                          class="hover:bg-blue-100 text-right px-2"
                        >
                          {{ parseFloat(intersectionData.area).toFixed(2) }}
                          sf
                        </li>
                        <li
                          v-if="roofIntersectionData.length > 1"
                          class="border-t border-gray-300 text-right px-2 ml-6"
                        >
                          Totals:
                          {{ totalIntersection(roofIntersectionData, "area") }}
                          sf
                        </li>
                      </ol>
                    </td>
                  </tr>
                </tbody>
              </table>

              <table>
                <tbody>
                  <tr
                    v-for="(
                      floorIntersectionData, floorIntersectionKey
                    ) in regionData['floor_intersections']"
                    :key="floorIntersectionKey"
                  >
                    <td class="w-24">
                      <span class="px-1">{{ floorIntersectionKey }}</span>
                    </td>
                    <td class="w-24">
                      <ol>
                        <li
                          v-for="(
                            intersectionData, intersectionDataIndex
                          ) in floorIntersectionData"
                          :key="intersectionDataIndex"
                          :data-pdf-coords="intersectionData.PDFCoords"
                          :data-page-index="index"
                          @mouseover="highlightArea"
                          @mouseout="clearHighlights"
                          class="hover:bg-blue-100 text-right px-2"
                        >
                          {{ parseFloat(intersectionData.area).toFixed(2) }}
                          sf
                        </li>
                        <li
                          v-if="floorIntersectionData.length > 1"
                          class="border-t border-gray-300 text-right px-2 ml-6"
                        >
                          Total:
                          {{ totalIntersection(floorIntersectionData, "area") }}
                          sf
                        </li>
                      </ol>
                    </td>
                  </tr>
                </tbody>
              </table>

              <table>
                <tbody>
                  <tr
                    v-for="(
                      wallIntersectionData, wallIntersectionKey
                    ) in regionData['wall_intersections']"
                    :key="wallIntersectionKey"
                  >
                    <td class="w-24">
                      <span class="px-1">{{ wallIntersectionKey }}</span>
                    </td>
                    <td class="w-24">
                      <ol>
                        <li
                          v-for="(
                            intersectionData, intersectionDataIndex
                          ) in wallIntersectionData"
                          :key="intersectionDataIndex"
                          :data-pdf-coords="intersectionData.PDFCoords"
                          :data-page-index="index"
                          @mouseover="highlightLine"
                          @mouseout="clearHighlights"
                          class="hover:bg-blue-100 text-right px-2"
                        >
                          {{ parseFloat(intersectionData.length).toFixed(2) }}
                          ft
                        </li>
                        <li
                          v-if="wallIntersectionData.length > 1"
                          class="border-t border-gray-300 text-right px-2 ml-2"
                        >
                          Total:
                          {{
                            totalIntersection(wallIntersectionData, "length")
                          }}
                          ft
                        </li>
                      </ol>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { defineProps } from "vue";

import { ref, onMounted } from "vue";
import * as d3 from "d3";

const pdfViewers = ref([]);
const pdfCanvases = ref([]);
let pdfDoc = null;
const zooms = ref([]);
const totalPages = ref(0);

const props = defineProps({
  file: {
    type: Object,
    required: true,
  },
  results: {
    type: Object,
    required: true,
  },
});

onMounted(() => {
  let pdfjsScript = document.createElement("script");
  pdfjsScript.setAttribute(
    "src",
    "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.269/pdf.min.mjs",
  );
  pdfjsScript.setAttribute("type", "module");
  document.head.appendChild(pdfjsScript);
  pdfjsScript.onload = () => {
    // Now that the script is loaded, call loadPdf
    loadPdf();
  };
});

const loadPdf = async () => {
  const { pdfjsLib } = globalThis;
  pdfjsLib.GlobalWorkerOptions.workerSrc =
    "//cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.269/pdf.worker.min.mjs";

  const arrayBuffer = await props.file.arrayBuffer();
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
  svgContainer.appendChild(svg.node());
};

const blink = (element, maxOpacity = 1) => {
  element
    .transition()
    .duration(250) // Duration of each blink in milliseconds
    .attr("opacity", 0) // Make the element invisible
    .transition()
    .duration(250)
    .attr("opacity", maxOpacity) // Make the element visible again
    .on("end", () => blink(element, maxOpacity)); // Repeat this process
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

const convertCoordsToD3PolylinePointsStr = (pdfCoords, canvasHeight, scale) => {
  const numbers = pdfCoords.split(",").map(Number);
  let points = [];
  for (let i = 0; i < numbers.length; i += 2) {
    let x = scale * numbers[i];
    let y = canvasHeight - scale * numbers[i + 1]; // Flip the y-coordinate
    points.push(`${x},${y}`);
  }
  return points.join(" ");
};

const highlightArea = (event) => {
  let el = event.target;
  let pdfCoords = el.dataset.pdfCoords; // Assuming pdfCoords is a JSON string
  let highlightType = el.dataset.pdfHighlightType;
  let pageIndex = el.dataset.pageIndex;
  let scale = zooms.value[pageIndex] / 100;

  const svgContainer = el
    .closest(".page-results")
    .querySelector(".svgContainer");
  svgContainer.innerHTML = ""; // Clear existing SVG to redraw
  const canvasHeight = pdfCanvases.value[pageIndex].height;
  const polylinePointsStr = convertCoordsToD3PolylinePointsStr(
    pdfCoords,
    canvasHeight,
    scale,
  );

  const svg = d3
    .create("svg")
    .attr("width", pdfCanvases.value[pageIndex].width)
    .attr("height", pdfCanvases.value[pageIndex].height);
  const polyline = svg
    .append("polygon")
    .attr("points", polylinePointsStr)
    .attr("stroke", "green") // or use highlightType to determine the color
    .attr("stroke-width", 6)
    .attr("fill", "yellow");

  blink(polyline, 0.5); // Make the polyline blink
  svgContainer.appendChild(svg.node());
};

const highlightLine = (event) => {
  let el = event.target;
  let pdfCoords = el.dataset.pdfCoords; // Assuming pdfCoords is a JSON string
  let pageIndex = el.dataset.pageIndex;
  let scale = zooms.value[pageIndex] / 100;

  const svgContainer = el
    .closest(".page-results")
    .querySelector(".svgContainer");
  svgContainer.innerHTML = ""; // Clear existing SVG to redraw
  const canvasHeight = pdfCanvases.value[pageIndex].height;
  const polylinePointsStr = convertCoordsToD3PolylinePointsStr(
    pdfCoords,
    canvasHeight,
    scale,
  );

  const svg = d3
    .create("svg")
    .attr("width", pdfCanvases.value[pageIndex].width)
    .attr("height", pdfCanvases.value[pageIndex].height);
  const polyline = svg
    .append("polyline")
    .attr("points", polylinePointsStr)
    .attr("stroke", "green") // or use highlightType to determine the color
    .attr("stroke-width", 6)
    .attr("fill", "none");

  blink(polyline); // Make the polyline blink
  svgContainer.appendChild(svg.node());
};

const totalIntersection = (arr, property = "length") => {
  // property could be "length" or "area"
  return parseFloat(
    arr.reduce((total, current) => {
      return total + (current[property] || 0);
    }, 0),
  ).toFixed(2);
};

const clearHighlights = (event) => {
  let el = event.target;
  let pageIndex = el.dataset.pageIndex;
  const svgContainer = el
    .closest(".page-results")
    .querySelector(".svgContainer");
  svgContainer.innerHTML = ""; // Clear existing SVG to redraw
};
</script>

<style scoped>
.page-results {
  th {
    padding: 0;
  }
  td {
    padding: 0;
    ol {
      list-style: none;
      padding: 0;
    }
    li {
    }
    li::marker {
      color: green;
      padding-left: 1em;
    }
  }
}
</style>
