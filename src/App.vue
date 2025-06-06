<script setup>
import { ref, onMounted, onUnmounted, watch, computed } from 'vue';
import presetsJson from './assets/presets.json';
// components
import Header from './components/Header.vue';
import Footer from './components/Footer.vue';
// icon
import Logo from './components/icons/Logo.vue';
import IconList from './components/icons/IconList.vue';
import IconPen from './components/icons/IconPen.vue';
import IconGroup from './components/icons/IconGroup.vue';

// 排除的特殊顏色列表，這些顏色不會被用作獎品顏色，避免與轉盤的其他元素顏色衝突
// 白色(#ffffff)、黑色(輪盤邊框 #000000)、紅色(指針色 #d93b3b)、深灰色(輪盤背景色 #333)、亮黃色(中獎色 #ffeb3b)
const excludedColors = ['#ffffff', '#000000', '#d93b3b', '#333', '#ffeb3b'];

// 預設獎品的背景色列表，如果獎品數量超過預設顏色數量，則會使用隨機顏色
const defaultColors = [
  '#f44336', // 紅
  '#ff9800', // 橘
  '#ffd600', // 黃
  '#4caf50', // 綠
  '#00bcd4', // 青
  '#2196f3', // 藍
  '#9c27b0', // 紫
  '#e91e63', // 粉
  '#8bc34a', // 青綠
  '#ffc107', // 金黃
  '#009688', // 藍綠
  '#3f51b5', // 靛藍
  '#ff5722', // 橘紅
  '#cddc39', // 檸檬黃
  '#607d8b', // 藍灰
  '#795548', // 咖啡
  '#e57373', // 淺紅
  '#ffb74d', // 淺橘
  '#fff176', // 淺黃
  '#81c784', // 淺綠
  '#4dd0e1', // 淺青
  '#64b5f6', // 淺藍
  '#ba68c8', // 淺紫
  '#f06292', // 淺粉
  '#aed581', // 淺青綠
  '#ffd54f', // 淺金黃
  '#4db6ac', // 淺藍綠
  '#7986cb', // 淺靛藍
  '#ff8a65', // 淺橘紅
  '#afb42b', // 深檸檬黃
  '#90a4ae', // 淺藍灰
  '#bcaaa4', // 淺咖啡
  '#d32f2f', // 深紅
  '#f57c00', // 深橘
  '#fbc02d', // 深黃
  '#388e3c', // 深綠
  '#00838f', // 深青
  '#1976d2', // 深藍
  '#6a1b9a', // 深紫
  '#ad1457', // 深粉
  '#689f38', // 深青綠
  '#ffa000', // 深金黃
  '#00695c', // 深藍綠
  '#283593', // 深靛藍
  '#bf360c', // 深橘紅
  '#37474f', // 深藍灰
  '#4e342e', // 深咖啡
  '#aeea00', // 螢光黃綠
  '#00e676', // 螢光綠
  '#00b8d4', // 螢光藍
  '#d500f9', // 螢光紫
];

// 旋轉方向的布林值，true 代表順時針旋轉，false 代表逆時針旋轉。
const rotateDirection = true;

// 音效引用
const spinSound = ref(null);
const endSound = ref(null);

/**
 * 獎品列表
 *
 * @description
 * 如果 localStorage 中有獎品資料，則從 localStorage 讀取，否則使用預設顏色生成獎品列表。
 */
const prizes = ref(
  localStorage.getItem('prizes') && localStorage.getItem('prizes') !== '[]'
    ? JSON.parse(localStorage.getItem('prizes'))
    : defaultColors.slice(0, 8).map((color, index) => ({
        name: `獎項名稱 ${index + 1}`,
        color,
      }))
);

let textareaTimer = null; // 用於節流的定時器
let waitTime = 300; // 節流等待時間，單位為毫秒
// 用於存儲 textarea 的內容，使用 computed 來實現雙向綁定
const textareaContent = computed({
  get() {
    return prizes.value.map((prize) => prize.name).join('\n'); // 將獎項名稱轉換為以換行符分隔的字符串
  },
  set(val) {
    if (textareaTimer) {
      clearTimeout(textareaTimer);
    }
    textareaTimer = setTimeout(() => {
      textareaTimer = null;
      updatePrizesFromTextarea(val); // 當 textarea 的內容改變時，更新獎項列表
    }, waitTime);
  },
});
const presets = ref({});

