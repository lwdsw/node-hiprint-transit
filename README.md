<a name="readme-top"></a>

# node-hiprint-transit

`node-hiprint-transit` 是 ArcoPrint 的 Socket.IO 中转服务，用来把 Web 前端的打印任务转发给运行在用户电脑上的 ArcoPrint 客户端。

它只做三件事：

- 维护 Web 前端和 ArcoPrint 客户端的连接
- 转发打印机列表
- 转发打印任务和打印结果

它不生成 PDF，不处理排版，也不直接访问打印机。真正执行打印的是 ArcoPrint 客户端。

## 整体流程

```text
Web 前端
  -> node-hiprint-transit 中转服务
  -> ArcoPrint 客户端
  -> 本机打印机
```

Web 前端和 ArcoPrint 客户端必须连接同一个中转服务，并使用同一个 Token。

## 部署中转服务

### 使用 dist 运行

把构建后的 `dist` 目录上传到服务器：

```bash
cd dist
node index.js
```

启动成功后会看到类似输出：

```text
node-hiprint-transit version: 1.0.0

Serve is running on
http://服务器IP:17521

Please make sure that the ports have been opened in the security group or firewall.
token: arcoprint
```

这里的 `token: arcoprint` 来自 `config.json`，不是固定协议值。你可以改成自己的 Token，但 Web 前端和 ArcoPrint 客户端要保持一致。

### 从源码运行

```bash
git clone git@github.com:lwdsw/node-hiprint-transit.git
cd node-hiprint-transit
npm install
node index.js
```

### Docker 运行

```bash
mkdir -p /var/hiprint/logs
cp config.json /var/hiprint/config.json
docker-compose up -d
```

如果修改端口，需要同时修改 `/var/hiprint/config.json` 和 `docker-compose.yml` 里的端口映射。

### Windows服务器运行

```bash
npm run build
npm run package
```

生成：

```text
out/transit-setup-1.0.0.exe
```

上传到 Windows 服务器后运行解压，修改解压目录里的 `config.json`，再执行 `start.bat`。

## 配置

`config.json` 示例：

```json
{
  "port": 17521,
  "token": "arcoprint",
  "useSSL": false,
  "lang": "en",
  "maxHttpBufferSizeMB": 100
}
```

| 字段 | 说明 |
| --- | --- |
| `port` | 服务端口，默认 `17521`，范围 `1024~65535` |
| `token` | 连接鉴权 Token，长度至少 6 位；Web 前端和 ArcoPrint 客户端必须一致 |
| `useSSL` | 是否启用 HTTPS/WSS |
| `lang` | 日志语言，支持 `en`、`zh` |
| `maxHttpBufferSizeMB` | Socket.IO 单条消息最大体积，单位 MB，默认 `100`，范围 `1~1024` |

修改配置后需要重启中转服务。

如果需要把端口改为 `9993`：

```json
{
  "port": 9993,
  "token": "你的 Token",
  "useSSL": false,
  "lang": "en",
  "maxHttpBufferSizeMB": 100
}
```

然后 Web 前端和 ArcoPrint 客户端都连接：

```text
http://服务器IP:9993
```

## ArcoPrint 使用

1. 先启动中转服务。
2. 打开 ArcoPrint 客户端设置页。
3. 开启中转服务连接。
4. 填写中转服务地址，例如 `http://服务器IP:17521`。
5. 填写 Token，必须和中转服务 `config.json` 里的 `token` 一致。
6. 保存后观察中转服务日志，看到 `(arcoprint)` 连接和 `update printerList` 即表示客户端已连上并上报打印机。

ArcoPrint 客户端连接中转服务时会携带：

```js
query: {
  client: "arcoprint"
}
```

这个标识由 ArcoPrint 客户端使用。Web 前端不要传 `query.client = "arcoprint"`，否则中转服务会把它当成打印客户端。

## 前端对接

当前中转服务使用 Socket.IO v4，前端建议使用 v4 客户端：

```bash
npm install socket.io-client
```

### 连接中转服务

```js
import { io } from "socket.io-client";

const socket = io("http://服务器IP:17521", {
  transports: ["websocket", "polling"],
  auth: {
    token: "你的 Token",
  },
});

socket.on("connect", () => {
  console.log("transit connected", socket.id);
});

socket.on("connect_error", (error) => {
  console.error("transit connect error", error.message);
});
```

