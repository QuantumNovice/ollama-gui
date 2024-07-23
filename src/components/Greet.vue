<script setup>
import { ref, onMounted, watch } from "vue";
import { invoke } from "@tauri-apps/api/tauri";
import axios from 'axios';
import * as Ollama from 'ollama/browser';
import MarkdownIt from "markdown-it";
import MarkdownItHighlightjs from "markdown-it-highlightjs";
import MarkdownItSub from "markdown-it-sub";
import MarkdownItSup from "markdown-it-sup";
import MarkdownItTasklists from "markdown-it-task-lists";
import MarkdownItTOC from "markdown-it-toc-done-right";




const markdown = new MarkdownIt()
  .use(MarkdownItHighlightjs)
  .use(MarkdownItSub)
  .use(MarkdownItSup)
  .use(MarkdownItTasklists)
  .use(MarkdownItTOC);;



//state

const greetMsg = ref("");
const text = ref("");
const models = ref(null)
const selected_model = ref('llama3:latest')
const response_q = ref([]);
const questsion_q = ref([]);
const page = ref('chat')

// DOM
const chat_div = ref(null)

const ollama = new Ollama.Ollama({ host: 'http://127.0.0.1:11434' })



async function greet() {
  // Learn more about Tauri commands at https://tauri.app/v1/guides/features/command
  greetMsg.value = await invoke("greet", { name: text.value });
  questsion_q.value.push(text.value)
  //generateResponse();
  await chat()
  text.value = ''
}

async function chat() {
  const response = await ollama.chat({
    model: selected_model.value,
    messages: [{ role: 'user', content: text.value }],
    stream: true
  })
  response_q.value.push([])
  //console.log(response_q.value)
  for await (const part of response) {
    //console.log(part.message.content)
    response_q.value.at(-1).push(part.message.content)
  }
  //console.log(response)


}

async function fetch_models() {
  models.value = await ollama.list()
}
// Lifecycle hook to perform actions when the component is mounted
onMounted(() => {



  fetch_models()

});

const scrollToBottom = () => {
  if (chat_div.value) {
    chat_div.value.scrollTop = chat_div.value.scrollHeight;
  }
};

watch(questsion_q.value, (n, o) => {

  console.log("watchers")
  // Scroll to bottom whenever items change
  scrollToBottom();
});


</script>

<template>

  <div class="container">

    <!-- Left Card-->
    <div class="left">

      <button @click="page = 'chat'">Chat</button>
      <button @click="page = 'character'">Character</button>

    </div>

    <!-- Right Card-->
    <div class="right">
      <h1>ITalab GPT</h1>
      <!-- Settings -->
      <div class="settings" v-if="models != null">

        <select v-model="selected_model">
          <option disabled value="">Please select one</option>
          <option v-for="model in models.models">{{ model.name }}</option>

        </select>

      </div>

      <!--Output-->
      <div v-if="page == 'chat'">
        <div ref="chat_div" class="output" v-if="response_q != null">
          <div v-for="message_block in response_q">
            <br>
            <div class="question">

              {{ questsion_q[response_q.indexOf(message_block)] }}
              <b>:Ask</b>
            </div>

            <b>Assistant:</b>
            <!--
          <span v-for="i in message_block" >
            {{ i }}
          </span>
          -->
            <div class="llm-answer" v-html="markdown.render(message_block.join(''))" />

          </div>
        </div>
      </div>
      <div class="input">
        <textarea class="text-area" v-model="text"></textarea>
        <button @click="greet" type="submit">Submit</button>

      </div>
    </div>
  </div>



</template>

<style>
button {
  display: inline-block;
  padding: 12px 24px;
  font-size: 16px;
  font-weight: bold;
  color: #fff;
  text-align: center;
  text-decoration: none;
  background-image: linear-gradient(45deg, #ff6b6b, #f06595);
  border: none;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  cursor: pointer;
  margin: 2px;
}

button:hover {
  background-image: linear-gradient(45deg, #ff6b6b, #ec407a);
  box-shadow: 0 6px 10px rgba(0, 0, 0, 0.15);
  transform: translateY(-2px);
}

button:active {
  background-image: linear-gradient(45deg, #ff6b6b, #d32f2f);
  box-shadow: 0 3px 6px rgba(0, 0, 0, 0.2);
  transform: translateY(0);
}

.container {
  display: flex;
  flex-direction: row;
  overflow: auto;

}

.left {
  width: 20vw;
  height: 98vh;
  background-color: #add9f952;
  margin-right: 5px;
  border: 1px solid #ccc;
  border-radius: 10px;
  padding: 4px;
}

.right {
  width: 80vw;
  display: flex;
  flex-direction: column;
  background-color: rgb(255, 255, 255);
  height: 98vh;
  border: 1px solid #ccc;
  border-radius: 10px;
  padding: 2px;
}

.settings {
  background-color: rgba(255, 255, 255, 0.763);
}

.input {

  margin-top: auto;
  background-color: rgb(255, 255, 255);
}



.text-area {
  height: 10vh;
  width: 92%;
  padding: 15px;
  border: 2px solid #ccc;
  border-radius: 10px;
  background-color: #fff;
  font-size: 16px;
  color: #333;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  resize: vertical;
  transition: border-color 0.3s, box-shadow 0.3s;
  resize: none;

}

.text-area:focus {
  border-color: #66afe9;
  box-shadow: 0 4px 8px rgba(0, 123, 255, 0.2);
  outline: none;
}

.text-area::placeholder {
  color: #aaa;
}

.output {
  overflow-y: auto;
}

.question {
  text-align: right;
  border: 1px solid #cccccc;
  border-radius: 10px;
  padding: 4px;
  background-color: rgba(58, 164, 240, 0.416);
}

.llm-answer {
  border: 1px solid #ccc;
  border-radius: 10px;
  padding-left: 4px;
  background-color: rgba(47, 212, 207, 0.34);
}
</style>