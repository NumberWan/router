# vLLM Router 項目結構詳細說明

## 項目概述

**vLLM Router** 是一個高性能、輕量級的請求轉發系統，專為 vLLM 大規模部署設計。它提供了先進的負載平衡方法和 Prefill/Decode 分離架構支持。

### 技術架構

- **核心語言**: Rust（用於高性能路由邏輯）
- **Python 綁定**: PyO3（提供 Python 接口）
- **Web 框架**: Axum（Rust 異步 HTTP 框架）
- **協議支持**: HTTP/HTTPS 和 gRPC

---

## 目錄結構

```
vllm-router-standalone/
├── src/                    # Rust 核心源代碼
├── py_src/                 # Python 包源代碼
├── tests/                  # Rust 測試
├── py_test/               # Python 測試
├── benches/               # Rust 性能基準測試
├── examples/              # 配置示例
├── docs/                  # 文檔
├── scripts/               # 部署和工具腳本
├── Cargo.toml            # Rust 項目配置
├── setup.py              # Python 構建配置
├── pyproject.toml        # Python 項目配置
└── README.md             # 項目說明文檔
```

---

## 詳細文件說明

### 一、根目錄配置文件

#### 1.1 構建配置文件

**`Cargo.toml`**
- Rust 項目的配置文件
- 定義項目名稱、版本、依賴項
- 配置編譯特性（features）：`grpc-client`、`grpc-server`
- 定義庫類型為 `cdylib`（動態庫）和 `rlib`（Rust 庫）
- 包含開發依賴和基準測試配置

**`setup.py`**
- Python 包的構建腳本
- 使用 `setuptools-rust` 來構建 Rust 擴展
- 支持通過環境變量 `VLLM_ROUTER_BUILD_NO_RUST=1` 跳過 Rust 編譯
- 創建 PyO3 綁定的 Python 模塊

**`pyproject.toml`**
- Python 項目的現代化配置文件（PEP 518）
- 定義項目元數據、依賴項、可選依賴
- 配置 `setuptools` 構建後端
- 定義命令行入口點：`vllm-router`

**`build.rs`**
- Rust 構建腳本（build script）
- 編譯 Protocol Buffers 定義（`.proto` 文件）
- 使用 `tonic-build` 和 `prost-build` 生成 gRPC 代碼

#### 1.2 其他配置文件

**`Makefile`**
- 定義常用的構建、測試、安裝命令
- 簡化開發流程

**`Dockerfile.router`**
- Docker 容器鏡像構建配置
- 用於部署 router 服務

**`pytest.ini`**
- pytest 測試框架配置
- 定義測試路徑、選項等

**`MANIFEST.in`**
- Python 分發包的文件清單
- 指定哪些文件包含在分發包中

---

### 二、Rust 核心源代碼 (`src/`)

#### 2.1 程序入口點

**`src/main.rs`**
- Rust 二進制程序的主入口點
- 解析命令行參數（使用 `clap`）
- 轉換 CLI 參數為 `RouterConfig`
- 驗證配置
- 創建 Tokio 運行時並啟動服務器
- 支持多種模式：
  - 常規模式（Regular）
  - Prefill/Decode 分離模式（PD Disaggregation）
  - vLLM PD 模式（vLLM Prefill/Decode）
  - OpenAI 後端模式
  - IGW（Inference Gateway）模式

**`src/lib.rs`**
- Rust 庫的入口點
- 定義 PyO3 Python 綁定
- 導出 `Router` 和 `PolicyType` 類到 Python
- 提供 Python 可調用的接口

#### 2.2 配置模塊 (`src/config/`)

**`src/config/mod.rs`**
- 配置模塊的主入口
- 定義配置錯誤類型（`ConfigError`）：
  - `ValidationFailed`: 驗證失敗
  - `InvalidValue`: 無效值
  - `IncompatibleConfig`: 不相容配置
  - `MissingRequired`: 缺少必需字段
- 定義配置結果類型（`ConfigResult`）

**`src/config/types.rs`**
- 所有配置類型的定義
- 主要類型包括：
  - `RouterConfig`: 路由器主配置
  - `RoutingMode`: 路由模式（Regular、PrefillDecode、VllmPrefillDecode、OpenAI）
  - `PolicyConfig`: 負載平衡策略配置
  - `ConnectionMode`: 連接模式（Http、Grpc）
  - `DiscoveryConfig`: Kubernetes 服務發現配置
  - `MetricsConfig`: Prometheus 指標配置
  - `RetryConfig`: 重試配置
  - `CircuitBreakerConfig`: 熔斷器配置
  - `HealthCheckConfig`: 健康檢查配置

**`src/config/validation.rs`**
- 配置驗證邏輯
- 確保配置參數的合法性和一致性
- 檢查互斥選項的組合

#### 2.3 核心模塊 (`src/core/`)

**`src/core/mod.rs`**
- 核心模塊的入口點
- 導出所有核心功能

**`src/core/worker.rs`**
- Worker 實例的定義和管理
- Worker 狀態追蹤（健康、不健康、未知）
- Worker 元數據（URL、ID、類型）
- Worker 負載統計

**`src/core/worker_registry.rs`**
- Worker 註冊表（Registry）實現
- 管理所有 worker 的註冊、更新、移除
- 提供 worker 查詢接口
- 支持多種 worker 類型（Regular、Prefill、Decode）
- 線程安全的 worker 狀態管理

**`src/core/circuit_breaker.rs`**
- 熔斷器（Circuit Breaker）實現
- 狀態機：`Closed` → `Open` → `HalfOpen` → `Closed`
- 防止故障 worker 持續接收請求
- 自動恢復機制

**`src/core/retry.rs`**
- 重試邏輯實現
- 支持指數退避（Exponential Backoff）
- 支持抖動（Jitter）以防止驚群效應
- 可配置重試次數和退避參數

**`src/core/token_bucket.rs`**
- 令牌桶（Token Bucket）限流實現
- 用於速率限制和並發控制
- 支持動態調整速率

**`src/core/error.rs`**
- 核心錯誤類型定義
- 使用 `thiserror` 進行錯誤處理

#### 2.4 負載平衡策略 (`src/policies/`)

**`src/policies/mod.rs`**
- 策略模塊入口
- 定義策略特徵（Trait）

**`src/policies/registry.rs`**
- 策略註冊表（Policy Registry）
- 管理所有可用的負載平衡策略
- 策略的創建和查詢

**`src/policies/factory.rs`**
- 策略工廠模式實現
- 根據配置創建對應的策略實例

**`src/policies/round_robin.rs`**
- **輪詢策略（Round Robin）**
- 按順序輪流分配請求到各個 worker
- 簡單且公平的分配方式
- 適用於一般用途

**`src/policies/random.rs`**
- **隨機策略（Random）**
- 均勻隨機選擇 worker
- 適用於簡單部署場景

**`src/policies/consistent_hash.rs`**
- **一致性哈希策略（Consistent Hashing）**
- 將相同的會話/用戶路由到相同的 worker
- 支持會話親和性（Session Affinity）
- 適用於多輪對話、KV 緩存重用場景
- 使用虛擬節點（Virtual Nodes）提高分佈均勻性

**`src/policies/power_of_two.rs`**
- **Power of Two 策略**
- 隨機選擇兩個 worker，選擇負載較低的一個
- 適用於對負載敏感的場景
- 平衡隨機性和負載感知

**`src/policies/cache_aware.rs`**
- **緩存感知策略（Cache Aware）**
- 優化前綴緩存命中率
- 使用緩存樹（Cache Tree）結構
- 適用於重複提示詞、少樣本學習場景
- 可配置緩存閾值、平衡閾值

#### 2.5 路由器實現 (`src/routers/`)

**`src/routers/mod.rs`**
- 路由器模塊入口
- 定義路由器特徵（`RouterTrait`）

**`src/routers/router_manager.rs`**
- 路由器管理器（Router Manager）
- 管理多個路由器實例
- 支持路由器 ID 標識

**`src/routers/factory.rs`**
- 路由器工廠
- 根據配置創建對應的路由器實例

**`src/routers/header_utils.rs`**
- HTTP 頭處理工具函數
- 提取請求 ID、會話 ID 等
- 支持自定義請求 ID 頭

**`src/routers/http/`**

- **`mod.rs`**: HTTP 路由器模塊入口

- **`router.rs`**: 
  - 基礎 HTTP 路由器實現
  - 處理標準 HTTP 請求轉發
  - 支持各種 OpenAI 兼容的 API 端點

- **`pd_router.rs`**: 
  - Prefill/Decode 分離路由實現
  - 將請求分為 prefill 和 decode 階段
  - 分別路由到不同的 worker 組

- **`vllm_pd_router.rs`**: 
  - vLLM 專用的 PD 路由實現
  - 支持 vLLM 特定的兩階段處理
  - 處理 vLLM 的特定協議

