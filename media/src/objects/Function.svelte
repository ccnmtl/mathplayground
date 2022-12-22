<script>
    import { onMount, onDestroy, createEventDispatcher } from "svelte";
    import {ParametricGeometry} from 'three/examples/jsm/geometries/ParametricGeometry.js';
    import M from "../M.svelte";
    import * as THREE from "three";
    import { create, all } from "mathjs";

    const config = {};
    const math = create(all, config);

    const dispatch = createEventDispatcher();

    import {
        lcm,
        marchingSegments,
        marchingSquares,
        ArrowBufferGeometry,
        blockGeometry,
    } from "../utils.js";

    export let camera,
    controls,
    animation = false,
    gridStep,
    showLevelCurves = false;

    export let params = {
        a: "-2",
        b: "2",
        c: "-2",
        d: "1 + sin(pi*x)/4",
        z: "exp(-(1 - (x^2 + y^2))^2)",
        t0: 0,
        t1: 0,
        color: "#3232ff",
    };

    let oldParams = params;

    // See Curve.svelte for explanation of this stuff
    $: {
        if (oldParams.color !== params.color) {
            updateColor();
            oldParams.color = params.color;
        }

        if (
            oldParams.a !== params.a
                || oldParams.b !== params.b
                || oldParams.a !== params.a
                || oldParams.c !== params.c
                || oldParams.d !== params.d
                || oldParams.z !== params.z
        ) {
            updateSurface();
            oldParams.a = params.a;
            oldParams.b = params.b;
            oldParams.c = params.c;
            oldParams.d = params.d;
            oldParams.z = params.z;
        }
    }

    let data = {
        rNum: 10,
        cNum: 10,
        nX: 50,
        nL: 16,
        N: 1,
        s: 0,
        t: 0,
        samp: 0,
        levelDelta: -1,
        shiftLevel: 0.0,
        tau: 0,
        animateTime: false,
    };

    if (!params.t0) {
        params.t0 = 0;
    }
    if (!params.t1) {
        params.t1 = 1;
    }
    if (!params.color) {
        params.color = "#3232ff";
    }

    export let scene;
    export let shadeUp;
    export let render = () => {};
    export let onClose = () => {};
    export let onUpdate = () => {};
    export let update = () => {};
    export let selected;
    export let objID;

    let hidden = false;

    const colorMaterial = new THREE.MeshPhongMaterial({
        color: 0xffffff,
        shininess: 80,
        side: THREE.FrontSide,
        vertexColors: true,
        transparent: false,
        opacity: 0.85,
    });
    const boxEdgeMaterial = new THREE.LineBasicMaterial({
        color: 0xffffff,
        linewidth: 2,
    });

    const pointMaterial = new THREE.MeshLambertMaterial({ color: 0xffff33 });
    const point = new THREE.Mesh(
        new THREE.SphereGeometry(gridStep / 8, 16, 16),
        pointMaterial
    );
    point.position.set(1, 1, 1);
    const arrows = {
        u: new THREE.Mesh(),
        v: new THREE.Mesh(),
        n: new THREE.Mesh(),
        grad: new THREE.Mesh(),
    };
    const ruColors = { u: 0x992525, v: 0x252599, grad: 0x259925, n: 0xb6b6b6 };
    for (let key of Object.keys(arrows)) {
        arrows[key].material = new THREE.MeshBasicMaterial({
            color: ruColors[key],
        });
        point.add(arrows[key]);
    }
    arrows.n.visible = false;
    arrows.grad.visible = false;

    const shardMaterial = new THREE.MeshPhongMaterial({
        color: 0x4b4b4b,
        shininess: 80,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.5,
    });
    const planeShard = new THREE.Mesh(undefined, shardMaterial);
    planeShard.visible = false;
    point.add(planeShard);

    point.visible = false;
    scene.add(point);

    const tangentVectors = function() {
        const arrowParams = {
            radiusTop: gridStep / 10,
            radiusBottom: gridStep / 20,
            heightTop: gridStep / 7,
        };

        const dx = 0.001,
              t = params.t0 + data.tau * (params.t1 - params.t0);

        const x = point.position.x;
        const y = point.position.y;
        const Z = math.parse(params.z).compile();

        const f0 = Z.evaluate({ x: x, y, t });

        const fx =
              (Z.evaluate({ x: x + dx, y, t }) - Z.evaluate({ x: x - dx, y, t })) /
              (2 * dx);
        const fy =
              (Z.evaluate({ x, y: y + dx, t }) - Z.evaluate({ x, y: y - dx, t })) /
              (2 * dx);

        for (let key of Object.keys(arrows)) {
            if (arrows[key].geometry.attributes.position !== undefined) {
                const geo = arrows[key].geometry;
                switch (key) {
                    case "u":
                        geo.adjustHeight(Math.sqrt(1 + fx * fx));
                        arrows[key].lookAt(1 + x, 0 + y, fx + f0);
                        break;

                    case "v":
                        geo.adjustHeight(Math.sqrt(1 + fy * fy));
                        arrows[key].lookAt(0 + x, 1 + y, fy + f0);
                        break;

                    case "n":
                        arrows[key].lookAt(-fx + x, -fy + y, 1 + f0);
                        break;

                    case "grad":
                        geo.adjustHeight(Math.sqrt(fx * fx + fy * fy));
                        arrows[key].lookAt(x + fx, y + fy, 0);
                        arrows[key].position.z = -f0;
                        break;

                    default:
                        break;
                }
            } else {
                arrows[key].geometry = new ArrowBufferGeometry({
                    ...arrowParams,
                    height: 1,
                });
            }
        }

        // tangent plane
        if (planeShard.geometry !== undefined) {
            planeShard.geometry.dispose();
        }

        const tangentPlaneGeometry = new ParametricGeometry(
            (u, v, vec) => {
                const U = -2 + 4 * u,
                      V = -2 + 4 * v;

                vec.set(U, V, U * fx + V * fy);
                vec.add(new THREE.Vector3(0, 0, 0.0001));
            },
            2,
            2
        );

        planeShard.geometry = tangentPlaneGeometry;
    }

    const whiteLineMaterial = new THREE.LineBasicMaterial({
        color: 0xffffff,
        linewidth: 2,
        depthTest: true,
    });

    whiteLineMaterial.polygonOffset = true;
    whiteLineMaterial.polygonOffsetFactor = 0.1;

    const redLineMaterial = new THREE.LineBasicMaterial({
        color: 0xbb3333,
        linewidth: 4,
    });

    const plusMaterial = new THREE.MeshPhongMaterial({
        color: params.color,
        shininess: 80,
        side: THREE.FrontSide,
        vertexColors: false,
        transparent: true,
        opacity: 0.7,
        depthTest: true,
    });
    const minusMaterial = new THREE.MeshPhongMaterial({
        color: 0xff3232,
        shininess: 80,
        side: THREE.BackSide,
        vertexColors: false,
        transparent: true,
        opacity: 0.7,
    });

    {
        const hsl = {};
        plusMaterial.color.getHSL(hsl);
        minusMaterial.color.setHSL((hsl.h + 0.618033988749895) % 1, hsl.s, hsl.l);
    }

    let cMin, dMax; // make these globals as useful for tangents.

    const tol = 1e-12; //tolerance for comparisons

    let surfaceMesh = new THREE.Object3D();

    const updateSurface = function() {
        const { a, b, c, d, z, t0, t1 } = params;
        const A = math.evaluate(a),
              B = math.evaluate(b);
        const [C, D, Z] = math.parse([c, d, z]);
        const geometry = new ParametricGeometry(
            (u, v, vec) => {
                const U = A + (B - A) * u;
                const V = (1 - v) * C.evaluate({ x: U }) + v * D.evaluate({ x: U });
                vec.set(U, V, Z.evaluate({ x: U, y: V, t: t0 + (t1 - t0) * data.tau }));
            },
            data.nX,
            data.nX
        );
        const meshGeometry = meshLines(params, data.rNum, data.cNum, data.nX);
        let material = plusMaterial;

        if (surfaceMesh.children.length > 0) {
            for (let i = 0; i < surfaceMesh.children.length; i++) {
                const mesh = surfaceMesh.children[i];
                mesh.geometry.dispose();
                mesh.geometry = i < 2 ? geometry : meshGeometry;
                if (i < 1) {
                    mesh.material = material;
                }
            }
        } else {
            const backMesh = new THREE.Mesh(geometry, minusMaterial);
            const frontMesh = new THREE.Mesh(geometry, material);

            surfaceMesh.add(frontMesh);
            surfaceMesh.add(backMesh);
            surfaceMesh.add(new THREE.LineSegments(meshGeometry, whiteLineMaterial));
            scene.add(surfaceMesh);
        }
        point.position.set(
            0,
            0,
            Z.evaluate({
                x: 0,
                y: 0,
                t: data.tau * (params.t1 - params.t0) + params.t0,
            })
        );

        if (showLevelCurves) {
            updateLevels();
        }

        if (boxMesh.visible) {
            updateBoxes();
        }

        render();
    }

    const meshLines = function(rData, rNum = 10, cNum = 10, nX = 30) {
        let { a, b, c, d, z } = rData;
        const time = params.t0 + data.tau * (params.t1 - params.t0);
        const A = math.evaluate(a),
              B = math.evaluate(b);

        const du = (B - A) / rNum;
        const dx = (B - A) / lcm(nX, cNum);
        const points = [];
        const args = { x: A, y: math.evaluate(c, { x: A }), t: time };
        for (let u = A; u <= B; u += du) {
            const C = math.evaluate(c, { x: u }),
                  D = math.evaluate(d, { x: u });
            const dy = (D - C) / lcm(nX, rNum);
            args.x = u;
            args.y = C;
            points.push(new THREE.Vector3(u, C, math.evaluate(z, args)));
            for (let v = C + dy; v < D; v += dy) {
                args.y = v;
                points.push(new THREE.Vector3(u, v, math.evaluate(z, args)));
                points.push(new THREE.Vector3(u, v, math.evaluate(z, args)));
            }
            args.y = D;
            points.push(new THREE.Vector3(u, D, math.evaluate(z, args)));
        }

        // v-Meshes
        args.x = A;
        (cMin = math.evaluate(c, args)), (dMax = math.evaluate(d, args));
        for (let u = A + dx; u <= B; u += dx) {
            args.x = u;
            cMin = Math.min(cMin, math.evaluate(c, args));
            dMax = Math.max(dMax, math.evaluate(d, args));
        }

        for (let v = cMin; v <= dMax; v += (dMax - cMin) / cNum) {
            const zs = marchingSegments(
                (x) =>
                (math.evaluate(c, { x: x }) - v) * (v - math.evaluate(d, { x: x })),
                A,
                B,
                nX
            );
            args["y"] = v;
            let nextZero = zs.shift();
            for (let u = A; u <= B - dx + tol; u += dx) {
                args.x = u;
                if (math.evaluate(c, args) <= v && v <= math.evaluate(d, args)) {
                    points.push(
                        new THREE.Vector3(args.x, args.y, math.evaluate(z, args))
                    );
                    if (nextZero < u + dx) {
                        args.x = nextZero;
                        points.push(
                            new THREE.Vector3(args.x, args.y, math.evaluate(z, args))
                        );
                        nextZero = zs.shift();
                    } else {
                        args.x = u + dx;
                        points.push(
                            new THREE.Vector3(args.x, args.y, math.evaluate(z, args))
                        );
                    }
                } else {
                    if (nextZero < u + dx) {
                        args.x = nextZero;
                        points.push(
                            new THREE.Vector3(args.x, args.y, math.evaluate(z, args))
                        );
                        nextZero = zs.shift();
                        args.x = u + dx;
                        points.push(
                            new THREE.Vector3(args.x, args.y, math.evaluate(z, args))
                        );
                    }
                }
            }
        }

        const geometry = new THREE.BufferGeometry().setFromPoints(points);
        return geometry;
    }

    const evolveSurface = function(t) {
        let Z;
        try {
            Z = math.parse(params.z).compile();
        } catch (error) {
            console.error('evolve error', error);
            return;
        }
        boxMesh.visible = false;

        for (let j = 0; j < 3; j += 2) {
            const geometry = surfaceMesh.children[j].geometry;
            const positions = geometry.attributes.position.array;

            let index = 0;
            for (let i = 0; i < positions.length / 3; i++) {
                const x = positions[index++],
                      y = positions[index++];
                positions[index++] = Z.evaluate({ x, y, t });
            }

            geometry.attributes.position.needsUpdate = true;

            if (j === 0) {
                const normals = geometry.attributes.normal.array;
                const vec = new THREE.Vector3(),
                      dx = 0.001;
                let index = 0;
                for (let i = 0; i < normals.length / 3; i++) {
                    const ix = index++;
                    const iy = index++;
                    const iz = index++;

                    const x = positions[ix],
                          y = positions[iy];

                    const fx =
                          Z.evaluate({ x: x + dx, y, t }) - Z.evaluate({ x: x - dx, y, t });
                    const fy =
                          Z.evaluate({ x, y: y + dx, t }) - Z.evaluate({ x, y: y - dx, t });

                    vec.set(-fx, -fy, 2 * dx).normalize();
                    normals[ix] = vec.x;
                    normals[iy] = vec.y;
                    normals[iz] = vec.z;

                    geometry.attributes.normal.needsUpdate = true;
                }

                geometry.computeBoundingBox();
                geometry.computeBoundingSphere();
            }
        }

        if (point.visible) {
            const x = point.position.x;
            const y = point.position.y;
            point.position.z = Z.evaluate({ x, y, t });

            tangentVectors();
        }
    }

    const levelHolder = new THREE.Object3D();
    scene.add(levelHolder);

    const curveBall = new THREE.LineSegments(
        new THREE.BufferGeometry(),
        redLineMaterial
    );
    scene.add(curveBall);

    const shiftInterpolation = function(t, L) {
        if (t < 2) {
            return L - (L / 2) * t;
        } else {
            return (L / 2) * ((3 - t) ** 3 - (3 - t) ** 2);
        }
    }

    const updateColor = function() {
        plusMaterial.color.set(params.color);
        const col = new THREE.Color(params.color);
        const hsl = {};
        col.getHSL(hsl);
        hsl.h = (hsl.h + 0.618033988749895) % 1;
        minusMaterial.color.setHSL(hsl.h, hsl.s, hsl.l);
        render();
    }

    update = function (dt) {
        if (data.animateTime) {
            data.tau += dt / (params.t1 - params.t0);
            data.tau %= 1;
            const t = params.t0 + data.tau * (params.t1 - params.t0);

            evolveSurface(t);
        }

        if (
            (data.shiftLevel < 3 && data.levelDelta > 0) ||
                (data.shiftLevel > 0 && data.levelDelta < 0)
        ) {
            const newLevel = data.shiftLevel + data.levelDelta * dt;
            data.shiftLevel = Math.max(0, Math.min(3, newLevel));
            changeLevels(data.shiftLevel);
        } else {
            if (data.shiftLevel === 3) {
                arrows.u.visible = false;
                arrows.v.visible = false;
                arrows.n.visible = false;
                arrows.grad.visible = true;
            } else {
                arrows.u.visible = true;
                arrows.v.visible = true;
                arrows.grad.visible = false;
            }
            if (!data.animateTime) {
                animation = false;
            }
        }
        render();
    };

    const changeLevels = function(t) {
        for (let index = 0; index < levelHolder.children.length; index++) {
            const element = levelHolder.children[index];
            element.position.set(0, 0, shiftInterpolation(t, element.level));
        }
    }

    const updateLevels = function() {
        for (let index = levelHolder.children.length - 1; index >= 0; index--) {
            const element = levelHolder.children[index];
            element.geometry.dispose();
            levelHolder.remove(element);
        }
        const { a, b, c, d, z } = params;
        const t = params.t0 + data.tau * (params.t1 - params.t0);
        let C = 0,
            D = 0,
            zMin = 0,
            zMax = 0;
        const [A, B] = [math.evaluate(a), math.evaluate(b)];
        for (let i = 0; i <= data.nL; i++) {
            C = Math.min(C, math.evaluate(c, { x: A + ((B - A) * i) / data.nL }));
            D = Math.max(D, math.evaluate(d, { x: A + ((B - A) * i) / data.nL }));
            for (let j = 0; j <= data.nL; j++) {
                const Z = math.evaluate(z, {
                    x: A + ((B - A) * i) / data.nL,
                    y: C + ((D - C) * j) / data.nL,
                    t,
                });
                zMin = Math.min(zMin, Z);
                zMax = Math.max(zMax, Z);
            }
        }

        for (
            let lev = zMin;
            lev <= zMax;
            lev += Math.max((zMax - zMin) / data.nL, 0.01)
        ) {
            const points = marchingSquares({
                f: (x, y) => {
                    return math.evaluate(z, { x, y, t });
                },
                xmin: A,
                xmax: B,
                ymin: C,
                ymax: D,
                level: lev,
                zLevel: 0,
                nX: data.nX,
                nY: data.nX,
            });

            if (points.length > 1) {
                const levelGeometry = new THREE.BufferGeometry();

                levelGeometry.setFromPoints(points);

                const levelMesh = new THREE.LineSegments(
                    levelGeometry,
                    new THREE.LineBasicMaterial({
                        color: new THREE.Color().setHSL(
                            (lev - zMin) / (zMax - zMin),
                            0.5,
                            0.5
                        ),
                        linewidth: 3,
                        transparent: false,
                    })
                );

                levelMesh.level = lev;
                levelMesh.position.set(0, 0, shiftInterpolation(data.shiftLevel, lev));

                levelHolder.add(levelMesh);
            }
        }

        // change selected level curve height
        curveBall.position.set(
            0,
            0,
            shiftInterpolation(data.shiftLevel, curveBall.level)
        );
    }

    const boxMesh = new THREE.Mesh();
    boxMesh.material = colorMaterial;
    boxMesh.visible = false;
    scene.add(boxMesh);
    const boxMeshEdges = new THREE.LineSegments(
        new THREE.BufferGeometry(),
        boxEdgeMaterial
    );
    boxMesh.add(boxMeshEdges);

    const updateBoxes = function() {
        const { a, b, c, d, z } = params;
        const t = params.t0 + data.tau * (params.t1 - params.t0);
        const [A, B, C, D] = [
            math.evaluate(a),
            math.evaluate(b),
            math.evaluate(c),
            math.evaluate(d),
        ];

        if (boxMesh.geometry) {
            boxMesh.geometry.dispose();
            boxMeshEdges.geometry.dispose();
        }
        boxMesh.geometry = blockGeometry(
            (x, y) => {
                return math.evaluate(z, { x, y, t });
            },
            A,
            B,
            C,
            D,
            data.N,
            data.N,
            data.s,
            data.t
        );
        boxMeshEdges.geometry = new THREE.EdgesGeometry(boxMesh.geometry);
    }

    onMount(() => {
        updateSurface();
        updateBoxes();
    });
    onDestroy(() => {
        for (const child of surfaceMesh.children) {
            child.geometry && child.geometry.dispose();
            child.material && child.material.dispose();
        }
        scene.remove(surfaceMesh);
        for (const child of levelHolder.children) {
            child.geometry && child.geometry.dispose();
            child.material && child.material.dispose();
        }
        scene.remove(levelHolder);
        scene.remove(point);
        window.removeEventListener("keydown", shiftDown);
        window.removeEventListener("keyup", shiftUp);
        render();
    });

    const raycaster = new THREE.Raycaster();

    let mouseVector = new THREE.Vector2();

    const onMouseMove = function(e) {
        // normalized mouse coordinates
        mouseVector.x = 2 * (e.clientX / window.innerWidth) - 1;
        mouseVector.y = 1 - 2 * (e.clientY / window.innerHeight);

        raycaster.setFromCamera(mouseVector, camera);

        const intersects = raycaster.intersectObjects(
            [surfaceMesh.children[0], surfaceMesh.children[1]],
            true
        );

        if (intersects.length > 0) {
            const intersect = intersects[0];
            point.position.x = intersect.point.x;
            point.position.y = intersect.point.y;
            point.position.z = intersect.point.z;

            tangentVectors();

            render();
        }
    }

    const activateLevelElevator = function() {
        animation = true;
        data.levelDelta *= -1;
        dispatch("animate");
    }

    const shiftDown = (e) => {
        if (shadeUp && selected === objID) {
            switch (e.key) {
                case "Shift":
                    window.addEventListener("mousemove", onMouseMove, false);
                    break;
                case "0":
                    activateLevelElevator();
                    break;
                case "c":
                    controls.target.set(
                        point.position.x,
                        point.position.y,
                        point.position.z
                    );
                    render();
                    break;
                case "Backspace":
                    surfaceMesh.visible = !surfaceMesh.visible;
                    render();
                    break;
                case "t":
                    point.visible = !point.visible;
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
                case "b":
                    boxMesh.visible = !boxMesh.visible;
                    render();
                    break;
                case ">":
                    data.N = Math.min(101, data.N + 1);
                    updateBoxes();
                    render();
                    break;
                case "<":
                    data.N = Math.max(1, data.N - 1);
                    updateBoxes();
                    render();
                    break;
                case "l":
                    showLevelCurves = !showLevelCurves;
                    if (showLevelCurves) {
                        updateLevels();
                        levelHolder.visible = true;
                    } else {
                        levelHolder.visible = false;
                    }
                    render();
                    break;
                case "p":
                    data.animateTime = !data.animateTime;
                    if (data.animateTime) {
                        animation = true;
                        dispatch("animate");
                        showLevelCurves = false;
                        levelHolder.visible = false;
                    }
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
        <strong>Graph of function</strong>
        <span>
            <button on:click={() => {hidden = !hidden;}}>
                <i class="fa fa-window-minimize" />
            </button>
            <button on:click={onClose}>
                <i class="fa fa-window-close" />
            </button>
        </span>
    </div>
    <div hidden={hidden}>
        <div class="container">
            <span class="box-1">
                <M size="sm">f(x,y[,t]) =</M>
            </span>
            <input
                type="text"
                bind:value={params.z}
                on:change={updateSurface}
                on:input={() => {data.animateTime = false;}}
                class="box box-2"
            />
            <input
                type="text"
                bind:value={params.a}
                on:change={() => {onUpdate(); updateSurface();}}
                class="box"
            />
            <span class="box box-3">
                <M size="sm">\leq x \leq</M>
            </span>
            <input
                type="text"
                bind:value={params.b}
                on:change={() => {onUpdate(); updateSurface();}}
                class="box"
            />
            <input
                type="text"
                bind:value={params.c}
                on:change={() => {onUpdate(); updateSurface();}}
                class="box"
            />
            <span class="box box-3">
                <M size="sm">\leq y \leq</M>
            </span>
            <input
                type="text"
                bind:value={params.d}
                on:change={() => {onUpdate(); updateSurface()}}
                class="box"
            />
            <span class="box-1">
                <M size="sm">x</M>-meshes
            </span>
            <input
                type="range"
                bind:value={data.rNum}
                min="0"
                max="20"
                step="1"
                on:input={() => {
                    const mesh = surfaceMesh.children[2];
                    mesh.geometry.dispose();
                    mesh.geometry = meshLines(
                        params, data.rNum, data.cNum, data.nX);
                    render();
                }}
                class="box box-2"
            />
            <span class="box-1">
                <M size="sm">y</M>-meshes
            </span>
            <input
                type="range"
                bind:value={data.cNum}
                min="0"
                max="20"
                step="1"
                on:input={() => {
                    const mesh = surfaceMesh.children[2];
                    mesh.geometry.dispose();
                    mesh.geometry = meshLines(
                        params, data.rNum, data.cNum, data.nX);
                    render();
                }}
                class="box box-2"
            />
            <span class="box-1">
                Resolution
            </span>
            <input
                type="range"
                bind:value={data.nX}
                min="10"
                max="60"
                step="5"
                on:input={updateSurface}
                class="box box-2"
            />
            <span class="box-1">
                Levels
            </span>
            <span class="box-2">
                <button on:click={activateLevelElevator} class="box-2">
                    {#if data.levelDelta === 1}
                        Up <i class="fa fa-caret-up" />
                    {:else}
                        Down <i class="fa fa-caret-down" />
                    {/if}
                </button>
            </span>
            <span class="box-1">
                View Graph
            </span>
            <label class="switch box box-2">
                <input
                    type="checkbox"
                    name="graphVisible"
                    id="graphVisible"
                    bind:checked={surfaceMesh.visible}
                    on:change={render}
                />
                <span class="slider round" />
            </label>
            <input type="number" bind:value={params.t0} class="box" />
            <span class="box box-3">
                <M size="sm">\leq t \leq</M>
            </span>
            <input type="number" bind:value={params.t1} class="box" />
            <span class="box-1">
                <M size="sm">t</M>
            </span>
            <input
                type="range"
                bind:value={data.tau}
                min="0"
                max="1"
                step="0.001"
                on:input={() => {
                    evolveSurface(data.tau * (params.t1 - params.t0));
                    render();
                }}
                class="box box-2"
            />
            <span class="play-buttons box-4">
                <button
                    on:click={() => {
                    data.animateTime = !data.animateTime;
                    if (data.animateTime) {
                    animation = true;
                    dispatch("animate");
                    }
                    }}
                    class="box-1"
                    >
                    {#if !data.animateTime}
                        <i class="fa fa-play" />
                    {:else}
                        <i class="fa fa-pause" />
                    {/if}
                </button>
                <button
                    on:click={() => {
                    data.animateTime = false;
                    render();
                    }}
                    class="box-3"
                    >
                    <i class="fa fa-stop" />
                </button>
                <button
                    on:click={() => {
                    data.tau = 0;
                    evolveSurface(params.t0);
                    }}
                    class="box-4"
                    >
                    <i class="fa fa-fast-backward" />
                </button>
            </span>
            <span class="box-1">
                Tangents
            </span>
            <label class="switch box box-2">
                <input
                    type="checkbox"
                    name="frameVisible"
                    id="frameVisible"
                    bind:checked={point.visible}
                    on:change={render}
                />
                <span class="slider round" />
            </label>
            <span class="box box-2">
                <input
                    type="color"
                    name="colorPicker"
                    id="colorPicker"
                    bind:value={params.color}
                    on:change={() => {
                        onUpdate();
                        updateColor();
                    }}
                    style="width:85%; padding: 1px 1px;"
                />
            </span>
            <span class="box-1">
                Integrate
            </span>
            <label class="switch box box-2">
                <input
                    type="checkbox"
                    name="doIntegral"
                    id="doIntegral"
                    bind:checked={boxMesh.visible}
                    on:change={() => {
                        if (boxMesh.visible) {updateBoxes();}
                        render();
                    }}
                />
                <span class="slider round" />
            </label>
            <span class="box-1">
                <M size="sm">N</M>
            </span>
            <input
                type="range"
                bind:value={data.N}
                min="1"
                max="81"
                step="1"
                on:input={() => {
                    updateBoxes();
                    render();
                }}
                class="box box-2"
            />
            <span class="box-1"> Sample </span>
            <input
                type="range"
                bind:value={data.samp}
                min="0"
                max="4"
                step="1"
                on:input={() => {
                    const pt = [
                        [0, 0],
                        [1, 0],
                        [1 / 2, 1 / 2],
                        [0, 1],
                        [1, 1],
                    ];
                    [data.s, data.t] = pt[data.samp];
                    updateBoxes();
                    render();
                }}
                class="box box-2"
            />
        </div>
    </div>
</div>
