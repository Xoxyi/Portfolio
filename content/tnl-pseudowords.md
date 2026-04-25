+++
title = "Multilingual Pseudo-Words to Reduce Ambiguity"
date = 2025-01-01
description = "A study on how pairing a word with its translation into another language reduces semantic ambiguity, using WordNet synsets as the evaluation framework."
+++

# Multilingual Pseudo-Words to Reduce Ambiguity

**Authors:** Andrea Zito, Davide Camino  
**Course:** TNL Project — A.A. 2024-2025  
**Affiliation:** Department of Computer Science, University of Turin

---

## Abstract

We present a study on how the ambiguity of a word can be reduced by considering the pair consisting of the word and its translation into another language. The idea behind the study is that words in different languages are ambiguous in different ways, and consequently, the word-translation pair is less ambiguous than the word in either the first or the second language.

---

## Introduction

In this project, we explore a way to reduce the semantic ambiguity of words by leveraging their translations into other languages. The selection of words, their translation, and the evaluation of ambiguity are based on annotated resources, with all the strengths and weaknesses that these entail.

The idea underlying this study is that ambiguity is intrinsic to every language, but different languages are ambiguous in different ways. In this sense, the word *pitch* has many meanings, and so does the Italian word *tono* (loudness, shade, …); however, the compound (pseudo-word) **pitch-tono** points to a specific meaning of both the English word *pitch* and the Italian word *tono*.

This project is structured into four sections. We begin by presenting the resources used and explaining why we chose them, then we look at how to create a pseudo-word and how to evaluate the ambiguity of words and pseudo-words; finally, we present some concluding remarks summarizing the work.

---

## Resources

### WordNet

The main resource used in this work is WordNet in its multilingual version. This resource is particularly well-suited to our objective because it allows easy access to a set of meanings and how these meanings are expressed in various languages (i.e., the words). WordNet is built by expert annotators, so we can rely on the relationship between a meaning and the words associated with it. It also allows us to define very simple functions to evaluate the ambiguity of words and pseudo-words.

As for the languages selected, they were chosen based on the completeness of their representation in WordNet. The languages examined are:

| Language | WordNet Coverage |
|---|---|
| English | 100% |
| French | 92% |
| Italian | 83% |
| Finnish | 100% |

These languages are rich in meanings and ambiguous terms. Furthermore — a crucial detail for our study — there is significant overlap among all of them. Using languages poorly represented in WordNet would have led to an underestimation of the intrinsic ambiguity of the language, and if the overlap between languages is too small, the results regarding ambiguity reduction would also be skewed.

### Basic Level

While exploring WordNet, we realized that both among senses and among words, there are many highly specific terms. These words are naturally minimally ambiguous and, in most cases, if they exist in one language, they do not have translations in the others. To limit this phenomenon, it is advisable to consider only the most ambiguous words, which usually correspond to the most frequently used ones.

---

## Generating Pseudo-Words

In this section, we describe how to generate pseudo-words and how to assign them a meaning. To achieve this, we leverage the synsets defined by WordNet.

Consider the word *pitch*, which belongs to several synsets — some with the meaning of "to throw", others with the meaning of "tone", etc. Each synset represents a different meaning of *pitch*. Similarly, the Italian word *tono* belongs to several synsets, one for each of its meanings.

Since synsets represent the meaning of a word, they do not belong to any specific language. If two words (from the same or different languages) share the same meaning, they will belong to the same synset, and vice versa.

To obtain a pseudo-word, we simply start with a word and pair it with one of its possible translations. To assign one or more meanings to the pseudo-word, we consider the intersection of all the synsets that the word in the first language belongs to and all the synsets that the translated word belongs to.

We are confident that each pseudo-word has at least one meaning, since the translation process involves starting from a word in the first language, selecting one of its synsets, and then choosing a word from the second language that belongs to that same synset. This ensures that the intersection is never empty.

Our Python implementation for generating pseudo-words is the following:

```python
def get_common_synsets(wordL1, wordL2, lang1, lang2):
    return list(set(wn.synsets(wordL1, lang=lang1)) & set(wn.synsets(wordL2, lang=lang2)))

def create_pseudoword(wordL1, wordL2, lang1, lang2, inv, ambiguity_reduction):
    syns = get_common_synsets(wordL1, wordL2, lang1, lang2)
    if syns == []:
        return None
    else:
        if inv:
            return (wordL2, wordL1, [syn.name() for syn in syns],
                    ambiguity_reduction(wordL2, wordL1, len(syns), lang2, lang1))
        else:
            return (wordL1, wordL2, [syn.name() for syn in syns],
                    ambiguity_reduction(wordL1, wordL2, len(syns), lang1, lang2))
```

For each pseudo-word we save the word pair, all the meanings associated with it, and the reduction in ambiguity obtained — the latter is addressed in the next section.

---

## Ambiguity Reduction

To estimate the reduction in ambiguity, we used the formula provided in the assignment:

$$\text{AmbiguityReduction}(x, y) = \frac{|S_x| + |S_y| - 2 \cdot |S_{x \cap y}|}{|S_x| + |S_y|}$$

