<svelte:options customElement={{ tag: "lantern-wc", shadow: "open" }} />

<script>
  import { onMount } from "svelte";
  import * as THREE from "three";
  import { gsap } from "gsap";
  import { ScrollTrigger } from "gsap/ScrollTrigger";

  const browser = typeof window !== "undefined";

  export let foregroundSrc = "foregroundImg.png";
  export let backgroundSrc = [
    'static/1.png', 'static/2.png', 'static/3.png', 'static/4.png',
    'static/5.png', 'static/6.png', 'static/7.png', 'static/8.png',
    'static/9.png', 'static/10.png', 'static/11.png', 'static/12.png',
    'static/13.png'
  ];
  export let textSrc = [
    "Text for Image 1", "Text for Image 2", "Text for Image 3", "Text for Image 4",
    "Text for Image 5", "Text for Image 6", "Text for Image 7", "Text for Image 8",
    "Text for Image 9", "Text for Image 10", "Text for Image 11", "Text for Image 12",
    "Text for Image 13"
  ];
  
  export let scrollspeed = 1;
  export let screenHeight = 1200;

  let stickyWrapper, stickyInner, canvasContainer, candleOverlay;
  let scene, camera, renderer, bgGroup, fgMesh, candleLight, loader;
  let animationId;
  let ctx;
  let isInitialized = false;
  let observer;

  // State
  let userScrolled = false;
  let autoRotationAngle = 0; 
  let activeIndex = 0; 
  let isClockMode = false;
  let textVisible = true; // Control text visibility during transitions

  let prevForegroundSrc = foregroundSrc;
  let prevBackgroundSrc = backgroundSrc;
  const scrollProgress = { value: 0 };

  const parseInput = (input, maxLen = 13) => {
    if (!input) return [];
    let items = [];
    if (Array.isArray(input)) {
      items = input;
    } else if (typeof input === "string") {
      try {
        const parsed = JSON.parse(input);
        if (Array.isArray(parsed)) items = parsed;
        else items = [input];
      } catch (e) {
        items = input.split(",").map((s) => s.trim()).filter((s) => s.length > 0);
      }
    }
    if (items.length === 0) return [];
    const result = [];
    for (let i = 0; i < maxLen; i++) {
      result.push(items[i % items.length]);
    }
    return result;
  };

  const loadTexture = (url, isPanel = false) => {
    if (!loader || !url || typeof url !== "string") return null;
    const tex = loader.load(url);
    tex.colorSpace = THREE.SRGBColorSpace;
    if (isPanel) {
      tex.wrapS = THREE.ClampToEdgeWrapping;
      tex.wrapT = THREE.ClampToEdgeWrapping;
      tex.repeat.set(1, 1);
    } else {
      tex.wrapS = THREE.RepeatWrapping;
      tex.wrapT = THREE.ClampToEdgeWrapping;
      tex.repeat.set(2, 1);
    }
    return tex;
  };

  const init = () => {
    if (isInitialized || !canvasContainer) return;

    loader = new THREE.TextureLoader();
    loader.setCrossOrigin("anonymous");

    scene = new THREE.Scene();
    scene.background = null; 

    const width = canvasContainer.clientWidth;
    const height = canvasContainer.clientHeight || 1;

    camera = new THREE.PerspectiveCamera(50, width / height, 0.1, 100);
    camera.position.set(0, 0, 35);
    camera.lookAt(0, 0, 0);

    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.setSize(width, height);
    renderer.outputColorSpace = THREE.SRGBColorSpace;

    canvasContainer.innerHTML = "";
    canvasContainer.appendChild(renderer.domElement);

    bgGroup = new THREE.Group();
    scene.add(bgGroup);

    const bgUrls = parseInput(backgroundSrc);
    const totalCount = bgUrls.length; // Should be 13
    const clockCount = 12; // We only arrange the first 12 in the circle

    // Geometry calculations
    const radius = 9;
    const panelHeight = 7;
    const panelWidth = 2 * radius * Math.sin(Math.PI / totalCount) * 1.02;

    const clockRadius = 22; 

    for (let i = 0; i < totalCount; i++) {
      const url = bgUrls[i];
      const tex = loadTexture(url, true);
      const geo = new THREE.PlaneGeometry(panelWidth, panelHeight);
      
      const mat = new THREE.MeshBasicMaterial({
        map: tex,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 1,
        color: 0xffffff,
      });
      const mesh = new THREE.Mesh(geo, mat);
      
      // -- Cylinder Calculation (All 13) --
      const angleStart = (i / totalCount) * Math.PI * 2;

      // -- Clock Calculation (First 12 only) --
      // 12 o'clock start (PI/2), clockwise (-angle)
      // Evenly distribute 12 items
      let clockAngle = 0;
      let clockX = 0; 
      let clockY = 0;
      let isClockVisible = false;

      if (i < clockCount) {
          clockAngle = (Math.PI / 2) - ((i / clockCount) * Math.PI * 2);
          clockX = Math.cos(clockAngle) * clockRadius;
          clockY = Math.sin(clockAngle) * clockRadius;
          isClockVisible = true;
      }

      mesh.position.x = Math.sin(angleStart) * radius;
      mesh.position.z = Math.cos(angleStart) * radius;
      mesh.rotation.y = angleStart;

      mesh.userData = {
        id: i,
        // Cylinder data
        cylAngle: angleStart,
        cylRadius: radius,
        // Clock data
        clockX: clockX,
        clockY: clockY,
        isClockVisible: isClockVisible,
        
        initialY: mesh.position.y,
        bobSpeed: 1 + Math.random() * 1.5,
        bobOffset: Math.random() * Math.PI * 2,
        bobAmp: 0.15 + Math.random() * 0.1,
      };

      bgGroup.add(mesh);
    }

    const fgTex = loadTexture(foregroundSrc, false);
    const fgGeo = new THREE.CylinderGeometry(15, 15, 10, 64, 1, true);
    const fgMat = new THREE.MeshStandardMaterial({
      map: fgTex,
      side: THREE.DoubleSide,
      transparent: true,
      opacity: 0.95,
      emissive: 0xff5500,
      emissiveIntensity: 0.5,
      alphaTest: 0.1,
      depthWrite: false,
    });
    fgMesh = new THREE.Mesh(fgGeo, fgMat);
    fgMesh.renderOrder = 1;
    scene.add(fgMesh);

    candleLight = new THREE.PointLight(0xff6600, 100, 45);
    scene.add(candleLight);
    scene.add(new THREE.AmbientLight(0xffaa33, 0.8));

    const clock = new THREE.Clock();
    
    const animate = () => {
      animationId = requestAnimationFrame(animate);
      const time = clock.getElapsedTime();

      // --- TIMELINE PHASES ---
      // 0.0 -> 0.50 : Lantern Spin
      // 0.50 -> 0.60 : Unfold Transition
      // 0.60 -> 1.00 : Clock Highlight Cycle
      const unfoldStart = 0.50;
      const highlightStart = 0.60;
      
      let spinPhaseProgress = 0;   // 0->1
      let transitionFactor = 0;    // 0->1 (Cylinder -> Clock)
      let highlightProgress = 0;   // 0->1 (Cycle through 12 items)

      // STATE MACHINE
      if (scrollProgress.value < unfoldStart) {
          // PHASE 1: SPIN
          isClockMode = false;
          textVisible = true;
          spinPhaseProgress = scrollProgress.value / unfoldStart;
          transitionFactor = 0;
          highlightProgress = 0;
      } else if (scrollProgress.value < highlightStart) {
          // PHASE 2: UNFOLD TRANSITION
          isClockMode = true;
          textVisible = false; // Hide text during messy movement
          spinPhaseProgress = 1;
          // Normalize 0->1 over the 0.1 gap
          transitionFactor = (scrollProgress.value - unfoldStart) / (highlightStart - unfoldStart);
          // Ease smooth
          transitionFactor = transitionFactor * transitionFactor * (3 - 2 * transitionFactor);
          highlightProgress = 0;
      } else {
          // PHASE 3: HIGHLIGHT CYCLE
          isClockMode = true;
          textVisible = true;
          spinPhaseProgress = 1;
          transitionFactor = 1;
          // Normalize rest of scroll
          highlightProgress = (scrollProgress.value - highlightStart) / (1.0 - highlightStart);
      }

      // 1. ROTATION (Only relevant for Cylinder phase)
      let currentRotationY = 0;
      if (!userScrolled) {
         currentRotationY = time * 0.2 * Number(scrollspeed);
         autoRotationAngle = currentRotationY;
      } else {
         const totalRotation = Math.PI * 2; 
         const scrollRot = -(spinPhaseProgress * totalRotation);
         currentRotationY = autoRotationAngle + scrollRot;
      }

      // 2. CAMERA ZOOM
      const startZ = 35;
      const endZ = 65; 
      camera.position.z = startZ + (endZ - startZ) * transitionFactor;
      camera.lookAt(0, 0, 0);

      // 3. INDEX CALCULATIONS
      let clockActiveIndex = -1;
      
      if (isClockMode && transitionFactor >= 1) {
          // Map 0-1 highlightProgress to 0-11 indices
          // Math.floor(progress * total)
          const rawIdx = Math.floor(highlightProgress * clockCount);
          clockActiveIndex = Math.min(rawIdx, clockCount - 1);
          // Safety clamp
          if (clockActiveIndex < 0) clockActiveIndex = 0;
          
          activeIndex = clockActiveIndex; // Sync for text
      } else if (!isClockMode) {
          // Cylinder logic
          let normalizedRot = (-currentRotationY) % (Math.PI * 2);
          if (normalizedRot < 0) normalizedRot += Math.PI * 2;
          const anglePerPanel = (Math.PI * 2) / totalCount;
          let rawIndex = Math.round(normalizedRot / anglePerPanel);
          activeIndex = rawIndex % totalCount;
      } else {
          // During transition
          activeIndex = -1;
      }

      // 4. MESH UPDATE LOOP
      if (bgGroup) {
        // Group rotation fades out to 0 in clock mode
        bgGroup.rotation.y = currentRotationY * (1 - transitionFactor);

        bgGroup.children.forEach((child) => {
          const u = child.userData;
          const bobY = Math.sin(time * u.bobSpeed + u.bobOffset) * u.bobAmp;

          // -- Position Interpolation --
          const cylX = Math.sin(u.cylAngle) * u.cylRadius;
          const cylZ = Math.cos(u.cylAngle) * u.cylRadius;
          const cylRotY = u.cylAngle;
          const cylY = u.initialY + bobY;

          const clockX = u.clockX;
          const clockY = u.clockY + bobY;
          const clockZ = 0; 
          const clockRotY = 0;

          child.position.x = cylX * (1 - transitionFactor) + clockX * transitionFactor;
          child.position.y = cylY * (1 - transitionFactor) + clockY * transitionFactor;
          child.position.z = cylZ * (1 - transitionFactor) + clockZ * transitionFactor;
          child.rotation.y = cylRotY * (1 - transitionFactor) + clockRotY * transitionFactor;

          // -- Visibility / Highlight Logic --
          if (transitionFactor > 0.9) {
              // We are effectively in Clock Mode
              if (!u.isClockVisible) {
                  // Hide the 13th image
                  child.visible = false;
              } else {
                  child.visible = true;
                  
                  // HIGHLIGHT LOGIC
                  // If we are in Phase 3 (Highlight Phase)
                  if (highlightProgress > 0) {
                      const isActive = (u.id === clockActiveIndex);
                      
                      // Target values
                      const targetScale = isActive ? 1.3 : 0.85;
                      const targetOpacity = isActive ? 1.0 : 0.3;
                      
                      // Lerp visual properties for smoothness
                      // Note: Standard lerp in loop is frame-dependent, usually ok for visual polish
                      const lerpSpeed = 0.1;
                      child.scale.setScalar(
                          child.scale.x + (targetScale - child.scale.x) * lerpSpeed
                      );
                      child.material.opacity += (targetOpacity - child.material.opacity) * lerpSpeed;
                      
                      // Ensure active is drawn on top
                      child.renderOrder = isActive ? 10 : 5;
                  } else {
                      // Just unfolded, no highlight yet (waiting for scroll)
                      // Reset to defaults
                      child.scale.setScalar(1);
                      child.material.opacity = 1;
                      child.renderOrder = 5;
                  }
              }
          } else {
              // Cylinder Mode
              child.visible = true;
              child.scale.setScalar(1);
              child.material.opacity = 1;
              child.renderOrder = 0;
          }
        });
      }

      // 5. FOREGROUND FADE
      if (fgMesh) {
         fgMesh.rotation.y = currentRotationY * 0.8;
         if (fgMesh.material) fgMesh.material.opacity = 0.95 * (1 - transitionFactor);
      }
      
      renderer.render(scene, camera);
    };
    animate();

    gsap.registerPlugin(ScrollTrigger);
    ctx = gsap.context(() => {
      let h = Number(screenHeight);
      if (window.innerWidth < 768) h = h * 0.8;
      
      // Tweak scroll height: Lantern(0.5) + Unfold(0.1) + HighlightCycle(0.4)
      // Cycle needs length to feel controllable
      if (stickyWrapper) stickyWrapper.style.height = `${h * (totalCount * 0.8)}px`;

      gsap.to(scrollProgress, {
        value: 1, 
        ease: "none",
        scrollTrigger: {
          trigger: stickyWrapper,
          start: "top top",
          end: "bottom bottom",
          pin: stickyInner,
          scrub: 0.5,
          onUpdate: (self) => {
            if (!userScrolled && self.progress > 0.01) {
              userScrolled = true;
            }
          }
        },
      });
    }, stickyWrapper);

    isInitialized = true;
  };

  const kill = () => {
    if (!isInitialized) return;
    cancelAnimationFrame(animationId);
    if (ctx) {
      ctx.revert();
      ctx = null;
    }
    if (bgGroup) {
        while (bgGroup.children.length > 0) {
            const child = bgGroup.children[0];
            bgGroup.remove(child);
            if (child.geometry) child.geometry.dispose();
            if (child.material) child.material.dispose();
        }
    }
    if (fgMesh) {
      fgMesh.geometry.dispose();
      fgMesh.material.dispose();
    }
    if (renderer) {
      renderer.dispose();
      if (canvasContainer) canvasContainer.innerHTML = "";
    }
    isInitialized = false;
  };

  const onResize = () => {
    if (isInitialized && camera && renderer && canvasContainer) {
      const width = canvasContainer.clientWidth;
      const height = canvasContainer.clientHeight || 1;
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
      renderer.setSize(width, height);
    }
  };

  const updateTextures = () => {
      if (!isInitialized) return;
  };

  onMount(() => {
    if (!browser) return;
    observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            if (!isInitialized) init();
          } else {
            if (isInitialized) kill();
          }
        });
      },
      { rootMargin: "100px 0px" }
    );

    if (stickyWrapper) observer.observe(stickyWrapper);
    window.addEventListener("resize", onResize);

    return () => {
      window.removeEventListener("resize", onResize);
      if (observer) observer.disconnect();
      kill();
    };
  });
  
  $: parsedTextData = parseInput(textSrc, 13);
