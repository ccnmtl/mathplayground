<script>
    import {onMount, onDestroy} from 'svelte';
    import M from "../M.svelte";

    import * as THREE from "three";

    import { create, all } from "mathjs";

    const config = {};
    const math = create(all, config);

    import {
        lcm, marchingSegments, ParametricGeometry
    } from "../utils.js";

    export let params = {
        a: "-2",
        b: "2",
        c: "-2",
        d: "1 + sin(pi*u)/2",
        x: "u",
        y: "v",
        z: "1 - sin(u^2 - v^2)/2",
        rNum: 10,
        cNum: 10,
        nX: 30,
    };

    let oldParams = params;

    // See Curve.svelte for explanation of this stuff
    $: {
        if (
            oldParams.a !== params.a
                || oldParams.b !== params.b
                || oldParams.a !== params.a
                || oldParams.c !== params.c
                || oldParams.d !== params.d
                || oldParams.x !== params.x
                || oldParams.y !== params.y
                || oldParams.z !== params.z
        ) {
            updateSurface();
            oldParams.a = params.a;
            oldParams.b = params.b;
            oldParams.c = params.c;
            oldParams.d = params.d;
            oldParams.x = params.x;
            oldParams.y = params.y;
            oldParams.z = params.z;
        }
    }

    export let scene;
    export let render = () => {};
    export let onClose = () => {};
    export let onUpdate = () => {};
    export let selected;
    export let objID;

    let hidden = false;

    params["rNum"] = 10;
    params["cNum"] = 10;
    params["nX"] = 60;

    const whiteLineMaterial = new THREE.LineBasicMaterial({
        color: 0xffffff,
        linewidth: 2,
    });

    whiteLineMaterial.polygonOffset = true;
    whiteLineMaterial.polygonOffsetFactor = 0.1;

    const minusMaterial = new THREE.MeshPhongMaterial({
        color: 0xff3232,
        shininess: 80,
        side: THREE.BackSide,
        vertexColors: false,
        transparent: true,
        opacity: 0.7,
    });
    const plusMaterial = new THREE.MeshPhongMaterial({
        color: 0x3232ff,
        shininess: 80,
        side: THREE.FrontSide,
        vertexColors: false,
        transparent: true,
        opacity: 0.7,
    });

    let cMin, dMax; // make these globals as useful for tangents.

    const tol = 1e-12 //tolerance for comparisons

    let surfaceMesh;

    const updateSurface = function() {
        const { a, b, c, d, x, y, z } = params;
        const A = math.parse(a).evaluate(),
              B = math.parse(b).evaluate();
        const [C, D, X, Y, Z] = math.parse([c, d, x, y, z]);
        const geometry = new ParametricGeometry(
            (u, v, vec) => {
                const U = A + (B - A) * u;
                const uv = {
                    u: U,
                    v: (1 - v) * C.evaluate({ u: U }) + v * D.evaluate({ u: U }),
                };
                vec.set(X.evaluate(uv), Y.evaluate(uv), Z.evaluate(uv));
            },
            params.nX || 30,
            params.nX || 30
        );
        const meshGeometry = meshLines(params, params.rNum, params.cNum, params.nX);
        let material = plusMaterial;

        if (surfaceMesh) {
            for (let i = 0; i < surfaceMesh.children.length; i++) {
                const mesh = surfaceMesh.children[i];
                mesh.geometry.dispose();
                mesh.geometry = i < 2 ? geometry : meshGeometry;
                if (i < 1) {
                    mesh.material = material;
                }

                if (i === 1) {
                    mesh.visible = true;
                }
            }
        } else {
            surfaceMesh = new THREE.Object3D();
            const backMesh = new THREE.Mesh(geometry, minusMaterial);
            const frontMesh = new THREE.Mesh(geometry, material);
            // mesh.add(new THREE.Mesh( geometry, wireMaterial ))
            surfaceMesh.add(frontMesh);
            surfaceMesh.add(backMesh);
            surfaceMesh.add(new THREE.LineSegments(meshGeometry, whiteLineMaterial));
            // mesh.visible = false;
            scene.add(surfaceMesh);
        }
        render();
    }

    const meshLines = function(rData, rNum = 10, cNum = 10, nX = 30) {
        let { a, b, c, d, x, y, z } = rData;
        // const N = lcm(lcm(rNum, cNum), nX);
        const A = math.parse(a).evaluate(),
              B = math.parse(b).evaluate();
        [c, d, x, y, z] = math.parse([c, d, x, y, z]);

        const du = (B - A) / rNum;
        const dx = (B - A) / lcm(nX, cNum);
        const points = [];
        for (let u = A; u <= B; u += du) {
            const C = c.evaluate({ u: u }),
                  D = d.evaluate({ u: u });
            const dy = (D - C) / lcm(nX, rNum);
            const params = { u: u, v: C };
            points.push(
                new THREE.Vector3(
                    x.evaluate(params),
                    y.evaluate(params),
                    z.evaluate(params)
                )
            );
            for (let v = C + dy; v < D; v += dy) {
                params.v = v;
                points.push(
                    new THREE.Vector3(
                        x.evaluate(params),
                        y.evaluate(params),
                        z.evaluate(params)
                    )
                );
                points.push(
                    new THREE.Vector3(
                        x.evaluate(params),
                        y.evaluate(params),
                        z.evaluate(params)
                    )
                );
            }
            params.v = D;
            points.push(
                new THREE.Vector3(
                    x.evaluate(params),
                    y.evaluate(params),
                    z.evaluate(params)
                )
            );
        }

        // v-Meshes
        const params = { u: A };
        (cMin = c.evaluate(params)), (dMax = d.evaluate(params));
        for (let u = A + dx; u <= B; u += dx) {
            params.u = u;
            cMin = Math.min(cMin, c.evaluate(params));
            dMax = Math.max(dMax, d.evaluate(params));
        }

        for (let v = cMin; v <= dMax; v += (dMax - cMin) / cNum) {
            const zs = marchingSegments(
                (x) => (c.evaluate({ u: x }) - v) * (v - d.evaluate({ u: x })),
                A,
                B,
                nX
            );
            params.v = v;
            let nextZero = zs.shift();
            for (let u = A; u <= B - dx + tol; u += dx) {
                params.u = u;
                if (c.evaluate(params) <= v && v <= d.evaluate(params)) {
                    points.push(
                        new THREE.Vector3(
                            x.evaluate(params),
                            y.evaluate(params),
                            z.evaluate(params)
                        )
                    );
                    if (nextZero < u + dx) {
                        params.u = nextZero;
                        points.push(
                            new THREE.Vector3(
                                x.evaluate(params),
                                y.evaluate(params),
                                z.evaluate(params)
                            )
                        );
                        nextZero = zs.shift();
                    } else {
                        params.u = u + dx;
                        points.push(
                            new THREE.Vector3(
                                x.evaluate(params),
                                y.evaluate(params),
                                z.evaluate(params)
                            )
                        );
                    }
                } else {
                    if (nextZero < u + dx) {
                        params.u = nextZero;
                        points.push(
                            new THREE.Vector3(
                                x.evaluate(params),
                                y.evaluate(params),
                                z.evaluate(params)
                            )
                        );
                        nextZero = zs.shift();
                        params.u = u + dx;
                        points.push(
                            new THREE.Vector3(
                                x.evaluate(params),
                                y.evaluate(params),
                                z.evaluate(params)
                            )
                        );
                    }
                }
            }
        }

        const geometry = new THREE.BufferGeometry().setFromPoints(points);
        return geometry;

    }

    onMount(updateSurface);
    onDestroy(() => {
        for (const child of surfaceMesh.children) {
            child.geometry && child.geometry.dispose();
            child.material && child.material.dispose();
        }
        scene.remove(surfaceMesh)
        render()
    })
