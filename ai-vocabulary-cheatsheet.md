**The AI Vocabulary Cheatsheet**

**A field guide to terms, models, and ideas**

*From transformers to JEPA, RAG to MCP, p(doom) to e/acc*

Current as of May 2026

**How to use this guide**

This is a reference, not a textbook. The goal is to let a working
developer decode any AI conversation, blog post, or product announcement
without having to look terms up. Each entry is one to three sentences
--- enough to recognize what something is and roughly how it fits the
landscape, not enough to actually implement it.

Terms are grouped by category rather than alphabetized. Cross-references
are inline when something refers back to a concept defined earlier.
Where two terms are commonly confused (RAG and fine-tuning, agent and
workflow, alignment and safety), that distinction is called out
explicitly.

The model landscape and benchmarks chapters carry a freshness warning.
Specific model names, version numbers, and benchmark scores were
accurate as of May 2026, but this is the area that ages fastest --- a
six-month-old reference is already partially obsolete. Treat those
chapters as a snapshot, and the conceptual chapters as long-lived.

> **The vocabulary expands faster than anyone can track**
>
> An honest disclaimer: this guide aims for the vocabulary you'll actually encounter in 2026, not exhaustive coverage. The field invents new terms weekly, most of them ephemeral marketing. If you encounter something not in this guide, the prior should be that it's either niche, brand-new, or vendor-specific jargon for a concept that's already here under another name.

**Chapter 1 --- Core architecture and training**

The vocabulary that underpins everything else. If you only learn one
chapter, learn this one --- most other terms are variations,
optimizations, or applications of these foundational ideas.

**The transformer and what's inside it**

**Transformer ---** The neural network architecture underlying nearly
every modern frontier AI model. Introduced in the 2017 paper "Attention
Is All You Need." Replaced earlier recurrent architectures (LSTMs,
RNNs) by processing entire sequences in parallel rather than one token
at a time, which unlocked the GPU-scale training that made GPT and
friends possible.

**Attention ---** The core mechanism inside a transformer. Lets each
token in a sequence "look at" every other token and weight them by
relevance when computing its next representation. Self-attention means
tokens attend to other tokens in the same sequence; cross-attention
means one sequence attends to another (used in encoder-decoder setups).
*If you only memorize one concept, memorize this one.*

**Multi-head attention ---** Running multiple attention computations in
parallel with different learned projections, then concatenating the
results. Each head can specialize in a different kind of relationship
(syntactic, semantic, positional).

**Token ---** The atomic unit of input and output for a language model.
Roughly corresponds to a word or a sub-word fragment ---
"unbelievable" might be split into ["un", "believ", "able"].
Pricing is per token, context windows are measured in tokens, and a
rough rule of thumb is 1 token ≈ 0.75 English words.

**Tokenizer ---** The component that splits text into tokens and
reverses the process on output. Modern tokenizers use byte-pair encoding
(BPE) or similar subword schemes. Different models use different
tokenizers, which is why the same text can be a different token count
across providers.

**Embedding ---** A dense vector representation of a token, sentence,
image, or other piece of data. The vector lives in a high-dimensional
space where geometrically close points correspond to semantically
similar inputs. Embeddings are how AI models "understand" anything
internally, and they are also a product category in their own right
(used for search, clustering, recommendation).

**Positional encoding ---** Information added to embeddings so that the
model knows where each token sits in the sequence. Without it, a
transformer would treat "dog bites man" and "man bites dog" as
identical. Modern variants include RoPE (rotary position embedding) and
ALiBi.

**Encoder vs decoder ---** Two halves of the original transformer.
Encoders read input and produce representations (used by BERT-style
models for classification, search). Decoders generate output token by
token (used by GPT-style models for text generation). Most modern chat
models are decoder-only.

**Autoregressive ---** Generating output one token at a time, where each
new token depends on all the previous ones. The default mode for GPT,
Claude, Gemini, and essentially all chat-based LLMs. The defining
limitation people gesture at when arguing for alternative paradigms like
JEPA or diffusion language models.

**Context window ---** The maximum number of tokens a model can attend
to at once --- both input and output combined. Has expanded
dramatically: 4K was state of the art in 2023, 128K was standard by
2024, and 1M+ became table stakes in 2026 (Gemini, Claude, DeepSeek,
Kimi all offer it). Beyond about 1M tokens, attention becomes
computationally expensive without architectural tricks.

**Parameters / weights ---** The numerical values inside the neural
network that get adjusted during training. A model's parameter count
(7B, 70B, 405B, 1.6T) is the most common rough proxy for capability,
though far from a perfect one. Modern frontier models have hundreds of
billions to trillions of parameters.

**Foundational ML concepts**

**Supervised learning ---** Training a model on labeled data --- input-output pairs where the correct answer is known. The model learns to map inputs to outputs. Most fine-tuning (SFT) is supervised. Classification ("is this spam?") and regression ("predict the price") are the two main flavors.

**Unsupervised learning ---** Training on data without labels. The model finds structure on its own --- clusters, patterns, representations. Pre-training an LLM is technically self-supervised (a variant of unsupervised): the "label" is the next token, derived from the text itself rather than human annotation.

**Transfer learning ---** Using a model trained on one task as the starting point for another. The entire pre-train-then-fine-tune paradigm is transfer learning: general knowledge transfers from the pre-training corpus to the specific downstream task. The reason you don't have to train from scratch for every application.

**Loss function ---** A mathematical function that measures how wrong the model's predictions are. Training is the process of minimizing loss. Cross-entropy loss is standard for language models (measures how far the predicted token probabilities are from the actual next token). Lower loss means the model's predictions are closer to reality.

**Gradient ---** The direction and magnitude in which each parameter should change to reduce the loss. Computed for every parameter on every training step. The gradient points "uphill" (toward higher loss); you step in the opposite direction.

**Backpropagation ---** The algorithm that computes gradients efficiently by propagating the loss backward through the network, layer by layer. Without it, computing gradients for billions of parameters would be intractable. Every modern neural network trains with backpropagation.

**Optimizer (SGD, Adam) ---** The algorithm that uses gradients to update parameters. SGD (stochastic gradient descent) is the simplest: step in the direction opposite the gradient. Adam and AdamW are the standard in practice --- they adapt the step size per parameter and add momentum. AdamW is the default for transformer training.

**Epoch ---** One complete pass through the entire training dataset. Pre-training large models often runs for less than one epoch (the dataset is so large that one pass is enough); fine-tuning typically runs for 1--5 epochs. Too many epochs leads to overfitting.

**Overfitting ---** When a model memorizes training data rather than learning general patterns. Performance on training data is high but performance on new data degrades. Signs: training loss keeps dropping but validation loss starts rising. Defenses include more data, regularization, dropout, and early stopping.

