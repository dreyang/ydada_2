<template>
  <div id="doAnswerPage">
    <a-card>
      <h1>{{ app.appName }}</h1>
      <p>{{ app.appDesc }}</p>
      <h2 sytle="margin-bottom: 16px">
        {{ current }}、{{ currentQuestion?.title }}
      </h2>
      <div>
        <a-radio-group
          v-model="currentAnswer"
          direction="vertical"
          :options="questionOptions"
          @change="onRadioChange"
          size="large"
        />
      </div>
      <div style="margin-top: 24px">
        <a-space size="large">
          <a-button
            type="primary"
            circle
            v-if="current < questionContent.length"
            :disabled="!currentAnswer"
            @click="current += 1"
          >
            下一题
          </a-button>
          <a-button
            type="primary"
            v-if="current === questionContent.length"
            :loading="submitting"
            circle
            :disabled="!currentAnswer"
            @click="doSubmit"
          >
            {{ submitting ? "分析中..." : "查看结果" }}
          </a-button>
          <a-button v-if="current > 1" circle @click="current -= 1">
            上一题
          </a-button>
        </a-space>
      </div>
    </a-card>
  </div>
</template>

<script setup lang="ts">
import {
  computed,
  defineProps,
  reactive,
  ref,
  watchEffect,
  withDefaults,
} from "vue";
import API from "@/api";
import message from "@arco-design/web-vue/es/message";
import { useRouter } from "vue-router";
import {
  addAppUsingPost,
  editAppUsingPost,
  getAppVoByIdUsingGet,
} from "@/api/appController";
import { APP_SCORING_STRATEGY_MAP, APP_TYPE_MAP } from "@/constant/app";
import {
  addQuestionUsingPost,
  editQuestionUsingPost,
  listQuestionVoByPageUsingPost,
} from "@/api/questionController";
import {
  addUserAnswerUsingPost,
  generateUserAnserIdUsingGet,
} from "@/api/userAnswerController";

interface Props {
  appId: string;
}
const props = withDefaults(defineProps<Props>(), {
  appId: () => {
    return "";
  },
});
const router = useRouter();

const onRadioChange = (value: string) => {
  currentAnswer.value = value;
  //  记录回答
  answerList[current.value - 1] = value;
};

const app = ref<API.AppVO>({});
//题目内容结构（理解为题目列表）
const questionContent = ref<API.QuestionContentDTO[]>([]);

// 当前题目序号（从 1 开始）
const current = ref(1);
// 当前题目
const currentQuestion = ref<API.QuestionContentDTO>({});
// 当前题目选项
const questionOptions = computed(() => {
  return currentQuestion.value?.options
    ? currentQuestion.value.options.map((option) => ({
        label: `${option.key}. ${option.value}`,
        value: option.key,
      }))
    : [];
});

// 当前答案
const currentAnswer = ref<string>();
// 回答列表
const answerList = reactive<string[]>([]);
//唯一id
const id = ref<number>();

const generateId = async () => {
  const res = await generateUserAnserIdUsingGet();
  if (res.data.code === 0) {
    id.value = res.data.data as any;
  } else {
    message.error("获取唯一id失败，" + res.data.message);
  }
};

//进入页面时生成唯一id
watchEffect(() => {
  generateId();
});

//删除题目
const deleteQuestion = (index: number) => {
  questionContent.value.splice(index, 1);
};

// 加载数据;
const loadData = async () => {
  if (!props.appId) {
    return;
  }
  //获取app
  let res: any = await getAppVoByIdUsingGet({
    id: props.appId as any,
  });
  if (res.data.code === 0) {
    app.value = res.data.data as any;
  } else {
    message.error("获取应用失败，" + res.data.message);
  }

  //获取题目
  res = await listQuestionVoByPageUsingPost({
    // eslint-disable-line
    appId: props.appId as any,
    current: 1,
    pageSize: 1,
    // sortOrder: "descend",
  });

  if (res.data.code === 0 && res.data.data?.records) {
    questionContent.value = res.data.data?.records[0].questionContent;
  } else {
    message.error("获取题目失败，" + res.data.message);
  }
};
//钩子函数，当loadData里的值发生更改时，会重新执行loadData
watchEffect(() => {
  loadData();
});
//改变current题号时，更新当前题目和答案
watchEffect(() => {
  currentQuestion.value = questionContent.value[current.value - 1];
  currentAnswer.value = answerList[current.value - 1];
});
// 提交;
const doSubmit = async () => {
  if (!props.appId || !answerList) {
    return;
  }
  submitting.value = true;
  const res = await addUserAnswerUsingPost({
    appId: props.appId as any,
    choices: answerList,
    id: id.value as any,
  });
  if (res.data.code === 0 && res.data.data) {
    router.push(`/answer/result/${res.data.data}`);
  } else {
    message.error("提交答案失败" + res.data.message);
  }
  submitting.value = false;
};
// 是否正在提交
const submitting = ref(false);
</script>