const canvasRef = ref(null); // 轉盤畫布引用
const spinning = ref(false); // 是否正在旋轉轉盤
const rotation = ref(0); // 轉盤旋轉角度，初始值為 0
const prizeIndex = ref(null); // 中獎獎項索引
const selectedPrize = ref(null); // 中獎獎項對象
const newPrizeName = ref(''); // 新增獎項名稱輸入框
const editingIndex = ref(null); // 正在編輯的獎項索引
const editingName = ref(''); // 編輯獎項名稱輸入框
const isTextareaMode = ref(true); // 控制是否顯示 textarea

// 轉盤狀態變數
let lastRotation = 0; // 追蹤最後旋轉角度
let accumulatedRotation = 0; // 追蹤累積旋轉角度
let lastSegmentCount = 0; // 追蹤累積經過的獎項數
let animationFrameId = null; // 用於 requestAnimationFrame 的 ID

/**
 * 隨機產生一個不重複於現有獎品顏色與排除色的十六進位色碼。
 * @returns {string} 隨機色碼（如 "#a1b2c3"）
 */
const getRandomColor = () => {
  let color;
  do {
    color = `#${Math.floor(Math.random() * 0x1000000)
      .toString(16)
      .padStart(6, '0')}`;
  } while (
    prizes.value.some(
      (prize) => prize.color.toLowerCase() === color.toLowerCase()
    ) ||
    excludedColors.includes(color.toLowerCase())
  );
  return color;
};

/**
 * 將文字裁切至最大寬度，超出則加「...」。
 * @param {CanvasRenderingContext2D} c - Canvas 2D context
 * @param {string} text - 文字內容
 * @param {number} maxWidth - 最大寬度（像素）
 * @returns {string} 處理後的文字
 */
const truncateText = (c, text, maxWidth) => {
  const width = c.measureText(text).width;
  if (width <= maxWidth) {
    return text;
  }

  const ellipsis = '...';
  const len = text.length;
  for (let i = len; i > 0; i--) {
    const str = text.substring(0, i) + ellipsis;
    if (c.measureText(str).width <= maxWidth) {
      return str;
    }
  }
};

/**
 * 在 canvas 上繪製轉盤與獎項。
 * 1. 設定高解析度畫布。
 * 2. 畫轉盤底色與分割扇形。
 * 3. 中獎區塊高亮。
 * 4. 扇形上顯示獎項名稱，過長自動截斷。
 * 5. 中心繪製紅色圓點。
 *
 * 依賴：canvasRef、prizes、prizeIndex。
 */
const drawWheel = () => {
  const canvas = canvasRef.value;
  const ctx = canvas.getContext('2d');

  /**
   * 設定畫布顯示尺寸與縮放：
   * 1. displayWidth, displayHeight：畫布在頁面上的實際顯示大小（單位：像素）。
   * 2. canvas.width, canvas.height：設為顯示尺寸乘上 devicePixelRatio，確保高解析度螢幕下畫面清晰。
   * 3. ctx.scale(dpr, dpr)：將繪圖上下文縮放，讓繪圖內容與顯示尺寸一致。
   */
  const displayWidth = 500;
  const displayHeight = 500;
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.floor(displayWidth * dpr);
  canvas.height = Math.floor(displayHeight * dpr);
  ctx.scale(dpr, dpr);

  // 計算中心點和半徑（使用顯示尺寸）
  const centerX = displayWidth / 2;
  const centerY = displayHeight / 2;
  const radius = displayWidth / 2;

  // 清除畫布（使用顯示尺寸)
  ctx.clearRect(0, 0, displayWidth, displayHeight);

  // 繪製圓形背景做為轉盤底色
  ctx.beginPath();
  ctx.arc(centerX, centerY, radius, 0, Math.PI * 2);
  ctx.closePath();
  ctx.fillStyle = '#333';
  ctx.fill();

  // 計算每個獎項的扇形角度
  const segmentAngle = 360 / prizes.value.length;
  // 繪製每個獎項
  prizes.value.forEach((prize, index) => {
    // 調整起始角度到上方
    let startAngle = index * segmentAngle - 90;
    let endAngle = (index + 1) * segmentAngle - 90;
    // 將角度轉換為弧度
    startAngle = (startAngle * Math.PI) / 180;
    endAngle = (endAngle * Math.PI) / 180;

    // 如果是中獎的獎項，繪製一個亮色的扇形背景
    if (prizeIndex.value === index) {
      // 繪製扇形
      ctx.beginPath();
      ctx.moveTo(centerX, centerY);
      ctx.arc(centerX, centerY, radius, startAngle, endAngle);
      ctx.closePath();
      ctx.fillStyle = '#ffeb3b';
      ctx.fill();
    }

    // 繪製獎項本身扇形
    let w = 8; // 扇形距離邊框的寬度
    ctx.beginPath();
    ctx.moveTo(centerX, centerY);
    ctx.arc(centerX, centerY, radius - w, startAngle, endAngle);
    ctx.closePath();
    ctx.fillStyle = prize.color;
    ctx.fill();

    // 繪製文字
    ctx.save();
    ctx.translate(centerX, centerY);
    ctx.rotate((startAngle + endAngle) / 2);
    ctx.textAlign = 'right';
    ctx.fillStyle = '#000';
    ctx.font = 'bold 32px Arial';
    ctx.fillText(truncateText(ctx, prize.name, 200), radius - w - 6, 12);
    ctx.restore();
  });

  // 繪製轉盤中心圓點
  ctx.beginPath();
  ctx.arc(centerX, centerY, 10, 0, Math.PI * 2);
  ctx.closePath();
  ctx.fillStyle = '#d93b3b'; // 中心圓點顏色
  ctx.strokeStyle = '#000'; // 中心圓點邊框顏色
  ctx.lineWidth = 2; // 中心圓點邊框寬度
  ctx.stroke();
  ctx.fill();
};

