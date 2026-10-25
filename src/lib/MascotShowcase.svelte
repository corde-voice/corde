<script lang="ts">
  import { onMount } from "svelte";

  export let mode: "idle" | "greeting" | "listening" | "thinking" | "sleeping" = "idle";
  export let pulseTrigger = 0;

  let canvas: HTMLCanvasElement;
  let loaded = false;
  let loadError = false;
  let isDragging = false;
  let dragStartX = 0;
  let userYawOffset = 0;

  function onPointerDown(e: PointerEvent) {
    isDragging = true;
    dragStartX = e.clientX;
    canvas?.setPointerCapture?.(e.pointerId);
  }

  function onPointerMove(e: PointerEvent) {
    if (!isDragging) return;
    const dx = e.clientX - dragStartX;
    dragStartX = e.clientX;
    userYawOffset += dx * 0.012;
  }

  function onPointerUp(e: PointerEvent) {
    isDragging = false;
    canvas?.releasePointerCapture?.(e.pointerId);
  }

  onMount(() => {
    let disposed = false;
    let animFrameId: number | undefined;
    let observer: ResizeObserver | undefined;
    let renderer: import("three").WebGLRenderer | undefined;

    async function mountMascot() {
      try {
        const [THREE, { GLTFLoader }, { MeshoptDecoder }] = await Promise.all([
          import("three"),
          import("three/addons/loaders/GLTFLoader.js"),
          import("three/addons/libs/meshopt_decoder.module.js"),
        ]);
        if (disposed) return;

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(30, 1, 0.1, 30);
        camera.position.set(0, 0.65, 2.55);

        // Balanced studio lighting for the denoised PBR mascot texture
        scene.add(new THREE.HemisphereLight(0xe4ecff, 0x262238, 2.35));
        const keyLight = new THREE.DirectionalLight(0xffffff, 2.15);
        keyLight.position.set(-1.6, 2.5, 3.2);
        scene.add(keyLight);

        const rimLight = new THREE.DirectionalLight(0x9b8cff, 1.55);
        rimLight.position.set(2.2, 1.1, -2.2);
        scene.add(rimLight);

        const bottomFill = new THREE.DirectionalLight(0x5be09a, 0.55);
        bottomFill.position.set(0, -2.0, 1.5);
        scene.add(bottomFill);

        renderer = new THREE.WebGLRenderer({
          canvas,
          alpha: true,
          antialias: true,
          powerPreference: "high-performance",
        });
        renderer.outputColorSpace = THREE.SRGBColorSpace;
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 1.06;
        renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));

        const loader = new GLTFLoader();
        await MeshoptDecoder.ready;
        loader.setMeshoptDecoder(MeshoptDecoder);
        const gltf = await loader.loadAsync("/mascot/corde-mascot.glb");

        if (disposed) {
          gltf.scene.traverse((object) => {
            if (object instanceof THREE.Mesh) {
              object.geometry.dispose();
            }
          });
          return;
        }

        const model = gltf.scene;
        const maxAnisotropy = renderer.capabilities.getMaxAnisotropy?.() || 4;

        // Apply texture denoising (clamp KHR_materials_specular & normal spikes on black visor/chest)
        model.traverse((object) => {
          if (object instanceof THREE.Mesh) {
            const materials = Array.isArray(object.material) ? object.material : [object.material];
            for (const mat of materials) {
              if (!mat) continue;
              if ("map" in mat && mat.map) {
                mat.map.colorSpace = THREE.SRGBColorSpace;
                mat.map.anisotropy = Math.min(8, maxAnisotropy);
                mat.map.generateMipmaps = false;
                mat.map.minFilter = THREE.LinearFilter;
                mat.map.magFilter = THREE.LinearFilter;
                mat.map.needsUpdate = true;
              }
              if ("specularIntensity" in mat) {
                (mat as any).specularIntensity = 0.24;
              }
              if ("specularColor" in mat && (mat as any).specularColor?.setRGB) {
                (mat as any).specularColor.setRGB(0.55, 0.55, 0.68);
              }
              if ("normalScale" in mat && (mat as any).normalScale?.set) {
                (mat as any).normalScale.set(0.22, 0.22);
              }
              if ("roughness" in mat && typeof (mat as any).roughness === "number") {
                (mat as any).roughness = Math.max((mat as any).roughness, 0.62);
              }
              mat.needsUpdate = true;
            }
          }
        });

        const bounds = new THREE.Box3().setFromObject(model);
        const center = bounds.getCenter(new THREE.Vector3());
        model.position.sub(center);

        const pivot = new THREE.Group();
        pivot.add(model);
        scene.add(pivot);

        const size = new THREE.Vector3();
        bounds.getSize(size);
        camera.position.set(0, size.y * 0.04, Math.max(size.y * 2.0, 1.88));
        camera.lookAt(0, 0, 0);
        camera.updateProjectionMatrix();

        const updateSize = () => {
          if (!renderer || !canvas) return;
          const width = Math.max(1, canvas.clientWidth);
          const height = Math.max(1, canvas.clientHeight);
          renderer.setSize(width, height, false);
          camera.aspect = width / height;
          camera.updateProjectionMatrix();
        };

        observer = new ResizeObserver(updateSize);
        observer.observe(canvas);
        updateSize();
        loaded = true;

        let lastTime = performance.now();
        let lastPulseSeen = pulseTrigger;
        let pulseBoost = 0;

        const animate = (now: number) => {
          if (disposed || !renderer) return;
          const dt = Math.min(0.1, (now - lastTime) / 1000);
          lastTime = now;
          const t = now * 0.001;

          if (pulseTrigger !== lastPulseSeen) {
            lastPulseSeen = pulseTrigger;
            pulseBoost = 1.0;
          }
          pulseBoost = Math.max(0, pulseBoost - dt * 1.8);

          // Gently decay manual drag rotation back toward front when not dragging
          if (!isDragging) {
            userYawOffset *= Math.pow(0.92, dt * 60);
          }

          let targetY = 0;
          let targetRotX = 0;
          let targetRotY = userYawOffset;
          let targetRotZ = 0;
          let scaleX = 1;
          let scaleY = 1;
          let scaleZ = 1;

          if (mode === "sleeping") {
            targetY = Math.sin(t * 1.3) * 0.02 - 0.03;
            targetRotX = 0.16 + Math.sin(t * 1.3) * 0.025;
            targetRotY += Math.sin(t * 0.65) * 0.07;
            targetRotZ = -0.055;
          } else if (mode === "greeting") {
            targetY = Math.abs(Math.sin(t * 8.5)) * 0.055 + Math.sin(t * 2.5) * 0.02;
            targetRotX = Math.sin(t * 10.0) * 0.075 - 0.04;
            targetRotY += Math.sin(t * 3.6) * 0.25;
            targetRotZ = Math.cos(t * 5.4) * 0.085;
            const talkSquash = Math.sin(t * 14.0) * 0.042;
            scaleY = 1 + talkSquash;
            scaleX = 1 - talkSquash * 0.48;
            scaleZ = 1 - talkSquash * 0.48;
          } else if (mode === "listening") {
            const fakeVoice = 0.45 + 0.45 * Math.abs(Math.sin(t * 6.8) * Math.cos(t * 3.7));
            targetY = Math.sin(t * 4.2) * 0.03 + fakeVoice * 0.055;
            targetRotX = -0.06 - fakeVoice * 0.08 + Math.sin(t * 5.5) * 0.025;
            targetRotY += Math.sin(t * 2.4) * 0.18;
            targetRotZ = Math.sin(t * 4.2) * (0.04 + fakeVoice * 0.06);
            const voiceSquash = fakeVoice * 0.065;
            scaleY = 1 + voiceSquash;
            scaleX = 1 - voiceSquash * 0.42;
            scaleZ = 1 - voiceSquash * 0.42;
          } else if (mode === "thinking") {
            targetY = Math.sin(t * 3.4) * 0.035;
            targetRotX = -0.05 + Math.sin(t * 4.8) * 0.065;
            targetRotY += Math.sin(t * 1.9) * 0.26;
            targetRotZ = Math.sin(t * 2.7) * 0.06;
          } else {
            // Lively floating idle
            targetY = Math.sin(t * 2.1) * 0.036;
            targetRotY += Math.sin(t * 1.1) * 0.22 + Math.sin(t * 0.45) * 0.08;
            targetRotX = Math.sin(t * 1.6) * 0.045;
            targetRotZ = Math.cos(t * 1.35) * 0.04;
            const breathe = Math.sin(t * 2.1) * 0.018;
            scaleY = 1 + breathe;
            scaleX = 1 - breathe * 0.4;
            scaleZ = 1 - breathe * 0.4;
          }

          if (pulseBoost > 0.01) {
            targetY += Math.abs(Math.sin(pulseBoost * Math.PI * 2)) * 0.075 * pulseBoost;
            targetRotZ += Math.sin(pulseBoost * Math.PI * 4) * 0.12 * pulseBoost;
          }

          const lerp = Math.min(1, dt * 10);
          pivot.position.y += (targetY - pivot.position.y) * lerp;
          pivot.rotation.x += (targetRotX - pivot.rotation.x) * lerp;
          pivot.rotation.y += (targetRotY - pivot.rotation.y) * lerp;
          pivot.rotation.z += (targetRotZ - pivot.rotation.z) * lerp;
          pivot.scale.x += (scaleX - pivot.scale.x) * lerp;
          pivot.scale.y += (scaleY - pivot.scale.y) * lerp;
          pivot.scale.z += (scaleZ - pivot.scale.z) * lerp;

          renderer.render(scene, camera);
          animFrameId = requestAnimationFrame(animate);
        };

        animFrameId = requestAnimationFrame(animate);
      } catch (err) {
        console.error("Failed to load 3D mascot:", err);
        if (!disposed) loadError = true;
      }
    }

    void mountMascot();

    return () => {
      disposed = true;
      if (animFrameId !== undefined) cancelAnimationFrame(animFrameId);
      observer?.disconnect();
      renderer?.dispose();
    };
  });