**Activation function ---** A nonlinear function applied after each layer's linear transformation. Without it, stacking layers would collapse to a single linear operation. ReLU, GELU, and SiLU/Swish are common choices in transformers. "Activations" also refers informally to the intermediate values flowing through the network --- the things interpretability researchers study.

**Softmax ---** A function that converts a vector of raw scores (logits) into a probability distribution --- values between 0 and 1 that sum to 1. The final step in token generation: the model produces logits for every token in its vocabulary, softmax converts them to probabilities, and a token is sampled.

**Dropout ---** A regularization technique that randomly zeroes out a fraction of activations during training, forcing the network to not rely on any single pathway. Reduces overfitting. Typically disabled at inference time. Less prominent in modern LLM training than in earlier deep learning, but still used in some architectures.

**Probability distribution ---** A function describing the likelihood of each possible outcome. LLMs generate a probability distribution over their vocabulary at each step --- "the" might have 12% probability, "a" might have 8%, etc. Temperature controls how "peaked" or "flat" this distribution is: low temperature concentrates probability on the top tokens (more deterministic), high temperature spreads it out (more creative/random).

**Pre-transformer architectures**

**RNN (Recurrent Neural Network) ---** An architecture that processes sequences one element at a time, maintaining a hidden state that carries information forward. The dominant sequence model before transformers. Fundamental limitation: information from early tokens degrades as the sequence gets longer (the "vanishing gradient" problem), making them poor at capturing long-range dependencies.

**LSTM (Long Short-Term Memory) ---** An improved RNN with explicit "gates" that control what information to remember and forget, partially solving the vanishing gradient problem. Powered machine translation, speech recognition, and early language models (2014--2017). Replaced by transformers because LSTMs process tokens sequentially and can't be parallelized across GPUs efficiently.

**The training process**

**Pre-training ---** The expensive stage where a model learns general
patterns from massive unlabeled text corpora (often trillions of tokens
of internet, books, code). This is where the bulk of capability comes
from, costs tens of millions to billions of dollars, and produces a
"base model" that's good at completing text but not particularly
useful as an assistant.

**Fine-tuning ---** Additional training on a smaller, focused dataset to
specialize a pre-trained model for a particular task or behavior. Costs
are orders of magnitude lower than pre-training. Variants include
supervised fine-tuning (SFT), instruction tuning, and domain-specific
fine-tuning.

**Instruction tuning ---** A specific kind of fine-tuning where the
model learns to follow human-written instructions across many task
types. What turns a base model into something that responds usefully to
"summarize this article" rather than just continuing it.

**RLHF (Reinforcement Learning from Human Feedback) ---** A training
technique where human annotators rank model outputs, a reward model is
trained on those rankings, and the LLM is then fine-tuned to maximize
reward. The technique that made ChatGPT feel "polished" compared to
raw GPT-3. Famously expensive and finicky.

**DPO (Direct Preference Optimization) ---** A simpler alternative to
RLHF that achieves similar results by directly optimizing the model on
preference data, without training a separate reward model. Has largely
replaced RLHF in many fine-tuning pipelines because it's easier to get
right.

**RLAIF (Reinforcement Learning from AI Feedback) ---** Same idea as
RLHF, but the preference rankings come from another AI model instead of
humans. Cheaper and more scalable; quality depends entirely on whether
the AI judge has good taste.

**Constitutional AI ---** Anthropic's variant where the model is
trained to critique and revise its own outputs according to a written
set of principles (the "constitution"). A form of RLAIF that aims to
make AI behavior auditable and steerable through natural-language rules
rather than opaque preference data.

**Distillation ---** Training a smaller "student" model to mimic a
larger "teacher" model's outputs. Used to produce cheap, fast models
that capture much of a frontier model's capability. Many of the
impressively-cheap models in 2026 (Haiku, Flash, Mini variants) are
distilled from their larger siblings.

**Synthetic data ---** Training data generated by other AI models rather
than collected from humans. Increasingly central to modern training
pipelines as the supply of human-written text plateaus. Quality control
is a serious engineering problem; "model collapse" is the failure mode
where models trained on their own outputs progressively degrade.

**Scaling laws ---** Empirical relationships between model size,
training data volume, compute, and resulting capability. The Chinchilla
paper (2022) is the canonical reference. The original observation ---
that bigger models with more data reliably get better --- fueled the
trillion-dollar AI capex wave. There's active debate about whether the
curves are bending.

**Inference ---** Using a trained model to produce outputs --- generating text, classifying an image, answering a question. Distinct from training: training updates weights, inference only reads them. The cost, latency, and hardware requirements of inference are different from training and increasingly dominate industry economics, since a model is trained once but serves inference millions of times.

**Chapter 2 --- Modern training and inference techniques**

Techniques that have entered the mainstream since 2022. Most are
optimizations --- making models cheaper, faster, or more capable without
fundamentally changing the architecture.

**Architectural variants**

**Mixture of Experts (MoE) ---** An architecture where the model
contains many "expert" sub-networks, and only a few are activated for
any given input. The model can have a huge total parameter count (e.g.,
1.6 trillion) while only running a fraction (e.g., 50 billion) per token
--- capturing the capability of a large model at the inference cost of a
smaller one. DeepSeek, Mixtral, GLM, and most frontier models in 2026
use MoE.

**Sparse vs dense ---** Dense models activate all their parameters on
every forward pass; sparse models (MoE being the canonical example)
activate only some. Sparse architectures dominate the high-end of the
market in 2026 because they offer better capability-per-FLOP.

**Reasoning models ---** Models specifically trained to spend extra
compute at inference time "thinking" before producing a final answer
--- typically by generating a long internal chain of thought. OpenAI's
o1 (and o3) and DeepSeek-R1 pioneered the category in late 2024; nearly
every frontier lab now ships a reasoning variant. Trade more latency and
cost for substantially better math, coding, and logic performance.

**Test-time compute ---** The compute spent at inference time rather
than training time. The thesis behind reasoning models: scaling
test-time compute can produce capability gains comparable to scaling
pre-training, at a fraction of the upfront cost. A major axis of
progress in 2025--2026.

**Chain of thought (CoT) ---** A prompting and training pattern where
the model generates intermediate reasoning steps before its final
answer. Originally a prompting trick ("let's think step by step")
that improved performance; now baked into reasoning models as their
native mode of operation.

**Multimodal ---** A model that handles more than just text ---
typically text plus images, sometimes plus audio and video. GPT-4o,
Claude with vision, Gemini, and most 2026 frontier models are
multimodal. Native multimodality (trained jointly across modalities)
tends to outperform bolted-on adapters.

