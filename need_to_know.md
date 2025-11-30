# Need to Know: OpenTelemetry & Istio Configuration

This document captures key architectural decisions and "need to know" concepts for the monitoring setup.

## 1. Why Port Discovery (`role: pod`) instead of Service Discovery?

In our OpenTelemetry Collector configuration, we use `role: pod` for discovering targets instead of the more common `role: endpoints` (Service Discovery).

*   **Sidecars are "Invisible" to Services**: A Kubernetes `Service` typically exposes only the *application's* port (e.g., 8080). It rarely exposes the *sidecar's* admin or metrics ports (e.g., 15020 for merged metrics or 15090 for Envoy stats). If we used Service Discovery, Prometheus would find the application port but miss the Istio sidecar entirely.
*   **Direct Access**: Using `role: pod` allows us to enumerate *every* port on the container. This enables us to find named ports like `http-envoy-prom` (Istio) or `http-prom` (Flux) even if they are not listed in a Service definition.
*   **Granularity**: We want to scrape every individual pod instance to detect if a specific replica is failing. While `role: endpoints` also supports this, `role: pod` is often more robust for finding non-service ports and metadata.

## 2. Why Separate Scrape Jobs instead of "One Giant Job"?

We configured separate scrape jobs (e.g., `istio-sidecar`, `flux-system`) instead of a single catch-all `kubernetes-pods` job.

*   **Metric Pollution & Filtering**: Istio produces **thousands** of metrics. Mixing these with application metrics makes "drop" rules complex and risky. By separating them:
    *   **`istio-sidecar` job**: We can apply aggressive filtering (e.g., dropping 90% of unused Envoy stats) without risking the loss of business metrics.
    *   **`flux-system` job**: We can target specific ports like `http-prom` without affecting other workloads.
*   **Different Scrape Intervals**: In a production environment, you might want to scrape:
    *   **Applications** every 10 seconds (for high-resolution latency data).
    *   **Infrastructure (Flux/Istio)** every 30-60 seconds (since infrastructure state changes more slowly).
    Separate jobs allow for this fine-grained tuning.
*   **Reliability**: If the Istio Control Plane goes down and its scrape job hangs or fails, it will not block the collector from scraping critical Application metrics.
