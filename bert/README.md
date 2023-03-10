# Paper Summary
- [BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding](https://arxiv.org/pdf/1810.04805.pdf)
- **The pre-trained BERT model can be fine-tuned with just one additional output layer to create state-of-the-art models for a wide range of tasks without substantial task-specific architecture modifications.**
## Related Works
- Feature-based approach
- Fine-tuning approach
  - **The fine-tuning approach, such as the GPT, introduces minimal task-specific parameters, and is trained on the downstream tasks by simply fine-tuning all pre-trained parameters.**
  - In OpenAI GPT, the authors use a left-to-right architecture, where every token can only at- tend to previous tokens in the self-attention layers of the Transformer. Such restrictions are sub-optimal for sentence-level tasks, and could be very harmful when applying fine-tuning based approaches to token-level tasks such as question answering, where it is crucial to incorporate context from both directions.
  - The BERT Transformer uses bidirectional self-attention, while the GPT Transformer uses constrained self-attention where every token can only attend to context to its left.
- BERT and OpenAI GPT are fine-tuning approaches, while ELMo is a feature-based approach.
## Train
- There are two steps in our framework: pre-training and fine-tuning.
### Pre-training
- During pre-training, the model is trained on unlabeled data over different pre-training tasks.
- We train with batch size of 256 sequences (256 sequences * 512 tokens = 128,000 tokens/batch) for 1,000,000 steps, which is approximately 40 epochs over the 3.3 billion word corpus. (Comment: 256 * 512 * 1,000,000 / 3,300,000,000 = 39.7) We use Adam with learning rate of 1e-4, $\beta_{1}$ = 0.9, $\beta_{2}$ = 0.999, L2 weight decay of 0.01, learning rate warmup over the first 10,000 steps, and linear decay of the learning rate. We use a dropout probability of 0.1 on all layers. We use a gelu activation (Hendrycks and Gimpel, 2016) rather than the standard relu, following OpenAI GPT.
- Longer sequences are disproportionately expensive because attention is quadratic to the sequence length. To speed up pretraing in our experiments, we pre-train the model with sequence length of 128 for 90% of the steps. Then, we train the rest 10% of the steps of sequence of 512 to learn the positional embeddings.
- Loss
  - The training loss is the sum of the mean masked LM likelihood and the mean next sentence prediction likelihood.
#### Masked Language Model (MLM)
- BERT alleviates the previously mentioned unidirectionality constraint by using a "masked lan- guage model" (MLM) pre-training objective.
- **The masked language model randomly masks some of the tokens from the input, and the objective is to predict the original vocabulary id of the masked word based only on its context. Unlike left-to-right language model pre-training, the MLM objective enables the representation to fuse the left and the right context, which allows us to pre-train a deep bidirectional Transformer.**
- In order to train a deep bidirectional representation, we simply mask some percentage of the input tokens at random, and then predict those masked tokens. In all of our experiments, we mask 15% of all WordPiece tokens in each sequence at random.
- We do not always replace "masked" words with the actual `"[MASK]"` token. The training data generator chooses 15% of the token positions at random for prediction. If the $i$-th token is chosen, we replace the $i$-th token with (1) the `"[MASK]"` token 80% of the time (2) a random token 10% of the time (3) the unchanged $i$-th token 10% of the time. Then, $T_{i}$ will be used to predict the original token with cross entropy loss.
- Assuming the unlabeled sentence is `"my dog is hairy"`, and during the random masking procedure we chose the 4-th token (which corresponding to `"hairy"`), our masking procedure can be further illustrated by
  ??? 80% of the time: Replace the word with the `"[MASK]"` token, e.g., `"my dog is hairy"` ??? `"my dog is [MASK]"`
  ??? 10% of the time: Replace the word with a random word, e.g., `"my dog is hairy"` ??? `"my dog is apple"`
  ??? 10% of the time: Keep the word unchanged, e.g., `"my dog is hairy"` ??? `"my dog is hairy"`.
- **The purpose of this is to bias the representation towards the actual observed word. The advantage of this procedure is that the Transformer encoder does not know which words it will be asked to predict or which have been replaced by random words, so it is forced to keep a distributional contextual representation of every input token.** (Comment: ??? ???????????? ????????? ????????? ?????? ????????????.)
- Additionally, because random replacement only occurs for 1.5% of all tokens (i.e., 10% of 15%), this does not seem to harm the model???s language understanding capability. **Compared to standard langauge model training, the masked LM only make predictions on 15% of tokens in each batch, which suggests that more pre-training steps may be required for the model to converge. We demonstrate that MLM does converge marginally slower than a left-to-right model (which predicts every token), but the empirical improvements of the MLM model far outweigh the increased training cost.**
- The LM masking is applied after WordPiece tokenization with a uniform masking rate of 15%, and no special consideration given to partial word pieces.
#### Next Sentence Prediction (NSP)
- In addition to the masked language model, we also use a "next sentence prediction" task that jointly pre-trains text-pair representations.
- When choosing the sentences A and B for each pre-training example, 50% of the time B is the actual next sentence that follows A (labeled as `"IsNext"`), and 50% of the time it is a random sentence from the corpus (labeled as `"NotNext"`). $C$ is used for next sentence prediction (NSP)
- To generate each training input sequence, we sample two spans of text from the corpus, which we refer to as "sentences" even though they are typically much longer than single sentences (but can be shorter also). The first sentence receives the A embedding and the second receives the B embedding. 50% of the time B is the actual next sentence that follows A and 50% of the time it is a random sentence, which is done for the "next sentence prediction" task. They are sampled such that the combined length is $\le 512$ tokens.
### Fine-tuning
- **For fine-tuning, the BERT model is first initialized with the pre-trained parameters, and all of the parameters are fine-tuned using labeled data from the downstream tasks. Each downstream task has separate fine-tuned models, even though they are initialized with the same pre-trained parameters.**
- For each task, we simply plug in the task-specific inputs and outputs into BERT and fine-tune all the parameters end-to-end. At the input, sentence A and sentence B from pre-training are analogous to (1) sentence pairs in paraphrasing, (2) hypothesis-premise pairs in entailment, (3) question-passage pairs in question answering, and 4) a degenerate text-??? pair in text classification or sequence tagging. At the output, the token representations are fed into an output layer for token-level tasks, such as sequence tagging or question answering, and the `"[CLS]"` representation is fed into an output layer for classification, such as entailment or sentiment analysis.
- Compared to pre-training, fine-tuning is relatively inexpensive.
- Our task-specific models are formed by incorporating BERT with one additional output layer, so a minimal number of parameters need to be learned from scratch. Among the tasks, (a) and (b) are sequence-level tasks while (c) and (d) are token-level tasks.
### Datasets
- For the pre-training corpus we use the BooksCorpus (800M words) (Zhu et al., 2015) and English Wikipedia (2,500M words). For Wikipedia we extract only the text passages and ignore lists, tables, and headers. It is critical to use a document-level corpus rather than a shuffled sentence-level corpus such as the Billion Word Benchmark (Chelba et al., 2013) in order to extract long contiguous sequences.
## Architecture
- A distinctive feature of BERT is its unified architecture across different tasks. There is minimal difference between the pre-trained architecture and the final downstream architecture.
- In this work, we denote the number of layers (i.e., Transformer blocks) as $L$, the hidden size as $H$, and the number of self-attention heads as $A$. We primarily report results on two model sizes: BERT-BASE ($L$ = 12, $H$ = 768, $A$ = 12, Total Parameters = 110M) and BERT-LARGE ($L$ = 24, $H$ = 1024, $A$ = 16, Total Parameters = 340M). BERT BASE was chosen to have the same model size as OpenAI GPT for comparison purposes.
- To make BERT handle a variety of down-stream tasks, our input representation is able to unambiguously represent both a single sentence and a pair of sentences in one token sequence. Throughout this work, a "sentence" can be an arbitrary span of contiguous text, rather than an actual linguistic sentence. A "sequence" refers to the input token sequence to BERT, which may be a single sentence or two sentences packed together.
- We use WordPiece embeddings with a 30,000 token vocabulary. (Comment: Vocab size)
- BERT pre-training and fine-tuning
  - <img src="https://production-media.paperswithcode.com/methods/new_BERT_Overall.jpg" width="500">
- BERT input
- We denote input embedding as $E$, The final hidden vector of the special `"[CLS]"` token as $C \in \mathbb{R}^{H}$, and the final hidden vector for the $i$th input token as $T_{i} \in \mathbb{R}^{H}$.
  - <img src="https://lh3.googleusercontent.com/stK9CWIWiSuF_aq75q7_6wUqyqfePKzeLxqVet9IVNqrcyJqqg9hXkhuFXBXXbIjaGY15gSF9Yr7kyjceVXs5HbDMpmkhet49fhbtLsm9-4E4iCYckzGTsYSxOqRaVGNTkkhWykg" width="300">
- The first token of every sequence is always a special classification token (`"[CLS]"`). The final hidden state corresponding to this token is used as the aggregate sequence representation for classification tasks.
- Sentence pairs are packed together into a single sequence. We differentiate the sentences in two ways. First, we separate them with a special token (`"[SEP]"`). Second, we add a learned embedding to every token indicating whether it belongs to sentence A or sentence B.
- For a given token, its input representation is constructed by summing the corresponding token, segment, and position embeddings.
