---
layout: page
permalink: /ipfs/
title: IPFS
---

# IPFS

There are four main IPFS nodes that I operate, which act as public gateways for a number of websites. You can add one or more of these nodes [as peers](https://bafybeietnh3jswlmwsg3dwvmhqmv5qloda6yrchxdztxnrntp5asaeuyzm.ipfs.aragon.ventures/how-to/peering-with-content-providers/) for faster access to content I'm hosting, or use these nodes as an HTTP gateway to IPFS. These gateways have unrestricted/wildcard CORS headers set, and provide subdomain content isolation.

| Hostname | Peer ID | Gateway | Gateway CA |
| -------- | ------- | ------- | -- |
| `/dnsaddr/ipfs-us-il.aragon.ventures` | `12D3KooWJgWWbtczgsQEXjUoy7WzEgtHz1o8oWjMtxJwq6ArEidj` | [ipfs.aragon.ventures](https://aragon.ventures/ipfs/bafybeifx7yeb55armcsxwwitkymga5xf53dxiarykms3ygqic223w5sk3m#x-ipfs-companion-no-redirect) | Let's Encrypt |
| `/dnsaddr/ipfs-us-ca.aragon.ventures` | `12D3KooWGU3REfsZSSSdAjDzvcd192ibgux3Fnqp9eqD6mez486E` | [ipfs-us-ca.aragon.ventures](https://ipfs-us-ca.aragon.ventures/ipfs/bafybeifx7yeb55armcsxwwitkymga5xf53dxiarykms3ygqic223w5sk3m#x-ipfs-companion-no-redirect) | ZeroSSL |
| `/dnsaddr/ipfs-us-ny.aragon.ventures` | `12D3KooWCefgj17peLNt2tUoned2qk2yZTcj4vXsfHzjVcyo3QmL` | [ipfs-us-ny.aragon.ventures](https://ipfs-us-ny.aragon.ventures/ipfs/bafybeifx7yeb55armcsxwwitkymga5xf53dxiarykms3ygqic223w5sk3m#x-ipfs-companion-no-redirect) | Let's Encrypt |
| `/dnsaddr/ipfs-nl-ams.aragon.ventures` | `12D3KooWFkKngZDxqhLr7ZJATGqTGU9cXoALk53PWGt3Ho9nz52f` | [ipfs-nl-ams.aragon.ventures](https://ipfs-nl-ams.aragon.ventures/ipfs/bafybeifx7yeb55armcsxwwitkymga5xf53dxiarykms3ygqic223w5sk3m#x-ipfs-companion-no-redirect) | Let's Encrypt |

Additionally, [ipfs-cdn.aragon.ventures](https://ipfs-cdn.aragon.ventures/ipfs/bafybeifx7yeb55armcsxwwitkymga5xf53dxiarykms3ygqic223w5sk3m#x-ipfs-companion-no-redirect) (Let's Encrypt) is a public gateway backed by a global Content Delivery Network which can be used for faster access to files. This gateway **does not** support origin isolation and is therefore only suitable for grabbing static content, not for grabbing sensitive data or for accessing dapps/websites where you would have to enter credentials. Basically, don't use it as the main public gateway in a web browser, but it's good for embedding files in a website.

Use these public gateways in a reasonable manner, abuse of the network will result in throttling or blocking as necessary.

**These public gateways are not data storage providers or website hosts. The gateways above allow users to view content hosted by third parties over which I exercise no control. The fact that certain content is viewable through these gateways does not mean it is hosted by the gateway or that I can do anything to delete that content.**

Because I am not hosting third-party data with these gateways, I cannot remove material accessible through these gateways from the internet. If you believe that material accessible through these gateways is illegal or violates your copyright, you are encouraged to directly notify whoever is hosting or controls that data.

While I do not host any of the data on these gateways, in appropriate circumstances I can disable the ability to view certain content via these gateways. This does not mean that the data itself has been deleted or removed from the network, it simply will not be accessible using the public gateways above. This also will not impact the availability of the data through other gateways run by other parties.

Report abuse by emailing me at jonah@triplebit.net. When appropriate, I will disable access through these gateways to the specific content set forth in your abuse report.
