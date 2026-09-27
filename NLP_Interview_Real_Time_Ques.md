# NLP Interview Questions — Verified and Collected from different Candidate-Reported Questions

## Research Standard

This document contains NLP questions that were **reported by candidates as being asked in actual interviews or technical interview rounds**.

### Evidence sources used

- **ZS Associates — Data Science Associate:** detailed first-person interview experience covering NLP preprocessing, feature engineering, Word2Vec, TF-IDF, N-grams, model selection, deep learning and the end-to-end NLP lifecycle.
- **Zycus — AI/Machine Learning Engineer:** candidate-reported questions on embeddings, Skip-gram, CBOW, GloVe, Word2Vec and sequence modeling.
- **Zee Entertainment — Data Scientist:** candidate-reported questions on term-document matrix, TF-IDF, word embeddings, text-to-numeric conversion, LSTM and evaluation metrics.
- **American Express — NLP Data Scientist:** candidate-reported preprocessing and regex question involving unidecode errors, punctuation and numbers.
- **JPMorganChase — NLP Summer:** candidate-reported RNN, Transformer and self-attention questions.
- **IBM — NLP Researcher:** candidate-reported RAG system design question.
- **KnowDis Data Science — Data Scientist:** 2025 candidate report covering RNN, LSTM, Transformer components, positional embeddings, multi-head attention, encoder/decoder, masked decoder, autoregressive LLMs and DistilBERT.
- **Level AI — Machine Learning Engineer - NLP:** 2025 candidate report covering self-attention vs multi-head attention, Sentence Transformers, positional embeddings, RoPE, decoder inputs/outputs, beam search and RAG.
- **Dimensionless Technologies — NLP Intern:** candidate-reported NLU/NLG question.
---

# Part 1 — NLP Preprocessing and Classical NLP

## 1. How will you clean and preprocess text data?

**Source:** ZS Associates — Data Science Associate

### Answer

I would first understand the task and then build a task-specific preprocessing pipeline.

Typical steps are:

1. Remove or normalize unwanted HTML, URLs and special characters.
2. Handle punctuation appropriately.
3. Handle numbers based on whether they contain useful information.
4. Normalize case if appropriate.
5. Tokenize the text.
6. Handle spelling variations, contractions and abbreviations if required.
7. Apply stemming or lemmatization when appropriate.
8. Remove stopwords only when they are not useful for the task.
9. Convert the processed text into numerical features such as TF-IDF, n-grams or embeddings.

The important point is that I would not blindly remove every number, punctuation mark or stopword because those can carry useful information.

---

## 2. You removed all numeric values from the text. Could numeric values contain useful information?

**Source:** ZS Associates — Data Science Associate

### Answer

Yes.

Numbers can contain useful information depending on the problem.

For example:

```text
2026 internship
5 years experience
20% discount
iPhone 15
COVID-19
```

Removing all numbers could remove useful predictive signals.

Instead, I would determine whether numbers have semantic value and either:

- preserve them,
- normalize them,
- replace them with special tokens, or
- remove them only when they are irrelevant.

---

## 3. Is there a better approach than dropping all non-alphabetic characters?

**Source:** ZS Associates — Data Science Associate

### Answer

Yes.

Instead of applying something like:

```python
re.sub('[^a-zA-Z]', ' ', text)
```

blindly, I would decide what should be retained based on the task.

For example:

```text
COVID-19
20%
₹999
C++
Python 3.12
```

can contain useful information.

A better approach is to selectively normalize unwanted characters while preserving meaningful tokens.

---

## 4. Why did you use stemming instead of lemmatization?

**Source:** ZS Associates — Data Science Associate

### Answer

Stemming is generally faster and simpler because it uses heuristic rules to reduce words to a root-like form.

Lemmatization attempts to return a valid dictionary/base form and generally uses more linguistic information.

Example:

```text
Studies → studi       # stemming example
Studies → study       # lemmatization
```

I would choose based on the task, dataset size, accuracy requirements and computational constraints.

---

## 5. When is lemmatization helpful compared with stemming?

