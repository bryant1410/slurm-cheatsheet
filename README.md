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
gpu_usage() {
  if [ $# -eq 0 ]; then
    echo "The partition name needs to be supplied"
    return 1
  fi

  join \
    -a 1 \
    -o auto \
    <(sinfo \
      --partition $1 \
      --Node \
      --noheader \
      -O NodeList,CPUsState,AllocMem,Memory,Gres,StateCompact,GresUsed
    ) \
    <(squeue \
      -t RUNNING \
      --partition $1 \
      --noheader \
      -o "%N %u %a" \
    | python -c "
import fileinput
import subprocess
from collections import defaultdict

# From https://github.com/NERSC/pytokio/blob/fdb8237/tokio/connectors/slurm.py#L81
def node_spec_to_list(node_spec):
  return subprocess.check_output(['scontrol', 'show', 'hostname', node_spec]).decode().strip().split()

users_by_node = defaultdict(list)

for line in fileinput.input():
  line = line.strip()

  node_spec, u, a = line.split()
  
  for node in node_spec_to_list(node_spec):
    users_by_node[node].append('{}({})'.format(u, a))

for node, node_info in sorted(users_by_node.items(), key=lambda t: t[0]):
  print('{} {}'.format(node, ','.join(node_info)))
    ") \
  | awk \
    'BEGIN{
      total_cpus_alloc = 0;
      total_cpus = 0;
      total_mem_alloc = 0;
      total_mem = 0;
      total_gpus_alloc = 0;
      total_gpus = 0;

      printf("%6s %5s %11s %11s %11s %s\n", "NODE", "STATE", "ALLOC_CPUS", "ALLOC_MEM", "ALLOC_GPUS", "USERS")
    };
    {
      split($2, cpu, "/");
      split($5, gres, ":");
      split($7, gres_used, ":|\\(");

      node = $1;

      state = $6;

      cpus_alloc = cpu[1];
      total_cpus_alloc += cpus_alloc;

      cpus = cpu[4];
      total_cpus += cpus;

      mem_alloc = $3 / 1024;
      total_mem_alloc += mem_alloc;

      mem = $4 / 1024;
      total_mem += mem;

      gpus_alloc = gres_used[3];
      total_gpus_alloc += gpus_alloc;

      gpus = gres[3];
      total_gpus += gpus;

      users = $8;

      printf("%6s %5s %6d/%4d %5d/%5d %7d/%3d %s\n", node, state, cpus_alloc, cpus, mem_alloc, mem, gpus_alloc, gpus, users)
    };
    END{
      printf("%6s %5s %6d/%4d %5d/%5d %7d/%3d\n", "TOTAL", "", total_cpus_alloc, total_cpus, total_mem_alloc, total_mem, total_gpus_alloc, total_gpus)
    }'
  pending_jobs=$(squeue --partition $1 -t PENDING --noheader)
  if [ ! -z "$pending_jobs" ]; then
    echo
    echo Pending jobs:
    echo "$pending_jobs"
  fi
}

# Usage: gpu_usage $PARTITION_NAME
```
