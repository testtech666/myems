# MyEMS 系统分析

## 1. MyEMS 整体架构

MyEMS (我的能源管理系统) 似乎遵循物联网和能源管理平台中常见的分层架构。该架构可大致分为：

*   **设备层 (Device Layer)**：该层包括实际的耗能或监控设备（例如传感器、计量表、HVAC 系统），这些设备可能通过 Modbus 等协议进行通信。
*   **网关层 (Gateway Layer)**：该层负责从设备层收集数据。像 `myems-modbus-tcp`这样的组件充当网关，将特定于设备的协议（如 Modbus TCP/IP）转换为适合平台的格式。
*   **平台层 (Platform Layer)**：这是 MyEMS 系统的核心。它处理数据采集、处理、存储，并为应用程序提供 API。此处的关键组件包括：
    *   数据采集服务（例如 `myems-modbus-tcp`）
    *   数据处理服务（`myems-cleaning`、`myems-normalization`、`myems-aggregation`）
    *   核心 API 服务（`myems-api`）
    *   数据库（`myems_system_db`、`myems_historical_db`、`myems_energy_db`）
*   **应用层 (Application Layer)**：该层提供用户界面和分析工具，用于与收集和处理的数据进行交互。这包括 `myems-admin`（管理后台 UI）和 `myems-web`（Web 端 UI）。

**常规数据流：**

1.  数据由网关组件（例如 `myems-modbus-tcp`）从终端设备收集。
2.  原始数据可能通过 `myems-api` 发送到平台。
3.  数据经过几个处理阶段：
    *   **清洗 (`myems-cleaning`)：** 消除错误、不一致和异常值。
    *   **标准化 (`myems-normalization`)：** 将数据转换为标准格式和单位。
    *   **聚合 (`myems-aggregation`)：** 按不同时间间隔（例如每小时、每天、每月的平均值或总计）汇总数据。
4.  处理后的数据存储在各种数据库中：
    *   系统配置和元数据存储在 `myems_system_db`。
    *   原始或接近原始的时间序列数据存储在 `myems_historical_db`。
    *   聚合和标准化的能源特定数据存储在 `myems_energy_db`。
5.  `myems-api` 为 `myems-admin` 和 `myems-web` 用户界面提供对此数据的访问，从而实现监控、配置和分析。

## 2. 组件分析

### `myems-modbus-tcp`

*   **主要功能**：充当 Modbus TCP/IP 网关。它从网络上启用 Modbus 的设备（传感器、计量表）轮询数据，并将此数据传输到 MyEMS 平台。
*   **关键技术/依赖**：Python（可能使用像 `pymodbus` 这样的库）、Modbus TCP/IP 协议。
*   **主要交互**：
    *   通过 Modbus TCP/IP 从物理设备读取数据。
    *   将收集的数据发送到 `myems-api` 或直接发送到消息队列（如 Kafka 或 RabbitMQ，尽管未明确提及），供 `myems-api` 或其他服务使用。
    *   可能通过 `myems-api` 从 `myems_system_db` 获取配置（例如要轮询的设备列表、轮询间隔、Modbus 寄存器映射）。

### `myems-api`

*   **主要功能**：为 MyEMS 平台提供中央 RESTful API。它是各种微服务、用户界面和潜在外部应用程序之间的主要通信枢纽。
*   **关键技术/依赖**：Python（可能使用像 Flask 或 FastAPI 这样的 Web 框架）、REST API 原则、JSON 用于数据交换。
*   **主要交互**：
    *   从像 `myems-modbus-tcp` 这样的网关组件接收数据。
    *   向 `myems-admin` 和 `myems-web` UI 提供数据以供显示和用户操作。
    *   与所有数据库（`myems_system_db`、`myems_historical_db`、`myems_energy_db`）交互以存储和检索数据。
    *   作为数据处理服务（`myems-cleaning`、`myems-normalization`、`myems-aggregation`）的接口，用于获取原始数据和存储处理后的数据。
    *   处理 UI 和其他组件的身份验证和授权。

### `myems-cleaning`

*   **主要功能**：负责清洗从传感器和计量表接收的原始数据。这包括处理缺失值、纠正错误、删除异常值，并在进一步处理或存储之前确保数据质量。
*   **关键技术/依赖**：Python、数据处理库（例如，如果使用 Pandas、NumPy）。
*   **主要交互**：
    *   通过 `myems-api` 从 `myems_historical_db` 或消息队列中获取原始数据。
    *   应用预定义或动态配置的清洗规则。
    *   将清洗后的数据存回 `myems_historical_db` 或通过 `myems-api` 或消息队列传递到下一阶段（`myems-normalization`）。
    *   可能记录清洗活动或异常。

