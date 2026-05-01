---
title: "从零开始接入 DN42 网络 - 4: 实现网络信息公网展示面板"
date: 2026-05-01
description: "写一个简单的 Web 面板来展示自己的 DN42 网络信息，然后部署。"
image: img/cover.png
tags:
    - 网络
    - 叠加网络
    - DN42
    - 工作流
    - 前端
categories:
    - 网络
---

<!-- markdownlint-disable MD029 -->

## 前言

整个初始的灵感来源是

- 一般很多大网 DN42 玩家或者自己喜欢玩儿的都会有一个公共的 Web 面板来展示自己的网络信息，包括 AS、地址段、节点信息、互联政策等
- MoonWX [纯 vibe 出来的面板](https://net.moonwx.net/) 还是很不错的，我基本就是希望拿 Vue 将其完整实现，添加几个他没有的功能，并稍做优化
- Kioubit 的 [面板](https://dn42.g-load.eu/) 也给了我很大启发，尤其是我希望能像他一样能够提供一个批量 Ping 的服务，从而让用户能快速判断连接哪个节点最优。
- 先做个大概吧，离 Kioubit 那种功能完善的面板还差远了
- 后续可能还会有 Looking Glass 之类的功能？再说吧

写完的面板在 [https://dn42.sisy.cc](https://dn42.sisy.cc)，开源在 [https://github.com/SisypheOvO/SISY-DN42-Website](https://github.com/SisypheOvO/SISY-DN42-Website)，欢迎访问。

## 技术选型

Vue3.5 (Setup) + Vite 8 + TypeScript 6.0 + TailwindCSS v4 + vue-i18n + Docker Compose + Nginx + Github Actions

起初的设计里是没有容器的，但是本地 dev 写完之后还是加上了，所以也多了个容器内 Nginx 配置。

一开始是真没想到，但推了一两个 commit，每次都要自己在服务器上拉下来执行部署操作，之后觉得实在是有点多余，就想到用 Github Actions 来实现 CI/CD，实现提交之后自动触发我自己服务器上的拉取、构建、起 Compose 的流程，从而实现自动部署。这个也算是自动化运维思维的小进步吧...

## 功能实现中的问题与解决

起手肯定还是小修小改，问题都不大，一些简单的数据层提取，项目架构分层，组件化，国际化等等，捣鼓捣鼓就行，无非是时间。

### vue-i18n 解析

```typescript
export default {
  policy: {
    title: "Peering Policy",
    intro: "We have an open peering policy. Please contact us to peer.",
    advertiseLabel: "We advertise the following prefixes:",
    disclaimer: "This network is experimental and educational, no SLA guaranteed. We will make our best efforts to maintain operation, but we are not responsible for any consequences caused by shutdown due to force majeure or personal financial reasons.",
    items: [
      "Only WireGuard tunnels are supported",
      "Connection may be terminated after long downtime",
      "MP-BGP + Extended Next Hop + IPv6 LLA is preferred",
      "Default port: 20000 + the last 4 digits of your ASN",
      "IPv4 / IPv6 single-stack endpoints are supported (use the 4. or 6. prefix)",
    ],
  },
}
```

在实现国际化的过程中，遇到了 vue-i18n 的一个问题。正常来说 Vue 模板部分里调用 vue-i18n 的 t 函数就行，例如以上这个 i18n 配置里的 `policy` 的对象，那么在模板里直接写

```html
<span>{{ $t("policy.title") }}</span>
```

即可直接拿到 title 的翻译内容。这是因为 `policy.title` 是一个字符串。但是如果你要用 `v-for` 来循环一个数组，例如

```html
<div v-for="item in $t('policy.items')" :key="item">
  <span>{{ $t(item) }}</span>
</div>
```

就会发现拿到的 `item` 的值是 undefined。这是因为 `policy.items` 是一个数组，t 函数默认只能解析字符串，无法直接解析数组。如果想用 t 函数来访问数组结构，需要通过索引来访问每一项：

```typescript
t('policy.items[0]')  // "Only WireGuard tunnels are supported"
t('policy.items[1]')  // "Connection may be terminated after long downtime"
```

所以在这个 v-for 的使用场景下就不能用 t 函数。如果想在模板里遍历渲染，常见的做法有两种：

1. 用 `tm()` 获取原始消息

用 `tm()` + `rt()` 来处理数组和复杂结构。vue-i18n v9+ 提供了 `tm()`（translation message），可以返回原始结构而非翻译后的字符串：

```html
<ul>
  <li v-for="(item, i) in tm('policy.items')" :key="i">
    {{ rt(item) }}
  </li>
</ul>
```

`tm()` 返回原始消息对象/数组，`rt()` 用来解析每一项为最终字符串。这也是官方推荐的方式。

2. 改为对象结构（避免数组）

如果不依赖 `tm/rt`，就得把数组改成带编号键的对象，然后在代码里手动组装：

```typescript
policy: {
  items: {
    0: "Only WireGuard tunnels are supported",
    1: "Connection may be terminated after long downtime",
    2: "MP-BGP + Extended Next Hop + IPv6 LLA is preferred",
    // ...
  }
}
```

那么出于优雅性考虑，obviously 应当选择第一种方式。

### ScrollSpy 实现

目标是判断当前视窗中活跃的是哪个 section，从而高亮对应的导航链接，并且自动把 url 里的 hash 更新为当前 section 的 id。

基本思路就是监控目前哪个 section 在视窗内且这个 section 的顶部的位置高于视口高度的 50%。

根据我搓完的经验，实现这个功能大致有两种方案，而造成有第二种方案的原因是如何处理底部的一个或几个 section：这些 section 可能很矮，在浏览器视口很高的情况下，它们的顶部位置可能永远都达不到视口高度的 50%，所以如果单纯地监控哪个 section 的顶部位置高于视口高度的 50%，就会导致最后一个或几个 section 永远无法被识别为活跃状态。

#### 方案一：使用 IntersectionObserver API

注意这个方法很难解决底部 section 的问题，可能可以结合 scroll 事件来做一些特殊处理？反正我没搞定。

首先写一个名为 `useScrollSpy` 的 composable 来实现这个功能，核心就是用 IntersectionObserver 来监控每个 section 的可见性和位置：

```typescript
import { onMounted, onBeforeUnmount, ref } from "vue"

export default function useScrollSpy(options?: { offset?: number; rootMargin?: string }) {
    const current = ref<string | null>(null)
    let observer: IntersectionObserver | null = null

    function init(ids: string[]) {
        if (typeof window === "undefined" || !window.IntersectionObserver) return

        const offset = options?.offset ?? 0
        const rootMargin = options?.rootMargin ?? `-50% 0px -50% 0px`

        const elements = ids.map((id) => document.getElementById(id)).filter(Boolean) as HTMLElement[]
        if (!elements.length) return

        const cb: IntersectionObserverCallback = (entries) => {
            // choose the entry with largest intersection ratio
            const visible = entries.filter((e) => e.isIntersecting).sort((a, b) => b.intersectionRatio - a.intersectionRatio)[0]

            if (visible) {
                const id = visible.target.id
                if (id && id !== current.value) {
                    current.value = id
                    // update URL hash without scrolling
                    try {
                        const url = new URL(window.location.href)
                        url.hash = `#${id}`
                        history.replaceState(null, document.title, url.toString())
                    } catch (e) {
                        window.location.hash = id
                    }
                }
            }
        }

        observer = new IntersectionObserver(cb, { root: null, rootMargin, threshold: [0, 0.25, 0.5, 0.75, 1] })
        elements.forEach((el) => observer?.observe(el))
    }

    function destroy() {
        observer?.disconnect()
        observer = null
    }

    onBeforeUnmount(() => destroy())

    return { current, init, destroy }
}

