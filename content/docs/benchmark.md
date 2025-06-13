---
title: Benchmark
weight: 4
---

Spegel is benchmarked using the [benchmark](https://github.com/spegel-org/benchmark). The tool creates pods in a cluster forcing image pulling, after all pods are running the tool with gather image pull metrics for each individual pod. The image is then updated causing a rolling update, which is configured to be 20% of the pods at a time. Once again the image pull time metric is gathered. These benchmarks are run as a suite for different image sizes and amount of layers. Refer to the [benchmark results](https://github.com/spegel-org/benchmark-results) for details about how to reproduce the results.

## Results

The results are compared to the [baseline results](https://github.com/spegel-org/benchmark-results/tree/main/results/baseline) which have been measured with the same setup but without Spegel running in the cluster.

<script src="https://cdn.jsdelivr.net/npm/echarts/dist/echarts.min.js"></script>

<script>

  const benchmarks = [
    "10MB-1",
    "10MB-4",
    "100MB-1",
    "100MB-4",
    "1GB-1",
    "1GB-4",
  ];

  const baseUrl = "https://raw.githubusercontent.com/spegel-org/benchmark-results/main/charts/";

  benchmarks.forEach(name => {
    fetch(`${baseUrl}${name}.json`)
      .then(res => {
        if (!res.ok) throw new Error(`Failed to load ${name}.json`);
        return res.json();
      })
      .then(option => {
        const el = document.getElementById(name);
        if (el) {
          const chart = echarts.init(el);
          chart.setOption(option);
        } else {
          console.warn(`Element #${name} not found`);
        }
      })
      .catch(err => console.error(err));
  });
</script>


{{< nesteddetails title="10 MB in 1 layer" closed="true" >}}
  <div id="10MB-1" style="width: 100%; height: 500px;"></div>
{{< /nesteddetails >}}

{{< nesteddetails title="10 MB in 4 layers" closed="true" >}}
  <div id="10MB-4" style="width: 100%; height: 500px;"></div>
{{< /nesteddetails >}}

{{< nesteddetails title="100 MB in 1 layer" closed="true" >}}
  <div id="100MB-1" style="width: 100%; height: 500px;"></div>
{{< /nesteddetails >}}

{{< nesteddetails title="100 MB in 4 layers" closed="true" >}}
  <div id="100MB-4" style="width: 100%; height: 500px;"></div>
{{< /nesteddetails >}}

{{< nesteddetails title="1 GB in 1 layer" closed="true" >}}
  <div id="1GB-1" style="width: 100%; height: 500px;"></div>
{{< /nesteddetails >}}

{{< nesteddetails title="1 GB in 4 layers" closed="true" >}}
  <div id="1GB-4" style="width: 100%; height: 500px;"></div>
{{< /nesteddetails >}}
