<template>
  <div class="scanner-container">
    <button @click="scanDocument" :disabled="isScanning" class="scan-button">
      {{ isScanning ? "Scanning..." : "Start Scan" }}
    </button>
    <div v-if="imageUrl" class="image-preview">
      <img :src="imageUrl" alt="Scanned Document" />
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";

const scannerBaseUrl = "http://192.168.0.5:443"; // スキャナーのIPに変更
const imageUrl = ref(null);
const isScanning = ref(false);

const scanDocument = async () => {
  isScanning.value = true;
  imageUrl.value = null; // 画像をリセット

  try {
    // スキャンジョブの作成
    const scanJobResponse = await fetch(`${scannerBaseUrl}/eSCL/ScanJobs`, {
      method: "POST",
      headers: { "Content-Type": "application/xml" },
      body: `
        <scan:ScanSettings xmlns:scan="http://schemas.hp.com/imaging/escl/2011/05/03">
          <scan:Intent>Photo</scan:Intent>
          <scan:Resolution>
            <scan:XResolution>300</scan:XResolution>
            <scan:YResolution>300</scan:YResolution>
          </scan:Resolution>
          <scan:ColorMode>RGB24</scan:ColorMode>
          <scan:MediaSize>
            <scan:Width>2550</scan:Width>
            <scan:Height>3507</scan:Height>
          </scan:MediaSize>
          <scan:DocumentFormatExt>image/jpeg</scan:DocumentFormatExt>
        </scan:ScanSettings>
      `,
    });

    if (!scanJobResponse.ok) throw new Error("Failed to create scan job");

    // 画像の取得URL
    const scanJobLocation = scanJobResponse.headers.get("Location");
    if (!scanJobLocation) throw new Error("Failed to get scan job location");

    const imageUrlFull = new URL(scanJobLocation, scannerBaseUrl).href;

    // 画像データを取得
    const imageResponse = await fetch(imageUrlFull+"/NextDocument");
    if (!imageResponse.ok) throw new Error("Failed to fetch scanned image");

    const imageBlob = await imageResponse.blob();
    console.log(imageBlob.type);

    imageUrl.value = URL.createObjectURL(imageBlob);
  } catch (error) {
    console.error(error);
    alert("Scanning failed. Please check your scanner.");
  } finally {
    isScanning.value = false;
  }
};
</script>

<style scoped>
.scanner-container {
  text-align: center;
  padding: 20px;
}

.scan-button {
  background-color: #007bff;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
  transition: background 0.3s;
}

.scan-button:disabled {
  background-color: gray;
  cursor: not-allowed;
}

.image-preview {
  margin-top: 20px;
}

.image-preview img {
  max-width: 100%;
  height: auto;
  border: 2px solid #ccc;
  border-radius: 5px;
}
</style>