### `myems-normalization`

*   **主要功能**：将清洗后的数据转换为标准化的格式、单位和结构。这确保了来自各种来源和设备类型的数据的一致性。例如，将所有温度读数转换为摄氏度或将所有能耗转换为 kWh。
*   **关键技术/依赖**：Python、数据处理库。
*   **主要交互**：
    *   从 `myems-cleaning`（或通过 `myems-api` 从 `myems_historical_db`）接收清洗后的数据。
    *   应用通常在 `myems_system_db` 中定义的标准化规则（例如单位转换、缩放）。
    *   通过 `myems-api` 将标准化数据存储到 `myems_historical_db`（可能在不同的表/集合中，或带有“normalized”标志）或 `myems_energy_db`。

### `myems-aggregation`

*   **主要功能**：执行数据聚合，计算标准化数据在不同时间段（例如每小时、每日、每周、每月的平均值、总和、最小值/最大值）的摘要和汇总。这对于报告、趋势分析和减少长期数据存储至关重要。
*   **关键技术/依赖**：Python、数据处理库，可能还有特定于数据库的聚合函数。
*   **主要交互**：
    *   通过 `myems-api` 从 `myems_historical_db` 或 `myems_energy_db` 获取标准化数据。
    *   根据 `myems_system_db` 中定义的规则执行聚合计算。
    *   通过 `myems-api` 将聚合数据存储到 `myems_energy_db`（非常适合能源特定摘要）或 `myems_historical_db` 中的摘要表。

### `myems-admin` (管理后台 UI)

*   **主要功能**：提供用于管理 MyEMS 系统的管理界面。这包括用户管理、设备配置、设置数据点、配置数据处理规则、监控系统健康状况以及其他管理任务。
*   **关键技术/依赖**：AngularJS (一个 JavaScript 框架)、HTML、CSS。
*   **主要交互**：
    *   与 `myems-api` 广泛通信以获取系统数据并发送配置更改。
    *   允许管理员管理存储在 `myems_system_db` 中的设置。
    *   可能提供对存储在 `myems_historical_db` 和 `myems_energy_db` 中的数据的视图，用于诊断或管理目的。

### `myems-web` (Web 端 UI)

*   **主要功能**：为最终用户提供一个面向用户的 Web 应用程序，用于可视化和分析能源数据、查看报告、仪表板，并可能与可控设备交互（如果支持）。
*   **关键技术/依赖**：ReactJS (一个用于构建用户界面的 JavaScript 库)、HTML、CSS、图表库（例如 Chart.js、D3.js）。
*   **主要交互**：
    *   与 `myems-api` 通信以获取处理和聚合的数据以供显示。
    *   以用户友好的格式（图表、表格、仪表板）呈现来自 `myems_energy_db` 和 `myems_historical_db` 的数据。
    *   可能允许用户自定义其视图或设置首选项，这些首选项将通过 `myems-api` 可能保存到 `myems_system_db` 中。

### 数据库

*   **`myems_system_db`**
    *   **主要功能**：存储系统配置、元数据和设置。
    *   **关键技术/依赖**：可能是关系数据库（例如 PostgreSQL、MySQL）或 NoSQL 文档数据库（例如 MongoDB），适用于存储各种配置对象。
    *   **存储数据**：用户帐户、角色、权限；设备配置（名称、类型、Modbus 地址、寄存器映射）；数据点定义；标准化规则；聚合规则；UI 设置；API 密钥。
    *   **主要交互**：由 `myems-api` 访问，为所有其他组件提供配置。`myems-admin` UI 通过 `myems-api` 与其进行大量交互以进行管理。

*   **`myems_historical_db`**
    *   **主要功能**：存储从设备收集的时间序列数据。这可以包括原始数据、清洗后的数据，以及在进行大规模聚合之前可能存在的某种程度的标准化数据。针对快速写入和基于时间的查询进行了优化。
    *   **关键技术/依赖**：时间序列数据库（例如 InfluxDB、TimescaleDB）或能够有效处理时间序列数据的 NoSQL 数据库（例如 Cassandra、具有适当索引的 MongoDB）。
    *   **存储数据**：来自传感器和计量表带时间戳的读数（例如温度、电压、电流、功率、原始消耗值）。
    *   **主要交互**：`myems-api` 在此处写入来自收集器的数据。`myems-cleaning` 和 `myems-normalization` 从此数据库读取数据并向其写入数据（或其中的数据版本）。`myems-aggregation` 从中读取数据。UI 可以通过 `myems-api` 访问详细的历史数据。

