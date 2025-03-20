<template>
  <div class="terminal-container">
    <pre class="terminal-output">{{ displayedText }}<span class="cursor">|</span></pre>
  </div>
</template>

<script>
export default {
  data() {
    return {
      commandLines: [
        { text: "whoami", output: "mara" },
        { text: "ls ~/projects", output: "linktree-vue  portfolio  cool-project" },
        { text: "uptime", output: "up 3 days, 4:25, 3 users" },
        { text: "echo Welcome!", output: "Welcome!" },
        { text: "clear", output: "" },
      ],
      prompt: 'mara@cafe:~$',
      showResult: false,
      displayedText: '',
      lineIndex: 0,
      charIndex: 0,
    }
  },
  methods: {
    terminalTyping() {

      if (this.lineIndex >= this.commandLines.length) {
        this.displayedText = this.prompt + " ";
        setTimeout(() => {
          this.lineIndex = 0;
          this.charIndex = 0;
          this.terminalTyping();
        }, 3000);
        return;
      }

      let line = this.commandLines[this.lineIndex];

      if (this.charIndex < line.text.length) {
        this.displayedText += line.text.charAt(this.charIndex);
        this.charIndex++;
        setTimeout(this.terminalTyping, 100);
      } else {
        if (!this.showResult) {
          setTimeout(() => {
            if (line.text == "clear") {
              this.displayedText = this.prompt + " ";
            } else if (line.text == "uptime") {
              const today = new Date();
              const time = today.getHours() + ":" + today.getMinutes() + ":" + today.getSeconds();
              this.displayedText += "\n> " + time + " " + line.output + "\n\n" + this.prompt + " ";
            } else {
              this.displayedText += "\n> " + line.output + "\n\n" + this.prompt + " ";
            }
            this.showResult = true;
            this.terminalTyping();
          }, 800);
        } else {
          setTimeout(() => {
            this.showResult = false;
            this.lineIndex++;
            this.charIndex = 0;
            this.terminalTyping();
          }, 3000);
        }
      }
    },
  },
  mounted() {
    this.displayedText = this.prompt + " ";
    setTimeout(this.terminalTyping, 3000);
  }
}
</script>

<style scoped>
.terminal-container {
  position: fixed;
  font-size: 140%;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: #1e1e2e;
  color: #cdd6f4;
  padding: 20px;
  overflow: auto;
}

.cursor {
  animation: blink 0.9s infinite;
  color: #f38ba8;
}

@keyframes blink {
  50% {
    opacity: 0;
  }
}

@media (max-width: 1250px) {
  .terminal-container {
    font-size: 65%;
  }
}
</style>
