# Slurm Cheat Sheet

| Action                              | Command                                                                                                             |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Show GPU partition queue         | `squeue -i 30 --partition gpu -o "%.18i %.9P %.40j %.8u %.8a %.8T %.6D %.4C %.5D %.6m %.13b %.10M %.10l %.28R"`   |
| User GPU utilization per account | `sreport cluster UserUtilizationByAccount start=mm/dd/yy end=mm/dd/yy account=test --tres gpu`                        |
| GPU node resource utilization    | `sinfo --partition gpu -N --states=alloc,idle,mix -o "%8N %13C %8O %8e %6m"`                                           |
| Time until next walltime         | ?                                                                                                                     |

Allocated GPU count:

```bash
echo Allocated GPUs: $(squeue -t RUNNING --partition gpu --noheader -o "%D %b" | cut -c 3-6 --complement | awk '{ print $1*$2 }' | awk '{s+=$1} END {print s}')/$(sinfo --partition gpu -N --states=alloc,idle,mix --noheader -o "%G" | cut -d : -f 3 | awk '{s+=$1} END {print s}')
```

Nodes GPU usage:

```bash
squeue -t RUNNING --partition gpu --noheader -o "%N:%b" | awk -F : '{ seen[$1] += $3 } END { for (i in seen) print i ":" seen[i] }' | sort
```