*   **`myems_energy_db`**
    *   **主要功能**：存储经过处理、标准化和聚合的能源特定数据，经过优化以用于报告和分析。该数据库通常保存用于能源消耗分析、计费和绩效指标的“最终”版本数据。
    *   **关键技术/依赖**：由于聚合数据的结构化特性，可能是关系数据库（PostgreSQL、MySQL），或者如果数据量非常大，则可能是数据仓库解决方案。
    *   **存储数据**：聚合的能源消耗（例如每小时、每日、每月 kWh）、电力需求、成本计算、碳足迹数据、关键绩效指标 (KPI)。
    *   **主要交互**：`myems-aggregation` 通过 `myems-api` 在此处写入其输出。`myems-web` UI 严重依赖此数据库（通过 `myems-api`）来获取仪表板和报告。`myems-api` 从中读取数据以服务于分析查询。

## 3. 数据流摘要

MyEMS 中的端到端数据流可总结如下：

1.  **采集**：`myems-modbus-tcp` 使用 Modbus TCP/IP 协议从终端设备（例如能源计量表、传感器）轮询数据。
2.  **注入**：由 `myems-modbus-tcp` 收集的原始数据发送到 `myems-api`。
3.  **初始存储和处理触发**：`myems-api` 最初可能将此原始数据存储在 `myems_historical_db` 中。此事件（新数据到达）可能会触发处理流水线。
4.  **清洗**：`myems-cleaning` 服务获取原始数据（通过 `myems-api` 从 `myems_historical_db` 获取），应用清洗算法，并将清洗后的数据存回 `myems_historical_db`（或将其传递下去）。
5.  **标准化**：`myems-normalization` 服务获取清洗后的数据，应用标准化规则（例如，基于通过 `myems-api` 从 `myems_system_db` 获取的配置进行单位转换），并将标准化数据存储起来，可能存储在 `myems_historical_db` 中，或为 `myems_energy_db` 做准备。
6.  **聚合**：`myems-aggregation` 服务处理标准化数据，计算聚合值（例如，基于 `myems_system_db` 中的规则计算每日总和、每小时平均值）。这些聚合主要通过 `myems-api` 存储在 `myems_energy_db` 中。
7.  **数据服务**：`myems-api` 从所有三个数据库（`myems_system_db` 用于配置，`myems_historical_db` 用于详细/原始数据，`myems_energy_db` 用于聚合/分析数据）读取数据以响应请求。
8.  **可视化与管理**：
    *   `myems-admin` (AngularJS UI) 与 `myems-api` 交互，以管理存储在 `myems_system_db` 中的系统配置（用户、设备、规则）并监控系统状态。
    *   `myems-web` (ReactJS UI) 与 `myems-api` 交互，以从 `myems_energy_db` 获取并显示处理和聚合的能源数据，并可能从 `myems_historical_db` 获取详细数据用于仪表板、报告和分析。

数据转换发生在每个处理步骤：原始 Modbus 读数被清除错误，然后标准化为一致的单位，最后聚合成有意义的摘要以供分析和报告。

## 4. 数据库概览

*   **`myems_system_db`**：
    *   **角色**：MyEMS 平台所有配置和元数据的中央存储库。
    *   **存储数据**：系统设置、用户凭据、设备定义、数据点配置、清洗规则、标准化规则和聚合规则。从本质上讲，它拥有系统的“智能”和操作参数。

*   **`myems_historical_db`**：
    *   **角色**：存储来自受监控设备的原始或最低限度处理的时间序列数据。它作为详细读数的主要历史存档。
    *   **存储数据**：来自传感器和计量表带时间戳的测量数据（例如电压、电流、温度、原始消耗脉冲）。此数据用于详细的取证分析、审计，并作为清洗、标准化和聚合过程的输入。

*   **`myems_energy_db`**：
    *   **角色**：存储经过提炼、聚合和分析的能源特定数据，可供最终用户应用程序、报告和仪表板使用。
    *   **存储数据**：计算得出的能源消耗（例如每小时、每日、每月 kWh）、需求数据、成本信息、KPI 以及从历史数据中得出的其他分析指标。该数据库针对汇总能源信息的快速查询进行了优化。

这种结构分离了关注点：系统配置、原始历史数据和已处理的分析数据，这是此类系统中常见且有效的做法。