```

然后将其注入到App.vue中监视活跃元素，

```vue
<template>
    <div id="top" class="relative min-h-screen overflow-x-hidden text-[#f7f1e8]">
        <Background />
        <TopBar :active-section="spy.current.value" />

        <main class="mx-auto w-[min(1200px,calc(100%-2rem))] pb-4 md:w-[min(1200px,calc(100%-4rem))]">
            <Hero />
            <Policy />
            <Nodes :nodes="nodes" />
            <Contact />
        </main>

        <Footer />
        <CopyTooltip />
    </div>
</template>

<script setup lang="ts">
import Background from "@/components/Background.vue"
import Contact from "@/components/Contact.vue"
// import ... components
import TopBar from "@/components/TopBar.vue"
import { nodes } from "@/data"
import useScrollSpy from "@/composables/useScrollSpy"
import { onMounted } from "vue"

const spy = useScrollSpy()

onMounted(() => {
    // observe main sections to keep URL hash in sync
    spy.init(["policy", "nodes", "contact"])
})
</script>
```

#### 方案二：纯粹监听 scroll 事件并特殊处理底部问题

把整个 `useScrollSpy` 的实现都改成使用以下监听 scroll 事件的方法来实现，App.vue 中的注册方式不变。

```typescript
import { onBeforeUnmount, ref } from "vue"

