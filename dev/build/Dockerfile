FROM harbor.gdalpha.com/alpha-tools/nodejs:26.12.1
LABEL maintainer="huanglshu26@gmail.com"

# 使用阿里云镜像源。
ENV NPM_CONFIG_REGISTRY=https://registry.npmmirror.com
RUN find /etc/apt -type f \( -name '*.list' -o -name '*.sources' \) \
        -exec sed -i 's|deb.debian.org|mirrors.aliyun.com|g; s|security.debian.org|mirrors.aliyun.com|g' {} + && \
    DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -qy --no-install-recommends \
    bash \
    build-essential \
    curl \
    fonts-liberation \
    git \
    gnupg \
    openssh-client \
    pandoc \
    tini \
    && rm -rf /var/lib/apt/lists/*
RUN mkdir -p /wiki && \
    mkdir -p /logs && \
    mkdir -p /wiki/data/content && \
    mkdir -p /wiki/data/repo && \
    chown -R node:node /wiki /logs

WORKDIR /wiki

COPY --chown=node:node ./assets ./assets
COPY --chown=node:node ./blocks/compiled ./blocks/compiled
COPY --chown=node:node ./backend ./backend
COPY --chown=node:node ./dev/build/config.yml ./config.yml
COPY --chown=node:node ./LICENSE ./LICENSE

USER node

ENV NODE_ENV=production

# Puppeteer 扩展使用的浏览器由 Puppeteer 自行安装，而不是使用发行版提供的浏览器：
# Puppeteer 只兼容其构建时对应的 Chrome 版本，而发行版软件包会随安全更新不断升级，
# 导致两者版本差距逐渐扩大，最终使页面渲染失败。此处由 Puppeteer 安装并固定浏览器版本。
#
# 渲染只需要无头浏览器，因此跳过完整浏览器安装。
# 缓存使用固定路径而不是 $HOME，以确保无论容器以哪个用户运行都能找到缓存；
# 在 OpenShift 上运行的任意 UID 可能没有自己的主目录。
ENV PUPPETEER_CACHE_DIR=/wiki/.cache/puppeteer
ENV PUPPETEER_CHROME_SKIP_DOWNLOAD=true

WORKDIR /wiki/backend
RUN npm ci --omit=dev

# Puppeteer 扩展用于服务端页面渲染。它没有声明在 `backend/package.json` 中，因为这是一个可选扩展：
# 只需要在编辑器中渲染页面的安装（正常保存时都会如此）不需要服务端浏览器，
# 源码检出时也不应为了安装后端而下载浏览器。
#
# 版本从扩展定义中读取，管理员手动安装 Puppeteer 时也使用同一版本：
# 只需在一个位置升级版本，镜像就不会与手动安装的实例产生偏差。
# 如果读取结果为空，构建会直接失败，而不是悄悄安装最新版本。
#
# Puppeteer 的 postinstall 会下载浏览器，`browsers install` 用于确保浏览器确实存在：
# npm 可能被配置为不执行安装脚本；如果浏览器已经存在，该命令只会输出其路径。
RUN PUPPETEER_VERSION="$(sed -n 's/^installVersion: *//p' modules/extensions/puppeteer/definition.yml)" && \
    test -n "$PUPPETEER_VERSION" && \
    npm install --no-save "puppeteer@${PUPPETEER_VERSION}" && \
    npx puppeteer browsers install chrome-headless-shell

