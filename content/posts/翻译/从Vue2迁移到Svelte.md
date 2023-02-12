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

<pre>
    <code class="language-{lang}">
        {@html Prism.highlight(code, Prism.languages[lang], lang)}
    </code>
</pre>
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

+ Svelte也有它自己的缺点

当然，Svelte也有它自己的短板。比如一个**相对比较小的社区**，不过这是因为它仅发布与2019年。但随着越来越多的开发人员接受它的优点以及用户友好的内容，这也将支持Svelte社区未来的发展壮大。Github上目前已经有定期的，解释友好的更新，可以很容易的访问。

因此，在审查了这个分析的结果后，我们决定迁移到Svelte和Svelte Kit， 即时SvelteKit在我们迁移的过程中仍然处于活跃开发期。(看下图：)

![Escape Benchmark about frontend stack](/img/Capture-d-e-cran-2022-11-18-a--16.29.14.png)

{{<imgda center "Escape Benchmark about frontend stack">}}

**我们选择了那种方法来处理迁移？**

**什么时候处理的迁移：** 我们处理迁移在8月份的时候，那个时候很少人使用APP。

**时间跨度：** 花了两周时间迁移从Vue迁移所有文件到Svelte

**多少开发人员：** 两个前端开发人员花费整整两周开发时间，另外一个开发人员花了一整周的时间，因此一共是3个开发人员参与了这个迁移。

**工作流程：** 首先，我们使用[Notion](https://www.notion.so/)为我们团队的开发人员开通了访问权限。接着，我们在[Storybook]创建了一个新的任务，最后，每个开发人员得到了一些需要使用Svelte重写的页面。

**作为创业公司**，它是容易重写的，因为我们没有数千个文件需要去重写，因此迁移过程表现的非常迅速。然而，它仍然是存在一些风险在我们开始去迁移的时候，因为当时的SvelteKit仍是处于一个活跃开发状态。这导致我们在仅仅迁移一个月之后就不得不做一个大的中断变更。但是，SvelteKit官方和他们知识渊博的团队提供给我们一个命令（npx svelte-migrate routes）和一个解释非常清楚的[迁移指南](https://github.com/sveltejs/kit/discussions/5774),它真的帮助我们迅速的适配了新的更新。

此外，在9月SvelteKit团队宣布改框架终于进入了[候选发布阶段](https://dev.to/mandrasch/svelte-summit-fall-2022-sveltekit-hits-rc-phase-52p6)，这意味着它的稳定性得到了保证！

**文件&组件的组织方式**

SvelteKit的“基于文件见的路由”给我们带来了许多。我们可以将页面拆分成子页面，以便可以重用标准变量的名称，例如：“loading”，“submit”。etc...可能被拒绝。此外，布局直接集成到关联的路由，因为在树内，即使文件组织增加它也是容易去访问的。

**我们得到了什么？**

除了上述好处外，还有一些其他关键因素值得探讨：

**得到提升的梗流程的性能**。一旦编译完成，我们就可以体会到应用的轻量级。这提升了相比其他框架的加载速度。这是因为其他框架在应用逻辑代码周边潜入了运行时。

**更好的开发者体验**。SvelteKit使用Vite模块，Vite是新一代的javascript构建工具，它利用浏览器中ES模块的可用性和编译绑定带来了最新的javascript技术带来的最佳开发者体验。

**更快的代码执行**。Svelte没有虚拟DOM，在进行页面更改时执行的层级少了一层。

**启动并运行服务端渲染（SSR）**。Svelte使代码更加可读和容易维护，面向组件编码可以被组织在同一个文件中：逻辑（javascript）,结构（HTML），样式（CSS）。特殊之处时所有的元素都编译在同一个.svelte文件中。

**固定的类型检查**。自从我们迁移到Svelte，我们已经设法通过类型检查解决了最初的问题。事实上，我们以前必须处理这种经常性通知，而今天不再出来了。**不再出现烦人的哨兵错误**。（见下图）

![Example of Sentry errors bot notifications on our Discord](/img/sentry-errors-2.jpg)

{{<imgda center "Example of Sentry errors bot notifications on our Discord">}}

总而言之，上述好处和收获是我们的开发体验更加愉悦，因此我们能够集中精力在我们的[Escape平台](https://app.escape.tech/login/?to=%2F)上发布更好和更快的功能。对于终端用户而言这是一个很大的优势。