**Source:** ZS Associates — Data Science Associate

### Answer

Lemmatization is useful when maintaining a linguistically meaningful base word is important.

For example:

```text
better → good
studies → study
running → run
```

Stemming may produce crude forms that are not valid words.

For simple large-scale text classification where speed is important, stemming can still be useful.

---

## 6. How would you handle punctuation and numbers in NLP preprocessing?

**Source:** American Express — NLP Data Scientist

### Answer

I would not automatically remove everything.

I would first determine whether punctuation or numbers contain information for the task.

For example, in sentiment analysis:

```text
Great!!!
```

the punctuation may contain useful sentiment information.

Similarly:

```text
₹999
5-star
20%
```

may be important in a product-review problem.

The preprocessing rule should therefore be task-specific.

---

## 7. How would you handle unidecode errors?

**Source:** American Express — NLP Data Scientist

### Answer

I would first inspect the source text and encoding.

A practical approach is:

1. Ensure the input has the expected Unicode encoding.
2. Normalize Unicode when required.
3. Handle malformed characters.
4. Apply transliteration only when appropriate.
5. Validate the output after normalization.

I would avoid silently converting every character because that can destroy information in multilingual text.

---

## 8. How will you vectorize text? What methods do you know?

**Source:** ZS Associates — Data Science Associate

### Answer

Common approaches include:

### Traditional representations

- One-hot encoding
- Bag of Words
- TF-IDF
- N-grams

### Dense representations

- Word2Vec
- GloVe
- FastText
- Sentence embeddings
- Transformer embeddings

The choice depends on the problem, dataset size, semantic requirements and model architecture.

---

# Part 2 — TF-IDF, BoW and N-Grams

## 9. Did you try Bag of Words, TF-IDF and N-grams? How did they perform?

**Source:** ZS Associates — Data Science Associate

### Answer

These are useful baseline representations.

### Bag of Words

Represents a document using word frequencies.

### TF-IDF

Weights terms according to their importance within a document relative to the corpus.

### N-grams

Capture sequences of words.

For example:

```text
"I love NLP"
```

Bigrams:

```text
I love
love NLP
```

I would compare them experimentally using the same train/validation setup and evaluation metrics rather than assuming one will always perform best.

---

## 10. What is a Term-Document Matrix?

**Source:** Zee Entertainment — Data Scientist

### Answer

A Term-Document Matrix represents the relationship between terms and documents.

Typically:

- Rows represent terms.
- Columns represent documents.
- Values represent frequency or another weighting such as TF-IDF.

Example:

```text
             Doc1   Doc2
NLP           2      1
Python        1      3
AI            0      2
```

It provides a numerical representation of text that can be used by traditional ML algorithms.

---

## 11. How did you convert text into numerical representation?

**Source:** Zee Entertainment — Data Scientist

### Answer

I would choose the representation based on the task.

For a classical ML baseline:

```text
Text
 ↓
Tokenization
 ↓
TF-IDF / BoW
 ↓
ML classifier
```

For semantic tasks:

```text
Text
 ↓
Embedding model
 ↓
Dense vector
 ↓
Similarity / classifier / downstream model
```

For modern NLP:

```text
Text
 ↓
Transformer tokenizer
 ↓
Transformer model
 ↓
Contextual representation
```

---

## 12. Why can averaging Word2Vec vectors lose information?

**Source:** ZS Associates — Data Science Associate

### Answer

Suppose we represent a sentence by averaging all word vectors.

The result loses:

- Word order
- Syntactic structure
- Token-level context
- Some information about which words are important

For example:

```text
"dog bites man"
```

and

```text
"man bites dog"
```

can produce similar averaged representations.

Better approaches include:

- TF-IDF-weighted embeddings
- RNN/LSTM
- CNN
- Transformer encoders
- Sentence Transformer models

---

# Part 3 — Word Embeddings

## 13. What are word embeddings?

**Source:** Zee Entertainment / Zycus interview reports

### Answer

Word embeddings represent words as dense numerical vectors.

The vectors are learned so that words appearing in similar contexts tend to have related representations.

