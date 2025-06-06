<template>
  <svg
    width="160"
    height="160"
    viewBox="0 0 40 40"
    xmlns="http://www.w3.org/2000/svg"
  >
    <defs>
      <!-- 為每個象限定義不同漸層 -->
      <linearGradient id="grad1" x1="0%" y1="0%" x2="100%" y2="100%">
        <stop offset="0%" stop-color="#f44336" />
        <!-- 紅 -->
        <stop offset="100%" stop-color="#ff9800" />
        <!-- 橘 -->
      </linearGradient>
      <linearGradient id="grad2" x1="100%" y1="0%" x2="0%" y2="100%">
        <stop offset="0%" stop-color="#ffd600" />
        <!-- 黃 -->
        <stop offset="100%" stop-color="#4caf50" />
        <!-- 綠 -->
      </linearGradient>
      <linearGradient id="grad3" x1="100%" y1="100%" x2="0%" y2="0%">
        <stop offset="0%" stop-color="#00bcd4" />
        <!-- 青 -->
        <stop offset="100%" stop-color="#2196f3" />
        <!-- 藍 -->
      </linearGradient>
      <linearGradient id="grad4" x1="0%" y1="100%" x2="100%" y2="0%">
        <stop offset="0%" stop-color="#9c27b0" />
        <!-- 紫 -->
        <stop offset="100%" stop-color="#f44336" />
        <!-- 回紅 -->
      </linearGradient>
    </defs>

    <!-- 四個外環漸層切片 -->
    <path d="M20,20 L20,1 A19,19 0 0,1 39,20 Z" fill="url(#grad1)" />
    <path d="M20,20 L39,20 A19,19 0 0,1 20,39 Z" fill="url(#grad2)" />
    <path d="M20,20 L20,39 A19,19 0 0,1 1,20 Z" fill="url(#grad3)" />
    <path d="M20,20 L1,20 A19,19 0 0,1 20,1 Z" fill="url(#grad4)" />

    <!-- 外圓環描邊 -->
    <circle
      cx="20"
      cy="20"
      r="19.5"
      fill="none"
      stroke="#facc15"
      stroke-width="1"
    />

    <!-- 中心紅點 -->
    <circle cx="20" cy="20" r="3" fill="#e11d48" />

    <!-- 指針 -->
    <polygon points="19.3,2.5 20.7,2.5 20,10" fill="#000000" />
  </svg>
</template>

<style lang="scss" scoped>
svg {
  width: 30px;
  height: 30px;
  vertical-align: middle;
  margin-right: 8px;
}
</style>