- **`openai_router.rs`**: 
  - OpenAI 兼容的路由器
  - 支持 OpenAI API 格式的請求
  - 可以路由到多個 OpenAI 兼容的後端

- **`logprobs_merge.rs`**: 
  - Logprobs 合併邏輯
  - 在多 worker 場景下合併 logprobs 結果

- **`dp_utils.rs`**: 
  - 數據並行（Data Parallel）工具函數
  - 支持 intra-node data parallel 模式

- **`vllm_service_discovery.rs`**: 
  - vLLM 服務發現實現
  - 處理 ZMQ 服務發現協議

**`src/routers/grpc/`**

- **`mod.rs`**: gRPC 路由器模塊入口

- **`router.rs`**: 
  - gRPC 路由器實現
  - 處理 gRPC 協議的請求轉發

- **`pd_router.rs`**: 
  - gRPC 協議的 PD 路由實現

#### 2.6 路由樹 (`src/routes/`)

**`src/routes/mod.rs`**
- 路由樹模塊入口

**`src/routes/interface.rs`**
- 路由接口定義
- 定義路由行為的抽象接口

**`src/routes/routing_tree_builder.rs`**
- 路由樹構建器
- 根據配置構建路由樹結構
- 將 URL 路徑映射到處理器

**`src/routes/single_server_route.rs`**
- 單服務器路由實現
- 用於單 worker 場景

**`src/routes/round_robin_route.rs`**
- 輪詢路由實現

**`src/routes/pool_route.rs`**
- 連接池路由實現
- 管理 worker 連接池

**`src/routes/prefill_decode_route.rs`**
- Prefill/Decode 路由實現
- 處理兩階段路由邏輯

#### 2.7 服務器 (`src/server.rs`)

**`src/server.rs`**
- HTTP 服務器的核心實現
- 使用 Axum 框架構建
- 定義應用上下文（`AppContext`）：
  - HTTP 客戶端
  - 路由器配置
  - 速率限制器
  - Tokenizer
  - Worker 註冊表
  - 策略註冊表
  - 響應存儲
- 實現主要 API 端點：
  - `/v1/chat/completions`: 聊天完成
  - `/v1/completions`: 文本完成
  - `/v1/embeddings`: 嵌入向量
  - `/v1/models`: 模型列表
  - `/health`: 健康檢查
  - `/metrics`: Prometheus 指標
- 服務器啟動邏輯（`startup` 函數）

#### 2.8 服務發現 (`src/service_discovery.rs`)

**`src/service_discovery.rs`**
- Kubernetes 服務發現實現
- 監聽 Kubernetes API 以發現 worker pod
- 自動添加和移除 worker
- 支持標籤選擇器（Label Selector）
- 支持命名空間過濾
- 支持 PD 模式的 prefill 和 decode worker 分別發現

#### 2.9 協議定義 (`src/protocols/`)

**`src/protocols/mod.rs`**
- 協議模塊入口

**`src/protocols/spec.rs`**
- 協議規格定義
- 定義各種請求類型：
  - `ChatCompletionRequest`: 聊天完成請求
  - `CompletionRequest`: 文本完成請求
  - `EmbeddingRequest`: 嵌入請求
  - `GenerateRequest`: 生成請求
  - `RerankRequest`: 重排序請求
  - `ResponsesRequest`: 響應請求
- 使用 Serde 進行序列化/反序列化

**`src/protocols/worker_spec.rs`**
- Worker API 規格定義
- `WorkerApiResponse`: Worker API 響應
- `WorkerConfigRequest`: Worker 配置請求
- `WorkerErrorResponse`: Worker 錯誤響應

**`src/protocols/validation.rs`**
- 請求驗證邏輯
- 驗證請求參數的合法性

#### 2.10 中間件 (`src/middleware.rs`)

**`src/middleware.rs`**
- HTTP 中間件實現
- 使用 Tower 中間件框架
- 包含的中間件：
  - 請求超時（Timeout）
  - 速率限制（Rate Limiting）
  - CORS（跨域資源共享）
  - 請求 ID 追踪
  - 壓縮（Gzip）
  - 日誌記錄
- 定義 `QueuedRequest` 類型用於請求隊列

#### 2.11 Tokenizer (`src/tokenizer/`)

**`src/tokenizer/mod.rs`**
- Tokenizer 模塊入口

**`src/tokenizer/traits.rs`**
- Tokenizer 特徵（Trait）定義
- 定義標準化的 tokenizer 接口

**`src/tokenizer/factory.rs`**
- Tokenizer 工廠
- 根據模型路徑創建對應的 tokenizer
- 支持 HuggingFace 和 TikToken

**`src/tokenizer/huggingface.rs`**
- HuggingFace tokenizer 實現
- 使用 `tokenizers` crate

**`src/tokenizer/tiktoken.rs`**
- TikToken 實現（OpenAI 的 tokenizer）
- 使用 `tiktoken-rs` crate

**`src/tokenizer/chat_template.rs`**
- 聊天模板處理
- 支持 Jinja2 模板語法（使用 `minijinja`）
- 處理對話格式轉換

**`src/tokenizer/sequence.rs`**
- Token 序列處理
- 處理停止序列、生成參數等

**`src/tokenizer/stream.rs`**
- 流式處理支持
- Server-Sent Events (SSE) 格式

**`src/tokenizer/stop.rs`**
- 停止條件處理
- 檢查停止 token 和停止序列

**`src/tokenizer/hub.rs`**
- HuggingFace Hub 集成
- 從 HuggingFace Hub 下載模型和 tokenizer

**`src/tokenizer/mock.rs`**
- Mock tokenizer（用於測試）

**`src/tokenizer/tests.rs`**
- Tokenizer 單元測試

**`src/tokenizer/README.md`**
- Tokenizer 模塊文檔

#### 2.12 數據連接器 (`src/data_connector/`)

**`src/data_connector/mod.rs`**
- 數據連接器模塊入口

**`src/data_connector/responses.rs`**
- 響應類型定義
- 定義各種 API 響應格式

**`src/data_connector/response_memory_store.rs`**
- 內存響應存儲實現
- 用於保存請求/響應歷史（History Backend）
- 支持多輪對話上下文

**`src/data_connector/response_noop_store.rs`**
- 無操作存儲實現
- 不保存任何歷史（禁用 History Backend）

#### 2.13 其他核心文件

**`src/handler.rs`**
- HTTP 請求處理器
- 處理各種 API 端點的邏輯
- 調用路由器進行請求轉發

**`src/metrics.rs`**
- Prometheus 指標實現
- 使用 `metrics` 和 `metrics-exporter-prometheus` crate
- 定義各種指標（請求數、延遲、錯誤率等）
- Prometheus 配置（`PrometheusConfig`）

**`src/logging.rs` / `src/logger.rs`**
- 日誌系統實現
- 使用 `tracing` 框架
- 支持結構化日誌
- 支持日誌文件輪轉
- 日誌配置（`LoggingConfig`）

**`src/tree.rs`**
- 緩存樹數據結構
- 用於 cache-aware 策略的前綴匹配
- 高效的字符串前綴查找

**`src/grpc/client.rs` / `src/grpc/mod.rs`**
- gRPC 客戶端實現
- 使用 Tonic 框架
- 處理 gRPC 協議的通信

**`src/proto/vllm_scheduler.proto`**
- Protocol Buffers 定義文件
- 定義 vLLM 調度器的 gRPC 接口

**`src/types.rs`**
- 通用類型定義
- 項目中共享的類型

**`src/utils/`**
- **`mod.rs`**: 工具模塊入口
- **`json.rs`**: JSON 處理工具函數

---

### 三、Python 包 (`py_src/vllm_router/`)

#### 3.1 包結構

**`py_src/vllm_router/__init__.py`**
- Python 包的初始化文件
- 導出公開 API

**`py_src/vllm_router/launch_router.py`**
- Python 命令行啟動工具
- 定義 `main()` 函數作為入口點
- 解析命令行參數
- 調用 Rust 綁定的 Router 啟動服務
- 提供自定義幫助格式化器

**`py_src/vllm_router/router.py`**
- Router 類的 Python 封裝
- 封裝 Rust 的 Router 類
- 提供 Python 友好的接口
- `Router.from_args()`: 從參數創建 Router 實例
- `router.start()`: 啟動路由器

**`py_src/vllm_router/router_args.py`**
- 參數解析和驗證
- `RouterArgs` 類：封裝所有路由參數
- `add_cli_args()`: 向 argparse 添加命令行參數
- `from_cli_args()`: 從命令行參數創建 RouterArgs
- `_validate_router_args()`: 驗證參數合法性