The formula gives a number between 0 and 1, where the higher the value, the greater the reduction in ambiguity. The code implementing it is a direct translation of the formula:

```python
def ambiguity_reduction(pseudoword):
    syn1  = len(wn.synsets(pseudoword[0], lang=l1))
    syn2  = len(wn.synsets(pseudoword[1], lang=l2))
    syn12 = len(pseudoword[2])
    return (syn1 + syn2 - 2 * syn12) / (syn1 + syn2)
```

A noteworthy limitation of the formula is that it does not penalize pseudo-words where only one of the component words appears in a large number of synsets.

---

## Results

### Word Ambiguity

To analyze the reduction in ambiguity in the creation of pseudo-words, we tested three combinations of languages:

- Italian — English
- Italian — French
- English — Finnish

With the provided notebook, it is possible to try any other pseudo-language combination. In addition to those above, Spanish, Swedish, and Japanese are also available.

Before comparing how pseudo-words behave in reducing ambiguity, it is worth examining the ambiguity of the original languages.

![Number of synsets for the top 1000 words](/images/tnl/Figure_1.png)

*Number of synsets for the top 1000 words in each language, ordered in non-increasing order. The pseudo-language curves (word pairs) are also shown, illustrating how much ambiguity is reduced by combining two languages.*

The graph shows the number of synsets for each word ordered in a non-increasing order. Having a greater number of synsets per word does not necessarily imply that the language is more ambiguous — it may reflect a greater effort by linguists in distinguishing different meanings in that language. This is visible in the graph, where Italian appears to be the least ambiguous of all, which is largely because it has the least WordNet coverage among the languages analyzed.

Contrary to our expectations, Italian-English exhibits greater ambiguity than Italian-French, despite the former being a combination of a Latin and a Germanic language, while the latter combines two Latin languages.

### Ambiguity Reduction

We now investigate how much the ambiguity of words is reduced when moving to pseudo-words. The graph shows the percentage of pseudo-words that achieved a given level of ambiguity reduction.

![Percentage of pseudo-words achieving a given ambiguity reduction](/images/tnl/Figure_2.png)

*Distribution of ambiguity reduction values across all pseudo-words, for each language pair. The x-axis represents the ambiguity reduction score (0 to 1) as defined by the formula above.*

We use percentages because the area under each histogram sums to 1. On the x-axis, the ambiguity reduction ranges from 0 to 1 according to the formula described above.

These graphs show that combining different languages can indeed significantly aid in disambiguation. The percentage of words with an ambiguity reduction of over 50% is substantial in all language pairs.

However, these graphs appear to contradict the earlier one: the best result comes from the combination of English and Finnish, not Italian and French. This is likely due to the fact that, being the most fully represented languages in WordNet, they benefit the most from the meaning intersection process.

Furthermore, it is important to note that the graphs do not represent exactly the same information: in the first figure, we focus on the 1000 most ambiguous words, while in the histograms, we consider the entire language.

It can also be noted in all three histograms that there is a very high percentage of pseudo-words that do not benefit from any ambiguity reduction. However, this percentage is overestimated. Due to the way the ambiguity reduction formula is constructed, when starting from two words that are already unambiguous (i.e., each belongs to only one synset), the formula returns 0. In a future analysis, it might be useful to filter out already unambiguous words and focus only on those with multiple meanings.

### Output Format

The dictionary of pseudo-languages is provided in a JSON file with the following structure:

```json
{
  "<synset01>": [
    ["<wordL1>", "<wordL2>", "<ambiguity_reduction>"],
    ["<wordL1>", "<wordL2>", "<ambiguity_reduction>"]
  ],
  "<synset02>": [
    ["<wordL1>", "<wordL2>", "<ambiguity_reduction>"],
    ["<wordL1>", "<wordL2>", "<ambiguity_reduction>"]
  ]
}
```

---

## Conclusions

This project demonstrates that pairing a word with its translation into another language is an effective strategy for reducing lexical ambiguity. By intersecting the synsets of two words across languages, the resulting pseudo-word inherits only the shared meanings, discarding the unrelated senses that each word carries individually.

The experiments confirm that the approach works across typologically diverse language pairs, with English-Finnish performing best among those tested — a result attributable to both languages enjoying 100% coverage in WordNet. The results also highlight an important limitation of the ambiguity reduction formula: it assigns a score of 0 to pairs where both component words are already monosemous, artificially inflating the count of "no reduction" cases. Filtering for genuinely polysemous words would provide a clearer picture in future work.

More broadly, the study points to an inherent limitation of any annotation-based approach: the results are only as reliable as the resource underlying them. WordNet's uneven language coverage means that low-coverage languages systematically appear less ambiguous, not because they are, but because fewer distinctions have been annotated. Expanding coverage or cross-validating with corpus-based measures of ambiguity would strengthen the analysis.

---

## References

1. C. Fellbaum (ed.), *WordNet: An Electronic Lexical Database*, MIT Press, 1998.
2. F. Bond and R. Foster, "Linking and extending an open multilingual wordnet," in *Proc. ACL 2013*, pp. 1352–1362.
