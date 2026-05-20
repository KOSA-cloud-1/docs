# Proxmox Cluster Hardware Summary
| Node   | CPU Model                      | Physical Cores | Threads | Memory    | Virtualization | Recommended Role       |
| ------ | ------------------------------ | -------------: | ------: | --------: | -------------- | ---------------------- |
| team11 | Intel Core i7-13700 (13th Gen) |             16 |      24 |   32 GB   | VT-x           | Worker 집중 노드         |
| team12 | Intel Core i7-13700 (13th Gen) |             16 |      24 |   32 GB   | VT-x           | Control Plane + Worker |
| team13 | Intel Core i7-13700 (13th Gen) |             16 |      24 |   32 GB   | VT-x           | Control Plane + Worker |
| team14 | Intel Core i7-13700 (13th Gen) |             16 |      24 |   32 GB   | VT-x           | Control Plane + Worker |

# 현재 Kubernetes VM 배치 구조
| Node   | 배치된 VM                   |
| ------ | ------------------------- |
| team11 | worker4, worker5, worker6 |
| team12 | cp1, worker1              |
| team13 | cp2, worker2              |
| team14 | cp3, worker3              |

# Control Plane Nodes
| Node | VM ID | Proxmox Node | 1G IP         | 10G IP      | vCPU |    Max Memory | Balloon Memory |
| ---- | ----: | ------------ | ------------- | ----------- | ---: | ------------: | -------------: |
| cp1  |  1001 | team12       | 172.17.128.21 | 10.10.10.51 |    4 | 4096 MB (4GB) |  2048 MB (2GB) |
| cp2  |  1002 | team13       | 172.17.128.22 | 10.10.10.52 |    4 | 4096 MB (4GB) |  2048 MB (2GB) |
| cp3  |  1003 | team14       | 172.17.128.23 | 10.10.10.53 |    4 | 4096 MB (4GB) |  2048 MB (2GB) |


# Worker Nodes
| Node    | VM ID | Proxmox Node | 1G IP         | 10G IP      | vCPU |      Max Memory | Balloon Memory |
| ------- | ----: | ------------ | ------------- | ----------- | ---: | --------------: | -------------: |
| worker1 |  1011 | team12       | 172.17.128.24 | 10.10.10.54 |    8 | 32768 MB (32GB) |  8192 MB (8GB) |
| worker2 |  1012 | team13       | 172.17.128.25 | 10.10.10.55 |    8 | 32768 MB (32GB) |  8192 MB (8GB) |
| worker3 |  1013 | team14       | 172.17.128.26 | 10.10.10.56 |    4 |   4096 MB (4GB) |  2048 MB (2GB) |
| worker4 |  1014 | team11       | 172.17.128.27 | 10.10.10.57 |    4 |   4096 MB (4GB) |  2048 MB (2GB) |
| worker5 |  1015 | team11       | 172.17.128.28 | 10.10.10.58 |    4 |   4096 MB (4GB) |  2048 MB (2GB) |
| worker6 |  1016 | team11       | 172.17.128.29 | 10.10.10.59 |    4 |   4096 MB (4GB) |  2048 MB (2GB) |
