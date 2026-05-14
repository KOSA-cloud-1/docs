# Kubernetes VIP 
172.17.130.10   API Server VIP (kube-vip)

# Kubernetes VM IP 할당표
| Hostname | VM ID | Proxmox Node | 1G IP           | 10G IP        | 역할           |
| -------- | ----: | ------------ | --------------- | ------------- | ------------- |
| cp1      |  1001 | team12       | `172.17.128.21` | `10.10.10.51` | Control Plane |
| cp2      |  1002 | team13       | `172.17.128.22` | `10.10.10.52` | Control Plane |
| cp3      |  1003 | team14       | `172.17.128.23` | `10.10.10.53` | Control Plane |
| worker1  |  1011 | team12       | `172.17.128.24` | `10.10.10.54` | Worker        |
| worker2  |  1012 | team13       | `172.17.128.25` | `10.10.10.55` | Worker        |
| worker3  |  1013 | team14       | `172.17.128.26` | `10.10.10.56` | Worker        |
| worker4  |  1014 | team11       | `172.17.128.27` | `10.10.10.57` | Worker        |
| worker5  |  1015 | team11       | `172.17.128.28` | `10.10.10.58` | Worker        |
| worker6  |  1016 | team11       | `172.17.128.29` | `10.10.10.59` | Worker        |

# Kubernetes Pod ip 대역
172.20.0.0/16

# Kubernetes Service ip 대역
10.96.0.0/12 (Calico default)
