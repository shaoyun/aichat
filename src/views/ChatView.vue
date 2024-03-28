<script setup>
import { ref, onMounted, watch } from 'vue'
import Dexie from 'dexie';
import Markdown from 'vue3-markdown-it';
import 'highlight.js/styles/a11y-dark.css';
import { ElMessage, ElMessageBox } from 'element-plus'

const selectedModel = ref();
const modelList = ref([
  { name: 'gpt-3.5-turbo', code: 'gpt-3.5-turbo' },
  { name: 'gpt-4', code: 'gpt-4' },
  { name: 'claude-3-haiku-20240307', code: 'claude-3-haiku-20240307' },
]);

const db = new Dexie('aichat');
db.version(1).stores({
  conversations: '++id, title, model',
  messages: '++id, conversation_id, role, content',
});
// state
const messages = ref([])

const inputContent = ref('')

const scrollableDiv = ref(null)

const conversations = ref([])
const currentChatId = ref(1)

const dbMessages = []

watch(selectedModel, async (newValue, oldValue) => {
  let conv = conversations.value.find(item => item.id === currentChatId.value)
  if (conv !== undefined) {
    conv.model = newValue
    await db.conversations.update(conv.id, {
      model: newValue
    })
  }
});

async function getConversationMessage(id) {
  return await db.messages.where("conversation_id").equals(id).toArray();
}

async function changeConversation(id) {
  let conv = conversations.value.find(item => item.id === id)
  messages.value = await getConversationMessage(id)
  currentChatId.value = id
  selectedModel.value = conv.model
}
onMounted(async () => {
  await reloadData()
})

async function reloadData() {
  conversations.value = await db.conversations.orderBy('id').reverse().toArray();
  if (conversations.value.length > 0) {
    changeConversation(conversations.value[0].id)
  }
}
// actions
async function sendContent() {
  const question = inputContent.value
  let conv = conversations.value.find(item => item.id === currentChatId.value)
  const msg = {
    "conversation_id": conv.id,
    "role": "user",
    "content": question
  }
  messages.value.push(msg);
  dbMessages.push(msg)
  await saveMsg2Db(msg)

  inputContent.value = ''
  scrollToBottom()
  setTimeout(async () => {
    await askLLM(conv.id, question)
  }, 1000);
}

async function saveMsg2Db(msg) {
  await db.messages.add(msg);
}


import OpenAI from 'openai';
import { OpenAIStream, StreamingTextResponse } from 'ai';


async function askLLM(conv_id, question) {
  const msg = {
    "conversation_id": conv_id,
    "role": "assistant",
    "content": ''
  }
  messages.value.push(msg)

  let resp = await callOpenAI(messages.value, selectedModel.value)

  const reader = resp.body.getReader();

  let result = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) {
      break;
    }
    const chunk = new TextDecoder().decode(value);
    result += chunk;
  }

  console.log(result);
  msg.content = result

  dbMessages.push(msg)
  await saveMsg2Db(msg)
  setTimeout(() => {
    scrollToBottom()
  }, 1000);
}


async function callOpenAI(history_messages, model) {
  const openai = new OpenAI({
    apiKey: '<your openai key>',
    baseURL: 'https://api.kksj.org/v1',
    dangerouslyAllowBrowser: true
  });

  const formatMessages = history_messages.map((m) => ({
    content: m.content,
    role: m.role,
  }));

  console.log('model name: ' + model)

  const response = await openai.chat.completions.create({
    model: model,
    messages: formatMessages,
    stream: true,
  });
  // return response

  let content = ''

  const stream = OpenAIStream(response, {
    onStart: async () => {
      // 流开始时调用此回调
      // 可以使用它来将提示保存到数据库中
      // await savePromptToDatabase(prompt)
      // console.log('start');
      content = '';
    },
    onToken: async (token) => {
      // 为流中的每个标记调用此回调
      // 可以使用它来调试流或将 token 保存到数据库中
      // console.log('onToken');
      // console.log('token:', token)
      content += token
      // console.log(messages.value)
      messages.value[messages.value.length - 1].content = content;
    },
    // onText: async (token) => {
    //   // 为流中的每个标记调用此回调
    //   // 可以使用它来调试流或将 token 保存到数据库中
    //   console.log('onText');
    //   console.log(token)
    // },
    onCompletion: async (completion) => {
      // 流完成时调用此回调
      // 可以使用它来将最终的完成保存到数据库中
      // await saveCompletionToDatabase(completion)
      console.log('onCompletion');
      content = '';
    },
    // onFinal: async (completion) => {
    //   // 流完成时调用此回调
    //   // 可以使用它来将最终的完成保存到数据库中
    //   // await saveCompletionToDatabase(completion)
    //   console.log('onFinal');
    // }
  });
  return new StreamingTextResponse(stream);
}

