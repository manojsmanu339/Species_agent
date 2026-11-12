# Species Agent

Identify **animals, plants and fungi** from a photo (species, genus, family, ... kingdom) -
and **keep learning** from labelled photos you give it.

* **Zero-shot brain:** [BioCLIP](https://github.com/Imageomics/bioclip) (Tree of Life, ~450k taxa).
* **Learning memory:** your labelled photos are stored as embeddings. No retraining, learning
  takes milliseconds, nothing is forgotten, memory per class is capped.
* **Safe:** if a photo looks unlike anything you taught, the agent falls back to zero-shot.

## Install from GitHub

```bash
pip install "git+https://github.com/<YOUR_USER>/<YOUR_REPO>.git"
# with the web UI:
pip install "species-agent[ui] @ git+https://github.com/<YOUR_USER>/<YOUR_REPO>.git"
```

or clone and use it locally:

```bash
git clone https://github.com/<YOUR_USER>/<YOUR_REPO>.git && cd <YOUR_REPO>
pip install -e ".[ui]"
```

## Website

```bash
pip install "species-agent[web] @ git+https://github.com/<YOUR_USER>/<YOUR_REPO>.git"
species-agent serve                  # -> http://127.0.0.1:8000
```

Three pages, one clean interface (works in light and dark mode, on phones too):

| Tab | What you can do |
|---|---|
| **Identify** | Drop a photo -> name, species, genus, full classification, confidence, alternatives. Wrong? Type the right name and the agent learns it instantly. |
| **Teach** | **1.** Upload images, `.zip` / `.tar.gz` archives or CSV files (large files are streamed, with upload progress). **2.** Or paste links: image, archive or CSV-of-image-links, downloaded server-side with a progress bar. **3.** Name the species (label). |
| **Knowledge** | Every species learned and how many photos each is based on. |

### Website structure

```
species_agent/
  server.py          FastAPI backend (API + serves the website)
  ingest.py          safe unpacking of uploads / archives / CSV / internet downloads
  agent.py memory.py embedder.py cli.py
  web/
    index.html       page structure
    css/style.css    design (tokens, light + dark)
    js/app.js        behaviour (no build step, no frameworks)
    favicon.svg
```

API: `POST /api/identify` · `POST /api/train` · `GET /api/jobs/{id}` · `GET /api/stats` ·
interactive docs at `/api/docs`.

Limits (override with env vars, in MB): `SPECIES_AGENT_MAX_UPLOAD_MB` (2048),
`SPECIES_AGENT_MAX_URL_MB` (4096), `SPECIES_AGENT_MAX_EXTRACT_MB` (8192).

**Security notes.** The server binds to `127.0.0.1`. Archives are extracted with path checks and
a size budget (images + `taxonomy.csv` only); links must be public `http(s)` addresses
(localhost and private networks are refused). There is no login: if you deploy it publicly,
put it behind authentication, because anyone who can reach `/api/train` can teach the agent.

## Use

```bash
species-agent identify photo.jpg                 # or several files / a folder
species-agent identify photos/ --json
species-agent ui                                 # minimal Gradio demo (pip install gradio)
```

## Teach it (the training data template)

Copy `train_data_template/` to `train_data/` and fill it:

```
train_data/
  taxonomy.csv              # optional: label,common_name,species,genus,family,...
  Panthera_leo/    *.jpg    # folder name = label
  Quercus_robur/   *.jpg
```

Use scientific names as folder names (`Panthera_leo`) so learned classes merge with the
zero-shot model's species. 10-50 varied photos per class works well.

```bash
species-agent train train_data/                  # learn (incremental, safe to re-run)
species-agent feedback wrong.jpg "Panthera leo"  # fix one mistake -> agent learns it
species-agent evaluate test_data/                # same layout, NEW photos: shows the gain
species-agent stats
```

`evaluate` prints zero-shot accuracy vs. accuracy with learning, so you can *measure*
the improvement as you add data. Knowledge is saved to `knowledge/knowledge.npz`
(override with `--kb` or `SPECIES_AGENT_KB`). Commit that file to share what the agent
learned; anyone who installs the repo gets it.

## Python API

```python
from species_agent import SpeciesAgent

agent = SpeciesAgent("knowledge/knowledge.npz")
agent.train_folder("train_data")
r = agent.identify("photo.jpg")
print(r.best.common_name, r.best.species, r.best.genus, r.best.confidence)
agent.learn("photo.jpg", "Panthera leo")   # teach from feedback
```

## How the learning works

Each new photo is embedded by BioCLIP and added to a per-class prototype + up to 30 stored
exemplars. At prediction time: `score = (1-b)*zero_shot + b*memory`, where
`b = n/(n+2)` grows with the number of photos taught for that class. Everything is O(1)
per photo and fixed-size per class.

## Limits

Accuracy depends on photo quality and on teaching varied, correctly labelled examples.
For safety-critical identification (mushrooms, berries, snakes) confirm with an expert.
