<script>
    import * as THREE from 'three';
    import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
    import {FontLoader} from 'three/examples/jsm/loaders/FontLoader.js';
    import {TextGeometry} from 'three/examples/jsm/geometries/TextGeometry.js';

    import {
        ButtonDropdown,
        DropdownItem,
        DropdownMenu,
        DropdownToggle
    } from 'sveltestrap';

    // import components
    import M from './M.svelte';
    import ParSurf from './objects/ParSurf.svelte';
    import Level from './objects/Level.svelte';
    import Curve from './objects/Curve.svelte';
    import Field from './objects/Field.svelte';
    import Function from './objects/Function.svelte';
    import Vector from './objects/Vector.svelte';
    import Settings from './Settings.svelte';
    import { onMount } from 'svelte';
    import { slide } from 'svelte/transition';
    import { quintOut } from 'svelte/easing';
    import Stats from 'stats.js';

    import Linear from './Linear.svelte';
    import Chapter from './Chapter.svelte';
    import Intro from './Intro.svelte';
    import Session from './modes/Session.svelte';
    import KeyboardControls from './KeyboardControls.svelte';

    import {
        getRoomId, makeSocket
    } from './rooms';
    import {
        drawAxes,
        drawGrid,
        labelAxes,
        makeHSLColor,
    } from './utils';
    import {
        makeObject, removeObject, updateObject,
        publishScene,
        handleSceneEvent
    } from './sceneUtils';
    import {handlePollEvent} from './polls/pollUtils';

    let debug = false,
        stats;
    // stats window for debugging
    if (debug) {
        stats = new Stats();
        stats.showPanel(1); // 0: fps, 1: ms, 2: mb, 3+: custom
        document.body.appendChild(stats.dom);
    }

    let flipInfo = false;
    let shadeUp = false;
    let scaleAnimation = false;
    let scaleUpdate;
    let selection = null;

    const selectObject = (uuid) => {
        selection = uuid;
    }

    let canvas;

    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(
        75,
        window.innerWidth / window.innerHeight,
        0.001,
        1000
    );

    const camera2 = new THREE.OrthographicCamera(
        -2,
        2,
        (window.innerHeight / window.innerWidth) * 2,
        (-window.innerHeight / window.innerWidth) * 2,
        0,
        100 * 2
    );

    let orthoCamera = false;

    let gridMax = 1,
        gridStep = 1 / 5;
    const pi = Math.PI;

    // Make z the default up
    THREE.Object3D.DefaultUp.set(0, 0, 1);

    camera.position.x = (gridMax * 2) / 2;
    camera.position.y = (-gridMax * 3) / 2;
    camera.position.z = (gridMax * 4.5) / 2;
    camera.up.set(0, 0, 1);
    camera.lookAt(0, 0, 0);

    camera2.position.x = (gridMax * 2) / 2;
    camera2.position.y = (-gridMax * 3) / 2;
    camera2.position.z = (gridMax * 4.5) / 2;
    camera2.up.set(0, 0, 1);
    camera2.lookAt(0, 0, 0);

    $: currentCamera = orthoCamera ? camera2 : camera;

    scene.background = new THREE.Color(0xddddef);
    // for greenscreen
    // scene.background = new THREE.Color(0x88ff88);

    // soft white light
    scene.add(new THREE.AmbientLight(0xa0a0a0));

    //something to make shiny things shine - a chandelier
    const chandelier = new THREE.Object3D();
    const candles = 3;
    for (let i = 0; i < candles; i++) {
        for (let j = 0; j < 2; j++) {
            const light = new THREE.PointLight(0xffffff, 0.5, 1000);
            light.position.set(
                20 * Math.cos((i * 2 * pi) / candles + (Math.pow(-1, j) * 1) / 2),
                20 * Math.sin((i * 2 * pi) / candles + (Math.pow(-1, j) * 1) / 2),
                Math.pow(-1, j) * 10
            );
            chandelier.add(light);
        }
    }
    scene.add(chandelier);

    let controls, controls2;

    let renderer;
    let frameRequested = false;

    // Grid
    const lineMaterial = new THREE.LineBasicMaterial({
        color: 0x551122,
        transparent: true,
        opacity: 1,
        depthTest: true,
    });

    let gridMeshes = drawGrid({ lineMaterial });
    gridMeshes.renderDepth = -1;
    scene.add(gridMeshes);

    // Axes
    const axesMaterial = new THREE.MeshLambertMaterial({ color: 0x320032 });
    let axesHolder = drawAxes({ gridMax, gridStep, axesMaterial });
    scene.add(axesHolder);

    const requestFrameIfNotRequested = function() {
        if (!frameRequested) {
            frameRequested = true;
            myReq = requestAnimationFrame(render);
        }
    }

    // Fonts
    const fontLoader = new FontLoader();
    let [axesText] = labelAxes({
        scene,
        render: requestFrameIfNotRequested,
    }, fontLoader, TextGeometry);

    // from https://threejsfundamentals.org
    const resizeRendererToDisplaySize = function(renderer) {
        const canvas = renderer.domElement;
        const width = canvas.clientWidth;
        const height = canvas.clientHeight;
        const needResize = canvas.width !== width || canvas.height !== height;
        if (needResize) {
            renderer.setSize(width, height, false);
        }
        return needResize;
    }

    let myReq,
        last,
        animating = false;


    const animateIfNotAnimating = function() {
        if (!animating) {
            cancelAnimationFrame(myReq);
            frameRequested = true;
            myReq = requestAnimationFrame(animate);
            animating = true;
        }
    }

    const animate = (time = 0) => {
        if (debug) {
            stats.begin();
        }

        if (!last) last = time;

        const dt = (time - last) / 1000;
        last = time;

        for (const b of objects.filter((b) => b.animation)) {
            b.update(dt);
        }

        if (scaleAnimation) {
            scaleUpdate(dt);
        }

        controls.update();
        renderer.render(scene, camera);
        if (debug) {
            stats.end();
        }
        if (scaleAnimation || objects.some((b) => b.animation)) {
            myReq = requestAnimationFrame(animate);
            frameRequested = true;
            animating = true;
        } else {
            cancelAnimationFrame(myReq);
            frameRequested = false;
            animating = false;
            last = false;
        }
    };

    const render = function() {
        frameRequested = false;

        for (let index = 0; index < axesText.length; index++) {
            const element = axesText[index];
            element.lookAt(camera.position);
        }

        if (resizeRendererToDisplaySize(renderer)) {
            const canvas = renderer.domElement;
            camera.aspect = canvas.clientWidth / canvas.clientHeight;
            camera2.top = canvas.clientHeight / 2;
            camera2.bottom = canvas.clientHeight / -2;
            camera2.right = canvas.clientWidth / 2;
            camera2.left = canvas.clientWidth / -2;
            camera.updateProjectionMatrix();
            camera2.updateProjectionMatrix();
        }

        controls.update();
        if (orthoCamera) {
            renderer.render(scene, camera2);
        } else {
            renderer.render(scene, camera);
        }
    }

    const resize = () => {
        renderer.setSize(window.innerWidth, window.innerHeight);
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
    };

    const createScene = (el) => {
        renderer = new THREE.WebGLRenderer({
            antialias: true,
            canvas: el
        });

        controls = new OrbitControls(camera, canvas);
        controls2 = new OrbitControls(camera2, canvas);

        controls.listenToKeyEvents(window);
        controls.enableDamping = true; // an animation loop is required when either damping or auto-rotation are enabled
        controls.dampingFactor = 0.05;

        controls.screenSpacePanning = false;

        // controls.minDistance = 100;
        // controls.maxDistance = 500;

        controls.maxPolarAngle = Math.PI;

        controls.addEventListener("change", requestFrameIfNotRequested);
        controls2.addEventListener("change", requestFrameIfNotRequested);

        resize();
        requestFrameIfNotRequested();
    };

    export let objects = [];
    export let currentPoll = null;

    if (window.SCENE_STATE && window.SCENE_STATE.objects) {
        objects = window.SCENE_STATE.objects;
    }

    export const blowUpObjects = () => {
        if(confirm('Remove all objects in the scene?')) {
            objects = [];
        }
    };

    window.addEventListener("resize", () => {
        resize();
        requestFrameIfNotRequested();
    });

    const makeQueryStringObject = function() {
        const flattenedObjects = {
            currentChapter,
            shadeUp,
            flipInfo,
            grid: gridMeshes.visible,
        };
        objects.forEach((object, index) => {
            const prefix = `obj${index}_`;
            flattenedObjects[prefix + "kind"] = object.kind;
            if (object.params) {
                for (const [key, value] of Object.entries(object.params)) {
                    flattenedObjects[prefix + "params_" + key] = value;
                }
            }
        });
        const urlParams = new URLSearchParams(flattenedObjects);
        window.location.search = urlParams.toString();
    }

    onMount(() => {
        createScene(canvas);
        const urlParams = new URLSearchParams(location.search);
        if (urlParams.keys()) {
            const objectHolder = {};
            urlParams.forEach((val, key) => {
                // This is bad and stupid, and hopefully it will be done better.
                // make a viewStatus object, maybe?
                if (key === "currentChapter") {
                    currentChapter = val;
                }
                if (key === "shadeUp") {
                    shadeUp = val;
                }
                if (key === "grid") {
                    gridMeshes.visible = val === "true";
                }
                if (key === "flipInfo") {
                    flipInfo = val;
                }
                if (key.slice(0, 3) === "obj") {
                    const keyParts = key.split("_");
                    if (keyParts[1] === "kind") {
                        objectHolder[keyParts[0]] = { kind: val, params: {} };
                    } else {
                        objectHolder[keyParts[0]].params[keyParts[2]] = val;
                    }
                }
            });

            for (const val of Object.values(objectHolder)) {
                objects = makeObject(
                    val.uuid, val.kind, val.params, objects);
            }
        }
    });

    const onPublishScene = function() {
        publishScene(objects, socket);
    };

    let currentChapter = 'Intro';
    let currentMode = 'intro';

    let pollResponses = [];

    const handleSocketMessage = function(e) {
        const data = JSON.parse(e.data);
        
        if (data.message.pollResponse) {
            pollResponses = [...pollResponses, data.message.pollResponse];
        } else if (data.message.broadcastPoll) {
            currentPoll = handlePollEvent(data);
        } else {
            objects = handleSceneEvent(data, objects);
        }
    };

    let socket = null;
    const router = {};
    const room = location.pathname.match(/\/rooms\/\d+\//);
    let roomId = null;

    if (room) {
        currentMode = 'session';
        router.room = true;
        roomId = getRoomId(window.location.pathname);
        socket = makeSocket(roomId, handleSocketMessage);
    }

    const altDown = (e) => {
        if (e.altKey) {
            e.preventDefault;
            let i = 0;
            if (e.code === "Space") {
                shadeUp = !shadeUp;
            } else if (objects.length > 0) {
                if (!selection) {
                    switch (e.code) {
                        case "BracketRight":
                            selection = objects[objects.length-1].uuid;
                            break;
                        case "BracketLeft":
                            selection = objects[0].uuid;
                            break;
                    }
                } else {
                    while (i < objects.length && objects[i].uuid !== selection) { i++; }
                    switch(e.code) {
                        case "BracketRight":
                            if (i === 0) {
                                selection = objects[objects.length-1].uuid;
                                break;
                            }
                            selection = objects[i-1].uuid;
                            break;
                        case "BracketLeft":
                            if (i > objects.length-2) {
                                selection = objects[0].uuid
                                break;
                            }
                            selection = objects[i+1].uuid;
                            break;
                    }
                }
            }
        }
    };
    window.addEventListener("keydown", altDown, false);
</script>
<main>
    <canvas bind:this={canvas} id="c" />

    <div class="info" class:flipInfo>
        <div class="info-inner">
            <div class="chapterBox">
                <div class="collapse-info" hidden={shadeUp}>
                    <div class="d-flex mb-2">
                        <h1 class="titlefont flex-grow-1 px-2">3Demos.xyz (βeta)</h1>
                        <ButtonDropdown class="px-2">
                            <DropdownToggle caret class="btn btn-light">
                                Modes
                            </DropdownToggle>
                            <DropdownMenu>
                                <DropdownItem
                                    active={currentMode === 'intro'}
                                    on:click={() => (currentMode = 'intro')}>
                                    Intro
                                </DropdownItem>
                                <DropdownItem
                                    active={currentMode === 'story'}
                                    on:click={() => (currentMode = 'story')}>
                                    Story
                                </DropdownItem>
                                <DropdownItem
                                    active={currentMode === 'creative'}
                                    on:click={() => (currentMode = 'creative')}>
                                    Creative
                                </DropdownItem>
                                <DropdownItem
                                    active={currentMode === 'session'}
                                    on:click={() => (currentMode = 'session')}>
                                    Session
                                </DropdownItem>
                            </DropdownMenu>
                        </ButtonDropdown>
                        <button
                            class="ms-auto btn btn-light px-2"
                            on:click={() => {
                                flipInfo = !flipInfo;
                            }}>
                            Object List 
                            <i class="fa fa-sliders" />
                        </button>
                    </div>
                    <div class="object-box-title d-flex">
                        <ButtonDropdown class="mb-2">
                            <DropdownToggle
                                caret
                                class="btn btn-light dropdown-toggle">
                                Chapters
                            </DropdownToggle>
                            <DropdownMenu>
                                <DropdownItem
                                    on:click={() => (currentChapter = "Intro")}>
                                    Intro
                                </DropdownItem>
                                <DropdownItem
                                    on:click={() => (currentChapter = "Chapter")}>
                                    Arc Length & Curvature
                                </DropdownItem>
                                <DropdownItem
                                    on:click={() => (currentChapter = "Linear")}>
                                    Linearization
                                </DropdownItem>
                                <DropdownItem
                                    on:click={() => (currentChapter = "Keyboard Controls")}>
                                    Keyboard Controls
                                </DropdownItem>
                            </DropdownMenu>
                        </ButtonDropdown>
                    </div>
                    {#if currentMode === 'session'}
                        <Session
                            bind:roomId
                            bind:socket
                            bind:currentPoll />
                    {:else}
                        {#if currentChapter === "Chapter"}
                            <Chapter bind:objects />
                        {:else if currentChapter === "Linear"}
                            <Linear bind:objects />
                        {:else if currentChapter === "Keyboard Controls"}
                            <KeyboardControls />
                        {:else if currentChapter === "Intro"}
                            <Intro />
                        {/if}
                    {/if}
                </div>
                <button class="btn btn-sm btn-light mt-1 raise-lower-button d-flex justify-content-center"
                        title="Raise/Lower window"
                        on:click={() => {
                    shadeUp = !shadeUp;
                    }}>
                    <i class={`bi bi-chevron-${shadeUp ? 'down' : 'up'}`}></i>
                </button>
            </div>

            <div class="objectBoxOuter">
                <div class="collapse-info" hidden={shadeUp}>
                    <div class="object-box-title d-flex mb-2">
                        <h2 class="flex-grow-1 px-2">3D Objects</h2>
                        <ButtonDropdown class="px-2">
                            <DropdownToggle caret class="btn btn-light">
                                Modes
                            </DropdownToggle>
                            <DropdownMenu>
                                <DropdownItem
                                    active={currentMode === 'intro'}
                                    on:click={() => (currentMode = 'intro')}>
                                    Intro
                                </DropdownItem>
                                <DropdownItem
                                    active={currentMode === 'story'}
                                    on:click={() => (currentMode = 'story')}>
                                    Story
                                </DropdownItem>
                                <DropdownItem
                                    active={currentMode === 'creative'}
                                    on:click={() => (currentMode = 'creative')}>
                                    Creative
                                </DropdownItem>
                                <DropdownItem
                                    active={currentMode === 'session'}
                                    on:click={() => (currentMode = 'session')}>
                                    Session
                                </DropdownItem>
                            </DropdownMenu>
                        </ButtonDropdown>
                        <button
                            class="btn btn-light ms-auto px-2"
                            on:click={() => {
                                flipInfo = !flipInfo;
                            }}
                        >
                            Chapters 
                            <i class="fa fa-book" />
                        </button>
                    </div>
                    <div class="btn-toolbar justify-content-between" role="toolbar">
                        <div class="btn-group mb-2">
                            <ButtonDropdown>
                                <DropdownToggle color="primary">
                                    Add Object
                                    <i class="fa fa-plus" />
                                </DropdownToggle>
                                <DropdownMenu>
                                    <DropdownItem on:click={() =>
                                        objects = makeObject(null, "vector", {
                                        a: "0.2",
                                        b: "-0.3",
                                        c: "0",
                                        x: "0",
                                        y: "0",
                                        z: "0",
                                        show: true,
                                        }, objects)
                                }>
                                    Vector <M size="sm">\mathbf v = \langle a, b, c \rangle</M>
                                </DropdownItem>
                                <DropdownItem on:click={() => 
                                    objects = makeObject(null, "curve", {
                                        a: "0",
                                        b: "2*pi",
                                        x: "cos(t)",
                                        y: "sin(t)",
                                        z: `cos(${Math.ceil(10 * Math.random()).toString()}*t)`,
                                        tau: 0,
                                        color: `#${makeHSLColor(Math.random()).getHexString()}`,
                                    },
                                    objects
                                )}
                                >
                                    Space Curve <M size="sm">\mathbf r(t)</M>
                                </DropdownItem>
                                <DropdownItem on:click={() =>
                                    objects = makeObject(null, "graph", {
                                        a: "-2",
                                        b: "2",
                                        c: "-2",
                                        d: "2",
                                        z: `cos(${Math.ceil(
                                        3 * Math.random()
                                        ).toString()}*x + ${Math.ceil(
                                        2 * Math.random()
                                        ).toString()}*y)/(1 + x^2 + y^2)`,
                                        tau: 0,
                                        // color: "#3232ff",
                                    }, objects)}>
                                    Graph <M size="sm">z = f(x,y)</M>
                                </DropdownItem>
                                <DropdownItem on:click={() =>
                                    objects = makeObject(null, "level", {
                                        g: "x^2 + 2 y^2 - z^2",
                                        k: "1",
                                        a: "-2",
                                        b: "2",
                                        c: "-2",
                                        d: "2",
                                        e: "-2",
                                        f: "2",
                                    }, objects)}>
                                    Level Surface <M size="sm">g(x,y,z) = k</M>
                                </DropdownItem>
                                <DropdownItem on:click={() =>
                                    objects = makeObject(null, "parsurf", {
                                        a: "0",
                                        b: "2*pi",
                                        c: "0",
                                        d: "2*pi",
                                        x: "cos(u)*(1 + sin(v)/3)",
                                        y: "sin(u)*(1 + sin(v)/3)",
                                        z: "cos(v)/3",
                                    }, objects)}>
                                    Parametric Surface <M size="sm">\mathbf r(u,v)</M>
                                </DropdownItem>
                                <DropdownItem on:click={() =>
                                    objects = makeObject(null, "field", {
                                        p: "x",
                                        q: "y",
                                        r: "-z",
                                        nVec: 6,
                                    }, objects)}>
                                    Vector Field<M size="sm">\mathbf F(x,y,z)</M>
                                </DropdownItem>
                                <DropdownItem on:click={() =>
                                    objects = makeObject(null, "box", {
                                        size: 2
                                        },
                                        objects
                                        )}>
                                        Random Box
                                    </DropdownItem>
                                </DropdownMenu>
                            </ButtonDropdown>
                            <button class="btn btn-danger" on:click={blowUpObjects}>
                                Clear Objects
                                <i class="fa fa-trash" />
                            </button>
                        </div>
                        {#if currentMode === 'session'}
                            <button class="btn btn-primary mb-2" on:click={onPublishScene}>
                                Publish Scene
                                <i class="bi bi-broadcast-pin"></i>
                            </button>
                        {/if}
                    </div>

                    <div class="objectBoxInner">
                        {#each objects as b}
                            <div transition:slide={{ delay: 0, duration: 300, easing: quintOut }}>
                                {#if b.kind === "parsurf"}
                                    <ParSurf
                                        {scene}
                                        render={requestFrameIfNotRequested}
                                        params={b.params}
                                        onClose={() => objects = removeObject(b.uuid, objects)}
                                        onUpdate={() => objects = updateObject(b, objects)}
                                        bind:update={b.update}
                                        bind:animation={b.animation}
                                        bind:selected={selection}
                                        on:click={selectObject(b.uuid)}
                                        on:keydown={altDown}
                                        objID={b.uuid}
                                    />
                                {:else if b.kind === "graph"}
                                    <Function
                                        {scene}
                                        camera={currentCamera}
                                        {controls}
                                        render={requestFrameIfNotRequested}
                                        params={b.params}
                                        onClose={() => objects = removeObject(b.uuid, objects)}
                                        onUpdate={() => objects = updateObject(b, objects)}
                                        bind:shadeUp
                                        bind:update={b.update}
                                        bind:animation={b.animation}
                                        on:animate={animateIfNotAnimating}
                                        bind:selected={selection}
                                        on:click={selectObject(b.uuid)}
                                        on:keydown={altDown}
                                        objID={b.uuid}
                                        {gridStep}
                                    />
                                {:else if b.kind === "level"}
                                    <Level
                                        {scene}
                                        camera={currentCamera}
                                        {controls}
                                        render={requestFrameIfNotRequested}
                                        onClose={() => objects = removeObject(b.uuid, objects)}
                                        onUpdate={() => objects = updateObject(b, objects)}
                                        params={b.params}
                                        bind:shadeUp
                                        bind:update={b.update}
                                        bind:animation={b.animation}
                                        on:animate={animateIfNotAnimating}
                                        bind:selected={selection}
                                        on:click={selectObject(b.uuid)}
                                        on:keydown={altDown}
                                        objID={b.uuid}
                                        {gridStep}
                                    />
                                {:else if b.kind === "curve"}
                                    <Curve
                                        {scene}
                                        render={requestFrameIfNotRequested}
                                        onClose={() => objects = removeObject(b.uuid, objects)}
                                        onUpdate={() => objects = updateObject(b, objects)}
                                        params={b.params}
                                        bind:update={b.update}
                                        bind:animation={b.animation}
                                        on:animate={animateIfNotAnimating}
                                        bind:selected={selection}
                                        on:click={selectObject(b.uuid)}
                                        on:keydown={altDown}
                                        objID={b.uuid}
                                        {gridStep}
                                    />
                                {:else if b.kind === "field"}
                                    <Field
                                        {scene}
                                        render={requestFrameIfNotRequested}
                                        onClose={() => objects = removeObject(b.uuid, objects)}
                                        onUpdate={() => objects = updateObject(b, objects)}
                                        bind:update={b.update}
                                        bind:animation={b.animation}
                                        on:animate={animateIfNotAnimating}
                                        params={b.params}
                                        bind:selected={selection}
                                        on:click={selectObject(b.uuid)}
                                        on:keydown={altDown}
                                        objID={b.uuid}
                                        {gridStep}
                                        {gridMax}
                                    />
                                {:else if b.kind === "vector"}
                                    <Vector
                                        {scene}
                                        render={requestFrameIfNotRequested}
                                        onClose={() => objects = removeObject(b.uuid, objects)}
                                        onUpdate={() => objects = updateObject(b, objects)}
                                        params={b.params}
                                        bind:selected={selection}
                                        on:click={selectObject(b.uuid)}
                                        on:keydown={altDown}
                                        objID={b.uuid}
                                        {gridStep}
                                        {gridMax}
                                    />
                                {/if}
                            </div>
                        {/each}
                    </div>
                </div>
                <button class="btn btn-sm btn-light mt-1 raise-lower-button d-flex justify-content-center"
                        title="Raise/Lower window"
                        on:click={() => {
                    shadeUp = !shadeUp;
                    }}>
                    <i class={`bi bi-chevron-${shadeUp ? 'down' : 'up'}`}></i>
                </button>
            </div>
        </div>
    </div>

    <div class="settings-tray">
        <Settings
            {scene}
            {camera}
            {controls}
            {controls2}
            bind:gridMax
            bind:gridStep
            {gridMeshes}
            {axesText}
            {axesHolder}
            {lineMaterial}
            {axesMaterial}
            bind:objects
            bind:socket
            bind:pollResponses
            encode={makeQueryStringObject}
            render={requestFrameIfNotRequested}
            bind:update={scaleUpdate}
            bind:animation={scaleAnimation}
            bind:orthoCamera
            bind:currentMode
            on:animate={animateIfNotAnimating}
            />
    </div>
</main>
<style>
    canvas {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        margin: 0;
    }

    .info {
        position: absolute;
        left: 3%;
        top: 0%;
        width: clamp(23ch, 50%, 75ch);
        background-color: transparent;
        perspective: 1000px; /* Remove this if you don't want the 3D effect */
    }

    .info-inner {
        position: relative;
        width: 100%;
        height: 100%;
        transition: transform 0.8s, opacity 0.8s;
        transform-style: preserve-3d;
    }

    .info.flipInfo .info-inner {
        transform: rotateY(180deg);
    }

    .chapterBox,
    .objectBoxOuter {
        position: absolute;
        width: 100%;
        height: fit-content;
        background-color: rgba(0, 0, 0, 0.6);
        border-radius: 0.5em;
        border-top-left-radius: 0rem;
        border-top-right-radius: 0rem;
        border: 1px solid black;
        padding: 5px;
        transition: opacity 0.8s;
        box-sizing: border-box;

        -webkit-backface-visibility: hidden;
        backface-visibility: hidden;
    }

    .chapterBox {
        text-align: unset;
    }

    .objectBoxOuter {
        transform: rotateY(180deg);
        opacity: 0;
    }

    .info.flipInfo .chapterBox {
        opacity: 0;
    }

    .info.flipInfo .objectBoxOuter {
        opacity: 1;
    }

    .objectBoxInner {
        display: flex;
        flex-direction: column-reverse;
        max-height: 75vh;
        overflow-y: auto;
        gap: 0.25em;
    }

    .dropdown {
        position: relative;
        display: inline-block;
    }

    .object-box-title {
        display: flex;
        font-size: 1.5em;
        justify-content: space-between;
    }

    .settings-tray {
        position: fixed;
        left: 3px;
        bottom: 0;
        display: flex;
        justify-content: flex-start;
        width: clamp(23ch, 30%, 45ch);
    }

    .raise-lower-button {
        width: clamp(50px, 20%, 200px);
        margin: 0 auto;

        /* Bold */
        -webkit-text-stroke: 1px;
    }
</style>
