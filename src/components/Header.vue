<script setup>
import Logo from './icons/Logo.vue';
</script>

<template>
  <header class="custom-header">
    <Logo />
    <h1 class="web-title">線上抽籤轉盤</h1>
  </header>
</template>

<style lang="scss" scoped>
.custom-header {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 20px;
  padding: 16px 0;
  background-color: #fffbe6;
  border-bottom: 1px solid #e0dab8;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.04);
}

.web-title {
  font-size: 28px;
  line-height: 1;
  color: #333;
  margin: 0;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.1);
}
</style>