export default function useScrollSpy() {
    const current = ref<string | null>(null)
    let elements: HTMLElement[] = []
    let lastId: string | null = null
    let ticking = false
    let onScroll: (() => void) | null = null

    function updateHash(id: string) {
        if (!id || id === current.value) return
        current.value = id
        try {
            const url = new URL(window.location.href)
            url.hash = `#${id}`
            history.replaceState(null, document.title, url.toString())
        } catch {
            window.location.hash = id
        }
    }

    function detect() {
        // touching the bottom of the page should activate the last section
        const atBottom = window.innerHeight + window.scrollY >= document.documentElement.scrollHeight - 2
        if (atBottom && lastId) {
            updateHash(lastId)
            return
        }

        // find the element whose top is closest to the center of the viewport but not below it
        const center = window.innerHeight / 2
        let closest: HTMLElement | null = null
        let minDist = Infinity

        for (const el of elements) {
            const rect = el.getBoundingClientRect()
            // only consider elements whose top is already in the viewport (top <= center)
            if (rect.top <= center) {
                const dist = center - rect.top
                if (dist < minDist) {
                    minDist = dist
                    closest = el
                }
            }
        }

        if (closest) updateHash(closest.id)
    }

    function init(ids: string[]) {
        if (typeof window === "undefined") return

        elements = ids.map((id) => document.getElementById(id)).filter(Boolean) as HTMLElement[]
        if (!elements.length) return

        lastId = ids[ids.length - 1]

        onScroll = () => {
            if (!ticking) {
                ticking = true
                requestAnimationFrame(() => {
                    detect()
                    ticking = false
                })
            }
        }

        window.addEventListener("scroll", onScroll, { passive: true })
        // detect in case the page is not scrolled to top when loaded
        detect()
    }

    function destroy() {
        if (onScroll) {
            window.removeEventListener("scroll", onScroll)
            onScroll = null
        }
    }

    onBeforeUnmount(() => destroy())

    return { current, init, destroy }
}
```

## 部署

### 配置与构建

Dockerfile、dockerignore、compose.yaml、容器内 Nginx.conf，宿主机 Nginx.conf，基本上你需要的就是这些了。

首先写个 Dockerfile 来构建这个 Vue 应用，分为 build 阶段和 runtime 阶段：

```dockerfile
FROM node:22-alpine3.22 AS build
RUN apk update && apk upgrade --no-cache
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:1.29-alpine AS runtime
RUN apk update && apk upgrade --no-cache
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

注意构建使用的镜像避免使用浮动标签，应该指定一个具体的版本号，例如 `node:22-alpine3.22` 和 `nginx:1.29-alpine`，以确保构建环境的一致性和可预测性。另外，在每个阶段使用

```dockerfile
RUN apk update && apk upgrade --no-cache
```

来打上最新的 Alpine 安全补丁，以确保基础镜像中的包都是最新的，避免潜在的安全问题。基础镜像的发布时间点和 build 的时间点之间，Alpine 上游可能已经推送了修复包，apk upgrade 会将这些修复拉进来。

接下来写一个 compose.yaml 来定义服务：

```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    image: sisy-dn42-website:latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:80"
```

这个 compose 使用当前目录下的 Dockerfile 来构建镜像，并将容器的 80 端口映射到宿主机的 8080 端口上。这样一来，宿主机 Nginx 将 80/443 流量反代到 8080 上，容器内 Nginx 监听到容器的 80 端口传来流量，并做相应处理。