Examples:

```text
king
queen
doctor
hospital
```

Unlike one-hot vectors, embeddings can capture relationships between words.

---

## 14. What is Word2Vec?

**Source:** ZS Associates / Zycus

### Answer

Word2Vec is a neural approach for learning word embeddings from context.

It has two major architectures:

```text
CBOW
Skip-gram
```

CBOW predicts a target word from surrounding context.

Skip-gram predicts surrounding context words from a target word.

---

## 15. What is Skip-gram?

**Source:** Zycus — AI/Machine Learning Engineer

### Answer

Skip-gram takes a target word and tries to predict nearby context words.

Example:

```text
"The cat sat on the mat"
```

For target:

```text
cat
```

the model may learn to predict nearby words such as:

```text
the
sat
```

The model learns useful word representations as a consequence of this prediction task.

---

## 16. What is CBOW?

**Source:** Zycus — AI/Machine Learning Engineer

### Answer

CBOW stands for Continuous Bag of Words.

It predicts the target word from surrounding context.

Example:

```text
The cat ___ on the mat
```

The surrounding words are used to predict:

```text
sat
```

CBOW and Skip-gram are two different Word2Vec training objectives.

---

## 17. What is the difference between CBOW and Skip-gram?

**Source:** Zycus — AI/Machine Learning Engineer

### Answer

```text
CBOW:
Context → Target

Skip-gram:
Target → Context
```

CBOW generally trains faster because it predicts a target from context.

Skip-gram can be useful for learning representations of less frequent words because it generates multiple context-prediction examples.

---

## 18. Did you use pretrained Word2Vec or train it on your own data?

**Source:** ZS Associates — Data Science Associate

### Answer

It depends on the domain and data.

### Pretrained embeddings

Advantages:

- Already trained on large corpora.
- Useful when my dataset is small.
- Can provide stronger general semantic representations.

### Training on domain data

Advantages:

- Can capture domain-specific vocabulary and usage.
- Useful when the application contains specialized terminology.

I would compare the alternatives using validation performance rather than assuming pretrained embeddings are always better.

---

## 19. Would pretrained embeddings perform better?

**Source:** ZS Associates — Data Science Associate

### Answer

Not necessarily.

Performance depends on:

- Domain similarity
- Corpus size
- Vocabulary
- Task
- Embedding quality
- Amount of training data

For a specialized domain, an embedding trained on domain-specific text can sometimes capture terminology better.

The correct approach is to evaluate both alternatives.

---

## 20. What is the difference between Word2Vec and GloVe?

**Source:** Zycus — AI/Machine Learning Engineer

### Answer

Both learn dense word representations but use different training approaches.

### Word2Vec

Learns embeddings through prediction tasks:

```text
CBOW
Skip-gram
```

### GloVe

Uses global word co-occurrence statistics to learn vectors.

A useful interview summary is:

```text
Word2Vec → predictive context-based training
GloVe    → global co-occurrence statistics
```

---

# Part 4 — NLP Modeling and Evaluation

## 21. What models did you try, and what were the results?

**Source:** ZS Associates — Data Science Associate

### Answer

I would compare models systematically.

For example:

```text
TF-IDF
   ↓
Logistic Regression
   ↓
Naive Bayes
   ↓
SVM
```

Then compare them using appropriate metrics.

For an imbalanced classification problem, I would not rely only on accuracy.

I would examine:

- Precision
- Recall
- F1-score
- Confusion matrix
- Class-wise performance

---

## 22. How did you handle class imbalance?

**Source:** ZS Associates — Data Science Associate

### Answer

First, I would measure the class distribution.

Possible approaches include:

- Class weights
- Oversampling
- Undersampling
- Data augmentation
- Threshold adjustment

I would evaluate using metrics appropriate for imbalance, such as:

```text
Precision
Recall
F1-score
PR-AUC
```

I would also inspect per-class performance.

---

## 23. How could you improve the model metrics?

**Source:** ZS Associates — Data Science Associate

### Answer

I would investigate the problem systematically:

```text
Data quality
↓
Label quality
↓
Class imbalance
↓
Preprocessing
↓
Feature engineering
↓
Representation
↓
Model
↓
Hyperparameters
↓
Error analysis
```

I would inspect false positives and false negatives to understand why the model fails.

---

## 24. Could Deep Learning improve the results?

**Source:** ZS Associates — Data Science Associate

### Answer

Potentially, but not automatically.

Deep learning may help when:

- There is enough training data.
- The task depends on complex context.
- Traditional representations are insufficient.
- We need learned semantic representations.

Possible models include:

- CNN
- RNN
- LSTM
- Transformer

I would compare against a strong classical baseline rather than assuming deep learning is always better.

---

# Part 5 — RNN and LSTM

## 25. How do RNNs work?

**Source:** JPMorganChase — NLP Summer

### Answer

An RNN processes a sequence one element at a time and maintains a hidden state.

Conceptually:

```text
x1 → RNN → h1
x2 → RNN → h2
x3 → RNN → h3
```

The hidden state carries information from previous tokens.

This makes RNNs suitable for sequential data.

---

## 26. What are the problems with RNNs?

**Source:** KnowDis — Data Scientist

### Answer

Major issues include:

- Vanishing gradients
- Exploding gradients
- Difficulty learning long-term dependencies
- Sequential computation

LSTM and GRU architectures were designed to improve the handling of long-term dependencies.

---

## 27. How can you solve RNN problems without moving directly to LSTM?

**Source:** KnowDis — Data Scientist

### Answer

Possible techniques include:

- Gradient clipping for exploding gradients.
- Better initialization.
- Appropriate activation functions.
- Architectural changes such as gated mechanisms.
- Careful sequence handling and truncated backpropagation.

For long-term dependencies, LSTM/GRU or Transformers are often more practical.

---

## 28. Explain LSTM architecture.

**Source:** Zee Entertainment / Zycus interview reports

### Answer

LSTM stands for Long Short-Term Memory.

An LSTM contains:

```text
Forget Gate
Input Gate
Output Gate
Cell State
Hidden State
```

The gates control what information should be:

```text
forgotten
stored
exposed
```

This helps reduce the long-term dependency problems of a vanilla RNN.

---

# Part 6 — Transformers

## 29. How do Transformers work?

**Source:** JPMorganChase — NLP Summer

### Answer

A Transformer processes tokens using attention rather than recurrence.

A simplified Transformer flow is:

```text
Input Tokens
     ↓
Token Embeddings
     +
Positional Information
     ↓
Self-Attention
     ↓
Feed Forward Network
     ↓
Repeated Transformer Layers
     ↓
Output Representation
```

The architecture enables much more parallel computation than traditional RNNs.

---

## 30. Explain self-attention.

**Source:** JPMorganChase — NLP Summer

### Answer

Self-attention allows each token to determine how much attention it should give to other tokens in the sequence.

It uses:

```text
Query
Key
Value
```

The standard attention calculation is:

```text
Attention(Q,K,V)
=
softmax(QKᵀ / √dₖ)V
```

This allows the model to build contextual representations.

---

## 31. What is the difference between self-attention and multi-head attention?

**Source:** Level AI — Machine Learning Engineer - NLP

### Answer

Self-attention calculates attention across the sequence.

Multi-head attention runs multiple attention heads in parallel.

Each head can learn different relationships.

Conceptually:

```text
Input
 ↓
Head 1 → relationship type 1
Head 2 → relationship type 2
Head 3 → relationship type 3
...
 ↓
Concatenate
 ↓
Linear projection
```

This allows the model to capture different types of relationships simultaneously.

---

## 32. What is positional embedding and why is it needed?

**Source:** KnowDis / Level AI

### Answer

Transformers do not inherently process tokens sequentially like RNNs.

Therefore, the model needs information about token positions.

For example:

```text
dog bites man
```

is different from:

```text
man bites dog
```

Positional information helps the model distinguish these orders.

---

## 33. What are the components of a Transformer?

**Source:** KnowDis — Data Scientist

