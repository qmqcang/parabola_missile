<template>
    <div class="container" ref="contDom"></div>
</template>

<script setup>
import * as THREE from 'three'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader'
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader'
import { OrbitControls } from 'three/addons/controls/OrbitControls'
import { ref, onMounted } from 'vue'

const contDom = ref(null)

let scene, camera, renderer, controls
let gridHelper

let BezierCurveLine, missileMesh, sphereMesh
let curve, imgData, points

const SPHERE_RADIUS = 100

onMounted(() => {
    initScene()
    initCamera()
    initRenderer()
    // initGird()
    initLight()
    initModels()
    initControls()

    render()

    window.addEventListener('resize', onWindowResize, false)
})

const initScene = () => {
    scene = new THREE.Scene()
}

const initCamera = () => {
    camera = new THREE.PerspectiveCamera(
        45,
        window.innerWidth / window.innerHeight,
        1,
        10000
    )
    camera.position.set(0, 0, 300)
    camera.lookAt(new THREE.Vector3(0, 0, 0))
    camera.updateProjectionMatrix()

    scene.add(camera)
}

const initRenderer = () => {
    renderer = new THREE.WebGLRenderer({
        antialias: true,
        logarithmicDepthBuffer: true
    })
    renderer.setPixelRatio(window.devicePixelRatio)
    renderer.setSize(window.innerWidth, window.innerHeight)

    renderer.setAnimationLoop(animate)

    contDom.value.append(renderer.domElement)
}

const initGird = () => {
    gridHelper = new THREE.GridHelper(500, 100, 0x0000ff, 0x666666)

    scene.add(gridHelper)
}

const initLight = () => {
    const ambientLight = new THREE.AmbientLight(0xffffff, 1)

    ambientLight.position.set(0, 0, 0)

    scene.add(ambientLight)
}

const initModels = () => {
    loadMissileMesh()
    loadSphereMesh()
    loadPointMesh()
}

const loadMissileMesh = () => {
    const loader = new GLTFLoader()
    const dracoLoader = new DRACOLoader()

    dracoLoader.setDecoderPath('/models/')
    loader.setDRACOLoader(dracoLoader)

    loader.load('/models/missile.glb', (gltf) => {
        missileMesh = gltf.scene

        missileMesh.scale.set(5, 5, 5)
        missileMesh.position.set(0, 0, 0)

        scene.add(missileMesh)
    })
}

const loadSphereMesh = () => {
    const sphereGeometry = new THREE.SphereGeometry(SPHERE_RADIUS, 256, 256)
    const sphereMaterial = new THREE.MeshStandardMaterial({
        map: new THREE.TextureLoader().load(
            './textures/earth_specular_2048.jpg'
        ),
        blending: THREE.AdditiveBlending,
        depthTest: false,
        depthWrite: false,
        transparent: true
    })

    sphereMesh = new THREE.Mesh(sphereGeometry, sphereMaterial)

    sphereMesh.position.set(0, 0, 0)

    scene.add(sphereMesh)

    imgToData()
}

const imgToData = () => {
    const imgData = []
    const image = new Image()
    image.src = './textures/earth_specular_2048.jpg'

    image.onload = () => {
        const canvas = document.createElement('canvas')

        canvas.width = image.width
        canvas.height = image.height

        const context = canvas.getContext('2d')
        context.drawImage(image, 0, 0, canvas.width, canvas.height)

        const data = context.getImageData(0, 0, canvas.width, canvas.height)
        const dLength = data.data.length

        for (let i = 0; i < dLength; i += 5) {
            let x = (i / 4) % canvas.width
            let y = (i / 4 - x) / canvas.width

            if ((i / 4) % 2 == 1 && y % 2 == 1 && data.data[i] !== 255) {
                let u = (360 / canvas.width) * x + 360
                let v = (180 / canvas.height) * y + 90

                let xyz = lglt2xyz(u, v)

                imgData.push(xyz)
            }
        }
        const pointGeometry = new THREE.BufferGeometry()
        const pointMaterial = new THREE.PointsMaterial({
            size: 2,
            opacity: 0.5,
            transparent: true,
            blending: THREE.AdditiveBlending
        })

        const positions = []

        imgData.forEach((e) => {
            positions.push(e.x, e.y, e.z)
        })
        pointGeometry.setAttribute(
            'position',
            new THREE.Float32BufferAttribute(positions, 3)
        )

        const points = new THREE.Points(pointGeometry, pointMaterial)

        scene.add(points)
    }
}

