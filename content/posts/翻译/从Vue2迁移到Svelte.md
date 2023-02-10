---
title: "从Vue2迁移到Svelte"
date: 2023-02-10T16:07:06+08:00
draft: false
categories: 翻译
keywords: Vue Vue2 Svelte 迁移 前端 font-end 
---

本文翻译自[https://escape.tech/blog/from-vue2-to-svelte/](https://escape.tech/blog/from-vue2-to-svelte/)

在使用Vue2作为我们的前端框架差不多快两年后,它被宣布不再继续维护，因此我们决定迁移到一个新的框架。但是哪一个是我们应该选择的呢？Vue3 OR Svelte。

需要注意的是在这次迁移中我们也需要提升我们的开发体验，特别是在类型检查, 高性能以及构建时间方面。我们没有考虑React，因为我们没有太多的时间去学习。同时相对于Vue和Svelte，它也没有提供一个开箱即用的方案。此外，Vue和Svelte使用了相同的单文件组件概念：逻辑(javascript), 结构(html), 样式(CSS)在同一个文件中。

我们做了一些研究，最终选定了Svelte。下面一些解释关于为什么选择Svelte:

+  Svelte PK Vue3

    Svelte拥有更好的学习留存率。我们选择了市场上两个新的前端框架，Vue3和Svelte。下面是一个插图，显示的是不同框架在过去5年的留存率。从[State of JS survey](https://2021.stateofjs.com/en-US/libraries/front-end-frameworks/#front_end_frameworks_experience_ranking)收集的该领域的开发人员的数据，我们可以看到Svelte来到了第2的位置，而Vue3仅排名第4。

    ![In 2021 Svelte was in the 2nd position and Vue 3 in 4th position (source: State of JS)](/img/2021-stateofjs.jpg)

    {{<imgda center "In 2021 Svelte was in the 2nd position and Vue 3 in 4th position (source: State of JS)">}}

    这张图显示过去使用Svelte的开发人员在将来更愿意使用他们

    **更好的类型检查**

    ![类型检查](/img/20230210165416.png)

    Svelte 通过更简单的组件设计过程和内置类型化事件提供更好的类型检查体验，这对我们来说非常人性化。

    **严格的全局访问**

    在Svelte中可以从其他文件导入枚举，并在模板中使用它们，在Vue3中是不存在这种情形的。

    ![Escape Benchmark about frontend stack](/img/Capture-d-e-cran-2022-11-18-a--16.28.08.png)

    {{<imgda center "Escape Benchmark about frontend stack">}}

    **语法**

    主观的，我认为Svelte的语法相比于Vue更加的人性化和友好。你可以看看下面这些代码块，想下自己是什么感觉。

    Svelte

    ```svelte
    <script>
        let firstName = "";
        let town = "";
        $: fullName = "Full name: " + firstName + ' ' + lastName;
        const reset = () => {
            firstName = "";
            lastName = "";
        }
    </script>
    <main>
        <div>
            <label>First name</label>
            <input type="text" bind:value={firstName}>
            <label>Last name</label>
            <input type="text" bind:value={lastName}>
            <button on:click={reset}>Reset</button>
        </div>
        <div>
            {fullName}
        </div>
    </main>
    <style>
    main{
        background-color: white;
    }
    </style>
    ```

    Vue

    ```vue
    <template>
    <main>
        <label>First name </label>
        <input type="text" v-model="firstName"/>
        <label>Last name </label>
        <input type="text" v-model="lastName"/>
        <div>
            Full name: {{fullName}} 
        </div>
        <button @click="handleReset">Reset</button>
    </main>
    </template>
    <script setup>
    import { ref, computed } from 'vue'
    const firstName = ref('')
    const lastName = ref('')

    const fullName = computed(() => { return firstName.value + " " + lastName.value; })

    function handleReset() {
        firstName.value = ""
        lastName.value = ""
    }
    </script>
    <style scoped>
    main{
    background-color: white;
    }
    </style>
    ```

    没有额外的HTML标签<template>,在Svelte中你可以直接写自己的html。

    **样式自动在Svelte中限定范围**, 这是有利于维护的一个好点，这可以避免CSS的一些副作用。每个组件的样式被限制在本组件内生效，不会影响到它的父组件和子组件。

    **更新数据的时候不要求计算属性**。在Svelte中，它看起来更像是纯的javascript编程，你只需要关注于编写js函数就可以了。

    ```javascript
    const reset = () => {firstName = "";lastName = "";}
    ```

    仅需要单括号在Svelte中

    ```javascript
    //Svelte
    {fullName}

    //Vue
    {{fullName}}

    ```

    需要注意是这个分析仅针对上面指定的代码模板，这里不打算对不同框架的异同做详细解释。要查看更多信息，可以参考网站 [feel free to look online](https://svelte.dev/blog/write-less-code)

    
    **简单的继承purec插件**。 下面是一个例子使用Svelete和Pure集成一个语法高亮
    
    ```svelte
        // Prism.svelte
        <script>
            import Prism from 'prismjs';

            export let code;
            export let lang = 'javascript';
        </script>

        <pre><code class="language-{lang}">{@html
            Prism.highlight(code, Prism.languages[lang], lang)
        }</code></pre>
    ```

    ```svelte
        // App.svelte
        <script>
            import Prism from './Prism.svelte';
            let code = 'export const hello =\n\t(name) => {console.log(`Hello ${name}!`)};';
        </script>

        <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/prism/1.28.0/themes/prism-dark.min.css" integrity="sha512-Njdz7T/p6Ud1FiTMqH87bzDxaZBsVNebOWmacBjMdgWyeIhUSFU4V52oGwo3sT+ud+lyIE98sS291/zxBfozKw==" crossorigin="anonymous" referrerpolicy="no-referrer" />

        <Prism {code} />
    ```

    [Try on the REPL](https://svelte.dev/repl/eadd4295c0e4472fbfe599c581646ad6?version=3.53.1)

    **编译代码没有虚拟DOM**。Svelte和Vue最大的不同是减少了app与浏览器之间的层数，这将带来更多的性能优化以及更快的任务处理速度。
    <br/>

    **自动更新**。借助声明变量，Svelte可以自动更新你的数据，不需要等待更改反应在虚拟结构中，这将带来更好的用户体验。

    