</script>

<div class="mascot-stage" class:dragging={isDragging}>
  {#if !loaded && !loadError}
    <div class="mascot-loading" aria-live="polite">
      <span class="mascot-spinner"></span>
      <small>Loading 3D Buddy…</small>
    </div>
  {/if}
  <canvas
    bind:this={canvas}
    class="mascot-canvas"
    class:ready={loaded}
    onpointerdown={onPointerDown}
    onpointermove={onPointerMove}
    onpointerup={onPointerUp}
    onpointercancel={onPointerUp}
    aria-label="Interactive 3D Cordė mascot — drag to rotate or click states below"
  ></canvas>
</div>

<style>
  .mascot-stage {
    position: relative;
    width: 100%;
    height: 100%;
    min-height: 280px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: grab;
    user-select: none;
    touch-action: pan-y;
  }

  .mascot-stage.dragging {
    cursor: grabbing;
  }

  .mascot-canvas {
    width: 100%;
    height: 100%;
    display: block;
    opacity: 0;
    transition: opacity 0.45s ease;
  }

  .mascot-canvas.ready {
    opacity: 1;
  }

  .mascot-loading {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 10px;
    color: rgba(255, 255, 255, 0.65);
    font-size: 12px;
    pointer-events: none;
  }

  .mascot-spinner {
    width: 28px;
    height: 28px;
    border-radius: 50%;
    border: 2px solid rgba(143, 131, 255, 0.25);
    border-top-color: #9b8cff;
    animation: spin3d 0.85s linear infinite;
  }

  @keyframes spin3d {
    to {
      transform: rotate(360deg);
    }
  }
</style>
