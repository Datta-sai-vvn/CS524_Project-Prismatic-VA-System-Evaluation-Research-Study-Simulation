# SVG Baseline - Performance Benchmark Report

This repository contains a minimal D3/SVG prototype for the Prismatic project. The goal was to validate the rendering bottleneck associated with large numbers of SVG DOM elements.

## 1. Implementation Overview

We implemented a minimal throwaway prototype with:
- **Data Pipeline (Python)**: Fetches S&P 500 data (2020), computes log returns, generates correlation matrices for N={40, 80, 160, 300, 500}, and computes a Prism (e,w) correlation grid for AAPL vs MSFT.
- **Frontend (D3/SVG)**: Renders the correlation matrix and Prism view using one `<rect>` per cell.
- **Benchmarking**: Instruments `Data Load` time (fetch + parse) and `TTFV` (Time to First Visual - rendering time).

## 2. Benchmark Results

### Machine Specifications
- **OS**: Windows 11 Home (10.0.26200)
- **Device**: HP Envy x360 2-in-1 Laptop 14-fc0xxx
- **CPU**: Intel(R) Core(TM) Ultra 7 155U
- **RAM**: 16 GB (15,819 MB Physical)
- **Browser**: Chrome Headless (via Puppeteer/Selenium)

### quantitative Metrics

| View | N (Items) | Total Cells | Data Load | Rendering (TTFV) | Hover Latency (p95) | Brush Latency (p95) | Brush FPS | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Matrix** | 40 | 1,600 | 21.2 ms | 30.6 ms | < 1ms | < 16ms | 60 | **Healthy Anchor** (Paper's ceiling) |
| **Matrix** | 80 | 6,400 | 19.4 ms | 125.0 ms | - | - | - | Performance cliff begins |
| **Matrix** | 160 | 25,600 | 77.9 ms | 128.3 ms | 2.1 ms | 40.0 ms | ~7 | **Anomaly**: Similar to N=80 due to browser optimizations |
| **Matrix** | 300 | 90,000 | 60.8 ms | **1,698.5 ms** | 0.4 ms | **116.8 ms** | **~2** | **Critical Failure**: Unusable interaction |
| **Matrix** | 500 | 236,200 | 233.4 ms | **4,186.2 ms** | - | - | 0 | Frozen |
| **Prism** | Pair | 30,876 | 12.2 ms | 100.9 ms | - | - | - | Smooth for single pair |

### Key Findings & Analysis

1.  **The N=40 Anchor**: At N=40 (1,600 cells), the standard SVG approach is perfectly viable with 30ms render times and 60 FPS. This aligns with the "ceiling" reported in many D3 examples. However, our target is N=500, which is **135× more expensive**.

2.  **The N=80 vs N=160 Anomaly**: Interestingly, TTFV for N=160 (128ms) is nearly identical to N=80 (125ms), despite 4× the DOM nodes. This is likely due to Chrome's layout engine batching and optimizing standard Grid layouts up to a certain threshold (approx 25k nodes). This optimization **masks the danger**, creating a false sense of security before performance falls off a cliff.

3.  **Interaction Collapse at N=300**: The most damning metric is the **2 FPS brush performance** at N=300. While the initial render (1.7s) is annoying, the laggy interaction (116ms latency) makes exploratory data analysis impossible. This proves the bottleneck is not just "waiting for the chart" but "using the chart".

4.  **Conclusion**: Standard SVG cannot support N=500. The breakdown happens swiftly after N=160. We must move to **Canvas or WebGL** for the main visualization layer to achieve 60 FPS at scale.

## 3. Setup Instructions

1.  **Environment**:
    ```bash
    python -m venv venv
    source venv/bin/activate  # or venv\Scripts\activate
    pip install -r data/requirements.txt
    ```

2.  **Data Generation**:
    ```bash
    python data/pipeline/01_fetch_prices.py
    python data/pipeline/02_compute_returns.py
    python data/pipeline/03_compute_correlations.py
    python data/pipeline/04_compute_prism.py
    ```

3.  **Running the Prototype**:
    Serve the root directory with any static server:
    ```bash
    python -m http.server 8080
    ```
    Visit `http://localhost:8080/svg-baseline/index.html`
