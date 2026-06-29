<a name="readme-top"></a>

# node-hiprint-transit

`node-hiprint-transit` 是 ArcoPrint 的 Socket.IO 中转服务，用于让公网 Web 系统把打印任务转发到内网或本地运行的 ArcoPrint 客户端。

它只负责连接管理和事件转发，不生成 PDF，不处理打印排版，也不直接访问打印机。真正打印由 ArcoPrint 客户端完成。


## 功能

- Web 与 ArcoPrint 客户端通过同一个 Token 加入同一组连接。
- Web 可以获取在线客户端列表和打印机列表。
- Web 可以指定某个 ArcoPrint 客户端下发打印任务。
- 支持 HTTP 或 HTTPS/WSS。
- 支持 Docker、Node.js、Windows 打包产物运行。

## 适用分支

ArcoPrint 目前有两类客户端分支：

| 客户端分支 | 推荐用途 | 中转打印方式 |
| --- | --- | --- |
| `simple` | 正式稳定使用 | 只推荐 `blob_pdf` |
| `sixinone` | 兼容旧方案和排查问题 | 支持 `html`、`url_pdf`、`blob_pdf`、`printByFragments`、`render-print`、`render-pdf/render-jpeg` |

正式业务对接建议使用 `blob_pdf`：Web 端生成最终 PDF，把 PDF Blob 交给 ArcoPrint 打印。这样中转服务和客户端都不参与 HTML 排版，结果更稳定。

## 安装

### Node.js 运行

```bash
git clone git@github.com:lwdsw/node-hiprint-transit.git
cd node-hiprint-transit
npm install
node init.js
node index.js
```

也可以使用已构建的 `dist` 目录运行：

```bash
cd dist
node init.js
node index.js
```

### Docker 运行

先准备配置文件和日志目录：

```bash
mkdir -p /var/hiprint/logs
cp config.json /var/hiprint/config.json
```

然后启动：

```bash
docker-compose up -d
```

`docker-compose.yml` 默认挂载：

```yaml
volumes:
  - /var/hiprint/config.json:/node-hiprint-transit/config.json
  - /var/hiprint/logs:/node-hiprint-transit/logs
```

如果启用 SSL，可以额外挂载：

```yaml
  - /var/hiprint/ssl.key:/node-hiprint-transit/src/ssl.key
  - /var/hiprint/ssl.pem:/node-hiprint-transit/src/ssl.pem
```

### Windows 运行

可以使用 `out/transit-setup-0.0.6.exe` 解压安装后运行 `start.bat`。

## 配置

`config.json` 示例：

```json
{
  "port": 17521,
  "token": "arcoprint",
  "useSSL": false,
  "lang": "en"
}
```

| 字段 | 说明 |
| --- | --- |
| `port` | 服务端口，默认 `17521` |
| `token` | 连接鉴权 Token，长度至少 6 位，支持 `*` 通配符 |
| `useSSL` | 是否启用 HTTPS/WSS |
| `lang` | 日志语言，支持 `en`、`zh` |

初始化向导：

```bash
node init.js
```

## SSL 证书

启用 SSL 时，服务会读取：

- `src/ssl.key`
- `src/ssl.pem`

测试环境可以生成自签证书：

```bash
openssl req -x509 -newkey rsa:2048 \
  -keyout src/ssl.key \
  -out src/ssl.pem \
  -days 3650 \
  -nodes \
  -subj "/CN=localhost"
```

正式环境建议使用可信域名证书，或者用 Nginx/Caddy 做 HTTPS 反向代理，Node 服务保持 HTTP 内网运行。

## 连接方式

### ArcoPrint 客户端连接中转服务

在 ArcoPrint 设置页开启中转服务，填写：

- 中转地址：`http://服务器 IP:17521` 或 `https://域名:17521`
- Token：与服务端 `config.json` 中匹配的 Token

连接成功后，中转服务会收到客户端信息和打印机列表。

### Web 连接中转服务

```js
import { io } from "socket.io-client";

const socket = io("http://your-server:17521", {
  transports: ["websocket", "polling"],
  auth: {
    token: "arcoprint",
  },
});

socket.on("connect", () => {
  console.log("transit connected", socket.id);
});

socket.on("connect_error", (error) => {
  console.error("transit connect error", error.message);
});
```

## 获取客户端和打印机

Web 连接后，服务端会自动推送：

- `serverInfo`
- `clients`
- `printerList`

也可以主动刷新：

```js
socket.emit("getClients");
socket.on("clients", (clients) => {
  console.log(clients);
});

socket.emit("refreshPrinterList");
socket.on("printerList", (printers) => {
  console.log(printers);
});
```

`printerList` 中每个打印机会带上 `server.clientId`，打印时需要把它作为 `client` 传回中转服务。

## PDF Blob 打印示例

推荐 Web 端生成 PDF 后，通过 `news` 事件发送 `blob_pdf`。

```js
async function printPdfBlob({ socket, clientId, printer, pdfBlob }) {
  const arrayBuffer = await pdfBlob.arrayBuffer();

  socket.emit("news", {
    client: clientId,
    type: "blob_pdf",
    pdf_blob: new Uint8Array(arrayBuffer),
    templateId: crypto.randomUUID(),
    printer: printer || "",
    pageSize: "A4",
    pageNum: 1,
  });
}

socket.on("success", (payload) => {
  console.log("打印成功", payload);
});

socket.on("error", (payload) => {
  console.error("打印失败", payload);
});
```

也可以发送 base64 或 data URI：

```js
socket.emit("news", {
  client: clientId,
  type: "blob_pdf",
  pdf_blob: "data:application/pdf;base64,JVBERi0xLjc...",
  templateId: "order-10001",
  printer: "",
  pageSize: "A4",
  pageNum: 1,
});
```

## 兼容事件

中转服务会转发以下打印事件：

| 事件 | 说明 | 建议 |
| --- | --- | --- |
| `news` | 常规打印任务。`simple` 分支只推荐 `type: "blob_pdf"` | 推荐 |
| `printByFragments` | 分片 HTML 打印 | 仅 `sixinone` 兼容 |
| `render-print` | 客户端渲染后打印 | 仅 `sixinone` 兼容 |
| `render-pdf` | 客户端渲染并返回 PDF | 仅 `sixinone` 兼容 |
| `render-jpeg` | 客户端渲染并返回图片 | 仅 `sixinone` 兼容 |

服务端只负责把这些事件转发给指定 `client`，成功和失败由 ArcoPrint 客户端回传：

- `success`
- `error`
- `${event}-success`
- `${event}-error`

## IPP 事件

中转服务也保留 IPP 相关转发：

- `ippPrint`
- `ippPrinterConnected`
- `ippPrinterCallback`
- `ippRequest`
- `ippRequestCallback`

这些事件同样需要指定 `client`。

## 运行日志

服务会把日志写入 `./logs` 目录。Docker 部署时建议挂载到宿主机目录，方便排查连接、鉴权和打印转发问题。

## 相关项目

| 项目 | 地址 | 说明 |
| --- | --- | --- |
| ArcoPrint 客户端 | https://github.com/lwdsw/hiprint-client | 本地静默打印客户端 |
| ArcoPrint Transit | https://github.com/lwdsw/node-hiprint-transit | Web 与 ArcoPrint 的中转服务 |

<p align="right"><a href="#readme-top">回到顶部</a></p>
