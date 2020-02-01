<template>
    <v-app>
        <v-content>
            <Editor :background-image="backgroundImage" v-model="polygons"/>
        </v-content>
    </v-app>
</template>

<script>
    import Editor from './components/Editor'

    export default {
        name: 'App',

        components: {
            Editor
        },

        data: () => ({
            polygons: [{
                "description": "Poly 1",
                "points": [{"x": 30.627871362940276, "y": 30.627871362940276}, {
                    "x": 30.627871362940276,
                    "y": 61.25574272588055
                }, {"x": 61.25574272588055, "y": 61.25574272588055}]
            }, {
                "description": "Полигон #2",
                "points": [{"x": 2217.4578866768757, "y": 686.0393013698072}, {
                    "x": 2235.8346094946405,
                    "y": 989.4203411192785
                }, {"x": 2395.0995405819294, "y": 1014.2828853144218}, {"x": 2370.597243491577, "y": 723.1179700860497}]
            }, {
                "description": "Полигон #3",
                "points": [{"x": 827.2539080674113, "y": 513.2438986998152}, {
                    "x": 824.8051852647163,
                    "y": 845.7029246356242
                }, {"x": 638.9546590718587, "y": 850.6158839065034}, {"x": 634.9123828144868, "y": 557.7245111867677}]
            }],
            backgroundImage: 'https://nash-sever.info/wp-content/uploads/2019/02/%D0%91%D0%B5%D0%B3_%D0%97%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B0-2000x1200.jpg'
        }),
    }
</script>