function scrollToBottom() {
  const div = scrollableDiv.value
  div.scrollTop = div.scrollHeight - div.clientHeight;
}

function handleKeydown(event) {
  // 检查是否同时按下了Command/Ctrl键和回车键
  if (event.metaKey && event.key === 'Enter') {
    // 在这里执行你想要的操作
    console.log('Command+Enter 或 Ctrl+Enter 被按下');
    // 例如，触发一个方法
    sendContent();
  }
}
async function newChat() {
  const conv = {
    title: "chat 1",
    model: "gpt-3.5-turbo"
  }
  conversations.value.push(conv)
  const id = await db.conversations.add(conv);
  changeConversation(id)
}
async function deleteConv(conv) {
  await db.messages.where("conversation_id").equals(conv.id)
  await db.conversations.where("id").equals(conv.id).delete()
  await reloadData()
}
const onDeleteConv = (conv) => {
  ElMessageBox.confirm(
    '你确认要删除会话？',
    'Warning',
    {
      confirmButtonText: '确认',
      cancelButtonText: '取消',
      type: 'warning',
    }
  )
    .then(() => {
      deleteConv(conv)
      ElMessage({
        type: 'success',
        message: '删除成功',
      })
    })
    .catch(() => {
      // ElMessage({
      //   type: 'info',
      //   message: 'Delete canceled',
      // })
    })
}
async function regenerate(message) {
  ElMessage({
    message: '功能待实现，敬请期待！',
    grouping: true,
    type: 'success',
  })
}
const showEditTitleBox = (conv) => {
  ElMessageBox.prompt('请输入会话标题', '提示', {
    confirmButtonText: '确认',
    cancelButtonText: '取消',
    inputPattern: /^\S.*\S$/,
    inputErrorMessage: '不能为空',
  })
    .then(({ value }) => {
      if (value === null || value.length === 0) {
        return ElMessage({
          type: 'info',
          message: '标题不能为空',
        })
      }
    })
    .catch(() => {
      // ElMessage({
      //   type: 'info',
      //   message: 'Input canceled',
      // })
    })
}
</script>