**`py_src/vllm_router/mini_lb.py`**
- 簡易負載均衡器（Mini Load Balancer）
- 用於調試和開發
- 純 Python 實現，不依賴 Rust 組件

**`py_src/vllm_router/version.py`**
- 版本信息定義

---

### 四、測試 (`tests/` 和 `py_test/`)

#### 4.1 Rust 測試 (`tests/`)

**`tests/common/`**
- **`mod.rs`**: 測試通用模塊入口
- **`mock_openai_server.rs`**: Mock OpenAI 服務器（用於測試）
- **`mock_worker.rs`**: Mock Worker（用於測試）
- **`test_app.rs`**: 測試應用程序設置

**`tests/api_endpoints_test.rs`**
- API 端點測試
- 測試各種 HTTP 端點的功能

**`tests/request_formats_test.rs`**
- 請求格式測試
- 測試不同請求格式的處理

**`tests/responses_api_test.rs`**
- 響應 API 測試

**`tests/streaming_tests.rs`**
- 流式處理測試
- 測試 Server-Sent Events (SSE) 流式響應

**`tests/test_chat_template.rs` / `tests/test_chat_template_loading.rs`**
- 聊天模板測試

**`tests/test_consistent_hash_policy.rs`**
- 一致性哈希策略測試

**`tests/test_openai_routing.rs`**
- OpenAI 路由測試

**`tests/test_pd_routing.rs`**
- Prefill/Decode 路由測試

**`tests/test_prompt_input.rs`**
- 提示詞輸入測試

**`tests/tokenizer_integration.rs`**
- Tokenizer 集成測試

**`tests/benchmark_integration.rs`**
- 基準測試集成

**`tests/cache_aware_backward_compat_test.rs`**
- 緩存感知策略向後兼容性測試

**`tests/policy_registry_integration.rs`**
- 策略註冊表集成測試

#### 4.2 Python 測試 (`py_test/`)

**`py_test/__init__.py`**
- Python 測試包初始化

**`py_test/conftest.py`**
- pytest 配置文件
- 定義測試夾具（fixtures）

**`py_test/run_suite.py`**
- 測試套件運行腳本

**`py_test/unit/`**
- **`__init__.py`**: 單元測試包初始化
- **`test_arg_parser.py`**: 參數解析器測試
- **`test_router_config.py`**: 路由器配置測試
- **`test_startup_sequence.py`**: 啟動序列測試
- **`test_validation.py`**: 驗證邏輯測試

**`py_test/integration/`**
- **`__init__.py`**: 集成測試包初始化
- **`conftest.py`**: 集成測試配置文件
- **`test_api_auth.py`**: API 認證測試
- **`test_circuit_breaker.py`**: 熔斷器測試
- **`test_fault_tolerance.py`**: 容錯測試
- **`test_payload_size.py`**: 負載大小測試
- **`test_pd_routing.py`**: PD 路由測試
- **`test_rate_limiting.py`**: 速率限制測試
- **`test_retries.py`**: 重試機制測試
- **`test_service_discovery_shim.py`**: 服務發現測試
- **`test_worker_management.py`**: Worker 管理測試
- **`load_balancing/`**:
  - **`test_round_robin.py`**: 輪詢策略測試
  - **`test_random.py`**: 隨機策略測試
  - **`test_power_of_two.py`**: Power of Two 策略測試
  - **`test_cache_aware.py`**: 緩存感知策略測試

**`py_test/e2e/`**
- **`__init__.py`**: 端到端測試包初始化
- **`conftest.py`**: E2E 測試配置文件
- **`test_regular_router.py`**: 常規路由器 E2E 測試
- **`test_pd_router.py`**: PD 路由器 E2E 測試
- **`test_e2e_embeddings.py`**: 嵌入 API E2E 測試
- **`pd_disagg_vllm/`**: PD 分離 vLLM 相關測試
  - **`test_pd_accuracy.py`**: PD 準確性測試
  - **`test_lm_eval_accuracy.py`**: LM Eval 準確性測試

**`py_test/fixtures/`**
- **`__init__.py`**: 測試夾具包初始化
- **`mock_worker.py`**: Mock Worker 實現
- **`router_manager.py`**: 路由器管理器夾具
- **`ports.py`**: 端口管理工具

**`py_test/test_consistent_hash_policy.py`**
- 一致性哈希策略測試

**`py_test/test_launch_router.py`**
- 路由器啟動測試

---

### 五、基準測試 (`benches/`)

**`benches/request_processing.rs`**
- 請求處理性能基準測試
- 測量路由器處理請求的延遲和吞吐量

**`benches/tokenizer_benchmark.rs`**
- Tokenizer 性能基準測試
- 測量 tokenizer 的編碼/解碼速度

---

### 六、示例和配置 (`examples/`)

**`examples/configs/`**
- **`single_server_config.json`**: 單服務器配置示例
- **`round_robin_config.json`**: 輪詢策略配置示例

---

### 七、文檔 (`docs/`)

**`docs/load_balancing/README.md`**
- 負載平衡策略詳細文檔
- 說明各種策略的適用場景和配置選項

---

### 八、腳本 (`scripts/`)

#### 8.1 安裝和部署腳本

**`scripts/install.sh`**
- 安裝腳本
- 安裝系統依賴和配置環境

**`scripts/install_aws_ofi_nccl.sh`**
- 安裝 AWS OFI NCCL（用於 AWS 環境）

**`scripts/setup-sccache.sh`**
- 設置 sccache（Rust 編譯緩存）

**`scripts/vllm_router_install.sh`**
- vLLM Router 安裝腳本

#### 8.2 Kubernetes 部署配置 (`scripts/k8s/`)

**`scripts/k8s/llama3/`**
- Llama 3 模型的 K8s 部署配置
- 包含原生部署和 PD 分離部署配置

**`scripts/k8s/llama3.1/`**
- Llama 3.1 模型的部署配置
- 包含負載平衡方法比較文檔

**`scripts/k8s/deepseek-v31/`**
- DeepSeek-V3.1 模型的部署配置

**`scripts/k8s/pd_disagg/`**
- PD 分離架構的通用部署配置

#### 8.3 其他工具腳本

**`scripts/run_benchmarks.py`**
- 運行基準測試的腳本

**`scripts/post_benchmark_comment.py`**
- 基準測試結果後處理腳本

**`scripts/deepseek/`**
- DeepSeek 模型相關的部署腳本和日誌

**`scripts/llama3.1/`**
- Llama 3.1 模型的啟動腳本
  - `start_router.sh`: 啟動路由器
  - `start_prefill.sh`: 啟動 Prefill worker
  - `start_decode.sh`: 啟動 Decode worker

---

## 核心架構流程

### 1. 啟動流程

```
命令行參數解析 (main.rs)
    ↓
構建 RouterConfig
    ↓
驗證配置
    ↓
創建 AppContext
    ↓
初始化組件:
  - Worker Registry
  - Policy Registry
  - Tokenizer (如需要)
  - Response Storage
    ↓
啟動服務發現 (如啟用)
    ↓
啟動 Prometheus 指標服務器
    ↓
啟動 HTTP 服務器 (Axum)
    ↓
註冊路由端點
    ↓
開始監聽請求
```

### 2. 請求處理流程

```
HTTP 請求到達
    ↓
中間件處理:
  - CORS
  - 請求 ID 追踪
  - 速率限制
  - 超時控制
    ↓
路由匹配 (根據路徑)
    ↓
解析請求體 (JSON)
    ↓
驗證請求參數
    ↓
選擇負載平衡策略
    ↓
從 Worker Registry 選擇 Worker
    ↓
檢查熔斷器狀態
    ↓
轉發請求到 Worker (HTTP/gRPC)
    ↓
接收響應
    ↓
處理重試 (如失敗)
    ↓
更新指標
    ↓
返回響應給客戶端
```

### 3. Prefill/Decode 分離流程

```
請求到達
    ↓
識別為兩階段請求
    ↓
階段 1: Prefill
  - 選擇 Prefill Worker
  - 發送 Prefill 請求
  - 接收 Prefill 響應
  - 提取 KV Cache 信息
    ↓
階段 2: Decode
  - 選擇 Decode Worker
  - 發送 Decode 請求（包含 KV Cache 引用）
  - 接收 Decode 響應（流式或非流式）
    ↓
合併響應
    ↓
返回給客戶端
```

### 4. 服務發現流程

```
啟動服務發現
    ↓
連接到 Kubernetes API
    ↓
監聽 Pod 變化事件
    ↓
根據 Label Selector 過濾 Pod
    ↓
提取 Pod IP 和 Port
    ↓
創建 Worker 實例
    ↓
添加到 Worker Registry
    ↓
定期健康檢查
    ↓
更新 Worker 狀態
```

---

## 關鍵技術決策

### 1. 為什麼選擇 Rust 而不是 C++/Python？

