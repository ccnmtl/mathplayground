<script>
    import { onMount, onDestroy, createEventDispatcher } from "svelte";
    import M from "../M.svelte";

    import * as THREE from "three";

    import { create, all } from "mathjs";

    import {
        ArrowBufferGeometry,
        rk4,
        norm1,
    } from "../utils.js";

    const config = {};
    const math = create(all, config);

    const dispatch = createEventDispatcher();

    export let params = {
        p: "x",
        q: "y",
        r: "-z",
        nVec: 6,
    };

    let oldParams = params;

    $: {
        if (
            oldParams.p !== params.p
                || oldParams.q !== params.q
                || oldParams.r !== params.r
                || oldParams.nVec !== params.nVec
        ) {
            updateField();
            oldParams.p = params.p;
            oldParams.q = params.q;
            oldParams.r = params.r;
            oldParams.nVec = params.nVec;
        }
    }

    $: nCubed = Math.pow(params.nVec, 3);

    export let scene;
    export let update = () => {};
    export let render = () => {};
    export let animation = false;
    export let onClose = () => {};
    export let onUpdate = () => {};
    export let gridMax, gridStep;
    export let selected;
    export let objID;

    let hidden = false;
    let flowTrails = true;
    let stopButton, rewButton;

    class FlowArrowMesh extends THREE.Mesh {
        constructor(geometry, material, lim = 1) {
            super(geometry, material);

            this.start = new THREE.Vector3();
            this.lim = lim;
            this.lastPosition = null;
        }

        initiate(F, dt = 0.01, maxSteps = 500, tol = 1e-3) {
            const vec = new THREE.Vector3();
            const vec1 = new THREE.Vector3();
            vec.copy(this.position);
            for (let i = 0; i < maxSteps; i++) {
                vec1.set(...rk4(vec.x, vec.y, vec.z, F, -dt));
                if (vec.clone().sub(vec1).length() < tol * this.lim) {
                    return this.start.copy(vec1);
                } else {
                    if (norm1(vec1) > this.lim) {
                        return this.start.copy(vec);
                    }
                }
                vec.copy(vec1);
            }
            return this.start.copy(vec1);
        }
    }

    const flowArrows = new THREE.Object3D();
    const fieldMaterial = new THREE.MeshLambertMaterial({ color: 0x373765 });
    const trailMaterial = new THREE.LineBasicMaterial({
        color: 0xffffff,
        vertexColors: true,
    });

    const trailGeometry = new THREE.BufferGeometry();

    const MAX_TRAIL_LENGTH = 250;
    let trailLength = 0;

    const trails = new THREE.LineSegments(trailGeometry, trailMaterial);
    const arrowGeometries = [],
          heightResolution = 150,
          vfScale = gridStep * 5;
    const arrowArgs = {
        radiusTop: vfScale / 30,
        radiusBottom: vfScale / 100,
        heightTop: vfScale / 8,
    };

    const setTrailColors = function(colorArray, start, total = MAX_TRAIL_LENGTH) {
        let index = 0,
            i = 0;

        while (index < colorArray.length) {
            for (let j = 0; j < nCubed; j++) {
                colorArray[(start + index++) % colorArray.length] = 1 - i / total; // red
                colorArray[(start + index++) % colorArray.length] = 1 - i / total; // green
                colorArray[(start + index++) % colorArray.length] = 1; // blue

                colorArray[(start + index++) % colorArray.length] = 1 - (i + 1) / total; // red
                colorArray[(start + index++) % colorArray.length] = 1 - (i + 1) / total; // green
                colorArray[(start + index++) % colorArray.length] = 1; // blue
            }
            i += 1;
        }
    }

    // Make a fixed set of different arrow lengths instead of regenerating each time.
    for (let i = 1; i <= heightResolution; i++) {
        const geometry = new ArrowBufferGeometry({
            radiusBottom: vfScale / 100,
            height: (i / heightResolution) * vfScale / 2,
            heightTop: Math.min(((i / heightResolution) * vfScale) / 3, vfScale / 8),
            radiusTop: Math.min(vfScale / 30, ((i / heightResolution) * vfScale) / 6),
        });
        arrowGeometries.push(geometry);
    }

    const initFlowArrows = function(arrows, lim = gridMax, N = params.nVec) {
        const vec = new THREE.Vector3();
        let maxLength = 0;
        const arrowDefaultGeometry = new ArrowBufferGeometry({
            ...arrowArgs,
            height: gridStep / gridMax,
        });

        for (let i = 0; i < N; i++) {
            for (let j = 0; j < N; j++) {
                for (let k = 0; k < N; k++) {
                    const arrow = new FlowArrowMesh(
                        arrowDefaultGeometry,
                        fieldMaterial,
                        1.2 * lim
                    );
                    arrow.scale.set(gridMax, gridMax, gridMax);
                    arrow.position.set(
                        (((i + 1 / 2) * 2) / N - 1) * lim + 0.01 * Math.random(),
                        (((j + 1 / 2) * 2) / N - 1) * lim + 0.01 * Math.random(),
                        (((k + 1 / 2) * 2) / N - 1) * lim + 0.01 * Math.random()
                    );
                    arrow.initiate(fieldF);
                    // const posr = new THREE.Vector3();

                    fieldF(arrow.position.x, arrow.position.y, arrow.position.z, vec);
                    const len = vec.length();
                    maxLength = Math.max(maxLength, len);

                    vec.add(arrow.position);
                    arrow.lookAt(vec);
                    arrows.add(arrow);
                }
            }
        }

        trailLength = 0;
        const trailPoints = new Float32Array(MAX_TRAIL_LENGTH * 2 * 3 * nCubed);
        trails.geometry.setAttribute(
            "position",
            new THREE.Float32BufferAttribute(trailPoints, 3)
        );

        const trailColors = new Float32Array(MAX_TRAIL_LENGTH * 2 * 3 * nCubed);
        setTrailColors(trailColors, 0);
        trails.geometry.setAttribute(
            "color",
            new THREE.Float32BufferAttribute(trailColors, 3)
        );

        trails.geometry.setDrawRange(0, trailLength);

        return maxLength; //
    }

    const updateFlowArrows = function(arrows, F, dt = 0.016) {
        const vec = new THREE.Vector3();
        let index;

        const trailPoints = trails.geometry.attributes.position.array;

        if (flowTrails) {
            index = (trailLength % MAX_TRAIL_LENGTH) * 2 * 3 * nCubed;
            setTrailColors(
                trails.geometry.attributes.color.array,
                trailLength * nCubed * 6,
                MAX_TRAIL_LENGTH
            );
            trails.geometry.attributes.color.needsUpdate = true;
            trailLength++;
        }

        arrows.children.forEach((arrow) => {
            const { x, y, z } = arrow.position;

            const pos1 = new THREE.Vector3();
            pos1.set(...rk4(x, y, z, F, dt));

            if (flowTrails) {
                trailPoints[index++] = arrow.position.x;
                trailPoints[index++] = arrow.position.y;
                trailPoints[index++] = arrow.position.z;

                trailPoints[index++] = pos1.x;
                trailPoints[index++] = pos1.y;
                trailPoints[index++] = pos1.z;
            }

            if (
                norm1(pos1) > arrow.lim ||
                    (dt > 1e-6 && pos1.clone().sub(arrow.position).length() < 1e-3)
            ) {
                arrow.position.copy(
                    arrow.start
                        .clone()
                        .add(
                            new THREE.Vector3(
                                Math.random() * 0.01,
                                Math.random() * 0.01,
                                Math.random() * 0.01
                            )
                        )
                );
            } else {
                arrow.position.copy(pos1);
            }

            let height = F(
                arrow.position.x,
                arrow.position.y,
                arrow.position.z,
                vec
            ).length();
            height = Math.round((height / maxLength) * heightResolution) - 1;

            arrow.geometry =
                arrowGeometries[
                    Math.max(0, Math.min(arrowGeometries.length - 1, height))
                ];

            arrow.lookAt(vec.add(arrow.position));
        });

        trails.geometry.setDrawRange(0, trailLength * 2 * nCubed);
        trails.geometry.attributes.position.needsUpdate = true;
    }

    const freeChildren = function(objectHolder) {
        for (let i = objectHolder.children.length - 1; i >= 0; i--) {
            const element = objectHolder.children[i];
            if (element.geometry.dispose) element.geometry.dispose();
            objectHolder.remove(element);
        }
    }

    const freeTrails = function() {
        trailLength = 0;
    }

    let fieldF;

    const updateField = function() {
        const { p, q, r } = params;
        const [P, Q, R] = [p, q, r].map((x) => math.parse(x).compile());
        fieldF = (x, y, z, vec) => {
            vec.set(
                P.evaluate({ x, y, z }),
                Q.evaluate({ x, y, z }),
                R.evaluate({ x, y, z })
            );
            return vec;
        };
    }

    let maxLength = 2;
    scene.add(flowArrows);
    scene.add(trails);

    onMount(() => {
        updateField();
        initFlowArrows(flowArrows);
        updateFlowArrows(flowArrows, fieldF, 0);
        render();
    });
    onDestroy(() => {
        freeChildren(flowArrows);
        freeChildren(trails);
        scene.remove(flowArrows);
        scene.remove(trails);
        trails.geometry.setAttribute(
            "position",
            new THREE.Float32BufferAttribute([], 3)
        );

        freeTrails();
        render();
    });

    // Set update function for animation
    update = (dt) => {
        updateFlowArrows(flowArrows, fieldF, dt);
    };