</script>

<section class="revolvingLatternCtn">
  <div class="revolvingLattern">
    <div class="stickyWrapper" bind:this={stickyWrapper}>
      <div class="stickyInner" bind:this={stickyInner}>
        
        <div class="stickyBackground">
          <div class="canvasContainer" bind:this={canvasContainer}></div>
        </div>
        
        <div class="foregroundTextContainer">
            {#each parsedTextData as text, i}
                <div class="textBox {i === activeIndex && textVisible ? 'active' : ''}">
                    <p>{text}</p>
                </div>
            {/each}
        </div>

        <div class="stickyForeground">
          <div class="candleOverlay" bind:this={candleOverlay}></div>
        </div>

      </div>
    </div>
  </div>
</section>

<style>
  :host {
    display: block;
    width: 100%;
    position: relative;
  }

  * { margin: 0; padding: 0; box-sizing: border-box; }

  .revolvingLattern {
    width: 100%;
    margin: 0 auto;
    background-color: #220500;
  }

  .stickyWrapper {
    position: relative;
    width: 100%;
  }

  .stickyInner {
    position: relative;
    width: 100%;
    height: 100vh;
    overflow: hidden;
  }

  .stickyBackground,
  .stickyForeground {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
  }

  .stickyBackground { z-index: 5; }
  .stickyForeground { z-index: 10; }

  .canvasContainer {
    width: 100%;
    height: 100%;
  }

  .foregroundTextContainer {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      z-index: 20; 
      display: flex;
      justify-content: center;
      align-items: center;
      pointer-events: none;
  }

  .textBox {
      position: absolute;
      width: 60%;
      max-width: 600px;
      text-align: center;
      color: #ffaa33;
      font-family: serif; 
      font-size: 2rem;
      opacity: 0;
      transform: translateY(20px);
      transition: opacity 0.3s ease, transform 0.3s ease;
      background: rgba(0,0,0,0.6); 
      padding: 20px;
      border-radius: 10px;
      text-shadow: 0 2px 4px rgba(0,0,0,0.8);
      pointer-events: auto;
  }

  .textBox.active {
      opacity: 1;
      transform: translateY(0);
  }

  .candleOverlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    background: radial-gradient(
      circle at center,
      rgba(255, 200, 100, 0.1) 10%,
      rgba(50, 10, 0, 0.3) 50%,
      rgba(20, 5, 0, 1) 90%
    );
    mix-blend-mode: multiply;
  }
</style>