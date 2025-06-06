<script setup>
import Logo from './icons/Logo.vue';
import IconGithub from './icons/IconGithub.vue';
</script>

<template>
  <footer class="custom-footer">
    <Logo class="footer-wheel-logo" />
    <span>線上抽籤轉盤 © 2025 – </span>
    <a href="https://github.com/YPINPIN" target="_blank">
      <IconGithub /><span>YPINPIN</span>
    </a>
  </footer>
</template>

<style lang="scss" scoped>
.custom-footer {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 14px;
  padding: 8px;
  background-color: #fffbe6;
  color: #555;
  border-top: 1px solid #e0dab8;
  .footer-wheel-logo {
    width: 20px;
    height: 20px;
  }

  a {
    display: flex;
    justify-content: center;
    align-items: flex-end;
    font-weight: 600;
    text-decoration: none;
    margin-left: 5px;
    color: #555;

    &:hover {
      color: #000;

      text-decoration: underline;
    }
  }
}
</style>