/**
 * 重置獎品狀態。
 * 將中獎索引（prizeIndex）和中獎獎項（selectedPrize）設為 null，
 * 以清除當前的中獎資訊。
 */
const resetPrizeState = () => {
  prizeIndex.value = null; // 重置中獎索引
  selectedPrize.value = null; // 重置中獎獎項
};

/**
 * 重置轉盤相關的狀態變數。
 * 將旋轉角度、最後旋轉角度、累積旋轉角度及經過的獎項數全部歸零，
 * 以便重新開始轉盤操作。
 */
const resetRotationState = () => {
  rotation.value = 0; // 重置轉盤旋轉角度
  lastRotation = 0; // 重置最後旋轉角度
  accumulatedRotation = 0; // 重置累積旋轉角度
  lastSegmentCount = 0; // 重置累積經過的獎項數
};

/**
 * 觸發轉盤旋轉的函式。
 *
 * 此函式會根據獎品數量隨機決定旋轉圈數，計算最終中獎的獎品索引，並在旋轉結束後顯示中獎結果。
 *
 * 流程說明：
 * 1. 若當前正在旋轉則直接返回。
 * 2. 重置中獎狀態並重新繪製輪盤。
 * 3. 計算最小旋轉圈數（根據獎品數量動態調整）。
 * 4. 隨機產生旋轉角度，並根據旋轉方向（順時針或逆時針）計算最終角度。
 * 5. 根據最終角度計算中獎獎品的索引。
 * 6. 設定旋轉動畫結束後，播放音效、顯示中獎效果並更新狀態。
 *
 * 注意：旋轉動畫持續時間為 8 秒，期間會鎖定旋轉狀態避免重複觸發。
 */
const spinWheel = () => {
  if (spinning.value) return;

  // 重置中獎狀態
  resetPrizeState();
  // 如果有獎項正在編輯，則重置編輯狀態
  resetEdit();
  // 重新繪製輪盤，恢復正常樣式
  drawWheel();

  spinning.value = true;

  // 隨機旋轉角度，根據獎品數量決定最小旋轉圈數
  // 獎品數量大於 5 時，最小旋轉圈數為 5；大於 3 時，最小旋轉圈數為 7；否則為 8
  const minSpins =
    prizes.value.length > 5 ? 5 : prizes.value.length > 3 ? 7 : 8;
  const randomSpin = Math.floor(Math.random() * 360) + 360 * minSpins;
  // 每個獎品的角度範圍
  const segmentAngle = 360 / prizes.value.length;

  let normalizedRotation = 0;
  let adjustedRotation = 0;

  if (rotateDirection) {
    // 順時針設置旋轉角度
    rotation.value += randomSpin;
    // 將旋轉角度歸一化到 0-360 度範圍內
    normalizedRotation = rotation.value % 360;
    // 由於旋轉及獎項繪製是順時針方向，所以需要將 normalizedRotation 轉換為逆時針方向的角度來計算中獎獎品
    adjustedRotation = 360 - normalizedRotation;
  } else {
    // 逆時針設置旋轉角度
    rotation.value -= randomSpin;
    // 將旋轉角度歸一化到 0-360 度範圍內
    adjustedRotation = Math.abs(rotation.value % 360);
  }

  // 計算中獎的獎品索引
  prizeIndex.value = Math.floor(adjustedRotation / segmentAngle);
  // console.log('Prize Index:', prizeIndex.value);

  // 停止旋轉後顯示結果
  setTimeout(() => {
    // 播放中獎音效
    endSound.value.play();

    // 重新繪製輪盤，顯示中獎效果
    drawWheel();

    // 顯示中獎提示
    selectedPrize.value = prizes.value[prizeIndex.value];
    spinning.value = false;
  }, 8000);
};