</script>

<div class={'boxItem' + (selected === objID ? ' selected': '')} on:click on:keydown>
    <div class="box-title">
        <strong>Parametric surface</strong> <span><button on:click={() => {hidden = !hidden}}><i class="fa fa-window-minimize"></i></button><button on:click={onClose}> <i class="fa fa-window-close"></i></button></span>
    </div>
    <div hidden={hidden}>
        <div class="container">
            <span class="box-1"><M size="sm">x(u,v) =</M></span>
            <input type="text" bind:value={params.x}
                   on:change={() => {
            onUpdate();
            updateSurface();
            }}
                   class="box box-2" />
            <span class="box-1"><M size="sm">y(u,v) =</M></span>
            <input type="text" bind:value={params.y}
                   on:change={() => {
            onUpdate();
            updateSurface();
            }}
                   class="box box-2" />
            <span class="box-1"><M size="sm">z(u,v) =</M></span>
            <input type="text" bind:value={params.z}
                   on:change={() => {
            onUpdate();
            updateSurface();
            }}
                   class="box box-2" />
            <input type="text" bind:value={params.a}
                   on:change={() => {
            onUpdate();
            updateSurface();
            }}
                   class="box" />
            <span class="box box-3"><M size="sm">\leq u \leq</M></span>

            <input type="text" bind:value={params.b}
                   on:change={() => {
            onUpdate();
            updateSurface();
            }}
                   class="box" />
            <input type="text" bind:value={params.c}
                   on:change={() => {
            onUpdate();
            updateSurface();
            }}
                   class="box" />
            <span class="box box-3"><M size="sm">\leq v \leq</M></span>
            <input type="text" bind:value={params.d}
                   on:change={() => {
            onUpdate();
            updateSurface();
            }}
                   class="box" />

            <span class="box-1"><M size="sm">u</M>-meshes</span>
            <input type="range" bind:value={params.rNum} min="0" max = "20" step="1" on:input={updateSurface} class="box box-2" />
            <span class="box-1"><M size="sm">v</M>-meshes</span>
            <input type="range" bind:value={params.cNum} min="0" max = "20" step="1" on:input={updateSurface} class="box box-2" />
            <span class="box-1">Resolution</span>
            <input type="range" bind:value={params.nX} min="10" max = "60" step="5" on:input={updateSurface} class="box box-2" />
        </div>
</div></div>

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

    .box-3 {
        color: white;
        vertical-align: middle;
        text-align: center;

        grid-column: 2 / 3;
    }

    .box-title {
        display: flex;
        justify-content: space-between;
        color: white;
        padding: .5em;
    }

    button {
        background-color: transparent;
        color:white;
        border: none;
    }

    button:hover {
        color: white;
    }

    button:active {
        color: gray;
    }
</style>
