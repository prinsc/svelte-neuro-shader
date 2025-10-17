<!-- NeuroShader.svelte -->
<script>
	import { untrack } from 'svelte';
	import { browser } from '$app/environment';

	// ===== PROPS =====
	let {
		baseColor = $bindable({ r: 0.05, g: 0.35, b: 0.15 }),
		iterations = 25,
		rotationAngle = 0.5,
		initialScale = 8.5,
		scaleGrowth = 1.19,
		timeSpeed = 0.0001,
		pointerStrength = 0.5,
		uvScale = 0.5,
		colorTransitionSpeed = 0.05
	} = $props();

	let canvas;
	let gl;
	let uniforms;
	let animationFrame;
	let currentColor = { ...baseColor };
	let targetColor = { ...baseColor };
	const isMobile =
		browser && /Android|iPhone|iPad|iPod|Opera Mini|IEMobile|WPDesktop/i.test(navigator.userAgent);
	const devicePixelRatio = browser ? (isMobile ? 1 : Math.min(window.devicePixelRatio, 2)) : 1;

	const mobileIterations = Math.floor(iterations * 0.4);
	const targetFPS = isMobile ? 30 : 60;
	const frameInterval = 1000 / targetFPS;
	let lastFrameTime = 0;

	$effect(() => {
		targetColor = { ...baseColor };

		if (currentColor) {
			currentColor = { ...baseColor };
		}
	});

	const pointer = {
		x: 0,
		y: 0,
		tX: 0,
		tY: 0
	};

	const vertexShaderSource = `
    precision mediump float;
    varying vec2 vUv;
    attribute vec2 a_position;

    void main() {
        vUv = .5 * (a_position + 1.);
        gl_Position = vec4(a_position, 0.0, 1.0);
    }
  `;

	const fragmentShaderSource = `
    precision lowp float;

    varying vec2 vUv;
    uniform float u_time;
    uniform float u_ratio;
    uniform vec2 u_pointer_position;
    uniform vec3 u_base_color;
    uniform float u_time_speed;
    uniform float u_pointer_strength;
    uniform float u_initial_scale;
    uniform int u_iterations;

    const float ROTATION_ANGLE = .5;
    const float SCALE_GROWTH = 1.19;
    const float SCALE_POINTER_INFLUENCE = .005;
    const float POINTER_FALLOFF = 2.;
    const float NOISE_POWER_1 = 6.;
    const float NOISE_MULTIPLIER = 2.;
    const float NOISE_POWER_2 = 9.;
    const float NOISE_THRESHOLD = .1;
    const float VIGNETTE_STRENGTH = .75;
    const float UV_SCALE = .5;

    vec2 rotate(vec2 uv, float th) {
        return mat2(cos(th), sin(th), -sin(th), cos(th)) * uv;
    }

    float neuro_shape(vec2 uv, float t, float p) {
        vec2 sine_acc = vec2(0.);
        vec2 res = vec2(0.);
        float scale = u_initial_scale;

        for (int j = 0; j < 25; j++) {
            if (j >= u_iterations) break;

            uv = rotate(uv, ROTATION_ANGLE);
            sine_acc = rotate(sine_acc, ROTATION_ANGLE);
            vec2 layer = uv * scale + float(j) + sine_acc - t;
            sine_acc += sin(layer);
            res += (.5 + .5 * cos(layer)) / scale;
            scale *= (SCALE_GROWTH - SCALE_POINTER_INFLUENCE * p);
        }
        return res.x + res.y;
    }

    void main() {
        vec2 uv = UV_SCALE * vUv;
        uv.x *= u_ratio;

        vec2 pointer = vUv - u_pointer_position;
        pointer.x *= u_ratio;
        float p = clamp(length(pointer), 0., 1.);
        p = u_pointer_strength * pow(1. - p, POINTER_FALLOFF);

        float t = u_time_speed * u_time;
        vec3 color = vec3(0.);

        float noise = neuro_shape(uv, t, p);

        noise = NOISE_MULTIPLIER * pow(noise, NOISE_POWER_1);
        noise += pow(noise, NOISE_POWER_2);
        noise = max(.0, noise - NOISE_THRESHOLD);
        noise *= (VIGNETTE_STRENGTH - length(vUv - .5));

        color = normalize(u_base_color);
        color = color * noise;

        gl_FragColor = vec4(color, noise);
    }
  `;

	function initShader() {
		gl = canvas.getContext('webgl', {
			alpha: true,
			antialias: false,
			powerPreference: isMobile ? 'low-power' : 'high-performance'
		});

		if (!gl) {
			console.warn('WebGL not supported');
			return;
		}

		function createShader(gl, sourceCode, type) {
			const shader = gl.createShader(type);
			gl.shaderSource(shader, sourceCode);
			gl.compileShader(shader);

			if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
				console.error('Shader compilation error: ' + gl.getShaderInfoLog(shader));
				gl.deleteShader(shader);
				return null;
			}

			return shader;
		}

		const vertexShader = createShader(gl, vertexShaderSource, gl.VERTEX_SHADER);
		const fragmentShader = createShader(gl, fragmentShaderSource, gl.FRAGMENT_SHADER);

		function createShaderProgram(gl, vertexShader, fragmentShader) {
			const program = gl.createProgram();
			gl.attachShader(program, vertexShader);
			gl.attachShader(program, fragmentShader);
			gl.linkProgram(program);

			if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
				console.error('Program link error: ' + gl.getProgramInfoLog(program));
				return null;
			}

			return program;
		}

		const shaderProgram = createShaderProgram(gl, vertexShader, fragmentShader);
		if (!shaderProgram) return;

		uniforms = getUniforms(shaderProgram);

		function getUniforms(program) {
			let uniforms = [];
			let uniformCount = gl.getProgramParameter(program, gl.ACTIVE_UNIFORMS);
			for (let i = 0; i < uniformCount; i++) {
				let uniformName = gl.getActiveUniform(program, i).name;
				uniforms[uniformName] = gl.getUniformLocation(program, uniformName);
			}
			return uniforms;
		}

		const vertices = new Float32Array([-1, -1, 1, -1, -1, 1, 1, 1]);
		const vertexBuffer = gl.createBuffer();
		gl.bindBuffer(gl.ARRAY_BUFFER, vertexBuffer);
		gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);

		gl.useProgram(shaderProgram);

		const positionLocation = gl.getAttribLocation(shaderProgram, 'a_position');
		gl.enableVertexAttribArray(positionLocation);
		gl.bindBuffer(gl.ARRAY_BUFFER, vertexBuffer);
		gl.vertexAttribPointer(positionLocation, 2, gl.FLOAT, false, 0, 0);

		resizeCanvas();
	}
	function lerpColor(a, b, t) {
		const safe = (x) => (typeof x === 'number' && !isNaN(x) ? x : 0);
		return {
			r: safe(a.r) + (safe(b.r) - safe(a.r)) * t,
			g: safe(a.g) + (safe(b.g) - safe(a.g)) * t,
			b: safe(a.b) + (safe(b.b) - safe(a.b)) * t
		};
	}
	function render(currentTime) {
		if (currentTime - lastFrameTime < frameInterval) {
			animationFrame = requestAnimationFrame(render);
			return;
		}
		lastFrameTime = currentTime;

		pointer.x += (pointer.tX - pointer.x) * 0.5;
		pointer.y += (pointer.tY - pointer.y) * 0.5;

		currentColor = lerpColor(currentColor, targetColor, colorTransitionSpeed);
		gl.uniform1f(uniforms.u_time, currentTime);
		gl.uniform2f(
			uniforms.u_pointer_position,
			pointer.x / window.innerWidth,
			1 - pointer.y / window.innerHeight
		);
		gl.uniform3f(uniforms.u_base_color, currentColor.r, currentColor.g, currentColor.b);
		gl.uniform1f(uniforms.u_time_speed, timeSpeed);
		gl.uniform1f(uniforms.u_pointer_strength, pointerStrength);
		gl.uniform1f(uniforms.u_initial_scale, initialScale);
		gl.uniform1i(uniforms.u_iterations, isMobile ? mobileIterations : iterations);

		gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
		animationFrame = requestAnimationFrame(render);
	}

	function resizeCanvas() {
		canvas.width = window.innerWidth * devicePixelRatio;
		canvas.height = window.innerHeight * devicePixelRatio;
		gl.uniform1f(uniforms.u_ratio, canvas.width / canvas.height);
		gl.viewport(0, 0, canvas.width, canvas.height);
	}

	function handlePointerMove(e) {
		if (isMobile) return;
		pointer.tX = e.clientX;
		pointer.tY = e.clientY;
	}

	$effect(() => {
		if (!canvas) return;

		initShader();
		render(0);

		window.addEventListener('resize', resizeCanvas);
		if (!isMobile) {
			window.addEventListener('pointermove', handlePointerMove);
		}

		return () => {
			if (animationFrame) {
				cancelAnimationFrame(animationFrame);
			}
			window.removeEventListener('resize', resizeCanvas);
			window.removeEventListener('pointermove', handlePointerMove);
		};
	});
</script>

<canvas bind:this={canvas} id="neuro"></canvas>

<style>
	canvas#neuro {
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		position: relative;
		z-index: -1;
		pointer-events: none !important;
	}
</style>