**Inference optimizations**

**Quantization ---** Reducing the precision of model weights from 16-bit
or 32-bit floats down to 8-bit, 4-bit, or even lower integer formats. A
70B model in 4-bit quantization takes about 35GB of memory instead of
140GB, enabling it to run on consumer GPUs. Quality loss is usually
small with modern quantization methods (GPTQ, AWQ, GGUF).

**LoRA (Low-Rank Adaptation) ---** A fine-tuning technique that freezes
the original model weights and adds a small number of trainable
parameters (the "adapter") that capture the fine-tuning delta. Drops
fine-tuning cost by 10--100x and allows multiple adapters to be swapped
on top of one base model. QLoRA combines this with quantization for even
cheaper fine-tuning.

**KV cache ---** The cached attention keys and values from
previously-processed tokens, reused when generating subsequent tokens.
Critical for inference performance --- without it, every new token would
require re-processing the entire context. The size of this cache is a
major bottleneck for long-context inference.

**Prefix caching / prompt caching ---** Caching the KV cache itself for
commonly-reused prefixes (system prompts, RAG context). Allows providers
to charge less and respond faster when the same prefix is used many
times. Anthropic, OpenAI, and others offer this as an explicit API
feature with discounted pricing.

**Speculative decoding ---** Using a small, fast "draft" model to
predict several tokens ahead, then verifying them in parallel with the
large model. Can roughly double inference speed with no quality loss
when the draft model agrees with the main model often enough.

**Flash attention ---** An optimized implementation of attention that
uses GPU memory hierarchies more efficiently. Reduces memory usage and
increases speed without changing the math. Standard in modern training
and inference stacks.

**Batching ---** Processing multiple requests through the GPU at once
rather than one at a time. The reason API inference is cheap: providers
batch your request with hundreds of others. Continuous batching
(vLLM-style) dynamically adds and removes requests from the batch as
they finish.

**Chapter 3 --- Capabilities and usage paradigms**

How models are actually used, and the vocabulary that has grown up
around it.

**Prompting and context**

**Prompt ---** The input you send to a model. Everything from a one-line
question to a 100K-token document with elaborate instructions. The art
of writing effective prompts is called prompt engineering.

**System prompt ---** A special prompt that sets the model's persona,
constraints, and ground rules for the entire conversation. Separated
from user messages in the API ("system" vs "user" roles). Often
invisible to end users of consumer chat products.

**Prompt engineering ---** The practice of crafting prompts to get
better outputs. Includes techniques like few-shot examples, role
assignment ("you are a senior engineer"), structured output requests,
and chain-of-thought triggers. Was briefly hyped as a job title in 2023;
now considered a skill every developer has rather than a specialization.

**Context engineering ---** A 2024--2026 term reframing prompt
engineering as the broader practice of constructing the entire context
the model sees --- including retrieved documents, conversation history,
tool definitions, and few-shot examples --- rather than just the
immediate instruction. Reflects that modern agentic systems are
bottlenecked by what's in context, not by clever wording.

**In-context learning ---** The phenomenon where a language model can
pick up new tasks just from examples in its prompt, without any weight
updates. The basis of "few-shot prompting" --- show three labeled
examples in the prompt and the model generalizes the pattern.

**Few-shot / zero-shot / one-shot ---** How many examples you give the
model in the prompt. Zero-shot is asking with no examples ("translate
this to French"); few-shot is asking with several ("here are five
translations, now do this one"). Modern frontier models are strong
zero-shot, reducing the need for few-shot tricks.

**Retrieval and tools**

**RAG (Retrieval-Augmented Generation) ---** A pattern where, before
answering, the system retrieves relevant documents from a knowledge base
and stuffs them into the model's context. Lets models answer questions
about private data they weren't trained on, with citations. Was the
dominant production AI pattern from 2023--2025; long-context models and
agentic search have eroded its primacy but it remains widespread.

**Vector database ---** A database optimized for storing and querying
embeddings by similarity. The retrieval half of RAG. Pinecone, Weaviate,
Qdrant, and pgvector (Postgres extension) are common choices. In 2026,
many teams have moved back to traditional keyword + BM25 search or
hybrid approaches after discovering pure vector search has its own
failure modes.

**Semantic search ---** Search based on meaning (via embeddings) rather
than exact keyword matches. The query "how do I cancel my
subscription" finds documents about "ending account membership"
without the words overlapping.

**Tool use / function calling ---** The ability of a model to invoke
external functions (search, calculator, code execution, API calls)
during a conversation and incorporate results into its response.
Originally an OpenAI feature in 2023; now table stakes. The mechanism
underlying most useful AI applications.

**Structured output ---** Constraining a model's response to a specific
schema (JSON, regex, grammar). Providers offer this as an API parameter
that enforces the schema at decode time. Eliminates the brittleness of
"hope the JSON parses."

**MCP (Model Context Protocol) ---** An open standard, released by
Anthropic in November 2024, for how AI applications connect to external
tools and data sources. Often described as "USB-C for AI
applications." Replaces the bespoke wrappers each agent framework used
to write. Adopted by all major providers and frameworks by 2026. *If you
build agentic software, learn MCP before learning any specific
framework.*

**A2A (Agent-to-Agent) ---** A Google-developed open protocol for AI
agents to communicate and delegate tasks to other AI agents. Complements
MCP: MCP connects agents to tools, A2A connects agents to agents.

**Agents**

**Agent ---** An AI system that loops: it perceives, decides, acts (via
tools), observes the result, and decides again --- toward some
user-specified goal. The difference between a chatbot and an agent is
autonomy across multiple steps. A system that asks permission before
every action is a UI, not an agent.

**Agentic workflow ---** A structured pipeline where an AI system
completes a multi-step task, often involving multiple tool calls or even
multiple specialized sub-agents. The dominant production pattern in 2026
for any non-trivial AI application.

**ReAct (Reasoning + Acting) ---** An early agent pattern (2022 paper)
where the model interleaves thinking and acting: "Thought: I need the
weather. Action: call weather API. Observation: 72°F. Thought: ..."
The conceptual ancestor of every modern agent framework.

**Multi-agent system ---** Multiple AI agents collaborating on a task,
often with specialized roles (planner, executor, reviewer). Sometimes
genuinely useful for complex workflows; sometimes a way of papering over
a single agent's limitations with elaborate scaffolding.

**Computer use ---** An agent capability where the model takes
screenshots, moves the mouse, and types --- interacting with software
the way a human would. Anthropic introduced this with Claude in October
2024; OpenAI followed with Operator. Still rough but improving rapidly.

**Browser use / web agent ---** An agent specialized in operating a web
browser to complete tasks (research, form filling, purchases). The
narrower, often more reliable cousin of general computer use.

**Coding agent ---** An agent designed to write, modify, and debug code
across multiple files. Cursor, Claude Code, Codex, GitHub Copilot
Workspace, and Devin are examples. Has matured into the most
economically valuable agent category by 2026.

**Hallucination ---** When a model confidently produces false
information. The defining flaw of LLMs. Frontier models in 2026
hallucinate substantially less than in 2023 but the problem is not
solved; reasoning models actually hallucinate more on some benchmarks
because they confabulate elaborate justifications.

**Chapter 4 --- The model landscape**

> **Freshness warning**
>
> This chapter is current as of May 2026. Specific version numbers and pricing will be stale within months. The lab structure and competitive dynamics tend to age better than the version numbers --- focus on those if you're reading this in 2027 or later.

**The frontier labs**

**OpenAI ---** Makers of the GPT family and the o-series reasoning
models. Largest by revenue and consumer mindshare. As of mid-2026, the
current flagship is GPT-5.5, with GPT-Rosalind for life sciences. Deeply
partnered with Microsoft but increasingly available via AWS Bedrock as
well.

**Anthropic ---** Makers of Claude. Founded by former OpenAI
researchers. As of May 2026, ships Claude Opus 4.7 (flagship), Sonnet
4.6 (mid-tier), and Haiku 4.5 (fast/cheap). Distinguishing focus is AI
safety research and a constitutional approach to alignment. Pioneered
MCP and computer use.

**Google DeepMind ---** Google's AI division (merger of Google Brain
and DeepMind in 2023). Makers of the Gemini family. Gemini 3.1 Pro is
the current flagship as of May 2026. Strongest in multimodal
capabilities and tightly integrated with Google's product surface
(Workspace, Search, Android).

