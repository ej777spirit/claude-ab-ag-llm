# Analysis of `paratope_epitope_mapper.py` Design

This document provides a detailed analysis of the design for the `paratope_epitope_mapper.py` module, the core engine for paratope-epitope inference in BEREAN Protocol v7.1. The design centers on using **Integrated Gradients (IG)** to attribute binding scores to individual residues, a scientifically robust and well-regarded approach for model interpretability.

## 1. Design and Methodology Assessment

The proposed design is excellent, demonstrating a clear and effective strategy for dissecting the complex interactions within the antibody-antigen binding interface. The choice of Integrated Gradients is particularly commendable.

### 1.1. Strengths of the Integrated Gradients Approach
Integrated Gradients is a powerful feature attribution method that is well-suited for this problem [1]. Its key advantages include:

- **Sensitivity**: It properly attributes importance to features, even when the model's output is saturated.
- **Implementation Invariance**: The method is not tied to any specific model architecture, making it a perfect fit for the model-agnostic `ModelInterface`.
- **Axiomatic Foundation**: It adheres to important axioms (like Completeness) that ensure the attributions are a faithful explanation of the model's prediction.

By using IG, the design ensures that the inferred paratope and epitope scores are a reliable and scientifically sound representation of the model's internal decision-making process.

### 1.2. Comprehensive and Modular Functionality
The module's purpose is clearly defined, encompassing a full suite of analysis capabilities:

- **Core IG Calculation**: The foundation of the module.
- **Paratope/Epitope Splitting**: The essential step of separating the combined IG scores.
- **Panel-based Importance Mapping**: Crucial for understanding conservation and variability across related antigens.
- **Synergy Matrix**: A sophisticated and powerful feature for analyzing functional interactions between residues.

This comprehensive scope, combined with the module's independence from the core model implementation, makes the design both powerful and maintainable.

## 2. Implementation Recommendations

Translating this excellent design into a robust implementation requires careful consideration of several technical details. The following recommendations are intended to guide the implementation of the `compute_integrated_gradients` function and the broader module.

### 2.1. Implementing `compute_integrated_gradients`

The core of the IG calculation involves integrating the gradients of the model's output with respect to its input embeddings along a straight-line path from a baseline.

**Recommendation**: Leverage a library like **Captum** [2], which provides a production-ready and highly optimized implementation of Integrated Gradients and other attribution algorithms for PyTorch models.

Here is a conceptual implementation using Captum:

```python
import torch
import numpy as np
from captum.attr import IntegratedGradients

# Assumes ModelInterface and other necessary imports

def compute_integrated_gradients(
    model_if: ModelInterface,
    ab_seq: str,
    ag_seq: str,
    head: str = "binding",
    steps: int = 32,
) -> tuple[np.ndarray, np.ndarray, np.ndarray]:

    # 1. Define a custom forward function for Captum
    # This function must take embeddings as input and return a scalar score.
    def model_forward(inputs_embeds: torch.Tensor, attention_mask: torch.Tensor) -> torch.Tensor:
        # Get the full model output using the provided embeddings
        outputs = model_if.model(inputs_embeds=inputs_embeds, attention_mask=attention_mask)
        
        # Select the appropriate score based on the 'head' parameter
        # This part needs to be implemented based on the model's output structure
        score = select_score_from_output(outputs, head)
        return score

    # 2. Prepare inputs and baseline
    tokenized_inputs = model_if.tokenize_fn(ab_seq, ag_seq)
    input_ids = tokenized_inputs["input_ids"].to(model_if.device)
    attention_mask = tokenized_inputs["attention_mask"].to(model_if.device)

    # The baseline is typically a tensor of zeros with the same shape as the input embeddings
    ref_token_id = model_if.tokenizer.pad_token_id # Or another appropriate baseline
    ref_input_ids = torch.full_like(input_ids, ref_token_id)

    # 3. Get input embeddings
    input_embeddings = model_if.model.get_input_embeddings()(input_ids)
    baseline_embeddings = model_if.model.get_input_embeddings()(ref_input_ids)

    # 4. Initialize Integrated Gradients with the custom forward function
    ig = IntegratedGradients(model_forward)

    # 5. Compute attributions
    attributions, delta = ig.attribute(
        inputs=input_embeddings,
        baselines=baseline_embeddings,
        additional_forward_args=(attention_mask,),
        n_steps=steps,
        return_convergence_delta=True
    )

    # Summarize attributions across the embedding dimension
    ig_scores = attributions.sum(dim=-1).squeeze(0).cpu().numpy()

    # Return scores and token info for splitting
    token_info = tokenized_inputs.get("token_type_ids", None)
    return ig_scores, input_ids.squeeze(0).cpu().numpy(), token_info
```