<template>
  <div class="container">
    <div class="side">
      <div class="conversation-item" @click="newChat">
        <div class="conv-icon"><img src="@/assets/openai.svg" class="logo vue" alt="Vue logo" /></div>
        <div style="font-weight: bold; flex: 1;">创建新对话</div>
        <div style="margin-top: 4px;"><img style="width: 18px; height: 18px;" src="@/assets/edit.svg" class="logo vue"
            alt="Vue logo" /></div>
      </div>
      <div v-for="item in conversations" :class="'conversation-item' + (currentChatId === item.id ? ' active' : '')"
        v-on:click="changeConversation(item.id)">
        <div class="conv-icon"><img src="@/assets/openai.svg" class="logo vue" alt="Vue logo" /></div>
        <div style="font-weight: bold; flex: 1;">{{ item.title }}</div>
        <div v-if="currentChatId === item.id" style="line-height: 24px; height: 24px; margin-top: 4px; width: 24px; margin-right: 3px;" @click="showEditTitleBox(item)">
          <svg width="18" height="18" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1024 1024" data-v-ea893728=""><path fill="currentColor" d="m199.04 672.64 193.984 112 224-387.968-193.92-112-224 388.032zm-23.872 60.16 32.896 148.288 144.896-45.696zM455.04 229.248l193.92 112 56.704-98.112-193.984-112-56.64 98.112zM104.32 708.8l384-665.024 304.768 175.936L409.152 884.8h.064l-248.448 78.336zm384 254.272v-64h448v64h-448z"></path></svg>
        </div>
        <div v-if="currentChatId === item.id" style="line-height: 24px; height: 24px; margin-top: 4px; width: 24px;" @click="onDeleteConv(item)">
          <svg width="18" height="18" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1024 1024" data-v-ea893728=""><path fill="currentColor" d="M160 256H96a32 32 0 0 1 0-64h256V95.936a32 32 0 0 1 32-32h256a32 32 0 0 1 32 32V192h256a32 32 0 1 1 0 64h-64v672a32 32 0 0 1-32 32H192a32 32 0 0 1-32-32zm448-64v-64H416v64zM224 896h576V256H224zm192-128a32 32 0 0 1-32-32V416a32 32 0 0 1 64 0v320a32 32 0 0 1-32 32m192 0a32 32 0 0 1-32-32V416a32 32 0 0 1 64 0v320a32 32 0 0 1-32 32"></path></svg>
        </div>
      </div>
    </div>
    <div class="main">
      <div class="toolbar">
        <div>
          <el-select
            v-model="selectedModel"
            placeholder="Select a Model"
            style="width: 220px"
          >
            <el-option
              v-for="item in modelList"
              :key="item.code"
              :label="item.name"
              :value="item.code"
            />
          </el-select>
        </div>

      </div>
      <div class="message-box" ref="scrollableDiv">
        <div v-for="(item, index) in messages" :class="'message-item role-' + item.role">
            <div style="width: 42px;">
              <div class="role-avatar">
                <svg width="16" height="16" viewBox="0 0 41 41" fill="none" xmlns="http://www.w3.org/2000/svg" stroke-width="1.5" class=""><path d="M37.5324 16.8707C37.9808 15.5241 38.1363 14.0974 37.9886 12.6859C37.8409 11.2744 37.3934 9.91076 36.676 8.68622C35.6126 6.83404 33.9882 5.3676 32.0373 4.4985C30.0864 3.62941 27.9098 3.40259 25.8215 3.85078C24.8796 2.7893 23.7219 1.94125 22.4257 1.36341C21.1295 0.785575 19.7249 0.491269 18.3058 0.500197C16.1708 0.495044 14.0893 1.16803 12.3614 2.42214C10.6335 3.67624 9.34853 5.44666 8.6917 7.47815C7.30085 7.76286 5.98686 8.3414 4.8377 9.17505C3.68854 10.0087 2.73073 11.0782 2.02839 12.312C0.956464 14.1591 0.498905 16.2988 0.721698 18.4228C0.944492 20.5467 1.83612 22.5449 3.268 24.1293C2.81966 25.4759 2.66413 26.9026 2.81182 28.3141C2.95951 29.7256 3.40701 31.0892 4.12437 32.3138C5.18791 34.1659 6.8123 35.6322 8.76321 36.5013C10.7141 37.3704 12.8907 37.5973 14.9789 37.1492C15.9208 38.2107 17.0786 39.0587 18.3747 39.6366C19.6709 40.2144 21.0755 40.5087 22.4946 40.4998C24.6307 40.5054 26.7133 39.8321 28.4418 38.5772C30.1704 37.3223 31.4556 35.5506 32.1119 33.5179C33.5027 33.2332 34.8167 32.6547 35.9659 31.821C37.115 30.9874 38.0728 29.9178 38.7752 28.684C39.8458 26.8371 40.3023 24.6979 40.0789 22.5748C39.8556 20.4517 38.9639 18.4544 37.5324 16.8707ZM22.4978 37.8849C20.7443 37.8874 19.0459 37.2733 17.6994 36.1501C17.7601 36.117 17.8666 36.0586 17.936 36.0161L25.9004 31.4156C26.1003 31.3019 26.2663 31.137 26.3813 30.9378C26.4964 30.7386 26.5563 30.5124 26.5549 30.2825V19.0542L29.9213 20.998C29.9389 21.0068 29.9541 21.0198 29.9656 21.0359C29.977 21.052 29.9842 21.0707 29.9867 21.0902V30.3889C29.9842 32.375 29.1946 34.2791 27.7909 35.6841C26.3872 37.0892 24.4838 37.8806 22.4978 37.8849ZM6.39227 31.0064C5.51397 29.4888 5.19742 27.7107 5.49804 25.9832C5.55718 26.0187 5.66048 26.0818 5.73461 26.1244L13.699 30.7248C13.8975 30.8408 14.1233 30.902 14.3532 30.902C14.583 30.902 14.8088 30.8408 15.0073 30.7248L24.731 25.1103V28.9979C24.7321 29.0177 24.7283 29.0376 24.7199 29.0556C24.7115 29.0736 24.6988 29.0893 24.6829 29.1012L16.6317 33.7497C14.9096 34.7416 12.8643 35.0097 10.9447 34.4954C9.02506 33.9811 7.38785 32.7263 6.39227 31.0064ZM4.29707 13.6194C5.17156 12.0998 6.55279 10.9364 8.19885 10.3327C8.19885 10.4013 8.19491 10.5228 8.19491 10.6071V19.808C8.19351 20.0378 8.25334 20.2638 8.36823 20.4629C8.48312 20.6619 8.64893 20.8267 8.84863 20.9404L18.5723 26.5542L15.206 28.4979C15.1894 28.5089 15.1703 28.5155 15.1505 28.5173C15.1307 28.5191 15.1107 28.516 15.0924 28.5082L7.04046 23.8557C5.32135 22.8601 4.06716 21.2235 3.55289 19.3046C3.03862 17.3858 3.30624 15.3413 4.29707 13.6194ZM31.955 20.0556L22.2312 14.4411L25.5976 12.4981C25.6142 12.4872 25.6333 12.4805 25.6531 12.4787C25.6729 12.4769 25.6928 12.4801 25.7111 12.4879L33.7631 17.1364C34.9967 17.849 36.0017 18.8982 36.6606 20.1613C37.3194 21.4244 37.6047 22.849 37.4832 24.2684C37.3617 25.6878 36.8382 27.0432 35.9743 28.1759C35.1103 29.3086 33.9415 30.1717 32.6047 30.6641C32.6047 30.5947 32.6047 30.4733 32.6047 30.3889V21.188C32.6066 20.9586 32.5474 20.7328 32.4332 20.5338C32.319 20.3348 32.154 20.1698 31.955 20.0556ZM35.3055 15.0128C35.2464 14.9765 35.1431 14.9142 35.069 14.8717L27.1045 10.2712C26.906 10.1554 26.6803 10.0943 26.4504 10.0943C26.2206 10.0943 25.9948 10.1554 25.7963 10.2712L16.0726 15.8858V11.9982C16.0715 11.9783 16.0753 11.9585 16.0837 11.9405C16.0921 11.9225 16.1048 11.9068 16.1207 11.8949L24.1719 7.25025C25.4053 6.53903 26.8158 6.19376 28.2383 6.25482C29.6608 6.31589 31.0364 6.78077 32.2044 7.59508C33.3723 8.40939 34.2842 9.53945 34.8334 10.8531C35.3826 12.1667 35.5464 13.6095 35.3055 15.0128ZM14.2424 21.9419L10.8752 19.9981C10.8576 19.9893 10.8423 19.9763 10.8309 19.9602C10.8195 19.9441 10.8122 19.9254 10.8098 19.9058V10.6071C10.8107 9.18295 11.2173 7.78848 11.9819 6.58696C12.7466 5.38544 13.8377 4.42659 15.1275 3.82264C16.4173 3.21869 17.8524 2.99464 19.2649 3.1767C20.6775 3.35876 22.0089 3.93941 23.1034 4.85067C23.0427 4.88379 22.937 4.94215 22.8668 4.98473L14.9024 9.58517C14.7025 9.69878 14.5366 9.86356 14.4215 10.0626C14.3065 10.2616 14.2466 10.4877 14.2479 10.7175L14.2424 21.9419ZM16.071 17.9991L20.4018 15.4978L24.7325 17.9975V22.9985L20.4018 25.4983L16.071 22.9985V17.9991Z" fill="currentColor"></path></svg>
              </div>
            </div>
            <div style="flex: 1;">
              <div class="role" v-if="item.role==='user'">You</div>
              <div class="role" v-else>OpenAI</div>
              <div style="clear: both; height: 5px;"></div>
              <div class="role-content">
                <Markdown :source="item.content" />
                <!-- <div class="message-tools">
                  <span v-if="(messages.length > 1) && (index === messages.length - 1) && item.role === 'assistant'" style="cursor: pointer;" @click="regenerate(item)">
                    <svg width="20" height="20" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1024 1024" data-v-ea893728=""><path fill="currentColor" d="M289.088 296.704h92.992a32 32 0 0 1 0 64H232.96a32 32 0 0 1-32-32V179.712a32 32 0 0 1 64 0v50.56a384 384 0 0 1 643.84 282.88 384 384 0 0 1-383.936 384 384 384 0 0 1-384-384h64a320 320 0 1 0 640 0 320 320 0 0 0-555.712-216.448z"></path></svg>
                  </span>
                </div> -->
              </div>
            </div>

        </div>
      </div>
      <div class="bottom-input">
          <div class="form-area" style="position: relative;">
            <el-input
              v-model="inputContent"
              :autosize="{ minRows: 3, maxRows: 4 }"
              type="textarea"
              placeholder="发送消息给 OpenAI..."
              @keydown="handleKeydown"
            />
            <div style="position: absolute; top:30%; right: 10px;"><el-button v-on:click="sendContent" :disabled="inputContent.length === 0">发送</el-button></div>
          </div>
          <!-- end bottom -->
      </div>
