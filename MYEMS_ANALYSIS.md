# MyEMS System Analysis

## 1. Overall MyEMS Architecture

MyEMS (My Energy Management System) appears to follow a layered architecture common in IoT and Energy Management platforms. This architecture can be broadly categorized into:

*   **Device Layer**: This layer consists of the actual energy-consuming or monitoring devices (e.g., sensors, meters, HVAC systems) that might communicate via protocols like Modbus.
*   **Gateway Layer**: This layer is responsible for collecting data from the Device Layer. Components like `myems-modbus-tcp` act as gateways, translating device-specific protocols (like Modbus TCP/IP) into a format suitable for the platform.
*   **Platform Layer**: This is the core of the MyEMS system. It handles data ingestion, processing, storage, and provides APIs for applications. Key components here include:
    *   Data acquisition services (e.g., `myems-modbus-tcp`)
    *   Data processing services (`myems-cleaning`, `myems-normalization`, `myems-aggregation`)
    *   Core API services (`myems-api`)
    *   Databases (`myems_system_db`, `myems_historical_db`, `myems_energy_db`)
*   **Application Layer**: This layer provides user interfaces and analytical tools for interacting with the collected and processed data. This includes the `myems-admin` (Admin UI) and `myems-web` (Web UI).

**General Data Flow:**

1.  Data is collected from end devices by gateway components (e.g., `myems-modbus-tcp`).
2.  This raw data is sent to the platform, likely via the `myems-api`.
3.  The data undergoes several processing stages:
    *   **Cleaning (`myems-cleaning`):** Removes errors, inconsistencies, and outliers.
    *   **Normalization (`myems-normalization`):** Converts data into a standard format and units.
    *   **Aggregation (`myems-aggregation`):** Summarizes data over different time intervals (e.g., hourly, daily, monthly averages or totals).
4.  Processed data is stored in various databases:
    *   System configuration and metadata in `myems_system_db`.
    *   Raw or near-raw time-series data in `myems_historical_db`.
    *   Aggregated and normalized energy-specific data in `myems_energy_db`.
5.  The `myems-api` provides access to this data for the `myems-admin` and `myems-web` user interfaces, enabling monitoring, configuration, and analysis.

## 2. Component Analysis

### `myems-modbus-tcp`

*   **Primary Function**: Acts as a Modbus TCP/IP gateway. It polls data from Modbus-enabled devices (sensors, meters) on the network and transmits this data to the MyEMS platform.
*   **Key Technologies/Dependencies**: Python (likely using libraries like `pymodbus`), Modbus TCP/IP protocol.
*   **Main Interactions**:
    *   Reads data from physical devices via Modbus TCP/IP.
    *   Sends collected data to `myems-api` or directly to a message queue (like Kafka or RabbitMQ, though not explicitly mentioned) that `myems-api` or other services consume.
    *   May fetch configuration (e.g., list of devices to poll, polling intervals, Modbus register maps) from `myems_system_db` via `myems-api`.

### `myems-api`

*   **Primary Function**: Provides a central RESTful API for the MyEMS platform. It serves as the main communication hub between various microservices, user interfaces, and potentially external applications.
*   **Key Technologies/Dependencies**: Python (likely using a web framework like Flask or FastAPI), REST API principles, JSON for data interchange.
*   **Main Interactions**:
    *   Receives data from gateway components like `myems-modbus-tcp`.
    *   Provides data to `myems-admin` and `myems-web` UIs for display and user actions.
    *   Interacts with all databases (`myems_system_db`, `myems_historical_db`, `myems_energy_db`) to store and retrieve data.
    *   Serves as the interface for data processing services (`myems-cleaning`, `myems-normalization`, `myems-aggregation`) to fetch raw data and store processed data.
    *   Handles authentication and authorization for UI and other components.

### `myems-cleaning`

*   **Primary Function**: Responsible for cleaning raw data received from sensors and meters. This includes handling missing values, correcting errors, removing outliers, and ensuring data quality before further processing or storage.
*   **Key Technologies/Dependencies**: Python, data processing libraries (e.g., Pandas, NumPy if used).
*   **Main Interactions**:
    *   Fetches raw data, possibly from `myems_historical_db` or a message queue, via `myems-api`.
    *   Applies predefined or dynamically configured cleaning rules.
    *   Stores cleaned data back into `myems_historical_db` or passes it to the next stage (`myems-normalization`) via `myems-api` or a message queue.
    *   May log cleaning activities or exceptions.