### Answer

Important components include:

```text
Token embeddings
Positional information
Multi-head attention
Feed-forward network
Residual connections
Layer normalization
```

The exact structure differs between encoder-only, decoder-only and encoder-decoder Transformer architectures.

---

## 34. What is the difference between Transformer encoder and decoder?

**Source:** KnowDis — Data Scientist

### Answer

### Encoder

Primarily builds contextual representations from input tokens.

Used in architectures such as:

```text
BERT
```

### Decoder

Generates tokens autoregressively in decoder-only models such as:

```text
GPT-style models
```

### Encoder-decoder

Uses an encoder to process input and a decoder to generate output.

Examples include many sequence-to-sequence Transformer architectures.

---

## 35. What is a masked decoder?

**Source:** KnowDis — Data Scientist

### Answer

A decoder used for autoregressive generation applies a causal mask so that a token cannot attend to future tokens.

For example:

```text
I love machine learning
```

When predicting:

```text
machine
```

the model can use:

```text
I love
```

but should not use future tokens:

```text
learning
```

This prevents information leakage from future positions during training.

---

## 36. What are autoregressive LLMs?

**Source:** KnowDis — Data Scientist

### Answer

An autoregressive language model predicts the next token based on previous tokens.

Example:

```text
Input:
"I love"

Prediction:
"machine"

Then:
"I love machine"

Prediction:
"learning"
```

This process continues token by token.

Decoder-only Transformer models commonly use this objective.

---

# Part 7 — Modern NLP / GenAI

## 37. What are Sentence Transformers?

**Source:** Level AI — Machine Learning Engineer - NLP

### Answer

Sentence Transformers are models designed to produce dense vector representations for sentences or larger text units.

They are commonly used for:

- Semantic similarity
- Semantic search
- Clustering
- Retrieval
- Duplicate detection

For example:

```text
"I bought a car."
"I purchased an automobile."
```

can receive similar sentence embeddings despite different wording.

---

## 38. What are the details you consider when building a RAG system?

**Source:** IBM — NLP Researcher; Level AI — Machine Learning Engineer - NLP

### Answer

I would consider the complete pipeline:

```text
Documents
   ↓
Parsing
   ↓
Chunking
   ↓
Embedding
   ↓
Vector / Hybrid Index
   ↓
Retrieval
   ↓
Optional reranking
   ↓
Context construction
   ↓
LLM
   ↓
Answer
```

Important design considerations include:

- Chunk size
- Chunk overlap
- Embedding model
- Retrieval strategy
- Top-k
- Metadata filtering
- Reranking
- Context limits
- Citation/grounding
- Evaluation
- Latency
- Cost

---

## 39. How would you design an NLP search system?

**Source:** KnowDis — Machine Learning Engineer interview

### Answer

A practical search system can contain:

```text
User Query
   ↓
Query preprocessing
   ↓
Candidate retrieval
   ↓
Keyword / BM25 / vector search
   ↓
Optional reranking
   ↓
Top results
```

For semantic search, embeddings can be used to retrieve documents based on meaning rather than exact keyword overlap.

A production system may combine:

```text
Keyword search + semantic/vector search
```

---

## 40. How would you approach an NLP case study where text must be classified into categories?

**Source:** ZS Associates — Data Science Associate

### Answer

I would approach it as an end-to-end ML problem:

```text
1. Understand the business objective
2. Inspect and label data
3. Explore the data
4. Check class distribution
5. Clean and preprocess text
6. Engineer useful features
7. Split data correctly
8. Build a baseline
9. Compare TF-IDF / n-grams / embeddings
10. Train multiple models
11. Evaluate using appropriate metrics
12. Perform error analysis
13. Tune the best approach
14. Deploy
15. Monitor and retrain
```

The important part is explaining **why** each decision was made.

---

# High-Priority Follow-Up Questions

These are not separate generic questions; they are especially useful because real interviews often drill deeper into the candidate's previous answer.

## If you say "I used TF-IDF"

Be ready for:

