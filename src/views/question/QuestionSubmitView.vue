<template>
  <div id="questionSubmitView">
    <a-form :model="searchParams" layout="inline">
      <a-form-item field="questionId" label="题目" style="min-width: 240px">
        <a-input v-model="searchParams.questionTitle" placeholder="请输入" />
      </a-form-item>
      <a-form-item field="language" label="编程语言" style="min-width: 240px">
        <a-select
          v-model="searchParams.language"
          :style="{ width: '320px' }"
          placeholder="选择编程语言"
        >
          <a-option>Java</a-option>
          <a-option>C++</a-option>
        </a-select>
      </a-form-item>
      <a-form-item>
        <a-button type="primary" @click="doSubmit">搜索</a-button>
      </a-form-item>
    </a-form>
    <a-divider size="0" />
    <a-table
      :ref="tableRef"
      :columns="columns"
      :data="dataList"
      :pagination="{
        showTotal: true,
        pageSize: searchParams.pageSize,
        current: searchParams.current,
        total,
      }"
      @page-change="onPageChange"
    >
      <template #createTime="{ record }">
        {{ moment(record.createTime).format("YYYY-MM-DD") }}
      </template>
      <template #status="{ record }">
        {{ statusMap[record.status] }}
      </template>
    </a-table>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, onUnmounted, ref, watchEffect } from "vue";
import {
  Question,
  QuestionControllerService,
  QuestionSubmitQueryRequest,
} from "../../../generated";
import message from "@arco-design/web-vue/es/message";
import { useRouter } from "vue-router";
import moment from "moment";

const tableRef = ref();

const dataList = ref([]);
const total = ref(0);

const statusMap = {
  0: "等待中",
  1: "判题中",
  2: "已判题",
  3: "判题失败",
};

const searchParams = ref<QuestionSubmitQueryRequest>({
  questionTitle: "",
  language: undefined,
  pageSize: 8,
  current: 1,
});

let pollingTimer: any = null;

const loadData = async () => {
  const res =
    await QuestionControllerService.listQuestionSubmitVoByPageUsingPost({
      ...searchParams.value,
      sortField: "create_time",
      sortOrder: "descend",
    });
  if (res.code === 0) {
    console.log(dataList.value);
    dataList.value = res.data.records;
    total.value = res.data.total;
  } else {
    message.error("加载失败，" + res.message);
  }
};

const startPolling = () => {
  if (pollingTimer) {
    // 如果已有计时器，先清除
    clearInterval(pollingTimer);
  }

  // 每3秒轮询一次
  pollingTimer = setInterval(async () => {
    await loadData();
  }, 3000);
};

const stopPolling = () => {
  if (pollingTimer) {
    clearInterval(pollingTimer);
    pollingTimer = null;
  }
};

/**
 * 监听 searchParams 变量，改变时触发页面的重新加载
 */
watchEffect(() => {
  loadData();
});

/**
 * 页面加载时，请求数据
 */
onMounted(() => {
  loadData();
  startPolling();
});

onBeforeUnmount(() => {
  stopPolling();
});

onUnmounted(() => {
  stopPolling();
});

const columns = [
  {
    title: "题目",
    dataIndex: "questionTitle",
  },
  {
    title: "编程语言",
    dataIndex: "language",
  },
  {
    title: "判题信息",
    dataIndex: "judgeInfo.message",
  },
  {
    title: "执行用时（ms）",
    dataIndex: "judgeInfo.time",
  },
  {
    title: "消耗内存（KB）",
    dataIndex: "judgeInfo.memory",
  },
  {
    title: "判题状态",
    slotName: "status",
  },
  {
    title: "提交者",
    dataIndex: "userName",
  },
  {
    title: "创建时间",
    slotName: "createTime",
  },
];

const onPageChange = (page: number) => {
  searchParams.value = {
    ...searchParams.value,
    current: page,
  };
};

const router = useRouter();

/**
 * 跳转到做题页面
 * @param question
 */
const toQuestionPage = (question: Question) => {
  router.push({
    path: `/view/question/${question.id}`,
  });
};

/**
 * 确认搜索，重新加载数据
 */
const doSubmit = () => {
  // 这里需要重置搜索页号
  searchParams.value = {
    ...searchParams.value,
    current: 1,
  };
};
</script>

<style scoped>
#questionSubmitView {
  max-width: 1280px;
  margin: 0 auto;
}
</style>
