<svelte:options customElement={{ tag: "spacestation-wgc", shadow: "open" }} />

<script>
  import { onMount } from "svelte";
  import * as THREE from "three";
  import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
  import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader.js";

  // --- HTML DOM Bindings ---
  let canvasEl;
  let dragZoneEl; 

  // --- Loading UI Performance State ---
  let isLoading = true;
  let loadingProgress = 0;

  // --- DOM Reference Cache ---
  let domCache = {
    dots: {}, labels: {}, lines: {},
    subDots: {}, subLabels: {}, subLines: {}
  };

  // --- Three.js Infrastructure Instances ---
  let scene, camera, renderer;
  let modelContainer, masterModelRef;
  let raycaster, pointer;
  let animationFrameId;
  const tempV3 = new THREE.Vector3();

  let containerWidth = 0;
  let containerHeight = 0;

  // --- Interaction & Animation State ---
  let currentMode = "wandering"; 
  let activeFocusId = null; 
  let targetRotation = { x: 0, y: 0, z: 0 };
  let isModelLoaded = false;
  let isDragging = false;
  let previousPointerPosition = { x: 0, y: 0 };
  let wanderTime = 0;
  let desiredScale = 1;

  let baseDistance = 75;
  let baseCameraY = 35;
  let targetCameraDistance = baseDistance;
  let cameraDistance = baseDistance;
  let targetCameraY = baseCameraY;
  let cameraY = baseCameraY;

  const defaultMasterPos = new THREE.Vector3();
  const targetMasterPosition = new THREE.Vector3();

  let labelsData = [
    {
      id: "tianhe",
      name: "Tianhe Core Module",
      x: 0, y: 0, dx: -220, dy: -140, rDx: -220, rDy: -140, visible: false,
      offset: new THREE.Vector3(0, 0, 0),
      clickable: true,
      viewRotX: 0.4, viewRotY: 1.8, viewRotZ: 0.1, viewY: 30, viewDist: 30,
      targetCollections: ["tianhe outter"],
      subAnnotations: [
        { id: "tianhe_docking", title: "Docking", type: "text", content: "Lorem ipsum...", offset: new THREE.Vector3(0, 0, -2.2), dx: -200, dy: -100, x: 0, y: 0, visible: false },
        { id: "tianhe_flexible_solar", title: "Flexible solar arrays", type: "text", content: "Lorem ipsum...", offset: new THREE.Vector3(2, 0, -0.5), dx: 200, dy: -200, x: 0, y: 0, visible: false },
        { id: "tianhe_robotic", title: "Robotic arms", type: "text", content: "Lorem ipsum...", offset: new THREE.Vector3(0, 0.5, 0.8), dx: 200, dy: -100, x: 0, y: 0, visible: false },
{
          id: "tianhe_living", title: "Astronaut habitats", type: "image",
          content: "https://multimedia.scmp.com/infographics/news/china/article/3355504/tiangong_interactive/images/habitant.png",
          offset: new THREE.Vector3(0, 0, -1.2), dx: -100, dy: 200, x: 0, y: 0, visible: false
        },
        {
          id: "tianhe_treadmill", // OPTIMIZATION: Fixed duplicate ID conflict with tianhe_living
          title: "Treadmill", type: "image",
          content: "https://multimedia.scmp.com/infographics/news/china/article/3355504/tiangong_interactive/images/treadmill.png",
          offset: new THREE.Vector3(-0.2, -0.1, -1.7), dx: -100, dy: -240, x: 0, y: 0, visible: false
        }
      ]
    },
    { id: "wentian", name: "Wentian Laboratory", x: 0, y: 0, dx: -260, dy: 60, rDx: -260, rDy: 60, visible: false, offset: new THREE.Vector3(-2, 0, 0.5), clickable: true, viewRotX: 0.4, viewRotY: -0.2, viewRotZ: 0, viewY: 26, viewDist: 26, targetCollections: ["wentian outter"], subAnnotations: [{ id: "wentian_solar", title: "Flexible Solar Arrays", type: "video", content: "https://...", offset: new THREE.Vector3(-3.5, 0, 0.5), dx: -200, dy: -140, x: 0, y: 0, visible: false }] },
    { id: "mengtian", name: "Mengtian Laboratory", x: 0, y: 0, dx: 260, dy: -60, rDx: 260, rDy: -60, visible: false, offset: new THREE.Vector3(2, 0, 0.5), clickable: true, viewRotX: 0.4, viewRotY: -0.2, viewRotZ: 0.1, viewY: 26, viewDist: 26, targetCollections: ["mengtian outter"], subAnnotations: [{ id: "mengtian_airlock", title: "Payload Airlock", type: "text", content: "Deploys...", offset: new THREE.Vector3(3.2, 0, 0.5), dx: 200, dy: 100, x: 0, y: 0, visible: false }] },
    { id: "shenzhou", name: "Shenzhou Spacecraft", x: 0, y: 0, dx: 220, dy: 140, rDx: 220, rDy: 140, visible: false, offset: new THREE.Vector3(0, 0, 2), clickable: false },
    { id: "shenzhou_bottom", name: "Shenzhou Spacecraft", x: 0, y: 0, dx: -180, dy: 200, rDx: -180, rDy: 200, visible: false, offset: new THREE.Vector3(0, -2, 0) },
    { id: "tianzhou", name: "Tianzhou Cargo Ship", x: 0, y: 0, dx: 220, dy: -160, rDx: 220, rDy: -160, visible: false, offset: new THREE.Vector3(0, 0, -3), clickable: false }
  ];

  $: focusedItem = labelsData.find(l => l.id === activeFocusId);

  function initDomCache() {
    const root = canvasEl.parentElement;
    labelsData.forEach(label => {
      domCache.dots[label.id] = root.querySelector(`[data-dot-id="${label.id}"]`);
      domCache.labels[label.id] = root.querySelector(`[data-label-id="${label.id}"]`);
      domCache.lines[label.id] = root.querySelector(`[data-line-id="${label.id}"]`);
      if (label.subAnnotations) {
        label.subAnnotations.forEach(sub => {
          domCache.subDots[sub.id] = root.querySelector(`[data-subdot-id="${sub.id}"]`);
          domCache.subLabels[sub.id] = root.querySelector(`[data-sublabel-id="${sub.id}"]`);
          domCache.subLines[sub.id] = root.querySelector(`[data-subline-id="${sub.id}"]`);
        });
      }
    });
  }

  function initThree() {
    scene = new THREE.Scene();
    // scene.background = new THREE.Color(0x000000);

    containerWidth = canvasEl.parentElement.clientWidth;
    containerHeight = canvasEl.parentElement.clientHeight;

    camera = new THREE.PerspectiveCamera(40, containerWidth / containerHeight, 1, 1000); 
    const isMobile = window.innerWidth < 768;
    baseDistance = isMobile ? 95 : 75;
    baseCameraY = isMobile ? 45 : 35;
    cameraDistance = baseDistance;
    targetCameraDistance = baseDistance;
    cameraY = baseCameraY;
    targetCameraY = baseCameraY;

    camera.position.set(0, cameraY, cameraDistance);
    camera.lookAt(0, 0, 0);

    renderer = new THREE.WebGLRenderer({
      canvas: canvasEl,
      antialias: !isMobile,
      alpha: true,
      powerPreference: "high-performance"
    });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.setSize(containerWidth, containerHeight);

    const ambientLight = new THREE.AmbientLight(0xdbe9ff, 0.8); 
    scene.add(ambientLight);
    const directionalLight = new THREE.DirectionalLight(0xfff3e3, 5.0); 
    directionalLight.position.set(30, 50, 60);
    scene.add(directionalLight);

    modelContainer = new THREE.Group();
    scene.add(modelContainer);

    raycaster = new THREE.Raycaster();
    pointer = new THREE.Vector2();
  }

  function loadModel() {
    const originalUrl = "https://multimedia.scmp.com/infographics/news/china/article/3355504/tiangong_interactive/model/tiangong_model.glb";
    const isDev = import.meta.env.DEV;
    const modelUrl = isDev ? originalUrl.replace("https://multimedia.scmp.com", "/api/multimedia") : originalUrl;

    const loader = new GLTFLoader();
    const dracoLoader = new DRACOLoader();
    dracoLoader.setDecoderPath("https://www.gstatic.com/draco/versioned/decoders/1.5.6/");
    loader.setDRACOLoader(dracoLoader);

    loader.load(
      modelUrl,
      (gltf) => {
        masterModelRef = gltf.scene;
        masterModelRef.traverse((child) => {
          child.frustumCulled = true; 
          if (child.isMesh && child.material) {
            const materials = Array.isArray(child.material) ? child.material : [child.material];
            materials.forEach((mat) => {
              child.userData.originalMaterial = child.material;
              if (mat.transparent) {
                mat.alphaTest = 0.1;
                mat.depthWrite = true;
              }
            });
          }
        });

        const box = new THREE.Box3().setFromObject(masterModelRef);
        const size = box.getSize(new THREE.Vector3());
        const maxDim = Math.max(size.x, size.y, size.z);
        desiredScale = 30 / maxDim;
        masterModelRef.scale.set(desiredScale, desiredScale, desiredScale);

        const scaledBox = new THREE.Box3().setFromObject(masterModelRef);
        const scaledCenter = scaledBox.getCenter(new THREE.Vector3());
        
        defaultMasterPos.set(-scaledCenter.x, -scaledCenter.y, -scaledCenter.z);
        targetMasterPosition.copy(defaultMasterPos);
        masterModelRef.position.copy(defaultMasterPos);

        labelsData.forEach((label) => {
          const anchorObj = new THREE.Object3D();
          anchorObj.position.copy(label.offset);
          masterModelRef.add(anchorObj);
          label.anchor = anchorObj;

          if (label.subAnnotations) {
            label.subAnnotations.forEach((sub) => {
              const subAnchor = new THREE.Object3D();
              subAnchor.position.copy(sub.offset);
              masterModelRef.add(subAnchor);
              sub.anchor = subAnchor;
            });
          }
        });

        modelContainer.add(masterModelRef);
        isModelLoaded = true;
        isLoading = false; 
        dracoLoader.dispose();
      },
      (xhr) => {
        if (xhr.total > 0) {
          loadingProgress = Math.round((xhr.loaded / xhr.total) * 100);
        }
      },
      (error) => console.error("An error occurred while loading the model:", error)
    );
  }

  function applyFocusVisualEffects(targetLabel) {
    if (!masterModelRef) return;

    masterModelRef.traverse((child) => {
      if (child.isMesh && child.userData.originalMaterial) {
        child.material = child.userData.originalMaterial;
      }
    });

    if (!targetLabel || targetLabel.id !== "tianhe") return;

    const hideTargets = targetLabel.targetCollections 
      ? targetLabel.targetCollections.map(name => name.toLowerCase().replace(/[\s_]/g, "")) 
      : [];
    const highlightKeyword = targetLabel.id.toLowerCase().replace(/[\s_]/g, "");

    masterModelRef.traverse((child) => {
      if (child.isMesh && child.material) {
        let belongsToHideTarget = false;
        let belongsToHighlightTarget = false;
        let currentElement = child;

        while (currentElement) {
          if (currentElement.name) {
            const cleanName = currentElement.name.toLowerCase().replace(/[\s_]/g, "");
            if (hideTargets.some(t => cleanName.includes(t))) belongsToHideTarget = true;
            if (highlightKeyword && cleanName.includes(highlightKeyword)) belongsToHighlightTarget = true;
          }
          currentElement = currentElement.parent;
        }

        if (belongsToHideTarget || belongsToHighlightTarget) {
          const isolateAndApply = (mat) => {
            const clonedMat = mat.clone();
            if (belongsToHideTarget) {
              clonedMat.transparent = true;
              clonedMat.opacity = 0.15; 
              clonedMat.depthWrite = false;
            }
            if (belongsToHighlightTarget && clonedMat.emissive) {
              clonedMat.emissive.setHex(0x42ffef); 
            }
            clonedMat.needsUpdate = true;
            return clonedMat;
          };

          child.material = Array.isArray(child.material) 
            ? child.material.map(isolateAndApply) 
            : isolateAndApply(child.material);
        }
      }
    });
  }

  function updateLabels() {
    const isMobile = window.innerWidth < 768;
    const hudScale = isMobile ? 0.55 : 1.0;

    for (let i = 0; i < labelsData.length; i++) {
      const label = labelsData[i];
      if (!label.anchor) continue;

      label.anchor.getWorldPosition(tempV3);
      tempV3.project(camera);

      const x = (tempV3.x * 0.5 + 0.5) * containerWidth;
      const y = (-(tempV3.y * 0.5) + 0.5) * containerHeight;
      const isVisible = tempV3.z <= 1 && (currentMode === "wandering" || currentMode === "manual");
      const rDx = label.dx * hudScale;
      const rDy = label.dy * hudScale;

      const dotEl = domCache.dots[label.id];
      const labelEl = domCache.labels[label.id];
      const lineEl = domCache.lines[label.id];

      if (isVisible) {
        if (dotEl) {
          dotEl.style.display = "block";
          dotEl.style.transform = `translate3d(${x}px, ${y}px, 0) translate(-50%, -50%)`;
        }
        if (labelEl) {
          labelEl.style.display = "block";
          labelEl.style.transform = `translate3d(${x + rDx}px, ${y + rDy}px, 0) translate(-50%, -50%)`;
        }
        if (lineEl) {
          lineEl.style.display = "block";
          lineEl.setAttribute("x1", x);
          lineEl.setAttribute("y1", y);
          lineEl.setAttribute("x2", x + rDx);
          lineEl.setAttribute("y2", y + rDy);
        }
      } else {
        if (dotEl) dotEl.style.display = "none";
        if (labelEl) labelEl.style.display = "none";
        if (lineEl) lineEl.style.display = "none";
      }

      if (label.subAnnotations) {
        const isCurrentFocused = currentMode === "focused" && activeFocusId === label.id;
        
        for (let j = 0; j < label.subAnnotations.length; j++) {
          const sub = label.subAnnotations[j];
          const subDotEl = domCache.subDots[sub.id];
          const subLabelEl = domCache.subLabels[sub.id];
          const subLineEl = domCache.subLines[sub.id];

          if (isCurrentFocused && sub.anchor) {
            sub.anchor.getWorldPosition(tempV3);
            tempV3.project(camera);
            
            const sx = (tempV3.x * 0.5 + 0.5) * containerWidth;
            const sy = (-(tempV3.y * 0.5) + 0.5) * containerHeight;
            const srDx = sub.dx * hudScale;
            const srDy = sub.dy * hudScale;
            const sVisible = tempV3.z <= 1;

            if (sVisible) {
              if (subDotEl) {
                subDotEl.style.display = "block";
                subDotEl.style.transform = `translate3d(${sx}px, ${sy}px, 0) translate(-50%, -50%)`;
              }
              if (subLabelEl) {
                subLabelEl.style.display = "block";
                subLabelEl.style.transform = `translate3d(${sx + srDx}px, ${sy + srDy}px, 0) translate(-50%, -50%)`;
              }
              if (subLineEl) {
                subLineEl.style.display = "block";
                subLineEl.setAttribute("x1", sx);
                subLineEl.setAttribute("y1", sy);
                subLineEl.setAttribute("x2", sx + srDx);
                subLineEl.setAttribute("y2", sy + srDy);
              }
            } else {
              if (subDotEl) subDotEl.style.display = "none";
              if (subLabelEl) subLabelEl.style.display = "none";
              if (subLineEl) subLineEl.style.display = "none";
            }
          } else {
            if (subDotEl) subDotEl.style.display = "none";
            if (subLabelEl) subLabelEl.style.display = "none";
            if (subLineEl) subLineEl.style.display = "none";
          }
        }
      }
    }
  }

  function animate() {
    animationFrameId = requestAnimationFrame(animate);

    if (isModelLoaded) {
      if (!isDragging && (currentMode === "wandering" || currentMode === "manual")) {
        wanderTime += 0.002; 
        modelContainer.rotation.y += 0.001 + Math.sin(wanderTime * 0.5) * 0.0003;
        modelContainer.rotation.x += Math.cos(wanderTime * 0.3) * 0.00008;
        modelContainer.rotation.z += Math.sin(wanderTime * 0.2) * 0.00008;

      } else if (currentMode === "focusing" || currentMode === "returning") {
        const speed = 0.06;

        modelContainer.rotation.x = THREE.MathUtils.lerp(modelContainer.rotation.x, targetRotation.x, speed);
        modelContainer.rotation.y = THREE.MathUtils.lerp(modelContainer.rotation.y, targetRotation.y, speed);
        modelContainer.rotation.z = THREE.MathUtils.lerp(modelContainer.rotation.z, targetRotation.z, speed);
        
        cameraY = THREE.MathUtils.lerp(cameraY, targetCameraY, speed);
        cameraDistance = THREE.MathUtils.lerp(cameraDistance, targetCameraDistance, speed);
        
        camera.position.set(0, cameraY, cameraDistance);
        camera.lookAt(0, 0, 0);
        masterModelRef.position.lerp(targetMasterPosition, speed);

        const rotDiff = Math.abs(modelContainer.rotation.y - targetRotation.y);
        const distDiff = Math.abs(cameraDistance - targetCameraDistance);
        if (rotDiff < 0.005 && distDiff < 0.05) {
          currentMode = currentMode === "focusing" ? "focused" : "wandering";
        }
      } else {
        masterModelRef.position.lerp(targetMasterPosition, 0.1);
      }

      updateLabels();
    }
    renderer.render(scene, camera);
  }

  // FIXED: Explicit function declarations to guarantee scope availability inside the template
  function focusOnModule(labelId) {
    const targetLabel = labelsData.find((l) => l.id === labelId);
    if (!targetLabel || !isModelLoaded || !targetLabel.clickable) return;
    
    activeFocusId = labelId; 
    targetMasterPosition.copy(targetLabel.offset).multiplyScalar(desiredScale).negate();

    targetRotation.x = targetLabel.viewRotX;
    targetRotation.y = targetLabel.viewRotY;
    targetRotation.z = targetLabel.viewRotZ || 0;
    
    const isMobile = window.innerWidth < 768;
    const mobileDistanceModifier = isMobile ? 1.8 : 1.0;

    targetCameraY = (targetLabel.viewY || 18) * mobileDistanceModifier;
    targetCameraDistance = (targetLabel.viewDist || 18) * mobileDistanceModifier;

    applyFocusVisualEffects(targetLabel);
    currentMode = "focusing";
  }
  // Expose to window fallback context if externally required outside shadow DOM bounds
  window.focusOnModule = focusOnModule;

  function triggerGoBack() {
    activeFocusId = null; 
    targetMasterPosition.copy(defaultMasterPos);
    
    const isMobile = window.innerWidth < 768;
    targetCameraY = isMobile ? 45 : 30;
    targetCameraDistance = isMobile ? 95 : 65;

    applyFocusVisualEffects(null);
    currentMode = "returning";
  }

  const onPointerDown = (event) => {
    if (!isModelLoaded || currentMode === "focusing" || currentMode === "returning") return;

    const rect = canvasEl.getBoundingClientRect();
    pointer.x = ((event.clientX - rect.left) / containerWidth) * 2 - 1;
    pointer.y = -((event.clientY - rect.top) / containerHeight) * 2 + 1;
    
    raycaster.setFromCamera(pointer, camera);

    const intersects = raycaster.intersectObjects(modelContainer.children, true);
    if (intersects.length > 0) {
      if (currentMode === "wandering") currentMode = "manual";
      isDragging = true;
      previousPointerPosition = { x: event.clientX, y: event.clientY };
    }
  };

  const onPointerMove = (event) => {
    if (!isDragging) return;
    const deltaX = event.clientX - previousPointerPosition.x;
    const deltaY = event.clientY - previousPointerPosition.y;
    modelContainer.rotation.y += deltaX * 0.005;
    modelContainer.rotation.x += deltaY * 0.005;
    previousPointerPosition = { x: event.clientX, y: event.clientY };
  };

  const onPointerUp = () => {
    isDragging = false;
    if (currentMode === "manual") currentMode = "wandering";
  };

  const handleResize = () => {
    containerWidth = canvasEl.parentElement.clientWidth;
    containerHeight = canvasEl.parentElement.clientHeight;

    const isMobile = window.innerWidth < 768;
    baseDistance = isMobile ? 95 : 65;
    baseCameraY = isMobile ? 45 : 30;

    if (currentMode === "wandering" || currentMode === "manual" || currentMode === "returning") {
      targetCameraDistance = baseDistance;
      targetCameraY = baseCameraY;
    } else if (currentMode === "focused" && focusedItem) {
      const mobileDistanceModifier = isMobile ? 1.8 : 1.0;
      targetCameraY = (focusedItem.viewY || 18) * mobileDistanceModifier;
      targetCameraDistance = (focusedItem.viewDist || 18) * mobileDistanceModifier;
    }

    camera.aspect = containerWidth / containerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(containerWidth, containerHeight);
  };

  function setupEventListeners() {
    if (dragZoneEl) {
      dragZoneEl.addEventListener("pointerdown", onPointerDown);
    }
    window.addEventListener("pointermove", onPointerMove, { passive: true });
    window.addEventListener("pointerup", onPointerUp, { passive: true });
    window.addEventListener("resize", handleResize, { passive: true });
  }

  function cleanupThree() {
    window.removeEventListener("resize", handleResize);
    if (dragZoneEl) {
      dragZoneEl.removeEventListener("pointerdown", onPointerDown);
    }
    window.removeEventListener("pointermove", onPointerMove);
    window.removeEventListener("pointerup", onPointerUp);
    cancelAnimationFrame(animationFrameId);
    if (renderer) renderer.dispose();
  }

  onMount(() => {
    initThree();
    loadModel();
    initDomCache(); 
    setupEventListeners();
    animate();

    return cleanupThree;
  });
