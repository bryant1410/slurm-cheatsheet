# Slurm Cheat Sheet

## Watch the GPU partition queue

```bash
squeue \
  -i 30 \
  --partition gpu \
  -o "%.18i %.9P %.40j %.8u %.8a %.8T %.6D %.4C %.5D %.6m %.13b %.10M %.10l %.28R"
```

## Show the GPU partition queue for an account

```bash
squeue \
  -A test \
  --partition gpu \
  -o "%.18i %.9P %.40j %.8u %.8a %.8T %.6D %.4C %.5D %.6m %.13b %.10M %.10l %.28R"
```

## Account billing

```bash
sreport \
  -T billing \
  cluster AccountUtilizationByUser \
    start=mm/dd/yy \
    end=mm/dd/yy \
    account=test \
| sed -r 's/(.*billing\s+)([0-9]+)\b(.*)/echo "\1 \\$$(echo scale=2\\; \2\/100000 \| bc)\3"/ge'
```

## Account GPU-hours utilization

```bash
sreport \
  -t hours \
  -T gres/gpu \
  cluster AccountUtilizationByUser \
    start=mm/dd/yy \
    end=mm/dd/yy \
    account=test
```

## Time until next walltime

???

## Allocated GPU count

```bash
echo Allocated GPUs: \
  $(squeue \
    -t RUNNING \
    --partition gpu \
    --noheader \
    -o "%D %b" \
  | cut -c 3-6 --complement \
  | awk '{ print $1*$2 }' \
  | awk '{s+=$1} END {print s}' \
  )/$(sinfo \
    --partition gpu \
    -N \
    --states=alloc,idle,mix \
    --noheader \
    -o "%G" \
  | cut -d : -f 3 \
  | awk '{s+=$1} END {print s}' \
  )
```

## GPU node resource utilization

```bash
join \
  -a 1 \
  -e 0 \
  -o auto \
  <(sinfo \
    --partition gpu \
    -N \
    --states=alloc,idle,mix \
    --noheader \
    -o "%8N %13C %8e %6m"
  ) \
  <(squeue \
    -t RUNNING \
    --partition gpu \
    --noheader \
    -o "%N %b %u %a" \
  | python3 -c "
import fileinput
import subprocess
from collections import defaultdict

# From https://github.com/NERSC/pytokio/blob/fdb8237/tokio/connectors/slurm.py#L81
def node_spec_to_list(node_spec):
  return subprocess.check_output(['scontrol', 'show', 'hostname', node_spec]).decode().strip().split()

nodes_info = defaultdict(lambda: {'gpus': 0, 'users': []})

for line in fileinput.input():
  line = line.strip()

  node_spec, gres, u, a = line.split()
  if gres.startswith('gpu:'):
    nodes = node_spec_to_list(node_spec)
    gpus = gres.split(':')[-1]

    for node in nodes:
      node_info = nodes_info[node]
      node_info['gpus'] += int(gpus)
      node_info['users'].extend(f'{u}({a})' for _ in gpus)

for node, node_info in sorted(nodes_info.items(), key=lambda t: t[0]):
  print(f\"{node} {node_info['gpus']} {','.join(node_info['users'])}\")
  ") \
| sed -e 's/ 0 0/ 0 /' \
| awk \
  'BEGIN{
    printf("%6s %10s %9s %10s %s\n", "NODE", "ALLOC_CPUS", "ALLOC_MEM", "ALLOC_GPUS", "USERS")
  };
  {
    split($2, cpu, "/");
    printf("%6s %7d/%2d %5.0f/%s %10s %s\n", $1, cpu[1], cpu[4], ($4 - $3)/1024, $4/1024, $5 "/2", $6)
  }'
```