這是一個常見且重要的問題。雖然 C++ 和 Python 更普及，但 Rust 在這個項目中提供了更好的平衡。以下是詳細的技術分析：

#### 1.1 Rust vs Python

**為什麼不用 Python？**

雖然 Python 非常普及且易於使用，但對於這個路由器項目來說，Python 有以下局限：

❌ **性能瓶頸**
- Python 的 GIL（全局解釋器鎖）限制真正的並發
- 解釋器開銷：即使使用 asyncio，每個請求的處理成本仍然較高
- 內存開銷：Python 對象的開銷比 Rust 更大
- 對於高吞吐量的路由場景，Python 可能成為瓶頸

❌ **延遲敏感場景**
- vLLM Router 需要處理大量並發請求（可能每秒數萬個）
- Python 的垃圾回收（GC）可能導致不可預測的延遲
- 在高頻路由場景下，Python 的延遲變異性更大

✅ **為什麼仍提供 Python 綁定？**
- 項目確實提供了 PyO3 Python 綁定（`py_src/`）
- 用戶可以使用 `vllm-router` Python 命令來啟動路由器
- 這樣既保留了 Python 的易用性，又獲得了 Rust 的性能

#### 1.2 Rust vs C++

**為什麼不用 C++？**

雖然 C++ 性能極佳且更普及，但 Rust 在這個項目中提供了更多優勢：

✅ **內存安全（無需手動管理）**
- **無需擔心內存洩漏**：Rust 的所有權系統在編譯時就防止了常見的內存錯誤
- **無需手動釋放資源**：RAII（資源獲取即初始化）自動處理，無需 `delete`、`free`
- **防止數據競爭**：編譯器保證線程安全，無需手動加鎖
- 對於路由器這種需要 7x24 運行的基礎設施，減少崩潰和內存洩漏至關重要

✅ **更安全的並發編程**
- Rust 的類型系統在編譯時就防止了數據競爭
- 無需像 C++ 那樣手動管理互斥鎖、條件變量
- Tokio 異步運行時提供了零成本抽象（Zero-Cost Abstractions）
- 代碼示例：
  ```rust
  // Rust: 編譯器保證線程安全
  let data = Arc::new(Mutex::new(Vec::new()));
  
  // C++: 需要手動確保所有訪問都加鎖，容易出錯
  // std::mutex mtx;
  // std::lock_guard<std::mutex> lock(mtx);
  ```

✅ **現代化的工具鏈**
- **Cargo 包管理器**：統一的構建和依賴管理工具，不需要配置 CMake/Conan 等
- **依賴管理**：`Cargo.toml` 清晰定義所有依賴，版本管理更簡單
- **編譯器錯誤信息**：Rust 編譯器提供非常友好的錯誤提示和修復建議
- **測試內建**：`cargo test` 內建測試支持，無需額外配置測試框架
- **統一的工具鏈**：一個 `cargo` 命令解決構建、測試、文檔、發佈等所有需求

⚠️ **關於開發效率的客觀說明**

這裡需要澄清：「開發效率」取決於多個因素，不能簡單地說 Rust 比 C++ 更容易開發：

**Rust 在某些方面可能更高效：**
- ✅ **統一的工具鏈**：Cargo 統一管理構建、依賴、測試，不需要學習多種工具（CMake、Conan、GTest 等）
- ✅ **編譯器輔助**：Rust 編譯器會給出詳細的錯誤信息和修復建議，減少調試時間
- ✅ **減少運行時錯誤**：編譯時捕獲更多錯誤，減少在生產環境中發現問題
- ✅ **自動化工具**：`rustfmt`、`clippy` 等工具自動化代碼格式化和檢查

**C++ 在某些方面可能更高效：**
- ✅ **更廣泛的知識庫**：更多教程、Stack Overflow 答案、社區資源
- ✅ **更成熟的 IDE 支持**：Visual Studio、CLion 等 IDE 支持非常成熟
- ✅ **更多的範例代碼**：網上有更多的 C++ 範例可以參考
- ✅ **學習曲線**：對於已經熟悉 C++ 的開發者，使用 C++ 可能更快

**對這個項目來說：**
- 作為一個**新項目**，使用 Rust 可以從頭開始，不受舊代碼庫限制
- **統一的工具鏈**（Cargo）降低了項目設置的複雜度
- **編譯時安全檢查**減少了調試時間，特別是在並發和內存管理方面
- 但如果團隊已經熟悉 C++ 且有完善的構建系統，C++ 也是很好的選擇

**結論**：開發效率很大程度上取決於團隊經驗和項目需求，而不是絕對的語言優勢。對於這個項目，Rust 的工具鏈和安全性特性帶來了實際的開發效率提升。

✅ **與現有生態系統的兼容性**
- 項目已經使用了很多 Rust 生態系統的庫：
  - `tokio`: 異步運行時
  - `axum`: Web 框架
  - `tonic`: gRPC 框架
  - `serde`: 序列化框架
  - `kube`: Kubernetes 客戶端
- 這些庫都是高質量、經過生產驗證的
- **注意**：C++ 也有 Boost、Poco 等優秀庫，但 Rust 的庫對於這個項目的需求已經足夠

#### 1.3 Rust 在路由場景的具體優勢

**高並發處理**
```rust
// Rust + Tokio: 可以輕鬆處理數萬個並發連接
// 零成本異步抽象，性能接近手寫的 epoll/kqueue 代碼
async fn handle_request(request: Request) -> Response {
    // 異步處理，不會阻塞線程
}
```

**內存效率**
- Rust 編譯後是靜態二進制，無需 Python 解釋器
- 更小的內存佔用：適合容器化部署
- 可預測的內存使用：無 GC 暫停

**類型安全減少錯誤**
- 編譯時捕獲配置錯誤（如路由規則錯誤）
- 類型系統防止了許多運行時錯誤
- 在路由器這種基礎設施中，減少錯誤至關重要

#### 1.4 項目的實際選擇策略

這個項目採用了一個**混合策略**，兼顧性能和易用性：

1. **核心邏輯用 Rust**（`src/`）
   - 高性能路由邏輯
   - 負載平衡算法
   - 請求處理和轉發

2. **Python 綁定**（`py_src/`）
   - 提供 Python 接口方便使用
   - 與 vLLM 的 Python 生態系統集成
   - 可以通過 `pip install vllm-router` 安裝

3. **靈活的部署方式**
   - 可以作為純 Rust 二進制部署：`./target/release/vllm-router`
   - 也可以作為 Python 包部署：`vllm-router --worker-urls ...`
   - 用戶可以根據需求選擇

#### 1.5 性能數據參考

雖然項目中沒有明確的性能對比數據，但根據 Rust 社區的普遍經驗：

- **Rust vs Python**: Rust 通常快 10-100 倍（取決於場景）
- **Rust vs C++**: Rust 性能相當，但有時稍慢 5-10%（由於額外的安全檢查）
- **內存佔用**: Rust 通常比 Python 小得多，比 C++ 稍大（由於額外的元數據）

對於路由器這種 I/O 密集型場景，Rust 的性能優勢特別明顯。

#### 1.6 總結

選擇 Rust 的原因是：

1. ✅ **性能足夠好**：接近 C++ 的性能，遠超 Python
2. ✅ **安全性更好**：編譯時保證內存安全，減少生產環境的錯誤
3. ✅ **工具鏈統一現代**：Cargo 統一管理構建、依賴、測試，降低項目設置複雜度
4. ✅ **生態系統完善**：Tokio、Axum 等庫都非常成熟，適合這個項目的需求
5. ✅ **仍支持 Python**：通過 PyO3 綁定，保持 Python 的易用性

**關於開發效率的說明**：
- 開發效率取決於團隊經驗、項目需求和上下文
- Rust 的統一工具鏈和安全檢查對於**新項目**和**減少運行時錯誤**有幫助
- 但如果團隊已經有成熟的 C++ 工具鏈和經驗，C++ 也是很好的選擇
- 對於 vLLM Router 這個項目，Rust 的優勢主要在於**安全性**和**統一工具鏈**，而不僅僅是開發速度

對於 vLLM Router 這種需要**高性能、高可靠性、長期運行**的基礎設施項目，Rust 是一個理想的選擇。

---

### 2. PyO3 Python 綁定

### 2. PyO3 Python 綁定

- **易用性**: Python 生態系統的易用性
- **兼容性**: 與 vLLM 的 Python 代碼庫集成
- **部署靈活性**: 可以作為純 Rust 二進制或 Python 包部署

### 3. Axum Web 框架

- **現代化**: 基於 Tokio 的現代異步框架
- **類型安全**: 強類型系統減少錯誤
- **中間件**: Tower 中間件生態系統

### 4. 多種負載平衡策略