// 監聽獎品列表的變更
watch(
  prizes,
  (newPrizes) => {
    // console.log('Watch Prizes updated:', newPrizes);
    // 將新的獎品列表存入 localStorage
    localStorage.setItem('prizes', JSON.stringify(newPrizes));
    // 當獎品列表變更時
    resetPrizeState(); // 重置中獎狀態
    resetRotationState(); // 重置轉盤狀態
    drawWheel(); // 重新繪製轉盤
  },
  { deep: true }
);

/**
 * 新增獎項的函數。
 * - 將用戶輸入的新獎項加入獎品列表。
 * - 優先使用未被佔用的預設顏色，否則隨機產生顏色。
 */
const addPrize = () => {
  if (newPrizeName.value) {
    // 找到第一個未使用的預設顏色
    const usedColors = prizes.value.map((prize) => prize.color.toLowerCase());
    const availableColor = defaultColors.find(
      (color) => !usedColors.includes(color.toLowerCase())
    );

    // 如果有未使用的預設顏色，使用該顏色；否則使用隨機顏色
    const color = availableColor || getRandomColor();

    prizes.value.push({ name: newPrizeName.value, color });
    newPrizeName.value = '';
  }
};

/**
 * 根據索引從獎品列表中移除指定獎項。
 */
const deletePrize = (index) => {
  prizes.value.splice(index, 1);
};

/**
 * 開始編輯指定索引的獎項名稱。
 *
 * @param {number} index - 要編輯的獎項索引。
 * @param {string} name - 當前獎項名稱。
 * @description
 * 如果轉盤正在旋轉、有其他或同一個獎項正在編輯，
 * 則不允許開始新的編輯。
 */
const startEdit = (index, name) => {
  if (
    spinning.value ||
    editingIndex.value !== null ||
    editingIndex.value === index
  ) {
    return;
  }
  editingIndex.value = index;
  editingName.value = name;
};

/**
 * 儲存當前正在編輯的獎品名稱。
 *
 * @description
 * 如果 `editingName.value` 有值，則將該名稱儲存到對應的獎品項目中，並重置編輯狀態。
 */
const saveEdit = () => {
  if (editingName.value) {
    prizes.value[editingIndex.value].name = editingName.value;
    resetEdit();
  }
};

/**
 * 重置編輯狀態的函式。
 *
 * @description
 * - 如果目前沒有正在編輯的項目則直接返回。
 * - 將 `editingIndex.value` 設為 `null`，並清空 `editingName.value`，以結束編輯。
 */
const resetEdit = () => {
  if (editingIndex.value === null) return;
  editingIndex.value = null;
  editingName.value = '';
};

// 切換顯示模式
const toggleEditMode = () => {
  resetEdit(); // 切換模式前先重置編輯狀態
  isTextareaMode.value = !isTextareaMode.value;
};

// 根據 textarea 更新獎項列表。
const updatePrizesFromTextarea = (textValue) => {
  // console.log('updatePrizesFromTextarea:', textValue);
  if (!textValue) {
    // 如果 textarea 為空，則清空獎品列表
    prizes.value = [];
  } else {
    // 將 textarea 的內容按行分割成獎品名稱
    const names = textValue.split('\n');
    // console.log('Parsed names:', names);

    let usedColors = prizes.value.map((prize) => prize.color.toLowerCase());
    // 根據輸入的名稱更新獎品列表
    prizes.value = names.map((newPrizeName, index) => {
      // 如果名稱為空，則使用預設名稱，並且限制名稱長度為 12
      let name = !newPrizeName.trim()
        ? '--預設獎項--'
        : newPrizeName.trim().slice(0, 12);
      let color;
      if (prizes.value[index]) {
        // 如果已有顏色，則使用原有顏色
        color = prizes.value[index].color;
      } else {
        // 如果沒有顏色，找到第一個未使用的預設顏色
        const availableColor = defaultColors.find(
          (color) => !usedColors.includes(color.toLowerCase())
        );
        // 如果沒有可用的預設顏色，則隨機產生一個顏色
        color = availableColor || getRandomColor();
        // 將新顏色加入已使用顏色列表
        usedColors.push(color);
      }
      return { name, color };
    });
  }
};

/**
 * 關閉中獎提示的函數。
 *
 * @description
 * 重置中獎狀態，隱藏中獎提示視窗。
 */
const closePrizePopup = () => {
  resetPrizeState();
};

let lastPlay = 0; // 用於節流的變數
/**
 * 控制播放旋轉音效的函數。
 *
 * @description
 * 控制播放旋轉音效，並加入 100 毫秒的節流機制，避免音效過於頻繁地觸發。
 */
