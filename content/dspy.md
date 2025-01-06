Title: Over the hump of DSPy 
Date: 2025-01-05 0:00

I finally got around to giving [DSPy](https://dspy.ai/) a real shot. I had heard some pretty polarizing opinions on it in the past. Some folks thought of it like another LangChain, overly complex and too much abstraction for a simple thing. The other camp stopped writing prompts by hand entirely and just DSPy use instead.  

My initial attempt a few months back to pick apart the (old) documentation lead to confusion and made my opinion lean towards the naysayers.  Signatures? Modules? It felt a bit abstract and the examples, at the time, were not very clear. However, coming back to this tool the past few days has been a much easier journey. The documentation is much cleaner and more consistent, and the examples are finally front and center.

My recommendation is to skip the [Learn DSPy](https://dspy.ai/learn/) documentation entirely when starting. Instead, work through a few of their [Tutorials](https://dspy.ai/tutorials/entity_extraction/).  The first tutorial is on RAG, and to be honest, doing yet another document retrieval based tutorial might induce permanent cranial damage to me at this point, so I skipped it.  I jumped ahead to the [entity extraction](https://dspy.ai/tutorials/entity_extraction/) example since it was much more relevant to my daily work, and it was incredible.

The example was generally pretty clear and concise, though I would recommend making sure you run `dspy.inspect_history(n=1)` after the initial test, before prompt optimization, to get a better feel for what has changed. That said, the workflow was great.  Being able to show up to a problem with a general statement or question, along with some labeled data, and automatically using that to bootstrap you way to a better prompt and few-shot examples is a dream. It will certainly save me a lot of time in the future.

The results were pretty impressive. Using `NousResearch/Meta-Llama-3-8B-Instruct` at `BF16` on `vLLM`, the initial accuracy was approximately 52%.  Not too surprising given the lack of context or examples for what really needs to be extracted.  Running the tuning step brought it up to 92% after DSPy improved the prompt and selected several few shot examples. Obviously 92% isn't anything to write home about for simple entity extraction models, but as a starting point from ~250 total labeled examples, its pretty good. Especially considering I did nothing except define an extremely simple signature.

I am looking forward to building out some ideas for a human in the loop system with DSPy, some LLMs, and some clustering models along with fine tuning a BERT model to see how quickly you can go from zero to cleaned data at scale.  These tools should really get us to a point where we can have a human focused on the decision boundary and not wasting their time elsewhere in the dataset.

---------

### Bonus

Reviewing the optimized prompt did provide a few laughts. I am not sure if the following is because I used a slightly dated Llama3 model instead of proprietary APIs, but I found it amusing to see that the idea of "saying your grandma will die if the LLM fails" is alive and well:

```
In adhering to this structure, your objective is: 
        In a high-stakes scenario where a critical decision is being made about the containment of a pandemic, extract all contiguous tokens referring to specific people from a list of string tokens, including their names, titles, and affiliations. Output a list of tokens, without combining multiple tokens into a single value. This information is crucial for identifying key individuals involved in the decision-making process and understanding their roles.
```