</script>

<section class="modelCtn">
  {#if isLoading}
    <div class="loaderOverlay">
      <div class="loaderSpinner"></div>
      <div class="loaderText">Loading Tiangong Model {loadingProgress}%</div>
      <div class="loaderBarContainer">
        <div class="loaderBar" style="width: {loadingProgress}%"></div>
      </div>
    </div>
  {/if}

  <canvas class="model" bind:this={canvasEl}></canvas>

  <div class="dragZone" bind:this={dragZoneEl}>
    <div class="mobileHint">Swipe here to rotate space station</div>
  </div>

  <svg class="hudOverlay">
    {#each labelsData as label}
      <line data-line-id={label.id} class="connectionLine" style="display: none;" />
    {/each}

    {#each labelsData as label}
      {#if label.subAnnotations}
        {#each label.subAnnotations as sub}
          <line data-subline-id={sub.id} class="connectionLine focusLine" style="display: none;" />
        {/each}
      {/if}
    {/each}
  </svg>

  {#each labelsData as label}
    <div data-dot-id={label.id} class="indicatorDot" style="display: none;"></div>
    <div data-label-id={label.id} class="spaceLabel" style="display: none;">
      <button class="textBox {label.clickable ? 'clickable' : 'static'}" on:click={() => { if (label.clickable) focusOnModule(label.id); }}>
        {label.name}
      </button>
    </div>
  {/each}

  {#each labelsData as label}
    {#if label.subAnnotations}
      {#each label.subAnnotations as sub}
        <div data-subdot-id={sub.id} class="indicatorDot subDot" style="display: none;"></div>
        <div data-sublabel-id={sub.id} class="spaceLabel mediaCard" style="display: none;">
          <div class="cardHeader">{sub.title}</div>
          <div class="cardContent">
            {#if sub.type === 'text'}
              <p class="bodyText">{sub.content}</p>
            {:else if sub.type === 'image'}
              <img src={sub.content} alt={sub.title} class="mediaElement" />
            {:else if sub.type === 'video'}
              <video src={sub.content} autoplay loop muted playsinline class="mediaElement"></video>
            {/if}
          </div>
        </div>
      {/each}
    {/if}
  {/each}

  {#if currentMode === "focused"}
    <button class="backBtn" on:click={triggerGoBack}>
      ← Back to overview
    </button>
  {/if}
</section>

<style>
  :global(html), :global(body) {
    overflow-x: hidden;
    overflow-y: auto !important;
    width: 100%;
    margin: 0;
    padding: 0;
  }
  :global(body) {
    min-width: 290px;
    -webkit-tap-highlight-color: transparent;
    -webkit-touch-callout: none;
    -webkit-font-smoothing: antialiased;
    font-family: "Barlow Semi Condensed", sans-serif;
    /* background-color: #000000; */
  }
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: "Barlow Semi Condensed", sans-serif;
  }
  
  .modelCtn {
    position: relative;
    width: 100%;
    height: 90vh; 
    /* background-color: #000000; */
    overflow: hidden;
  }

  .loaderOverlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #000000;
    z-index: 200;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    gap: 16px;
  }
  .loaderSpinner {
    width: 40px;
    height: 40px;
    border: 3px solid rgba(255, 255, 255, 0.1);
    border-top-color: #00d2ff;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }
  .loaderText {
    color: #ffffff;
    font-size: 14px;
    text-transform: uppercase;
    letter-spacing: 1.5px;
  }
  .loaderBarContainer {
    width: 200px;
    height: 4px;
    background: rgba(255, 255, 255, 0.1);
    border-radius: 2px;
    overflow: hidden;
  }
  .loaderBar {
    height: 100%;
    background: #00d2ff;
    transition: width 0.1s ease-out;
  }
  @keyframes spin {
    to { transform: rotate(360deg); }
  }
  
  .model {
    display: block;
    width: 100%;
    height: 100%;
  }

  .dragZone {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 2; 
    cursor: grab;
    touch-action: none; 
  }
  .dragZone:active {
    cursor: grabbing;
  }

  .mobileHint {
    display: none;
  }

  @media (max-width: 767px) {
    .dragZone {
      top: 12%;
      height: 76%;
      left: 5%;
      width: 90%;
      border: 1px dashed rgba(255, 255, 255, 0.15);
      border-radius: 8px;
    }
    .mobileHint {
      display: block;
      position: absolute;
      top: -24px;
      left: 50%;
      transform: translateX(-50%);
      color: rgba(255, 255, 255, 0.4);
      font-size: 11px;
      text-transform: uppercase;
      letter-spacing: 1px;
      white-space: nowrap;
      pointer-events: none;
    }
  }

  .hudOverlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 5;
  }
  .connectionLine {
    stroke: #ffffff;
    stroke-width: 1px;
    opacity: 0.6;
    stroke-dasharray: 1 2;
  }
  .connectionLine.focusLine {
    stroke: #00ffaa; 
    stroke-width: 1px;
  }

  .spaceLabel {
    position: absolute;
    top: 0;
    left: 0;
    pointer-events: auto; 
    z-index: 10;
    will-change: transform;
  }
  .indicatorDot {
    position: absolute;
    top: 0;
    left: 0;
    width: 6px;
    height: 6px;
    background-color: #00d2ff;
    border-radius: 50%;
    box-shadow: 0 0 8px #00d2ff;
    z-index: 10;
    pointer-events: none;
    will-change: transform;
  }
  .indicatorDot.subDot {
    background-color: #00d2ff;
    box-shadow: 0 0 8px #00d2ff;
  }
  .textBox {
    color: #ffffff;
    padding: 8px 10px;
    font-size: 12px;
    font-weight: 200;
    text-transform: uppercase;
    white-space: nowrap;
    background: rgba(10, 10, 10, 0.5);
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 4px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.5);
    backdrop-filter: blur(4px);
    transition: transform 0.2s ease, background 0.2s ease, border-color 0.2s ease;
  }
  @media (max-width: 767px) {
    .textBox {
      font-size: 12px;
      padding: 6px 8px;
    }
  }
  .textBox.clickable {
    cursor: pointer;
  }
  .textBox.clickable:hover {
    border-color: #00d2ff;
    background: #00d2ff;
    color: #000000;
    box-shadow: 0 0 15px rgba(0, 210, 255, 0.6);
  }
  .textBox.static {
    color: rgba(255, 255, 255, 0.724);
  }
  .mediaCard {
    width: 240px;
    max-width: 65vw; 
    background: rgba(0, 0, 0, 0.18);
    border: 1px solid #ffffff28;
    border-radius: 6px;
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.7);
    backdrop-filter: blur(8px);
    overflow: hidden;
    color: #ffffff;
  }

  .cardHeader {
    background: rgba(0, 255, 170, 0.15);
    font-size: 14px;
    font-weight: 600;
    text-transform: uppercase;
    border-bottom: 1px solid rgba(0, 255, 170, 0.2);
    color: #ffffff;
  }

  .bodyText {
    font-size: 12px;
    color: rgb(255, 255, 255);
  }

  .mediaElement {
    display: block;
    width: 100%;
    height: 120px;
    object-fit: cover;
    border-radius: 4px;
    background: #050505;
  }

  .backBtn {
    position: absolute;
    bottom: 40px;
    left: 50%;
    transform: translateX(-50%);
    color: white;
    background: rgba(10, 10, 10, 0.4);
    border: 1px solid rgba(255, 255, 255, 0.5);
    border-radius: 4px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.5);
    backdrop-filter: blur(4px);
    transition: transform 0.2s ease, background 0.2s ease;
    padding: 12px 24px;
    font-size: 14px;
    cursor: pointer;
    z-index: 100;
    white-space: nowrap;
  }
  @media (max-width: 767px) {
    .backBtn {
      padding: 10px 18px;
      font-size: 12px;
      bottom: 25px;
    }
  }
  .backBtn:hover {
    border-color: #00d2ff;
    background: #00d2ff;
    color: #000000;
    box-shadow: 0 0 15px rgba(0, 210, 255, 0.6);
  }
</style>