容器内的 Nginx 简单配置一下，宿主机的 Nginx 就不谈了：

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(?:css|js|mjs|json|svg|png|jpg|jpeg|gif|ico|webp|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable";
        access_log off;
    }
}
```

### 部署问题

`Dockerfile` 中要求的 `npm ci` 需要 lockfile 文件，如果没有这个文件，构建会失败。另外，还需要特别确保项目根目录下的 lockfile 文件与 `package.json` 中的依赖版本一致。为此，本地可能需要把 `package-lock.json` 和 `node_modules` 彻底删除，重新使用 `npm install` 生成一次 lockfile。

解决该问题后，起 compose 应该就没问题了：

```bash
docker compose up -d --build
```

## 自动化部署

每次修改代码推送到 Github 上之后都要在服务器上 pull 并重启服务，感觉有点麻烦，所以就想用 Github Actions 来实现自动部署。

### 方案

使用 GitHub Actions 将服务部署到自己的远程主机主要有两种方式：

1. 通过 SSH 部署到远程服务器

这是最常见的方式。在 workflow 中通过 SSH 连接到自己的服务器，执行部署命令。常用的第三方 Action 如 `appleboy/ssh-action`：

```yaml
- name: Deploy via SSH
  uses: appleboy/ssh-action@v1
  with:
    host: ${{ secrets.SERVER_HOST }}
    username: ${{ secrets.SERVER_USER }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: |
      cd /path/to/your/app
      git pull
      docker compose up -d --build
```

2. 使用 Self-hosted Runner

在自己的服务器上安装 GitHub Actions Runner，让 workflow 直接在服务器上执行，无需 SSH：

```yaml
jobs:
  deploy:
    runs-on: self-hosted   # 指定使用自托管的 runner
    steps:
      - uses: actions/checkout@v4
      - run: docker compose up -d --build
```

相比之下 SSH 方案更简单易行，只需把 SSH 密钥存到 GitHub Secrets 即可，适合大多数用户，而 Self-hosted Runner 则提供了更高的安全性和灵活性，Runner 常驻在你的服务器上，不需要暴露 SSH 端口，适合需要访问内网资源或有更复杂 CI/CD 流程的场景，但需要额外的维护工作。

除此之外，还可以通过 `rsync`、`scp` 等方式传输构建产物到远程主机，或者利用 `docker/build-push-action` 推送镜像到私有仓库后再在远程主机上拉取。总之 GitHub Actions 在这方面非常灵活，基本上能在终端里做的事情，都可以在 workflow 中实现。

对比各种方案之下，还是 SSH 方案比较简易且优雅，下面写个 workflow 来实现这个方案：

```yaml
name: Deploy via SSH

on:
  push:
    branches: [ "main" ]
  workflow_dispatch: {}

jobs:
  deploy:
    name: Deploy to server via SSH
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          # optional ssh port
          port: ${{ secrets.SERVER_PORT }}
          script: |
            cd ${{ secrets.SERVER_DEPLOY_PATH }}
            git pull
            docker compose up -d --build
```

这个 workflow 会在每次 push 到 main 分支或者手动触发时执行，Github 的一台临时主机将通过 SSH 连接到服务器，进入指定目录，拉取最新代码，并重建并启动 Docker Compose 服务。

然而最终发现此方案需要调整很多东西，不像表面看上去那么简单，只需要几个 Github Secrets 就能搞定。

### 你可能会遇到的问题

1. 设计上，首先考虑到安全问题，不应该使用 root 用户来操作，所以你需要一个配置好权限的 deployer 用户：

```bash
adduser deployer
chown -R deployer:deployer /path/to/your/app # 确保 deployer 用户对部署目录的所有子目录和子内容有读写权限
```

2. 其次你需要注意仓库的的父目录不能处于 `/root/` 等 deployer 用户无法访问的目录下，否则即使内部目录权限正确，也无法进入它。

否则你会在 CI 中遇到报错：

```plaintext
sh: 1: cd: can't cd to ***
fatal: not a git repository (or any of the parent directories): .git
no configuration file provided: not found
```

3. 此外，在 `appleboy/ssh-action` 的 script 中，`~` 可能不会被正确解析。你必须使用绝对路径来指定部署目录，例如 `/home/deployer/app`，而不能使用 `~/app`。
4. 你传入 Github Secrets 的 SSH 私钥必须是属于 deployer 用户的，而不是 root 用户的，且公钥应被添加到其 ~/.ssh/authorized_keys 文件中，否则 CI 无法正确连接到 deployer 用户。

```bash
# 以 deployer 身份生成 SSH key
su - deployer -c "ssh-keygen -t ed25519 -C 'deployer' -f /home/deployer/.ssh/id_ed25519 -N ''"
cat /home/deployer/.ssh/id_ed25519.pub
cat /home/deployer/.ssh/id_ed25519
# 验证能否正常 pull
su - deployer -c "cd /home/deployer/app && git pull"
```

否则你会在 CI 中遇到报错：

```plaintext
Host key verification failed.
fatal: Could not read from remote repository.
Please make sure you have the correct access rights
and the repository exists.
```

如果你的项目是用 root 用户 git clone 下来并移动过来的，那么你可能还需要为 deployer 用户设置 git 的 `safe.directory` 配置

```bash
# 以 deployer 身份写入 known_hosts
su - deployer -c "ssh-keyscan github.com >> ~/.ssh/known_hosts"

# 解决 git safe.directory 问题（为 deployer 用户配置）
su - deployer -c "git config --global --add safe.directory /home/deployer/SISY-DN42-Website"

# 检查 git remote
su - deployer -c "cd /home/deployer/SISY-DN42-Website && git remote -v"
```

否则在执行 `git pull` 时会遇到以下错误：

```plaintext
fatal: detected dubious ownership in repository at '/home/deployer/app'
To add an exception for this directory, call:
    git config --global --add safe.directory /home/deployer/app
```

5. `docker compose up -d --build` 需要 deployer 用户有权限执行 Docker 命令。这意味着 deployer 用户需要被添加到 `docker` 组中：

```bash
usermod -aG docker deployer
su - deployer # 或者重新登录 deployer 用户会话以使组权限生效
docker ps # 确认 deployer 用户现在有权限执行 Docker 命令
```

否则你会在 CI 中遇到报错：

```plaintext
unable to get image 'sisy-dn42-website:latest': permission denied while trying to connect to the docker API at unix:///var/run/docker.sock
```

## 结语

这样一来整个面板的开发和部署流程就都走了一遍了，后续可能还会继续完善一些功能。over!
