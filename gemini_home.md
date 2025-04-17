<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Three.js 房间平面图 (带信息弹窗)</title>
    <style>
        body { margin: 0; overflow: hidden; font-family: sans-serif; }
        canvas { display: block; }
        #info { /* Top instruction text */
            position: absolute;
            top: 10px;
            width: 100%;
            text-align: center;
            color: #ffffff;
            background-color: rgba(0,0,0,0.5);
            padding: 5px;
            z-index: 100;
        }
        /* --- Tooltip/Popup Styles --- */
        #tooltip {
            position: absolute; /* Positioned via JavaScript */
            display: none; /* Hidden by default */
            background-color: rgba(0, 0, 0, 0.7); /* Semi-transparent black background */
            color: white; /* White text */
            padding: 10px 15px;
            border-radius: 8px; /* Rounded corners */
            font-size: 14px;
            line-height: 1.4;
            pointer-events: none; /* Prevent tooltip from blocking clicks on objects behind it */
            white-space: nowrap; /* Prevent text wrapping */
            z-index: 101; /* Ensure it's above the info text */
            border: 1px solid rgba(255, 255, 255, 0.3); /* Subtle border */
            box-shadow: 0 2px 5px rgba(0,0,0,0.2); /* Soft shadow */
        }
        #tooltip strong { /* Style for device name */
             display: block;
             margin-bottom: 5px;
             font-size: 16px;
             color: #66ccff; /* Light blue for the name */
        }
         #tooltip span { /* Style for status/power lines */
             display: block;
         }
         #tooltip .status-on {
             color: #90ee90; /* Light green for 'On' status */
         }
         #tooltip .status-off {
             color: #ff7f7f; /* Light red for 'Off' status */
         }
    </style>