### `myems-normalization`

*   **Primary Function**: Converts cleaned data into a standardized format, units, and structure. This ensures consistency across data from various sources and types of devices. For example, converting all temperature readings to Celsius or all energy consumption to kWh.
*   **Key Technologies/Dependencies**: Python, data processing libraries.
*   **Main Interactions**:
    *   Receives cleaned data from `myems-cleaning` (or `myems_historical_db` via `myems-api`).
    *   Applies normalization rules (e.g., unit conversions, scaling) often defined in `myems_system_db`.
    *   Stores normalized data, potentially into `myems_historical_db` (perhaps in different tables/collections or with a 'normalized' flag) or `myems_energy_db`, via `myems-api`.

### `myems-aggregation`

*   **Primary Function**: Performs data aggregation, calculating summaries and roll-ups of normalized data over various time periods (e.g., hourly, daily, weekly, monthly averages, sums, min/max values). This is crucial for reporting, trend analysis, and reducing storage for long-term data.
*   **Key Technologies/Dependencies**: Python, data processing libraries, possibly database-specific aggregation functions.
*   **Main Interactions**:
    *   Fetches normalized data from `myems_historical_db` or `myems_energy_db` via `myems-api`.
    *   Performs aggregation calculations based on rules defined in `myems_system_db`.
    *   Stores aggregated data into `myems_energy_db` (ideal for energy-specific summaries) or summary tables in `myems_historical_db` via `myems-api`.

### `myems-admin` (Admin UI)

*   **Primary Function**: Provides an administrative interface for managing the MyEMS system. This includes user management, device configuration, setting up data points, configuring data processing rules, monitoring system health, and other administrative tasks.
*   **Key Technologies/Dependencies**: AngularJS (a JavaScript framework), HTML, CSS.
*   **Main Interactions**:
    *   Communicates extensively with `myems-api` to fetch system data and send configuration changes.
    *   Allows administrators to manage settings stored in `myems_system_db`.
    *   May provide views into data stored in `myems_historical_db` and `myems_energy_db` for diagnostic or administrative purposes.

### `myems-web` (Web UI)

*   **Primary Function**: Offers a user-facing web application for end-users to visualize and analyze energy data, view reports, dashboards, and potentially interact with controllable devices (if supported).
*   **Key Technologies/Dependencies**: ReactJS (a JavaScript library for building user interfaces), HTML, CSS, charting libraries (e.g., Chart.js, D3.js).
*   **Main Interactions**:
    *   Communicates with `myems-api` to fetch processed and aggregated data for display.
    *   Presents data from `myems_energy_db` and `myems_historical_db` in user-friendly formats (charts, tables, dashboards).
    *   May allow users to customize their views or set preferences, which would be saved via `myems-api` likely into `myems_system_db`.

### Databases

*   **`myems_system_db`**
    *   **Primary Function**: Stores system configuration, metadata, and settings.
    *   **Key Technologies/Dependencies**: Likely a relational database (e.g., PostgreSQL, MySQL) or a NoSQL document database (e.g., MongoDB) suitable for storing varied configuration objects.
    *   **Data Stored**: User accounts, roles, permissions; device configurations (names, types, Modbus addresses, register maps); data point definitions; normalization rules; aggregation rules; UI settings; API keys.
    *   **Main Interactions**: Accessed by `myems-api` to provide configuration to all other components. `myems-admin` UI heavily interacts with it via `myems-api` for management.

*   **`myems_historical_db`**
    *   **Primary Function**: Stores time-series data collected from devices. This can include raw data, cleaned data, and possibly some level of normalized data before extensive aggregation. Optimized for fast writes and time-based queries.
    *   **Key Technologies/Dependencies**: Time-series database (e.g., InfluxDB, TimescaleDB) or a NoSQL database capable of handling time-series data efficiently (e.g., Cassandra, MongoDB with proper indexing).
    *   **Data Stored**: Timestamped readings from sensors and meters (e.g., temperature, voltage, current, power, raw consumption values).
    *   **Main Interactions**: `myems-api` writes data from collectors here. `myems-cleaning` and `myems-normalization` read from and write to this database (or versions of data within it). `myems-aggregation` reads from it. UIs may access detailed historical data via `myems-api`.