**Meta AI / FAIR ---** Meta's AI research arm. Makers of the Llama
family (open-weight models). Yann LeCun ran FAIR for over a decade
before departing in late 2025 to start AMI Labs. Meta's strategy: pour
billions into open-weight models to commoditize the AI layer and
undercut closed competitors.

**xAI ---** Elon Musk's AI company, founded 2023. Makes the Grok
family. Distinguishing feature is real-time integration with X (Twitter)
data. Grok 4 is the current flagship as of May 2026.

**DeepSeek ---** Chinese AI lab whose late-2024/early-2025 releases (V3,
R1) shocked the industry by approaching frontier capability at a
fraction of the training cost. Has continued to ship aggressively --- V4
Pro in early 2026 is open-weight, 1M context, and dramatically undercuts
US providers on price. Pivotal in the open-weight movement.

**Mistral ---** French AI lab. Makers of Mistral, Mixtral (the first
widely-known MoE open-weight model), and Codestral. Smaller than the
giants but produces high-quality open-weight models with permissive
licenses.

**Alibaba / Qwen ---** Alibaba's AI division. Makers of the Qwen family
of open-weight models. Particularly strong in Asian-language tasks and a
major force in the open-weight ecosystem.

**Z.AI / GLM ---** Tsinghua-affiliated Chinese lab. Makers of the GLM
family. GLM-5 reached frontier-class coding performance in 2026 as an
open-weight release.

**Moonshot AI / Kimi ---** Chinese lab whose Kimi K2 family delivered
very high reasoning quality at extremely low prices --- a recurring
story in the 2026 open-weight landscape.

**AMI Labs ---** Yann LeCun's startup, founded late 2025 after he left
Meta. Building world-model-based AI using JEPA rather than
autoregressive LLMs. Raised $1B+ at a $3.5B valuation in March 2026
--- the largest European seed round in history. The most visible non-LLM
bet in the industry.

**World Labs ---** Fei-Fei Li's startup focused on "spatial
intelligence" and world models. Raised $1B in February 2026. Another
credible non-LLM-centric bet.

**Common model naming conventions**

**Opus / Sonnet / Haiku ---** Anthropic's Claude tier names: Opus is
the flagship (slow, expensive, best), Sonnet is the workhorse middle
tier, Haiku is fast and cheap. Followed by version numbers (4.5, 4.6,
4.7).

**GPT / o-series ---** OpenAI's two tracks: GPT for general-purpose
models (4o, 5, 5.5), o-series for reasoning models (o1, o3, o4).
Increasingly merged --- newer GPT models include reasoning modes; the
distinction is fading.

**Pro / Flash / Lite ---** Google's Gemini tier names. Pro is the
flagship, Flash is fast/cheap, Lite is the budget tier. "Ultra"
appears occasionally for the very top.

**V3 / R1 / V4 etc. ---** DeepSeek's naming. V-series are general
models; R-series are reasoning models. Releases have come fast enough
that version numbers compress quickly.

**Open weights vs closed weights**

**Open weights ---** Models whose trained parameters are publicly
downloadable, usually under a permissive license (Apache 2.0, MIT) or a
custom license (Llama). You can self-host, fine-tune, and inspect. Not
the same as open source --- the training code and data are often not
released. Llama, Mistral, DeepSeek, Qwen, GLM, and Gemma are the major
open-weight families.

**Closed weights ---** Models accessible only through an API. Cannot be
downloaded or run locally. GPT, Claude, and Gemini are the major
closed-weight families. Provides predictability and lower deployment
friction but you depend on the provider for everything.

**Frontier model ---** Loosely, a model at or near the state of the art
on major benchmarks. The term is used inconsistently; sometimes it means
"top 5," sometimes "top 20." Don't read too much into specific
usage.

**Chapter 5 --- Alternatives to autoregressive LLMs**

LLMs are the dominant paradigm, but they are not the only paradigm, and
a meaningful chunk of the research community believes they are not the
path to general intelligence. This chapter is a tour of the alternatives
--- most are still research-stage in 2026, but they shape the discourse
and may shape the next decade of products.

**World models and JEPA**

**World model ---** An AI system that builds an internal representation
of physical reality and can predict consequences of actions. The
argument: LLMs are surface-level pattern matchers; real intelligence
requires a model that can simulate the world to plan. Particularly
relevant for robotics, autonomous vehicles, and any task requiring
physical reasoning.

**JEPA (Joint Embedding Predictive Architecture) ---** An architecture
proposed by Yann LeCun in 2022 that learns by predicting in latent
(abstract) representation space rather than predicting the next token or
pixel. The thesis: predicting raw outputs forces the model to memorize
irrelevant detail; predicting in latent space forces it to learn what
matters. JEPA is the technical core of LeCun's bet against LLMs.