- **靈活性**: 不同場景需要不同策略
- **可擴展**: 易於添加新的策略
- **性能優化**: 針對不同工作負載優化

### 5. Prefill/Decode 分離

- **資源優化**: Prefill 和 Decode 階段對資源需求不同
- **可擴展性**: 獨立擴展 Prefill 和 Decode worker
- **成本效益**: 更好的資源利用率

---

## 擴展點

本節詳細說明如何擴展項目的核心功能。每個擴展點都包含完整的代碼示例和詳細的逐行註釋，適合新手理解。

---

### 1. 添加新的負載平衡策略

本節說明如何添加一個新的負載平衡策略（例如：最少連接策略 "Least Connections"）。

#### 步驟 1：在 `src/policies/` 創建新的策略文件

創建文件 `src/policies/least_connections.rs`：

```rust
//! 最少連接負載平衡策略
//! 這個策略會選擇當前連接數最少的 worker

// 導入必要的模塊和類型
use super::{get_healthy_worker_indices, LoadBalancingPolicy, RequestHeaders};
use crate::core::Worker;  // Worker trait，定義了 worker 的基本接口
use crate::metrics::RouterMetrics;  // 指標系統，用於記錄路由決策
use std::sync::Arc;  // Arc = Atomic Reference Counted，線程安全的引用計數指針

/// 最少連接策略結構體
/// 這個策略會追蹤每個 worker 的連接數，並選擇連接數最少的 worker
#[derive(Debug, Default)]
pub struct LeastConnectionsPolicy {
    // 存儲每個 worker URL 對應的當前連接數
    // HashMap<String, AtomicUsize> 表示：
    // - String: worker 的 URL（例如 "http://worker1:8000"）
    // - AtomicUsize: 該 worker 的當前連接數（原子類型，線程安全）
    // Arc<Mutex<...>> 用於在線程間共享和修改連接計數
    connection_counts: Arc<std::sync::Mutex<std::collections::HashMap<String, std::sync::atomic::AtomicUsize>>>,
}

impl LeastConnectionsPolicy {
    /// 創建一個新的最少連接策略實例
    pub fn new() -> Self {
        Self {
            // 初始化連接計數器為空 HashMap
            connection_counts: Arc::new(std::sync::Mutex::new(std::collections::HashMap::new())),
        }
    }
}

impl LoadBalancingPolicy for LeastConnectionsPolicy {
    /// 選擇一個 worker（實現核心邏輯）
    /// 
    /// 參數說明：
    /// - `workers`: 可用的 worker 列表（切片，每個元素是 Arc<dyn Worker>）
    /// - `request_text`: 可選的請求文本（有些策略需要這個來做決策，但我們不需要）
    /// - `headers`: 可選的 HTTP 請求頭（有些策略需要這個，但我們不需要）
    /// 
    /// 返回值：
    /// - `Option<usize>`: 選中的 worker 在 workers 列表中的索引，如果沒有健康的 worker 則返回 None
    fn select_worker_with_headers(
        &self,
        workers: &[Arc<dyn Worker>],  // 所有可用的 worker
        _request_text: Option<&str>,  // 下劃線前綴表示這個參數我們不使用
        _headers: Option<&RequestHeaders>,  // 同樣不使用這個參數
    ) -> Option<usize> {
        // 步驟 1: 獲取所有健康的 worker 的索引
        // get_healthy_worker_indices 會過濾掉不健康或熔斷器開路的 worker
        let healthy_indices = get_healthy_worker_indices(workers);

        // 如果沒有健康的 worker，返回 None（表示無法選擇）
        if healthy_indices.is_empty() {
            return None;
        }

        // 步驟 2: 獲取連接計數器的鎖
        // lock() 會阻塞直到獲取到鎖，確保線程安全
        let counts = self.connection_counts.lock().unwrap();

        // 步驟 3: 找到連接數最少的 worker
        let mut min_connections = std::usize::MAX;  // 初始化為最大值
        let mut selected_idx: Option<usize> = None;  // 將要選擇的 worker 索引

        // 遍歷所有健康的 worker
        for &idx in &healthy_indices {
            let worker = &workers[idx];  // 獲取 worker 的引用
            let worker_url = worker.url();  // 獲取 worker 的 URL 作為唯一標識

            // 從 HashMap 中獲取該 worker 的連接數
            // 如果這個 worker 還沒有被記錄過，默認為 0
            let connections = counts
                .get(&worker_url)  // 嘗試獲取該 URL 的計數器
                .map(|counter| counter.load(std::sync::atomic::Ordering::Relaxed))  // 讀取原子計數器的值
                .unwrap_or(0);  // 如果不存在，默認為 0

            // 如果這個 worker 的連接數更少，更新選中的 worker
            if connections < min_connections {
                min_connections = connections;
                selected_idx = Some(idx);
            }
        }

        // 步驟 4: 如果找到了 worker，增加它的連接數並記錄指標
        if let Some(idx) = selected_idx {
            let worker_url = workers[idx].url();

            // 在 HashMap 中確保這個 worker 有計數器，並增加連接數
            let counter = counts
                .entry(worker_url.clone())  // 獲取或插入條目
                .or_insert_with(|| std::sync::atomic::AtomicUsize::new(0));  // 如果不存在，創建一個新的計數器
            counter.fetch_add(1, std::sync::atomic::Ordering::Relaxed);  // 原子性地增加 1

            // 記錄指標（用於 Prometheus 監控）
            RouterMetrics::record_processed_request(&worker_url);  // 記錄處理的請求
            RouterMetrics::record_policy_decision(self.name(), &worker_url);  // 記錄策略決策

            // 釋放鎖（離開作用域時自動釋放）
        }

        // 返回選中的 worker 索引
        selected_idx
    }

    /// 當請求完成時調用，減少 worker 的連接數
    /// 
    /// 參數：
    /// - `worker_url`: 完成請求的 worker 的 URL
    /// - `success`: 請求是否成功（在這個策略中我們不關心成功與否，都要減少連接數）
    fn on_request_complete(&self, worker_url: &str, _success: bool) {
        // 獲取鎖
        let counts = self.connection_counts.lock().unwrap();

        // 如果這個 worker 存在於 HashMap 中，減少它的連接數
        if let Some(counter) = counts.get(worker_url) {
            // 原子性地減少 1（但不能小於 0）
            let current = counter.load(std::sync::atomic::Ordering::Relaxed);
            if current > 0 {
                counter.fetch_sub(1, std::sync::atomic::Ordering::Relaxed);
            }
        }
        // 鎖在這裡自動釋放
    }

    /// 返回策略的名稱（用於日誌和指標）
    fn name(&self) -> &'static str {
        "least_connections"  // 策略名稱，會被記錄到指標中
    }

    /// 實現類型擦除所需的 as_any 方法（用於向下轉型）
    fn as_any(&self) -> &dyn std::any::Any {
        self  // 返回自身的引用，可以轉換為具體類型
    }
}
```

#### 步驟 2：在 `src/policies/mod.rs` 中導出新策略

修改 `src/policies/mod.rs`：

```rust
// ... 其他導入 ...

// 添加新策略模塊的聲明（告訴 Rust 編譯器這個模塊存在）
mod least_connections;  // 這行聲明了一個名為 least_connections 的子模塊

// ... 其他模塊聲明 ...

// 添加新策略的導出（讓其他模塊可以使用這個策略）
pub use least_connections::LeastConnectionsPolicy;  // 這行讓外部代碼可以使用 LeastConnectionsPolicy

// ... 其他導出 ...
```

#### 步驟 3：在 `factory.rs` 中註冊策略

修改 `src/policies/factory.rs`：

```rust
// ... 其他導入 ...

// 導入新策略（從模塊中導入）
use super::{
    // ... 其他策略 ...
    LeastConnectionsPolicy,  // 添加這行來導入新策略
    LoadBalancingPolicy,
};

impl PolicyFactory {
    /// 從配置創建策略實例
    pub fn create_from_config(config: &PolicyConfig) -> Arc<dyn LoadBalancingPolicy> {
        match config {
            // ... 其他策略分支 ...
            
            // 添加新的分支來處理 LeastConnections 配置
            PolicyConfig::LeastConnections => {
                // 創建策略實例並包裝在 Arc 中（Arc 用於線程間共享）
                Arc::new(LeastConnectionsPolicy::new())
            }
            
            // ... 其他分支 ...
        }
    }

    /// 通過策略名稱創建策略實例
    pub fn create_by_name(name: &str) -> Option<Arc<dyn LoadBalancingPolicy>> {
        match name.to_lowercase().as_str() {  // 轉換為小寫以支持大小寫不敏感的匹配
            // ... 其他策略名稱 ...
            
            // 添加新策略名稱的匹配
            "least_connections" | "leastconnections" => {
                Some(Arc::new(LeastConnectionsPolicy::new()))
            }
            
            _ => None,  // 如果名稱不匹配任何策略，返回 None
        }
    }
}
```

