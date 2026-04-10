<template>
  <div class="main">
    <h1 class="main__title">PDF转图片工具箱</h1>
    <div class="upload__container">
      <el-button type="primary" :icon="UploadFilled" @click="uploadRef.click()">点击上传 PDF 文件</el-button>
      <input ref="uploadRef" type="file" @change="onFileChange" accept="application/pdf" style="display: none" />
    </div>
    <div class="pdf__container" v-if="pdfPagesNum">
      <h2 class="pdf__container-title">
        <span>当前 PDF 文件共有：</span>
        <span style="color: red">{{ pdfPagesNum }}&nbsp;</span>
        <span>页</span>
      </h2>
      <el-card :header="fileName">
        <div class="preview__container">
          <div class="img__card" v-for="i in imagesList" :key="i.index" @click="handleImageDetail(i)">
            <img class="img__card-item" :src="i.src" :alt="i.alt" />
            <p class="img__card-desc">{{ i.alt }}</p>
          </div>
        </div>
      </el-card>
      <div class="download__container">
        <el-button type="primary" :icon="Files" @click="downloadAllImages">打包下载所有图片</el-button>
      </div>
    </div>
    <el-dialog v-model="showDetail" :title="`${imageDetail.alt} 详情`" top="8vh" body-class="detail__dialog">
      <el-text type="info">再次点击图片进入全屏预览</el-text>
      <el-image class="detail__img" :src="imageDetail.src" :alt="imageDetail.alt" :preview-src-list="imagesPreviewList" :initial-index="imageDetail.index"></el-image>
      <div class="detail__download">
        <el-button type="primary" :icon="Download" @click="saveAs(imageDetail.src, imageDetail.alt)">下载单张图片</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script lang="ts" setup>
import { ElButton, ElCard, ElDialog, ElImage, ElText, ElLoading } from 'element-plus';
import { UploadFilled, Download, Files } from '@element-plus/icons-vue';

import type { Image } from '@/types';

import { ref, computed } from 'vue';
import JSZip from 'jszip';
import { saveAs } from 'file-saver';

import * as pdfjsLib from 'pdfjs-dist';
import pdfWorker from 'pdfjs-dist/build/pdf.worker?worker';

// 配置 pdf worker
pdfjsLib.GlobalWorkerOptions.workerPort = new pdfWorker();

let uploadRef = ref();

let pdfPagesNum = ref(0);
let imagesList = ref<Image[]>([]);
let fileName = ref('');

async function onFileChange($event: Event) {
  const target = $event.target as HTMLInputElement;
  const file = target.files?.[0];

  if (!file) {
    return;
  }

  imagesList.value = [];

  fileName.value = file.name;

  // 先将选择的PDF文件转成 arrayBuffer
  const arrayBuffer = await file.arrayBuffer();
  // 调用 pdfjs-dist 插件读取数据
  const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise;
  // 获取PDF文件页码
  pdfPagesNum.value = pdf.numPages;

  // 遍历PDF文件，对每一页文件进行处理
  for (let i = 1; i <= pdfPagesNum.value; i++) {
    const page = await pdf.getPage(i);
    // 控制缩放比例
    const viewport = page.getViewport({ scale: 1.0 });
    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');

    if (!context) {
      throw new Error('无法获取 canvas 上下文');
    }

    canvas.width = viewport.width;
    canvas.height = viewport.height;

    await page.render({ canvasContext: context, viewport, canvas }).promise;
    const url = canvas.toDataURL('image/png');
    imagesList.value.push({ index: i - 1, src: url, alt: `图片 ${i}`, width: `${canvas.width}`, height: `${canvas.height}` });
  }
}

// 查看图片详情
let showDetail = ref(false);
let imageDetail = ref<Image>({ index: 1, src: '', alt: '', height: '', width: '' });

function handleImageDetail(image: Image) {
  showDetail.value = true;
  imageDetail.value = image;
}

let imagesPreviewList = computed(() => {
  let result = imagesList.value.map((item) => {
    return item.src;
  });
  return result;
});

// 下载全部图片
async function downloadAllImages() {
  const urlList = imagesPreviewList.value;
  const zip = new JSZip();

  // 如果图片列表超过100个则显示加载动画
  let loading = undefined;
  if (imagesPreviewList.value.length > 100) {
    loading = ElLoading.service({ text: '处理中...', background: '#7a7a7acc' });
  }

  for (let i = 0; i < urlList.length; i++) {
    const url = urlList[i];
    const response = await fetch(url);
    const blob = await response.blob();
    zip.file(`图片-${i + 1}.png`, blob);
  }
  const content = await zip.generateAsync({ type: 'blob' });
  saveAs(content, '图片合集.zip');

  if (loading) {
    loading.close();
  }
}
</script>

<style lang="scss" scoped>
.main {
  .main__title {
    margin: 16px 0;
    text-align: center;
  }

  .upload__container {
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .pdf__container {
    width: 90%;
    margin: 0 auto;
    padding-bottom: 32px;

    .pdf__container-title {
      margin: 16px 0;
      text-align: center;
    }

    .preview__container {
      display: grid;
      grid-template-columns: 1fr 1fr 1fr;

      .img__card {
        padding: 24px;
        box-sizing: border-box;
        box-shadow:
          1px 0 0 0 #e8e8e8,
          0 1px 0 0 #e8e8e8,
          1px 1px 0 0 #e8e8e8,
          1px 0 0 0 #e8e8e8 inset,
          0 1px 0 0 #e8e8e8 inset;
        transition: all 0.3s;
        cursor: pointer;
        display: flex;
        flex-direction: column;
        align-items: center;

        &:hover {
          box-shadow: 0 2px 8px #00000026;
        }

        .img__card-item {
          width: 100%;
          height: 100%;
          object-fit: contain;
        }

        .img__card-desc {
          margin: 8px;
          text-align: center;
          color: #909399;
        }
      }
    }

    .download__container {
      width: 100%;
      padding: 16px 0;
      display: flex;
      justify-content: center;
    }
  }

  :deep(.detail__dialog) {
    .detail__img {
      width: 100%;

      img {
        height: 100%;
        max-height: 400px;
        margin: 10px 0;
        object-fit: contain;
      }
    }

    .detail__download {
      width: 100%;
      display: flex;
      justify-content: center;
    }
  }
}
</style>
