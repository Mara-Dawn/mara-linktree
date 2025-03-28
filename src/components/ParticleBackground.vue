<!-- ParticleBackground.vue -->
<template>
    <div class="particle-container" ref="particleContainer" @mousemove="handleMouseMove">
        <canvas ref="particleCanvas"></canvas>
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

        // Resize handling
        window.addEventListener('resize', this.handleResize);

        this.$refs.particleContainer.addEventListener('mouseleave', this.handleMouseLeave);
        this.$refs.particleContainer.addEventListener('mouseenter', this.handleMouseEnter);
    },
    beforeUnmount() {
        cancelAnimationFrame(this.animationFrame);
        window.removeEventListener('resize', this.handleResize);
        this.$refs.particleContainer.removeEventListener('mouseleave', this.handleMouseLeave);
        this.$refs.particleContainer.removeEventListener('mouseenter', this.handleMouseEnter);
    },
    methods: {
        initializeCanvas() {
            this.canvas = this.$refs.particleCanvas;
            this.ctx = this.canvas.getContext('2d');

            // Set canvas to full size of container
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
            // Calculate based on screen area and density
            const area = this.canvas.width * this.canvas.height;
            const count = Math.floor(area * this.particleDensity);

            // Set minimum and maximum limits
            return Math.max(30, Math.min(count, 300));
        },

        updateGrid() {
            this.grid = {};

            // Insert each particle into the appropriate grid cell
            this.particles.forEach((particle, index) => {
                // Calculate grid cell coordinates for this particle
                const cellX = Math.floor(particle.x / this.cellSize);
                const cellY = Math.floor(particle.y / this.cellSize);
                const cellKey = `${cellX},${cellY}`;

                // Create the cell if it doesn't exist
                if (!this.grid[cellKey]) {
                    this.grid[cellKey] = [];
                }

                // Store the particle index in the cell
                this.grid[cellKey].push(index);

                // Store the cell coordinates on the particle for quick access
                particle.cellX = cellX;
                particle.cellY = cellY;
            });
        },

        getNeighborParticles(cellX, cellY) {
            const neighbors = [];

            // Check surrounding 9 cells (including the current cell)
            for (let x = cellX - 1; x <= cellX + 1; x++) {
                for (let y = cellY - 1; y <= cellY + 1; y++) {
                    const cellKey = `${x},${y}`;
                    if (this.grid[cellKey]) {
                        // Add indices of particles in this cell
                        neighbors.push(...this.grid[cellKey]);
                    }
                }
            }

            return neighbors;
        },

        updateParticles(deltaTime) {
            const timeScale = this.baseSpeed * Math.min(deltaTime, 0.1);

            this.particles.forEach(particle => {

                if (this.isMouseInside) {
                    // Check for mouse influence
                    const dx = particle.x - this.mouseX;
                    const dy = particle.y - this.mouseY;
                    const distanceSq = dx * dx + dy * dy;

                    // If mouse is close enough, affect the particle
                    if (distanceSq < this.mouseRadius * this.mouseRadius) {
                        const distance = Math.sqrt(distanceSq);
                        const force = (this.mouseRadius - distance) / this.mouseRadius * this.mouseForceModifier;
                        particle.speedX = particle.originalSpeedX + (dx / distance) * force;
                        particle.speedY = particle.originalSpeedY + (dy / distance) * force;
                    }
                }

                // Apply damping to gradually return to original speed
                if (this.isMouseInside) {
                    particle.speedX = particle.speedX * (1 - 0.05 * timeScale) + particle.originalSpeedX * (0.05 * timeScale);
                    particle.speedY = particle.speedY * (1 - 0.05 * timeScale) + particle.originalSpeedY * (0.05 * timeScale);
                } else {
                    // If mouse is outside, return all particles to original speed more quickly
                    particle.speedX = particle.speedX * (1 - 0.1 * timeScale) + particle.originalSpeedX * (0.1 * timeScale);
                    particle.speedY = particle.speedY * (1 - 0.1 * timeScale) + particle.originalSpeedY * (0.1 * timeScale);
                }

                // Apply velocity limit to prevent excessive speeds
                const speedSq = particle.speedX * particle.speedX + particle.speedY * particle.speedY;
                if (speedSq > this.maxSpeed * this.maxSpeed) {
                    const speed = Math.sqrt(speedSq);
                    particle.speedX = (particle.speedX / speed) * this.maxSpeed;
                    particle.speedY = (particle.speedY / speed) * this.maxSpeed;
                }

                particle.x += particle.speedX * timeScale * 60; // Scale by 60 to maintain similar speed to 60fps
                particle.y += particle.speedY * timeScale * 60;

                // Handle edge conditions
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

                    // Skip self
                    if (i === neighborIndex) continue;

                    const otherParticle = this.particles[neighborIndex];
                    const dx = particle.x - otherParticle.x;
                    const dy = particle.y - otherParticle.y;
                    const distanceSq = Math.sqrt(dx * dx + dy * dy);

                    // Skip calculations if particles are too far apart
                    if (distanceSq > this.connectionDistanceSq) continue;

                    const distance = Math.sqrt(dx * dx + dy * dy);

                    // Apply repulsion force if particles are too close
                    if (distance < this.repulsionRadius && distance > 0 && this.frameCount % 2 === 0) {
                        // Quadratic falloff for smoother repulsion
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
        },

        handleResize() {
            this.canvas.width = this.$refs.particleContainer.offsetWidth;
            this.canvas.height = this.$refs.particleContainer.offsetHeight;

            // Recreate particles with the new screen size
            this.createParticles();
        },

        handleMouseEnter(e) {
            this.isMouseInside = true;
            // Update mouse position immediately to avoid sudden jumps
            this.handleMouseMove(e);
        },

        handleMouseLeave() {
            this.isMouseInside = false;
        }
    }
};
</script>

<style scoped>
.particle-container {
    position: relative;
    width: 100%;
    height: 100%;
    overflow: hidden;
    background-color: #1e1e2e;
}

canvas {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}
</style>
