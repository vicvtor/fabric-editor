<template>
    <v-container fluid fill-height>
        <v-row class="align-stretch align-self-stretch">
            <v-col md="8">
                <v-layout class="fill-height container--fluid" :id="this.holderId">
                    <canvas id="canvas"/>
                </v-layout>
            </v-col>
            <v-col md="4">
                <v-card class="mx-auto" tile>
                    <v-toolbar color="light-blue" dark>
                        <v-toolbar-title>Регионы</v-toolbar-title>
                        <v-spacer></v-spacer>
                        <v-btn icon @click="drawPolygon">
                            <v-icon>mdi-plus</v-icon>
                        </v-btn>
                    </v-toolbar>
                    <v-list class="px-2" dense>
                        <v-alert type="info" v-if="this.getPolygonsFromCanvas.length === 0">
                            Размеченные регионы отсутствуют.
                        </v-alert>
                        <v-list-item two-line v-for="(item, i) in getPolygonsFromCanvas"
                                     :key="i">
                            <v-list-item-avatar :color=item.stroke>
                                <span class="white--text headline"></span>
                            </v-list-item-avatar>
                            <v-list-item-content>
                                <v-text-field
                                        label="Regular"
                                        single-line
                                        v-model="item.description"
                                        :readonly="item !== activePolygon"
                                ></v-text-field>
                            </v-list-item-content>
                            <v-list-item-action class="ml-1">
                                <v-btn icon @click="editPolygon(item)" v-if="item !== activePolygon">
                                    <v-icon color="blue lighten-1">mdi-pencil-circle</v-icon>
                                </v-btn>
                                <v-btn icon @click="finishEditPolygon(item)" v-else>
                                    <v-icon color="green lighten-1">mdi-check-circle</v-icon>
                                </v-btn>
                            </v-list-item-action>
                            <v-list-item-action class="ml-0">
                                <v-btn icon @click="removePolygon(item.description)">
                                    <v-icon color="red lighten-1">mdi-close-circle</v-icon>
                                </v-btn>
                            </v-list-item-action>
                        </v-list-item>
                    </v-list>
                </v-card>
            </v-col>
        </v-row>
    </v-container>
</template>