</script>

<div class={'boxItem' + (selected === objID ? ' selected': '')} on:click on:keydown>
    <div class="box-title">
        <strong>Vector Field</strong>
        <span
            ><button
                 on:click={() => {
                hidden = !hidden;
                }}><i class="fa fa-window-minimize" /></button
                                                          ><button on:click={onClose}>
                <i class="fa fa-window-close" /></button
                                                    ></span
                                                         >
    </div>
    <div hidden={hidden}>
        <div class="container">
            <span class="box-1"><M size="sm">P(x,y,z) =</M></span>
            <input
                type="text"
                bind:value={params.p}
                on:change={() => {
            onUpdate();
            updateField();
            }}
                class="box box-2"
                />
            <span class="box-1"><M size="sm">Q(x,y,z) =</M></span>
            <input
                type="text"
                bind:value={params.q}
                on:change={() => {
            onUpdate();
            updateField();
            }}
                class="box box-2"
                />
            <span class="box-1"><M size="sm">R(x,y,z) =</M></span>
            <input
                type="text"
                bind:value={params.r}
                on:change={() => {
            onUpdate();
            updateField();
            }}
                class="box box-2"
                />
            <span class="box-1">Resolution</span>
            <input
                type="range"
                bind:value={params.nVec}
                on:change={onUpdate}
                min="1"
                max="10"
                step="1"
                class="box box-2"
                on:input={() => {
            rewButton.click();
            }}
            />
            <span class="box-1">Trails</span>
            <label class="switch box box-2">
                <input
                    type="checkbox"
                    name="trailsVisible"
                    id="trailsVisible"
                    bind:checked={flowTrails}
                    on:change={freeTrails}
                />
                <span class="slider round" />
            </label>

        <button
            on:click={() => {
            flowArrows.visible = true;
            animation = !animation;
            if (animation) dispatch("animate");
            }}
            class="box-1"
            >
            {#if !animation}
                <i class="fa fa-play" />
            {:else}
                <i class="fa fa-pause" />
            {/if}
        </button>
        <button
            on:click={() => {
            animation = false;
            flowArrows.visible = false;
            freeTrails();
            freeChildren(flowArrows);
            maxLength = initFlowArrows(flowArrows, gridMax, params.nVec);
            render();
            }}
            class="box-3"
            bind:this={stopButton}
            >
            <i class="fa fa-stop" />
        </button>
        <button
            on:click={() => {
            freeChildren(flowArrows);
            maxLength = initFlowArrows(flowArrows, gridMax, params.nVec);
            updateFlowArrows(flowArrows, fieldF, 0);
            freeTrails();
            render();
            }}
            class="box-4"
            bind:this={rewButton}
            >
            <i class="fa fa-fast-backward" />
        </button>
    </div>
</div>
</div>