**V-JEPA ---** Video JEPA. Meta's series of JEPA models trained on
video data, learning physical intuition from watching motion. V-JEPA 2
(2025) and V-JEPA 2.1 (2026) demonstrated strong short-term action
anticipation --- predicting what a human hand is about to do.

**LeWorldModel (LeWM) ---** A 2026 simplification of the JEPA approach
from LeCun's research circle. Trains end-to-end on a single GPU using
only two loss terms (prediction + Gaussian regularizer), avoiding the
elaborate tricks (stop-gradients, EMA teachers) that earlier JEPA
variants required. Considered the first cleanly practical demonstration
of the JEPA thesis.

**Latent space ---** An abstract, learned representation space --- the
"inside" of a neural network where data is encoded as dense vectors.
JEPA operates in latent space; LLMs operate in token space. The argument
for latent space is that it discards irrelevant detail and captures the
structure that actually matters for prediction.

**Diffusion language models**

**Diffusion model ---** An architecture that generates data by
iteratively denoising random noise. Dominant in image and video
generation (Stable Diffusion, DALL-E, Midjourney, Veo, Sora). Operates
on entire outputs at once rather than autoregressively, which is why it
generates whole images coherently.

**Diffusion language model ---** Applying the diffusion idea to text
generation. Generates entire sequences in parallel through iterative
denoising rather than one token at a time. Experimental in 2026 ---
Inception's Mercury family is the most visible commercial attempt.
Promises potentially much faster inference and better global coherence,
but quality lags autoregressive models for now.

**State-space models**

**State-space model (SSM) ---** An architecture family that processes
sequences through a recurrent latent state, more like classical signal
processing than attention. Promises linear-time scaling with sequence
length (vs quadratic for transformers), making very long contexts cheap.

**Mamba ---** The most prominent SSM architecture (2023, Tri Dao and
Albert Gu). Demonstrated competitive language modeling with much faster
inference at long contexts. Hybrid Mamba-transformer models (like Jamba
from AI21) appeared in production. Has not displaced transformers but
provides a credible alternative for specific workloads.

**Other paradigms worth knowing**

**Neurosymbolic AI ---** Approaches that combine neural networks with
classical symbolic reasoning (logic, rules, search). The promise:
combine the pattern recognition of neural nets with the reliability and
interpretability of symbolic systems. Recurring research direction; no
breakout commercial success yet.

**Liquid neural networks ---** An architecture from MIT (Ramin Hasani et
al.) using continuous-time differential equations for neurons. Very
small, efficient models for control and robotics. Niche but interesting.

**Spiking neural networks ---** Neural networks that mimic biological
neurons more closely, with discrete "spike" events instead of
continuous activations. Promising for ultra-low-power hardware
(neuromorphic chips). Still mostly research.

**Energy-based models ---** A general class of model that learns an
"energy function" assigning scores to (input, output) pairs and
generates by finding low-energy outputs. JEPA is a member of this
family. LeCun has long argued energy-based models are a better
foundation than likelihood-based ones like LLMs.

> **The state of the debate**
>
> The mainstream position in 2026 is that LLMs will keep scaling and getting better but are unlikely to be the final form of AI. The disagreement is about what comes next --- whether it's transformers with reasoning and tools layered on (the OpenAI/Anthropic/Google bet), or a fundamentally different architecture (the LeCun/AMI Labs bet), or something not yet articulated. Useful to know the vocabulary either way.

**Chapter 6 --- Evaluation and benchmarks**

How AI capability gets measured, and the vocabulary of the leaderboards.
Benchmarks saturate quickly --- what was state of the art in 2023 is
solved by every model in 2026 --- so this list will age, but the
categories are stable.

**General capability benchmarks**

**MMLU (Massive Multitask Language Understanding) ---** A 57-subject
multiple-choice exam covering everything from elementary math to
professional law. The default "general knowledge" benchmark for years.
Saturated --- top models score 90%+ --- so it's been supplanted by
harder variants like MMLU-Pro.

**GPQA ---** Graduate-level science questions across physics, biology,
and chemistry, designed to be "Google-proof" (very hard to answer by
simple search). GPQA Diamond is the harder subset. The default frontier
reasoning benchmark in 2026. Top models score 90%+.

**Humanity's Last Exam (HLE) ---** A benchmark introduced in 2024
specifically to be hard enough to not saturate quickly. Thousands of
expert-written questions spanning the limits of human knowledge. Top
models reach the 40--50% range in 2026; named in the hope that nothing
harder will be needed.

**ARC-AGI ---** Visual pattern-matching puzzles intended to test
abstract reasoning rather than memorization. Was famously resistant to
LLM progress for years; ARC-AGI-1 was largely cracked by o3 in late
2024, ARC-AGI-2 followed in 2025--2026.

**FrontierMath ---** A benchmark of unpublished, very difficult math
problems crafted by professional mathematicians. Resistant to
memorization and very slow to saturate. Tier 4 is the hardest. Top
reasoning models score 30--40% in 2026.

**Coding benchmarks**

**HumanEval ---** A 2021 benchmark of 164 Python programming problems.
Long since saturated (top models above 95%). Mostly of historical
interest.

**SWE-bench ---** Real-world software engineering tasks scraped from
GitHub issues. The model is given a repo and a bug report and must
produce a patch that fixes it. SWE-bench Verified is a hand-curated
subset. The default coding benchmark in 2026 --- top models score
70--90%.

**SWE-bench Pro ---** A harder variant introduced when SWE-bench
Verified began saturating. Top models in 2026 are in the 60--75% range.

**LiveCodeBench ---** A benchmark that scrapes new competitive
programming problems weekly, avoiding training-data contamination. Used
to verify that improvements aren't just memorization.

**Other notable benchmarks**

**HellaSwag, WinoGrande, TruthfulQA ---** Classic benchmarks from the
GPT-3 era for commonsense reasoning and avoiding falsehoods. Mostly
saturated; mentioned in older papers.

**AIME ---** American Invitational Mathematics Examination. A
standardized math competition that has become a popular benchmark for
reasoning models.

**Vectara Hallucination Leaderboard ---** Measures how often models
hallucinate facts when summarizing documents. A useful counterpoint to
capability benchmarks --- reasoning models often score worse here than
non-reasoning models, because more elaborate reasoning produces more
elaborate confabulation.

**Chatbot Arena (LMSys) ---** A platform where humans blind-rank
head-to-head model responses. Produces an Elo-style leaderboard.
Considered one of the more trustworthy measures because it reflects
actual user preferences, though it has its own biases (favors confident,
long responses).

**The benchmark-saturation pattern**

