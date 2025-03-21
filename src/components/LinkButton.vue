<template>
  <a v-if="url" class="link" :href="url" target="_blank">
    <span class="icon"><i :class="icon"></i></span>{{ label }}
  </a>
  <div v-else class="link">
    <span class="icon"><i :class="icon"></i></span>{{ label }}
  </div>
</template>

<script>
export default {
  props: ["url", "icon", "label"],
}
</script>

<style scoped>
.link {
  background-color: #313244;
  color: #cdd6f4;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  max-width: 350px;
  padding: 14px;
  margin: 12px auto;
  box-sizing: border-box;
  transition: all 0.2s;
  text-decoration: none;
}

.link:hover {
  background-color: #45475a;
  color: #b4befe;
  transform: translateY(-3px);
}

.icon {
  font-size: 28px;
  margin-right: 12px;
  color: #cba6f7;
}
</style>
