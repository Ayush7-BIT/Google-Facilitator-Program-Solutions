Task 2 - replace the class SimpleArabicCharacterTokenizer cell

```/text
class SimpleArabicCharacterTokenizer:
    """A simple character tokenizer."""

    def __init__(self):
        pass

    def character_tokenize(self, text: str) -> list[str]:
        """Splits a given Arabic text into character tokens."""
        # Tokenize the string into character tokens.
        tokens = list(text)

        return tokens

    def join_text(self, tokens: list[str]) -> str:
        """Combines a list of tokens into a single string."""
        # Join the list of character tokens into a string.
        text = "".join(tokens)

        return text
```

Task 3 - first run the "N-gram model functions" cell and the argmax cell, then replace the def generate_text_from_ngram_model cell
```/text
def generate_text_from_ngram_model(
        start_prompt: str,
        n_tokens: int,
        ngram_model: dict[str, dict[str, float]],
        tokenizer: SimpleArabicCharacterTokenizer,
        sampling_mode: Literal["random", "greedy"] = "random"
) -> str:
    """Generate text based on a starting prompt using an ngram model."""
    # Tokenize the starting prompt.
    start_tokens = tokenizer.character_tokenize(start_prompt)

    generated_tokens = start_tokens + []

    # The context is n-1 tokens long = length of any key in the model.
    context_size = len(next(iter(ngram_model)))

    for _ in range(n_tokens):
        # Context = the last n-1 tokens generated so far.
        context = tokenizer.join_text(generated_tokens[-context_size:])
        if context not in ngram_model:
            break  # unseen context: stop generating

        next_tokens = list(ngram_model[context].keys())
        probabilities = list(ngram_model[context].values())

        if sampling_mode == "random":
            # Random sampling, weighted by the n-gram probabilities.
            next_token = random.choices(next_tokens, weights=probabilities, k=1)[0]
        elif sampling_mode == "greedy":
            # Greedy: the token with the highest probability.
            next_token = next_tokens[argmax(probabilities)]
        else:
            raise ValueError("sampling_mode must be 'random' or 'greedy'")

        generated_tokens.append(next_token)

    # Generated tokens are converted back to str.
    generated_text = tokenizer.join_text(generated_tokens)
    return generated_text
```


Task 4a - first run the class EnhancedTokenizer cell, then replace the def segment_encoded_sequence cell

```/text
def segment_encoded_sequence(
        sequence: list[int],
        max_length: int,
        n_overlap: int
) -> list[list[int]]:
    """Segment a long encoded sequence into overlapping subsequences."""
    subsequences = []

    # Each new segment starts (max_length - n_overlap) tokens after the
    # previous one, so the last n_overlap tokens of a segment start the next.
    step = max_length - n_overlap
    start = 0
    while start < len(sequence):
        subsequences.append(list(sequence[start:start + max_length]))
        if start + max_length >= len(sequence):
            break  # reached the end; this last segment may be shorter
        start += step

    return subsequences
```

Task 4b - replace the def create_training_sequences cell

```/text
def create_training_sequences(
        dataset: list[str],
        context_length: int,
        n_overlap: int,
        tokenizer: EnhancedTokenizer
) -> tuple[np.ndarray, np.ndarray]:
    """Create training input-target sequence pairs from text dataset."""

    segmentation_length = context_length + 1
    # The segments are one token longer than the model's maximum input length,
    # because the target (next) tokens to predict are the input tokens shifted
    # by one position.

    pad_token_id = tokenizer.pad_token_id
    encoded_tokens = []

    for text in dataset:
        # Encode the text into a sequence of token ids.
        sequence = tokenizer.encode(text)
        # Segment the sequence into overlapping parts.
        segments = segment_encoded_sequence(
            sequence, segmentation_length, n_overlap)
        # Add every segment (a list of ints) to encoded_tokens.
        encoded_tokens.extend(segments)

    # Create padded sequences one token longer than the maximum input length.
    padded_sequences = keras.preprocessing.sequence.pad_sequences(
            encoded_tokens,
            maxlen=segmentation_length,
            padding="post",
            value=pad_token_id)

    # Create inputs and targets from padded sequences.
    inputs = padded_sequences[:, :-1]
    targets = padded_sequences[:, 1:]
    return inputs, targets
```




