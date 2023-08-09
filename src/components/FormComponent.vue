<script lang="ts">
import {ref} from 'vue';

const API_URL = 'https://jnx038ry5j.execute-api.us-east-1.amazonaws.com';

export interface FrqOutput {
  context: string
  question: string
  rubric: string
}

export interface EvaluationOutput {
  feedback: string
  scores: number[]
}

export interface MetricsOutput {
  totalQuestions: number,
  totalIterations: number,
  totalError: number,
  ftr: number
  errorRate: number,
}

const TIMEOUT = 10000;

export default {
  name: 'FormComponent',
  setup() {
    const userId = ref('');
    const topic = ref('');
    const topics = ref('');
    const response = ref('');
    const resultGenerate = ref(undefined as FrqOutput | undefined);
    const resultEvaluate = ref(undefined as EvaluationOutput | undefined);
    const resultMetrics = ref(undefined as MetricsOutput | undefined);
    const resultDebug = ref('');
    const isLoadingGenerate = ref(false);
    const isLoadingEvaluate = ref(false);
    const frqId = ref('');
    const evaluationId = ref('');

    const handleSubmitGenerate = async () => {
      const formData = {userId: userId.value, topic: topic.value};

      // Sending form data to the API endpoint
      const response = await fetch(API_URL + '/frq', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(formData),
      });
      const frq = await response.json() as { id: string };
      frqId.value = frq.id;
      isLoadingGenerate.value = true;
      checkForResultGenerate();
      // Pulling another endpoint for the result
    };
    const handleSubmitGenerateMulti = async () => {
      const topicList = topics.value.split(',');
      for(const topicItem of topicList){
      const formData = {userId: userId.value, topic: topicItem};
      await fetch(API_URL + '/frq', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(formData),
      });}
    };

    const checkForResultGenerate = async () => {
      const res = await fetch(API_URL + '/frq/' + frqId.value);
      const data = await res.json();


      if (data.content.status === 'Pending') {
        setTimeout(() => {
          checkForResultGenerate();
        }, TIMEOUT);
        isLoadingGenerate.value = true;
      } else {
        isLoadingGenerate.value = false;
        resultGenerate.value = data.content;
        const resDebug = await fetch(API_URL + '/frq/' + frqId.value + '/debug');
        const dataDebug = await resDebug.json();
        resultDebug.value = JSON.stringify(dataDebug, null, 2);
      }
    };

    const handleSubmitEvaluate = async () => {
      const formData = {response: response.value};

      // Sending form data to the API endpoint
      const apiResponse = await fetch(API_URL + '/evaluate/' + frqId.value + '/' + userId.value, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(formData),
      });
      const evaluation = await apiResponse.json() as { id: string };
      evaluationId.value = evaluation.id;
      isLoadingEvaluate.value = true;
      checkForResultEvaluate();
    };

    const checkForResultEvaluate = async () => {
      const res = await fetch(API_URL + '/evaluate/' + frqId.value + '/' + userId.value);
      const data = (await res.json())[0];


      if (data.content.status === 'Pending') {
        setTimeout(() => {
          checkForResultEvaluate();
        }, TIMEOUT);
        isLoadingEvaluate.value = true;
      } else {
        isLoadingEvaluate.value = false;
        resultEvaluate.value = data.content;
      }
    };

    const handleSubmitMetrics = async () => {
      const res = await fetch(API_URL + '/metrics');
      resultMetrics.value = (await res.json()) as MetricsOutput;
    };
    return {
      userId,
      topic,
      topics,
      response,
      resultGenerate,
      resultDebug,
      resultEvaluate,
      resultMetrics,
      frqId,
      isLoadingGenerate,
      isLoadingEvaluate,
      handleSubmitGenerate,
      handleSubmitGenerateMulti,
      handleSubmitEvaluate,
      handleSubmitMetrics
    };
  },
};
</script>
<template>
  <div>
    <h1>Generate FRQ</h1>
    <form @submit.prevent="handleSubmitGenerate">
      User Name <input v-model="userId" placeholder="JohnDoe"/><br>
      Topic <input v-model="topic" placeholder="baseball"/><br>
      <button type="submit">Submit</button>
    </form>

    <div v-if="frqId">QueryId: {{ frqId }}</div>
    <div v-if="isLoadingGenerate">Pending...</div>
    <div v-if="resultGenerate?.context">
      <h2>Final query</h2>
      <div><h4>Context</h4> {{ resultGenerate.context }}</div>
      <div><h4>Question</h4> {{ resultGenerate.question }}</div>
      <div><h4>Rubric</h4> <span style="white-space: pre;">{{ resultGenerate.rubric }}</span></div>
    </div>
    <div v-if="resultDebug">
      <h2>Debug</h2>
      <pre>{{ resultDebug }}</pre>
    </div>
    <h1>Generate Multi FRQ</h1>
    <form @submit.prevent="handleSubmitGenerateMulti">
      User Name <input v-model="userId" placeholder="JohnDoe"/><br>
      Topics (comma separated) <input v-model="topics" placeholder="baseball"/><br>
      <button type="submit">Submit</button>
    </form>
    <h1>Evaluate FRQ response</h1>
    <form @submit.prevent="handleSubmitEvaluate">
      Query Id <input v-model="frqId" placeholder="JohnDoe"/><br>
      User Name <input v-model="userId" placeholder="JohnDoe"/><br>
      Response <textarea v-model="response" placeholder=""/><br>
      <button type="submit">Submit</button>
    </form>
    <div v-if="isLoadingEvaluate">Pending...</div>
    <div v-if="resultEvaluate?.feedback">
      <h2>Final query</h2>
      <div><h4>Feedback</h4> {{ resultEvaluate.feedback }}</div>
      <div><h4>Scores</h4> {{ resultEvaluate.scores }}</div>
    </div>
    <h1>Metrics</h1>
    <form @submit.prevent="handleSubmitMetrics">
      <button type="submit">Load Metrics</button>
    </form>
    <div v-if="resultMetrics">
      <div>Total Questions: {{ resultMetrics.totalQuestions }}</div>
      <div>Total Iterations: {{ resultMetrics.totalIterations }}</div>
      <div>Total Errors: {{ resultMetrics.totalError }}</div>
      <div>FTR: {{ resultMetrics.ftr }}%</div>
      <div>Error rate: {{ resultMetrics.errorRate }}%</div>
    </div>

  </div>
</template>