const playSpinSound = () => {
  // 添加 100ms 節流
  if (!lastPlay || Date.now() - lastPlay >= 100) {
    spinSound.value.currentTime = 0; // 重置音效播放時間
    spinSound.value.play();
    lastPlay = Date.now();
  }
};

/**
 * 檢查轉盤旋轉狀態的函數。
 *
 * @description
 * 用於持續監聽轉盤的旋轉角度變化，並根據當前旋轉角度判斷是否進入新的獎品段。
 * 當進入新的獎品段時，會觸發旋轉音效播放。
 *
 * 主要流程：
 * 1. 若轉盤未在旋轉，則取消動畫幀並結束函數。
 * 2. 取得轉盤當前的旋轉矩陣，計算出目前的旋轉角度（0~359度）。
 * 3. 計算本次與上次的角度差值，並處理角度跳變（如359到0）。
 * 4. 累加實際旋轉角度，判斷是否跨越新的獎品段。
 * 5. 若跨越新的獎品段，則播放旋轉音效。
 * 6. 持續以 requestAnimationFrame 方式遞迴檢查旋轉狀態。
 */
const checkRotation = () => {
  if (!spinning.value) {
    cancelAnimationFrame(animationFrameId);
    return;
  }

  // 獲取當前轉盤的旋轉矩陣
  const matrix = new DOMMatrix(getComputedStyle(canvasRef.value).transform);
  let currentRotation = Math.round(
    Math.atan2(matrix.b, matrix.a) * (180 / Math.PI)
  );
  currentRotation =
    currentRotation < 0 ? currentRotation + 360 : currentRotation;

  if (currentRotation !== lastRotation) {
    // 計算每個獎品段的角度
    const segmentAngle = 360 / prizes.value.length;

    // 計算角度差值
    let rotationDiff = currentRotation - lastRotation;
    // 處理角度跳變（從359到0或從0到359）
    if (Math.abs(rotationDiff) > 180) {
      if (rotationDiff > 0) {
        rotationDiff -= 360;
      } else {
        rotationDiff += 360;
      }
    }

    // 累加實際旋轉角度
    accumulatedRotation += Math.abs(rotationDiff);
    // 檢查是否達到新的獎品段
    const currentSegmentCount = Math.floor(accumulatedRotation / segmentAngle);
    if (currentSegmentCount > lastSegmentCount) {
      // 播放旋轉音效
      playSpinSound();
      lastSegmentCount = currentSegmentCount;
    }
    lastRotation = currentRotation;
  }
  animationFrameId = requestAnimationFrame(checkRotation);
};

/**
 * 過渡動畫開始時的處理函式。
 * 調用 checkRotation() 方法以檢查元素的旋轉狀態。
 */
const transitionStartHandler = () => {
  checkRotation();
};
/**
 * 過渡動畫結束時的處理函式。
 * 通過 cancelAnimationFrame(animationFrameId) 取消動畫幀請求，以防止不必要的動畫執行。
 */
const transitionEndHandler = () => {
  cancelAnimationFrame(animationFrameId);
};

// 載入預設獎品的函式
const loadPreset = (type) => {
  const preset = presets.value[type];
  if (preset) {
    textareaContent.value = preset.items.join('\n');
    toggleMenu();
  }
};

// 處理點擊外部關閉菜單的函式
const handleClickOutside = (e) => {
  if (presetMenu.value && !presetMenu.value.contains(e.target)) {
    isOpen.value = false;
  }
};

const isOpen = ref(false);
const presetMenu = ref(null);

/**
 * 切換菜單的函式
 *
 * Toggle the preset menu.
 */
// 切換菜單的函式
function toggleMenu() {
  // Toggle the menu's open state
  isOpen.value = !isOpen.value;
}

onMounted(() => {
  presets.value = presetsJson;
  drawWheel();
  document.addEventListener('click', handleClickOutside);
  // 監聽 transition
  const canvas = canvasRef.value;
  if (canvas) {
    canvas.addEventListener('transitionstart', transitionStartHandler);
    canvas.addEventListener('transitionend', transitionEndHandler);
  }
});

onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside);
  const canvas = canvasRef.value;
  if (canvas) {
    transitionStartHandler &&
      canvas.removeEventListener('transitionstart', transitionStartHandler);
    transitionEndHandler &&
      canvas.removeEventListener('transitionend', transitionEndHandler);
  }
});

// 自定義指令：自動聚焦
const vFocus = {
  mounted: (el) => el.focus(),
};
</script>

