# slurm-cheatsheet

| Action                              | Command                                                                                              |
| ----------------------------------- | ------------------------------------------------------------------------------------------------    |
| Show live GPU partition utilization | `squeue -i 30 --partition gpu -o "%.18i %.9P %.40j %.8u %.8T %.10M %.9l %.6D %R %C %S %D %m %Y"`   |
| User GPU utilization per account    | `sreport cluster UserUtilizationByAccount start=mm/dd/yy end=mm/dd/yy account=test --tres gpu`    |