前端连接的是中转服务地址，不是用户电脑上的 ArcoPrint 本地地址。

### 获取打印机列表

```js
socket.emit("refreshPrinterList");

socket.on("printerList", (printers) => {
  console.log(printers);
});
```

返回示例：

```js
[
  {
    name: "Air_Printer",
    displayName: "Air Printer",
    label: "Air Printer",
    value: "Air_Printer",
    client: "ArcoPrint 客户端 socket id",
    clientId: "ArcoPrint 客户端 socket id",
    isDefault: true,
    status: 3,
    server: {
      clientId: "ArcoPrint 客户端 socket id"
    }
  }
]
```

打印时需要带上这两个值：

```js
{
  client: printer.clientId,
  printer: printer.value
}
```

`client` 用来指定把任务发给哪一个在线 ArcoPrint 客户端，`printer` 用来指定该客户端上的打印机。

### PDF Blob 打印

推荐前端生成 PDF 后，通过 `news` 事件发送 `blob_pdf`。

```js
async function printPdfBlob({ socket, printer, pdfBlob }) {
  const arrayBuffer = await pdfBlob.arrayBuffer();

  socket.emit("news", {
    client: printer.clientId,
    printer: printer.value,
    type: "blob_pdf",
    pdf_blob: new Uint8Array(arrayBuffer),
    templateId: crypto.randomUUID(),
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

`pdf_blob` 也可以传 base64 或 data URI：

```js
socket.emit("news", {
  client: printer.clientId,
  printer: printer.value,
  type: "blob_pdf",
  pdf_blob: "data:application/pdf;base64,JVBERi0xLjc...",
  templateId: "order-10001",
  pageSize: "A4",
  pageNum: 1,
});
```

## 大 PDF 限制

PDF Blob 可能超过 Socket.IO 默认的单条消息大小限制。中转服务通过 `maxHttpBufferSizeMB` 控制单条消息最大体积，默认 `100MB`。

如果 PDF 超过限制，服务端可能在进入 `news` 事件前断开连接，日志里通常看不到 `send news to ...`。这时可以：

- 查看浏览器控制台是否已经执行 `socket.emit("news", payload)`
- 查看中转服务 `logs` 目录里的断开原因
- 适当调大 `maxHttpBufferSizeMB`

`maxHttpBufferSizeMB` 不建议无限调大。公网部署时应开启 Token 鉴权，并限制可信来源；如果 PDF 很大或并发较高，建议后续改为分片传输。

## 日志排查

日志写入运行目录下的 `logs`：

```bash
tail -f logs/$(date +%F).log
```

正常链路通常会看到：

```text
Client connected: xxx | 你的 Token | (arcoprint)
xxx update printerList, count: 4, total: 4
Client connected: yyy | 你的 Token | (web-client)
yyy request refreshPrinterList
yyy send printerList, count: 4
yyy send news to xxx
xxx client: print success, templateId: order-10001
```

如果没有打印机列表：

- ArcoPrint 客户端可能没有连接中转服务
- Web 前端和 ArcoPrint 客户端使用的 Token 可能不一致
- ArcoPrint 客户端所在电脑可能没有可用打印机

如果没有 `send news to ...`：

- 前端可能没有发送 `news`
- 请求里可能缺少 `client`
- `client` 可能不是当前在线 ArcoPrint 客户端的 socket id
- PDF 可能超过 `maxHttpBufferSizeMB`

## HTTPS/WSS

启用 SSL 时，服务会读取：

- `src/ssl.key`
- `src/ssl.pem`

生产环境建议使用 Nginx/Caddy 做 HTTPS 反向代理，Node 服务保持 HTTP 内网运行。

## 相关项目

| 项目 | 地址 | 说明 |
| --- | --- | --- |
| ArcoPrint 客户端 | https://github.com/lwdsw/hiprint-client | 本地打印客户端 |
| ArcoPrint Transit | https://github.com/lwdsw/node-hiprint-transit | Web 与 ArcoPrint 的中转服务 |

<p align="right"><a href="#readme-top">回到顶部</a></p>
