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
            particleDensity: 0.00007,
            colors: ['#585b70', '#cba6f7', '#45475a', '#6c7086', '#7f849c'],
            minRadius: 1,
            maxRadius: 5,

            lineDistance: 140,
            lineWidth: 1,
            lineOpacity: 0.4,

            maxLinesPerFrame: 3000,
            lineSkipDensity: 1.5,
            connectionCheckFrequency: 1,

            animationFrame: null,
            lastFrameTime: 0,
            frameCount: 0,
            prevScreenWidth: 0,
            prevScreenHeight: 0,
            maxParticleCount: 600,
            minParticleCount: 30,

            mouseX: 0,
            mouseY: 0,
            mouseRadius: 200,
            mouseForceModifier: 300,
            isMouseInside: false,

            grid: {},

            baseSpeed: 2.5,
            speedModifier: 1,
            impulseDecay: 0.57,
            returnRate: 0.9,
            repulsionRadius: 35,
            repulsionStrength: 2.5,
            repulsionFactor: 0.4,

            massInfluence: 0.3,     // (0 = no influence, 1 = normal, 2 = exaggerated)
            massSpeedFactor: 1.5,   // How much faster smaller particles move
            massMouseFactor: 0.1,   // How much more smaller particles respond to mouse
            massEmitterFactor: 0.15, // How much more larger particles push others
            massReceiverFactor: 1.8,  // How much more smaller particles get pushed
            maxSpeedBase: 25,        // Base maximum speed


        };
    },

    mounted() {
        this.initializeCanvas();
        this.createParticles();
        this.animate(performance.now());

        window.addEventListener('mousemove', this.handleMouseMove);
        window.addEventListener('mouseout', this.handleMouseLeave);
        window.addEventListener('mouseover', this.handleMouseEnter);

        window.addEventListener('touchstart', this.handleTouch, { passive: false });
        window.addEventListener('touchmove', this.handleTouch, { passive: false });
        window.addEventListener('touchend', this.handleTouchEnd);

        window.addEventListener('resize', this.handleResize);
    },

    beforeUnmount() {
        cancelAnimationFrame(this.animationFrame);

        window.removeEventListener('mousemove', this.handleMouseMove);
        window.removeEventListener('mouseout', this.handleMouseLeave);
        window.removeEventListener('mouseover', this.handleMouseEnter);

        window.removeEventListener('touchstart', this.handleTouch);
        window.removeEventListener('touchmove', this.handleTouch);
        window.removeEventListener('touchend', this.handleTouchEnd);

        window.removeEventListener('resize', this.handleResize);
    },

    methods: {
        initializeCanvas() {
            this.canvas = this.$refs.particleCanvas;
            this.ctx = this.canvas.getContext('2d');

            const dpr = window.devicePixelRatio || 1;

            const viewportWidth = window.innerWidth;
            const viewportHeight = window.innerHeight;
            this.$refs.particleContainer.style.width = `${viewportWidth}px`;
            this.$refs.particleContainer.style.height = `${viewportHeight}px`;

            this.canvas.style.width = `${viewportWidth}px`;
            this.canvas.style.height = `${viewportHeight}px`;

            this.canvas.width = viewportWidth * dpr;
            this.canvas.height = viewportHeight * dpr;

            this.ctx.scale(dpr, dpr);

            this.cellSize = this.lineDistance * 1.1;
        },

        createParticles() {
            this.particles = [];
            const particleCount = this.calculateParticleCount();

            const visibleWidth = window.innerWidth;
            const visibleHeight = window.innerHeight;

            for (let i = 0; i < particleCount; i++) {
                let radius = Math.random() * (this.maxRadius - this.minRadius) + this.minRadius;

                const padding = radius * 2;
                const x = padding + Math.random() * (visibleWidth - padding * 2);
                const y = padding + Math.random() * (visibleHeight - padding * 2);

                let color = this.colors[Math.floor(Math.random() * this.colors.length)];

                if (i === 0) {
                    color = '#f38ba8';
                    radius = this.maxRadius + 5;
                }

                const mass = Math.PI * radius * radius;

                this.particles.push({
                    x: x,
                    y: y,
                    radius: radius,
                    mass: mass,
                    color: color,
                    speedX: (Math.random() * 1 - 0.5) * this.speedModifier * (this.massSpeedFactor / mass),
                    speedY: (Math.random() * 1 - 0.5) * this.speedModifier * (this.massSpeedFactor / mass),
                    originalSpeedX: (Math.random() * 1 - 0.5) * this.speedModifier * (this.massSpeedFactor / mass),
                    originalSpeedY: (Math.random() * 1 - 0.5) * this.speedModifier * (this.massSpeedFactor / mass),
                    impulseX: 0,
                    impulseY: 0
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

            this.updateGrid();
            this.updateParticles(deltaTime);

            this.animationFrame = requestAnimationFrame(this.animate);

            this.frameCount++;
            if (this.frameCount % 10 === 0) {
                for (let i = 0; i < this.particles.length; i++) {
                    delete this.particles[i].localDensity;
                }
            }
        },

        calculateParticleCount() {
            const visibleWidth = window.innerWidth;
            const visibleHeight = window.innerHeight;

            const visibleArea = visibleWidth * visibleHeight;

            const count = Math.floor(visibleArea * this.particleDensity);

            return Math.max(this.minParticleCount, Math.min(count, this.maxParticleCount));
        },

        calculateLocalDensity(cellX, cellY) {
            let count = 0;

            for (let x = cellX - 1; x <= cellX + 1; x++) {
                for (let y = cellY - 1; y <= cellY + 1; y++) {
                    const cellKey = `${x},${y}`;
                    if (this.grid[cellKey]) {
                        count += this.grid[cellKey].length;
                    }
                }
            }

            const avgDensity = count / 9; // 3x3 grid of cells

            return avgDensity / this.particles.length * 100;
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

            const repulsionRadiusSq = this.repulsionRadius * this.repulsionRadius;
            const lineDistanceSq = this.lineDistance * this.lineDistance;
            const mouseRadiusSq = this.mouseRadius * this.mouseRadius;
            let drawnLineCount = 0;

            for (let i = 0; i < this.particles.length; i++) {
                const particle = this.particles[i];
                this.ctx.beginPath();
                this.ctx.arc(particle.x, particle.y, particle.radius, 0, Math.PI * 2);
                this.ctx.fillStyle = particle.color;
                this.ctx.fill();
            }

            for (let i = 0; i < this.particles.length; i++) {

                const particle = this.particles[i];
                const inverseMass = this.massInfluence > 0
                    ? Math.pow(1 / particle.mass, this.massInfluence)
                    : 1.0; // If massInfluence is 0, all particles have the same inverse mass

                // Mouse interaction
                if (this.isMouseInside) {
                    const dx = particle.x - this.mouseX;
                    const dy = particle.y - this.mouseY;
                    const distanceSq = dx * dx + dy * dy;

                    if (distanceSq < mouseRadiusSq && distanceSq > 0) {
                        const distance = Math.sqrt(distanceSq);

                        const force = (this.mouseRadius - distance) / this.mouseRadius;
                        const impulseStrength = force * this.mouseForceModifier * timeScale *
                            (this.massInfluence > 0 ? inverseMass * this.massMouseFactor : 1.0);

                        particle.impulseX = (dx / distance) * impulseStrength;
                        particle.impulseY = (dy / distance) * impulseStrength;
                    }
                }

                const neighbors = this.getNeighborParticles(particle.cellX, particle.cellY);

                // Repulsion and Line connections
                for (let j = 0; j < neighbors.length; j++) {
                    const neighborIndex = neighbors[j];

                    if (i === neighborIndex) continue;

                    const otherParticle = this.particles[neighborIndex];
                    const dx = particle.x - otherParticle.x;
                    const dy = particle.y - otherParticle.y;
                    const distanceSq = dx * dx + dy * dy;

                    if (distanceSq < Math.max(repulsionRadiusSq, lineDistanceSq) && distanceSq > 0) {
                        const distance = Math.sqrt(distanceSq);

                        if (particle.localDensity === undefined) {
                            particle.localDensity = this.calculateLocalDensity(particle.cellX, particle.cellY);
                        }

                        // Repulsion
                        if (distanceSq < repulsionRadiusSq) {

                            if (particle.localDensity > this.lineSkipDensity * 2 &&
                                distanceSq > repulsionRadiusSq * 0.5) {
                                continue;
                            }

                            const forceFactor = Math.pow(1 - distance / this.repulsionRadius, 2);

                            const emitterFactor = this.massInfluence > 0
                                ? Math.pow(otherParticle.mass, this.massInfluence) * this.massEmitterFactor
                                : 1.0;
                            const receiverFactor = this.massInfluence > 0
                                ? inverseMass * this.massReceiverFactor
                                : 1.0;

                            const impulseStrength = Math.min(
                                this.repulsionStrength * forceFactor * timeScale *
                                this.repulsionFactor * emitterFactor * receiverFactor,
                                10
                            );

                            if (distance > 0) {
                                particle.impulseX += (dx / distance) * impulseStrength;
                                particle.impulseY += (dy / distance) * impulseStrength;
                            }
                        }

                        // Line connections
                        if (neighborIndex > i && distanceSq < lineDistanceSq) {

                            if (drawnLineCount > this.maxLinesPerFrame) {
                                continue;
                            }

                            if (particle.localDensity > this.lineSkipDensity) {
                                const skipFactor = Math.min(0.8, (particle.localDensity - this.lineSkipDensity) / 6);
                                const hashValue = (i * 31 + neighborIndex) % 100 / 100;

                                if (hashValue < skipFactor) {
                                    continue;
                                }
                            }

                            const gradient = this.ctx.createLinearGradient(
                                particle.x, particle.y,
                                otherParticle.x, otherParticle.y
                            );
                            gradient.addColorStop(0, particle.color);
                            gradient.addColorStop(1, otherParticle.color);

                            this.ctx.beginPath();
                            this.ctx.strokeStyle = gradient;
                            this.ctx.globalAlpha = this.lineOpacity * (1 - distance / this.lineDistance);
                            this.ctx.lineWidth = this.lineWidth;
                            this.ctx.moveTo(particle.x, particle.y);
                            this.ctx.lineTo(otherParticle.x, otherParticle.y);
                            this.ctx.stroke();
                            this.ctx.globalAlpha = 1;

                            drawnLineCount++;
                        }

                    }
                }

                // Apply impulse
                particle.speedX += particle.impulseX;
                particle.speedY += particle.impulseY;

                particle.impulseX *= this.impulseDecay;
                particle.impulseY *= this.impulseDecay;

                if (Math.abs(particle.impulseX) < 0.001) particle.impulseX = 0;
                if (Math.abs(particle.impulseY) < 0.001) particle.impulseY = 0;

                particle.speedX = particle.speedX * (1 - this.returnRate * timeScale) +
                    particle.originalSpeedX * (this.returnRate * timeScale);
                particle.speedY = particle.speedY * (1 - this.returnRate * timeScale) +
                    particle.originalSpeedY * (this.returnRate * timeScale);

                const maxSpeedForMass = this.massInfluence > 0
                    ? this.maxSpeedBase * (this.massSpeedFactor / Math.pow(particle.mass, this.massInfluence))
                    : this.maxSpeedBase;
                const currentSpeedSq = particle.speedX * particle.speedX + particle.speedY * particle.speedY;
                if (currentSpeedSq > maxSpeedForMass * maxSpeedForMass) {
                    const speed = Math.sqrt(currentSpeedSq);
                    particle.speedX = (particle.speedX / speed) * maxSpeedForMass;
                    particle.speedY = (particle.speedY / speed) * maxSpeedForMass;
                }

                particle.x += particle.speedX * timeScale * 60;
                particle.y += particle.speedY * timeScale * 60;


                // Boundary check
                const visibleWidth = window.innerWidth;
                const visibleHeight = window.innerHeight;

                if (particle.x < -particle.radius) {
                    particle.x = visibleWidth + particle.radius;
                } else if (particle.x > visibleWidth + particle.radius) {
                    particle.x = -particle.radius;
                }

                if (particle.y < -particle.radius) {
                    particle.y = visibleHeight + particle.radius;
                } else if (particle.y > visibleHeight + particle.radius) {
                    particle.y = -particle.radius;
                }

                if (isNaN(particle.x) || isNaN(particle.y) ||
                    !isFinite(particle.x) || !isFinite(particle.y) ||
                    Math.abs(particle.speedX) > this.maxSpeedBase * 10 ||
                    Math.abs(particle.speedY) > this.maxSpeedBase * 10) {

                    particle.x = Math.random() * this.canvas.width;
                    particle.y = Math.random() * this.canvas.height;
                    particle.speedX = particle.originalSpeedX;
                    particle.speedY = particle.originalSpeedY;
                    particle.impulseX = 0;
                    particle.impulseY = 0;
                }

            }
        },

        handleResize() {
            const dpr = window.devicePixelRatio || 1;
            const viewportWidth = window.innerWidth;
            const viewportHeight = window.innerHeight;

            this.$refs.particleContainer.style.width = `${viewportWidth}px`;
            this.$refs.particleContainer.style.height = `${viewportHeight}px`;

            this.canvas.style.width = `${viewportWidth}px`;
            this.canvas.style.height = `${viewportHeight}px`;

            this.canvas.width = viewportWidth * dpr;
            this.canvas.height = viewportHeight * dpr;
            this.ctx.scale(dpr, dpr);

            if (Math.abs(this.prevScreenWidth - viewportWidth) > 250 ||
                Math.abs(this.prevScreenHeight - viewportHeight) > 250) {

                this.prevScreenWidth = viewportWidth;
                this.prevScreenHeight = viewportHeight;
                this.createParticles();
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

        handleTouch(e) {
            const rect = this.canvas.getBoundingClientRect();
            const touch = e.touches[0]; // Get the first touch point

            this.mouseX = touch.clientX - rect.left;
            this.mouseY = touch.clientY - rect.top;

            // Check if touch is within canvas or close to it (within mouseRadius)
            const isNearCanvas =
                this.mouseX >= -this.mouseRadius &&
                this.mouseX <= rect.width + this.mouseRadius &&
                this.mouseY >= -this.mouseRadius &&
                this.mouseY <= rect.height + this.mouseRadius;

            if (isNearCanvas) {
                this.isMouseInside = true;
            }
        },

        handleTouchEnd() {
            this.isMouseInside = false;
        },

        handleMouseLeave() {
            this.isMouseInside = false;
        },

        handleMouseEnter(e) {
            this.isMouseInside = true;
            this.handleMouseMove(e);
        },
    }
};
</script>

<style scoped>
.particle-container {
    position: fixed;
    width: 100vw;
    height: 100vh;
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
