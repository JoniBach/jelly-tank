<script>
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
	import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer';
	import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass';
	import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass';
	import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader';

	let canvas;

	onMount(() => {
		const scene = new THREE.Scene();
		const clock = new THREE.Clock(); // For time-based animation
		const camera = new THREE.PerspectiveCamera(
			75,
			window.innerWidth / window.innerHeight,
			0.1,
			1000
		);
		const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
		renderer.setSize(window.innerWidth, window.innerHeight);
		renderer.outputEncoding = THREE.sRGBEncoding;

		// OrbitControls
		const controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;
		controls.dampingFactor = 0.05;
		controls.target.set(0, 0, 0);

		// Post-processing: Bloom effect
		const composer = new EffectComposer(renderer);
		const renderPass = new RenderPass(scene, camera);
		composer.addPass(renderPass);

		const bloomPass = new UnrealBloomPass(
			new THREE.Vector2(window.innerWidth, window.innerHeight),
			3.0,
			0.6,
			0.8
		);
		composer.addPass(bloomPass);

		// Environment map for reflections and refraction
		let envMap;
		new RGBELoader().load('/path-to-your-hdr.hdr', (texture) => {
			texture.mapping = THREE.EquirectangularReflectionMapping;
			scene.environment = texture;
			envMap = texture;
		});

		// Lighting
		const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
		scene.add(ambientLight);

		const directionalLight = new THREE.DirectionalLight(0xffffff, 1.5);
		directionalLight.position.set(5, 10, 7.5);
		scene.add(directionalLight);

		// Function to create a gradient texture
		const createGradientTexture = (width, height, colors) => {
			const canvas = document.createElement('canvas');
			canvas.width = width;
			canvas.height = height;
			const context = canvas.getContext('2d');

			// Create gradient
			const gradient = context.createLinearGradient(0, height, 0, 0);
			colors.forEach(([offset, color]) => gradient.addColorStop(offset, color));

			// Fill the canvas with the gradient
			context.fillStyle = gradient;
			context.fillRect(0, 0, width, height);

			// Create texture
			return new THREE.CanvasTexture(canvas);
		};

		// Create the gradient texture (base map)
		const gradientTexture = createGradientTexture(256, 256, [
			[0, '#D013C1'],
			[0.4, '#CE4F63'],
			[0.6, '#17A898'],
			[1, '#17A898']
		]);

		// Load the GLTF model
		const loader = new GLTFLoader();
		loader.load(
			'/jellytank-fish.gltf',
			(gltf) => {
				console.log('Model loaded successfully:', gltf);

				gltf.scene.traverse((node) => {
					if (node.isMesh) {
						// Apply jelly-like material with a custom vertex shader for wave animation
						const originalMaterial = new THREE.MeshPhysicalMaterial({
							map: gradientTexture,
							emissive: new THREE.Color(0xffffff),
							emissiveMap: gradientTexture,
							emissiveIntensity: 1.5,
							transparent: true,
							opacity: 0.8,
							roughness: 0.1,
							metalness: 0.05,
							clearcoat: 1.0,
							clearcoatRoughness: 0.05,
							transmission: 1.0,
							thickness: 1.5,
							envMap: envMap,
							envMapIntensity: 1.2,
							ior: 1.45,
							sheen: 0.3,
							sheenColor: new THREE.Color('#ff00ff')
						});

						// Modify the vertex shader to add wave animation effect
						originalMaterial.onBeforeCompile = (shader) => {
							shader.uniforms.time = { value: 0 };

							// Inject time uniform into the vertex shader
							shader.vertexShader = `
								uniform float time;
								${shader.vertexShader}
							`.replace(
								`#include <begin_vertex>`,
								`#include <begin_vertex>
									float wave = sin(position.y * 3.0 + time * 2.0) * 0.1;
									transformed.x += wave;
									transformed.z += wave;
								`
							);

							// Update the shader material in the render loop
							node.material.userData.shader = shader;
						};

						node.material = originalMaterial;
					}
				});

				scene.add(gltf.scene);

				// Center the camera
				const box = new THREE.Box3().setFromObject(gltf.scene);
				const center = new THREE.Vector3();
				const size = new THREE.Vector3();
				box.getCenter(center);
				box.getSize(size);

				camera.position.set(center.x, center.y + size.y / 2, center.z + size.z * 2);
				camera.lookAt(center);
				controls.target.copy(center);
				controls.update();
			},
			undefined,
			(error) => {
				console.error('Error loading the model:', error);
			}
		);

		const onWindowResize = () => {
			camera.aspect = window.innerWidth / window.innerHeight;
			camera.updateProjectionMatrix();
			renderer.setSize(window.innerWidth, window.innerHeight);
			composer.setSize(window.innerWidth, window.innerHeight);
		};
		window.addEventListener('resize', onWindowResize);

		const animate = () => {
			requestAnimationFrame(animate);

			// Update time uniform for the animation
			const elapsedTime = clock.getElapsedTime();
			scene.traverse((node) => {
				if (node.isMesh && node.material.userData.shader) {
					node.material.userData.shader.uniforms.time.value = elapsedTime;
				}
			});

			controls.update(); // Update OrbitControls
			composer.render();
		};
		animate();

		return () => {
			window.removeEventListener('resize', onWindowResize);
			renderer.dispose();
			composer.dispose();
		};
	});
</script>

<canvas bind:this={canvas}></canvas>

<style>
	canvas {
		display: block;
		width: 100vw;
		height: 100vh;
	}
</style>
