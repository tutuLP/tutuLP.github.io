模板

```vue
<template>
  <section class="c-Component">
    <header class="c-Component__header">
      <slot name="header">默认标题</slot>
    </header>
    <main class="c-Component__body">
      <slot />
    </main>
    <footer class="c-Component__footer">
      <slot name="footer">默认底部</slot>
    </footer>
  </section>
</template>

<script setup lang="ts">
// 组件选项
// defineOptions 可用于命名组件或配置
// 推荐在大型项目中为组件设置 name，方便调试和 keep-alive

defineOptions({
  name: 'ComponentTemplate'
})

// Props 示例-子组件声明自己接受的参数
const props = defineProps<{
  title?: string
}>()

// Emits 示例-向父组件发送可能触发的事件
defineEmits<{ (e: 'update'): void }>()

// 暴露方法示例
defineExpose({})
</script>

<style lang="scss" scoped>
.c-Component {
  display: flex;
  flex-direction: column;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 12px;

  &__header {
    font-weight: bold;
    margin-bottom: 8px;
  }
}
</style>

```

