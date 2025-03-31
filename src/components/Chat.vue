<template>
  <v-container
    style="height: 100vh"
    class="d-flex flex-column justify-space-between"
  >
    <v-row>
      <v-col cols="12">
        <!-- Messages Display in Individual Cards -->
        <div ref="messagesContainer" id="messages-container">
          <div 
            v-for="msg in messages"
            :key="msg.content"
            :class="['d-flex','mb-4', 'px-2', msg.role === 'user' ? 'justify-end' : 'justify-start']"
          >
            <Message :msg="msg" />
          </div>
        </div>
      </v-col>
    </v-row>

    <v-row
      v-if="messages.length === 0"
      class="justify-space-between flex-grow-0"
    >
      <v-spacer></v-spacer>
      <v-card
        class="ma-4"
        variant="tonal"
        color="primary"
        @click="handleSubmitTemplatePrompt('Build a python game')"
      >
        <v-card-text>Build a python game</v-card-text>
      </v-card>
      <v-card
        class="ma-4"
        variant="tonal"
        color="primary"
        @click="handleSubmitTemplatePrompt('Help me write a manifesto')"
      >
        <v-card-text>Help me write a manifesto</v-card-text>
      </v-card>
      <v-card
        class="ma-4"
        variant="tonal"
        color="primary"
        @click="handleSubmitTemplatePrompt('Assist in a task')"
      >
        <v-card-text>Assist in a task</v-card-text>
      </v-card>
      <v-spacer></v-spacer>
    </v-row>

    <v-row class="flex-grow-0">
      <v-col cols="12">
        <v-textarea
          v-model="message"
          label="Enter your message"
          variant="outlined"
          @click:append="sendMessage"
          @keydown.enter.prevent="sendMessage"
          single-line
          rows="2"
          no-resize
          rounded="lg"
        >
          <template v-slot:append-inner>
            <v-btn
              @click="sendMessage"
              :disabled="asyncState.isLoading"
              icon="mdi-send"
              variant="plain"
            ></v-btn>
            <v-btn
              @click="recordMessage"
              :disabled="asyncState.isLoading || message.length>0"
              :icon="recognizing ? 'mdi-record' :'mdi-microphone'"
              :color="recognizing ? 'red-lighten-3': 'default'"
              :class="recognizing ? 'animated' : 'not-animated'"
              variant="plain"
            ></v-btn>
          </template>
        </v-textarea>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup lang="ts">
import { IMessage } from "@/models/message";
import { nextTick, reactive, ref, watch } from "vue";
import Message from "./Message.vue";


const messagesContainer = ref<HTMLDivElement | null>(null);
const message = ref("");
const messages = ref<IMessage[]>([]);
const asyncState = reactive({ isLoading: false, error: null });

const openAIKey = import.meta.env.VITE_APP_OPENAI_KEY;

