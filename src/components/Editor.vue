<template>
    <v-container fluid fill-height>
        <v-row class="align-stretch align-self-stretch">
            <v-col md="8">
                <v-layout class="fill-height container--fluid" :id="this.holderId">
                    <canvas id="canvas"/>
                </v-layout>
            </v-col>
            <v-col md="4">
                <p>rois</p>
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
                painingMode: false,
                regions: [],
                pointArray: [],
                lineArray: [],
                activeLine: null,
                activeRegion: null,
                activeShape: false,
                strokeColor: '#000000',
                fillColor: '#cccccc',
                opacity: 1
            }
        },
        props: {},
        mounted() {
            this.initializeCanvas()
            this.drawPolygon()
        },
        methods: {
            /**
             * Инициализация canvas.
             */
            initializeCanvas() {
                let canvasHolder = document.getElementById(this.holderId)
                this.canvas = new fabric.Canvas('canvas')
                this.canvas.setWidth(canvasHolder.clientWidth)
                this.canvas.setHeight(canvasHolder.clientHeight)

                this.canvas.on('mouse:down', (options) => {
                    if (options.target && options.target.id === this.pointArray[0].id) {
                        this.generatePolygon(this.pointArray);
                    }
                    if (this.painingMode) {
                        this.addPoint(options);
                    }
                })

                this.canvas.on('mouse:up', (options) => {
                    window.console.log(options)
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
                        objectCaching: false
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
                        objectCaching: false
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
                this.painingMode = true;
                this.pointArray = [];
                this.lineArray = [];
                this.activeLine = null;
            },

            /**
             * Создание полигона.
             * @param points
             */
            generatePolygon(polygonPoints) {
                var points = [];
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
                var polygon = new fabric.Polygon(points, {
                    stroke: '#333333',
                    strokeWidth: 0.5,
                    fill: 'red',
                    opacity: 1,
                    hasBorders: false,
                    hasControls: false
                });
                this.canvas.add(polygon);

                this.activeLine = null;
                this.activeShape = null;
                this.painingMode = false;
                this.canvas.selection = true;
            },

            /**
             * Случайный цвет.
             * @returns {string}
             */
            randomColor() {
                return "#" + ((1 << 24) * Math.random() | 0).toString(16)
            }
        }
    }
</script>

<style scoped>
    canvas {
        border: 1px solid #cccccc;
    }
</style>