#### 步驟 4：在 `types.rs` 中添加策略配置類型

修改 `src/config/types.rs`：

```rust
/// 負載平衡策略配置枚舉
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum PolicyConfig {
    // ... 其他策略變體 ...
    
    /// 最少連接策略
    /// 選擇當前連接數最少的 worker
    #[serde(rename = "least_connections")]
    LeastConnections,
    
    // ... 其他變體 ...
}
```

#### 步驟 5：在 `main.rs` 中支持命令行參數（可選）

修改 `src/main.rs`，在命令行參數解析中添加支持：

```rust
/// 負載平衡策略選擇
#[arg(long, default_value = "cache_aware", value_parser = [
    "random", 
    "round_robin", 
    "cache_aware", 
    "power_of_two", 
    "consistent_hash",
    "least_connections"  // 添加新策略名稱
])]
policy: String,

// ... 在 parse_policy 方法中添加 ...

fn parse_policy(&self, policy_str: &str) -> PolicyConfig {
    match policy_str {
        // ... 其他匹配 ...
        "least_connections" => PolicyConfig::LeastConnections,  // 添加這行
        _ => PolicyConfig::RoundRobin,  // 默認策略
    }
}
```

---

### 2. 添加新的 API 端點

本節說明如何添加一個新的 API 端點（例如：`/v1/custom/action`）。

#### 步驟 1：在 `src/protocols/spec.rs` 中定義請求/響應類型

修改 `src/protocols/spec.rs`，添加新的請求/響應類型：

```rust
// ... 其他導入和類型定義 ...

/// 自定義動作請求
/// 這是一個示例請求類型，用於展示如何定義新的 API 請求
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CustomActionRequest {
    /// 動作類型（例如："analyze", "process", "transform"）
    /// String 類型表示動作的名稱
    pub action: String,
    
    /// 要處理的數據
    /// Option<String> 表示這個字段是可選的
    #[serde(skip_serializing_if = "Option::is_none")]  // 如果是 None，序列化時跳過
    pub data: Option<String>,
    
    /// 可選的參數映射
    /// HashMap 用於存儲鍵值對參數
    #[serde(skip_serializing_if = "Option::is_none")]
    pub parameters: Option<HashMap<String, Value>>,  // Value 是 serde_json::Value，可以表示任何 JSON 值
}

/// 自定義動作響應
/// 響應類型包含操作結果和可選的元數據
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CustomActionResponse {
    /// 操作是否成功
    /// bool 類型：true 表示成功，false 表示失敗
    pub success: bool,
    
    /// 結果數據
    /// String 類型：操作產生的結果
    pub result: String,
    
    /// 可選的錯誤信息
    /// 只有在 success=false 時才會有值
    #[serde(skip_serializing_if = "Option::is_none")]
    pub error: Option<String>,
    
    /// 元數據（用於傳遞額外信息）
    #[serde(skip_serializing_if = "Option::is_none")]
    pub metadata: Option<HashMap<String, Value>>,
}
```

#### 步驟 2：在 `src/server.rs` 中添加路由處理函數

修改 `src/server.rs`，添加處理函數：

```rust
// ... 其他導入 ...

// 導入新定義的請求/響應類型
use crate::protocols::spec::{CustomActionRequest, CustomActionResponse};

// ... 其他處理函數 ...

/// 處理自定義動作請求
/// 
/// 這個函數的簽名說明：
/// - `State(state)`: 從 Axum 框架中提取應用狀態（AppState）
/// - `headers: http::HeaderMap`: HTTP 請求頭（用於提取認證、會話 ID 等）
/// - `Json(body)`: 自動解析的 JSON 請求體（類型為 CustomActionRequest）
/// - 返回值：`Response` 是 Axum 的 HTTP 響應類型
async fn v1_custom_action(
    State(state): State<Arc<AppState>>,  // 獲取共享的應用狀態
    headers: http::HeaderMap,  // 獲取 HTTP 請求頭
    Json(body): Json<CustomActionRequest>,  // 自動將 JSON 請求體解析為 CustomActionRequest
) -> Response {
    // 調用路由器的處理方法
    // state.router 是實現了 RouterTrait 的路由器實例
    // route_custom_action 是我們需要在 RouterTrait 中定義的方法
    // Some(&headers) 傳遞請求頭（Some 表示有值）
    // &body 傳遞請求體（引用，避免所有權轉移）
    // None 表示沒有額外的流式參數
    state.router.route_custom_action(Some(&headers), &body, None).await
}
```

#### 步驟 3：在 `src/server.rs` 的 `build_app` 函數中註冊路由

修改 `src/server.rs` 的 `build_app` 函數：

```rust
pub fn build_app(
    app_state: Arc<AppState>,
    max_payload_size: usize,
    request_id_headers: Vec<String>,
    cors_allowed_origins: Vec<String>,
) -> Router {
    // 創建需要認證的路由（protected routes）
    let protected_routes = Router::new()
        // ... 其他路由 ...
        
        // 添加新路由
        // .route(path, method) 定義路由：
        // - "/v1/custom/action" 是 URL 路徑
        // - post(...) 表示這個路由只接受 POST 請求
        // - v1_custom_action 是處理函數（上面定義的）
        .route("/v1/custom/action", post(v1_custom_action))
        
        // ... 其他路由 ...
        
        // 應用認證中間件（確保這些路由需要認證）
        .route_layer(axum::middleware::from_fn_with_state(
            app_state.clone(),  // 傳遞應用狀態給中間件
            middleware::auth_middleware,  // 認證中間件函數
        ));
    
    // ... 其他路由組的定義 ...
    
    // 合併所有路由
    Router::new()
        .merge(protected_routes)  // 合併需要認證的路由
        // ... 合併其他路由組 ...
        .with_state(app_state)  // 將應用狀態注入到所有路由中
}
```

#### 步驟 4：在 `src/routers/traits.rs` 或 `src/routers/mod.rs` 中定義 RouterTrait 方法

找到 `RouterTrait` 的定義（通常在 `src/routers/mod.rs` 或單獨的 traits 文件），添加新方法：

```rust
/// 路由器特徵（Trait）定義
/// 所有路由器都必須實現這個特徵
#[async_trait]
pub trait RouterTrait: Send + Sync {
    // ... 其他方法定義 ...
    
    /// 處理自定義動作請求
    /// 
    /// 參數說明：
    /// - `headers`: 可選的 HTTP 請求頭（用於提取會話 ID、認證信息等）
    /// - `request`: 自定義動作請求體（引用，不獲取所有權）
    /// - `stream`: 可選的流式參數（這裡不使用，傳 None）
    /// 
    /// 返回值：
    /// - `Response`: HTTP 響應（包含狀態碼、頭、響應體）
    async fn route_custom_action(
        &self,
        headers: Option<&http::HeaderMap>,  // 可選的請求頭
        request: &CustomActionRequest,  // 請求體的引用
        _stream: Option<()>,  // 流式參數（不使用，用下劃線前綴）
    ) -> Response;
}
```

#### 步驟 5：在路由器實現中添加具體邏輯

修改 `src/routers/http/router.rs`（或其他路由器實現文件），實現新方法：