async function fetchOpenAIResponse(userMessage: string) {
  const copyOfUserMessage = userMessage;
  asyncState.isLoading = true;
  const currentMessageIndex = messages.value.length;
  messages.value.push({ role: "system", content: "", isLoading:true });
  try {
    const response = await fetch("https://api.openai.com/v1/chat/completions", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${openAIKey}`,
      },
      body: JSON.stringify({
        model: "gpt-3.5-turbo",
        messages: [...messages.value, { role: "user", content: userMessage }],
        stream: true,
      }),
    });
    const reader = response.body?.getReader();
    while (true && reader) {
      const { done, value } = await reader.read();
      if (done) break;
      const decoded = new TextDecoder("utf-8").decode(value);
      const lines = decoded
        .replaceAll(/^data: /gm, "")
        .split("\n")
        .filter((c) => c);

      for (let line of lines) {
        if (line !== "[DONE]") {
          try {
            const chunk = JSON.parse(line);
            if (
              chunk.choices &&
              chunk.choices[0] &&
              chunk.choices[0].delta &&
              chunk.choices[0].delta.content
            ) {
              messages.value[currentMessageIndex].content +=
                chunk.choices[0].delta.content;
            }
          } catch (error) {
            console.error("Error parsing JSON line:", error, line);
          }
        }
      }
    }
  } catch (error) {
    message.value = copyOfUserMessage; // Restore the user message
    console.error("Error fetching OpenAI response:", error);
    return "Sorry, I couldn't process that.";
  } finally {
    asyncState.isLoading = false;
    messages.value[currentMessageIndex].isLoading = false;
  }
}

async function sendMessage() {
  if (asyncState.isLoading) return;
  if (message.value.trim()) {
    // Append user message to the chat history
    messages.value.push({ role: "user", content: message.value, isLoading: false });
    const userInput = message.value; // Save the user input before clearing the message
    message.value = "";

    // Fetch LLM response and append it to the chat history
    const LlmResponse = await fetchOpenAIResponse(userInput);
  }
}

var final_transcript = ref('');
var recognizing = ref(false);
var ignore_onend: boolean;
var start_timestamp: number;
const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
const recognition = new SpeechRecognition();
recognition.continuous = true;
recognition.interimResults = true;

interface SpeechRecognitionEvent extends Event {
  timeStamp: number;
}

interface SpeechRecognitionErrorEvent extends SpeechRecognitionEvent {
  error: string;
}

interface SpeechRecognitionResultEvent extends SpeechRecognitionEvent {
  results: any[];
  resultIndex: number;
}

recognition.onstart = function() {
  recognizing.value = true;
  showInfo('info_speak_now');
};

recognition.onerror = function(event: SpeechRecognitionErrorEvent) {
  if (event.error == 'no-speech') {
    showInfo('info_no_speech');
    ignore_onend = true;
  }
  if (event.error == 'audio-capture') {
    showInfo('info_no_microphone');
    ignore_onend = true;
  }
  if (event.error == 'not-allowed') {
    if (event.timeStamp - start_timestamp < 100) {
      showInfo('info_blocked');
    } else {
      showInfo('info_denied');
    }
    ignore_onend = true;
  }
};

recognition.onend = function() {
  recognizing.value = false;
  if (ignore_onend) {
    return;
  }
  if (!final_transcript.value) {
    showInfo('info_no_transcript');
    return;
  }
  const finalSpan = document.getElementById('final_span');
  if (finalSpan && window.getSelection) {
    const selection = window.getSelection();
    if (selection) {
      selection.removeAllRanges();
      const range = document.createRange();
      range.selectNode(finalSpan);
      selection.addRange(range);
    }
  }
};

recognition.onresult = function(event: SpeechRecognitionResultEvent) {
  var interim_transcript = '';
  if (typeof(event.results) == 'undefined') {
    recognition.onend = null;
    recognition.stop();
    return;
  }
  for (var i = event.resultIndex; i < event.results.length; ++i) {
    if (event.results[i].isFinal) {
      final_transcript.value += event.results[i][0].transcript;
    } else {
      interim_transcript += event.results[i][0].transcript;
    }
  }
  final_transcript.value = capitalize(final_transcript.value);
  message.value = final_transcript.value
};

function recordMessage(event: SpeechRecognitionEvent) {
  if (recognizing.value) {
    recognition.stop();
    recognizing.value = false;
    return;
  }
  final_transcript.value = '';
  recognition.lang = 'fr-FR';
  console.log("LANG :", recognition.lang)
  recognition.start();
  ignore_onend = false;
  start_timestamp = event.timeStamp;
}

const showInfo = (message: string) => {
  console.log(message)
}

var first_char = /\S/;
function capitalize(s: string) {
  return s.replace(first_char, function(m) { return m.toUpperCase(); });
}

const handleSubmitTemplatePrompt = (template: string) => {
  message.value = template;
  sendMessage();
};

watch(
  messages,
  () => {
    //scroll to the bottom of the messages container
    nextTick(() => {
      if (messagesContainer.value) {
        messagesContainer.value.scrollTop =
          messagesContainer.value.scrollHeight;
      }
    });
  },
  { deep: true },
);
</script>

<style>
#messages-container {
  max-height: 80vh;
  overflow-y: auto;
}

@keyframes grow {
  0% {
    font-size: 1em;
  }
  50% {
    font-size: 1.5em;
  }
  100% {
    font-size: 1em;
  }
}


.animated {
  animation: grow 1s ease-in infinite;
}
</style>
