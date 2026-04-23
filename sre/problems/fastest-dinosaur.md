# The Fastest Dinosaur (Join Two Data Files)

**Difficulty:** Easy–Medium
**Topic:** [[sre/topics/linux-cli]]
**Pattern:** File Parsing / Hash Map Join / Data Munging
**Companies:** [[sre/companies/meta]]

## Problem

This is a famous Meta Production Engineer interview problem.

You are given two CSV files:

**`dinosaurs.csv`** (name, stride length in meters, stance: `biped` or `quadruped`):
```
NAME,LEG_LENGTH,DIET
Hadrosaurus,1.4,herbivore
Struthiomimus,0.72,omnivore
Velociraptor,1.8,carnivore
Triceratops,0.47,herbivore
Euoplocephalus,0.86,herbivore
Stegosaurus,1.40,herbivore
Tyrannosaurus Rex,2.5,carnivore
```

**`dataset2.csv`** (name, stride length in meters, stance: `biped` or `quadruped`):
```
NAME,STRIDE_LENGTH,STANCE
Euoplocephalus,1.87,quadruped
Stegosaurus,1.87,quadruped
Tyrannosaurus Rex,4.76,biped
Hadrosaurus,1.3,biped
Deinonychus,1.21,biped
Struthiomimus,1.78,biped
Velociraptor,2.62,biped
```

A dinosaur's **speed** = `(stride_length / leg_length - 1) × √(leg_length × 9.8)`.

**Output:** The name of the **fastest bipedal** dinosaur, sorted alphabetically if there is a tie.

## Why This Is An Interview Problem

Meta uses this to test:
1. **File joining** on a common key — the production equivalent of SQL JOIN.
2. **Filtering** a dataset by a condition.
3. **Applying a formula** to compute a derived field.
4. **Finding the extremum** of the derived field.
5. **Clean, readable Python** using `csv`, `dict`, and standard library only.

The data itself is whimsical, but the pattern is serious: joining log files, merging metrics from two sources, combining CSVs from different systems.

## Approach

1. Read `dinosaurs.csv` → build `name → leg_length` dict. (Only keep what we need.)
2. Read `dataset2.csv` → for each row, filter to `biped` only. Join with leg_length.
3. Compute speed from formula.
4. Track the maximum speed and corresponding name.

## Solution (Python)

```python
import csv
import math
import sys

def compute_speed(leg_length: float, stride_length: float) -> float:
    """Speed formula: (stride/leg - 1) * sqrt(leg * g)"""
    g = 9.8
    return (stride_length / leg_length - 1) * math.sqrt(leg_length * g)

def fastest_biped(dino_file: str, stride_file: str) -> str:
    # Pass 1: build leg-length lookup from dinosaur file
    leg_lengths: dict[str, float] = {}
    with open(dino_file, newline='') as f:
        reader = csv.DictReader(f)
        for row in reader:
            try:
                leg_lengths[row['NAME']] = float(row['LEG_LENGTH'])
            except (ValueError, KeyError):
                pass  # skip malformed rows

    # Pass 2: compute speed for bipeds; find the fastest
    fastest_name = None
    fastest_speed = float('-inf')

    with open(stride_file, newline='') as f:
        reader = csv.DictReader(f)
        for row in reader:
            if row.get('STANCE', '').strip().lower() != 'biped':
                continue
            name = row['NAME']
            if name not in leg_lengths:
                continue  # can't compute speed without leg length

            try:
                stride = float(row['STRIDE_LENGTH'])
                leg = leg_lengths[name]
                if leg <= 0:
                    continue  # guard against division by zero / bad data
                speed = compute_speed(leg, stride)
            except (ValueError, KeyError):
                continue

            if speed > fastest_speed:
                fastest_speed = speed
                fastest_name = name

    return fastest_name or "No bipedal dinosaurs found"

if __name__ == "__main__":
    dino_csv   = sys.argv[1] if len(sys.argv) > 1 else "dinosaurs.csv"
    stride_csv = sys.argv[2] if len(sys.argv) > 2 else "dataset2.csv"
    print(fastest_biped(dino_csv, stride_csv))
```

## Expected Output

For the sample data above:
```
Tyrannosaurus Rex
```

## Solution (One-liner Pipeline for Verification)

```bash
# Quick verification using join command (assumes sorted files)
join -t, -1 1 -2 1 \
     <(sort dinosaurs.csv) \
     <(sort dataset2.csv | grep biped) \
  | awk -F, 'NR>1 { leg=$2; stride=$4; speed=(stride/leg - 1)*sqrt(leg*9.8); print speed, $1 }' \
  | sort -rn | head -1
```

## Complexity

- **Time:** O(D + S) where D = rows in dinosaurs file, S = rows in stride file.
- **Space:** O(D) for the leg-length dict.

## Key Insight

This is a **hash join**: load the smaller table into memory as a dict, then stream the larger table and probe the dict. The same pattern scales to joining TB log files: load the smaller reference table (dim table), stream the larger event table (fact table).

If both files are too large for memory, the approach changes to:
1. Sort both files by the join key.
2. Merge-join the two sorted streams.

## Variants / Follow-ups

- "Return the top 3 fastest bipeds" → maintain a min-heap of size 3.
- "What if both files are 10GB?" → Sort-merge join. The dinosaur file (metadata) is usually small; load it all. Only the stride file needs streaming.
- "What if names don't match exactly (case differences, trailing whitespace)?" → Normalize keys: `name.strip().lower()`.
- "What if you need to update the CSV and write results back?" → Write to a new output CSV using `csv.DictWriter`; never modify the input in-place.
- "Do this with SQL instead?" → `SELECT d.NAME, (s.STRIDE_LENGTH / d.LEG_LENGTH - 1) * sqrt(d.LEG_LENGTH * 9.8) AS speed FROM dinosaurs d JOIN strides s ON d.NAME = s.NAME WHERE s.STANCE = 'biped' ORDER BY speed DESC LIMIT 1`

## Connection to Real PE Work

| Interview scenario | Real-world equivalent |
|---|---|
| Join two CSV files | Join a metrics file with a service registry |
| Filter by `biped` | Filter by `region='us-east-1'` or `status='error'` |
| Compute speed formula | Compute p99 latency from raw timing data |
| Find maximum | Find the host with highest memory utilization |
| Handle missing data | Handle services missing from the registry |

## Sources
- [[sre/companies/meta]]
