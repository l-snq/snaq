<script lang="ts">
  import { onMount } from 'svelte';
  import * as THREE from 'three';

  onMount(() => {
    const canvas = document.getElementById('hero-3d-canvas') as HTMLCanvasElement;
    if (!canvas) return;

    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: false });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.1;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x06080f);
    scene.fog = new THREE.FogExp2(0x06080f, 0.035);

    const camera = new THREE.PerspectiveCamera(55, 1, 0.1, 200);
    camera.position.set(0, 1, 7);

    // ── Lights ──
    scene.add(new THREE.AmbientLight(0x0a0d18, 1.0));

    const keyLight = new THREE.DirectionalLight(0xa8c0ff, 2.0);
    keyLight.position.set(4, 6, 4);
    scene.add(keyLight);

    const rimLight = new THREE.DirectionalLight(0x3d6eff, 1.2);
    rimLight.position.set(-5, -2, -3);
    scene.add(rimLight);

    const pt1 = new THREE.PointLight(0x3d6eff, 3.0, 12);
    pt1.position.set(0, 0, 3);
    scene.add(pt1);

    const pt2 = new THREE.PointLight(0x1a3d9e, 1.5, 10);
    pt2.position.set(3, -3, 0);
    scene.add(pt2);

    // ── Stars ──
    const starCount = 1800;
    const starPositions = new Float32Array(starCount * 3);
    for (let i = 0; i < starCount * 3; i++) {
      starPositions[i] = (Math.random() - 0.5) * 160;
    }
    const starGeo = new THREE.BufferGeometry();
    starGeo.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
    const starMat = new THREE.PointsMaterial({ color: 0xa8c0ff, size: 0.12, sizeAttenuation: true, transparent: true, opacity: 0.7 });
    scene.add(new THREE.Points(starGeo, starMat));

    // ── Central icosahedron ──
    const icoGeo = new THREE.IcosahedronGeometry(1.1, 0);
    const icoMat = new THREE.MeshStandardMaterial({
      color: 0x0a0d18,
      roughness: 0.1,
      metalness: 0.95,
      envMapIntensity: 1,
    });
    const icoMesh = new THREE.Mesh(icoGeo, icoMat);
    scene.add(icoMesh);

    // Wireframe over ico
    const icoEdges = new THREE.LineSegments(
      new THREE.EdgesGeometry(new THREE.IcosahedronGeometry(1.12, 0)),
      new THREE.LineBasicMaterial({ color: 0x3d6eff, transparent: true, opacity: 0.9 })
    );
    scene.add(icoEdges);

    // ── Torus rings ──
    const ring1 = new THREE.Mesh(
      new THREE.TorusGeometry(2.0, 0.03, 8, 100),
      new THREE.MeshStandardMaterial({ color: 0x3d6eff, roughness: 0.1, metalness: 1.0, emissive: 0x1a3d9e, emissiveIntensity: 0.4 })
    );
    ring1.rotation.x = Math.PI / 2;
    const ringGroup1 = new THREE.Group();
    ringGroup1.add(ring1);
    scene.add(ringGroup1);

    const ring2 = new THREE.Mesh(
      new THREE.TorusGeometry(2.4, 0.018, 8, 100),
      new THREE.MeshStandardMaterial({ color: 0x1a3d9e, roughness: 0.2, metalness: 1.0, emissive: 0x091a46, emissiveIntensity: 0.3 })
    );
    ring2.rotation.set(Math.PI / 3, Math.PI / 5, 0);
    const ringGroup2 = new THREE.Group();
    ringGroup2.add(ring2);
    scene.add(ringGroup2);

    const ring3 = new THREE.Mesh(
      new THREE.TorusGeometry(1.6, 0.012, 8, 100),
      new THREE.MeshStandardMaterial({ color: 0xa8c0ff, roughness: 0.1, metalness: 1.0, emissive: 0x3d6eff, emissiveIntensity: 0.2, transparent: true, opacity: 0.6 })
    );
    ring3.rotation.set(-Math.PI / 4, Math.PI / 6, 0);
    const ringGroup3 = new THREE.Group();
    ringGroup3.add(ring3);
    scene.add(ringGroup3);

    // ── Floating tetrahedra ──
    const tetraPositions: [number, number, number][] = [
      [2.2, 1.4, 0.5], [-2.0, 1.8, -0.3],
      [1.8, -1.6, 0.8], [-1.6, -1.4, -0.6],
      [0.4, 2.4, -1.0], [-0.6, -2.2, 1.2],
      [2.8, 0.2, -0.8], [-2.6, -0.4, 0.4],
    ];
    const tetraMeshes = tetraPositions.map((pos, i) => {
      const size = 0.12 + Math.random() * 0.14;
      const mesh = new THREE.Mesh(
        new THREE.TetrahedronGeometry(size, 0),
        new THREE.MeshStandardMaterial({
          color: i % 3 === 0 ? 0x3d6eff : i % 3 === 1 ? 0xa8c0ff : 0x1a3d9e,
          roughness: 0.1,
          metalness: 0.9,
          emissive: i % 3 === 0 ? 0x1a3d9e : 0x091a46,
          emissiveIntensity: 0.5,
        })
      );
      mesh.position.set(...pos);
      mesh.rotation.set(Math.random() * Math.PI, Math.random() * Math.PI, Math.random() * Math.PI);
      return mesh;
    });

    const mainGroup = new THREE.Group();
    mainGroup.add(icoMesh, icoEdges, ringGroup1, ringGroup2, ringGroup3);
    tetraMeshes.forEach(m => mainGroup.add(m));
    scene.add(mainGroup);

    // ── Resize ──
    function resize() {
      const w = canvas.clientWidth;
      const h = canvas.clientHeight || window.innerHeight;
      if (canvas.width !== w || canvas.height !== h) {
        renderer.setSize(w, h, false);
        camera.aspect = w / h;
        camera.updateProjectionMatrix();
      }
    }

    // ── Mouse parallax ──
    let mouseX = 0, mouseY = 0;
    const onMouseMove = (e: MouseEvent) => {
      mouseX = (e.clientX / window.innerWidth - 0.5) * 2;
      mouseY = (e.clientY / window.innerHeight - 0.5) * 2;
    };
    window.addEventListener('mousemove', onMouseMove);

    let t = 0, last = 0;
    let rafId: number;

    function animate(now: number) {
      rafId = requestAnimationFrame(animate);
      const delta = Math.min((now - last) / 1000, 0.05);
      last = now;
      t += delta;

      resize();

      // Main rotation
      mainGroup.rotation.y = t * 0.22;
      mainGroup.rotation.x = Math.sin(t * 0.18) * 0.12;

      // Rings counter-rotate at different speeds
      ringGroup1.rotation.z = t * 0.5;
      ringGroup2.rotation.y = -t * 0.35;
      ringGroup3.rotation.x = t * 0.6;

      // Floating tetrahedra spin & bob
      tetraMeshes.forEach((m, i) => {
        m.rotation.x += delta * (0.4 + i * 0.08);
        m.rotation.y += delta * (0.3 + i * 0.06);
        m.position.y = tetraPositions[i][1] + Math.sin(t * 0.8 + i * 1.1) * 0.12;
      });

      // Subtle camera parallax
      camera.position.x += (mouseX * 0.6 - camera.position.x) * 0.04;
      camera.position.y += (-mouseY * 0.4 + 1 - camera.position.y) * 0.04;
      camera.lookAt(0, 0, 0);

      // Pulse the point light
      pt1.intensity = 2.5 + Math.sin(t * 2.1) * 0.6;

      renderer.render(scene, camera);
    }
    rafId = requestAnimationFrame(animate);

    return () => {
      cancelAnimationFrame(rafId);
      window.removeEventListener('mousemove', onMouseMove);
      renderer.dispose();
    };
  });
</script>