</head>
<body>
    <div id="info">使用鼠标拖动旋转，滚轮缩放，右键拖动平移。点击设备模型进行交互并查看信息。</div>
    <div id="tooltip"></div>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
    <script>
        // === 基本设置 ===
        let scene, camera, renderer, controls;
        let raycaster, mouse;
        const clickableObjects = [];
        const wallHeight = 2.5;
        const wallThickness = 0.15;
        const deviceStates = {};
        const originalMaterials = new Map();
        let tooltipElement; // Reference to the HTML tooltip div

        function init() {
            tooltipElement = document.getElementById('tooltip'); // Get tooltip element

            // --- Scene, Camera, Renderer, Lighting, Floor, Walls (Same as before) ---
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0xcccccc);
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(5, 8, 10);
            camera.lookAt(0, 0, 0);
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.shadowMap.enabled = true;
            document.body.appendChild(renderer.domElement);
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
            scene.add(ambientLight);
            const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
            directionalLight.position.set(10, 15, 10);
            directionalLight.castShadow = true;
            directionalLight.shadow.mapSize.width = 1024; directionalLight.shadow.mapSize.height = 1024;
            directionalLight.shadow.camera.near = 0.5; directionalLight.shadow.camera.far = 50;
            directionalLight.shadow.camera.left = -10; directionalLight.shadow.camera.right = 10;
            directionalLight.shadow.camera.top = 10; directionalLight.shadow.camera.bottom = -10;
            scene.add(directionalLight);
            const floorGeometry = new THREE.PlaneGeometry(15, 15);
            const floorMaterial = new THREE.MeshStandardMaterial({ color: 0x808080, side: THREE.DoubleSide, roughness: 0.8, metalness: 0.2 });
            const floor = new THREE.Mesh(floorGeometry, floorMaterial);
            floor.rotation.x = -Math.PI / 2;
            floor.receiveShadow = true;
            scene.add(floor);
            const wallSegments = [ [-5, 5, 5, 5], [-5, -5, 5, -5], [-5, 5, -5, -5], [5, 5, 5, -5], [-1, 5, -1, 0], [-1, 0, 2, 0], [2, 0, 2, -5], [-5, 2, -1, 2] ];
            const wallMaterial = new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0.9, metalness: 0.1 });
            function createWall(x1, z1, x2, z2) { /* ... wall creation logic ... */
                const length = Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(z2 - z1, 2));
                const angle = Math.atan2(z2 - z1, x2 - x1);
                const wallGeometry = new THREE.BoxGeometry(length, wallHeight, wallThickness);
                const wall = new THREE.Mesh(wallGeometry, wallMaterial);
                wall.position.set((x1 + x2) / 2, wallHeight / 2, (z1 + z2) / 2);
                wall.rotation.y = -angle;
                wall.castShadow = true; wall.receiveShadow = true;
                scene.add(wall);
             }
            wallSegments.forEach(segment => createWall(segment[0], segment[1], segment[2], segment[3]));

            // --- 创建设备模型 (Create Device Models with Power Data) ---

            function createLamp(position, name, power) { /* ... lamp creation ... */
                const lampGroup = new THREE.Group();
                lampGroup.position.set(position[0], 0, position[1]);
                const baseHeight = 0.1, standHeight = 0.8, shadeRadius = 0.25, shadeHeight = 0.3;
                const baseGeo = new THREE.CylinderGeometry(shadeRadius * 0.8, shadeRadius * 0.8, baseHeight, 16);
                const baseMat = new THREE.MeshStandardMaterial({ color: 0x555555, roughness: 0.6, metalness: 0.4 });
                const baseMesh = new THREE.Mesh(baseGeo, baseMat); baseMesh.position.y = baseHeight / 2; baseMesh.castShadow = true; lampGroup.add(baseMesh);
                const standGeo = new THREE.CylinderGeometry(0.05, 0.05, standHeight, 8);
                const standMat = new THREE.MeshStandardMaterial({ color: 0x777777, roughness: 0.4, metalness: 0.6 });
                const standMesh = new THREE.Mesh(standGeo, standMat); standMesh.position.y = baseHeight + standHeight / 2; standMesh.castShadow = true; lampGroup.add(standMesh);
                const shadeGeo = new THREE.ConeGeometry(shadeRadius, shadeHeight, 16);
                const shadeMat = new THREE.MeshStandardMaterial({ color: 0xffffcc, roughness: 0.8, metalness: 0.1, side: THREE.DoubleSide });
                const shadeMesh = new THREE.Mesh(shadeGeo, shadeMat); shadeMesh.position.y = baseHeight + standHeight + shadeHeight / 2; shadeMesh.castShadow = true; lampGroup.add(shadeMesh);
                originalMaterials.set(shadeMesh.uuid, shadeMat.clone());
                // Add power consumption data
                lampGroup.userData = { name: name, isOn: false, interactiveMesh: shadeMesh, powerConsumption: power };
                deviceStates[name] = false; clickableObjects.push(lampGroup); scene.add(lampGroup); return lampGroup;
            }

            function createAirConditioner(position, name, power, rotationY = 0) { /* ... AC creation ... */
                const acGroup = new THREE.Group();
                acGroup.position.set(position[0], position[1], position[2]); acGroup.rotation.y = rotationY;
                const acWidth = 0.8, acHeight = 0.25, acDepth = 0.15;
                const acGeo = new THREE.BoxGeometry(acWidth, acHeight, acDepth);
                const acMat = new THREE.MeshStandardMaterial({ color: 0xf0f0f0, roughness: 0.7, metalness: 0.1 });
                const acMesh = new THREE.Mesh(acGeo, acMat); acMesh.castShadow = true; acGroup.add(acMesh);
                originalMaterials.set(acMesh.uuid, acMat.clone());
                 // Add power consumption data
                acGroup.userData = { name: name, isOn: false, interactiveMesh: acMesh, powerConsumption: power };
                deviceStates[name] = false; clickableObjects.push(acGroup); scene.add(acGroup); return acGroup;
            }

             function createTV(position, name, power, rotationY = 0) { /* ... TV creation ... */
                const tvGroup = new THREE.Group();
                tvGroup.position.set(position[0], position[1], position[2]); tvGroup.rotation.y = rotationY;
                const screenWidth = 1.2, screenHeight = 0.7, screenDepth = 0.05;
                const screenGeo = new THREE.BoxGeometry(screenWidth, screenHeight, screenDepth);
                const screenMat = new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 0.9, metalness: 0.1 });
                const screenMesh = new THREE.Mesh(screenGeo, screenMat); screenMesh.castShadow = true; tvGroup.add(screenMesh);
                const backGeo = new THREE.BoxGeometry(screenWidth * 0.95, screenHeight * 0.95, screenDepth * 2);
                const backMat = new THREE.MeshStandardMaterial({ color: 0x333333, roughness: 0.8, metalness: 0.2 });
                const backMesh = new THREE.Mesh(backGeo, backMat); backMesh.position.z = screenDepth; tvGroup.add(backMesh);
                originalMaterials.set(screenMesh.uuid, screenMat.clone());
                 // Add power consumption data
                tvGroup.userData = { name: name, isOn: false, interactiveMesh: screenMesh, powerConsumption: power };
                deviceStates[name] = false; clickableObjects.push(tvGroup); scene.add(tvGroup); return tvGroup;
            }

            // --- 实例化设备 (Instantiate Devices with Power) ---
            createLamp([-3, 3.5], '卧室灯', 15); // Lamp in top-left area, 15W
            createAirConditioner([0, wallHeight - 0.3, -4.9], '客厅空调', 1200, 0); // AC on bottom wall, 1200W
            createTV([4.8, 0.8, 0], '客厅电视', 150, -Math.PI / 2); // TV on right wall, 150W

            // --- 交互设置 (Interaction Setup) ---
            raycaster = new THREE.Raycaster();
            mouse = new THREE.Vector2();
            // Use 'pointerdown' for better mobile compatibility and to hide tooltip before potentially showing a new one
            window.addEventListener('pointerdown', onPointerDown, false);

            // --- 控制器 (Controls) ---
            controls = new THREE.OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true; controls.dampingFactor = 0.05;
            controls.screenSpacePanning = false; controls.minDistance = 2; controls.maxDistance = 50;
            controls.maxPolarAngle = Math.PI / 2 - 0.05;

            // --- 窗口大小调整处理器 (Window Resize Handler) ---
            window.addEventListener('resize', onWindowResize, false);

            // --- 启动动画循环 (Start Animation Loop) ---
            animate();
        }

        // --- 3D to 2D Projection Function ---
        function project3DTo2D(vector3) {
            const vector = vector3.clone().project(camera); // Project 3D point to Normalized Device Coordinates (NDC)
            const x = (vector.x + 1) / 2 * window.innerWidth;
            const y = (-vector.y + 1) / 2 * window.innerHeight;
            return { x, y };
        }

        // --- 更新和显示 Tooltip ---
        function updateTooltip(groupData, screenPos) {
            const deviceName = groupData.name;
            const isOn = groupData.isOn;
            const power = groupData.powerConsumption;

            const statusText = isOn ? `<span class="status-on">状态：开启</span>` : `<span class="status-off">状态：关闭</span>`;
            const powerText = isOn ? `<span>功率：${power} W</span>` : `<span>功率：0 W</span>`;

            tooltipElement.innerHTML = `<strong>${deviceName}</strong>${statusText}${powerText}`;
            tooltipElement.style.left = `${screenPos.x + 15}px`; // Position slightly offset from cursor/object
            tooltipElement.style.top = `${screenPos.y - 10}px`;
            tooltipElement.style.display = 'block'; // Make it visible
        }

        // --- 隐藏 Tooltip ---
        function hideTooltip() {
            tooltipElement.style.display = 'none';
        }

        // --- 指针按下事件处理 (Pointer Down Event Handler) ---
        function onPointerDown(event) {
            // Always hide tooltip on any click/touch first
            hideTooltip();

            // Calculate mouse position
            // Adjust for potential multi-touch, use the first touch point if available
            const clientX = event.touches ? event.touches[0].clientX : event.clientX;
            const clientY = event.touches ? event.touches[0].clientY : event.clientY;
            mouse.x = (clientX / window.innerWidth) * 2 - 1;
            mouse.y = - (clientY / window.innerHeight) * 2 + 1;

            raycaster.setFromCamera(mouse, camera);
            const intersects = raycaster.intersectObjects(clickableObjects, true);

            if (intersects.length > 0) {
                let clickedGroup = null;
                let intersectedObject = intersects[0].object;
                let parent = intersectedObject;
                while (parent) {
                    if (parent.userData && parent.userData.name) {
                        clickedGroup = parent;
                        break;
                    }
                    parent = parent.parent;
                }

                if (clickedGroup && clickedGroup.userData.interactiveMesh) {
                    const groupData = clickedGroup.userData;
                    const deviceName = groupData.name;
                    const interactiveMesh = groupData.interactiveMesh;
                    const originalMaterial = originalMaterials.get(interactiveMesh.uuid);

                    // --- Toggle State ---
                    groupData.isOn = !groupData.isOn;
                    deviceStates[deviceName] = groupData.isOn;

                    // --- Update Visuals ---
                    if (groupData.isOn) {
                        interactiveMesh.material = interactiveMesh.material.clone();
                        interactiveMesh.material.emissive.set(0xffaa00);
                        interactiveMesh.material.emissiveIntensity = 1.0;
                        if (deviceName === '客厅电视') {
                             interactiveMesh.material.color.set(0x87ceeb);
                             interactiveMesh.material.emissive.set(0x87ceeb);
                             interactiveMesh.material.emissiveIntensity = 0.4;
                        }
                        console.log(`${deviceName} 已开启`);
                    } else {
                        if (originalMaterial) {
                             interactiveMesh.material = originalMaterial.clone();
                             interactiveMesh.material.emissive.set(0x000000);
                             interactiveMesh.material.emissiveIntensity = 0;
                        }
                         console.log(`${deviceName} 已关闭`);
                    }
                    interactiveMesh.material.needsUpdate = true;

                    // --- Show Tooltip ---
                    // Calculate position based on the center of the clicked group's bounding box for better placement
                    const box = new THREE.Box3().setFromObject(clickedGroup);
                    const center = box.getCenter(new THREE.Vector3());
                    const screenPos = project3DTo2D(center);
                    updateTooltip(groupData, screenPos);

                    // Prevent OrbitControls from activating on the same click that interacted with an object
                    event.stopPropagation();
                }
            }
            // If no interactive object was clicked, the tooltip remains hidden (handled at the start)
        }


        // --- 窗口大小调整函数 (Window Resize Function) ---
        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
            hideTooltip(); // Hide tooltip on resize to avoid wrong positioning
        }

        // --- 动画循环 (Animation Loop) ---
        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
            // Note: Tooltip position is NOT updated every frame here.
            // It's positioned once on click. If you want it to stick perfectly,
            // you'd need to store the active group and update its position here.
        }

        // --- 初始化 (Initialize) ---
        init();

    </script>
</body>
</html>
