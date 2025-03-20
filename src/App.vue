<template>
  <div id="app">
    <TerminalBackground />
    <div class="container">
      <img class="user-avatar" :src="avatar" alt="Avatar">
      <h2>{{ username }}</h2>

      <LinkButton v-for="link in links" :key="link.account" :url="link.url" :icon="link.icon" :label="link.account" />

      <LinkButton :icon="discordIcon" :label="discordName" />
    </div>
  </div>
</template>

<script>
import TerminalBackground from '@/components/TerminalBackground.vue'
import LinkButton from '@/components/LinkButton.vue'

export default {
  name: 'App',
  components: { TerminalBackground, LinkButton },
  data() {
    return {
      username: 'Mara',
      avatar: '/avatar.png',
      discordName: 'maradawn',
      discordIcon: 'fab fa-discord',
      links: [
        { url: 'https://github.com/Mara-Dawn', icon: 'fab fa-github', account: '@Mara-Dawn' },
        { url: 'https://www.youtube.com/@MaraDawn', icon: 'fab fa-youtube', account: '@MaraDawn' },
      ],
    }
  },
}
</script>

<style>
@import '@/assets/style.css';
</style>
