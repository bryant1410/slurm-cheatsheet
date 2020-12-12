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
    -o "%8N %13C %8e %6m %10G"
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
  nodes = node_spec_to_list(node_spec)
  gpus = gres.split(':')[-1] if gres.startswith('gpu:') else '0'

  for node in nodes:
    node_info = nodes_info[node]
    try:
      node_info['gpus'] += int(gpus)
    except ValueError:
      pass
    node_info['users'].extend(f'{u}({a})' for _ in gpus)

for node, node_info in sorted(nodes_info.items(), key=lambda t: t[0]):
  print(f\"{node} {node_info['gpus']} {','.join(node_info['users'])}\")
  ") \
| sed -e 's/ 0 0/ 0 /' \
| awk \
  'BEGIN{
    total_cpus_alloc = 0;
    total_cpus = 0;
    total_mem_alloc = 0;
    total_mem = 0;
    total_gpus_alloc = 0;
    total_gpus = 0;

    printf("%6s %10s %9s %10s %s\n", "NODE", "ALLOC_CPUS", "ALLOC_MEM", "ALLOC_GPUS", "USERS")
  };
  {
    split($2, cpu, "/");
    split($5, gres, ":");

    node = $1;

    cpus_alloc = cpu[1];
    total_cpus_alloc += cpus_alloc;

    cpus = cpu[4];
    total_cpus += cpus;

    mem_alloc = ($4 - $3) / 1024;
    total_mem_alloc += mem_alloc;

    mem = $4 / 1024;
    total_mem += mem;

    gpus_alloc = $6;
    total_gpus_alloc += gpus_alloc;

    gpus = gres[3];
    total_gpus += gpus;

    users = $7;

    printf("%6s %6d/%3d %4d/%4d %7d/%2d %s\n", node, cpus_alloc, cpus, mem_alloc, mem, gpus_alloc, gpus, users)
  };
  END{
    printf("%6s %6d/%3d %4d/%4d %7d/%2d\n", "TOTAL", total_cpus_alloc, total_cpus, total_mem_alloc, total_mem, total_gpus_alloc, total_gpus)
  }'
```