```text
Why TF-IDF?
Why not BoW?
Why not Word2Vec?
What is IDF?
Why do common words get lower weight?
What happens with unseen words?
Did you use unigrams or bigrams?
How did you choose max_features?
How did you avoid data leakage?
```

## If you say "I used Word2Vec"

Be ready for:

```text
CBOW vs Skip-gram?
Pretrained or trained yourself?
Why?
What is the context window?
How are embeddings learned?
What happens with unknown words?
Why average the vectors?
What information is lost by averaging?
Why not use an LSTM/Transformer?
```

## If you say "I used LSTM"

Be ready for:

```text
Why LSTM?
Why not RNN?
What is vanishing gradient?
What are the gates?
What is cell state?
What is hidden state?
How does forget gate work?
Why not Transformer?
```

## If you say "I used Transformer"

Be ready for:

```text
Why Transformer?
What is self-attention?
What are Q, K and V?
Why divide by sqrt(dk)?
What is multi-head attention?
Why positional encoding?
Encoder vs decoder?
What is causal masking?
How does autoregressive generation work?
```

## If you say "I built RAG"

Be ready for:

```text
Why RAG?
How did you chunk documents?
Why that chunk size?
Which embedding model?
Why that embedding model?
How many chunks did you retrieve?
Why top-k?
Vector search vs keyword search?
Did you rerank?
How did you evaluate retrieval?
How did you evaluate the final answer?
How did you reduce hallucination?
```

---

# Most Important 20 to Memorize First

If you have limited preparation time, prioritize these verified questions:

```text
1. How will you clean and preprocess text?
2. Should numbers always be removed?
3. Stemming vs lemmatization?
4. Why stemming instead of lemmatization?
5. How do you vectorize text?
6. BoW vs TF-IDF vs N-grams?
7. What is a Term-Document Matrix?
8. What are word embeddings?
9. What is Word2Vec?
10. CBOW vs Skip-gram?
11. Word2Vec vs GloVe?
12. Why can averaging word vectors lose information?
13. How do you handle class imbalance?
14. How do you improve NLP model metrics?
15. How does RNN work?
16. What are the problems with RNN?
17. Explain LSTM architecture.
18. How does a Transformer work?
19. Explain self-attention and multi-head attention.
20. How would you design a RAG system?
```

These 20 cover the major themes that recur across the verified candidate reports: **preprocessing, representation, embeddings, sequence models, Transformers and modern RAG**.

---

# Source Evidence

### [S1] ZS Associates — Data Science Associate

Candidate reported a complete NLP classification interview, including preprocessing, numeric-value handling, stemming vs lemmatization, Word2Vec, pretrained embeddings, BoW, TF-IDF, N-grams, averaged embeddings, model selection, deep learning and metric improvement.

### [S2] Zycus — AI/Machine Learning Engineer

Candidate reported questions on embeddings, Skip-gram, CBOW, GloVe, Word2Vec, latent-space representation and sequence modeling.

### [S3] Zee Entertainment — Data Scientist

Candidate reported questions on text-to-numeric conversion, Term-Document Matrix, TF-IDF, word embeddings, precision/recall and LSTM architecture.

### [S4] American Express — NLP Data Scientist

Candidate reported a regex and preprocessing question involving unidecode errors, punctuation and numbers.

### [S5] JPMorganChase — NLP Summer

Candidate reported questions on RNNs, Transformers and self-attention.

### [S6] IBM — NLP Researcher

Candidate reported a question about designing a RAG system, including retrievers and generator considerations.

### [S7] KnowDis Data Science — Data Scientist

Candidate reported questions on RNN problems, LSTM gates, Transformer components, positional embeddings, multi-head attention, encoder/decoder architecture, masked decoders, autoregressive LLMs and DistilBERT.

### [S8] Level AI — Machine Learning Engineer - NLP

Candidate reported questions on self-attention vs multi-head attention, Sentence Transformers, positional embeddings, RoPE, decoder behavior, beam search and RAG.

### [S9] Dimensionless Technologies — NLP Intern

Candidate reported the question: "What is NLU and NLG?"

---