*   **`myems_energy_db`**
    *   **Primary Function**: Stores processed, normalized, and aggregated energy-specific data, optimized for reporting and analytics. This database typically holds the "final" version of data used for energy consumption analysis, billing, and performance indicators.
    *   **Key Technologies/Dependencies**: Could be a relational database (PostgreSQL, MySQL) due to structured nature of aggregated data, or a data warehouse solution if volumes are very large.
    *   **Data Stored**: Aggregated energy consumption (e.g., hourly, daily, monthly kWh), power demand, cost calculations, carbon footprint data, key performance indicators (KPIs).
    *   **Main Interactions**: `myems-aggregation` writes its output here via `myems-api`. `myems-web` UI heavily relies on this database (via `myems-api`) for dashboards and reports. `myems-api` reads from it to serve analytical queries.

## 3. Data Flow Summary

The end-to-end data flow in MyEMS can be summarized as follows:

1.  **Acquisition**: `myems-modbus-tcp` polls data from end devices (e.g., energy meters, sensors) using the Modbus TCP/IP protocol.
2.  **Ingestion**: The raw data collected by `myems-modbus-tcp` is sent to the `myems-api`.
3.  **Initial Storage & Processing Trigger**: `myems-api` might initially store this raw data in `myems_historical_db`. This event (new data arrival) could trigger the processing pipeline.
4.  **Cleaning**: `myems-cleaning` service fetches raw data (from `myems_historical_db` via `myems-api`), applies cleaning algorithms, and stores the cleaned data back into `myems_historical_db` (or passes it on).
5.  **Normalization**: `myems-normalization` service takes the cleaned data, applies normalization rules (e.g., unit conversions based on configurations from `myems_system_db` via `myems-api`), and stores the normalized data, possibly in `myems_historical_db` or prepares it for `myems_energy_db`.
6.  **Aggregation**: `myems-aggregation` service processes the normalized data, calculating aggregates (e.g., daily sums, hourly averages based on rules from `myems_system_db`). These aggregates are primarily stored in `myems_energy_db` via `myems-api`.
7.  **Data Serving**: `myems-api` reads from all three databases (`myems_system_db` for configurations, `myems_historical_db` for detailed/raw data, `myems_energy_db` for aggregated/analytical data) to serve requests.
8.  **Visualization & Management**:
    *   `myems-admin` (AngularJS UI) interacts with `myems-api` to manage system configurations (users, devices, rules) stored in `myems_system_db` and to monitor system status.
    *   `myems-web` (ReactJS UI) interacts with `myems-api` to fetch and display processed and aggregated energy data from `myems_energy_db` and potentially detailed data from `myems_historical_db` for dashboards, reports, and analysis.

Data transformation occurs at each processing step: raw Modbus readings are cleaned of errors, then normalized to consistent units, and finally aggregated into meaningful summaries for analysis and reporting.

## 4. Database Overview

*   **`myems_system_db`**:
    *   **Role**: Central repository for all configuration and metadata of the MyEMS platform.
    *   **Data Stored**: System settings, user credentials, device definitions, data point configurations, rules for cleaning, normalization, and aggregation. Essentially, it holds the "intelligence" and operational parameters of the system.

*   **`myems_historical_db`**:
    *   **Role**: Stores raw or minimally processed time-series data from the monitored devices. It serves as the primary historical archive for detailed readings.
    *   **Data Stored**: Timestamped measurements from sensors and meters (e.g., voltage, current, temperature, raw consumption pulses). This data is used for detailed forensic analysis, auditing, and as input for the cleaning, normalization, and aggregation processes.

*   **`myems_energy_db`**:
    *   **Role**: Stores refined, aggregated, and analyzed energy-specific data ready for consumption by end-user applications, reports, and dashboards.
    *   **Data Stored**: Calculated energy consumption (e.g., hourly, daily, monthly kWh), demand data, cost information, KPIs, and other analytical metrics derived from the historical data. This database is optimized for fast querying of summarized energy information.

This structure separates concerns: system configuration, raw historical data, and processed analytical data, which is a common and effective practice in such systems.
