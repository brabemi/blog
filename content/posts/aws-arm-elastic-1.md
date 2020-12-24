---
title: "AWS Graviton2 and Elasticsearch - the first impression"
date: 2020-12-24T15:00:12+01:00
draft: false
tags: [aws, arm, elasticsearch]
---

You may have noticed there is lot of noise about ARM vs x86. I would say mainly because of new mac books wit apple silicon. But if you are AWS user you may have noticed that Amazon has arm based EC2 instances for a while.

## Motivation
At the moment there is 2nd generation of AWS Graviton processors and available EC2 T4g, M6g, C6g, and R6g instances, and their variants with local NVMe-based SSD storage, that provide up to 40% better price performance over comparable current generation x86-based instances [[1]](#sources). From my SRE/DevOps engineer perspective potential 40% reduction of AWS EC2 bill seems very interesting.

In our company big portion of our EC2 is used to running Elasticsearch. During the 2020 we were able to move majority of our server to the Elasticsearch 7 so the fact that since Elasticsearch 7.8.0 ARM and AArch64 architecture is officially supported [[2]](#sources) was quite interesting news for us. It seems to me as the best option to start testing ARM instances in our infrastructure on Elasticsearch because:

1. it is supported (we don’t have to build ARM version on our own)
2. we have lot of Elasticsearch servers (one deployment can cover big portion of infrastructure)
3. Elasticsearch is distributed (you can start with 1 server in cluster and slowly continue in conversion)
4. Elasticsearch performs many parallel tasks, so it might benefit from real physical cores instead of logical cores (simultaneous multithreading in Intel and AMD x86 chips)

## Setup
I have found that there is nice benchmark tool to Elasticsearch setup called `esrally` [[3]](#sources) with multiple benchmarks for different use cases [[4]](#sources). Install and run the benchmark in default settings on ubuntu 20.04 is quite easy.

### Install
```
sudo apt-get install build-essential python3-dev openjdk-11-jdk python3-pip
python3 -m pip install esrally --user
```
### Run

**AMD64**
```
export JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"
./.local/bin/esrally configure
./.local/bin/esrally --distribution-version=7.8.0
```
**ARM64**
```
export JAVA_HOME="/usr/lib/jvm/java-11-openjdk-arm64"
./.local/bin/esrally configure
./.local/bin/esrally --distribution-version=7.8.0
```
### EC2 Instances

To my tests I have decided to use default track (benchmark) `geonames` with default settings. The instances EC2 families were T3 (Intel based x86), T3a (AMD based x86) and T4g (ARM based) all of them in medium size (2 vCPU cores and 4GB of RAM) with unlimited CPU credits. The selected EBS for all instances was 30GB gp3 volume with 3000 IOPS (default IOPS count for gp3). The memory consumption of java process was ~1.5GB so the 4GB is okay for recommended 50:50 ratio between heap and file cache [[5]](#sources).

| EC2 instance | CPU | vCPUs | RAM | clock speed | Price/hr* |
| - | - | - | - | - | - |
| t3.medium | Intel Xeon Platinum 8000 | 2 | 4 | 3.1 GHz | $0.0456 |
| t3a.medium | AMD EPYC 7000 | 2 | 4 | 2.5 GHz | $0.0408 |
| t4g.medium | AWS Graviton2 | 2 | 4 | 2.5 GHz | $0.0368 |

\* On-demand Europe (Ireland) [[6]](#sources)

## Results

### Median Throughput (higher is better)
Number of operations that Elasticsearch can perform within a certain time period, usually per second. [[7]](#sources)
| Task | Value | t3.medium | t3a.medium | t4g.medium |
| - | - | - | - | - |
| index-append | docs/s | 16336.3 | 11865.3 | 20130.9 |
| index-stats | ops/s | 90.06 | 90.06 | 90.07 |
| node-stats | ops/s | 90.09 | 90.07 | 90.1 |
| default | ops/s | 50.04 | 50.04 | 50.04 |
| term | ops/s | 100.08 | 100.06 | 100.08 |
| phrase | ops/s | 110.06 | 110.05 | 110.06 |
| country_agg_uncached | ops/s | 1.41 | 1.17 | 1.65 |
| country_agg_cached | ops/s | 100.05 | 100.04 | 100.05 |
| scroll | pages/s | 20.04 | 20.03 | 20.04 |
| expression | ops/s | 0.8 | 0.74 | 1.02 |
| painless_static | ops/s | 0.56 | 0.46 | 0.78 |
| painless_dynamic | ops/s | 0.58 | 0.47 | 0.78 |
| decay_geo_gauss_function_score | ops/s | 0.77 | 0.66 | 0.8 |
| decay_geo_gauss_script_score | ops/s | 0.73 | 0.61 | 0.8 |
| field_value_function_score | ops/s | 1.5 | 1.5 | 1.5 |
| field_value_script_score | ops/s | 1.23 | 1.1 | 1.5 |
| large_terms | ops/s | 0.55 | 0.43 | 0.76 |
| large_filtered_terms | ops/s | 0.55 | 0.47 | 0.81 |
| large_prohibited_terms | ops/s | 0.56 | 0.48 | 0.84 |
| desc_sort_population | ops/s | 1.5 | 1.5 | 1.5 |
| asc_sort_population | ops/s | 1.5 | 1.5 | 1.5 |
| asc_sort_with_after_population | ops/s | 1.5 | 1.5 | 1.5 |
| desc_sort_geonameid | ops/s | 6.02 | 6.02 | 6.02 |
| desc_sort_with_after_geonameid | ops/s | 2.65 | 2.67 | 3.59 |
| asc_sort_geonameid | ops/s | 6.02 | 6.02 | 6.02 |
| asc_sort_with_after_geonameid | ops/s | 3.19 | 3.17 | 4.22 |

### 90th percentile latency (lower is better)
Time period between submission of a request and receiving the complete response. It also includes wait time, i.e. the time the request spends waiting until it is ready to be serviced by Elasticsearch. [[7]](#sources)

| Task | Value | t3.medium | t3a.medium | t4g.medium |
| - | - | - | - | - |
| index-stats | ms | 4.56241 | 4.78031 | 4.44132 |
| node-stats | ms | 4.47714 | 5.34818 | 4.15083 |
| default | ms | 4.61649 | 5.09174 | 4.41905 |
| term | ms | 3.63299 | 4.28009 | 3.64795 |
| phrase | ms | 5.17816 | 6.07525 | 4.47468 |
| country_agg_uncached | ms | 124891 | 166506 | 94687.3 |
| country_agg_cached | ms | 3.34225 | 4.02981 | 3.04555 |
| scroll | ms | 670.819 | 765.98 | 570.697 |
| expression | ms | 217096 | 247556 | 138891 |
| painless_static | ms | 320471 | 431436 | 179742 |
| painless_dynamic | ms | 304484 | 427699 | 179525 |
| decay_geo_gauss_function_score | ms | 89906.5 | 150580 | 74991.4 |
| decay_geo_gauss_script_score | ms | 105799 | 182819 | 71880.2 |
| field_value_function_score | ms | 633.798 | 672.882 | 445.357 |
| field_value_script_score | ms | 43596.9 | 71071.8 | 608.366 |
| large_terms | ms | 266805 | 412260 | 117055 |
| large_filtered_terms | ms | 261997 | 353390 | 94404.3 |
| large_prohibited_terms | ms | 255688 | 336764 | 81920.8 |
| desc_sort_population | ms | 226.389 | 300.21 | 222.337 |
| asc_sort_population | ms | 231.237 | 346.524 | 217.209 |
| asc_sort_with_after_population | ms | 319.709 | 441.591 | 321.494 |
| desc_sort_geonameid | ms | 62.3339 | 85.2928 | 32.1476 |
| desc_sort_with_after_geonameid | ms | 61356.7 | 60678.6 | 32569.2 |
| asc_sort_geonameid | ms | 22.8396 | 41.6798 | 16.0042 |
| asc_sort_with_after_geonameid | ms | 42734.5 | 43248.4 | 20625.2 |

### 90th percentile service time (lower is better)

Time period between start of request processing and receiving the complete response. This metric can easily be mixed up with `latency` but does not include waiting time. This is what most load testing tools refer to as “latency” (although it is incorrect). [[7]](#sources)

| Task | Value | t3.medium | t3a.medium | t4g.medium |
| - | - | - | - | - |
| index-stats | ms | 2.93328 | 3.53074 | 2.61744 |
| node-stats | ms | 3.30236 | 4.06215 | 3.09698 |
| default | ms | 2.83038 | 3.68959 | 2.44764 |
| term | ms | 2.70983 | 3.10945 | 2.59679 |
| phrase | ms | 4.11032 | 4.67643 | 3.54806 |
| country_agg_uncached | ms | 707.677 | 847.525 | 608.296 |
| country_agg_cached | ms | 2.12763 | 2.60659 | 1.84201 |
| scroll | ms | 665.936 | 764.753 | 568.912 |
| expression | ms | 1249.61 | 1364.56 | 998.854 |
| painless_static | ms | 1770.04 | 2149.38 | 1302.5 |
| painless_dynamic | ms | 1723.03 | 2135.88 | 1305.81 |
| decay_geo_gauss_function_score | ms | 1318.65 | 1531.22 | 1268.05 |
| decay_geo_gauss_script_score | ms | 1373.07 | 1635.57 | 1257.33 |
| field_value_function_score | ms | 632.535 | 669.613 | 443.951 |
| field_value_script_score | ms | 821.202 | 915.698 | 607.475 |
| large_terms | ms | 1801 | 2290.12 | 1291.52 |
| large_filtered_terms | ms | 1788.23 | 2092.46 | 1213.08 |
| large_prohibited_terms | ms | 1767.71 | 2038.35 | 1166.24 |
| desc_sort_population | ms | 225.353 | 298.996 | 220.656 |
| asc_sort_population | ms | 229.828 | 345.713 | 215.475 |
| asc_sort_with_after_population | ms | 317.484 | 440.449 | 319.929 |
| desc_sort_geonameid | ms | 61.1497 | 83.0399 | 31.0503 |
| desc_sort_with_after_geonameid | ms | 380.078 | 379.3 | 295.567 |
| asc_sort_geonameid | ms | 21.4847 | 39.6202 | 15.2917 |
| asc_sort_with_after_geonameid | ms | 314.37 | 316.442 | 250.036 |

## Conclusion

### Median Throughput Ratio (higher is better)

`t3.medium` as 1.00, lines where throughput was the same for all 3 instances were removed.

| Task | t3.medium | t3a.medium | t4g.medium |
| - | - | - | - |
| country_agg_uncached | 1.00 | 0.83 | 1.17 |
| expression | 1.00 | 0.93 | 1.28 |
| painless_static | 1.00 | 0.82 | 1.39 |
| painless_dynamic | 1.00 | 0.81 | 1.34 |
| decay_geo_gauss_function_score | 1.00 | 0.86 | 1.04 |
| decay_geo_gauss_script_score | 1.00 | 0.84 | 1.10 |
| field_value_script_score | 1.00 | 0.89 | 1.22 |
| large_terms | 1.00 | 0.78 | 1.38 |
| large_filtered_terms | 1.00 | 0.85 | 1.47 |
| large_prohibited_terms | 1.00 | 0.86 | 1.50 |
| desc_sort_with_after_geonameid | 1.00 | 1.01 | 1.35 |
| asc_sort_with_after_geonameid | 1.00 | 0.99 | 1.32 |
| **average values** | **1.00** | **0.87** | **1.30** |



Let's talk just about the tasks where there are some difference between instances. The tasks with the same values aren't probably CPU bounded.

We can see 87% of performance when we compare `t3a.medium` with `t3.medium`. This  is probably expected since ration between 2.5Ghz for AMD and 3.1Ghz for Intel is ~81%. The both values are for turbo CPU clock speed, so the real clock speed might be slightly different and there might be small difference in IPC. The price of `t3a.medium` is 90% of  `t3.medium`, so the performance per price is slightly in favor of Intel based instances.

Now the interesting part  `t4g.medium` vs `t3.medium`. 30% more performance for ARM is quite surprising to me and when we combine this with 20% lower price, performance per price is amazing. It is hard to say how much can Elasticsearch benefit from SMT on Intel and AMD processors vs the real cores on AWS Graviton2, but it might be an explanation why Graviton2 instance has 30% higher score than Intel and 50% higher score than AMD.
I am looking forward to do more tests and maybe try add few Graviton2 into our current Elasticsearch cluster to test some real world scenarios.


## Sources
[1][https://aws.amazon.com/ec2/graviton/](https://aws.amazon.com/ec2/graviton/)

[2][https://www.elastic.co/blog/elasticsearch-on-arm](https://www.elastic.co/blog/elasticsearch-on-arm)

[3][https://github.com/elastic/rally](https://github.com/elastic/rally)

[4][https://github.com/elastic/rally-tracks](https://github.com/elastic/rally-tracks)

[5][https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#heap-size-settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#heap-size-settings)

[6][https://aws.amazon.com/ec2/pricing/on-demand/](https://aws.amazon.com/ec2/pricing/on-demand/)

[7][https://esrally.readthedocs.io/en/stable/summary_report.html](https://esrally.readthedocs.io/en/stable/summary_report.html)
