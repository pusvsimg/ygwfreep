# BPB Panel 部署教程：个人专属免费 VPN 节点

BPB Panel 是一个结合 Cloudflare Workers 和 Pages 的代理面板项目，可以帮助用户轻松搭建免费 VPN，实现永久免费节点订阅，为使用 singbox-core 和 xray-core 的跨平台客户端提供配置。

由于 Cloudflare 官方收紧对 BPB 等项目的审查，如果直接使用源码或者原作者提供的混淆代码，很容易出现 1101 的报错（可能代码中包含敏感关键词、或者使用了与他人相同的混淆代码）。

解决办法是利用未混淆的源码进行自定义加密混淆，从而生成独一无二的混淆代码，成功绕过 Cloudflare 的限制。

## 一、准备工作
- **GitHub 账号**：通过 Github Action 自动同步最新 BPB 源代码，并执行代码混淆。
- **Cloudflare 账号**：用于部署 BPB Panel 项目。
- **域名**：建议使用域名（解决 Cloudflare Pages 自带域名被墙的问题）。

## 二、Github 部署

### 新建 Github 仓库
创建一个新的私有仓库，将 BPB Panel 项目代码同步到该仓库。

### 配置 Github Actions
在仓库根目录下创建 `.github/workflows` 文件夹，并在其中创建 `Obfuscate.yml` 文件。
将以下代码粘贴进去：

```yaml
name: Build Obfuscate BPB Panel

on:
  push:
    branches:
      - main
  schedule:
        # Runs everyday at 1:00 AM
        - cron: "0 1 * * *"

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Check out the code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "latest"

      - name: Install dependencies
        run: |
          npm install -g javascript-obfuscator

      - name: Clone BPB workjs
        run: |
          wget -O origin.js https://raw.githubusercontent.com/bia-pain-bache/BPB-Worker-Panel/refs/heads/main/build/unobfuscated-worker.js

      - name: Obfuscate BPB worker js
        run: |
          javascript-obfuscator origin.js --output _worker.js \
          --compact true \
          --control-flow-flattening false \
          --dead-code-injection false \
          --identifier-names-generator hexadecimal \
          --rename-globals true \
          --string-array true \
          --string-array-encoding 'base64' \
          --string-array-threshold 0.75 \
          --transform-object-keys false \
          --unicode-escape-sequence false

      - name: Commit changes
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          branch: main
          commit_message: ':arrow_up: update latest bpb panel'
          commit_author: 'github-actions[bot] <github-actions[bot]@users.noreply.github.com>'
          pull_before_push: true
```

**注意**：此 Action 将在每次 push 到 main 分支和每天凌晨 1 点自动执行，下载最新未混淆的 BPB 源代码（origin.js），并生成混淆后的 _worker.js 文件，作为你个人专属的 BPB 代码。我们采用了轻量级混淆方案，以避免 Cloudflare 的 CPU 限制错误，同时保持代码的安全性。

### 执行 Github Actions
GitHub 仓库通过 Obfuscate.yml 会自动下载最新的 BPB 源代码，并执行混淆：
- **origin.js**：最新未加密的 BPB 源代码
- **_worker.js**：混淆后的个人专属 BPB 代码

## 三、Cloudflare 部署

### 登录 Cloudflare，创建 Pages 部署
1. 在 Cloudflare 控制台中进入 Workers 和 Pages，选择 Pages 部署。
2. 连接到你的 Github 仓库，选择刚才新建的 BPB 项目仓库，然后点击开始部署。
3. 点击"保存并部署"。

### 绑定自定义域名（可选）
在 Pages 项目的 自定义域 选项卡，点击设置自定义域。

### 设置变量
部署成功后，在 Pages 项目界面点击 设置 -> 环境变量，添加以下变量：

- **UUID**：使用 UUID 生成器 随机生成一个新的 UUID。
- **PROXYIP**：填写代理 IP 地址，可从 随机代理 IP 站点 获取，或使用优选域名（例如 cdn-b100.xn--b6gac.eu.org）。
- **TR_PASS**：填写一个复杂字符串，作为密码。

### 绑定 KV 命名空间
**创建 KV**:
1. 点击左侧Workers & Pages。
2. 选择 KV。
3. 创建一个新的 KV 命名空间。
   注: 名称自定义但不能包含"bpb"。

**绑定 KV**:
1. 回到创建的 Pages 界面。
2. 点击 设置 -> 函数 -> KV 命名空间绑定 -> 添加绑定。
3. 变量名称: kv (小写)。
4. KV 命名空间: 选择您创建的命名空间。
5. 点击 保存。

### 重试部署 Pages
返回 Pages 项目, 找到右侧..., 点击重试部署

## 四、BPB 面板设置

### 验证部署是否成功
打开浏览器输入: https://[自定义域名] 或者你的项目地址, 后面加上/panel, 检查是否能正常访问BPB面板

### 修改 BPB 面板密码
第一次打开 BPB 面板会提示修改密码, 请设置一个复杂密码, 避免被盗用。使用您之前设置的 TR_PASS 作为旧密码, 并设置一个 *新的* 强密码。

### 配置 BPB 面板参数
- **FakeDNS**：设置enable
- **Proxy IPs / Domains**：填写 IP 或者域名，PROXYIP 获取地址：点击访问
- **Clean IPs / Domains**：优秀 IP 软件下载【点击 Download Scanner】 | 在线优选 IP：点击访问
- **TLS 端口**：需要使用的端口打勾就可以，默认是 443 端口
- **ROUTING RULES**：
  - Bypass xxx是指 xxx 不走代理（直连访问）
  - Block xxx是指 xxx 被屏蔽访问（无法访问）
  - Bypass LAN：绕过本地局域网
  - Block Ads：屏蔽广告网址
  - Bypass Iran：绕过伊朗
  - Block Porn：屏蔽颜色网站
  - Bypass China：绕过中国大陆
  - Block QUIC：屏蔽 QUIC 协议
  - Bypass Russia：绕过俄罗斯
- **CUSTOM RULES**：除了上面预设的规则外，你可以在这里自定义一些需要直连（Bypass）和屏蔽（Block）的 IP 地址/网站。

### 保存设置
点击 APPLY SETTINGS 保存 BPB 面板配置

## 五、VPN 节点部署完成

### 导出节点订阅链接
根据你所使用的代理软件，点击对应的 COPY SUB 按钮，复制 BPB 面板生成的订阅链接。

### V2rayN 客户端导入节点订阅链接并使用
1. 打开 V2rayN，进入【订阅】->【订阅设置】->【添加】,将订阅链接粘贴进去
2. 点击【订阅】->【更新订阅】，获取最新节点信息
3. 测试节点延迟,确认节点有效后,开启系统代理,即可使用 VPN

---

现在 BPB 面板 VPN 部署全部结束。通过以上步骤,你可以利用 Cloudflare 和 BPB Panel 搭建一个永久免费 VPN，同时通过对 BPB 源代码进行轻量级加密混淆生成专属混淆代码，成功绕过 Cloudflare 的审查和 CPU 限制，解决 1101 报错问题。本教程不仅支持 singbox-core 和 xray-core 等跨平台客户端等配置，还实现了永久免费节点订阅，满足大多数用户的使用需求。大家可以一起部署折腾，有什么问题，请在评论区留言，一起研究学习。
