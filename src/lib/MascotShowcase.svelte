<script lang="ts">
  import { onMount } from "svelte";

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
        let autoYaw = 0;

        const animate = (now: number) => {
          if (disposed || !renderer) return;
          const dt = Math.min(0.1, (now - lastTime) / 1000);
          lastTime = now;

          // Smooth slow turntable rotation (no bouncing)
          if (!isDragging) {
            autoYaw += dt * 0.4;
          }

          pivot.rotation.y = autoYaw + userYawOffset;
          pivot.position.y = 0;
          pivot.rotation.x = 0.02;
          pivot.rotation.z = 0;
          pivot.scale.set(1, 1, 1);

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
    min-height: 260px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: grab;
    user-select: none;
    touch-action: pan-y;
  }

  @media (max-width: 560px) {
    .mascot-stage {
      min-height: 170px;
      height: 170px;
    }
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
