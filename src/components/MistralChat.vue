<template>
  <div class="chat-mistral">
    <h1 class="title">Chat avec l'API Mistral</h1>
    <form class="form" @submit.prevent="sendQuestion">
      <div class="data">
        <label class="apikeylabel" for="apiKey">API Key:</label>
        <input class="apikeyinput" type="text" id="apiKey" v-model="apiKey" />
      </div>

      <div class="models">
        <label class="modellabel" for="modelSelect">Model:</label>
        <select class="modelselect" id="modelSelect" v-model="selectedModel">
          <option disabled value="">Model</option>
          <option value="mistral-tiny">Mistral Tiny</option>
          <option value="mistral-small">Mistral Small</option>
          <option value="mistral-medium">Mistral Medium</option>
        </select>
      </div>
      <div class="chat-area">
        <div
          v-for="(message, index) in messages"
          :key="index"
          class="message"
          :class="{
            user: message.role === 'user',
            assistant: message.role === 'assistant',
          }"
        >
          <span class="icon" v-if="message.role === 'user'">&#128100;</span>
          <span class="icon" v-if="message.role === 'assistant'"
            >&#128187;</span
          >
          <div class="messagecontent" v-html="message.content"></div>
        </div>
        <div class="loading" v-if="loading">...</div>
      </div>
      <div class="questioncontainer">
        <textarea
          id="question"
          v-model="userQuestion"
          @keydown="handleTextareaKeydown"
        ></textarea>
        <button class="submitbutton" type="submit">Send</button>
      </div>
    </form>
  </div>
</template>

<script>
import axios from "axios";

export default {
  name: "MistralChat",
  data() {
    return {
      apiKey: "",
      selectedModel: "",
      userQuestion: "",
      response: null,
      loading: false,
      messages: [],
    };
  },
  mounted() {
    this.apiKey = localStorage.getItem("apiKey") || "";
    this.selectedModel = localStorage.getItem("selectedModel") || "";

    const storedMessages = localStorage.getItem("messages");
    if (storedMessages) {
      this.messages = JSON.parse(storedMessages);
    }
    this.scrollToBottom();
  },
  watch: {
    apiKey(newVal) {
      localStorage.setItem("apiKey", newVal);
    },
    selectedModel(newVal) {
      localStorage.setItem("selectedModel", newVal);
    },
    messages: {
      handler(newMessages) {
        localStorage.setItem("messages", JSON.stringify(newMessages));
        this.scrollToBottom();
      },
      deep: true,
      immediate: true,
    },
  },
  methods: {
    scrollToBottom() {
      this.$nextTick(() => {
        const chatArea = this.$el.querySelector(".chat-area");
        if (chatArea) {
          chatArea.scrollTop = chatArea.scrollHeight;
        }
      });
    },
    escapeHtml(code) {
      return code.replace(/</g, "&lt;").replace(/>/g, "&gt;");
    },
    handleTextareaKeydown(event) {
      if (event.key === "Enter" && !event.shiftKey) {
        event.preventDefault();
        this.sendQuestion();
      }
    },
    sendQuestion() {
      this.loading = true;
      this.response = null;

      const userQuestion = this.userQuestion;
      this.userQuestion = "";

      this.messages.push({ role: "user", content: userQuestion });

      axios
        .post(
          "https://api.mistral.ai/v1/chat/completions",
          {
            model: this.selectedModel,
            messages: [{ role: "user", content: userQuestion }],
          },
          {
            headers: {
              "Content-Type": "application/json",
              Accept: "application/json",
              Authorization: `Bearer ${this.apiKey}`,
            },
          }
        )
        .then((response) => {
          if (response.data.choices && response.data.choices.length > 0) {
            let content = response.data.choices[0].message.content;
            this.messages.push({
              role: "assistant",
              content: this.formatCodeResponse(content),
            });
          } else {
            this.messages.push({
              role: "assistant",
              content: "Pas de réponse de l'API.",
            });
          }
          this.scrollToBottom();
        })
        .catch((error) => {
          console.error("Erreur API: ", error);
          this.messages.push({
            role: "assistant",
            content: "Erreur lors de la connexion à l'API Mistral.",
          });
        })
        .finally(() => {
          this.loading = false;
        });
    },
    formatCodeResponse(content) {
      let escapedContent = this.escapeHtml(content);

      if (escapedContent.includes("```")) {
        const segments = escapedContent.split("```");
        let formattedContent = "";

        for (let i = 0; i < segments.length; i++) {
          formattedContent +=
            (i % 2 === 1 ? "<pre><code>" : "") +
            segments[i] +
            (i % 2 === 1 ? "</code></pre>" : "");
        }

        return formattedContent;
      } else {
        return escapedContent;
      }
    },
  },
};
</script>

<style lang="scss">
#app {
  margin: 0 5%;
  font-size: 32px;
}
* {
  color: white;
  background-color: rgb(30, 30, 30);
}
.chat-mistral {
  height: 96vh;
  .title {
    height: 10%;
    margin: 0;
  }
  .form {
    height: 90%;
    gap: 2%;
    display: flex;
    flex-direction: column;
    .data,
    .models {
      display: flex;
      align-items: center;

      .apikeylabel,
      .modellabel {
        margin-right: 10px;
        font-weight: bold;
        color: white;
        text-align: left;
      }

      .apikeyinput,
      .modelselect {
        flex-grow: 1;
        padding: 8px;
        border-radius: 4px;
        border: 1px solid #ccc;
        font-size: 32px;
      }
    }
    .modelselect {
      display: block;
      width: 100%;
    }
    .chat-area {
      height: 80%;
      overflow-y: auto;
      padding: 10px;
      border: 1px solid #ccc;
      border-radius: 10px;
      display: flex;
      flex-direction: column;
      .message {
        margin-bottom: 10px;
        padding: 10px;
        border-radius: 5px;
        display: flex;
        font-size: 32px;
        border-bottom: solid 1px white;
        .icon {
          margin-right: 10px;
          font-size: 30px;
        }
        .messagecontent {
          pre {
            font-family: "Courier New", Courier, monospace;
            color: white;
            text-align: left;
            code {
              background: black;
              padding: 10px;
              border-radius: 5px;
              overflow: hidden;
              justify-content: left;
              display: flex;
            }
          }
        }
      }
      .user {
        text-align: left;
      }
      .assistant {
        text-align: left;
      }
      .loading {
        font-weight: bold;
        font-size: 40px;
        text-align: center;
        animation: blinker 1.5s linear infinite;
      }
    }
    .questioncontainer {
      display: flex;
      gap: 10px;
      #question {
        width: 100%;
        padding: 2vh 10px;
        font-size: 32px;
      }
    }
    .submitbutton {
      font-size: 32px;
      border-radius: 20px;
    }
  }
}

@keyframes blinker {
  50% {
    opacity: 0;
  }
}
</style>
