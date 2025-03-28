<template>
    <div class="particle-container" ref="particleContainer">
        <canvas ref="particleCanvas"></canvas>
    </div>
    <div class="content-wrapper">
        <slot></slot>
    </div>
</template>

<script>
export default {
    name: 'ParticleBackground',
    data() {
        return {
            canvas: null,
            ctx: null,
            particles: [],
            particleDensity: 0.00005,
            colors: ['#585b70', '#cba6f7', '#45475a', '#6c7086', '#7f849c'],
            animationFrame: null,
            mouseX: 0,
            mouseY: 0,
            mouseRadius: 300,
            speedModifier: 0.25,
            lineDistance: 160,
            lineDistanceSq: 160 * 160,
            mouseForceModifier: 4,
            isMouseInside: false,
            lastFrameTime: 0,
            baseSpeed: 1,
            grid: {},
            cellSize: 160,
            repulsionRadius: 30,
            repulsionRadiusSq: 30 * 30,
            repulsionStrength: 0.03,
            frameCount: 0,
            maxSpeed: 3,
        };
    },

    mounted() {
        this.initializeCanvas();
        this.createParticles();
        this.animate(performance.now());

        window.addEventListener('mousemove', this.handleMouseMove);
        window.addEventListener('mouseout', this.handleMouseLeave);
        window.addEventListener('mouseover', this.handleMouseEnter);
        window.addEventListener('resize', this.handleResize);
    },

    beforeUnmount() {
        cancelAnimationFrame(this.animationFrame);

        window.removeEventListener('resize', this.handleResize);
        window.removeEventListener('mousemove', this.handleMouseMove);
        window.removeEventListener('mouseout', this.handleMouseLeave);
        window.removeEventListener('mouseover', this.handleMouseEnter);
    },

    methods: {
        initializeCanvas() {
            this.canvas = this.$refs.particleCanvas;
            this.ctx = this.canvas.getContext('2d');

            this.canvas.width = this.$refs.particleContainer.offsetWidth;
            this.canvas.height = this.$refs.particleContainer.offsetHeight;

            this.lineDistanceSq = this.lineDistance * this.lineDistance;
            this.repulsionRadiusSq = this.repulsionRadius * this.repulsionRadius;
        },

        createParticles() {
            this.particles = [];
            const particleCount = this.calculateParticleCount();

            for (let i = 0; i < particleCount; i++) {
                this.particles.push({
                    x: Math.random() * this.canvas.width,
                    y: Math.random() * this.canvas.height,
                    radius: Math.random() * 3 + 1,
                    color: this.colors[Math.floor(Math.random() * this.colors.length)],
                    speedX: (Math.random() * 1 - 0.5) * this.speedModifier,
                    speedY: (Math.random() * 1 - 0.5) * this.speedModifier,
                    originalSpeedX: (Math.random() * 1 - 0.5) * this.speedModifier,
                    originalSpeedY: (Math.random() * 1 - 0.5) * this.speedModifier,
                });
            }
        },

        animate(currentTime) {
            if (!this.lastFrameTime) {
                this.lastFrameTime = currentTime;
            }

            const deltaTime = (currentTime - this.lastFrameTime) / 1000;
            this.lastFrameTime = currentTime;

            this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

            this.updateParticles(deltaTime);
            this.updateGrid();
            this.drawParticles(deltaTime);

            this.animationFrame = requestAnimationFrame(this.animate);
        },

        calculateParticleCount() {
            const area = this.canvas.width * this.canvas.height;
            const count = Math.floor(area * this.particleDensity);

            return Math.max(30, Math.min(count, 300));
        },

        updateGrid() {
            this.grid = {};

            this.particles.forEach((particle, index) => {
                const cellX = Math.floor(particle.x / this.cellSize);
                const cellY = Math.floor(particle.y / this.cellSize);
                const cellKey = `${cellX},${cellY}`;

                if (!this.grid[cellKey]) {
                    this.grid[cellKey] = [];
                }

                this.grid[cellKey].push(index);

                particle.cellX = cellX;
                particle.cellY = cellY;
            });
        },

        getNeighborParticles(cellX, cellY) {
            const neighbors = [];

            for (let x = cellX - 1; x <= cellX + 1; x++) {
                for (let y = cellY - 1; y <= cellY + 1; y++) {
                    const cellKey = `${x},${y}`;
                    if (this.grid[cellKey]) {
                        neighbors.push(...this.grid[cellKey]);
                    }
                }
            }

            return neighbors;
        },

        updateParticles(deltaTime) {
            const timeScale = this.baseSpeed * Math.min(deltaTime, 0.1);
            const dampingFactor = this.isMouseInside ? 0.15 : 0.25;
            const maxSpeedSq = this.maxSpeed * this.maxSpeed;

            this.particles.forEach(particle => {
                let hasMouseInfluence = false;

                if (this.isMouseInside) {
                    const dx = particle.x - this.mouseX;
                    const dy = particle.y - this.mouseY;
                    const distanceSq = dx * dx + dy * dy;

                    if (distanceSq < this.mouseRadius * this.mouseRadius) {
                        const distance = Math.sqrt(distanceSq);
                        const force = (this.mouseRadius - distance) / this.mouseRadius * this.mouseForceModifier;
                        particle.speedX = particle.originalSpeedX + (dx / distance) * force;
                        particle.speedY = particle.originalSpeedY + (dy / distance) * force;
                        hasMouseInfluence = true;
                    }
                }

                if (!hasMouseInfluence) {
                    particle.speedX = particle.speedX * (1 - dampingFactor * timeScale) +
                        particle.originalSpeedX * (dampingFactor * timeScale);
                    particle.speedY = particle.speedY * (1 - dampingFactor * timeScale) +
                        particle.originalSpeedY * (dampingFactor * timeScale);
                }

                const currentSpeedSq = particle.speedX * particle.speedX + particle.speedY * particle.speedY;
                if (currentSpeedSq > maxSpeedSq) {
                    const speedFactor = this.maxSpeed / Math.sqrt(currentSpeedSq);
                    particle.speedX *= speedFactor;
                    particle.speedY *= speedFactor;
                }

                if (currentSpeedSq > maxSpeedSq * 0.7) {
                    if (Math.random() < 0.1) {
                        particle.speedX *= 0.9;
                        particle.speedY *= 0.9;
                    }
                }

                particle.x += particle.speedX * timeScale * 60;
                particle.y += particle.speedY * timeScale * 60;

                if (particle.x < 0) particle.x = this.canvas.width;
                if (particle.x > this.canvas.width) particle.x = 0;
                if (particle.y < 0) particle.y = this.canvas.height;
                if (particle.y > this.canvas.height) particle.y = 0;
            });
        },

        drawParticles(deltaTime) {
            const timeScale = this.baseSpeed * Math.min(deltaTime, 0.1) * 60;

            for (let i = 0; i < this.particles.length; i++) {
                const particle = this.particles[i];
                this.ctx.beginPath();
                this.ctx.arc(particle.x, particle.y, particle.radius, 0, Math.PI * 2);
                this.ctx.fillStyle = particle.color;
                this.ctx.fill();
            }

            for (let i = 0; i < this.particles.length; i++) {
                const particle = this.particles[i];
                const neighbors = this.getNeighborParticles(particle.cellX, particle.cellY);

                for (let j = 0; j < neighbors.length; j++) {
                    const neighborIndex = neighbors[j];

                    if (i === neighborIndex) continue;

                    const otherParticle = this.particles[neighborIndex];
                    const dx = particle.x - otherParticle.x;
                    const dy = particle.y - otherParticle.y;
                    const distanceSq = Math.sqrt(dx * dx + dy * dy);

                    if (distanceSq > this.connectionDistanceSq) continue;

                    const distance = Math.sqrt(dx * dx + dy * dy);

                    if (distance < this.repulsionRadius && distance > 0 && this.frameCount % 2 === 0) {
                        const forceFactor = Math.pow(1 - distance / this.repulsionRadius, 2);
                        const force = this.repulsionStrength * forceFactor;

                        particle.speedX += (dx / distance) * force * timeScale;
                        particle.speedY += (dy / distance) * force * timeScale;
                    }

                    if (neighborIndex > i && distance < this.lineDistance) {
                        const gradient = this.ctx.createLinearGradient(
                            particle.x, particle.y,
                            otherParticle.x, otherParticle.y
                        );
                        gradient.addColorStop(0, particle.color);
                        gradient.addColorStop(1, otherParticle.color);

                        this.ctx.beginPath();
                        this.ctx.strokeStyle = gradient;
                        this.ctx.globalAlpha = 0.3 * (1 - distance / this.lineDistance);
                        this.ctx.lineWidth = 1.5;
                        this.ctx.moveTo(particle.x, particle.y);
                        this.ctx.lineTo(otherParticle.x, otherParticle.y);
                        this.ctx.stroke();
                        this.ctx.globalAlpha = 1;
                    }
                }
            }
        },

        handleMouseMove(e) {
            const rect = this.canvas.getBoundingClientRect();

            this.mouseX = e.clientX - rect.left;
            this.mouseY = e.clientY - rect.top;

            this.isMouseInside =
                this.mouseX >= 0 &&
                this.mouseX <= rect.width &&
                this.mouseY >= 0 &&
                this.mouseY <= rect.height;
        },

        handleResize() {
            this.canvas.width = this.$refs.particleContainer.offsetWidth;
            this.canvas.height = this.$refs.particleContainer.offsetHeight;

            this.createParticles();
        },

        handleMouseLeave() {
            this.isMouseInside = false;
        },

        handleMouseEnter() {
            this.isMouseInside = true;
        },
    }
};
</script>

<style scoped>
.particle-container {
    position: fixed;
    width: 100%;
    height: 100%;
    overflow: hidden;
    background-color: #1e1e2e;
    z-index: 0;
}

canvas {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}

.content-wrapper {
    position: relative;
    z-index: 1;
    pointer-events: none;
}

.content-wrapper>>>* {
    pointer-events: auto;
}
</style>
