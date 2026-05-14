# LLM Cheatsheet

## Core Concepts
- Token: piece of text
- Embedding: vector representation
- Parameters: learned weights
- Context Window: max tokens model sees
- Inference: using model
- Training: updating weights

## Architecture
- Transformer: core model
- Attention: token relationships
- Layer: repeated block
- Head: parallel attention
- MLP: feedforward layer

## Training
- Loss: error
- Gradient: direction to improve
- Backpropagation: compute gradients
- Learning Rate: step size
- Batch: samples per step
- AdamW: optimizer

## Data
- Tokenization: text → tokens
- Dataset: training data
- Deduplication: remove repeats
- Data Ablation: test subsets

## Advanced
- RAG: retrieval + generation
- Fine-tuning: specialize model
- LoRA: efficient fine-tuning
- Quantization: reduce precision
- MoE: mixture of experts

## Evaluation
- Perplexity: model confidence
- Overfitting: memorization
- Hallucination: false outputs

## Strategy
- Scaling laws: size vs data
- Tokens/parameter ≈ 20 rule
- Pretraining → Fine-tuning → Deployment

## Mental Model
Tokens → vectors → layers → logits → softmax → output
