<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { FontLoader, Font } from "three/addons/loaders/FontLoader.js";
    import { TextGeometry } from "three/addons/geometries/TextGeometry.js";
    import { FirstPersonControls } from "three/addons/controls/FirstPersonControls.js";

    import { addGround, onWindowResize, loadModels } from "$lib/Helper-3D";

    import * as d3 from "d3";

    // --------------------------------------------------------------
    // 🔥 1. LOAD REAL MOVIE DATA
    // --------------------------------------------------------------

    type TMovie = {
        genres: string[];
    };

    let movies: TMovie[] = [];

    async function loadMovieData() {
        const rows = await d3.csv("/summer_movies.csv");

        movies = rows.map((d: any) => ({
            genres: d.genres ? d.genres.split(",").map(s => s.trim()) : []
        }));
    }

    // --------------------------------------------------------------
    // 🔥 2. COMPUTE GENRE COUNTS (REPLACES DUMMY DATA)
    // --------------------------------------------------------------

    function getGenreCounts() {
        const counts: Record<string, number> = {};

        movies.forEach(m => {
            m.genres.forEach(g => {
                if (!counts[g]) counts[g] = 0;
                counts[g]++;
            });
        });

        return counts;
    }

    let genreData: Record<string, number> = {};  // replaces dummyData

    // --------------------------------------------------------------
    // 🔥 3. LOAD DATA BEFORE INIT
    // --------------------------------------------------------------

    onMount(async () => {
        await loadMovieData();        // 1. load CSV
        genreData = getGenreCounts(); // 2. compute real genre counts
        init(window.innerWidth, window.innerHeight);
    });

    // --------------------------------------------------------------
    // (YOUR ORIGINAL CODE BELOW – UNCHANGED)
    // --------------------------------------------------------------

    let container: HTMLElement;
    let camera: THREE.PerspectiveCamera;
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;

    const FLOOR = -250;

    const morphs: Array<THREE.Mesh> = [];
    let mixer: THREE.AnimationMixer;

    const clock = new THREE.Clock();

    function init(SCREEN_WIDTH: number, SCREEN_HEIGHT: number) {
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(SCREEN_WIDTH, SCREEN_HEIGHT);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFShadowMap;
        container.appendChild(renderer.domElement);

        camera = new THREE.PerspectiveCamera(
            23,
            SCREEN_WIDTH / SCREEN_HEIGHT,
            10,
            3000,
        );
        camera.position.set(0, 50, 1900);

        scene = new THREE.Scene();

        new THREE.TextureLoader().load("/3D/sky.jpg", (texture) => {
            texture.repeat.set(0.8, 1);
            scene.background = texture;
        });

        const ambient = new THREE.AmbientLight(0xffffff);
        scene.add(ambient);

        const light = new THREE.DirectionalLight(0xffffff, 3);
        light.position.set(0, 1500, 1000);
        light.castShadow = true;
        Object.assign(light.shadow.camera, {
            top: 2000,
            bottom: -2000,
            left: -2000,
            right: 2000,
            near: 1200,
            far: 2500,
        });
        light.shadow.bias = 0.0001;
        light.shadow.mapSize.width = 2048;
        light.shadow.mapSize.height = 1024;
        scene.add(light);

        addGround(scene, FLOOR, "/3D/grasslight-big.jpg");

        const fontLoader = new FontLoader();

        fontLoader.load("/3D/helvetiker_bold.typeface.json", (font: Font) => {
            const textGeo = new TextGeometry("summer movies", {
                font: font,
                size: 40,
                depth: 15,
            });

            textGeo.computeBoundingBox();
            const centerOffset =
                -0.5 * (textGeo.boundingBox!.max.x - textGeo.boundingBox!.min.x);

            const textMaterial = new THREE.MeshStandardMaterial({
                color: 0x449900,
            });

            const titleMesh = new THREE.Mesh(textGeo, textMaterial);
            titleMesh.position.x = centerOffset;
            titleMesh.position.y = FLOOR + 500;
            titleMesh.castShadow = true;
            scene.add(titleMesh);

            // 🔥 USE REAL GENRE DATA HERE
            createBars(scene, font, genreData);
        });

        const models = [
            {
                path: "/3D/Horse.glb",
                speed: 300,
                duration: 1,
                x: 100 - Math.random() * 1000,
                y: FLOOR,
                z: 300,
                scale: 0.5,
            },
            {
                path: "/3D/Horse.glb",
                speed: 300,
                duration: 1,
                x: 100 - Math.random() * 1000,
                y: FLOOR,
                z: 100,
                scale: 0.5,
            },
            {
                path: "/3D/Flamingo.glb",
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 100,
                scale: 0.5,
            },
            {
                path: "/3D/Flamingo.glb",
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 200,
                scale: 0.5,
            },
            {
                path: "/3D/Parrot.glb",
                speed: 350,
                duration: 0.5,
                x: 500 - Math.random() * 500,
                y: FLOOR + 500,
                z: 700,
                scale: 0.5,
            },
        ];

        mixer = loadModels(models, scene, mixer, morphs);

        window.addEventListener("resize", () =>
            onWindowResize(
                camera,
                renderer,
                window.innerWidth,
                window.innerHeight,
            ),
        );

        renderer.setAnimationLoop(animate);
    }

    function createBars(scene: THREE.Scene, font: Font, data: any) {
        const maxHeight = 400;
        const barMaxWidth = window.innerWidth * 0.9;   // 90% of screen width


        const xScale = d3
            .scaleBand()
            .domain(Object.keys(data))
            .range([-barMaxWidth / 2, barMaxWidth / 2])
            .padding(0.2);


        const yScale = d3
            .scaleLinear()
            .domain([0, Math.max(...Object.values(data).map((d) => d))])
            .range([0, maxHeight]);

        Object.keys(data).forEach((genre) => {
            const bar = createOneBar(
                yScale(data[genre]),
                xScale.bandwidth(),
            );
            bar.position.set(
                xScale(genre),
                FLOOR + yScale(data[genre]) / 2,
                0,
            );
            scene.add(bar);

            addLabelToBar(
                scene,
                `${genre}: ${data[genre]}`,
                xScale(genre)! - xScale.bandwidth() / 2,
                FLOOR + 5,
                bar.position.z + xScale.bandwidth(),
                font,
            );
        });
    }

    function createOneBar(height: number, width: number) {
        const geometry = new THREE.CylinderGeometry(
            width / 2,
            width / 2,
            height,
            32,
        );

        const material = new THREE.MeshStandardMaterial({
            map: new THREE.TextureLoader().load("/3D/wood-texture.jpg"),
        });

        const bar = new THREE.Mesh(geometry, material);
        bar.castShadow = true;
        bar.receiveShadow = true;
        return bar;
    }

    function addLabelToBar(
        scene: THREE.Scene,
        text: string,
        x: number,
        y: number,
        z: number,
        font: Font,
    ) {
        const textGeometry = new TextGeometry(text, {
            font: font,
            size: 12,
            depth: 4,
        });

        const textMaterial = new THREE.MeshPhysicalMaterial({
            color: 0xffffff,
        });

        const textMesh = new THREE.Mesh(textGeometry, textMaterial);

        textMesh.position.set(x, y, z);
        textMesh.castShadow = true;
        textMesh.receiveShadow = false;

        scene.add(textMesh);
    }

    function animate() {
        const delta = clock.getDelta();
        mixer.update(delta);

        morphs.forEach((morph) => {
            morph.position.x += morph.speed * delta;
            if (morph.position.x > window.innerWidth / 2) {
                morph.position.x = -window.innerWidth / 2 - Math.random() * 200;
            }
        });

        renderer.render(scene, camera);
    }
</script>

<div bind:this={container} class="container"></div>

<style>
    div.container {
        width: 100%;
        height: 100%;
    }
</style>
