# Slurm Cheat Sheet

| Action                   | Command                                                                                                             |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Show GPU partition queue | `squeue -i 30 --partition gpu -o "%.18i %.9P %.40j %.8u %.8a %.8T %.6D %.4C %.5D %.6m %.13b %.10M %.10l %.28R"` |
| Account billing | `sreport -T billing cluster AccountUtilizationByUser start=mm/dd/yy end=mm/dd/yy account=test | sed -r 's/(.*billing\s+)([0-9]+)\b(.*)/echo "\1    \\$$((\2\/100000))\3"/ge'` |
| Account GPU-hours utilization | `sreport -t hours -T gres/gpu cluster AccountUtilizationByUser start=mm/dd/yy end=mm/dd/yy account=test` |
| Time until next walltime | ? |

Allocated GPU count:

```bash
echo Allocated GPUs: $(squeue -t RUNNING --partition gpu --noheader -o "%D %b" | cut -c 3-6 --complement | awk '{ print $1*$2 }' | awk '{s+=$1} END {print s}')/$(sinfo --partition gpu -N --states=alloc,idle,mix --noheader -o "%G" | cut -d : -f 3 | awk '{s+=$1} END {print s}')
```

GPU nodes resource utilization:

```bash
join -a 1 -e 0 -o auto <(sinfo --partition gpu -N --states=alloc,idle,mix --noheader -o "%8N %13C %8e %6m") <(squeue -t RUNNING --partition gpu --noheader -o "%N:%b" | awk -F : '{ m = gensub(/^(.*)\[(([[:digit:]]+),)?([[:digit:]]+)-([[:digit:]]+)\].*$/, "\\1-\\2-\\3-\\4-\\5", 1, $1); if(m ~ /-/) { split(m, ms, "-"); for (i = int(ms[4]); i <= int(ms[5]); i++) { print ms[1] i " " $3 }; if(ms[3]) { print ms[1] ms[3] " " $3 } } else { print $1 " " $3 } }' | awk '{ seen[$1] += $2 } END { for (i in seen) print i " " seen[i] }' | sort) | awk 'BEGIN{ printf("%6s %10s %9s %10s\n", "NODE", "ALLOC_CPUS", "ALLOC_MEM", "ALLOC_GPUS") }; { split($2, cpu, "/"); printf("%6s %7d/%2d %5.0f/%s %10s\n", $1, cpu[1], cpu[4], ($4 - $3)/1024, $4/1024, $5 "/2") }'
```