<!--  end of main    -->
    </div>
  </div>
</template>

<style scoped lang="less">
@main-font-color: rgb(55, 65, 81);
.container {
  display: flex;
  height: 100%;
  .side {
    width: 260px;
    background-color: #F7F7F8;
    padding: .75rem;
    .conversation-item {
      line-height: 30px;
      width: 100%;
      margin-bottom: 5px;
      padding: 5px 10px;
      cursor: pointer;
      border-radius: 6px;
      display: flex;
      &:hover {
        background-color: #E3E3E3;
      }

      &.active {
        background-color: #CDCDCD;
      }

      display: flex;
    }
    .conv-icon {
      width: 24px;
      margin-top: 3px;
    }
  }
  .main {
    flex: 1;
    display: flex;
    flex-direction: column;
    height: 100vh;

    .toolbar {
      height: 60px;
      padding: 10px;
    }
    .message-box {
      flex: 1;
      overflow-y: scroll;
      max-height: 100%;
      .message-item {
        padding: 5px 10px;
        max-width:47rem;
        margin: 0 auto 10px auto;
        color: #222222;
        font-family: Inter, sans-serif;
        font-size: 16px;
        line-height: 28px;
        display: flex;
        &.role-assistant {
          text-align: center;
          line-height: 24px;
        }
        &.role-user {
          text-align: center;
          line-height: 24px;
        }
        .role {
          float: left;
          line-height: 28px;
          font-weight: bold;
          color: #000;
        }
        .role-avatar {
          width: 28px;
          height: 28px;
          border-radius: 50%;
          background-color: rgb(25, 195, 125);
          border: 1px solid #eee;
          float: left;
          margin-right: 10px;
          color: #fff;
          padding-top: 3px;
        }
        .role-content {
          text-align: left;
          line-height: 28px;
        }
        .message-tools {
          margin-top: 3px;
        }
      }
    }
    .bottom-input {
      height: 130px;
      padding: 15px;
      align-items: center;
      .form-area {
        max-width:48rem;
        margin: 0 auto;
      }
    }
  }

}
</style>