const loadPointMesh = () => {
    const pointGeometry = new THREE.CircleGeometry(2, 32)
    const pointMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 })

    const startPoint = new THREE.Mesh(pointGeometry, pointMaterial)
    const startPointPosition = lglt2xyz(116, 40)

    startPoint.position.set(
        startPointPosition.x,
        startPointPosition.y,
        startPointPosition.z
    )
    startPoint.quaternion.setFromUnitVectors(
        new THREE.Vector3(0, 0, 1).normalize(),
        startPoint.position.clone().normalize()
    )

    const endPoint = startPoint.clone()
    const endPointPosition = lglt2xyz(0, 45)

    endPoint.position.set(
        endPointPosition.x,
        endPointPosition.y,
        endPointPosition.z
    )
    endPoint.quaternion.setFromUnitVectors(
        new THREE.Vector3(0, 0, 1).normalize(),
        endPoint.position.clone().normalize()
    )

    const pointGroup = new THREE.Group()

    pointGroup.add(startPoint, endPoint)

    sphereMesh.add(pointGroup)

    loadBezierCurveLine(startPointPosition, endPointPosition)
}

const lglt2xyz = (longitude, latitude, radius = SPHERE_RADIUS) => {
    const phi = THREE.MathUtils.degToRad(90 - latitude)
    const theta = THREE.MathUtils.degToRad(90 + longitude)

    return new THREE.Vector3().setFromSphericalCoords(radius, phi, theta)
}

const loadBezierCurveLine = (v0, v3) => {
    const { v1, v2 } = BezierCurveControlPoint(v0, v3)

    curve = new THREE.CubicBezierCurve3(v0, v1, v2, v3)

    const points = curve.getPoints(200)

    const parabolaGeometry = new THREE.BufferGeometry().setFromPoints(points)
    const parabolaMaterial = new THREE.LineBasicMaterial()

    BezierCurveLine = new THREE.Line(parabolaGeometry, parabolaMaterial)

    scene.add(BezierCurveLine)
}

const BezierCurveControlPoint = (v0, v3) => {
    // 将两向量之间的弧度转为角度
    let deg = THREE.MathUtils.radToDeg(v0.angleTo(v3))

    let origin = new THREE.Vector3()
    let p1 = getVCenter(v0.clone(), v3.clone())
    let ray = new THREE.Ray(origin, p1.clone().normalize())
    // ray.at() 没有返回值
    let p2 = new THREE.Vector3()
    ray.at((2 * SPHERE_RADIUS - p1.distanceTo(origin)) * deg, p2)

    let v1 = getLerpVector3(v0.clone(), p2, p1)
    let v2 = getLerpVector3(v3.clone(), p2, p1)

    return { v1, v2 }
}

// 获取两点间的中点
const getVCenter = (v1, v2) => {
    return v1.add(v2).divideScalar(2)
}

const getLerpVector3 = (v, p2, p1) => {
    let vp1Len = v.distanceTo(p1)
    let vp2Len = v.distanceTo(p2)

    return v.lerp(p2, (vp1Len / vp2Len) * 1.4)
}

const initControls = () => {
    controls = new OrbitControls(camera, renderer.domElement)

    controls.enableDamping = true
}

const render = () => {
    controls.update()

    renderer.render(scene, camera)

    requestAnimationFrame(render)
}

const animate = (msTime) => {
    let time = msTime / 1000 / 10

    time %= 1

    missileMesh?.position.copy(curve.getPointAt(time))
    missileMesh?.lookAt(curve.getPointAt(time + 0.01))
}

const onWindowResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()

    renderer.setPixelRatio(window.devicePixelRatio)
    renderer.setSize(window.innerWidth, window.innerHeight)
}
</script>

<style scoped></style>
