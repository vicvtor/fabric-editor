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
                    <v-toolbar color="light-blue" dark >
                        <v-toolbar-title>Регионы</v-toolbar-title>
                        <v-spacer></v-spacer>
                        <v-btn icon @click="drawPolygon">
                            <v-icon>mdi-plus</v-icon>
                        </v-btn>
                    </v-toolbar>
                    <v-list class="pa-2">
                        <v-alert type="info" v-if="this.getCanvasObjects.length === 0">
                            Размеченные регионы отсутствуют.
                        </v-alert>
                        <v-list-item two-line v-for="(item, i) in getCanvasObjects"
                                     :key="i">
                            <v-list-item-avatar :color=getRegionColor(item)>
                                <span class="white--text headline"></span>
                            </v-list-item-avatar>
                            <v-list-item-content>
                                <v-list-item-title>{{item.description}}</v-list-item-title>
                                <v-list-item-subtitle>{{JSON.stringify(item)}}</v-list-item-subtitle>
                            </v-list-item-content>
                            <v-list-item-action>
                                <v-btn icon>
                                    <v-icon color="grey lighten-1">mdi-pencil</v-icon>
                                </v-btn>
                            </v-list-item-action>
                            <v-list-item-action>
                                <v-btn icon @click="removePolygon(item.description)">
                                    <v-icon color="grey lighten-1">mdi-delete</v-icon>
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
                startEditing: false,
                regions: [],
                pointArray: [],
                lineArray: [],
                activeLine: null,
                activeRegion: null,
                activeShape: false,
                imageScale: 1
            }
        },
        props: {
            backgroundImage: null
        },
        mounted() {
            this.initializeCanvas()
        },
        methods: {

            /**
             * Возвращает цвет для иконки региона в списке. Игнорирует черный.
             */
            getRegionColor(region) {
                return (region.stroke === '#000000') ? '#FFFFFF' : region.stroke
            },

            /**
             * Инициализация canvas.
             */
            initializeCanvas() {
                var holder = document.getElementById(this.holderId)
                this.canvas = new fabric.Canvas('canvas')

                fabric.Image.fromURL(this.backgroundImage, (oImg) => {

                    if (oImg.width > holder.clientWidth) {
                        oImg.scaleToWidth(holder.clientWidth)
                    }

                    this.canvas.setWidth(oImg.width * oImg.scaleX)
                    this.canvas.setHeight(oImg.height * oImg.scaleX)
                    this.canvas.setBackgroundImage(oImg)
                });

                this.canvas.on('mouse:down', (options) => {
                    if (options.target && options.target.id === this.pointArray[0].id) {
                        this.generatePolygon(this.pointArray);
                    }
                    if (this.startEditing) {
                        this.addPoint(options);
                    }
                });

                this.canvas.on('mouse:move', (options) => {
                    if (this.activeLine && this.activeLine.class === "line") {
                        var pointer = this.canvas.getPointer(options.e);
                        this.activeLine.set({x2: pointer.x, y2: pointer.y});

                        var points = this.activeShape.get("points");
                        points[this.pointArray.length] = {
                            x: pointer.x,
                            y: pointer.y
                        }
                        this.activeShape.set({
                            points: points
                        });
                        this.canvas.renderAll();
                    }
                    this.canvas.renderAll();
                });

                // По нажатию на Esc недостроенный полигон отменяется
                fabric.util.addListener(document.body, 'keydown', (options) => {
                    if (options.keyCode === 27) {
                        this.completeEditing()
                    }
                });

            },

            /**
             * Добавление новой точки в полигон.
             * @param options
             */
            addPoint(options) {
                var random = Math.floor(Math.random() * (999999 - 99 + 1)) + 99;
                var id = new Date().getTime() + random;
                var circle = new fabric.Circle({
                    radius: 5,
                    fill: '#ffffff',
                    stroke: '#333333',
                    strokeWidth: 0.5,
                    left: (options.e.layerX / this.canvas.getZoom()),
                    top: (options.e.layerY / this.canvas.getZoom()),
                    selectable: false,
                    hasBorders: false,
                    hasControls: false,
                    originX: 'center',
                    originY: 'center',
                    id: id,
                    objectCaching: false
                });
                if (this.pointArray.length == 0) {
                    circle.set({
                        fill: 'red'
                    })
                }
                var points1 = [(options.e.layerX / this.canvas.getZoom()), (options.e.layerY / this.canvas.getZoom()), (options.e.layerX / this.canvas.getZoom()), (options.e.layerY / this.canvas.getZoom())];
                var line = new fabric.Line(points1, {
                    strokeWidth: 2,
                    fill: '#999999',
                    stroke: '#999999',
                    class: 'line',
                    originX: 'center',
                    originY: 'center',
                    selectable: false,
                    hasBorders: false,
                    hasControls: false,
                    evented: false,
                    objectCaching: false
                });
                if (this.activeShape) {
                    var pos = this.canvas.getPointer(options.e);
                    var points = this.activeShape.get("points");
                    points.push({
                        x: pos.x,
                        y: pos.y
                    });
                    var polygon1 = new fabric.Polygon(points, {
                        stroke: '#333333',
                        strokeWidth: 1,
                        fill: '#cccccc',
                        opacity: 0.3,
                        selectable: false,
                        hasBorders: false,
                        hasControls: false,
                        evented: false,
                        objectCaching: false,
                        accessory: true

                    });
                    this.canvas.remove(this.activeShape);
                    this.canvas.add(polygon1);
                    this.activeShape = polygon1;
                    this.canvas.renderAll();
                } else {
                    var polyPoint = [{
                        x: (options.e.layerX / this.canvas.getZoom()),
                        y: (options.e.layerY / this.canvas.getZoom())
                    }];
                    var polygon = new fabric.Polygon(polyPoint, {
                        stroke: '#333333',
                        strokeWidth: 1,
                        fill: '#cccccc',
                        opacity: 0.3,
                        selectable: false,
                        hasBorders: false,
                        hasControls: false,
                        evented: false,
                        objectCaching: false,
                        accessory: true
                    });
                    this.activeShape = polygon;
                    this.canvas.add(polygon);
                }
                this.activeLine = line;

                this.pointArray.push(circle);
                this.lineArray.push(line);

                this.canvas.add(line);
                this.canvas.add(circle);
                this.canvas.selection = false;
            },

            /**
             * Отрисовка полигона.
             */
            drawPolygon() {
                this.startEditing = true;
                this.pointArray = [];
                this.lineArray = [];
                this.activeLine = null;
            },

            /**
             * Создание полигона.
             * @param points
             */
            generatePolygon(polygonPoints) {
                let points = [];
                polygonPoints.forEach((it) => {
                    window.console.log(it)
                    points.push({
                        x: it.left,
                        y: it.top
                    });
                    this.canvas.remove(it);
                });
                this.lineArray.forEach((it) => {
                    this.canvas.remove(it);
                });

                this.canvas.remove(this.activeShape).remove(this.activeLine);
                let polygon = new fabric.Polygon(points, {
                    stroke: this.getRandomColor(),
                    strokeWidth: 2.5,
                    fill: 'rgba(255, 255 ,255, 0.3)',
                    hasBorders: false,
                    hasControls: false,
                    description: `Полигон #${this.getCanvasObjects.length + 1}`
                });
                this.canvas.add(polygon);
                this.completeEditing()
            },

            /**
             * Возвращает лучайный цвет.
             * @returns {string}
             */
            getRandomColor() {
                return "#" + ((1 << 24) * Math.random() | 0).toString(16)
            },

            /**
             * Отменяет действие создания полигона.
             */
            completeEditing() {
                this.canvas.selection = true;
                this.startEditing = false
                this.pointArray = []
                this.lineArray = []
                this.activeLine = null
                this.activeRegion = null
                this.activeShape = false
                if (this.canvas !== null) {
                    this.canvas
                        .getObjects()
                        .filter(it => it.type !== 'polygon' || it.accessory)
                        .forEach(it => this.canvas.remove(it))
                }

            },

            removePolygon(descr) {
                window.console.log(descr)
                this.getCanvasObjects
                    .filter(it => {
                        window.console.log(it.description)
                        return it.description === descr
                    })
                    .forEach(it => this.canvas.remove(it))
            }
        },
        computed: {
            /**
             * Возвращает массив полигонов, размещенных в canvas
             * @returns {*[]} - массив полигонов.
             */
            getCanvasObjects() {
                if (this.canvas !== null) {
                    window.console.log(JSON.stringify(this.canvas.getObjects()[0]))
                    return this.canvas.getObjects().filter(it => it.type === 'polygon' && !it.accessory)
                } else {
                    return []
                }
            },
        }
    }
</script>

<style scoped>
    canvas {
        border: 1px solid #cccccc;
    }
</style>
