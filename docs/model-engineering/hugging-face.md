# Hugging Face

Use the Hub to discover models and datasets, inspect licenses and files, pin revisions, and reproduce experiments. A safe workflow is: identify task and constraints, read the model card, check license and data notes, pin a revision, run a small benchmark, then measure latency and cost.

```python
from transformers import pipeline

classifier = pipeline("sentiment-analysis", model="distilbert-base-uncased-finetuned-sst-2-english")
print(classifier("A small reproducible experiment."))
```

Do not upload private data or credentials. Treat model outputs as untrusted until evaluated.
