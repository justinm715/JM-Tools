<template>
  <div class="container mx-auto py-4">
    <div
      class="border-dashed border-4 rounded py-2 text-center"
      :class="{ 'bg-blue-100': isDragOver, 'border-gray-200': !isDragOver }"
      @drop.prevent="handleFileDrop"
      @dragover.prevent="isDragOver = true"
      @dragenter.prevent="isDragOver = true"
      @dragleave.prevent="isDragOver = false"
    >
      <p>Drag and drop your PDF file here, or click to select a file</p>
      <!-- Hidden File Input -->
      <input
        type="file"
        accept=".pdf"
        @change="handleFileSelect"
        hidden
        ref="fileInput"
      />

      <!-- File Select Button -->
      <button
        class="mt-4 bg-blue-500 hover:bg-blue-600 text-white py-2 px-4 text-sm rounded inline-flex items-center"
        @click="triggerFileInput"
      >
        <span>Select File</span>
      </button>
      <button
        class="mt-4 bg-blue-500 text-white py-2 px-4 ml-2 text-sm rounded"
        :class="{ 'opacity-50 cursor-not-allowed': isUploading }"
        @click="uploadFile"
        :disabled="isUploading"
      >
        Upload PDF
      </button>

      <div class="mt-4">
        <span class="mr-2" v-if="fileName">{{ fileName }}</span>
        <progress :value="uploadProgress" max="100"></progress>
        <p class="text-green-500" v-if="successMessage">{{ successMessage }}</p>
      </div>
    </div>

    <!-- Render tables from response data -->
    <div v-if="responseData.length > 0">
      <div
        v-for="(pageData, pageIndex) in responseData"
        :key="pageIndex"
        class="mb-8"
      >
        <div class="mb-2">
          <span class="text-lg font-semibold"
            >Page {{ pageData[0]["Page"] }}</span
          ><br />
          <span
            class="cursor-pointer text-xs rounded text-gray-400 underline"
            @click="copyTableData(pageIndex)"
            ><Icon name="external" />
            Copy Data
          </span>
        </div>
        <table class="min-w-full divide-y divide-gray-200 text-sm">
          <thead>
            <tr class="bg-gray-100">
              <th class="px-2 py-1 text-left">Region</th>
              <th class="px-2 py-1 text-left">Element Type</th>
              <th class="px-2 py-1 text-left">Element Label</th>
              <th class="px-2 py-1 text-right">Height (ft)</th>
              <th class="px-2 py-1 text-right">Weight (psf)</th>
              <th class="px-2 py-1 text-right">Snow Weight (psf)</th>
              <th class="px-2 py-1 text-right">Length (ft)</th>
              <th class="px-2 py-1 text-right">Area (sf)</th>
              <th class="px-2 py-1 text-right">Total Weight (lbs)</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in pageData" :key="item">
              <td class="px-2 py-1">{{ item["Region"] }}</td>
              <td class="px-2 py-1">{{ item["Element Type"] }}</td>
              <td class="px-2 py-1">{{ item["Element Label"] }}</td>
              <td class="px-2 py-1 text-right">
                {{ formatNumber(item["Height (ft)"]) }}
              </td>
              <td class="px-2 py-1 text-right">
                {{ formatNumber(item["Weight (psf)"]) }}
              </td>
              <td class="px-2 py-1 text-right">
                {{ formatNumber(item["Snow Weight (psf)"]) }}
              </td>
              <td class="px-2 py-1 text-right">
                {{ formatNumber(item["Length (ft)"]) }}
              </td>
              <td class="px-2 py-1 text-right">
                {{ formatNumber(item["Area (sf)"]) }}
              </td>
              <td class="px-2 py-1 text-right">
                {{ formatNumber(item["Total Weight (lbs)"]) }}
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";

const file = ref(null);
const uploadProgress = ref(0);
const responseData = ref([]);
const fileName = ref("");
const isDragOver = ref(false);
const fileInput = ref(null);
const isUploading = ref(false);
const successMessage = ref("");

const handleFileSelect = (event) => {
  if (event.target.files.length > 0) {
    file.value = event.target.files[0];
    fileName.value = file.value.name;
    successMessage.value = ""; // Clear the success message
  }
};

const formatNumber = (num) => {
  if (typeof num === "number") {
    return num.toFixed(2);
  }
  return num;
};

const triggerFileInput = () => {
  fileInput.value.click();
};

const handleFileDrop = (event) => {
  isDragOver.value = false;
  if (event.dataTransfer.files.length > 0) {
    file.value = event.dataTransfer.files[0];
    fileName.value = file.value.name;
    // Manually trigger the file selection handler
    handleFileSelect({ target: { files: event.dataTransfer.files } });
  }
  successMessage.value = ""; // Clear the success message
};

const uploadFile = () => {
  if (!file.value) {
    alert("Please select a file to upload.");
    return;
  }

  isUploading.value = true; // Disable the upload button
  uploadProgress.value = 0;
  const formData = new FormData();
  formData.append("file", file.value);

  const xhr = new XMLHttpRequest();
  xhr.open(
    "POST",
    "https://jm-tools-python-serverless-api.vercel.app/api/region_tribs/process_pdf",
    true,
  );

  xhr.onload = () => {
    isUploading.value = false; // Re-enable the upload button
    uploadProgress.value = 0; // Reset upload progress
    if (xhr.status === 200) {
      responseData.value = JSON.parse(xhr.responseText);
      successMessage.value = "File uploaded successfully!";
    } else {
      alert("An error occurred while uploading the file.");
      successMessage.value = ""; // Clear the message in case of an error
    }
  };

  xhr.upload.onprogress = (e) => {
    if (e.lengthComputable) {
      uploadProgress.value = (e.loaded / e.total) * 100;
    }
  };

  xhr.onloadend = () => {
    isUploading.value = false; // Re-enable the upload button
    if (xhr.status === 200) {
      responseData.value = JSON.parse(xhr.responseText);
    } else {
      alert("An error occurred while uploading the file.");
    }
  };

  xhr.send(formData);
};

const copyTableData = (pageIndex) => {
  const pageData = responseData.value[pageIndex];

  // Define the desired order of the columns
  const columnOrder = [
    "Region",
    "Element Type",
    "Element Label",
    "Height (ft)",
    "Weight (psf)",
    "Snow Weight (psf)",
    "Length (ft)",
    "Area (sf)",
    "Total Weight (lbs)",
  ];

  // Check if pageData is not empty
  if (pageData && pageData.length > 0) {
    // Create the headers row from the columnOrder array
    const headers = columnOrder.join("\t");

    // Map the rows according to the column order
    const rows = pageData
      .map((row) =>
        columnOrder
          .map((column) => {
            const cell = row[column];
            // If the cell contains special characters, wrap it in quotes
            return typeof cell === "string" && /[\r\n\t,]/.test(cell)
              ? `"${cell.replace(/"/g, '""')}"`
              : cell;
          })
          .join("\t"),
      )
      .join("\n");

    // Combine headers and rows
    const tabDelimitedText = `${headers}\n${rows}`;

    // Copy to clipboard
    navigator.clipboard.writeText(tabDelimitedText).then(
      () => {
        alert("Table data copied to clipboard!");
      },
      (err) => {
        alert(`Failed to copy table data: ${err}`);
      },
    );
  }
};
</script>

<style scoped>
/* Add Tailwind CSS styles here */
</style>
