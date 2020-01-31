<template>
  <v-app>
    <v-content>
      <Editor :background-image="backgroundImage"/>
    </v-content>
  </v-app>
</template>

<script>
import Editor from './components/Editor';

export default {
  name: 'App',

  components: {
    Editor
  },

  data: () => ({
    backgroundImage: 'https://nash-sever.info/wp-content/uploads/2019/02/%D0%91%D0%B5%D0%B3_%D0%97%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B0-2000x1200.jpg'
  }),
};
</script>