<template>
  <div class="app-container">
    <Header />
    <audio ref="spinSound" src="/sounds/spin-sound.mp3"></audio>
    <audio ref="endSound" src="/sounds/win-sound.mp3"></audio>

    <main class="wheel-wrapper">
      <div class="wheel-container">
        <canvas
          ref="canvasRef"
          :style="{
            transform: `rotate(${rotation}deg)`,
            transition: spinning
              ? 'transform 8s cubic-bezier(0.4, 0, 0.2, 1)'
              : 'none',
          }"
        ></canvas>
        <div class="pointer"></div>
        <div class="wheel-controls">
          <button
            @click="spinWheel"
            :disabled="spinning || prizes.length === 0"
            class="spin-button"
          >
            抽籤
          </button>
        </div>
      </div>

      <!-- 中獎提示 -->
      <div v-if="selectedPrize" class="prize-mask">
        <div class="prize-popup">
          <div class="prize-popup-content">
            <div class="prize-popup-header">
              <Logo />
              <p>恭喜你抽中：</p>
            </div>
            <div class="prize-popup-info">
              <span
                class="prize-popup-color"
                :style="{ backgroundColor: selectedPrize.color }"
              ></span>
              <span class="prize-popup-name">{{ selectedPrize.name }}</span>
            </div>
          </div>
          <button @click="closePrizePopup" class="close-popup-button">
            關閉
          </button>
        </div>
      </div>

      <div class="prizes-container">
        <div class="prizes-section">
          <h2 class="prizes-section-title">獎項管理</h2>
          <div ref="presetMenu" class="preset-menu" :class="{ open: isOpen }">
            <button
              class="preset-toggle"
              @click="toggleMenu"
              :disabled="spinning"
            >
              <span><IconGroup />範例</span>
            </button>
            <div class="preset-buttons">
              <button
                v-for="(preset, key) in presets"
                :key="key"
                @click="loadPreset(key)"
              >
                {{ preset.label }}
              </button>
            </div>
          </div>
        </div>
        <div class="add-prize-form">
          <input
            v-model.trim="newPrizeName"
            placeholder="新增獎項名稱 (最多12字)"
            maxlength="12"
            :disabled="spinning"
            @focus="resetEdit"
          />
          <button @click="addPrize" :disabled="spinning">新增</button>
        </div>
        <div class="prizes-list">
          <div class="prizes-header">
            <h3>獎項列表</h3>
            <button
              @click="toggleEditMode"
              class="toggle-button"
              :disabled="spinning"
            >
              <span v-if="isTextareaMode"><IconList />列表</span>
              <span v-else><IconPen /> 文本 </span>
            </button>
          </div>
          <p>
            {{
              isTextareaMode
                ? '可新增、修改、刪除文本區域中的獎項'
                : '點擊獎項可以編輯獎項名稱'
            }}
          </p>
          <template v-if="isTextareaMode">
            <textarea
              v-model.trim="textareaContent"
              class="prizes-textarea"
              :disabled="spinning"
            ></textarea>
          </template>
          <template v-else>
            <ul>
              <li
                v-for="(prize, index) in prizes"
                :key="prize.name + prize.color"
                class="prize-item"
                @click="startEdit(index, prize.name)"
                :class="{
                  editing: !spinning && editingIndex === index,
                  disabled: spinning || editingIndex !== null,
                }"
              >
                <span
                  class="prize-color"
                  :style="{ backgroundColor: prize.color }"
                ></span>
                <template v-if="!spinning && editingIndex === index">
                  <input
                    ref="editInput"
                    v-model.trim="editingName"
                    class="edit-prize-input"
                    maxlength="12"
                    v-focus
                  />
                  <button class="btn btn-edit" @click.stop="saveEdit">
                    保存
                  </button>
                </template>
                <template v-else>
                  <span class="prize-name">{{ prize.name }}</span>
                  <button
                    class="btn btn-delete"
                    @click.stop="deletePrize(index)"
                    :disabled="spinning || editingIndex !== null"
                  >
                    刪除
                  </button>
                </template>
              </li>
            </ul>
          </template>
        </div>
      </div>
    </main>

    <Footer />
  </div>
</template>

<style lang="scss">
*,
*::before,
*::after {
  box-sizing: border-box;
}