<script>
    import {fabric} from 'fabric'

    export default {
        name: 'Editor',
        data() {
            return {
                canvas: null,
                holderId: 'canvas-holder',
                creating: false,
                editing: false,
                regions: [],
                pointArray: [],
                lineArray: [],
                activeLine: null,
                activePolygon: null,
                activeShape: false,
                imageScale: 1,
                strokeColor: '#FFFFFF'
            }
        },
        props: ['backgroundImage', 'value'],
        mounted() {
            this.initializeCanvas()
        },
        methods: {

            /**
             * Инициализация canvas.
             */
            initializeCanvas() {
                let holder = document.getElementById(this.holderId)
                this.canvas = new fabric.Canvas('canvas')

                fabric.Image.fromURL(this.backgroundImage, (oImg) => {
                    if (oImg.width > holder.clientWidth) {
                        oImg.scaleToWidth(holder.clientWidth)
                        this.imageScale = oImg.scaleX

                    }
                    this.canvas.setWidth(oImg.width * oImg.scaleX)
                    this.canvas.setHeight(oImg.height * oImg.scaleX)
                    this.canvas.setBackgroundImage(oImg)

                    //Загрузка сохраненных полигонов
                    this.value.forEach(it => {
                        this.strokeColor = this.getRandomColor()
                        this.loadPolygon(it.points, it.description)
                    })

                });

                // Событие нажатия на область отрисовки
                this.canvas.on('mouse:down', (options) => {
                    if (this.pointArray.length > 0 && options.target && options.target.id === this.pointArray[0].id) {
                        this.generatePolygon(this.pointArray)
                    }
                    if (this.creating) {
                        this.addPoint(options)
                    }
                });

                // Событие перемещения курсора на область отрисовки
                this.canvas.on('mouse:move', (options) => {
                    if (this.activeLine && this.activeLine.class === "line") {
                        let pointer = this.canvas.getPointer(options.e)
                        this.activeLine.set({x2: pointer.x, y2: pointer.y})

                        let points = this.activeShape.get("points")
                        points[this.pointArray.length] = {
                            x: pointer.x,
                            y: pointer.y
                        }
                        this.activeShape.set({
                            points: points
                        });
                        this.canvas.renderAll()
                    }
                    this.canvas.renderAll()
                })

                // Перемещение объекта в области отрисовки
                this.canvas.on('object:moving', (options) => {
                    if (options.target && options.target.type === 'circle') {
                        let p = options.target
                        this.activePolygon.points[p.pointIndex] = {
                            x: p.getCenterPoint().x / this.canvas.getZoom(),
                            y: p.getCenterPoint().y / this.canvas.getZoom()
                        }
                    }
                })

                // Событие двойного нажатия на область отрисовки
                this.canvas.on('mouse:dblclick', (options) => {
                    if (options.target == null && !this.editing) {
                        this.drawPolygon()
                        this.addPoint(options);
                    } else if (options.target && options.target.type === 'polygon') {
                        this.editPolygon(options.target)
                    } else {
                        this.finishEditPolygon()
                        this.finishCreating()
                    }
                })

                // По нажатию на Esc недостроенный полигон отменяется
                fabric.util.addListener(document.body, 'keydown', (options) => {
                    if (options.keyCode === 27) {
                        this.finishCreating()
                    }
                })

            },

            /**
             * Добавление новой точки в полигон.
             * @param options
             */
            addPoint(options) {
                let random = `f${(~~(Math.random()*1e8)).toString(16)}`;
                let id = new Date().getTime() + random;
                let commonProps = {
                    fill: '#ffffff',
                    stroke: '#333333',
                    hasBorders: false,
                    hasControls: false,
                    objectCaching: false,
                    evented: false,
                    selectable: false,
                    strokeWidth: 1.5
                }
                let circle = new fabric.Circle({
                    ...commonProps,
                    radius: 5,
                    strokeWidth: 0.5,
                    left: (options.e.layerX / this.canvas.getZoom()),
                    top: (options.e.layerY / this.canvas.getZoom()),
                    originX: 'center',
                    originY: 'center',
                    evented: true,
                    id: id,
                })
                if (this.pointArray.length === 0) {
                    circle.set({
                        fill: 'red'
                    })
                    this.strokeColor = this.getRandomColor()
                }
                let points = [
                    (options.e.layerX / this.canvas.getZoom()),
                    (options.e.layerY / this.canvas.getZoom()),
                    (options.e.layerX / this.canvas.getZoom()),
                    (options.e.layerY / this.canvas.getZoom())
                ]
                let line = new fabric.Line(points, {
                    ...commonProps,
                    fill: this.strokeColor,
                    stroke: this.strokeColor,
                    class: 'line',
                    originX: 'center',
                    originY: 'center',
                })
                if (this.activeShape) {
                    let pos = this.canvas.getPointer(options.e)
                    let points = this.activeShape.get('points')
                    points.push({
                        x: pos.x,
                        y: pos.y
                    });
                    let polygon = new fabric.Polygon(points, {
                        ...commonProps,
                        opacity: 0.3,
                        accessory: true
                    })
                    this.canvas.remove(this.activeShape)
                    this.canvas.add(polygon)
                    this.activeShape = polygon
                    this.canvas.renderAll()
                } else {
                    let polyPoint = [{
                        x: (options.e.layerX / this.canvas.getZoom()),
                        y: (options.e.layerY / this.canvas.getZoom())
                    }]
                    let polygon = new fabric.Polygon(polyPoint, {
                        ...commonProps,
                        opacity: 0.3,
                        accessory: true
                    });
                    this.activeShape = polygon
                    this.canvas.add(polygon)
                }
                this.activeLine = line

                this.pointArray.push(circle)
                this.lineArray.push(line)

                this.canvas.add(line)
                this.canvas.add(circle)
                this.canvas.selection = false
            },

            /**
             * Отрисовка полигона.
             */
            drawPolygon() {
                this.finishEditPolygon()
                this.creating = true
                this.pointArray = []
                this.lineArray = []
                this.activeLine = null
            },

            /**
             * Создание полигона.
             * @param points - массив точек полигона.
             */
            generatePolygon(polygonPoints) {
                let points = [];
                polygonPoints.forEach((it) => {
                    points.push({
                        x: it.left || it.x,
                        y: it.top || it.y
                    });
                    this.canvas.remove(it);
                });
                this.lineArray.forEach((it) => {
                    this.canvas.remove(it)
                });

                this.canvas.remove(this.activeShape).remove(this.activeLine);
                let polygon = new fabric.Polygon(points, {
                    stroke: this.strokeColor,
                    strokeWidth: 1.5,
                    fill: 'rgba(255, 255 ,255, 0.3)',
                    hasBorders: false,
                    hasControls: false,
                    selectable: false,
                    objectCaching: false,
                    perPixelTargetFind: true,
                    description: `Полигон #${this.getPolygonsFromCanvas.length + 1}`})
                this.canvas.add(polygon);
                this.finishCreating()
            },

            /**
             * Создание полигона.
             * @param points - массив точек полигона.
             */
            loadPolygon(polygonPoints, description) {
                let points = [];
                polygonPoints.forEach((it) => {
                    window.console.log(this.imageScale)

                    window.console.log(it.x, it.x * this.imageScale)
                    points.push({
                        x: it.x * this.imageScale,
                        y: it.y * this.imageScale
                    });
                    this.canvas.remove(it);
                });
                let polygon = new fabric.Polygon(points, {
                    stroke: this.strokeColor,
                    strokeWidth: 1.5,
                    fill: 'rgba(255, 255 ,255, 0.3)',
                    hasBorders: false,
                    hasControls: false,
                    selectable: false,
                    objectCaching: false,
                    perPixelTargetFind: true,
                    description: description || `Полигон #${this.getPolygonsFromCanvas.length + 1}`})
                this.canvas.add(polygon);
            },

            /**
             * Возвращает случайный цвет.
             * @returns {string}
             */
            getRandomColor() {
                return  "hsl(" + Math.random() * 360 + ", 100%, 60%)"
            },

            /**
             * Отменяет действие создания полигона.
             */
            finishCreating() {
                this.canvas.selection = true;
                this.creating = false
                this.pointArray = []
                this.lineArray = []
                this.activeLine = null
                this.activeShape = false
                this.canvas
                    .getObjects()
                    .filter(it => it.type !== 'polygon' || it.accessory)
                    .forEach(it => this.canvas.remove(it))
            },

            /**
             * Удаляет полигон.
             */
            removePolygon(descr) {
                if (confirm(`Действительно удалить ${descr}?`)) {
                    this.finishEditPolygon()
                    this.finishCreating()

                    this.getPolygonsFromCanvas
                        .filter(it => {
                            return it.description === descr
                        })
                        .forEach(it => this.canvas.remove(it))
                }
                this.updateRegions()
            },

            /**
             * Редактирует полигон.
             */
            editPolygon(polygon) {
                this.finishEditPolygon()
                if (!this.editing) {
                    this.editing = true
                    this.activePolygon = polygon
                    polygon.points.forEach((it, index) => {
                        let circle = new fabric.Circle({
                            radius: 5,
                            fill: '#ffffff',
                            stroke: '#333333',
                            strokeWidth: 0.5,
                            left: (it.x / this.canvas.getZoom()),
                            top: (it.y / this.canvas.getZoom()),
                            hasBorders: false,
                            hasControls: false,
                            originX: 'center',
                            originY: 'center',
                            pointIndex: index
                        })
                        this.canvas.add(circle)
                    })
                    polygon.selectable = false
                    this.canvas.renderAll()
                }
            },

            /**
             * Заверщает редактирование полигона.
             */
            finishEditPolygon() {
                this.canvas.getObjects()
                    .filter(it => it.type === 'circle')
                    .forEach(it => this.canvas.remove(it))
                this.activePolygon = null
                this.editing = false
                this.updateRegions()
            },

            updateRegions() {
                window.console.log(this.imageScale)
                window.console.log(this.canvas.getZoom())
                this.$emit('input', this.getPolygonsFromCanvas.map(it => {
                    return {
                        description: it.description,
                        points: it.points.map(p => { return {
                            x: p.x / this.imageScale,
                            y: p.y / this.imageScale
                        }})
                    }
                }))
            }
        },
        computed: {
            /**
             * Возвращает массив полигонов, размещенных в canvas
             * @returns {*[]} - массив полигонов.
             */
            getPolygonsFromCanvas() {
                if (this.canvas !== null) {
                    return this.canvas.getObjects().filter(it => it.type === 'polygon' && !it.accessory)
                } else {
                    return []
                }
            }
        }
    }
</script>

<style scoped>
</style>
