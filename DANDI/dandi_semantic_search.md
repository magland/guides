# DANDI Semantic Search

Search DANDI archive using natural language queries.

## Usage

```python
from z9_tools.dandi import dandi_semantic_search

dandiset_ids = dandi_semantic_search("olfactory bulb rat")
# Returns: ['001539', '001170', '001566', ...]
```

Returns a list of dandiset IDs ordered by relevance.
