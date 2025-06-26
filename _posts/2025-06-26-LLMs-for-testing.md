---
layout: post
title:  "Experimenting with LLMs for semantic testing"
date:   2025-06-26 13:00:00
categories: LLMs, typescript, testing
---


A common problem when building modern websites with searches, content recommendations and personalisation is that standard unit and integration testing can fairly easily show if the site is working, functionally, but its much harder to show if these complex content structures return content as expected by the user, as that require an understanding of what the user is expecting and a semantic understanding of the content they get back.


> Semantic testing is a test design technique that focuses on evaluating the meaning and logical correctness of inputs and outputs based on the intended behavior of the system. It aims to detect errors related to misunderstandings of requirements, domain rules, or data interpretations rather than just syntactic mistakes.

### Some examples: 

1. If I search for “black dress shoes” does my store return fancy shoes, or black dresses? 
2. If the site does content recommendations, does these work well with the primary content and do they match what a given user is expecting?
3. Does the content in our articles and product pages follow the specified tone of voice

Using standard testing tools, you can get an indication, but would require html parsing, knowledge of the html element structure etc, to get a red/green judgement, hence we have historically used QA teams of humans to catch these kind of issues. 

Instead we can introduce an LLM into our testing setup, as I’m deploying my website in a container, it’s a low barrier to just use Jest, [Testcontainers](https://testcontainers.com/) and [Docker Model Runner](https://docs.docker.com/ai/model-runner/) which both run on my mac, and in CI/CD (with gpu available)

All snippets in this post are shortened down for brewity, but can be found in [this github repo](https://github.com/perploug/LLMs-for-semantic-testing)

## Setup
Get a local model running, and ensure it follows a specific output schema, for jest to be able to assert success/fail on.

```bash
docker model pull ai/qwen3
```

and then access it using the normal openai js library:

```typescript
const ai_client = new OpenAI({
    baseURL: "http://localhost:12434/engines/v1",
    apiKey: "NOTNEEDED",
});
const response = await ai_client.chat.completions.create({
    model: "ai/qwen3",
    messages: messages,
    temperature: 0.7,
});
```

Importantly, for all test responses I instructured the model to always respond in json using
a simple format that allows me to grade quality and get a reason for failing tests, this is
appended to all test prompts: 

```
  Only respond in json, with the following format: 
  {"rating": "good|medium|bad|", "reason": string }
```

For the testcontainers part, we simply reuse the existing compose file and get the hostname
and port of the site

```typescript
// load Umbraco into testcontainers
const env = await new DockerComposeEnvironment(
    `/path`,
    `docker-compose.yml`
).up();

let umbraco = env.getContainer("umbracodocker-1");
let siteurl = umbraco.getHost() + ":" + umbraco.getFirstMappedPort();
```


## Order of blog articles
Determine if all the articles linked on the page have fitting headlines and they are sorted in a way that makes logical sense either by date or alphabetically.

For this I created a small agent (narrator: this is not an agent)

```typescript

it("Blog posts should be listed with punchy headlines and sorted by date", async () => {
    let url = instance.baseURL + `/blog`;

    const persona = `You are an umbraco community member, with a developer
                    background, highly interested in community events
                    and eager to find out more about umbraco
                    `;

    const a = new persona_agent(persona);

    // fetches the url, prompts the LLM with the person and the intent, and has a html selector
    // to narrow down the html to examine (to reduce context side and hallucinations)
    // in this case the <article> element
    const result = await a.validateContentWithIntent(
    url,
    `I am browsing this page to find news about umbraco and its community
    I am only interested in click-baity and exciting news, if I see anything
    boring I will rage and quit the site - I expect blog posts to have interesting
    headlines and be sorted by date with the latest posts first`,
    "article"
    );

    expect(result.rating, result.reason).toBe("good");
});
```

What I personally find lovely here, is the sheer readability of test, I can see
what is happening, what the test is supposed to do, just by reading the prompt.

Jest runs, spins up Umbraco, performs the test and throws away the environment:

```bash
  console.log
      {
      rating: 'good',
      reason: "Posts have engaging headlines, sorted by date with latest first, aligning with the user's interest in community news."
      }

  PASS  src/tests/umbraco.content.quality.test.ts (31.092 s)
  Content Quality
      ✓ umbraco should run (2 ms)
      ✓ Blog posts should be listed with punchy headlines and sorted by date (24018 ms)

  Test Suites: 1 passed, 1 total
  Tests:       2 passed, 2 total
  Snapshots:   0 total
  Time:        31.147 s, estimated 36 s
  Ran all test suites.
```


### Tone of voice 
Okey, lets try something else, namely determining whether content on a page matches
the corporate tone of voice guidelines. To do this, we will provide the LLM with a styleguide from [mailchimp]() and then asking the test to validate html on a given url. 

```typescript
it("Tone of voice should be followed", async () => {
    let url =
    instance.baseURL + `/blog/join-the-umbraco-community-on-mastodon/`;

    const a = new toneOfVoice_agent(styleGuide);
    const result = await a.validateUrl(url);

    expect(result.rating, result.reason).toBe("medium");
});
```

This also passes, but if I add the requirement in the  styleguide that content should
always be written in danish, the test breaks:

```
  Language
      Mailchimp only publishes content in the danish language, 
      this is a critical requirement
```

Jest failure: 

```bash

 FAIL  src/tests/umbraco.content.quality.test.ts (25.976 s)
    Content Quality
        ✕ Tone of voice should be followed (19516 ms)

    ● Content Quality › Tone of voice should be followed
        Content is in English instead of Danish, which violates the critical language requirement. While the tone is clear and informative, it lacks the subtle humor and warmth specified in the styleguide.

        Expected: "medium"
        Received: "bad"
```

This took a couple of prompt tweaks, some times it started returning the reasoning in danish, some times it just ignored it, so
the quality of your test very much comes down to your prompt engineering ability. 


## Search Results
Next, lets try to validate the search results we get back, against a repository of matching and not-matching content,
For your specific use case, you can add in common edge cases, misspellings etc, the Umbraco out-of-the-box search isvery basic, so we will keep it basic as well:

For this we need the LLM for 2 things, one to generate content on flying cats, and one on flying cars, I want to ensure 
our search page correctly ranks pages, for both "flying", "cars" and "flying cars" 

I've created an editor agent, which I can give a writer persona and then instructions on what to write, reusing the styleguide 
alongside the expected output schema format that I want the content in. 

```typescript

  const a = new editor_agent(editor_persona, styleGuide, instance.baseURL);
  await a.authenticate();
  const content = await a.createContent(
    10,
    "Articles about the latest and greatest flying car and how it will change the world, title should be punchy and clickbaity, subtitle should be 100 words or less and repeat the main topic to optimise for search results, description should be 300 words and full of cool, innovative made up words, only use clear text without any formatting",
    `{"title": string, "subtitle": string "description": string}`
  );

  ... 

  let url = instance.baseURL + `/search/?q=flying-car`;
  const p = new persona_agent(persona);
  const result = await p.validateContentWithIntent(
    url,
    `I am searching for pages on 'flying cars' and expect to only see atleast 3 search results about flying cars.
    Given the user query and the list of search results (title + snippet), assess the relevance and rank appropriateness of each result.
    `,
    "#search"
  );

```

Validation is a bit crude here, and the prompts could likely be built out further, or give the LLM more context
for simplicity I used the same persona/intent validation, but search results validation could benefit from more specific
input on the expected behavior of the system.

```bash

 FAIL  src/tests/umbraco.content.quality.test.ts (17.41 s)
    Content Quality
        ✕ Search returns correctly ranked results (11229 ms)

    ● Content Quality › Search returns correctly ranked results

        Custom message:
        Only 1 result found, and it's about a flying car, not flying cats.
```

This is a very basic setup, to validate the ideas, but I believe this could be a useful tool, with a number of other test scenarios:

- Clarity of CTAs
- Bias checks
- Validate metadata with page content, title etc
- Compare content variants or translations
- Determine helpfullness of error messages/labels/etc

Again, source code is [here](https://github.com/perploug/LLMs-for-semantic-testing) 
