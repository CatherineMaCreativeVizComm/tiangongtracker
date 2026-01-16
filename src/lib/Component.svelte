<svelte:options customElement={{ tag: "lantern-wc", shadow: "open" }} />

<script>
  import { onMount } from "svelte";
  import * as THREE from "three";
  import { gsap } from "gsap";
  import { ScrollTrigger } from "gsap/ScrollTrigger";

  const browser = typeof window !== "undefined";

  export let foregroundSrc = "https://multimedia.scmp.com/components/2026/lantern-wc/foregroundImg.png";
  export let backgroundSrc = [
	  'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/1.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/2.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/3.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/4.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/5.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/6.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/7.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/8.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/9.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/10.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/11.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/12.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/13.png'
  ];
  export let scrollspeed = 1;
  export let screenHeight = 1200;

  let stickyWrapper, stickyInner, canvasContainer, candleOverlay;
  let scene, camera, renderer, bgGroup, fgMesh, candleLight, loader;
  let animationId;
  let ctx;
  let isInitialized = false;
  let observer;

  let prevForegroundSrc = foregroundSrc;
  let prevBackgroundSrc = backgroundSrc;
  const scrollProgress = { value: 0 };

  const parseBackgroundImages = (input) => {
    if (!input) return [];
    let urls = [];
    if (Array.isArray(input)) {
      urls = input;
    } else if (typeof input === "string") {
      try {
        const parsed = JSON.parse(input);
        if (Array.isArray(parsed)) urls = parsed;
        else urls = [input];
      } catch (e) {
        urls = input
          .split(",")
          .map((s) => s.trim())
          .filter((s) => s.length > 0);
      }
    }
    if (urls.length === 0) return [];
    const result = [];
    for (let i = 0; i < 13; i++) {
      result.push(urls[i % urls.length]);
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
    scene.background = new THREE.Color(0x220500);
    scene.fog = new THREE.FogExp2(0x220500, 0.02);

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

    const totalCount = 13;
    const finalCount = 12;
    const radius = 9;
    const panelWidth = 2 * radius * Math.sin(Math.PI / totalCount) * 1.02;
    const panelHeight = 7;

    const bgUrls = parseBackgroundImages(backgroundSrc);

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
      const angleStart = (i / totalCount) * Math.PI * 2;
      const angleEnd = (i / finalCount) * Math.PI * 2;

      mesh.position.x = Math.sin(angleStart) * radius;
      mesh.position.z = Math.cos(angleStart) * radius;
      mesh.rotation.y = angleStart;

      mesh.userData = {
        id: i,
        angleStart: angleStart,
        angleEnd: angleEnd,
        radius: radius,
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

      let transition = 0;
      const startT = 0.3;
      const endT = 0.8;

      if (scrollProgress.value > startT) {
        transition = (scrollProgress.value - startT) / (endT - startT);
        if (transition > 1) transition = 1;
      }

      const smoothT = transition * transition * (3 - 2 * transition);

      camera.position.y = 0 + 40 * smoothT;
      camera.position.z = 35 - 34.9 * smoothT;
      camera.lookAt(0, 0, 0);

      if (bgGroup) {
        bgGroup.children.forEach((child, index) => {
          const u = child.userData;

          if (index === 12) {
            child.visible = smoothT < 0.9;
            if (child.material) child.material.opacity = 1 - smoothT;
          } else {
            const currentAngle = u.angleStart * (1 - smoothT) + u.angleEnd * smoothT;
            child.position.x = Math.sin(currentAngle) * u.radius;
            child.position.z = Math.cos(currentAngle) * u.radius;
            child.rotation.y = currentAngle * (1 - smoothT);
            child.rotation.x = (-Math.PI / 2) * smoothT;
          }

          child.position.y =
            u.initialY + Math.sin(time * u.bobSpeed + u.bobOffset) * u.bobAmp;
        });
      }

      if (candleLight) candleLight.intensity = 100 + Math.sin(time * 15) * 20;

      const speedMult = 1 - 0.5 * smoothT;
      const autoRot = -time * (Number(scrollspeed) * 0.1) * speedMult;
      const scrollRot = -scrollProgress.value * (Math.PI * 2);

      if (bgGroup) bgGroup.rotation.y = autoRot * 1.5 + scrollRot * 1.5;
      if (fgMesh) fgMesh.rotation.y = autoRot * 0.8 + scrollRot * 0.8;

      renderer.render(scene, camera);
    };
    animate();

    gsap.registerPlugin(ScrollTrigger);
    ctx = gsap.context(() => {
      let h = Number(screenHeight);
      if (window.innerWidth < 768) h = h * 0.8;
      if (stickyWrapper) stickyWrapper.style.height = `${h * 4}px`;

      if (candleOverlay) {
        gsap.to(candleOverlay, {
          opacity: 0.9,
          duration: 0.2,
          repeat: -1,
          yoyo: true,
          ease: "rough({ strength: 1, points: 20, randomize: true })",
        });
      }

      gsap.to(scrollProgress, {
        value: 1,
        ease: "none",
        scrollTrigger: {
          trigger: stickyWrapper,
          start: "top top",
          end: "bottom bottom",
          pin: stickyInner,
          scrub: 0.5,
          invalidateOnRefresh: true,
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

    scene = null;
    camera = null;
    renderer = null;
    bgGroup = null;
    fgMesh = null;
    candleLight = null;
    loader = null;
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
    if (!isInitialized || !loader || !bgGroup || !fgMesh) return;

    if (
      foregroundSrc &&
      foregroundSrc !== prevForegroundSrc &&
      fgMesh.material
    ) {
      fgMesh.material.map = loadTexture(foregroundSrc, false);
      fgMesh.material.needsUpdate = true;
      prevForegroundSrc = foregroundSrc;
    }

    if (
      backgroundSrc &&
      JSON.stringify(backgroundSrc) !== JSON.stringify(prevBackgroundSrc)
    ) {
      const bgUrls = parseBackgroundImages(backgroundSrc);
      bgGroup.children.forEach((child, i) => {
        const url = bgUrls[i % bgUrls.length];
        child.material.map = loadTexture(url, true);
        child.material.needsUpdate = true;
      });
      prevBackgroundSrc = backgroundSrc;
    }
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

  $: if (browser && isInitialized && (foregroundSrc || backgroundSrc)) {
    updateTextures();
  }
</script>

<section class="revolvingLatternCtn">
  <div class="revolvingLattern">
    <div class="stickyWrapper" bind:this={stickyWrapper}>
      <div class="stickyInner" bind:this={stickyInner}>
        <div class="stickyBackground">
          <div class="canvasContainer" bind:this={canvasContainer}></div>
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

  :global(body) {
    overflow-x: hidden;
    min-width: 290px;
    width: 100%;
    height: 100%;
    -webkit-tap-highlight-color: transparent;
    -webkit-touch-callout: none;
    -webkit-font-smoothing: antialiased;
  }

  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  .revolvingLattern {
    width: 100%;
    margin: 0 auto;
    background-color: #220500;
  }

  .stickyWrapper {
    position: relative;
    width: 100%;
    min-height: 100vh;
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
  }

  .canvasContainer {
    width: 100%;
    height: 100%;
  }

  .candleOverlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 10;
    background: radial-gradient(
      circle at center,
      rgba(255, 200, 100, 0.1) 10%,
      rgba(50, 10, 0, 0.3) 50%,
      rgba(20, 5, 0, 1) 90%
    );
    mix-blend-mode: multiply;
  }
</style>