### 2.2. Splitting Paratope and Epitope Scores
Once the combined IG scores are computed, they must be split. The `token_type_ids` (or a similar mechanism from the tokenizer) are essential for this.

**Recommendation**: Use the `token_type_ids` to create boolean masks for the antibody and antigen tokens. These masks can then be used to select the corresponding scores from the `ig_scores` array.

### 2.3. Implementing the Synergy Matrix
The synergy matrix (functional interaction map) is a more advanced analysis that cannot be computed with a single IG pass. It requires a perturbation-based approach.

**Recommendation**: Implement a separate function, `compute_synergy_matrix`, that performs in-silico double perturbations. The algorithm would be:
1.  Get the baseline binding score for the original sequences.
2.  Iterate through all pairs of residues (one from the paratope, one from the epitope).
3.  For each pair, perform a double mutation (e.g., to Alanine).
4.  Calculate the new binding score.
5.  The synergy score for the pair is `score(WT) - score(mut)`. A large positive value indicates a critical interaction.

This is computationally expensive and should be used judiciously.

## 3. Proposed Class Structure

To organize the different analysis methods within the module, a class-based structure is recommended.

**Recommendation**: Create a `ParatopeEpitopeMapper` class that is initialized with the `ModelInterface`.

```python
class ParatopeEpitopeMapper:
    def __init__(self, model_if: ModelInterface):
        self.model_if = model_if

    def compute_integrated_gradients(self, ...):
        # ... implementation from section 2.1 ...

    def get_paratope_epitope_scores(self, ab_seq: str, ag_seq: str, ...):
        ig_scores, input_ids, token_info = self.compute_integrated_gradients(...)
        # ... logic to split scores ...
        return paratope_scores, epitope_scores

    def compute_synergy_matrix(self, ab_seq: str, ag_seq: str, ...):
        # ... implementation of the double perturbation method ...
        return synergy_matrix
```

This structure provides a clean, object-oriented API for the rest of the application.

## 4. Conclusion

The design for the `paratope_epitope_mapper.py` module is robust, scientifically sound, and well-aligned with the overall architecture of BEREAN Protocol v7.1. By leveraging Integrated Gradients, the module will provide powerful and reliable insights into the mechanisms of antibody-antigen binding.

The recommendations provided here—particularly the use of a library like Captum for the core IG calculation and the adoption of a class-based structure—will help ensure that the implementation is efficient, maintainable, and extensible. This module represents a significant step forward for the BEREAN platform, enabling a deeper level of analysis that will be invaluable for antibody engineering and discovery.

---

### References
[1] Sundararajan, M., Taly, A., & Yan, Q. (2017). Axiomatic Attribution for Deep Networks. *Proceedings of the 34th International Conference on Machine Learning*, 70, 3319-3328. Available from https://proceedings.mlr.press/v70/sundararajan17a.html

[2] Kokhlikyan, N., et al. (2020). Captum: A unified and generic model interpretability library for PyTorch. *arXiv preprint arXiv:2009.07896*. Available from https://arxiv.org/abs/2009.07896