```rust
#[async_trait]
impl RouterTrait for Router {
    // ... 其他方法實現 ...
    
    /// 實現自定義動作路由邏輯
    async fn route_custom_action(
        &self,
        headers: Option<&http::HeaderMap>,
        request: &CustomActionRequest,
        _stream: Option<()>,
    ) -> Response {
        // 步驟 1: 提取請求頭中的會話 ID（如果需要的話）
        let session_id = headers
            .and_then(|h| h.get("x-session-id"))  // 嘗試獲取 x-session-id 頭
            .and_then(|v| v.to_str().ok())  // 將 HeaderValue 轉換為字符串
            .map(|s| s.to_string());  // 轉換為 String

        // 步驟 2: 從 worker registry 中選擇一個 worker
        // 獲取所有可用的 worker（只包括健康的）
        let workers: Vec<Arc<dyn Worker>> = self.worker_registry.get_all_workers()
            .into_iter()
            .filter(|w| w.is_healthy() && w.circuit_breaker().can_execute())  // 過濾出健康的 worker
            .collect();

        // 檢查是否有可用的 worker
        if workers.is_empty() {
            // 沒有可用的 worker，返回 503 Service Unavailable
            return (
                StatusCode::SERVICE_UNAVAILABLE,
                Json(CustomActionResponse {
                    success: false,
                    result: String::new(),
                    error: Some("No healthy workers available".to_string()),
                    metadata: None,
                }),
            )
                .into_response();
        }

        // 步驟 3: 使用負載平衡策略選擇 worker
        let policy = self.policy_registry.get_policy();  // 獲取當前使用的策略
        let selected_idx = policy.select_worker(&workers, None);  // 選擇一個 worker

        // 檢查是否成功選擇了 worker
        let selected_worker = match selected_idx {
            Some(idx) => &workers[idx],  // 使用選中的 worker
            None => {
                // 策略沒有選擇任何 worker，返回錯誤
                return (
                    StatusCode::SERVICE_UNAVAILABLE,
                    Json(CustomActionResponse {
                        success: false,
                        result: String::new(),
                        error: Some("Failed to select worker".to_string()),
                        metadata: None,
                    }),
                )
                    .into_response();
            }
        };

        // 步驟 4: 構建轉發到 worker 的請求
        // 將請求序列化為 JSON
        let request_body = serde_json::to_string(request)
            .map_err(|e| {
                // 如果序列化失敗，返回錯誤響應
                (
                    StatusCode::BAD_REQUEST,
                    Json(CustomActionResponse {
                        success: false,
                        result: String::new(),
                        error: Some(format!("Invalid request: {}", e)),
                        metadata: None,
                    }),
                )
                    .into_response()
            })?;

        // 步驟 5: 發送請求到選中的 worker
        // 構建完整的 URL（worker URL + 路徑）
        let worker_url = format!("{}/v1/custom/action", selected_worker.url());

        // 創建 HTTP 請求
        let mut http_request = self.client
            .post(&worker_url)  // POST 請求
            .header("Content-Type", "application/json")  // 設置 Content-Type 頭
            .body(request_body);  // 設置請求體

        // 如果配置了 API Key，添加到請求頭
        if let Some(ref api_key) = self.api_key {
            http_request = http_request.header("Authorization", format!("Bearer {}", api_key));
        }

        // 發送請求並等待響應
        let response = http_request.send().await;

        // 步驟 6: 處理響應
        match response {
            Ok(resp) => {
                // 請求成功，解析響應
                let status = resp.status();  // 獲取 HTTP 狀態碼
                let body = resp.text().await.unwrap_or_default();  // 讀取響應體

                // 嘗試將響應體解析為 CustomActionResponse
                let action_response: Result<CustomActionResponse, _> =
                    serde_json::from_str(&body);

                match action_response {
                    Ok(mut response) => {
                        // 記錄成功的請求
                        policy.on_request_complete(selected_worker.url(), true);

                        // 構建 HTTP 響應
                        (status, Json(response)).into_response()
                    }
                    Err(_) => {
                        // 響應體解析失敗，返回錯誤
                        (
                            StatusCode::INTERNAL_SERVER_ERROR,
                            Json(CustomActionResponse {
                                success: false,
                                result: String::new(),
                                error: Some("Failed to parse worker response".to_string()),
                                metadata: None,
                            }),
                        )
                            .into_response()
                    }
                }
            }
            Err(e) => {
                // 請求失敗（網絡錯誤、超時等）
                policy.on_request_complete(selected_worker.url(), false);

                // 返回錯誤響應
                (
                    StatusCode::BAD_GATEWAY,
                    Json(CustomActionResponse {
                        success: false,
                        result: String::new(),
                        error: Some(format!("Worker request failed: {}", e)),
                        metadata: None,
                    }),
                )
                    .into_response()
            }
        }
    }
}
```

---

### 3. 添加新的後端支持

本節說明如何添加一個新的後端支持（例如：支持 TensorRT-LLM 後端）。

#### 步驟 1：在 `src/routers/http/` 創建新的路由器文件

創建文件 `src/routers/http/tensorrt_router.rs`：

