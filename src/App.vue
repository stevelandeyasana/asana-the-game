<template>
  <div class="container">
    <input type="text" v-model="authToken" placeholder="your auth token">
    <button v-on:click="storeAuthToken">Store auth token</button>
  </div>
</template>

<script>
import wretch from "wretch";

let api = null;

const Constants = {
  workspaceID: "15793206719",
  projectID: "916537440743762",
};

function beginLoad() {
  api = wretch()
    .url("https://app.asana.com/api/1.0/")
    .auth(`Bearer ${ localStorage.authToken }`);

  api.url(`projects/${Constants.projectID}/tasks?opt_fields=name,custom_fields&opt_expand=custom_fields`).get().json((result) => {
    console.log("Tasks:", result);
    const situationData = result.data.map(taskToSituation);
    console.log("Situations:", situationData);
  });
}

function taskToSituation(task) {
  const result = {
    id: task.name,
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
  return result;
}

export default {
  name: 'App',
  created: function() {
    if (localStorage.authToken) {
      beginLoad();
    }
  },
  data() {
    return {
      authToken: "",
    };
  },
  methods: {
    storeAuthToken() {
      localStorage.authToken = this.authToken;
      this.authToken = "";
      beginLoad();
    }
  }
};
</script>

<style scoped>
  .container {
    /* width: 600px;
    margin: 50px auto;
    text-align: center; */
  }
</style>