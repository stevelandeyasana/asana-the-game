<template>
  <div class="container">
    <div v-if="!hasLoaded">
      <input type="text" v-model="authToken" placeholder="your auth token">
      <button v-on:click="storeAuthToken">Store auth token</button>
    </div>
    <div v-if="isLoading">Loading...</div>
    <div ref="gameContainer" id="js-gameContainer"></div>
  </div>
</template>

<script>
import wretch from "wretch";
import {jumbogrove} from "jumbogrove";
import {Howl, Howler} from 'howler';

let api = null;

const Constants = {
  workspaceID: "15793206719",
  projectID: "916537440743762",
};

function beginLoad(onComplete) {
  api = wretch()
    .url("https://app.asana.com/api/1.0/")
    .auth(`Bearer ${ localStorage.authToken }`);

  api.url(`projects/${Constants.projectID}/tasks?opt_fields=name,custom_fields,html_notes,attachments&opt_expand=custom_fields,attachments`).get().json((result) => {
    console.log("Tasks:", result);
    const situationData = result.data.map(taskToSituation);
    console.log("Situations:", situationData);
    if (onComplete) {
      onComplete(situationData);
    }
  });
}

const attachmentIndex = {};
const musicIndex = {};
window.musicIndex = musicIndex;

function taskToSituation(task) {
  const result = {
    id: task.name,
    content: parseHTML(task.html_notes),
  };
  for (let field of task.custom_fields) {
    if (field.name.indexOf("jg_") === 0) {
      const name = field.name.slice(3);
      if (field.text_value) {
        result[name] = field.text_value;
      } else if (field.number_value !== undefined) {
        result[name] = field.number_value;
      }
    }
  }

  if (result.choices) {
    result.choices = result.choices.split(',').map((c) => c.trim());
  }

  if (result.tags) {
    result.tags = result.tags.split(',').map((c) => c.trim());
  }

  if (result.music && result.music !== "no music") {
    musicIndex[result.music] = new Howl({
      src: [result.music + '.mp3'],
      loop: true,
      // volume: 0.5,
    });
  }

  result.enter = (model, ui, fromSituation) => {
    model.globalState[`visitedScenes.${result.id}`] = true;
    if (result.set) {
      for (let item of result.set.split(',')) {
        model.globalState[item.trim()] = true;
      }
    }
    if (result.unset) {
      for (let item of result.unset.split(',')) {
        model.globalState[item.trim()] = false;
      }
    }
    if (result.music && model.globalState.music != result.music) {
      for (let k of Object.keys(musicIndex)) {
        musicIndex[k].stop();
      }
      model.globalState.music = result.music;
      if (result.music == "no music") return;
      console.log("play", result.music);
      musicIndex[result.music].play();
    }

    task.attachments.forEach(({id}) => {
      if (attachmentIndex[id]) {
        ui.writeHTML(`<img id="image-${id}" src="${attachmentIndex[id]}">`);
      } else {
        ui.writeHTML(`<img id="image-${id}" style="display: none;">`);
      }
    });

    return true;
  };

  result.getCanSee = (model, hostSituation) => {
    if (result.require) {
      for (let item of result.require.split(',')) {
        const k = item.trim();
        if (result.require[0] == "!") {
          k = k.slice(1);
          if ((model.globalState[k] || false)) { return false }
        } else if (!(model.globalState[k] || false)) {
          return false;
        }
      }
    }
    return true;
  };

  task.attachments.forEach(({id}) => loadAttachment(id));

  return result;
}

function loadAttachment(id) {
  api.url(`attachments/${id}`).get().json((result) => {
    const url = result.data.download_url;
    const image = new Image();
    image.src = url;
    attachmentIndex[id] = url;
    for (let el of document.querySelectorAll(`#image-${id}`)) {
      el.src = url;
      el.style.display = 'inline';
    }
  });
}

function parseHTML(html) {
  return html.replace("<body>", "").replace("</body>", "");
}

export default {
  name: 'App',
  created: function() {
    if (localStorage.authToken) {
      this.isLoading = true;
      beginLoad(this.play.bind(this));
    }
  },
  data() {
    return {
      authToken: "",
      isLoading: false,
      hasLoaded: false,
    };
  },
  methods: {
    storeAuthToken() {
      localStorage.authToken = this.authToken;
      this.authToken = "";
      this.isLoading = true;
      beginLoad(this.play.bind(this));
    },
    play(situationData) {
      this.hasLoaded = true;
      this.isLoading = false;
      jumbogrove('#' + this.$refs.gameContainer.id, {
        id: 'asana-the-game',
        showAside: false,
        situations: situationData,
      });
    },
  },
};
</script>

<style scoped>
</style>