```rust
//! TensorRT-LLM 路由器實現
//! 這個路由器處理與 TensorRT-LLM 後端的通信

use crate::config::types::RetryConfig;
use crate::core::{BasicWorker, CircuitBreakerConfig, HealthConfig, Worker, WorkerRegistry, WorkerType};
use crate::metrics::RouterMetrics;
use crate::policies::{LoadBalancingPolicy, PolicyRegistry};
use crate::protocols::spec::{ChatCompletionRequest, GenerationRequest};
use crate::routers::{RouterTrait, WorkerManagement};
use axum::{
    body::Body,
    extract::Request,
    http::{HeaderMap, StatusCode},
    response::{IntoResponse, Response},
    Json,
};
use reqwest::Client;
use std::sync::Arc;
use tracing::{debug, error, warn};

/// TensorRT-LLM 路由器
/// 這個結構體包含了路由器所需的所有組件
#[derive(Debug)]
pub struct TensorRTRouter {
    /// Worker 註冊表（管理所有 worker）
    worker_registry: Arc<WorkerRegistry>,
    
    /// 負載平衡策略註冊表
    policy_registry: Arc<PolicyRegistry>,
    
    /// HTTP 客戶端（用於發送請求到 worker）
    client: Client,
    
    /// API Key（用於認證，可選）
    api_key: Option<String>,
    
    /// 重試配置
    retry_config: RetryConfig,
    
    /// 熔斷器配置
    circuit_breaker_config: CircuitBreakerConfig,
}

impl TensorRTRouter {
    /// 創建一個新的 TensorRT-LLM 路由器
    /// 
    /// 參數：
    /// - `worker_urls`: Worker 的 URL 列表
    /// - `ctx`: 應用上下文（包含配置、客戶端等）
    /// 
    /// 返回值：
    /// - `Result<Self, String>`: 成功返回路由器實例，失敗返回錯誤信息
    pub async fn new(
        worker_urls: Vec<String>,  // Worker URL 列表（例如：["http://worker1:8000", ...]）
        ctx: &Arc<crate::server::AppContext>,  // 應用上下文引用
    ) -> Result<Self, String> {
        // 步驟 1: 更新指標中的活躍 worker 數量
        RouterMetrics::set_active_workers(worker_urls.len());

        // 步驟 2: 轉換配置中的熔斷器配置為核心類型
        let circuit_breaker_config = CircuitBreakerConfig {
            failure_threshold: ctx.router_config.circuit_breaker.failure_threshold,
            success_threshold: ctx.router_config.circuit_breaker.success_threshold,
            timeout_duration: std::time::Duration::from_secs(
                ctx.router_config.circuit_breaker.timeout_duration_secs,
            ),
            window_duration: std::time::Duration::from_secs(
                ctx.router_config.circuit_breaker.window_duration_secs,
            ),
        };

        // 步驟 3: 創建 worker 註冊表
        let worker_registry = Arc::new(WorkerRegistry::new());

        // 步驟 4: 為每個 worker URL 創建 Worker 實例並註冊
        for url in &worker_urls {
            // 創建 BasicWorker 實例
            let worker = BasicWorker::new(url.clone(), WorkerType::Regular)
                .with_circuit_breaker_config(circuit_breaker_config.clone())  // 設置熔斷器配置
                .with_health_config(HealthConfig {
                    timeout_secs: ctx.router_config.health_check.timeout_secs,
                    check_interval_secs: ctx.router_config.health_check.check_interval_secs,
                    endpoint: ctx.router_config.health_check.endpoint.clone(),
                });

            // 將 worker 添加到註冊表
            worker_registry.add_worker(Arc::new(worker));
        }

        // 步驟 5: 創建策略註冊表（使用配置中的策略）
        let policy_registry = Arc::new(PolicyRegistry::new(ctx.router_config.policy.clone()));

        // 步驟 6: 返回路由器實例
        Ok(Self {
            worker_registry,
            policy_registry,
            client: ctx.client.clone(),  // 使用上下文中的 HTTP 客戶端
            api_key: ctx.router_config.api_key.clone(),  // 複製 API Key
            retry_config: ctx.router_config.retry.clone(),  // 複製重試配置
            circuit_breaker_config,
        })
    }

    /// TensorRT-LLM 特定的請求轉換函數
    /// 將 OpenAI 格式的請求轉換為 TensorRT-LLM 格式
    /// 
    /// 參數：
    /// - `request`: OpenAI 格式的請求
    /// 
    /// 返回值：
    /// - `serde_json::Value`: TensorRT-LLM 格式的 JSON 值
    fn convert_to_tensorrt_format(&self, request: &ChatCompletionRequest) -> serde_json::Value {
        // 構建 TensorRT-LLM 格式的請求
        // 這是一個示例，實際轉換邏輯取決於 TensorRT-LLM 的 API 規範
        serde_json::json!({
            "prompt": self.extract_prompt(request),  // 提取提示詞
            "max_tokens": request.max_tokens.unwrap_or(100),  // 最大 token 數
            "temperature": request.temperature.unwrap_or(0.7),  // 溫度參數
            "top_p": request.top_p.unwrap_or(1.0),  // top_p 參數
            // ... 其他參數的轉換 ...
        })
    }

    /// 從 ChatCompletionRequest 中提取提示詞
    fn extract_prompt(&self, request: &ChatCompletionRequest) -> String {
        // 將消息列表轉換為單一提示詞字符串
        // 這是簡化版本，實際實現可能需要更複雜的邏輯
        request
            .messages
            .iter()
            .map(|msg| format!("{}: {}", msg.role(), msg.content()))
            .collect::<Vec<_>>()
            .join("\n")
    }

    /// 將 TensorRT-LLM 格式的響應轉換為 OpenAI 格式
    fn convert_from_tensorrt_format(&self, tensorrt_response: &serde_json::Value) -> Response {
        // 解析 TensorRT-LLM 響應並轉換為 OpenAI 格式
        // 這是一個示例，實際轉換邏輯取決於 TensorRT-LLM 的響應格式
        
        // 提取響應中的文本
        let text = tensorrt_response
            .get("text")
            .and_then(|v| v.as_str())
            .unwrap_or("")
            .to_string();

        // 構建 OpenAI 格式的響應
        let openai_response = serde_json::json!({
            "id": "chatcmpl-tensorrt-123",  // 生成唯一 ID
            "object": "chat.completion",
            "created": chrono::Utc::now().timestamp(),  // 當前時間戳
            "model": "tensorrt-model",
            "choices": [{
                "index": 0,
                "message": {
                    "role": "assistant",
                    "content": text,
                },
                "finish_reason": "stop",
            }],
            "usage": {
                "prompt_tokens": 0,  // 實際應該從響應中提取
                "completion_tokens": 0,
                "total_tokens": 0,
            },
        });

        // 返回 HTTP 響應
        (StatusCode::OK, Json(openai_response)).into_response()
    }
}

/// 實現 RouterTrait（這是路由器必須實現的特徵）
#[async_trait::async_trait]
impl RouterTrait for TensorRTRouter {
    /// 處理聊天完成請求（實現核心路由邏輯）
    async fn route_chat(
        &self,
        headers: Option<&HeaderMap>,
        request: &ChatCompletionRequest,
        _stream: Option<()>,
    ) -> Response {
        // 步驟 1: 獲取所有健康的 worker
        let workers: Vec<Arc<dyn Worker>> = self.worker_registry.get_all_workers()
            .into_iter()
            .filter(|w| w.is_healthy() && w.circuit_breaker().can_execute())
            .collect();

        if workers.is_empty() {
            return (StatusCode::SERVICE_UNAVAILABLE, "No healthy workers available").into_response();
        }

        // 步驟 2: 使用負載平衡策略選擇 worker
        let policy = self.policy_registry.get_policy();
        let selected_idx = policy.select_worker(&workers, None);

        let selected_worker = match selected_idx {
            Some(idx) => &workers[idx],
            None => {
                return (StatusCode::SERVICE_UNAVAILABLE, "Failed to select worker").into_response();
            }
        };

        // 步驟 3: 轉換請求格式（從 OpenAI 格式轉換為 TensorRT-LLM 格式）
        let tensorrt_request = self.convert_to_tensorrt_format(request);

        // 步驟 4: 構建發送到 worker 的 HTTP 請求
        let worker_url = format!("{}/generate", selected_worker.url());  // TensorRT-LLM 的端點可能是 /generate

        let mut http_request = self.client
            .post(&worker_url)
            .header("Content-Type", "application/json")
            .json(&tensorrt_request);  // 自動序列化為 JSON

        // 添加認證頭（如果配置了 API Key）
        if let Some(ref api_key) = self.api_key {
            http_request = http_request.header("Authorization", format!("Bearer {}", api_key));
        }

        // 步驟 5: 發送請求並處理響應
        match http_request.send().await {
            Ok(resp) => {
                // 讀取響應體
                let body = resp.text().await.unwrap_or_default();
                
                // 解析 JSON 響應
                match serde_json::from_str::<serde_json::Value>(&body) {
                    Ok(tensorrt_response) => {
                        // 記錄成功的請求
                        policy.on_request_complete(selected_worker.url(), true);
                        
                        // 轉換響應格式（從 TensorRT-LLM 格式轉換為 OpenAI 格式）
                        self.convert_from_tensorrt_format(&tensorrt_response)
                    }
                    Err(e) => {
                        error!("Failed to parse TensorRT-LLM response: {}", e);
                        (StatusCode::INTERNAL_SERVER_ERROR, "Invalid response format").into_response()
                    }
                }
            }
            Err(e) => {
                error!("TensorRT-LLM worker request failed: {}", e);
                policy.on_request_complete(selected_worker.url(), false);
                (StatusCode::BAD_GATEWAY, format!("Worker error: {}", e)).into_response()
            }
        }
    }

    // ... 實現其他必需的方法（liveness, readiness, health 等）...
    
    fn liveness(&self) -> Response {
        (StatusCode::OK, "OK").into_response()
    }

    fn readiness(&self) -> Response {
        // 檢查是否有健康的 worker
        let has_healthy_workers = self.worker_registry.get_all_workers()
            .iter()
            .any(|w| w.is_healthy());
        
        if has_healthy_workers {
            (StatusCode::OK, "OK").into_response()
        } else {
            (StatusCode::SERVICE_UNAVAILABLE, "No healthy workers").into_response()
        }
    }

    async fn health(&self, _req: Request) -> Response {
        self.readiness()
    }

    // ... 其他必需的方法實現 ...
}
```

#### 步驟 2：在路由器工廠中註冊新路由器

修改 `src/routers/factory.rs`：

```rust
// ... 其他導入 ...

// 導入新路由器
use super::http::tensorrt_router::TensorRTRouter;

/// 路由器工廠
/// 根據配置創建對應的路由器實例
pub struct RouterFactory;

impl RouterFactory {
    /// 創建路由器實例
    /// 
    /// 參數：
    /// - `config`: 路由器配置
    /// - `ctx`: 應用上下文
    /// 
    /// 返回值：
    /// - `Result<Arc<dyn RouterTrait>, String>`: 成功返回路由器實例，失敗返回錯誤
    pub async fn create(
        config: &RouterConfig,
        ctx: &Arc<AppContext>,
    ) -> Result<Arc<dyn RouterTrait>, String> {
        match &config.mode {
            // ... 其他路由模式的處理 ...
            
            // 添加新的後端類型檢查
            // 這裡我們可以通過配置中的某個字段來判斷是否使用 TensorRT-LLM
            // 例如：如果 worker URL 包含 "tensorrt"，或者有專門的配置字段
            RoutingMode::Regular { worker_urls } if is_tensorrt_backend(worker_urls) => {
                // 創建 TensorRT-LLM 路由器
                let router = TensorRTRouter::new(worker_urls.clone(), ctx).await?;
                Ok(Arc::new(router))
            }
            
            // ... 其他分支 ...
        }
    }

    /// 輔助函數：判斷是否為 TensorRT-LLM 後端
    /// 可以通過檢查 worker URL 或其他標識來判斷
    fn is_tensorrt_backend(worker_urls: &[String]) -> bool {
        // 示例邏輯：如果任何 worker URL 包含 "tensorrt"，則認為是 TensorRT-LLM 後端
        // 實際實現可能需要更複雜的邏輯，例如檢查配置中的 backend_type 字段
        worker_urls.iter().any(|url| url.contains("tensorrt"))
    }
}
```

#### 步驟 3：在 `src/routers/http/mod.rs` 中導出新路由器

修改 `src/routers/http/mod.rs`：

```rust
// ... 其他模塊聲明和導出 ...

// 添加新路由器模塊的聲明
mod tensorrt_router;

// 導出新路由器（如果需要）
pub use tensorrt_router::TensorRTRouter;
```

---

## 總結

上述三個擴展點的示例展示了：

1. **添加新負載平衡策略**：展示了如何實現策略 trait、註冊策略、配置類型定義
2. **添加新 API 端點**：展示了如何定義請求/響應類型、添加路由處理函數、實現路由器方法
3. **添加新後端支持**：展示了如何創建新的路由器實現、協議轉換、註冊到工廠

每個示例都包含了詳細的逐行註釋，解釋了：
- 每行代碼的作用
- 為什麼需要這樣寫
- 如何與項目中的其他組件交互

這些示例可以作為模板，幫助你快速擴展項目的功能。

---

## 總結

vLLM Router 是一個設計精良的請求轉發系統，具有以下特點：

1. **高性能**: Rust 實現的核心邏輯提供低延遲和高吞吐量
2. **靈活性**: 支持多種負載平衡策略和部署模式
3. **可擴展性**: 模塊化設計易於擴展和定製
4. **生產就緒**: 包含熔斷器、重試、監控等企業級功能
5. **易用性**: Python 綁定和完整的文檔

該項目為 vLLM 的大規模部署提供了一個強大且靈活的解決方案。