body {
  margin: 0;
  padding: 0;
  font-family: Arial, sans-serif;
  min-height: 100vh;
  background: linear-gradient(135deg, #fff8e1 0%, #ffecb3 50%, #ffe082 100%);
}

.app-container {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.wheel-wrapper {
  flex: 1;
  width: 90%;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 30px;
  margin: 0 auto;
  // 增加內邊距，避免指針被遮擋
  padding: 20px 0;
}

.wheel-container {
  position: relative;
  text-align: center;
  width: 500px;
  max-width: 100%;
}

canvas {
  display: block; /* 移除 inline 空隙 */
  width: 100%;
  border-radius: 50%;
  border: 5px solid #000;
  transform-origin: center;
  transition: transform 8s cubic-bezier(0.4, 0, 0.2, 1);
}

/* 指針邊框粗細變數 */
:root {
  --pointer-border-width: 5px;
}

/* 調整箭頭樣式 */
.pointer {
  position: absolute;
  top: -15px;
  left: 50%;
  transform: translateX(-50%);
  width: 38px;
  height: 40px;
  z-index: 10;
  pointer-events: none;
}

.pointer::before {
  content: '';
  position: absolute;
  left: 0;
  top: 0;
  width: 38px;
  height: 40px;
  /* 外層黑色三角形（邊框） */
  clip-path: polygon(50% 100%, 0% 0%, 100% 0%);
  background: #000;
  z-index: 1;
}

.pointer::after {
  content: '';
  position: absolute;
  left: calc(1.5 * var(--pointer-border-width));
  top: var(--pointer-border-width);
  width: calc(38px - 3 * var(--pointer-border-width));
  height: calc(40px - 3 * var(--pointer-border-width));
  /* 內層紅色三角形（指針本體） */
  clip-path: polygon(50% 100%, 0% 0%, 100% 0%);
  background: #d93b3b;
  z-index: 2;
}

.spin-button {
  margin-top: 20px;
  padding: 12px 24px;
  font-size: 18px;
  font-weight: bold;
  color: #fff;
  background-color: #00701a;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  transition: background-color 0.3s ease, transform 0.2s ease;

  &:hover {
    background-color: #005f17;
  }
}

.spin-button:disabled {
  background-color: #ccc;
  color: #9e9e9e;
  cursor: not-allowed;
  box-shadow: none;
}

/* 中獎提示遮罩層樣式 */
.prize-mask {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.6);
  z-index: 999;
  display: flex;
  justify-content: center;
  align-items: center;
}

/* 中獎提示彈窗樣式 */
.prize-popup {
  background-color: #fff;
  padding: 30px 40px;
  border-radius: 15px;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.4);
  text-align: center;
  animation: popup-fade-in 0.3s ease-out;
  width: 90%;
  max-width: 400px;
  border: 3px solid #ffa726;
}

