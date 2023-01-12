<script>
    import { onDestroy } from "svelte";
    import M from "../M.svelte";

    import * as THREE from "three";

    import { create, all } from "mathjs";

    import {updateParams} from './levelWorker.js';

    import {
        marchingCubes,
        ArrowBufferGeometry
    } from "../utils.js";

    const config = {};
    const math = create(all, config);

    export let params = {
        g: "x^2 - y^2 + z^2",
        k: "1",
        a: "-2",
        b: "2",
        c: "-2",
        d: "2",
        e: "-2",
        f: "2",
    };

    let oldParams = params;

    $: {
        if (
            oldParams.g !== params.g
                || oldParams.k !== params.k
                || oldParams.a !== params.a
                || oldParams.b !== params.b
                || oldParams.a !== params.a
                || oldParams.c !== params.c
                || oldParams.d !== params.d
                || oldParams.e !== params.e
                || oldParams.f !== params.f
        ) {
            updateLevel();
            oldParams.g = params.g;
            oldParams.k = params.k;
            oldParams.a = params.a;
            oldParams.b = params.b;
            oldParams.c = params.c;
            oldParams.d = params.d;
            oldParams.e = params.e;
            oldParams.f = params.f;
        }
    }

    export let scene;
    export let shadeUp;
    export let render = () => {};
    export let onClose = () => {};
    export let onUpdate = () => {};
    export let selected;
    export let objID;

    export let camera,
    controls,
    // animation = false,
    gridStep;
    // showLevelCurves = false;

    let hidden = false;
    let loading = false;

    const geometry = new THREE.BufferGeometry();
    const xTraceGeometry = new THREE.BufferGeometry();
    const yTraceGeometry = new THREE.BufferGeometry();
    const zTraceGeometry = new THREE.BufferGeometry();

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

    const whiteLineMaterial = new THREE.LineBasicMaterial({
        color: 0xffffff,
        linewidth: 2,
    });

    const mesh = new THREE.Object3D();
    mesh.add(new THREE.Mesh(geometry, plusMaterial));
    mesh.add(new THREE.Mesh(geometry, minusMaterial));
    mesh.visible = false;
    scene.add(mesh);

    const updateLevel = function() {
        loading = true;
        updateParams(params).then((data) => {
            levelWorkerSuccessHandler(data);
        });
    }

    updateLevel();

    const levelWorkerSuccessHandler = (data) => {
        const { normals, vertices, xpts, ypts, zpts } = data;
        geometry.setAttribute(
            "position",
            new THREE.Float32BufferAttribute(vertices, 3)
        );
        geometry.setAttribute(
            "normal",
            new THREE.Float32BufferAttribute(normals, 3)
        );

        mesh.visible = true;

        {
            xTraceGeometry.setAttribute(
                "position",
                new THREE.Float32BufferAttribute(xpts, 3)
            );

            const trace = new THREE.LineSegments(xTraceGeometry, whiteLineMaterial);

            trace.rotation.y = Math.PI / 2;
            trace.rotation.z = Math.PI / 2;

            mesh.add(trace);
        }
        {
            yTraceGeometry.setAttribute(
                "position",
                new THREE.Float32BufferAttribute(ypts, 3)
            );

            const trace = new THREE.LineSegments(yTraceGeometry, whiteLineMaterial);

            trace.rotation.x = -Math.PI / 2;
            trace.rotation.z = -Math.PI / 2;

            mesh.add(trace);
        }
        {
            zTraceGeometry.setAttribute(
                "position",
                new THREE.Float32BufferAttribute(zpts, 3)
            );

            const trace = new THREE.LineSegments(zTraceGeometry, whiteLineMaterial);

            mesh.add(trace);
        }

        point.position.set(0, 0, 0);

        tangentVectors({ point });

        loading = false;
        render();
    };

    onDestroy(() => {
        geometry.dispose();
        plusMaterial.dispose();
        minusMaterial.dispose();
        scene.remove(mesh);
        scene.remove(tanFrame);
        render();
    });

    // Select a point
    const tanFrame = new THREE.Object3D();
    const arrows = {
        u: new THREE.Mesh(),
        v: new THREE.Mesh(),
        n: new THREE.Mesh(),
    };
    const ruColors = { u: 0x992525, v: 0x252599, n: 0xb6b6b6 };
    for (let key of Object.keys(arrows)) {
        arrows[key].material = new THREE.MeshBasicMaterial({
            color: ruColors[key],
        });
        tanFrame.add(arrows[key]);
    }

    const pointMaterial = new THREE.MeshLambertMaterial({ color: 0xffff33 });
    const point = new THREE.Mesh(
        new THREE.SphereGeometry(0.2 / 8, 16, 16),
        pointMaterial
    );

    tanFrame.add(point);

    const shardMaterial = new THREE.MeshPhongMaterial({
        color: 0x4b4b4b,
        shininess: 80,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.5,
    });
    const planeShard = new THREE.Mesh(new THREE.BufferGeometry(), shardMaterial);
    tanFrame.add(planeShard);

    tanFrame.visible = false;

    scene.add(tanFrame);

    const nFrame = function({
        f = (x, y, z) => math.evaluate(params.g, { x, y, z }),
        point = point,
        eps = 1e-4,
    } = {}) {
        const [u, v, w] = [point.position.x, point.position.y, point.position.z];

        let h;

        h = Math.max(u * eps, (2 * eps) ** 2);
        const fx = (f(u + h / 2, v, w) - f(u - h / 2, v, w)) / h;
        h = Math.max(v * eps, (2 * eps) ** 2);
        const fy = (f(u, v + h / 2, w) - f(u, v - h / 2, w)) / h;
        h = Math.max(w * eps, (2 * eps) ** 2);
        const fz = (f(u, v, w + h / 2) - f(u, v, w - h / 2)) / h;

        return { p: point.position, n: new THREE.Vector3(fx, fy, fz) };
    }

    // Construct tangent vectors at a point u,v (both 0 to 1)
    const tangentVectors = function({ point, eps = 1e-4, plane = true } = {}) {
        const { p, n } = nFrame({
            point,
            eps,
        });

        // point.position.copy(dr.p);

        const arrowParams = {
            radiusTop: gridStep / 10,
            radiusBottom: gridStep / 20,
            heightTop: gridStep / 7,
        };

        const arrow = arrows.n;
        const pos = p.clone();
        arrow.position.copy(pos);
        if (arrow.geometry) arrow.geometry.dispose();
        arrow.geometry = new ArrowBufferGeometry({
            ...arrowParams,
            height: n.length() / 8,
        });
        arrow.lookAt(pos.add(n));

        planeShard.geometry.dispose();

        if (plane) {
            const { a, b, c, d, e, f } = params;

            const tangentPlaneGeometry = marchingCubes({
                f: (x, y, z) => {
                    const xVec = new THREE.Vector3(x, y, z);
                    return xVec.dot(n);
                },
                level: n.dot(p) + 1e-8, // offset to avoid overlap artifacts
                xMin: math.evaluate(String(a)),
                xMax: math.evaluate(String(b)),
                yMin: math.evaluate(String(c)),
                yMax: math.evaluate(String(d)),
                zMin: math.evaluate(String(e)),
                zMax: math.evaluate(String(f)),
                N: 2,
            });

            planeShard.geometry = tangentPlaneGeometry;
        }
    }

    const raycaster = new THREE.Raycaster();

    let mouseVector = new THREE.Vector2();

    const onMouseMove = function(e) {
        // normalized mouse coordinates
        mouseVector.x = 2 * (e.clientX / window.innerWidth) - 1;
        mouseVector.y = 1 - 2 * (e.clientY / window.innerHeight);

        raycaster.setFromCamera(mouseVector, camera);

        const intersects = raycaster.intersectObjects(
            [mesh.children[0], mesh.children[1]],
            true
        );

        if (intersects.length > 0) {
            const intersect = intersects[0];
            point.position.x = intersect.point.x;
            point.position.y = intersect.point.y;
            point.position.z = intersect.point.z;

            tangentVectors({ point });

            render();
        }
    }

    const shiftDown = (e) => {
        if (shadeUp && selected === objID) {
            switch (e.key) {
                case "Shift":
                    window.addEventListener("mousemove", onMouseMove, false);
                    break;
                case "c":
                    controls.target.set(
                        point.position.x,
                        point.position.y,
                        point.position.z
                    );
                    render();
                    break;
                case "t":
                    tanFrame.visible = !tanFrame.visible;
                    render();
                    break;
                case "y":
                    planeShard.visible = !planeShard.visible;
                    render();
                    break;
                case "n":
                    arrows.n.visible = !arrows.n.visible;
                    render();
                    break;
            }
        }
    };

    const shiftUp = (e) => {
        if (e.key === "Shift") {
            window.removeEventListener("mousemove", onMouseMove);
        }
    };

    window.addEventListener("keydown", shiftDown, false);
    window.addEventListener("keyup", shiftUp, false);
