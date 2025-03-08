<script setup lang="ts">
import {
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form'
import { Input } from '@/components/ui/input'

import { toTypedSchema } from '@vee-validate/zod'
import { useForm } from 'vee-validate'
import * as z from 'zod'
import { ref } from 'vue'
import { useFetch } from '#app' // Nuxt3 内置的 useFetch

// 定义表单校验规则
const formSchema = toTypedSchema(z.object({
  word: z.string().min(1, "请输入单词"),
}))

// 初始化表单
const { handleSubmit } = useForm({
  validationSchema: formSchema,
})

// 存储 API 返回的单词数据
const queryResult = ref<{ word: string; definition: string } | null>(null)
const loading = ref(false)
const errorMessage = ref<string | null>(null)

// 处理提交事件
const onSubmit = handleSubmit(async (values) => {
  loading.value = true
  errorMessage.value = null
  queryResult.value = null

  try {
    const url = `http://zibo.curesky.site:8000/find/${encodeURIComponent(values.word)}`

    const { data, error } = await useFetch<{ word: string; definition: string }>(url, {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
      },
    })

    if (error.value) {
      errorMessage.value = "查询失败，请重试"
    } else if (!data.value || !data.value.definition) {
      errorMessage.value = "未找到该单词的解释"
    } else {
      queryResult.value = data.value
    }
  } catch (error) {
    errorMessage.value = "网络错误，请检查连接"
  } finally {
    loading.value = false
  }
})

// 🔥 处理 definition 的格式化（新增排版优化）
const formatDefinition = (text: string) => {
  return text
    .replace(/\*\s/g, '<br>📌 ') // 关键短语前加图标
    .replace(/\b(\w+):/g, '<br><strong>$1:</strong>') // 冒号后加粗
    .replace(/=>/g, '<br>➡ ') // "=>" 符号换行并加符号
    .replace(/\*/g, '<strong>*</strong>') // 星号加粗
    .replace(/\/\s*([\w\W]+?)\s*\//g, '<span class="text-sm text-gray-500">/$1/</span>') // 处理音标
    .replace(/\((fig 比喻)\)/g, '<span class="text-sm text-gray-500">($1)</span>') // 处理 (fig 比喻)
    .replace(/\((idm 习语)\)/g, '<br><strong class="text-blue-600">$1</strong>') // 处理 (idm 习语)
    .replace(/\[([\w\s.,]+?)\]/g, '<span class="text-sm text-gray-500">[$1]</span>') // 处理 [Tn, Tn.pr] 结构
    .replace(/➡\s+kill\./g, '<br>➡ <strong>kill.</strong>') // 处理习语结尾
}
</script>

<template>
  <div class="flex flex-col items-center justify-center min-h-screen bg-gray-50 px-4 relative">
    <!-- 侧边作者声明 -->
    <div class="fixed right-4 bottom-4 bg-white p-3 rounded-lg shadow-md text-sm text-gray-600">
      此查询由
      <a href="https://blog.curesky.site" target="_blank" class="text-blue-500 hover:underline">cureSky</a>
      提供，仅供交流学习使用。
    </div>

    <!-- 表单容器 -->
    <div class="w-full max-w-lg bg-white p-6 rounded-lg shadow-lg">
      <h2 class="text-2xl font-bold text-gray-700 text-center mb-4">📖 牛津词典在线查询</h2>

      <form class="space-y-6" @submit.prevent="onSubmit">
        <!-- 输入框 -->
        <FormField v-slot="{ componentField }" name="word">
          <FormItem>
            <FormLabel class="text-gray-600">输入单词</FormLabel>
            <FormControl>
              <Input 
                type="text"
                placeholder="请输入要查询的单词"
                class="w-full p-3 border border-gray-300 rounded-lg focus:ring focus:ring-blue-200"
                v-bind="componentField"
              />
            </FormControl>
            <FormMessage class="text-red-500 text-sm mt-1" />
          </FormItem>
        </FormField>

        <button type="submit" 
          class="w-full bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 rounded-lg transition duration-300">
          🔍 查询
        </button>
      </form>
    </div>

    <!-- 结果显示 -->
    <div v-if="loading" class="mt-6 text-blue-500 text-lg font-semibold">查询中...</div>
    <div v-if="errorMessage" class="mt-6 text-red-500 text-lg font-semibold">{{ errorMessage }}</div>

    <div v-if="queryResult" class="mt-6 w-full max-w-lg bg-gray-100 p-6 rounded-lg shadow-md">
      <h3 class="text-xl font-bold text-gray-700">查询结果：</h3>
      <p class="text-lg font-semibold text-blue-600 mt-2">{{ queryResult.word }}</p>
      <p class="text-gray-800 mt-2 whitespace-pre-wrap leading-relaxed" v-html="formatDefinition(queryResult.definition)"></p>
    </div>
  </div>
</template>
