# Slurm Cheat Sheet

| Action                              | Command                                                                                                             |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Show live GPU partition utilization | `squeue -i 30 --partition gpu -o "%.18i %.9P %.40j %.8u %.8a %.8T %.6D %.4C %.5D %.6m %.13b %.10M %.10l %.28R"`   |
| User GPU utilization per account    | `sreport cluster UserUtilizationByAccount start=mm/dd/yy end=mm/dd/yy account=test --tres gpu`                   |
