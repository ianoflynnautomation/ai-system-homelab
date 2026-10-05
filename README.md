# ai system homelab

[![GitOps: Flux CD](https://img.shields.io/badge/GitOps-Flux%20CD-5468ff)](https://fluxcd.io)
[![Runners: ARC 0.14.2](https://img.shields.io/badge/Runners-ARC%200.14.2-2088ff)](https://github.com/actions/actions-runner-controller)
[![Cluster: k3s](https://img.shields.io/badge/Cluster-k3s-ffc61c)](https://k3s.io)


# Architecture

```
Ubuntu host — installs k3s, helm, and the flux CLI. Nothing else.
│
├── flux-system         reconciles this repository, corrects drift
│
├── arc-systems         PSA: restricted
│   ├── arc controller  scoped to arc-runners only
│   └── listener        polls GitHub, scales runners 0 → maxRunners
│
├── arc-runners         PSA: baseline · NetworkPolicy · ResourceQuota
│   ├── runner pod      uid 1001, no capabilities, ephemeral
│   └── job pod         the workflow's container: — no token, bounded
│
├── ai-system           PSA: baseline · NetworkPolicy · ResourceQuota
│   ├── ollama          qwen2.5:3b, CPU only, 4 Gi guaranteed, 20 Gi PVC
│   ├── n8n             NodePort 30678, sqlite, 8 Gi PVC
│   └── open-webui      NodePort 30080, 8 Gi PVC
│
└── observability       PSA: privileged (required by node-exporter)
    ├── prometheus      7 d retention, 10 Gi local-path
    ├── grafana         grafana-operator, NodePort 30300
    └── node-exporter   node CPU, memory, thermal, disk
```

Build tooling lives in job container images, not on the host.

---

# Example workflow: scheduled AI job digest

![ai job digest](image.png)

A worked example of the `ai-system` stack doing something end to end: n8n on a
schedule, Ollama scoring text locally, and an email digest at the far end.

---

## What it does

Daily: read a résumé, derive job titles from it, scrape LinkedIn for those
titles, score every posting against the résumé with a local model, and email
whatever clears the bar.

```
Schedule Trigger
  → Edit Fields            résumé text (a literal, not a file)
  → Basic LLM Chain        résumé → {primary_titles, similar_titles, skills}
  → Parse Titles           that JSON → one item PER TITLE, capped at 5
  → Loop Over Items        batches titles; done[0] carries the scraped jobs
      ↳ main[1] → HTTP Request → Apify LinkedIn scraper → back into the loop
  → Trim Descriptions      bound the prompt (see Capacity)
  → Basic LLM Chain1       job + résumé → {score, fit_summary, recommendation}
  → Code in JavaScript     parse that JSON onto each item
  → Filter                 score >= 70 AND recommendation = Apply
  → Code in JavaScript1    build the HTML digest
  → Send an Email          Gmail SMTP
```

# Sample Email 

![Email sample](image-1.png)

---

## License

No license has been applied to this repository yet, so default copyright
applies and no reuse rights are granted. Add a `LICENSE` file to publish it
under explicit terms — `MIT` or `Apache-2.0` for permissive reuse — and update
this section to match.
