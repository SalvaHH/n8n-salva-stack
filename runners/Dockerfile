FROM n8nio/runners:2.42.7

USER root

# pacchetti Python che vuoi usare nei Code node
'N8N_RUNNERS_EXTERNAL_ALLOW':'*'

# patch del config del launcher: allowlist dentro il JSON (le env var da sole non bastano)
RUN /opt/runners/task-runner-python/.venv/bin/python -c "import json;p='/etc/n8n-task-runners.json';c=json.load(open(p));[r.setdefault('env-overrides',{}).update({'N8N_RUNNERS_STDLIB_ALLOW':'*','N8N_RUNNERS_EXTERNAL_ALLOW':'pandas,numpy,requests'}) for r in c['task-runners'] if r['runner-type']=='python'];json.dump(c,open(p,'w'),indent=2)"

USER runner
