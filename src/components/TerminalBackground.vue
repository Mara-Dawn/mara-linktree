<template>
    <div class="terminal-container">
        <pre class="terminal-output">
            <div v-for="(line, index) in displayedLines" :key="index" v-bind:class="{ output: line.output }">
                <span v-if="line.output" class="prompt-symbol">&gt;</span><span>{{ line.text }}</span><span
                    v-if="index == displayedLines.length - 1" class="cursor">|</span>
            </div>
        </pre>
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
            displayedLines: [],
            typingLine: '',
            prompt: 'mara@cafe:~$',
            showResult: false,
            outputLine: false,
            displayedText: '',
            lineIndex: 0,
            charIndex: 0,
        }
    },
    methods: {
        terminalTyping() {

            if (this.lineIndex >= this.commandLines.length) {
                this.typingline = this.prompt + " ";

                setTimeout(() => {
                    this.displayedLines = [];
                    this.displayedLines.push({
                        text: this.typingline,
                        output: false
                    });
                    this.outputLine = '';
                    this.lineIndex = 0;
                    this.charIndex = 0;
                    this.terminalTyping();
                }, 3000);
                return;
            }

            const line = this.commandLines[this.lineIndex];

            if (this.charIndex < line.text.length) {
                this.displayedLines[this.displayedLines.length - 1].text += line.text.charAt(this.charIndex);
                this.charIndex++;
                setTimeout(this.terminalTyping, 100);

            } else {

                if (!this.showResult) {

                    setTimeout(() => {

                        if (line.text == "clear") {

                            this.displayedLines = [];

                        } else if (line.text == "uptime") {

                            const today = new Date();
                            const time = today.getHours() + ":" + today.getMinutes() + ":" + today.getSeconds();

                            let newLine = time + " " + line.output;

                            this.displayedLines.push({
                                text: newLine,
                                output: true
                            });

                        } else {

                            let newLine = line.output;
                            this.displayedLines.push({
                                text: newLine,
                                output: true
                            });

                        }

                        this.typingline = this.prompt + " ";

                        this.displayedLines.push({
                            text: this.typingline,
                            output: false
                        });

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
        this.typingLine = this.prompt + " ";
        this.displayedLines.push({
            text: this.typingLine,
            output: false
        });
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
    color: #cdd6f4;
    padding: 20px;
    overflow: auto;
    font-weight: bold;
}

.terminal-output,
.terminal-output * {
    white-space: normal;
}

.output {
    margin-bottom: 1em;
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

.prompt-symbol {
    color: #f38ba8;
    margin-right: 6px;
    font-weight: bolder;
}

@media (max-width: 850px) {
    .terminal-container {
        font-size: 80%;
    }
}
</style>