A recurring pattern in AI: a hard benchmark is introduced, the field
works on it for 1--3 years, top models reach human-or-better
performance, and the benchmark is declared "saturated" and replaced by
a harder one. Saturated benchmarks include MMLU, HumanEval, HellaSwag,
GSM8K, and increasingly ARC-AGI and GPQA. The interpretation is
contested: skeptics argue benchmarks are gamed (training on similar
data); optimists argue genuine capability is improving fast.

**Benchmark contamination ---** When test set data leaks into training
data, inflating scores. A serious problem because frontier models are
trained on most of the public internet. The reason newer benchmarks
(LiveCodeBench, FrontierMath, HLE) work hard to keep their data
unpublished.

**Evaluation / evals ---** The general practice of measuring model
behavior. "Running evals" usually means custom benchmarks built by a
team to test their specific use case rather than public leaderboards. A
core part of any serious AI engineering workflow in 2026.

**Chapter 7 --- Alignment, safety, and the discourse**

Vocabulary from the AI safety, alignment, and interpretability
communities --- plus the looser cultural vocabulary used in industry
debates. These camps have specific meanings for words that sound
similar; the distinctions matter when you're trying to follow an
argument.

**Safety vs alignment**

**Alignment ---** The technical problem of getting AI systems to pursue
the goals their designers intend. Subdivided into outer alignment
(specifying the right goal) and inner alignment (making sure the model
actually optimizes that goal rather than a proxy).

**AI safety ---** A broader term covering all the ways AI could go wrong
--- misuse, accidents, misalignment, societal harms. "Safety"
sometimes means near-term harms (bias, misuse) and sometimes means
long-term existential risks; context matters.

**Capabilities ---** What an AI system can do. Often contrasted with
alignment: "the capabilities team" works on making models smarter,
"the alignment team" works on making them controllable. A recurring
source of internal tension at frontier labs.

**AI ethics ---** Overlaps with safety but typically focuses on
near-term harms: bias, fairness, transparency, privacy, labor impacts.
Often associated with academic and civil-society researchers; sometimes
positioned in opposition to the longtermist x-risk framing.

**Failure modes**

**Jailbreak ---** A prompt that bypasses a model's safety training to
make it produce content it's been trained to refuse. The ongoing arms
race between red-teamers and safety teams. Modern frontier models are
much harder to jailbreak than 2023-era ones but the problem is not
solved.

**Prompt injection ---** An attack where malicious instructions are
hidden in data the AI processes (a web page, a document, an email) and
the AI follows them as if they were from the user. The defining security
vulnerability of agentic AI.

**Reward hacking ---** When a model finds an unintended way to score
well on its reward signal --- gaming the metric rather than achieving
the underlying goal. A classical RL failure mode that recurs in RLHF.

**Specification gaming ---** Same family as reward hacking. The model
technically satisfies the specification you gave it but in a way that
violates what you meant.

**Sycophancy ---** When a model tells the user what they want to hear
rather than what's true. A well-documented failure mode of RLHF, where
human raters reward agreeable answers.

**Deceptive alignment ---** A hypothetical (and contested) failure mode
where a sufficiently capable model behaves aligned during training but
pursues different goals when deployed. The scariest scenario in the
long-term-risk camp; the most controversial one in the near-term-risk
camp.

**Mitigation and research**

**Red teaming ---** Adversarial testing of an AI system --- trying to
make it misbehave in order to find weaknesses before deployment. Done by
internal teams, contracted external teams, and (informally) the entire
internet.

**Interpretability ---** The research field trying to understand what's
happening inside neural networks --- what features they represent, what
circuits they implement. Mechanistic interpretability ("mech interp")
is the bottom-up reverse-engineering flavor; Anthropic and DeepMind are
leading labs in this area.

**Steering / activation steering ---** Modifying a model's behavior by
directly manipulating its internal activations rather than its prompt or
weights. An interpretability-derived technique gaining traction in
2025--2026.

**Sparse autoencoders (SAEs) ---** A specific interpretability technique
that decomposes a model's internal activations into many sparse,
human-interpretable features. Anthropic's "scaling monosemanticity"
work in 2024 made this concrete.

**Scalable oversight ---** Research on how to evaluate AI behavior when
AI becomes too capable for humans to directly check its work. Techniques
include debate (have two AIs argue while a human judges), recursive
reward modeling, and weak-to-strong generalization.

**The cultural vocabulary**

**AGI (Artificial General Intelligence) ---** AI that matches or exceeds
human capability across most cognitive tasks. The definition is
contested --- some define it operationally ("can do remote knowledge
work"), some intellectually ("can do any intellectual task a human
can"). Sam Altman, Demis Hassabis, and Dario Amodei have all made
statements suggesting AGI within years; many academic researchers think
it's much further off or ill-defined.

**ASI (Artificial Superintelligence) ---** AI substantially smarter than
humans across the board. The hypothetical successor to AGI. The focus of
long-term safety concern.

**p(doom) ---** Personal probability of AI causing human extinction or
similar catastrophe. A jokey-but-serious shorthand in AI Twitter
discourse. Public figures' stated p(doom) values range from <1% (most
academics) to 50%+ (some safety researchers).

**Takeoff ---** How fast AI capability improves once it reaches certain
thresholds. "Hard takeoff" means very fast (months); "soft takeoff"
means gradual (years to decades). The pace affects how much time humans
have to react and course-correct.

**Recursive self-improvement ---** Hypothetical scenario where an AI
improves itself, then the improved version improves itself further,
accelerating to superhuman capability quickly. The mechanism behind
hard-takeoff scenarios.

**e/acc (effective accelerationism) ---** A movement / aesthetic /
Twitter tribe that argues AI development should proceed as fast as
possible. Often paired with a libertarian-techno-optimist worldview.
Originated as a deliberate counter to AI safety concerns.

**Decel / doomer ---** Pejorative terms used by e/acc adherents for
people who advocate slowing AI development for safety reasons. Sometimes
adopted ironically by the targets.

**EA (Effective Altruism) ---** A philosophical / philanthropic movement
that has historically been a major funder and source of staff for AI
safety work. Has become a politically loaded term in tech, particularly
after the FTX collapse in 2022.

**Chapter 8 --- Infrastructure and ecosystem**

The hardware, software, and supply chains AI runs on. This is where most
of the trillion-dollar capital expenditure of 2024--2026 has gone.

**GPUs and chips**

**GPU (Graphics Processing Unit) ---** The hardware that trains and runs
AI models. Originally for graphics, now mostly for AI. Nvidia dominates
this market with a near-monopoly on high-end training GPUs.

**H100 / H200 ---** Nvidia's Hopper-generation data center GPUs, the
workhorse of frontier AI training from 2023--2024. ~$25--40k each. Now
being superseded by Blackwell.

