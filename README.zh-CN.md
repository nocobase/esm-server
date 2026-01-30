# ESM Server

NocoBase 前端 RunJS 支持动态加载 ESM 模块。可通过环境变量 `ESM_CDN_BASE_URL` 配置 ESM CDN 地址，默认为 `https://esm.sh`。内网使用时，可将 https://esm.sh 加入白名单，或自建 ESM CDN 服务（基于 [esm.sh](https://github.com/esm-dev/esm.sh)）。

ESM 模块不能像 UMD 模块那样直接通过 `<script src="...">` 引入即可使用。即便是 npm 包里已经构建好的 ESM（如 `dist/esm.mjs`），也不能在浏览器里直接使用：浏览器不支持裸说明符（如 `import "react"`）、依赖链需要逐层解析、还有 CORS 等限制，所以必须由服务端按「包名 + 版本」解析依赖、产出可被浏览器请求的 URL。UMD 是「打包好、可直接在浏览器里跑」的格式；ESM 则依赖 `import`/`export`、模块解析和 npm 包转换（如 UMD/CJS → ESM），因此需要一套 **ESM CDN 服务**（如 [esm.sh](https://github.com/esm-dev/esm.sh)）在服务端完成构建与转换，再通过 URL 提供给前端。本仓库即用于在自有环境中部署这类服务。

## 前置要求

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

## 在 NocoBase 中配置

```bash
ESM_CDN_BASE_URL=http://localhost:8060
```

## 目录结构

```
esm-server/
├── README.md              # 本说明
├── docker-compose.yml     # 含 Verdaccio 的示例（见下文）
├── verdaccio-config.yaml  # Verdaccio 配置（私有 npm 场景）
└── data/                  # 运行时生成，需持久化
    ├── esm/               # esm.sh 构建缓存（离线部署需拷贝）
    │   ├── esm/
    │   └── data/
    └── verdaccio/         # Verdaccio 存储（仅私有 npm 场景）
```

## 部署方式

### 方式一：仅使用公共 npm（无私有包）

适用于只拉取公共 npm 包、且可访问外网的环境。

1. 新建 `docker-compose.yml`（或单独文件如 `docker-compose.public.yml`），内容如下：

```yaml
version: "3.9"

services:
  esm-server:
    image: ghcr.io/esm-dev/esm.sh:latest
    restart: always
    ports:
      - "8060:80"
    volumes:
      - ./data/esm/esm:/esm
      - ./data/esm/data:/data
    environment:
      - NODE_ENV=production
      - NPM_REGISTRY=https://registry.npmmirror.com
      - STORAGE_TYPE=fs
      - STORAGE_ENDPOINT=/data
```

2. 启动：

```bash
docker compose up -d
```

3. 在 NocoBase 中设置 `ESM_CDN_BASE_URL=http://localhost:8060`。

### 方式二：使用私有 npm 镜像（Verdaccio）

适用于需要私有 npm 包或统一走内网 npm 源的场景。当前仓库中的 `docker-compose.yml` 即为此种方式。

```yml
version: "3.9"

services:
  verdaccio:
    image: verdaccio/verdaccio
    volumes:
      - ./data/verdaccio:/verdaccio/storage
      - ./verdaccio-config.yaml:/verdaccio/conf/config.yaml
    environment:
      # 在容器网络中使用服务名 verdaccio，而不是宿主机 localhost
      - VERDACCIO_PUBLIC_URL=http://verdaccio:4873

  esm-server:
    image: ghcr.io/esm-dev/esm.sh:latest
    restart: always
    ports:
      # 宿主机 8090 -> 容器内 80（镜像默认监听 80）
      - "8060:80"
    volumes:
      # esm.sh 构建缓存目录
      - ./data/esm/esm:/esm
      - ./data/esm/data:/data
    environment:
      - NODE_ENV=production
      - NPM_REGISTRY=http://verdaccio:4873
      - STORAGE_TYPE=fs
      - STORAGE_ENDPOINT=/data
```

启动

```bash
docker compose up -d
```

- **ESM 服务**：端口 `8060`，使用 Verdaccio 作为 npm 源（`NPM_REGISTRY=http://verdaccio:4873`）。
- **Verdaccio**：仅容器内网 `http://verdaccio:4873`，如需从宿主机发布/拉取私有包，需在 `verdaccio-config.yaml` 中配置并暴露端口（可自行添加 `ports: - "4873:4873"`）。

`verdaccio-config.yaml` 中已配置对公共包的代理（如 `npmmirror.com`），私有包可按需配置 `packages` 与 `publish`。

## 内网/离线 Docker 部署

在内网或无外网环境中，esm.sh 无法实时拉取 npm 包，需先用「有外网的机器」生成缓存，再拷贝到内网使用。

1. **在有外网的机器上**  
   - 按上文任选一种方式启动（建议方式一，简单）：  
     `docker compose up -d`  
   - 在 NocoBase（或直接请求）中使用 RunJS 加载需要的 npm 包，或主动请求一次目标 ESM 地址（如 `http://localhost:8060/vue@3`），使 esm.sh 构建并写入缓存。

2. **同步缓存**  
   - 将整份 **`./data/esm`** 目录拷贝到内网机器的同一路径下（例如内网机同样使用 `./data/esm`）。

3. **在内网机器上**  
   - 使用同一 `docker-compose` 配置启动：  
     `docker compose up -d`  
   - 已构建过的包会由本地缓存响应；未在缓存中的包会请求失败，需在可外网机器上先请求一次，再重新同步 `./data/esm`。

## 非 Docker 部署

不依赖 Docker 时，可从源码编译并直接运行 [esm.sh](https://github.com/esm-dev/esm.sh)。

### 前置要求

- [Go](https://go.dev/dl/) 1.21+（用于编译）
- 可访问外网或已配置内网 npm 源（用于拉取 npm 包）

### 构建与运行

```bash
git clone https://github.com/esm-dev/esm.sh.git
cd esm.sh
go build -ldflags="-s -w -X 'github.com/esm-dev/esm.sh/server.VERSION=main'" -o esmd server/esmd/main.go
```

**直接运行**（使用默认配置，监听 80 端口，数据目录 `~/.esmd`）：

```bash
./esmd
```

**指定配置文件运行**（推荐，便于改端口和 npm 源等）：

```bash
./esmd -config /path/to/config.json
```

### 配置文件

配置为 JSON/JSONC 格式。官方示例见 [config.example.jsonc](https://github.com/esm-dev/esm.sh/blob/main/config.example.jsonc)。

示例 `config.json`（监听 8060、使用国内源、本地存储）：

```json
{
  "port": 8060,
  "npmRegistry": "https://registry.npmmirror.com",
  "storage": {
    "type": "fs",
    "endpoint": "~/.esmd/storage"
  },
  "workDir": "~/.esmd"
}
```

内网/私有包场景：将 `npmRegistry` 改为内网 Verdaccio 地址（如 `http://verdaccio-host:4873`），必要时配置 `npmScopedRegistries`、`npmToken` 等。

## 常见问题

- **端口**：文档与示例使用 `8060`，可按需在 `ports` 中修改宿主机端口（如 `"8090:80"`），并相应修改 NocoBase 的 `ESM_CDN_BASE_URL`。
- **镜像拉取**：若无法访问 `ghcr.io`，可先在有外网环境拉取镜像并导出，再在内网导入后使用。
- **私有包 404**：确认 Verdaccio 已发布该包，且 esm-server 的 `NPM_REGISTRY` 指向该 Verdaccio 地址（容器内使用服务名 `http://verdaccio:4873`）。
