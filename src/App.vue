<template>
    <ParticleBackground>
        <TerminalBackground />
        <div class="content">
            <img class="user-avatar" :src="avatar" alt="Avatar">
            <h2>{{ title }}</h2>

            <LinkButton v-for="link in links" :key="link.account" :url="link.url" :icon="link.icon"
                :label="link.account" />

            <LinkButton :icon="discordIcon" :label="discordName" />
        </div>
    </ParticleBackground>
</template>

<script>
import TerminalBackground from '@/components/TerminalBackground.vue'
import LinkButton from '@/components/LinkButton.vue'
import ParticleBackground from '@/components/ParticleBackground.vue'

export default {
    name: 'App',
    components: { TerminalBackground, ParticleBackground, LinkButton },
    data() {
        return {
            title: 'Mara\'s Socials',
            avatar: '/avatar.png',
            discordName: 'maradawn',
            discordIcon: 'nf nf-fa-discord',
            links: [
                { url: 'https://github.com/Mara-Dawn', icon: 'nf nf-dev-github', account: '@Mara-Dawn' },
                { url: 'https://www.youtube.com/@MaraDawn', icon: 'nf nf-fa-youtube', account: '@MaraDawn' },
                { url: 'https://github.com/Mara-Dawn/dotfiles', icon: 'nf nf-md-arch', account: '.dotfiles' },
            ],
        }
    },
}
</script>

<style>
@import '@/assets/style.css';
</style>