</script>

<div class={'boxItem' + (selected === objID ? ' selected': '')} on:click on:keydown>
    <div class="box-title">
        <span>
            <strong>Level surface </strong>
            <span class:hidden={!loading}
                  ><i class="fa fa-spinner fa-pulse fa-fw" />
                <span class="sr-only">Loading...</span>
            </span>
        </span>
        <span
            ><button class="btn"
                 on:click={() => {
                hidden = !hidden;
                }}><i class="fa fa-window-minimize" /></button>
            <button class="btn" on:click={onClose}>
                <i class="fa fa-window-close" /></button></span>
    </div>
    <div hidden={hidden}>
        <div class="container">
            <span class="box-1"><M size="sm">g(x,y,z) =</M></span>
            <input
                type="text"
                bind:value={params.g}
                on:change={() => {
            onUpdate();
            updateLevel();
            }}
                class="box box-2"
                />
            <span class="box-1"><M size="sm">k =</M></span>
            <input
                type="number"
                bind:value={params.k}
                on:change={() => {
            onUpdate();
            updateLevel();
            }}
                class="box box-2"
                />

            <input
                type="number"
                bind:value={params.a}
                on:change={() => {
            onUpdate();
            updateLevel();
            }}
                class="box"
                />
            <span class="box box-3"><M size="sm">\leq x \leq</M></span>
            <input
                type="number"
                bind:value={params.b}
                on:change={() => {
            onUpdate();
            updateLevel();
            }}
                class="box"
                />
            <input
                type="number"
                bind:value={params.c}
                on:change={() => {
            onUpdate();
            updateLevel();
            }}
                class="box"
                />
            <span class="box box-3"><M size="sm">\leq y \leq</M></span>
            <input
                type="number"
                bind:value={params.d}
                on:change={() => {
            onUpdate();
            updateLevel();
            }}
                class="box"
                />
            <input
                type="number"
                bind:value={params.e}
                on:change={() => {
            onUpdate();
            updateLevel();
            }}
                class="box"
                />
            <span class="box box-3"><M size="sm">\leq z \leq</M></span>
            <input
                type="number"
                bind:value={params.f}
                on:change={() => {
            onUpdate();
            updateLevel();
            }}
                class="box"
                />

            <span class="box-1">Tangents</span>
            <label class="switch box box-2">
                <input
                    type="checkbox"
                    value="false"
                    name="frameVisible"
                    id="frameVisible"
                    bind:checked={tanFrame.visible}
                    on:change={render}
                    />
                <span class="slider round" />
            </label>
        </div>
    </div>
</div>
