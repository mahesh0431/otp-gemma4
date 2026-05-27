# otp-gemma4

## Vision

`otp-gemma4` is a focused project for detecting OTPs from SMS-style messages more accurately, faster, and more efficiently.

The core idea is simple:

1. Establish a strong deterministic baseline.
2. Evaluate Gemma 4 E2B on OTP extraction.
3. Measure failures against real-looking SMS examples and hard negatives.
4. Improve extraction behavior with targeted fine-tuning.
5. Keep the final system small enough for local inference.

This is not just about extracting any number from a message. The target behavior is to identify the right OTP or verification code while avoiding false positives such as order IDs, dates, amounts, phone numbers, ticket IDs, and card-ending digits.

## Model

Primary model:

```text
google/gemma-4-E2B-it
```

Why this model:

- Smallest practical Gemma 4 model.
- Good fit for local inference when quantized.
- Strong enough for structured extraction tasks.
- Suitable for LoRA/QLoRA-based improvement experiments.

Known size notes:

```text
Gemma 4 E2B:
  2.3B effective parameters
  5.1B parameters with embeddings
  128K context
  text, image, audio input
```

Local inference target:

```text
bartowski/google_gemma-4-E2B-it-GGUF:Q4_K_M
```

The Q4_K_M GGUF is about 3.46 GB and is the right first local test target for this machine.

Sources:

- https://huggingface.co/google/gemma-4-E2B
- https://huggingface.co/docs/google-cloud/examples/vertex-ai-notebooks-fine-tune-gemma-4
- https://huggingface.co/bartowski/google_gemma-4-E2B-it-GGUF

## Local Machine Notes

Current local checks:

```text
Architecture: arm64
macOS: 26.4.1
Python: 3.13.5
uv: installed
Ollama CLI: installed
Ollama server: not currently running
llama.cpp: not installed
huggingface-cli: not installed
Free disk: about 21 GiB
Existing ~/.ollama usage: about 3.7 GiB
```

Practical direction:

- Use this laptop for local inference, evaluation, and failure analysis.
- Avoid full BF16 model downloads locally.
- Use quantized inference locally.
- Use cloud GPU only when model improvement requires training.

## Dataset

Start with:

```text
gandharvbakshi/SMS-dataset-sample-10k
```

Dataset page:

```text
https://huggingface.co/datasets/gandharvbakshi/SMS-dataset-sample-10k
```

Dataset use:

- Inspect schema and label quality first.
- Build train, validation, and test splits.
- Keep test data isolated for honest measurement.
- Add synthetic hard negatives where the public data is weak.

Suggested split:

```text
Train:      8,000 examples
Validation: 1,000 examples
Test:       1,000 examples
```

## Output Contract

The extractor should return strict JSON only.

OTP detected:

```json
{
  "otp": "839204",
  "intent": "login",
  "confidence": 0.98
}
```

No OTP detected:

```json
{
  "otp": null,
  "intent": null,
  "confidence": 0.0
}
```

Extraction policy:

- Extract only OTPs or verification codes.
- Do not extract order IDs.
- Do not extract ticket IDs.
- Do not extract dates.
- Do not extract phone numbers.
- Do not extract amounts.
- Do not extract card last-four digits unless the message clearly says it is an OTP/code.
- If multiple numbers exist, choose the number most semantically tied to OTP/code/verification/login/transaction language.
- If uncertain, return `otp: null`.
- Never invent a code.
- Return JSON only.

## Evaluation

Build evaluation before model changes.

The same eval set should compare:

```text
regex baseline
base Gemma 4 E2B
improved Gemma 4 E2B
```

Metrics:

```text
json_valid_rate
otp_exact_match_accuracy
no_otp_false_positive_rate
otp_false_negative_rate
intent_accuracy
multiple_number_policy_accuracy
average_latency_ms
tokens_per_second
```

Hard negatives:

```text
Order ID: 839204
Amount: Rs. 8392.04
Phone: 9876543210
Date: 25/05/2026
Card ending: 1234
Tracking ID: 839204
Use code 4419 to login. Order #839204 shipped.
Never share your OTP with anyone; bank employees will not ask for it.
Someone is asking for your OTP on a call.
```

## Local Inference

Preferred path:

```bash
ollama run hf.co/bartowski/google_gemma-4-E2B-it-GGUF:Q4_K_M
```

Alternative path:

```bash
brew install llama.cpp
llama-cli -hf bartowski/google_gemma-4-E2B-it-GGUF:Q4_K_M
```

Because local disk is tight, do not pull the model until the eval harness exists and the download is explicitly approved.

## Improvement Path

Phase 1: Baseline

- Inspect the SMS dataset.
- Build deterministic regex extraction.
- Create eval/test splits.
- Run baseline metrics.

Phase 2: Base model eval

- Run Gemma 4 E2B locally with the same prompts.
- Capture raw outputs.
- Measure JSON validity, exact OTP match, false positives, and latency.

Phase 3: Targeted improvement

- Prepare training examples from dataset rows.
- Add synthetic hard negatives.
- Use LoRA/QLoRA on cloud GPU if baseline errors justify it.
- Keep the same JSON output contract.

Phase 4: Re-evaluation

- Run the exact same evals after improvement.
- Compare against regex baseline and base model.
- Review failure cases manually.

Phase 5: Policy tuning

Only if needed:

- Create chosen/rejected pairs.
- Prefer correct OTP over nearby IDs or amounts.
- Penalize false positives on non-OTP numbers.
- Test DPO/ORPO-style preference tuning.

## Cloud GPU

Use local machine for evaluation and inference.

Use cloud GPU for training:

```text
L4 24GB: good first choice
A10G 24GB: good first choice
H100: fastest, usually unnecessary
T4 16GB: possible with aggressive QLoRA, but not ideal
```

Do not spend on training until the baseline evals show exactly what needs to improve.

## Next Steps

1. Create dataset inspection script.
2. Create eval split generator.
3. Create regex baseline.
4. Create model eval runner for Ollama or llama.cpp.
5. Run baseline eval on a small sample.
6. Pull the local quantized model only after the harness is ready.
7. Prepare cloud fine-tuning only if the evals show meaningful gaps.
