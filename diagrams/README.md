# Diagrams

The PNGs in this folder are **generated** from the Mermaid sources in
[`src/`](src). Edit the `.mmd` source, then regenerate — do not hand-edit the PNGs.

| Source | Output | Used in |
|--------|--------|---------|
| `src/topology.mmd` / `src/topology.ko.mmd` | `topology.png` / `topology.ko.png` | README, architecture-overview, network-topology |
| `src/flow-company-to-idc.mmd` / `.ko.mmd` | `flow-company-to-idc.png` / `.ko.png` | packet-flow |
| `src/flow-idc-to-company.mmd` / `.ko.mmd` | `flow-idc-to-company.png` / `.ko.png` | packet-flow |
| `src/firewall-layers.mmd` / `.ko.mmd` | `firewall-layers.png` / `.ko.png` | architecture-overview |
| `src/source-nat.mmd` / `.ko.mmd` | `source-nat.png` / `.ko.png` | network-topology |
| `src/troubleshooting-tree.mmd` / `.ko.mmd` | `troubleshooting-tree.png` / `.ko.png` | troubleshooting |

## Regenerate

Requires [`@mermaid-js/mermaid-cli`](https://github.com/mermaid-js/mermaid-cli)
(`npm install -g @mermaid-js/mermaid-cli`).

```bash
cd diagrams
for f in src/*.mmd; do
  mmdc -i "$f" -o "$(basename "${f%.mmd}").png" -b white -s 3
done
```