# 安装该浏览器需要的系统库，这些库不再作为发行版软件包的依赖自动安装。
# 每个 Chrome for Testing 构建都会在 `deb.deps` 中按架构列出所需依赖，
# 这里以 root 身份执行 `apt-get satisfy`，等同于 `browsers install --install-deps` 的操作，
# 避免以 root 运行 Puppeteer 导致缓存文件归属 root。
USER root
RUN DEPS="$(find "$PUPPETEER_CACHE_DIR" -name deb.deps)" && \
    test "$(printf '%s\n' "$DEPS" | grep -c .)" -eq 1 && \
    apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get satisfy -qy --no-install-recommends "$(paste -sd, "$DEPS")" && \
    rm -rf /var/lib/apt/lists/*
USER node

# 按渲染器的方式启动浏览器并访问一次页面，以便发现版本不匹配或安装不完整的问题。
# 无法渲染的浏览器会在镜像构建阶段直接报错，而不是等到每个实例运行时才失败。
#
# 仅在原生构建时执行完整启动检查。多平台构建会在 QEMU 用户态模拟下运行其他架构，
# Chrome 可能启动并接受 DevTools 连接，但在第一次协议调用时以 “Target closed” 退出，
# 即使安装本身没有问题。因此这种构建只检查二进制文件是否属于目标架构，
# 以发现旧版本固定错误导致的架构不匹配。ELF 头部偏移 18 处的 e_machine：
# x86-64 为 0x3e，aarch64 为 0xb7。
ARG BUILDARCH
ARG TARGETARCH
RUN if [ -z "$TARGETARCH" ] || [ "$TARGETARCH" = "$BUILDARCH" ]; then \
      node --input-type=module -e " \
        import puppeteer from 'puppeteer'; \
        const browser = await puppeteer.launch({ headless: 'shell', args: ['--no-sandbox', '--disable-dev-shm-usage'] }); \
        const page = await browser.newPage(); \
        await page.goto('data:text/html,<p>ok</p>'); \
        if ((await page.\$eval('p', (e) => e.textContent)) !== 'ok') throw new Error('The browser did not render.'); \
        console.log('Puppeteer drives ' + (await browser.version())); \
        await browser.close();"; \
    else \
      node --input-type=module -e " \
        import { openSync, readSync } from 'node:fs'; \
        import puppeteer from 'puppeteer'; \
        const file = await puppeteer.executablePath({ headless: 'shell' }); \
        const header = Buffer.alloc(20); \
        readSync(openSync(file), header, 0, 20, 0); \
        const machine = header.readUInt16LE(18); \
        const expected = { x64: 0x3e, arm64: 0xb7 }[process.arch]; \
        if (machine !== expected) throw new Error(file + ' is not a ' + process.arch + ' binary (e_machine 0x' + machine.toString(16) + ').'); \
        console.log('Emulated build, not launched: ' + file + ' is ' + process.arch);"; \
    fi

WORKDIR /wiki

VOLUME ["/wiki/data/content"]
VOLUME ["/wiki/data/repo"]

EXPOSE 3000
EXPOSE 3443

# 在 node 前使用 init 进程，因为 PID 1 会继承容器内所有孤儿进程并负责回收，而 node 不会。
# git 在每次 commit、pull 和 rebase 时都会产生孤儿进程：它退出时执行
# `git maintenance run --auto --detach`，无论是否有内容需要维护都会转入后台，随后父进程退出。
# 因此使用 git 存储时，每次保存页面都会留下一个僵尸进程，繁忙的 wiki 中可能积累数千个。
# Puppeteer 驱动的浏览器也会以相同方式产生自己的辅助进程。
#
# `-s` 让 tini 注册为 subreaper，因此即使它不是 PID 1 本身（例如 `docker run --init`，
# 或共享进程命名空间的 Pod），也能回收这些进程。信号会原样转发给 node，
# 因此 `index.ts` 中的优雅关闭逻辑仍能收到与之前相同的 SIGTERM。
ENTRYPOINT ["tini", "-s", "--"]

# 关闭 Web Storage，恢复 Node 26 默认开启该功能之前的行为。
#
# 这里没有使用 `localStorage`，但 yjs 依赖的 `lib0` 在加载时会探测它，
# 因为该库需要同时运行在浏览器和 Node.js 中。如果没有指定 `--localstorage-file`，
# 探测会产生实验性功能警告，导致每个容器日志的第一行都是该警告。
# 关闭后全局对象不存在，`lib0` 会静默地走 Node.js 分支。
CMD ["node", "--no-experimental-webstorage", "backend"]