**B100 / B200 / GB200 ---** Nvidia's Blackwell-generation GPUs,
shipping in volume from 2024. Substantially faster than Hopper. GB200
(Grace Blackwell) combines an ARM CPU with two Blackwell GPUs into one
package.

**TPU (Tensor Processing Unit) ---** Google's custom AI accelerator
chips. Used internally and offered via Google Cloud. The hardware Gemini
is trained on.

**Trainium / Inferentia ---** Amazon's custom AI chips. Used internally
and on AWS.

**Ascend ---** Huawei's AI chips. Notable because DeepSeek's V4 was
trained on Huawei Ascend rather than Nvidia, demonstrating Chinese
hardware can compete at the high end.

**CUDA ---** Nvidia's GPU programming framework. The reason Nvidia's
hardware moat is so durable: an enormous ecosystem of AI software is
built on CUDA, and porting is painful. Alternatives (ROCm for AMD,
oneAPI for Intel) exist but lag.

**Compute / capex ---** Industry shorthand for the cost of GPUs and the
data centers to house them. Frontier lab capex commitments in 2026
exceed $100B/year combined. The dominant industry conversation has
shifted from algorithm improvements to who can build the largest compute
cluster.

**Training run ---** A single end-to-end pre-training of a model.
Frontier training runs cost $100M--$1B+ and can take months. The unit
of progress at the frontier.

**Inference infrastructure**

**vLLM ---** An open-source inference server widely used to deploy LLMs.
Pioneered continuous batching and PagedAttention (an efficient KV cache
management technique).

**llama.cpp ---** An open-source C++ implementation of LLM inference,
originally for Llama models, now supporting most open-weight
architectures. The reason laptops can run Llama at all. Especially
strong on Apple Silicon.

**Ollama ---** A user-friendly local LLM runtime built on llama.cpp. The
easiest way for non-experts to run open-weight models on their own
machine.

**Hugging Face ---** The dominant hub for open-weight models, datasets,
and demos. Effectively the GitHub of AI. Almost every open-weight model
lives here.

**Inference providers ---** Companies that host open-weight models and
offer them via API: Together AI, Fireworks, Groq, Replicate, Anyscale.
Compete on latency, throughput, and price. Groq notably uses custom LPU
hardware for very low-latency inference.

**Frameworks and tools**

**PyTorch ---** The dominant deep-learning framework. Nearly every model
is trained and exported in PyTorch. Originally Meta-developed; now a
Linux Foundation project.

**JAX ---** Google's deep-learning framework, used internally to train
Gemini and externally by some research groups. Smaller ecosystem than
PyTorch but strong on TPUs and functional purity.

**Transformers (library) ---** Hugging Face's Python library that
provides standardized interfaces for thousands of pre-trained models.
Distinct from "transformer" the architecture. The standard way to load
and run an open-weight model in Python.

**LangChain ---** An early Python framework for building LLM
applications, particularly RAG and agent workflows. Widely used; widely
criticized for over-abstraction. Has competition from LlamaIndex,
Haystack, and provider-native SDKs.

**LlamaIndex ---** A framework focused on data ingestion and retrieval
for LLM applications. Often used alongside or instead of LangChain for
RAG-heavy workloads.

