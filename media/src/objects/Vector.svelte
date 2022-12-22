<script>
    import { onMount, onDestroy } from "svelte";
    import M from "../M.svelte";

    import * as THREE from "three";

    import { create, all } from "mathjs";

    import {
        ArrowBufferGeometry
    } from "../utils.js";

    const config = {};
    const math = create(all, config);

    // export let paramString;

    export let params = {
        a: "-1",
        b: "1",
        c: "2",
        x: "0",
        y: "0",
        z: "0",
        nX: 30,
        show: true,
    };

    if (!Object.hasOwn(params, 'show')) {
        params.show = true;
    }

    export let scene;
    export let render = () => {};
    export let onClose = () => {};
    export let onUpdate = () => {};
    export let gridStep;
    export let selected;
    export let objID;
    // export let gridMax;

    let hidden = false;

    const arrowMaterial = new THREE.MeshPhongMaterial({
        color: 0xaa0000,
        shininess: 80,
        side: THREE.DoubleSide,
        vertexColors: false,
        transparent: false,
        opacity: 0.7,
    });
    const vfScale = gridStep * 5;
    const arrowArgs = {
        radiusTop: vfScale / 24,
        radiusBottom: vfScale / 75,
        heightTop: vfScale / 8,
    };

    let arrow;

    const updateCurve = function() {
        const { a, b, c, x, y, z } = params;

        const [A, B, C, X, Y, Z] = math
              .parse([a, b, c, x, y, z])
              .map((item) => item.evaluate());

        const v = new THREE.Vector3(A, B, C);

        let geometry = new ArrowBufferGeometry({
            ...arrowArgs,
            height: v.length(),
        });

        if (arrow) {
            arrow.geometry.dispose();
            arrow.geometry = geometry;
        } else {
            arrow = new THREE.Mesh(geometry, arrowMaterial);
            scene.add(arrow);
            // colorBufferVertices( tube, (x,y,z) => blueUpRedDown(1));
        }
        arrow.position.set(X, Y, Z);

        arrow.lookAt(v.add(arrow.position));

        render();
    };

    // call updateCurve() when params change
    $: params && updateCurve();

    // Exercises
    //

    onMount(updateCurve);
    onDestroy(() => {
        if (arrow) {
            arrow.geometry && arrow.geometry.dispose();
            arrow.material && arrow.material.dispose();
        }
        scene.remove(arrow);
        render();
    });
</script>

<div class={'boxItem' + (selected === objID ? ' selected': '')} hidden={!params.show} on:click on:keydown>
    <div class="box-title">
        <span><strong>Vector</strong> <M size="sm">\langle a, b, c \rangle</M></span>
        <span>
            <button
                on:click={() => { hidden = !hidden; }}
                aria-label="Minimize Object Window"
            >
                <i class="fa fa-window-minimize" />
            </button>
            <button
                on:click={onClose}
                aria-label="Delete Object"
            >
                <i class="fa fa-window-close" />
            </button>
        </span>
    </div>
    <div hidden={hidden}>
        <div class="container">
            <span class="box-1"><M size="sm">a =</M></span>
            <input
                type="text"
                bind:value={params.a}
                on:change={() => {
            onUpdate()
            updateCurve()
            }}
                class="box box-2"
                />
            <span class="box-1"><M size="sm">b =</M></span>
            <input
                type="text"
                bind:value={params.b}
                on:change={() => {
            onUpdate()
            updateCurve()
            }}
                class="box box-2"
                />
            <span class="box-1"><M size="sm">c =</M></span>
            <input
                type="text"
                bind:value={params.c}
                on:change={() => {
            onUpdate()
            updateCurve()
            }}
                class="box box-2"
                />

            Plot at position <M size="sm">(p_1, p_2, p_3)</M>:

            <span class="box-1"><M size="sm">p_1 =</M></span>
            <input
                type="text"
                bind:value={params.x}
                on:change={() => {
            onUpdate()
            updateCurve()
            }}
                class="box box-2"
                />

            <span class="box-1"><M size="sm">p_2 =</M></span>
            <input
                type="text"
                bind:value={params.y}
                on:change={() => {
            onUpdate()
            updateCurve()
            }}
                class="box box-2"
                />
            <span class="box-1"><M size="sm">p_3 =</M></span>
            <input
                type="text"
                bind:value={params.z}
                on:change={() => {
            onUpdate()
            updateCurve()
            }}
                class="box box-2"
                />

            <span class="box-1">Scale</span>
            <input
                type="range"
                bind:value={params.nX}
                min="10"
                max="60"
                step="5"
                on:input={() => {
            onUpdate()
            updateCurve()
            }}
                class="box box-2"
                />
        </div>
    </div>
</div>

<style>
    .container {
        display: grid;

        grid-template-columns: 1fr auto 1fr;
        grid-template-rows: auto;

        grid-gap: 10px 15px;

        padding: 10px;
    }

    .box-1 {
        text-align: right;
        grid-column: 1 / 2;
        color: white;
        vertical-align: middle;
    }

    .box-2 {
        grid-column-start: 2;
        grid-column-end: 4;
    }
</style>
