# Slurm Cheat Sheet

| Action                   | Command                                                                                                             |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Show GPU partition queue | `squeue -i 30 --partition gpu -o "%.18i %.9P %.40j %.8u %.8a %.8T %.6D %.4C %.5D %.6m %.13b %.10M %.10l %.28R"` |
| Account GPU utilization  | `sreport cluster UserUtilizationByAccount start=mm/dd/yy end=mm/dd/yy account=test --tres gpu`                   |
| Time until next walltime | ?                                                                                                               |

Allocated GPU count:

```bash
echo Allocated GPUs: $(squeue -t RUNNING --partition gpu --noheader -o "%D %b" | cut -c 3-6 --complement | awk '{ print $1*$2 }' | awk '{s+=$1} END {print s}')/$(sinfo --partition gpu -N --states=alloc,idle,mix --noheader -o "%G" | cut -d : -f 3 | awk '{s+=$1} END {print s}')
```

GPU nodes resource utilization:

```bash
(echo 'NODELIST CPUS(A/I/O/T) CPU_LOAD FREE_MEM MEMORY ALLOC_GPUS' && join -a 1 -e 0 -o auto <(sinfo --partition gpu -N --states=alloc,idle,mix --noheader -o "%8N %13C %8O %8e %6m") <(squeue -t RUNNING --partition gpu --noheader -o "%N:%b" | awk -F : '{ m = gensub(/^(.*)\[(.+)-(.+)\](.*)$/, "\\1-\\2-\\3", 1, $1); if(m ~ /-/) { split(m, ms, "-"); for (i = int(ms[2]); i <= int(ms[3]); i++) { print ms[1] i " " $3 } } else { print $1 " " $3 } }'| awk '{ seen[$1] += $2 } END { for (i in seen) print i " " seen[i] }' | sort)) | awk '{ printf("%8s %13s %8s %8s %6s %10s\n", $1, $2, $3, $4, $5, $6) }'
```