.prize-popup-content {
  margin-bottom: 25px;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.prize-popup-content .prize-popup-header {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 20px;

  p {
    font-size: 24px;
    line-height: 1;
    font-weight: bold;
    color: #333;
    margin: 0;
  }
}

.prize-popup-content .prize-popup-info {
  display: flex;
  align-items: center;
  gap: 10px;
}

.prize-popup-content .prize-popup-color {
  width: 40px;
  height: 40px;
  display: inline-block;
  border-radius: 50%;
  border: 2px solid #ccc;
  flex-shrink: 0;
}

.prize-popup-content .prize-popup-name {
  font-size: 22px;
  font-weight: bold;
  color: #333;
}

.close-popup-button {
  padding: 12px 24px;
  font-size: 18px;
  font-weight: bold;
  color: #fff;
  background-color: #424242;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background-color 0.3s ease;
  &:hover {
    background-color: #2e2e2e;
  }
}

@keyframes popup-fade-in {
  from {
    opacity: 0;
    transform: translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.prizes-container {
  flex-shrink: 0;
  width: 400px;
  max-width: 100%;
  border: 1px solid #ddd;
  border-radius: 5px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  background-color: #fffbe6;
  padding: 10px;
}

.prizes-section {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 10px 0;
}

.prizes-section-title {
  text-align: center;
  font-size: 24px;
  color: #333;
  margin: 0;
}

/* 外層容器 */
.preset-menu {
  position: relative;
  display: inline-block;
  // margin-left: 5px;
}

/* 按鈕樣式 */
.preset-toggle,
.toggle-button {
  padding: 5px 10px;
  margin-left: 10px;
  font-size: 14px;
  font-weight: bold;
  color: #fff;
  background-color: #6a1b9a;
  border: none;
  border-radius: 5px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  cursor: pointer;
  transition: background-color 0.3s ease;
  &:hover {
    background-color: #58117f;
  }
  &:disabled {
    background-color: #ccc;
    color: #9e9e9e;
    cursor: not-allowed;
    box-shadow: none;
  }

  span {
    display: flex;
    justify-content: center;
    align-items: center;
  }
}
/* 展開的按鈕區塊 */
.preset-buttons {
  position: absolute;
  top: 110%;
  right: 0;
  display: none;
  flex-wrap: wrap;
  gap: 8px;
  background: #fff;
  border: 1px solid #ccc;
  padding: 10px;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  z-index: 100;
}

/* 展開時顯示 */
.preset-menu.open .preset-buttons {
  display: flex;
}

/* 每個預設按鈕 */
.preset-buttons button {
  padding: 6px 10px;
  border: 1px solid #ddd;
  background: #f9f9f9;
  border-radius: 6px;
  cursor: pointer;
  white-space: nowrap;
  font-size: 14px;
}

.preset-buttons button:hover {
  background: #ffeeba;
}

.add-prize-form {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-wrap: wrap;
  gap: 10px;
  margin-bottom: 10px;
}

.add-prize-form input {
  flex: 1;
  max-width: 100%;
  padding: 10px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 5px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: border-color 0.3s ease, box-shadow 0.3s ease;

  &:focus {
    border-color: #fb8c00;
    box-shadow: 0 0 5px rgba(251, 140, 0, 0.5);
    outline: none;
  }
}

.add-prize-form button {
  padding: 8px 16px;
  font-size: 16px;
  font-weight: bold;
  color: #fff;
  background-color: #f57c00;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: background-color 0.3s ease, transform 0.2s ease;
  &:hover {
    background-color: #e65100;
  }
}

.add-prize-form input:disabled,
.add-prize-form button:disabled {
  background-color: #ccc;
  color: #9e9e9e;
  cursor: not-allowed;
  box-shadow: none;
}

.prizes-list .prizes-header {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 5px;
}

.prizes-list .prizes-header h3 {
  margin: 0;
  font-size: 18px;
  color: #333;
  text-align: center;
}

/* 新增 textarea 樣式 */
.prizes-textarea {
  display: block;
  width: 100%;
  padding: 10px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 5px;
  resize: none;
  min-height: 300px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  margin: 0 0 10px;
  transition: border-color 0.3s ease, box-shadow 0.3s ease;
  &:disabled {
    background-color: #f9f9f9;
    color: #9e9e9e;
    cursor: not-allowed;
  }

  &:focus {
    border-color: #fb8c00;
    box-shadow: 0 0 5px rgba(251, 140, 0, 0.5);
    outline: none;
  }
}

.prizes-list p {
  margin: 0 0 10px;
  font-size: 14px;
  color: #666;
  text-align: center;
}

.prizes-list ul {
  list-style: none;
  margin: 0 0 10px;
  max-height: 300px;
  overflow-y: auto;
  border: 1px solid #ddd;
  border-radius: 5px;
  padding: 10px;
  background-color: #f9f9f9;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.prize-item {
  margin-bottom: 6px;
  display: flex;
  align-items: center;
  padding: 6px 8px;
  border-radius: 5px;
  transition: background-color 0.3s ease;
  cursor: pointer;

  &:hover {
    background-color: #f1f1f1;
  }
  &.editing {
    background-color: #e0e0e0;
    cursor: default;
  }
  &.disabled {
    background-color: transparent;
    cursor: not-allowed;
  }

  .prize-color {
    width: 20px;
    height: 20px;
    display: inline-block;
    border-radius: 50%;
    margin-right: 8px;
    border: 1px solid #ccc;
    flex-shrink: 0;
  }

  .prize-name {
    flex: 1;
    font-size: 16px;
    color: #333;
    text-align: left;
  }

  .edit-prize-input {
    flex: 1;
    width: 100%;
    margin-right: 8px;
    padding: 4px 8px;
    font-size: 16px;
    border: 1px solid #ffa726;
    border-radius: 4px;

    &:focus {
      border-color: #fb8c00;
      box-shadow: 0 0 5px rgba(251, 140, 0, 0.5);
      outline: none;
    }
  }

  .btn {
    flex-shrink: 0;
    padding: 5px 10px;
    font-size: 14px;
    font-weight: bold;
    color: #fff;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    transition: background-color 0.3s ease, transform 0.2s ease;

    &-edit {
      background-color: #0277bd;
      &:hover {
        background-color: #01579b;
      }
    }

    &-delete {
      background-color: #b71c1c;
      &:hover {
        background-color: #7f0000;
      }
    }

    &:disabled {
      background-color: #ccc;
      color: #9e9e9e;
      cursor: not-allowed;
      box-shadow: none;
    }
  }
}

button:active {
  transform: scale(0.97);
}

/* 響應式設計 */
@media (min-width: 850px) {
  .wheel-wrapper {
    flex-direction: row;
    justify-content: center;
    align-items: flex-start;
  }

  .prizes-container {
    width: 350px;
  }
}
</style>
