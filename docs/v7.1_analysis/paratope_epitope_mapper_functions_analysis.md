# Analysis of Paratope-Epitope Mapping Functions

This document provides a detailed analysis of the three additional functions designed for the `paratope_epitope_mapper.py` module: `split_ab_ag_importance`, `compute_epitope_importance_across_panel`, and `compute_paratope_importance`. These functions build upon the core `compute_integrated_gradients` function to provide a comprehensive suite of tools for analyzing antibody-antigen interactions.

## 1. `split_ab_ag_importance`

### Design and Purpose
This function is a critical utility that serves as the bridge between the raw, token-level Integrated Gradients (IG) scores and the scientifically meaningful residue-level importance scores for the antibody and antigen. The design is robust, correctly identifying that the tokenizer's internal offsets and special tokens are the key to reliably splitting the combined score array.

By encapsulating this logic into a dedicated function, the design promotes code reuse and separates the concern of score splitting from the more complex task of IG calculation.

### Implementation Recommendations

#### 1.1. Aggregating Token-Level Scores to Residue-Level
A crucial step not explicitly detailed in the function description is the aggregation of subword token scores into a single score for each amino acid residue. Modern tokenizers (like WordPiece or BPE) often split a single residue (e.g., "TRYPTOPHAN") into multiple tokens (e.g., "TRYP", "##TO", "##PHAN"). The IG scores will be at this subword level.

**Recommendation**: Implement a robust aggregation strategy to map token-level importance back to the residue level. A common and effective approach is to sum the IG scores of all tokens corresponding to a single residue.

```python
import numpy as np

def aggregate_token_scores_to_residues(token_scores, offsets):
    """Aggregates subword token scores to the residue level."""
    residue_scores = []
    current_residue_score = 0.0
    
    for i, (score, (start, end)) in enumerate(zip(token_scores, offsets)):
        # A new residue starts when the previous token did not end where the current one begins
        if i > 0 and offsets[i-1][1] != start:
            residue_scores.append(current_residue_score)
            current_residue_score = 0.0
        
        current_residue_score += score

    residue_scores.append(current_residue_score) # Append the last residue's score
    return np.array(residue_scores)

# The `split_ab_ag_importance` function would use this utility after splitting the raw IG scores.
```

This aggregation step is fundamental to ensuring the final output arrays (`ab_importance`, `ag_importance`) have the correct lengths (`L_Ab`, `L_Ag`) and that the scores accurately reflect the importance of each full amino acid.

## 2. `compute_epitope_importance_across_panel`

### Design and Purpose
This function is designed for a key scientific application: identifying conserved or variable epitopes by analyzing an antibody's binding footprint across a panel of related antigens (e.g., different viral variants). The design is excellent, correctly specifying that it should iterate through the panel, compute IG scores for each antigen, and then aggregate the results. The proposed output structure, containing both per-antigen scores and a panel-wide mean, is highly practical for downstream analysis.

### Implementation Recommendations

#### 2.1. Performance and Parallelization
This function is inherently computationally expensive, as it requires running the full `compute_integrated_gradients` calculation for every antigen in the panel. For a large panel, sequential execution would be prohibitively slow.

**Recommendation**: Parallelize the IG calculations across the antigen panel. Python's `concurrent.futures.ThreadPoolExecutor` or `ProcessPoolExecutor` can be used to distribute the workload across multiple CPU cores or, with careful management, multiple GPU devices.

```python
from concurrent.futures import ThreadPoolExecutor

def compute_epitope_importance_across_panel(...):
    # ...
    per_antigen_scores = {}
    
    def process_antigen(ag_id, ag_seq):
        # This function computes IG and splits to get the epitope score
        epitope_score = compute_and_split_ig(model_if, ab_seq, ag_seq, ...)
        return ag_id, epitope_score

    with ThreadPoolExecutor(max_workers=4) as executor: # Adjust max_workers as needed
        futures = [executor.submit(process_antigen, ag_id, ag_seq) 
                   for ag_id, ag_seq in ag_panel.items()]
        
        for future in futures:
            ag_id, epitope_score = future.result()
            per_antigen_scores[ag_id] = epitope_score
    
    # ... proceed to calculate panel_mean ...
```

#### 2.2. Handling Sequence Length Variation
When calculating the `panel_mean`, it is critical to handle variations in the lengths of the antigen sequences. A simple mean is only possible if all epitope importance vectors have the same length.

**Recommendation**: Before averaging, the epitope importance vectors must be aligned. This typically involves performing a multiple sequence alignment (MSA) on the antigen panel sequences. The importance scores can then be mapped onto the aligned sequences, with gaps assigned a score of zero. The mean can then be computed for each column in the alignment.

## 3. `compute_paratope_importance`

### Design and Purpose
This function provides a focused analysis of the antibody's binding surface, leveraging existing CDR annotations to highlight the importance of these critical regions. The design is a prime example of good modularity, correctly identifying that it should consume the `cdr_annotation` provided by the `Antibody research module` rather than re-implementing that logic. The proposed output, which breaks down importance by CDR, is extremely useful for antibody engineering efforts.

### Implementation Recommendations

#### 3.1. Configurable Blending of Importance Metrics
The design note mentions the possibility of combining IG scores with attention-based importance. This is a powerful idea, as different methods can provide complementary insights.

**Recommendation**: Make this blending an optional and configurable feature. The function should accept a parameter, `blending_alpha`, which controls the weight given to the IG scores versus the attention scores. A value of `1.0` (the default) would use only IG, `0.0` would use only attention, and `0.5` would be an equal blend.

```python
def compute_paratope_importance(
    ...,
    attention_scores: np.ndarray = None, # Optional attention scores
    blending_alpha: float = 1.0
) -> dict:
    # ... compute ab_importance using IG ...
    
    if attention_scores is not None and 0.0 <= blending_alpha < 1.0:
        # Ensure scores are normalized before blending
        final_ab_importance = (blending_alpha * ab_importance) + 
                              ((1 - blending_alpha) * attention_scores)
    else:
        final_ab_importance = ab_importance

    # ... proceed to slice by CDR using final_ab_importance ...
```

This parameter could be exposed in the main `paratope_epitope.yaml` configuration file.

## 4. Conclusion

These three functions form a cohesive and powerful analysis suite that logically extends the core IG calculation. They are well-designed, modular, and focused on providing scientifically meaningful outputs.

By implementing them with the recommendations provided—**residue-level score aggregation**, **parallelization for panel analysis**, **handling of variable sequence lengths**, and **configurable blending of metrics**—the `paratope_epitope_mapper.py` module will be a robust, performant, and highly effective tool for dissecting the molecular details of antibody-antigen recognition. The clear definition of attachment nodes and dependencies ensures that the module will integrate seamlessly into the broader BEREAN Protocol architecture.