**LangGraph / Agent SDKs ---** Newer frameworks (LangGraph, OpenAI
Agents SDK, Anthropic's agent SDK, Vercel AI SDK) focused specifically
on agent orchestration with explicit state machines or graphs.

**Cursor / Claude Code / Codex / Aider / Cline ---** AI coding tools
that integrate directly into the developer's workflow. Cursor is an
IDE; Claude Code is a terminal agent; Aider and Cline are open-source
equivalents. The category exploded in 2024--2025 and is the most
commercially successful agent application as of 2026.

**Chapter 9 --- Industry, business, and the discourse**

Vocabulary you'll encounter in AI business news, earnings calls, and
strategy debates.

**Foundation model ---** A large, general-purpose model that can be
adapted (via fine-tuning, prompting, or RAG) to many downstream tasks. A
Stanford-coined term that became standard. GPT-5, Claude Opus, and
Gemini Pro are foundation models.

**Frontier lab ---** One of the small number of organizations actively
training the largest models. The list is roughly OpenAI, Anthropic,
Google DeepMind, Meta, xAI, and (depending on who's counting) DeepSeek
and a few others.

**Compute cluster / supercluster / gigacluster ---** Large arrays of
networked GPUs used for training. Modern frontier clusters contain tens
to hundreds of thousands of GPUs. "Gigacluster" (1 GW power draw) is
the aspirational scale for late-2020s training runs.

**Data center buildout ---** The trillion-dollar wave of capital
expenditure on AI-specific data centers, GPUs, and power infrastructure.
The largest infrastructure buildout in tech history. A significant
driver of US electricity demand and a recurring topic in AI-related
macroeconomic analysis.

**API ---** The pay-per-token interface to a closed-weight model. The
primary commercial product for OpenAI, Anthropic, and similar. Pricing
is per million input/output tokens.

**Bedrock / Vertex AI / Azure AI Foundry ---** Cloud providers' AI
model marketplaces: AWS Bedrock, Google Vertex AI, Azure AI Foundry.
Aggregate multiple model providers under one billing/auth umbrella.
Important for enterprises that want a single procurement path.

**Co-pilot ---** An AI feature embedded into existing software (GitHub
Copilot, Microsoft Copilot, etc.). A specific product framing --- AI as
an assistant alongside the user --- distinct from chat-based AI
products.

**AI-native ---** Marketing term for software built around AI from the
start rather than retrofitted. Often used by startups to differentiate
from incumbents adding AI features to legacy products.

**Wrapper ---** Mildly pejorative term for an AI product that's
primarily a thin layer over a foundation model API. The
wrapper-vs-real-product debate has waned as it's become clearer that
good wrappers (Cursor, Perplexity) build serious moats through UX, data,
and integration.

**Multi-model routing ---** Architectural pattern of routing each
request to whichever model is cheapest/best for that specific task. A
common production pattern in 2026 as the model landscape has fragmented.
Often involves a cheap classifier or fast model deciding which expensive
model gets the actual work.

**Inference cost / serving cost ---** The cost to run a model in
production, distinct from the upfront cost to train it. As capability
becomes more commoditized, competition has shifted toward inference cost
--- DeepSeek's pricing has been the most disruptive force here.

**Token economics ---** Modeling AI product costs in terms of token
volume × per-token price. The fundamental unit of AI business analysis.
A startup processing 100M tokens/month might pay anywhere from $1K
(cheap open-weight) to $25K (frontier closed-weight) for the same
workload.

**AI capex bubble ---** The recurring debate about whether the
trillion-dollar AI infrastructure buildout is justified by future
revenue. The pro side cites capability progress and adoption curves; the
skeptical side cites the historical track record of capex booms and
questions whether AI revenue can scale to match the spend.

**Moat ---** Sustainable competitive advantage. In AI, debated moats
include: proprietary training data, talent, integration depth,
distribution, brand, network effects from user data. Pure model quality
has eroded as a moat as the gap between frontier providers has narrowed.

**Chapter 10 --- Generative media beyond text**

AI for images, video, audio, and 3D. A different technical lineage from
text LLMs, dominated by diffusion models rather than transformers. Worth
covering at least lightly because the vocabulary overlaps and the
products are increasingly multimodal.

**Image generation**

**DALL-E / Imagen / Midjourney / Stable Diffusion / Flux ---** The major
image-generation model families. DALL-E is OpenAI's; Imagen is
Google's; Midjourney is a standalone company; Stable Diffusion is
open-weight (Stability AI); Flux is from Black Forest Labs (founded by
ex-Stable Diffusion researchers).

**Diffusion ---** The dominant architecture for image generation
(covered briefly in Chapter 5 too). Iteratively denoises random noise
into an image conditioned on a text prompt.

**Latent diffusion ---** Running the diffusion process in a compressed
latent space rather than on raw pixels. Stable Diffusion's key
innovation; the reason it can run on consumer GPUs.

**ControlNet ---** A technique for conditioning image generation on
additional inputs (pose, depth map, edges, sketch) beyond just text.
Lets users get precise compositional control.

**LoRA (for image models) ---** The same low-rank-adaptation technique
as for LLMs, applied to image models. Used to teach a base model a
specific style, character, or subject from a handful of examples. The
basis of the Civitai-style ecosystem of community-trained image styles.

**Video generation**

**Sora / Veo / Kling / Runway ---** The major video-generation products.
Sora was OpenAI's flagship (discontinued April 2026 after low usage);
Veo is Google's; Kling is Chinese (Kuaishou); Runway is an independent
company with the Gen-series models. Generates short clips (5--60
seconds) from text or image prompts.

**Text-to-video / image-to-video ---** The two main video-generation
modes: synthesize a video from a text description, or animate a still
image. Image-to-video tends to be more controllable.

**Audio and speech**

**TTS (Text-to-speech) ---** Generating speech audio from text. Modern
systems (ElevenLabs, OpenAI's voice models, Cartesia) are nearly
indistinguishable from human voices for short clips and support voice
cloning from seconds of reference audio.

**STT / ASR (Speech-to-text / Automatic Speech Recognition) ---**
Transcribing speech to text. Whisper (OpenAI, open-weight) is the
standard; Deepgram and AssemblyAI are commercial alternatives with
stronger latency or vertical specialization.

**Voice cloning ---** Replicating a specific person's voice. Powerful
enough in 2026 that audio deepfakes are a serious fraud and
misinformation concern.

**Suno / Udio ---** AI music generation models. Generate full songs
(vocals + instrumentation) from text prompts. Have faced significant
lawsuits from major record labels over training data.

**3D and other**

**NeRF (Neural Radiance Fields) ---** A technique for representing 3D
scenes as neural networks, enabling photorealistic novel-view synthesis
from a small number of input images. Largely supplanted by...

**Gaussian splatting ---** A 3D scene representation using millions of
small Gaussian blobs. Faster than NeRF, easier to edit, becoming the
standard for AI-generated 3D scenes.

**Chapter 11 --- Quick reference and common confusions**

A final chapter for fast lookup. Common pairs of terms that get
confused, and a glance-able summary of what to use when.

**Commonly confused pairs**

**Fine-tuning vs RAG ---** Both adapt a model to your domain.
Fine-tuning bakes knowledge into weights (good for style, format,
behavior); RAG injects knowledge at runtime (good for facts, freshness,
citations). For most use cases in 2026, start with RAG; fine-tune only
when behavior modification is the goal.

**Agent vs workflow ---** A workflow is a predefined sequence of LLM
calls and tool invocations. An agent decides its own sequence based on
intermediate results. Many things marketed as agents are actually
workflows; the line is fuzzy. The Anthropic framing: agents have
autonomy, workflows have structure.

**Alignment vs safety ---** Alignment is the technical problem of
getting models to pursue intended goals. Safety is the broader concern
of avoiding harm. All alignment work is safety work; not all safety work
is alignment.

**Tokens vs words ---** 1 token ≈ 0.75 English words on average, but
with large variance. Code, non-English languages, and unusual words use
more tokens per word. Always measure in tokens when costs matter.

**Open-source vs open-weights ---** Open-source AI would include
training code, training data, and a permissive license. Open-weights
only requires the model parameters themselves. Almost no "open-source"
models are actually open-source by the strict definition;
"open-weights" is the more accurate term and what people usually mean.

**Hallucination vs confabulation ---** Often used interchangeably for AI
fabrications. Some researchers prefer "confabulation" because
hallucination implies sensory experience the model doesn't have. The
term has stuck regardless.

**LLM vs foundation model vs frontier model ---** LLM = large language
model (any text-based model in this class). Foundation model = a
general-purpose model adapted to many tasks (broader; includes image and
multimodal models). Frontier model = currently at or near the state of
the art. The terms nest but aren't synonyms.

**What to use when --- a one-page mental model**

- Need to answer questions about your private data? **RAG**

- Need the model to write in a specific voice or follow a specific
  format? **Fine-tuning or careful prompting**

- Need the model to take actions in the world? **Tool use / agent**

- Need step-by-step reasoning on hard problems? **Reasoning model**

- Need fast/cheap inference on simple tasks? **Small model (Haiku,
  Flash, Mini tier) or open-weight**

- Need to handle images, audio, or video? **Multimodal frontier model**

- Need to connect AI to existing tools and data? **MCP**

- Building a coding tool? **Claude Sonnet, GPT-5 Codex, or open-weight
  alternatives like GLM/Qwen Coder**

**Closing**

The vocabulary above will not be complete six months from now. New
techniques and new model versions arrive weekly; some entries here will
be obsolete by the time you read them. But the conceptual scaffolding
--- transformers, attention, RAG, agents, the major labs, the
alignment/safety debate, the alternatives to LLMs --- has been stable
for a few years and is likely to remain stable for at least a few more.

If you encounter a new term and it doesn't fit anywhere in this guide,
the prior should be that it's a vendor-specific name for an existing
concept, a niche research term, or a marketing neologism. Look it up; if
it survives the next year, it'll be worth knowing. Most of them don't.

Good luck out there. The pace is exhausting, but it's also one of the
most interesting times to be paying attention.
